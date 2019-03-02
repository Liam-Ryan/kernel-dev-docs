## Kernel locking 

The most important types of locks for driver development are Mutex and Spinlock. 
Semaphore is also very widely used in the kernel but not in drivers

### Mutex locks
Use lists internally similar to wait queue
```c
struct mutex {
	/* 1 unlocked, 0 locked, < 0 locked with possibly other processes waiting for lock */
	atomic_t count;
	spinlock_t wait_lock;
	struct list_head wait_list;
	...
};
```
Other processes that want to acquire the mutex are removed from the scheduler and put on the wait list in a sleep state
The kernel continues scheduling other tasks
When the lock is released the kernel wakes something on the wait queue, removes it from wait and schedules it

#### Working with mutexes
Rules for mutex use are in `include/linux/mutex.h` and include
* Multiple unlocks are not permitted
* Must use api to initialize
* May not exit while holding the mutex
* never free areas containing held locks
* never reinitialize a held lock
* cannot be used in atomic code since they involve rescheduling

Mutexes do not poll, the only way a lock is freed and passed to another waiting thread is when the thread with the lock releases it

You can declare mutex locks in 2 ways
```c
/* Static */
DEFINE_MUTEX(my_mutex);

/* Dynamic */
struct mutex mtxlck;
mutex_init(&my_mutex);
```

Locking
```c
/*Dangerous - will not return if a signal is received, even ctrl+c*/
void mutex_lock(struct mutex *lock);
/*Prefer using interruptible as it allows driver to be interrupted by any signal*/
int mutex_lock_interruptible(struct mutex *lock);
/*Only signals that kill the process can interrupt*/
int mutex_lock_killable(struct mutex *lock);
```

Unlocking
```c
void mutex_unlock(struct mutex *lock);
```

Check if locked
```c
int mutex_is_locked(struct mutex *lock);
```

Acquire lock if available
```c
/* returns 1 if it acquires the lock, 0 if it can't*/
int mutex_trylock(struct mutex *lock);
```

Sample mutex
```c
struct mutex mtxlck;
mutex_init(&mtxlck);

/*inside work or thread*/
mutex_lock(&mtxlck);
access_shared_resource();
mutex_unlock(&mtxlck);
```

### Spinlocks
Spinlocks have two states - locked and unlocked
A thread that needs to acquire a spinlock needs to active-loop until it's acquired when it breaks the loop
Spinlocks are heavily CPU intensive and should only be used for very quick acquires
Ideally the time to hold the spinlock is less than the time to reschedule and they should be released asap
When a process with a spinlock is running the kernel disables preemption which means the holder can't move off the run queue
This means that the kernel just disables preemption in response to `spin_lock(spinlock_t *lock` on a single core system

Since you don't know in advance how many cores a system will have you should use `spin_lock_irqsave(spinlock_t *lock, unsigned long flags)` before taking the spinlock and unlock with `spin_unlock_irqrestore()`

Example
```c
/*declared anywhere*/
spinlock_t my_spinlock;
spin_lock_init(my_spinlock);

static irqreturn_t irq_handler(int irq, void *data)
{
	unsigned long status, flags;
	spin_lock_irqsave(&my_spinlock, flags);
	status = access_shared_resource();

	spin_unlock_irqrestore(&gpio->slock, flags);
	return IRQ_HANDLED;
}
```

### Spinlocks vs Mutexes
* Mutexes protect process critical resources, Spinlocks protect IRQ handler critical sections
* Mutexes put contenders to sleep where spinlocks infinitely spin a loop until the lock is acquired
* Spinlocks should never be held for a long time

##### Note that with spinlocks preemption is disabled for threads holding spinlocks, not waiters that are spinning
