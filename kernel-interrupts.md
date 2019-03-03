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
