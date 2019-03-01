
### Types of time

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
