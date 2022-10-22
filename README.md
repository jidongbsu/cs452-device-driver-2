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

### The Intel 8042 Controller

The provided virtual machine has an Intel 8042 controller, which allows a PS/2 keyboard and a PS/2 mouse to connect to the machine.

**side note**: it is my own understanding that most laptops contain an Intel 8042 controller and the keyboard comes with the laptop is considered as a PS2 keyboard. Correct me if this is not the case on your laptop. However, no matter you are using a PS/2 keyboard or not, it should not affect you complete this assignment, because the provided virtual machine would always treat your keyboard as a PS/2 keyboard. Even if your real keyboard is not a PS/2 keyboard, the virtual machine will make it act like a PS/2 keyboard.

The following picture, downloaded from the [osdev.org](https://wiki.osdev.org/%228042%22_PS/2_Controller) website, shows the structure of an Intel 8042 controller.

![alt text](8042.png "Intel 8042 Controller")

As we can see from the picture, the PS/2 keyboard has two I/O ports, whose addresses are 0x60 and 0x64. 0x60 is called the data port, a port that allows us to transfer data to and from the keyboard. 0x64 is called the status port, or status register, which has 8 bits. In particular:

1. bit 0 of this status register indicates the output buffer status (0=empty, 1=full), when this bit is 1, it means there is something for us to read from the data port.
2. bit 1 of this status register indicates the input buffer status (0=empty, 1=full), when this bit is 0, it means the input buffer has space, and writing data to the data port is therefore allowed.

# The Starter Code

The starter code looks like this:

You will be completing the lincoln.c file. You should not modify the lincoln.h file.

The starter code already provides you with the code for a kernel module called lincoln. To install the module, run *make* and then *sudo insmod lincoln.ko*; to remove it, run *sudo rmmod lincoln*. Yes, in rmmod, whether or not you specify ko does not matter; but in insmod, you must have that ko.

## The Proc Interface

The starter code creates a proc interface for users to communicate with the keyboard. More specifically, when the module is loaded, a file called */proc/lincoln/cmd* will be created. Users can write commands into this file, and the commands are expected to pass to the keyboard, via a function you are going to implement - *lincoln_kbd_write*().

## Intercepting the Default Keyboard Interrupt Handler

The starter code also hijacks the default keyboard driver, so that when an interrupt occurs, the default interrupt handler will not be called, rather, it is an interrupt handler that you are going to implement - *lincoln_irq_handler(*(), which will be called.

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

this is the interrupt handler. Every time the keyboard raises an interrupt, this function will get called. There are two situations when a keyboard raises an interrupts:

1. User input. This is the most obvious reason. As the computer user, you type something from the keyboard, the keyboard needs to send a code (known as a scan code) corresponding to the key (you just pressed or released) to the upper layer of the system, and eventually the application will receive that key. Here, the second parameter of the interrupt handler, which is *data*, stores the scan code.

2. Sometimes the user does not input anything, but the keyboard may still want to tell the CPU that something is happening. In this case, the keyboard also produces a scan code, which is known as a protocol scan code - in contrast, a scan code produced in the above situation is called an ordinary scan code. Still, the second parameter of the interrupt handler, which is *data*, stores the scan code. Some examples of the protocol scan code include:

1. 0xaa. This is called the BAT successful code. When you boot your computer, the keyboard performs a diagnostic self-test referred to as BAT (Basic Assurance Test) and configure the keyboard to its default values. When entering BAT, the keyboard enables its three LED indicators, and turns them off when BAT has completed. At this time, a BAT completion code of either 0xAA (BAT successful) or 0xFC (Error) is sent to the host. Besides power-on, a software reset would also trigger the keyboard to perform BAT.

2. 0xfa. This is called the acknowledge code, or "ack" code. When the host sends a command to the keyboard, the keyboard may respond with an **ack** code, indicating the command is received by the keyboard.

A typical keyboard also defines other protocol scan codes. In this assignment, your interrupt handler only need to handle these two protocol scan codes, as well as all ordinary scan codes.

## Testing

## Submission

Due: 23:59pm, November 20th, 2022. Late submission will not be accepted/graded.

## Project Layout

All files necessary for compilation and testing need to be submitted, this includes source code files, header files, and Makefile. The structure of the submission folder should be the same as what was given to you.

## Grading Rubric (Undergraduate and Graduate)

Grade: /100

- [ 80 pts] Functional Requirements:
  - to be added here.

- [10 pts] Compiler warnings:
  - Each compiler warning will result in a 3 point deduction.
  - You are not allowed to suppress warnings.
  - You won't get these points if you didn't implement any of the above functional requirements.

- [10 pts] Documentation:
  - README.md file (rename this current README file to README.orig and rename the README.template to README.md.)
  - You are required to fill in every section of the README template, missing 1 section will result in a 2-point deduction.
