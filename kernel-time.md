##### Also see Documentation/timers/timers-howto.txt in kernel src
## Types of time

Kernel has two types of time - real time and relative time. 
Real(absolute) time uses the real time clock.
Relative time uses the CPU timer and are called kernel timers
There are two types of kernel timers
* Standard/system timers
* High resolution timers

### Standard / system timers & Jiffies
Standard timers use "jiffies" which is defined in linux/jiffies.h
Jiffy time length is based on the constant HZ (think hertz) 
HZ is the number of times a jiffy is increased per second and depends on both the hardware and kernel version
HZ sets how frequently the clock interrupt fires
The programmable interrupt timer (PIT) is a hardware component which provides this
Jiffies can overflow, on 32 bit systems 1000HZ overflows every 50 days, therefore a second variable is in jiffies/linux.h - jiffies\_64
For 32 bit systems jiffies points to the low order 32 bits and jiffies\_64 points to high order. 
For 64 bit systems jiffies = jiffies\_64

#### Timer API
A timer is represented as a timer\_list
```c
#include <linux/timer.h>

struct timer_list {
	struct list_head entry;
	unsigned long expires; /* Absolute value in jiffies */
	struct tvec_t_base_s *base;
	void (*function) (unsigned long);
	unsigned long data; /* optional, passed to callback */
}
```

To set up the timer you can use setup\_timer or init\_timer
```c
void timer_setup(struct timer_list *timer, void (*function)(struct timer_list *timer), unsigned long data);

void init_timer(struct timer_list *timer);
```

You then need to set the expiration before the callback is fired
```c
int mod_timer(struct timer_list *timer, unsigned long expires);
```

When you no longer need the timer you need to release it like freeing
```c
void del_timer(struct timer_list *timer);
int del_timer_sync(struct timer_list *timer);
```
`del_timer_sync` waits for the handler to finish so if you hold a lock on the handler you will deadlock. You should release the timer in the module `__exit` or cleanup.

to see if the timer is running or not use
```c
int timer_pending(const struct timer_list *timer);
```
It tells you if there are any fired timer callbacks pending

#### Timer example module
```c
#include <linux/kernel.h>
#include <linux/module.h>
#include <linux/timer.h>

static struct timer_list timerlst;

static void timer_callback(struct timer_list *list)
{
	printk("%s called (%ld).\n", __FUNCTION__, jiffies);
}

static int __init onload(void)
{
	int res;
	printk("Timer module loaded\n");

	/*Set up timer, see commit e99e88a9d2b067465adaa9c111ada99a041bef9a for api change */
	timer_setup(&timerlst, timer_callback, 0);
	printk("Setup timer to fire in 500ms (%ld)\n", jiffies);

	/*mod_timer updates the expires for the timer_list*/
        res = mod_timer(&timerlst, jiffies + msecs_to_jiffies(500));
	if (res) 
		printk("Update timer failed\n");
	return 0;
}

static void __exit onunload(void)
{
	int res;
        res = del_timer(&timerlst);
	if (res)
		printk("Timer is still in use...\n");
	printk("Timer module unloaded\n");
}

module_init(onload);
module_exit(onunload);
MODULE_AUTHOR("Liam Ryan <liamryandev@gmail.com>");
MODULE_LICENSE("GPL");
```

### High resolution timers (HRTs)
For real time applications you should use HRTs
HRTs must be enabled via Kconfig `CONFIG_HIGH_RES_TIMERS`
HRTs have a resolution of nano/microseconds depending on arch where standard timers have a resolution of milliseconds
HRT implementation is based on ktime rather than jiffies
HRTs need architecture-dependent code for your hardware in order to be used

#### HRT API
HRTs use a hrtimer struct which is in `<linux/hrtimer.h>`
```c
struct hrtimer {
	struct timerqueue_node node;
	ktime_t _softexpires;
	enum hrtimer_restart (*function)(struct hrtimer *);
	struct hrtimer_clock_base *base;
	u8 state;
	u8 is_rel
};
```

To initialise the hrtimer you need to set up a ktime ( ktime represents time duration)
```c
void hrtimer_init(struct hrtimer *time, clockid_t clock_type, enum hrtimer_mode mode);
```

After initialising you need to start by calling `hrtimer_start`
```c
int hrtimer_start(struct hrtimer *timer, ktime_t time, const enum hrtimer_mode mode);
```
For both init and start the mode is the expiry mode ( `HRTIMER_MODE_ABS` for absolute, `HRTIME_MODE_REL` for a value relative to nowl

Cancelling a hrtimer 
```c
int hrtimer_cancel(struct hrtimer *timer);
int hrtimer_try_to_cancel(struct hrtimer *timer);
```
`hrtimer_cancel` will wait until the callback finishes, try to cancel will return -1 if timer is active or the callback is running

You can check if a hrtimer callback is running with 
```c
int hrtimer_callback_running(struct hrtimer *timer);
```

You need to have the callback return `HRTIMER_NORESTART` to prevent the timer from automatically restarting

You can check if HRTs are available on a system by 
* checking the kernel config file for something like `CONFIG_HIGH_RES_TIMERS=y: zcat /proc/configs.gz | grep CONFIG_HIGH_RES_TIMERS`
* `cat /proc/timer_list | grep resolution` shows 1nsecs and `event_hander` shows `hrtimer_interrupts`
* checking the `clock.getres` system call
* within kernel code using `#ifdef CONFIG_HIGH_RES_TIMERS`

## Tickless kernel
Normally the kernel is interrupted HZ times per second to reschedule tasks, even if it's idle. This increases power consumption
A tickless kernel disables these ticks and instead schedules ticks based on the next action
When the run queue is empty in a tickless kernel the scheduler switches to idle thread and disables periodic ticks until next timer expires
The kernel maintains a list of task timeouts and if the next tick is further away than the lowest timeout for tasks the kernel programs the timer with the timeout value
After the timer expires the kernel reverts to periodic ticks and invokes the scheduler, then when the run queue is empty again we repeat this process

## Delays and sleeps
There are 2 types of delays, one for atomic and one for nonatomic code. The delay header is `#include <linux/delay.h>`

### Atomic
Atomic tasks can't sleep and can't be scheduled ( eg ISR (interrupt handlers / Interrput Service Routines)
busy-wait loops are used to delay atomic code and are exposed via the `Xdelay` functions in the kernel
the code spends a set amount of time (based on jiffies) in the loop before returning for execution again
```c
ndelay(unsigned long nsecs)
udelay(unsigned long usecs)
mdelay(unsigned long msecs)
```
udelay should always be used since ndelay() depends on how accurate the hardware timer is and mdelay is discouraged as it's not very accurate in kernel timelines
timer handlers (callback functions) are atomic and can't sleep which *includes things resulting in sleep like memory allocation and locking a mutex*

### Nonatomic
Nonatomic code can use the sleep family of functions, which function you use depends on how long to sleep
< ~10 microseconds `udelay(unsigned long usecs)` - note this is the same as used for atomic context and uses a busy-wait loop
10 microseconds to 20 milliseconds `usleep_range(unsigned long min, unsigned long max)` - uses hrtimers so needs support for real time
10+ milliseconds `msleep(unsigned long msecs)` - jiffies and legacy timers, use for larger sleeps
