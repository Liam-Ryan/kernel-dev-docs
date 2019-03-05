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

Files in the kernel are actually represented by struct inode in `include/linux/fs.h` (not struct file!!!). There are 3 different struct pointers contained in the inode which will be set depending on the device type.
```c
struct inode {
	...
	struct pipe_inode_info *i_pipe;	//used if this is a pipe
	struct block_device *i_bdev; //used if this is a block
	structcdev *i_cdev; //used if this is a character device
}
```
`struct inode` is a filesystem data structure, `struct file` relies on the `struct inode` and specifically represents an open file in the kernel

```c
struct file {
	...
	struct path f_path; //file path
	struct inode *f_inode;
	const struct file_operations *f_op;
	loff_t f_pos; //position of the cursor in the file
	void *private_data; //driver-specific data
}
```

