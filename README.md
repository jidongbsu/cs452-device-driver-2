# Overview

In this assignment, we will write a Linux kernel module called lincoln. This module will serve as a keyboard device driver. You should still use the cs452 VM (username:cs452, password: cs452) which you used for your tesla, lexus, infiniti, and toyota, as loading and unloading the kernel module requires the root privilege. 

## Learning Objectives

- Learning how to interact with an I/O device.
- Gaining a better understanding of interrupts.

## Important Notes

You MUST build against the kernel version (3.10.0-1160.el7.x86_64), which is the default version of the kernel installed on the cs452 VM.

## Book References

Operating Systems: Three Easy Pieces: [I/O Devices](https://pages.cs.wisc.edu/~remzi/OSTEP/file-devices.pdf).

This chapter explains what roles I/O devices play in a computer system, and how device drivers work in general. In particular, it describes how an operating system interacts with I/O devices, and how interrupts work and why interrupts can lower the CPU's overhead. The chapter also explains what an interrupt handler is - in this assignment, a part of your job is to implement an interrupt handler for a keyboard.

## Background

### The Linux Input Subsystem



# The Starter Code



# Specification

You are required to implement the following functions.

```c
static irqreturn_t lincoln_irq_handler(struct serio *serio, unsigned char data, unsigned int flags);
```

this is the interrupt handler. Every time the keyboard raises an interrupt, this function will get called. There are two situations when a keyboard raises an interrupts:

1. User input. This is the most obvious reason. As the computer user, you type something from the keyboard, the keyboard needs to send a code (known as a scan code) corresponding to the key (you just typed) to the upper layer of the system, and eventually the application will get receive that key. Here, the second parameter of the interrupt handler, which *data*, stores the scan code.

2. Sometimes the user does not input anything, but the keyboard may still want to tell the CPU that something is happening. In this case, the keyboard produces a scan code, which is known as a protocol scan code - in contrast, a scan code produced in the above situation is called an ordinary scan code.

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
