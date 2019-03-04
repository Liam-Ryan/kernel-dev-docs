### Device files (/dev)
Files in /dev are the driver device files. 
There are a variety of operations that can be performed on these files and a driver needs to implement the callback for a particular function to support it. 
The operations interface is in `include/linux/fs.h` and is struct `file_operations`. Some of the most important and often used operations are listed below
```c
struct file_operations {
	struct module *owner;
	loff_t (*llseek) (struct file *, loff_t, int);
	ssize_t (*read) (struct file *, char __user *, size_t, loff_t *);
	ssize_t (*write) (struct file *, const char __user *, size_t, loff_t *);
	unsigned int (*poll) (struct file *, struct poll_table_struct *);
	int (*mmap) (struct file *, struct vm_area_struct *);
	int (*open) (struct inode *, struct file *);
	long (*unlocked_ioctl) (struct file *, unsigned int, unsigned long);
	int (*release) (struct inode *, struct file *);
	int (*fsync) (struct file*, loff_t, loff_t, int datasync);
	int (*fasync) (int, struct file *, int);
	int (*lock) (struct file *, int struct file_lock *);
	int (*flock) (struct file *, int, struct file_lock *);
	...
};
```
Every callback is linked to a system call and all are optional. If userspace calls a system call which correspondes to an unimplemented callback the kernel returns an error code that varies based on the system call. 

Files in the kernel are actually represented by struct inode in `include/linux/fs.h` (not struct file!!!)


