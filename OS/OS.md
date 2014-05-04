# MorosOS: *"We probably sometimes know what we're doing"* #

*This is the beginnings of a kernel specification for MorosOS.  It contains
what's known about design goals, implementation details and open questions about
the system.*

## Objective ##
In our system, MorosOS is first and foremost responsible for the
following:
  * Secure management of privileges and policy
  * Virtualizing hardware resources
  * Scheduling/providing a uniform execution environment

While a lot of elements are still open for discussion, as few things are
clear: we will have a microkernel that uses an IPC model to communicate
between userland services, as well as a set of core components modeled
loosely after the 410 Pebbles kernel, and a standard library to aid
implementation.

## Microkernel Design ##
MorosOS is a uniprocessor operating system designed in the spirit of a
[microkernel](http://en.wikipedia.org/wiki/Microkernel): a very simple
abstraction over hardware that keeps as many
services as possible in userland.  As this guy on Wikipedia puts it:

> "A concept is tolerated inside the microkernel only if moving it outside
> the kernel, i.e., permitting competing implementations, would prevent
> the implementation of the systems required functionality"

Examples of userland programs include device drivers and system storage
(though we may end up cheating a bit and placing some drivers in the
kernel for boot).    

![](http://upload.wikimedia.org/wikipedia/commons/thumb/6/67/OS-structure.svg/450px-OS-structure.svg.png
"Image from Wikipedia")

Like other microkernels, we use IPC to implement cross-address space
communication.  The kernel is divided into a handful of individual
processes we call "servers" (see image above), which use these messages to
request services from one another.  If possible, we want to steal some
performance tricks from the [L4 IPC
approach](http://www.read.seas.harvard.edu/~kohler/class/aosref/liedtke93improving.pdf):
completely synchronous
messages, explicit send/receive syscalls, and a selective "half context
switch" to pass messages in registers.  

Historically, Microkernels have had simple, solid security policies that
make ensuring protections easy.  At this point, however, our security
model is very up in the air.

## I/O and Userland ##
Since we're placing a lot of key services there, the MorosOS userland
is going to be big, and that's going to be our problem.  

Programs are able to request access to I/O and device memory via syscalls, and
then manipulate these resources via the appropriate servers: for
instance, a thread that wants to write to a frame buffer might undergo
the following:
- Thread 1 requests a frame buffer using the get_frame_buffer syscall, passing
  address 0xADD000 as its argument
- Kernel responds to the syscall by allocating a buffer with ID 4 to
Thread 1, mapping the buffer to 0xADD000 and returning the ID
- get_frame_buffer returns to Thread 1 with buffer ID 4
- Thread 1 sends a use_frame_buffer request to the Console Server with
buffer ID 4
- Console Server receives request from Thread 1 with buffer ID 4
- Console Server decides whether the request should succeed based on
console privileges and allocations.
    - If the request should succeed, Console Server calls the
    use_frame_buffer syscall with TID 1 and buffer ID 4
        - kernel verifies that Thread 1 has access to buffer 4, and completes
        the write to the console
    - if the request should fail, the Console Server ignores it

The same protocol can be used for other I/O resources, such as sound.  

Additionally, the kernel and hardware will have no notion of text, so we will have
instead a text-mode server that manages text-based display services. 

Expect VGA graphics.  

We expect to have a sound driver, keyboard driver and a timer driver.  A
disk driver, mouse driver and network driver, however, are also in the
realm of possibility.

We also need to flesh out how these userspace drivers are going to
handle interrupts.

We are also going to have a thread library that will be similar in
objective to 410 P2.  Other than the strange conventions, this should be
relatively simple to implement.

## Nuts and Bolts: What Pebbles does, and what we do differently ##

It's only natural that we derive a lot of inspiration from [the Pebbles
kernel](http://www.cs.cmu.edu/~410/p2/kspec.pdf): really what this means is,
we've decided to do some things in a
conventional manner, and Pebbles provides a good shared language to talk
about such things as a team.

First and foremost, our kernel will have the following Pebbles syscalls: 
* fork/exec 
* thread\_fork
* wait/vanish
* get_ticks
* sleep
* new\_pages/remove\_pages
* gettid
* yield.  

Others, such as set\_status, might be implemented depending on necessity.
Others still, such as print or readline/getchar, might have the same
API, but may exist in a very different way than their Pebbles
implementation: for instance, I have a hunch a lot more of the legwork
is going to get done in userland (the same almost certainly goes for
console manipulation).  Furthermore, an IPC model may allow us to do
something a little more clever than deschedule/make\_runnable.     

Pebbles uses an arguably hackish memory layout that involves direct
mapping n pages of memory and calling it a kernel; we will probably want
to look this over and decide if MorosOS would benefit from another
approach.
Since we have a software-loaded TLB, paging is now going to be the
responsibility of the OS.  Essentially, what this looks like is if a
thread tries to access a page that isn't loaded into the TLB, it
generates a fault, which the kernel handles accordingly.  

MorosOS will implement COW for fork.  It also wouldn't hurt to consider
something other than 410's "argument packet" model for passing arguments
to syscalls (like, I dunno, specifying a convention for using multiple
registers)   

## Kernel Libraries ##
A proposed breakdown of library work was the following: all functions
that touch hardware are the responsibility of the language team,
anything purely software is the OS team.  Assuming this is how things
play out, we will have to think long and hard about what library
functions we expect to have when we implement the rest of the OS.  I
suspect this will be an arduous back-and-forth of implementing the
library we expect to have underneath us while we write the kernel.
Pieces we think of that we are almost certainly going to want as we
begin development should go here.    

  Some components that immediately come to mind are the memory
allocators.  This will be tricky because we need to ensure that pages
being used for kernel memory do not get mapped out to processes.
Debugging facilities will also want asserts, and maybe even some
form of outputting text to a debug console.    

## Conversations to have with other teams ##
As the OS acts as the glue that holds the other parts of the system
together, many of our decisions are going to involve a discussion with
the Hardware Team and the Language Team, either to decide on protocol,
or simply to learn what the protocol is going to be.  Before any real
development can start, the following will need to be discussed:

### Hardware Team 
  * PIC, IDT: really, we just need to know where they are.  The same
  probably goes for other hardware components as well 
  * Boot Process: If we handle this more intuitively than 410 has to, the
  process will be much less toilsome
  * Exceptions: where they are coming from, what we should expect to see.

### Language/Compiler Team
  * Calling conventions: deciding on these is important for when we need
  to dig into asm
  * Stack: we need to make sure they know the stack is going to grow up.
  It's important
 
Finally, it is important that all three teams discuss a security model,
and what guarantees and assumptions each component is going to have.

