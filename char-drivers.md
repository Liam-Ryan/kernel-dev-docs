## Character drivers

Character drivers are the simplest form of drivers. They transfer a stream of characters like a serial port. 
Character drivers are represented by cdev struct in the kernel in `include/linux/cdev.h`
```c
struct cdev {
	struct kobject kobj;
	struct module *owner;
	const struct file_operations *ops;
	struct list_head list;
	dev_t dev;
	unsigned int count;
};
```

### Major / Minor versions
If you `ls -l` in /dev you see many devices prepended with c, these are char devices. 
Char and block devices both have a major and minor version column under `ls -l`. For instance you might see
```
crw-rw----  1 root tty       7,   1 Mar  4 16:37 vcs1
```
In the snippet above 7 is major and 1 is minor. In the kernel the major and minor are both encoded in a `dev_t` which translates to a `u32` (unsigned 32 bit long)

The major version is used to register a device group or type and the minor version is used to index individual local devices registered to that type.

The major takes up the first 12 bits and the minor takes up the remaining 20 bits. There are several relevant macros in `include/linux/kdev_t.h` for working with `dev_t` 
```c
MAJOR(dev_t dev); //Extract the major version from dev_t
MINOR(dev_t dev); //Extract the minor version 

/* Here's how the kernel deals with dev_t and allows you to create a dev_t */
#define MINORBITS	20
#define MINORMASK	((1U << MINORBITS) - 1)

/* How minormask works 
** 1U is 1 as an unsigned long so in binary 00000000000000000000000000000001
** left shift by 20 bits = 00000000000100000000000000000000
** take away 1 means flip the bits from the current position(bit 12) 00000000000011111111111111111111
** you now have a mask which will give you the minor version when you & with a dev_t
*/

/* right-shifting by the minor mask drops off the minor bits leaving you with only the major bits */
#define MAJOR(dev)	((unsigned int) ((dev) >> MINORBITS))
#define MINOR(dev)	((unsigned int) ((dev) & MINORMASK)) // explained above under definition of MINORMASK

/* left-shift the major by the minor bits which leaves 0 in the lower 20 bits then or with the minor which will have 0 in the 12 most significant bits */
#define MKDEV(ma,mi)	((ma) << MINORBITS) | (mi))
```

### Device number allocation
If absolutely required you can try to assign to a major/minor yourself using the following but it's almost never correct to use 
`int register_chrdev_region(dev_t first, unsigned int count, char *name)`
first should be the major number you require and the first minor number, count is the total number of minor numbers.

The recommended way is to request a major from the kernel using alloc
`int alloc_chrdev_region(dev_t *dev, unsigned int firstminor, unsigned int count, char *name)`
Here dev is an output parameter which is the first number assigned by the kernel.
firstminor is where you want your minor numbering scheme to begin

In normal driver development you don't need to call alloc directly, there's frameworks for the various driver types that handle it

### Per device data with `open()`
Each `open()` on a character driver receives a `struct inode` parameter which is the kernel representation of that driver file. The inode has a `struct cdev i_cdev` field which points to the cdev we init via `cdev_init`. By embedding the cdev in our device data we can get a pointer to the data using `container_of` on that cdev.
```c
struct myfirstchardev {
	struct cdev firstdriver_cdev;
	unsigned char *data;
	int firstchardatasize;
	...
}

static unsigned int major = 0;
static size_t data_size = 64;
static struct class *myfirstchardev_class = NULL;

static int myfirstchardev_open(struct inode *inode, struct file *filp)
{
	unsigned int maj = imajor(inode);
	unsigned int min = iminor(inode);

	struct myfirstchardev *myfichdev = NULL;
	myfichdev = container_of(inode->i_cdev, struct myfirstchardev, firstdriver_cdev);
	myfichdev->firstchardatasize = data_size;
	if(maj != major || min < 0) {
		pr_err("device not found\n");
		return -ENODEV;
	}

	/* prepare the buffer if the device is opened for the first time */ 
	if (myfichdev->data == NULL) { 
		myfichdev->data = kzalloc(myfichdev->firstchardatasize, GFP_KERNEL); 
		if (myfichdev->data == NULL) { 
			pr_err("Open: memory allocation failed\n"); 
			return -ENOMEM; 
		} 
	} 
	filp->private_data = myfichdev; 
	return 0; 
}

```

### release()
Release is where you free up any resources allocated to the driver and shut down the device if it can be shut down

```c
static int mydev_release(struct inode *inode, struct file *filp)
{
	struct myfirstchardev *device = NULL;
	device = container_of(inode->i_cdev, struct my_device, firstdriver_cdev);

	mutex_lock(&device_list_lock);
	filp->private_data = NULL;

	myfirstchardev->users--;
	if (!myfirstchardev->users) {
		kfree(mydevbuffer);
		mydevbuffer = NULL;
		kfree(myfirstchardev->data);
		myfirstchardev->data = NULL;
		...
		
		if (global_struct)
			kfree(global_struct);
	}
	mutex_unlock(&device_list_lock);
	return 0;
	
}
```

### write() 
`ssize_t (*write) (struct file *filp, const char __user *buf, size_t count, loff_t *pos)`
* return value is # of bytes written
* `*buf` is the data buffer from user space
* count is the size of the transfer
* pos is the position in the file to start writing to

##### Best practice 
* Make sure requests are valid - don't trust user input
* Ensure that you have sufficient device space for any write
* Use `copy_from_user`first to a buffer and then move data again to the device
* Track the position in the file (pos), make sure to update `filp->f_pos`
* When writing to the device return the number of bbytes written and return a negative error on failure

### read()
`ssize_t (*read) (struct file *filp, char __user *buf, size_t count, loff_t *pos)`
* return value is bytes read 
* All other variables are the same as for write with pos showing where to start the read from

##### Best practice
* Check if you're going to read beyond the end of the file and just return to the end
* Otherwise follow the same best practices as for `write()`

### llseek()
`loff_t (*llseek) (struct file *filp, loff_t offset, int whence)`
llseek is used to move the cursor within a file. The classic use case is floppy drives

* returns the new position in the file
* `offset` is the offset relative to whence
* `whence` defines where to seek from
  * `SEEK_SET` - beginning of file
  * `SEEK_CUR` - current file position
  * `SEEK_END` - end of file

##### Best practice

* Check for all 3 whence types and handle
* make sure the new position is valid
* Update filp `f_pos` with the new position
* Return the new position

### poll()
`unsigned int (*poll) (struct file *, struct poll_table_struct *)`
polling is used by user space to check if a device is available. userspace can call `poll()` or `select()`, both are serviced by poll

Within the driver you need to call `void poll_wait(struct file *filp, wait_queue_head_t * wait_address, poll_table *p)`from `<linux/poll.h>`. This adds the device associated with filp to a list of devices which can wake processes sleeping in the `wait_address` based on events registered in the `poll_table` struct

Usually you use a wait queue for each event type (usually reading and writing) based on the events from userspace call

The return value must have POLLIN | POLLRDNORM set if there's data to read, POLLOUT | POLLWRNORM if it's writeable and 0 if there's no data and the device is not writeable

If poll is not defined it is assumed the device is readable and writeable at all times

##### Best practice

* declare a wait queue for each event type (read, write, error) you need a passive wait for
* Notify the wait queue when there is new data or device becomes writeable (using `wake_up_interruptible`)

### ioctl()
`long ioctl(struct file *f, unsigned int cmd, unsigned long arg)`

The ioctl method is used to handle device-specific system calls like shutdown, reset, etc.
The ioctl command needs to be identified by a unique number which is generated using one of the 4 macros 
* `_IO(MAGIC, SEQ_NO)` - doesn't need data transfer
* `_IOW(MAGIC, SEQ_NO, TYPE)` - needs write params(`copy_from_user` or `get_user`
* `_IOR(MAGIC, SEQ_NO, TYPE)` - needs read commands(`copy_to_user` or `put_user`
* `_IORW(MAGIC, SEQ_NO, TYPE)` - needs both

`MAGIC` is an 8 bit number (0-255)
`SEQ_NO` is a command id, also 8 bits
`TYPE` is an optional data type that will tell the kernel about the size to be copied

##### Best practice
* Generate the ioctl number in a dedicated header file since userspace will need the header to call the ioctl 
* multiple arguments should be gathered in a struct and the pointer passed to ioctl as the ulong

###File operations structure
Once you have defined your function you should use designated initializers to intialize the fops struct 
```c
static const struct file_operations my_fops = {
	.owner =	THIS_MODULE,
	.read =		my_read,
	.write =	my_write, 
	...
};
```
