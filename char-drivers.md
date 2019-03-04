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
