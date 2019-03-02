## Kernel work deferral mechanisms
There are 3 ways for deferred work to be carried out in the kernel 
* softirq provided by ksoftirqd (software interrupts) *atomic*
* tasklets (instances of ksoftirqd) *atomic*
* Workqueues 

### SoftIRQs & ksoftirqd
Only used for very fast processing
You almost never want to deal with softirq directly - only network and block devices really need it
Tasklets are an instantiation of softirqs and are sufficient in most cases

Softirqs are usually scheduled in hardware interrupts by an interrupt handler. Ksoftirqds are responsible for late execution
There is a ksoftirqd per cpu kernel thread which handles unserved softirqs
softirqs are in `kernel/softirq.c`

### Tasklets
Tasklets are a bottom-half mechanism built on top of softirqs

```c
struct tasklet_struct {
	struct tasklet_struct *next;
	unsigned long state;
	atomic_t count;
	void (*func)(unsigned long);
	unsigned long data;
};
```
Tasklets are not re-entrant by nature (re-entrant - can be interreputed and safely resumed)
Tasklets are designed such that a tasklet can run on only one core at a time, even on a SMP system, which is the core it was scheduled on.
Different tasklets may run simultaneously on different CPUs

Declaring a tasklet
```c
/* Dynamic */
void tasklet_init(struct tasklet_struct *t, void(*func)(unsigned long), unsigned long data);

/* Statically */
DECLARE_TASKLET(tasklet_name, tasklet_function, tasklet_data);
DECLARE_TASKLET_DISABLED(name, func, data);
```
`tasklet_init` creates a disabled tasklet, `DECLARE_TASKLET` creates an enabled tasklet which you need to disable first
A disabled tasklet has the count field set to 1, an enabled one has it set to 0

enabling and disabling tasklets
```c
/* enable */
void tasklet_enable(struct tasklet_struct *);

/* disable, return when tasklet terminates */
void tasklet_disable(struct tasklet_sruct *);

/* disable, return immediately(tasklet may still be running) */
void tasklet_disable_nosync(struct tasklet_struct *);
```

Scheduling Tasklets
```c
/* normal */
void tasklet_schedule(struct tasklet_struct *t);

/* high priority */
void tasklet_hi_schedule(struct tasklet_struct *t);
```
Normal priority and high priority tasklets live in two different lists and the function you use changes the list it's added to

Kill a tasklet(stops it from running again or waits before killing it if it's already scheduled)
```c
	void tasklet_kill(struct tasklet_struct *t);
```

#### Special tasklet properties
* Calling `tasklet_schedule` on an already-scheduled tasklet which has not started executing has no effect, it still runs only once
* `tasklet_schedule` can be called in the tasklet, rescheduling itself
* high priority tasklets are always executed before normal ones, only use them for very quick functions

#### Tasklet sample
```c
#include <linux/kernel.h>
#include <linux/module.h>
#include <linux/interrupt.h> 

char tasklet_data[] = "Using a string but it could easily be struct pointer\n";

/* Tasklet handler, just prints data held in the tasklet */
void tasklet_work(unsigned long data)
{
	printk("%s\n", (char *) data);
}

DECLARE_TASKLET(my_tasklet, tasklet_work, (unsigned long) tasklet_data);

static int __init onload(void)
{
	tasklet_schedule(&my_tasklet);
	return 0;
}

void onunload(void)
{
	tasklet_kill(&my_tasklet);
}

module_init(onload);
module_exit(onunload);
MODULE_AUTHOR("Liam Ryan <liamryandev@gmail.com>");
MODULE_LICENSE("GPL");
```

### Work Queues
Work queues are the only pre-emptible deferring task in the kernel meaning if you need to sleep in the bottom-half you use work queues
Work queues are built on top of kernel threads 
There's 2 ways to deal with kernel work queues
* default shared work queue - handled by a set of kernel threads on cores
* run the work queue in a dedicated kernel thread where your thread is woken any time the queue handler is called

#### Global work queue (shared queue)
* The global work queue should be the default choice for submitting occasional tasks
* Since it's shared make sure not to monopolize the system
* Don't sleep for a long time because no other task in the queue will run until you wake up

scheduling work on the global queue 
```c
/* schedule on current cpu */
int schedule_work(struct work_struct *work);
/* with delay */
static inline bool schedule_delayed_work(struct delayed_work *dwork, unsigned long delay);

/* schedule on particular cpu */
int schedule_work_on(int cpu, struct work_struct *work);
/* with delay */
int scheduled_delayed_work_on(int cpu, struct delayed_work *dwork, unsigned long delay);
```
The system shared queue is `system_wq` defined in `kernel/workqueue.c`
```c
stuct workqueue_struct *system_wq ))read_mostly;
EXPORT_SYMBOL(system_wq);
```

Work already submitted to the global queue can be cancelled with `cancel_delayed_work`
You can flush the queue with `void flush_scheduled_work(void);` but that could take any amount of time

#### Global work queue example
```c
#include <linux/kernel.h>
#include <linux/module.h>
#include <linux/sched.h>
#include <linux/wait.h>
#include <linux/time.h>
#include <linux/delay.h>
#include <linux/slab.h>
#include <linux/workqueue.h>

static int sleep = 0;

struct work_data {
	struct work_struct my_work;
	wait_queue_head_t my_waitqueue;
	int struct_data;
};

static void work_handler(struct work_struct *work)
{
	struct work_data *this_data = container_of(work, struct work_data, my_work);
	printk("Work queue module handler %s, data is %d\n", __FUNCTION__, this_data->struct_data);
	msleep(2000);
	/* comment sleep = 1 to wait indefinitely until ctrl+c interrupts*/
	sleep = 1;
	wake_up_interruptible(&this_data->my_waitqueue);
	kfree(this_data);
}

static int __init onload(void)
{
	struct work_data *my_data;

	my_data = kmalloc(sizeof(struct work_data), GFP_KERNEL);
	my_data->struct_data = 42;

	INIT_WORK(&my_data->my_work, work_handler);
	init_waitqueue_head(&my_data->my_waitqueue);

	schedule_work(&my_data->my_work);
	printk("Going to sleep...\n");
	wait_event_interruptible(my_data->my_waitqueue, sleep != 0);
	printk("I have awoken...\n");
	return 0;
}

static void __exit onunload(void) 
{
	printk("Work queue module exit: %s %d\n", __FUNCTION__, __LINE__);
}

module_init(onload);
module_exit(onunload);
MODULE_AUTHOR("Liam Ryan <liamryandev@gmail.com>");
MODULE_LICENSE("GPL");
```

