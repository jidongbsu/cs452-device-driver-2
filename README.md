# Overview

In this assignment, we will write a Linux kernel module called lincoln. This module will serve as a keyboard device driver. You should still use the cs452 VM (username:cs452, password: cs452) which you used for your tesla, lexus, infiniti, and toyota, as loading and unloading the kernel module requires the root privilege. 

## Learning Objectives

- Learning how to interact with an I/O device.
- Gaining a better understanding of interrupts.

## Important Notes

You MUST build against the kernel version (3.10.0-1160.el7.x86_64), which is the default version of the kernel installed on the cs452 VM.

In this README, you will see the term **host**. It is used to refer to the computer to which the keyboard is connected.

## Book References

Operating Systems: Three Easy Pieces: [I/O Devices](https://pages.cs.wisc.edu/~remzi/OSTEP/file-devices.pdf).

This chapter explains what roles I/O devices play in a computer system, and how device drivers work in general. In particular, it describes how an operating system interacts with I/O devices, and how interrupts work and why interrupts can lower the CPU's overhead. The chapter also explains what an interrupt handler is - in this assignment, a part of your job is to implement an interrupt handler for a PS/2 keyboard, which is the default keyboard used in the provided virtual machine.

## Background

### The Linux Input Subsystem

According to chapter 7 (Input Drivers) of the book "Essential Linux Device Drivers", written by Sreekrishnan Venkateswaran, the Linux kernel has an input subsystem which "was created to unify scattered drivers that handle diverse classes of data-input devices such as keyboards, mice, trackballs, joysticks, roller wheels, touch screens, accelerometers, and tablets." The input subsystem looks like this:

![alt text](linux-input.jpg "The Linux Input Subsystem")

The chapter says the above figure "illustrates the operation of the input subsystem. The subsystem contains two classes of drivers that work in tandem: event drivers and device drivers. Event drivers are responsible for interfacing with applications, whereas device drivers are responsible for low-level communication with input devices. The mouse event generator, mousedev, is an example of the former, and the PS/2 mouse driver is an example of the latter. Both event drivers and device drivers can avail the services of an efficient, bug-free, reusable core, which lies at the heart of the input subsystem."

You are recommended to read the chapter, but to briefly explain this figure and relate it to this assignment: the part we are working on is a keyboard device driver, which belongs to the **input device drivers** class, and we will interact with the keyboard event driver, which is already existing in the Linux kernel and belongs to the **input event drivers** class. An input device driver does not directly interact with applications, rather it interacts with an input event driver, and that input event driver interacts with applications. Therefore, when a key press/release event occurs, we, as the keyboard device driver, capture this event and report this event to the keyboard event driver. See the [Input Event APIs](#input_event_apis) section for the APIs we can use to report events.

### The Intel 8042 Controller

The provided virtual machine has an (emulated) Intel 8042 controller, which allows a PS/2 keyboard and a PS/2 mouse to connect to the machine.

**side note**: it is my own understanding that most laptops contain an Intel 8042 controller and the keyboard comes with the laptop is considered as a PS/2 keyboard. Correct me if this is not the case on your laptop. However, no matter you are using a PS/2 keyboard or not, it should not affect you complete this assignment, because the provided virtual machine would always treat your keyboard as a PS/2 keyboard. Even if your real keyboard is not a PS/2 keyboard, the virtual machine will make it act like a PS/2 keyboard.

The following picture, downloaded from the [osdev.org](https://wiki.osdev.org/%228042%22_PS/2_Controller) website, shows the structure of an Intel 8042 controller.

![alt text](8042.png "Intel 8042 Controller")

As we can see from the picture, the PS/2 keyboard has two I/O ports, whose addresses are 0x60 and 0x64. You can interpret these two ports as two 8-bit registers. 0x60 represents the data register, a register which allows us to transfer data to and from the keyboard. 0x64 represents the status register, which also has 8 bits. A PS/2 keyboard typically has two buffers, input buffer and output buffer. And certain bits of the status register tells us the status of these two buffers, more specifically:

1. bit 0 of this status register indicates the output buffer status (0=empty, 1=full), when this bit is 1, it means there is something for us to read from the data register.
2. bit 1 of this status register indicates the input buffer status (0=empty, 1=full), when this bit is 0, it means the input buffer has space, and writing data to the data register is therefore allowed.

As the chapter describes: "By reading and writing these registers, the operating system can control device behavior".

# The Starter Code

The starter code looks like this:

```console
[cs452@xyno cs452-device-driver-2]$ ls
8042.png  lincoln.c  linux-input.jpg  Makefile  README.md  README.template
```

You will be completing the lincoln.c file. You should not modify the lincoln.h file.

The starter code already provides you with the code for a kernel module called lincoln. To install the module, run *make* and then *sudo insmod lincoln.ko*; to remove it, run *sudo rmmod lincoln*. Yes, in rmmod, whether or not you specify ko does not matter; but in insmod, you must have that ko.

## The Proc Interface

The starter code creates a proc interface for users to communicate with the keyboard. More specifically, when the module is loaded, a file called */proc/lincoln/cmd* will be created. Users can write commands into this file, and the commands are expected to pass to the keyboard, via a function you are going to implement - *lincoln_kbd_write*().

## Intercepting the Default Keyboard Interrupt Handler

The starter code also hijacks the default keyboard driver, so that when an interrupt occurs, the default interrupt handler will not be called, rather, it is an interrupt handler that you are going to implement - *lincoln_irq_handler*(), which will be called.

# Specification

You are required to implement the following functions.

```c
static int lincoln_kbd_write(struct serio *port, unsigned char c);
```

this function sends a single byte to the keyboard. A PS/2 keyboard supports 17 host-to-keyboard commands. For example, command *0xf5* means disabling the keyboard, and command *0xf4* means enabling the keyboard, and command 0xff means resetting the keyboard. In this assignment, we want to allow the user to disable, enable and reset the keyboard. More specifically, when the user runs this:

```console
# sudo echo 'D' > /proc/lincoln/cmd
```

we need to disable the keyboard. In other words, we want to send the *0xf5* command to the keyboard - write this 0xf5 to the data port. The expected effect of this command is, the keyboard will become unresponsive.

And then when the user runs this:

```console
# sudo echo 'E' > /proc/lincoln/cmd
```

we need to enable the keyboard. In other words, we want to send the *0xf4* command to the keyboard - write this 0xf4 to the data port. The expected effect of this command is, the keyboard will become responsive again.

**Question**: when the keyboard is not responsive at all, how can the user even enter the second command - which enables the keyboard?

And then when the user runs this:

```console
# sudo echo 'R' > /proc/lincoln/cmd
```

we want to keyboard to reset. In other words, we want to send the *0xff* command to the keyboard - write this 0xf4 to the data port. The expected effect of this command is, the keyboard will perform a BAT test and when the BAT test is complete, the keyboard will send either 0xAA (BAT successful) or 0xFC (Error) to the host. Keep reading this README and you will soon find out what BAT is.

**Note**: the starter code is implemented in such a way that when the user run the above *sudo echo* commands, your *lincoln_kbd_write*() will get called, and the command is passed as the second parameter of your function, i.e., *unsigned char c*.

```c
static irqreturn_t lincoln_irq_handler(struct serio *serio, unsigned char data, unsigned int flags);
```

this is the interrupt handler. Every time the keyboard raises an interrupt, this function will get called. There are two **situations** when a keyboard raises an interrupts:

**Situation** 1. User input. This is the most obvious reason. As the computer user, you type something from the keyboard, the keyboard needs to send a code (known as a scan code) corresponding to the key (you just pressed or released) to the upper layer of the system, and eventually the application will receive that key. Here, the second parameter of the interrupt handler, which is *data*, stores the scan code.

**Situation** 2. Sometimes the user does not input anything, but the keyboard may still want to tell the CPU that something is happening. In this case, the keyboard also produces a scan code, which is known as a protocol scan code - in contrast, a scan code produced in the above situation is called an ordinary scan code. Still, the second parameter of the interrupt handler, which is *data*, stores the scan code. Some examples of the protocol scan code include:

  - 0xaa. This is called the BAT successful code. When you boot your computer, the keyboard performs a diagnostic self-test referred to as BAT (Basic Assurance Test) and configure the keyboard to its default values. When entering BAT, the keyboard enables its three LED indicators, and turns them off when BAT has completed. At this time, a BAT completion code of either 0xAA (BAT successful) or 0xFC (Error) is sent to the host. Besides power-on, a software reset would also trigger the keyboard to perform BAT.

  - 0xfa. This is called the acknowledge code, or "ack" code. When the host sends a command to the keyboard, the keyboard may respond with an **ack** code, indicating the command is received by the keyboard.

A typical keyboard also defines other protocol scan codes. In this assignment, your interrupt handler only needs to handle these two protocol scan codes, as well as all ordinary scan codes.

As described above, our code is a part of a keyboard device driver. When a keyboard raises an interrupt, and if it is because of **situation** 1, i.e., user input, then our interrupt handler should pass this event to the keyboard event driver; when a keyboard raises an interrupt, but if it is because of **situation** 2, i.e., not an user input, our interrupt handler should react to it, but should not pass this event to the keyboard event driver. This is because the keyboard event driver's job is to collect events from the keyboard device driver and notify applications, when there is no user input, applications should not be notified by the keyboard event driver.

**Special Requirement**: Your interrupt handler must achieve this: when the user types every key other than *l* or *s*, the user should observe normal behaviors; but when the user types *l*, it should be interpreted as *s*, and displayed as *s*; when the user types *s*, it should be interpreted as *l*, and displayed as *l*.

## Accessing the Status and Data Registers

The book chapter describes:"on x86, the *in* and *out* instructions can be used to communicate with devices". Indeed, in this assignment, we will use the *in* instruction to inquire the status of our device - which in this assignment, means the keyboard, and we will use the *out* instruction to send our command to the device.

Your *lincoln_kbd_write*() therefore should have the following logic:

```c
while(STATUS == BUSY)
;
Write data to DATA register
```

In this assignment, we do not intend to use the *in* and *out* assembly instructions directly, rather, we use C functions provided by the Linux kernel:

1. *inb*(): this function encapsulates the *in* assembly instruction and it reads one byte from one specific (8-bit) register. For example, if we want to read one byte from a register whose address is at 0x64, then we call:

```c
inb(0x64)
```

2. *outb*(): this function encapsulates the *out* assembly instruction and it write one byte to one specific (8-bit) register. For example, if we want to write one byte *c* to a register whose address is at 0x60, then we call:

```c
outb(c, 0x60)
```

## Input Event APIs

In the interrupt handler function, to report events to the input event drivers layer, you can call *input_event*() and *input_sync*() like this:

```c
struct atkbd *atkbd = serio_get_drvdata(serio);	// here serio is the first parameter of the interrupt handler function.
struct input_dev *dev = atkbd->dev;
input_event(dev, EV_KEY, keycode, value); // this function generates the event
input_sync(dev); // this function indicates that the input subsystem can collect previously generated events into an evdev packet and send it to user space via /dev/input/inputX
```

Here *keycode* refers to the code that is corresponding to the key that is pressed or released, and *value* refers to the action of press or release. If the action is a key press, then value must be 1, if the action is a key release, then value must be 0. We can derive both the keycode and the value from the second parameter (i.e., *data*) of the interrupt handler function. The parameter *data*, as an unsigned char data type, has 8 bits, and

1. bit 7 represents the action: 1==release, 0==press.
2. bit 6 to bit 0 represents the keycode.

More explanation of EV_KEY. The second parameter of *input_event*() tells the input subsystem what event type is generated. The Linux input subsystem defines several event types: the type EV_KEY is used to describe state changes of keyboards, buttons, or other key-like devices; the type EV_REL is used to describe relative axis value changes, e.g. moving the mouse 5 units to the left; the type EV_ABS is used to describe absolute axis value changes, e.g. describing the coordinates of a touch on a touchscreen; the type EV_LED is used to turn LEDs on devices on and off; the type EV_SND is used to output sound to devices. If you want to know more about these events, see the [documentation](https://www.kernel.org/doc/Documentation/input/event-codes.txt) comes with the Linux kernel source code.

## Testing

### Part 1.

When the module is loaded, the user, in the starter code directory, types *sl* from a local console inside the virtual machine, should see *ls* and the result of the *ls* command:

```console
[cs452@xyno cs452-device-driver-2]$ ls
8042.png  lincoln.c  Makefile  README.md  README.template
```

**Note**: here we see *ls*, but I actually typed *sl*.

when the user actually types *ls*, the user should see *sl*, and get the following:

```console
[cs452@xyno cs452-device-driver-2]$ sl
bash: sl: command not found...
Similar command is: 'ls'
```

Other keys should work as expected:

```console
[cs452@xyno cs452-device-driver-2]$ pwd
/home/cs452/cs452-device-driver-2
[cs452@xyno cs452-device-driver-2]$ who
cs452    pts/0        2022-09-15 10:25 (192.168.56.1)
[cs452@xyno cs452-device-driver-2]$ whoami
cs452
```

**Note**: we do not need to test special keys, such as ESC, Left SHIFT, Right SHIFT etc.

### Part 2.

When the user runs:

```console
# sudo echo 'D' > /proc/lincoln/cmd
```

The keyboard becomes unresponsive.

When the user then runs:

```console
# sudo echo 'E' > /proc/lincoln/cmd
```

The keyboard becomes responsive again.

When the user runs:

```console
# sudo echo 'R' > /proc/lincoln/cmd
```

We must see this message printed in the kernel log:

```console
we get an ACK from the keyboard.
```

And this message must be printed by your interrupt handler function.

## Submission

Due: 23:59pm, November 20th, 2022. Late submission will not be accepted/graded.

## Project Layout

All files necessary for compilation and testing need to be submitted, this includes source code files, header files, and Makefile. The structure of the submission folder should be the same as what was given to you.

## Grading Rubric (Undergraduate and Graduate)

Grade: /100

- [ 80 pts] Functional Requirements:
  - *ls* shows (and runs) as *sl*, and *sl* shows (and runs) as *ls*. /20
  - Commands *pwd*, *who*, and *whoami* work as expected. /20
  - Keyboard disable and then enable works. /20
  - Keyboard reset works and we get that ack message in the kernel log. /10
  - Module can be installed and removed without crashing the system: /10
    - You won't get these points if your module doesn't implement any of the above functional requirements.

- [10 pts] Compiler warnings:
  - Each compiler warning will result in a 3 point deduction.
  - You are not allowed to suppress warnings.
  - You won't get these points if you didn't implement any of the above functional requirements.

- [10 pts] Documentation:
  - README.md file (rename this current README file to README.orig and rename the README.template to README.md.)
  - You are required to fill in every section of the README template, missing 1 section will result in a 2-point deduction.
