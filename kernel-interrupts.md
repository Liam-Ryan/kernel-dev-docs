## Interrupts
To listen for interrupts from a device you need to register to the IRQ, providing an interrupt handler function

### Registering an interrupt handler
```c
/* declared in linux/interrupt.h */
int request_irq(unsigned int irq, irq_handler_t handler,
	unsigned long flags, const char *name, void *dev)
```
* returns 0 on success
* flags are a bitmask of the masks in `<linux/interrupt.h>`, some examples-
	`IRQF_TIMER` - came from a system timer interrupt
	`IRQF_SHARED` - interrupt line can be shared
	`IRQF_ONESHOT` - don't re-enable the interrupt after handler finishes
	
* name identifies your driver in `/proc/interrupts` and `/proc/irq`
* dev passed as an argument to the handler, usually a device-specific struct
```c
struct dev_data {
	struct device *device;
	char name[64];
	char addr[32];
};

static irqreturn_t irq_handler(int irq, void *dev) 
{
	struct dev_data *devdata = dev;
	unsigned char nextstate = read_state(lp);
	/* Check whether this device raised the irq or not */
	...
	return IRQ_HANDLED;
}

/* somewhere in the code in the probe function */
int ret;
struct dev_data *dev;
dev = kzalloc(sizeof(*md), GFP_KERNEL);

ret = request_irq(client->irq, irq_handler, IRQF_TRIGGER_LOQ | IRQF_ONESHOT, DRV_NAME, dev);

/* in the release function */
free_irq(client->irq, dev);
```


#### Handler
handler is an `irq_handler_t` defined in `linux/interrupt.h` which is a pointer to a function returning `irqreturn_t`. eg. `static irqreturn_t my_handler(int irq, void *dev)`

Both params are passed to the handler and it can only return `IRQ_NONE`(I'm not the device that caused the interrupt) or `IRQ_HANDLED` ( I did the interrupt)

You can use `IRQ_RETVAL(val)` macro which returns `IRQ_HANDLED` for nonzero or `IRQ_NONE`

handlers don't need to worry about re-entry since the IRQ line is disabled while it's running

To free a registered handler call `void free_irq(unsigned int irq, void *dev`)
If the irq is not shared it will disable the line. If it's shared the dev device is removed but the IRQ stays active until the last dev is removed. `free_irq`` will block until executing interrupts complete so do not have request_irq and free_irq in the interrupt context or you will lock

#### Locking in a handler

An interrupt handler by its nature cannot be pre-empted so only use spinlocks
There is a special type of spinlock which must be used for global data accessible in user code(system call) and interrupt code - `spin_lock_irqsave()`. This is to ensure a user process is not pre-empted by an interrupt handler while updating shared memory that the interrupt handler will access. `spin_lock_irqsave` needs to be called by the user code and it will disable all interrupts on the local CPU until the lock is released.
`spin_lock_irqsave()` should also be used when sharing data between interrupt handlers ( same driver managing multiple devices )
```c
ssize_t read(struct file *filep, char __user *buf, size_t count, loff_t *filepos)
{
	unsigned long flags;
	...
	spin_lock_irqsave(&my_lock, flags);
	data++;
	spin_lock_irqrestore(&my_lock, flags);
	...
}

static irqreturn_t interrupt_handler(int irq, void *dev)
{
	/* IRQ handlers can't be preempted so only need spin_lock */
	spin_lock(&my_lock);
	...
	spin_unlock(&my_lock);
	return IRQ_HANDLED;
}
```

### Bottom and Top Halves
Since IRQs disable incoming interrupts and can't be pre-empted we need a way to make sure we don't miss interrupts while a handler is running.
To do this the kernel uses "halves"
* The top half or hard IRQ acts as quickly as possible to do the minimum required, schedules the bottom half and acknowledges the irq line. All disabled interrupts myst be re-enabled just before exiting bottom half
* The bottom half takes care of anything time-consuming and runs with interrupts re-enabled.
* Bottom halves use deferral mechanisms softirqs, tasklets, workqueues and threaded irqs

Softirqs and tasklets execute in software interrupt context where preemption is disabled. 
Workqueues and IRQs can usually be preempted but you can change properties using `CONFIG_PREEMPT` or `CONFIG_PREEMPT_VOLUNTARY` behaviour

Bottom halves are not always possible but when they are you should use them

### Tasklets as bottom halves example
```c
struct dev_data {
	int i;
	struct tasklet_struct *tasklet;
	int dma_request;
};

static void tasklet_work(unsigned long data)
{
	...
}

stuct dev_data *devdata = init_devdata;

/* in probe or init function */
	...
	tasklet_init(&dev_data->tasklet, tasklet_work, (unsigned long) dev_data;
	...

static irqreturn_t my_irq_handler(int irq, void *devdata)
{
	struct data *dev_data = devdata;
	tasklet_schedule(&dev_data.tasklet);

	return IRQ_HANDLED;
}
```

### Work queues as bottom halves example
```c
static DECLARE_WAIT_QUEUE_HEAD(my_wait_queue);
static struct work_struct work;

/* in the probe function */
INIT_WORK(work, work_handler);

static irqreturn_t interrupt_handler(int irq, void *devdata) 
{
	uint32_t val;
	struct dev_data = devdata;
	val = devdata->wake;
	if (val) {
		my_data->done = true;
		wake_up_interruptible(&my_wait_queue);
	} else {
		schedule_work(&work);
	}
	return IRQ_HANDLED
}
```
here we either wake a process on the wait queue or schedule work depending on value of `devdata->wake`

### Threaded IRQs

By using `request_threaded_irq` the core schedules the bottom half in a dedicated kernel thread
```c
int request_threaded_irq(unsigned int irq, irq_handler_t handler, irq_handler_t thread_fn, unsigned long irqflags, const char *devname, void *devdata);
```
handler is the same as `request_irq` except that if it can handle the request within itself (the top half) it returns `IRQ_HANDLED` but if it needs the bottom half it returns `IRQ_WAKE_THREAD` which results in the core scheduling the `thread_fn`
`thread_fn` is the bottom half, it must return `IRQ_HANDLED` when complete. After being executed it won't be rescheduled it needs to be called again by handler.

Threaded irqs can be used whereever we'd have used the work queue. If you only want to provide a bottom half you can provide a null handler and non-null `thread_fn`

```c
static irqreturn_t irq_handler(int irq, void *devdata)
{
	struct custom_data *dd = devdata;
	unsigned char nextstate = read_state(dd);
	
	if (dd->laststate != nextstate) {
		//do things
	}
	return IRQ_HANDLED;
}

static int probe(struct i2c_client *client, const struct i2c_device_id *id)
{
	struct custom_data *lp = init_custom_data();
	ret = request_threaded_irq(client->irq, NULL, irq_handler, IRQF_TRIGGER_LOW | IRQF_ONESHOT, DRV_NAME, lp);
	if(ret) {
		dev_err(&client->dev, "IRQ %d is not free\n", client->irq_);
		goto fail_free_device
	}
	ret = input_register_device(idev);
	...
}
```
When a handler is executed the serviced IRQ is always disabled on all CPIs and re-enabled after the handler(top half) finishes. However if you need the IRQ line to stay disabled for the bottom half you need to `request_threaded_irq` with `IRQF_ONESHOT` enabled as shown above. The IRQ line re-enables after the bottom half if you do
