### Kernel Userspace data exchange
When drivers `write()` they are reading data from userspace to kernel space and vice versa when they `read`.Userspace is untrusted and the kernel needs to know when you are dealing with userspace data. 

The `__user` tag is a 'cookie' used by sparse, the kernel semantics tool. Tagging something with user lets the kernel flag that you should use the specific kernel functions for dealing with userspace memory

```c
unsigned long copy_from_user(void *to, const void __user *from, unsigned long n);
unsigned long copy_to_user(void __user *to, const void *from, unsigned long n);
```
Note the sparse cookie used for each of the userspace fields within the functions themselves. For both functions n is the number of bytes to copy and the returned long is the number of bytes that couldn't be copied (should be 0 on success). 
For `copy_to_user` if the kernel can't copy all bytes it will pad the copied data using zero bytes until it gets the desired size in the destination address.

For simple variable types you can use the macros `put_user(x, ptr)` and `get_user(x, ptr)`
`put_user` copies the value x to the address of ptr in user space. The pointers must be of same type and returns 0 on success.
`get_user` copies the value x to the address of the ptr in kernel space. Same rules as put but in addition if there is a failure x is set to 0 for the address given.

#### `open()`
`int (*open) (struct inode *inode, struct file *filp);`
Calls to open will always be successful if it's not defined in your `file_operations` struct.
This method is called every time someone opens the device file. 
It usually performs init and returns 0 on success or negative error if something goes wrong


