.. _threads_v2:

Threads
^^^^^^^

This section describes kernel services for creating, scheduling, and deleting
independently executable threads of instructions.

.. contents::
    :local:
    :depth: 1

.. _lifecycle_v2:

Lifecycle
#########

A :dfn:`thread` is a kernel object that is used for application processing
that is too lengthy or too complex to be performed by an ISR.

Concepts
********

Any number of threads can be defined by an application. Each thread is
referenced by a :dfn:`thread id` that is assigned when the thread is spawned.

A thread has the following key properties:

* A **stack area**, which is a region of memory used for the thread's stack.
  The **size** of the stack area can be tailored to conform to the actual needs
  of the thread's processing. Special macros exist to create and work with
  stack memory regions.

* A **thread control block** for private kernel bookkeeping of the thread's
  metadata. This is an instance of type :c:type:`struct k_thread`.

* An **entry point function**, which is invoked when the thread is started.
  Up to 3 **argument values** can be passed to this function.

* A **scheduling priority**, which instructs the kernel's scheduler how to
  allocate CPU time to the thread. (See :ref:`scheduling_v2`.)

* A set of **thread options**, which allow the thread to receive special
  treatment by the kernel under specific circumstances.
  (See :ref:`thread_options_v2`.)

* A **start delay**, which specifies how long the kernel should wait before
  starting the thread.

* An **execution mode**, which can either be supervisor or user mode.
  By default, threads run in supervisor mode and allow access to
  privileged CPU instructions, the entire memory address space, and
  peripherals. User mode threads have a reduced set of privileges.
  This depends on the :option:`CONFIG_USERSPACE` option. See :ref:`usermode`.

.. _spawning_thread:

Thread Creation
===============

A thread must be created before it can be used. The kernel initializes
the thread control block as well as one end of the stack portion. The remainder
of the thread's stack is typically left uninitialized.

Specifying a start delay of :c:macro:`K_NO_WAIT` instructs the kernel
to start thread execution immediately. Alternatively, the kernel can be
instructed to delay execution of the thread by specifying a timeout
value -- for example, to allow device hardware used by the thread
to become available.

The kernel allows a delayed start to be canceled before the thread begins
executing. A cancellation request has no effect if the thread has already
started. A thread whose delayed start was successfully canceled must be
re-spawned before it can be used.

Thread Termination
==================

Once a thread is started it typically executes forever. However, a thread may
synchronously end its execution by returning from its entry point function.
This is known as **termination**.

A thread that terminates is responsible for releasing any shared resources
it may own (such as mutexes and dynamically allocated memory)
prior to returning, since the kernel does *not* reclaim them automatically.

.. note::
    The kernel does not currently make any claims regarding an application's
    ability to respawn a thread that terminates.

Thread Aborting
===============

A thread may asynchronously end its execution by **aborting**. The kernel
automatically aborts a thread if the thread triggers a fatal error condition,
such as dereferencing a null pointer.

A thread can also be aborted by another thread (or by itself)
by calling :cpp:func:`k_thread_abort()`. However, it is typically preferable
to signal a thread to terminate itself gracefully, rather than aborting it.

As with thread termination, the kernel does not reclaim shared resources
owned by an aborted thread.

.. note::
    The kernel does not currently make any claims regarding an application's
    ability to respawn a thread that aborts.

Thread Suspension
=================

A thread can be prevented from executing for an indefinite period of time
if it becomes **suspended**. The function :cpp:func:`k_thread_suspend()`
can be used to suspend any thread, including the calling thread.
Suspending a thread that is already suspended has no additional effect.

Once suspended, a thread cannot be scheduled until another thread calls
:cpp:func:`k_thread_resume()` to remove the suspension.

.. note::
   A thread can prevent itself from executing for a specified period of time
   using :cpp:func:`k_sleep()`. However, this is different from suspending
   a thread since a sleeping thread becomes executable automatically when the
   time limit is reached.

.. _thread_options_v2:

Thread Options
==============

The kernel supports a small set of :dfn:`thread options` that allow a thread
to receive special treatment under specific circumstances. The set of options
associated with a thread are specified when the thread is spawned.

A thread that does not require any thread option has an option value of zero.
A thread that requires a thread option specifies it by name, using the
:literal:`|` character as a separator if multiple options are needed
(i.e. combine options using the bitwise OR operator).

The following thread options are supported.

:c:macro:`K_ESSENTIAL`
    This option tags the thread as an :dfn:`essential thread`. This instructs
    the kernel to treat the termination or aborting of the thread as a fatal
    system error.

    By default, the thread is not considered to be an essential thread.

:c:macro:`K_FP_REGS` and :c:macro:`K_SSE_REGS`
    These x86-specific options indicate that the thread uses the CPU's
    floating point registers and SSE registers, respectively. This instructs
    the kernel to take additional steps to save and restore the contents
    of these registers when scheduling the thread.
    (For more information see :ref:`float_v2`.)

    By default, the kernel does not attempt to save and restore the contents
    of these registers when scheduling the thread.

:c:macro:`K_USER`
    If :option:`CONFIG_USERSPACE` is enabled, this thread will be created in
    user mode and will have reduced privileges. See :ref:`usermode`. Otherwise
    this flag does nothing.

:c:macro:`K_INHERIT_PERMS`
    If :option:`CONFIG_USERSPACE` is enabled, this thread will inherit all
    kernel object permissions that the parent thread had, except the parent
    thread object.  See :ref:`usermode`.

Implementation
**************

Spawning a Thread
=================

A thread is spawned by defining its stack area and its thread control block,
and then calling :cpp:func:`k_thread_create()`. The stack area must be defined
using :c:macro:`K_THREAD_STACK_DEFINE` to ensure it is properly set up in
memory.

The thread spawning function returns its thread id, which can be used
to reference the thread.

The following code spawns a thread that starts immediately.

.. code-block:: c

    #define MY_STACK_SIZE 500
    #define MY_PRIORITY 5

    extern void my_entry_point(void *, void *, void *);

    K_THREAD_STACK_DEFINE(my_stack_area, MY_STACK_SIZE);
    struct k_thread my_thread_data;

    k_tid_t my_tid = k_thread_create(&my_thread_data, my_stack_area,
                                     K_THREAD_STACK_SIZEOF(my_stack_area),
                                     my_entry_point,
                                     NULL, NULL, NULL,
                                     MY_PRIORITY, 0, K_NO_WAIT);

Alternatively, a thread can be spawned at compile time by calling
:c:macro:`K_THREAD_DEFINE`. Observe that the macro defines
the stack area, control block, and thread id variables automatically.

The following code has the same effect as the code segment above.

.. code-block:: c

    #define MY_STACK_SIZE 500
    #define MY_PRIORITY 5

    extern void my_entry_point(void *, void *, void *);

    K_THREAD_DEFINE(my_tid, MY_STACK_SIZE,
                    my_entry_point, NULL, NULL, NULL,
                    MY_PRIORITY, 0, K_NO_WAIT);

User Mode Constraints
---------------------

This section only applies if :option:`CONFIG_USERSPACE` is enabled, and a user
thread tries to create a new thread. The :c:func:`k_thread_create()` API is
still used, but there are additional constraints which must be met or the
calling thread will be terminated:

* The calling thread must have permissions granted on both the child thread
  and stack parameters; both are tracked by the kernel as kernel objects.

* The child thread and stack objects must be in an uninitialized state,
  i.e. it is not currently running and the stack memory is unused.

* The stack size parameter passed in must be equal to or less than the
  bounds of the stack object when it was declared.

* The :c:macro:`K_USER` option must be used, as user threads can only create
  other user threads.

* The :c:macro:`K_ESSENTIAL` option must not be used, user threads may not be
  considered essential threads.

* The priority of the child thread must be a valid priority value, and equal to
  or lower than the parent thread.

Dropping Permissions
====================

If :option:`CONFIG_USERSPACE` is enabled, a thread running in supervisor mode
may perform a one-way transition to user mode using the
:cpp:func:`k_thread_user_mode_enter()` API. This is a one-way operation which
will reset and zero the thread's stack memory. The thread will be marked
as non-essential.

Terminating a Thread
====================

A thread terminates itself by returning from its entry point function.

The following code illustrates the ways a thread can terminate.

.. code-block:: c

    void my_entry_point(int unused1, int unused2, int unused3)
    {
        while (1) {
            ...
	    if (<some condition>) {
	        return; /* thread terminates from mid-entry point function */
	    }
	    ...
        }

        /* thread terminates at end of entry point function */
    }

If CONFIG_USERSPACE is enabled, aborting a thread will additionally mark the
thread and stack objects as uninitialized so that they may be re-used.

Suggested Uses
**************

Use threads to handle processing that cannot be handled in an ISR.

Use separate threads to handle logically distinct processing operations
that can execute in parallel.

.. _custom_data_v2:

Custom Data
###########

A thread's :dfn:`custom data` is a 32-bit, thread-specific value that may be
used by an application for any purpose.

Concepts
********

Every thread has a 32-bit custom data area.
The custom data is accessible only by the thread itself, and may be used by the
application for any purpose it chooses.
The default custom data for a thread is zero.

.. note::
   Custom data support is not available to ISRs because they operate
   within a single shared kernel interrupt handling context.

Implementation
**************

Using Custom Data
=================

By default, thread custom data support is disabled. The configuration option
:option:`CONFIG_THREAD_CUSTOM_DATA` can be used to enable support.

The :cpp:func:`k_thread_custom_data_set()` and
:cpp:func:`k_thread_custom_data_get()` functions are used to write and read
a thread's custom data, respectively. A thread can only access its own
custom data, and not that of another thread.

The following code uses the custom data feature to record the number of times
each thread calls a specific routine.

.. note::
    Obviously, only a single routine can use this technique,
    since it monopolizes the use of the custom data feature.

.. code-block:: c

    int call_tracking_routine(void)
    {
        u32_t call_count;

        if (k_is_in_isr()) {
	    /* ignore any call made by an ISR */
        } else {
            call_count = (u32_t)k_thread_custom_data_get();
            call_count++;
            k_thread_custom_data_set((void *)call_count);
	}

        /* do rest of routine's processing */
        ...
    }

Suggested Uses
**************

Use thread custom data to allow a routine to access thread-specific information,
by using the custom data as a pointer to a data structure owned by the thread.

.. _system_threads_v2:

System Threads
##############

A :dfn:`system thread` is a thread that the kernel spawns automatically
during system initialization.

Concepts
********

The kernel spawns the following system threads.

**Main thread**
    This thread performs kernel initialization, then calls the application's
    :cpp:func:`main()` function (if one is defined).

    By default, the main thread uses the highest configured preemptible thread
    priority (i.e. 0). If the kernel is not configured to support preemptible
    threads, the main thread uses the lowest configured cooperative thread
    priority (i.e. -1).

    The main thread is an essential thread while it is performing kernel
    initialization or executing the application's :cpp:func:`main()` function;
    this means a fatal system error is raised if the thread aborts. If
    :cpp:func:`main()` is not defined, or if it executes and then does a normal
    return, the main thread terminates normally and no error is raised.

**Idle thread**
    This thread executes when there is no other work for the system to do.
    If possible, the idle thread activates the board's power management support
    to save power; otherwise, the idle thread simply performs a "do nothing"
    loop. The idle thread remains in existence as long as the system is running
    and never terminates.

    The idle thread always uses the lowest configured thread priority.
    If this makes it a cooperative thread, the idle thread repeatedly
    yields the CPU to allow the application's other threads to run when
    they need to.

    The idle thread is an essential thread, which means a fatal system error
    is raised if the thread aborts.

Additional system threads may also be spawned, depending on the kernel
and board configuration options specified by the application. For example,
enabling the system workqueue spawns a system thread
that services the work items submitted to it. (See :ref:`workqueues_v2`.)

Implementation
**************

Writing a main() function
=========================

An application-supplied :cpp:func:`main()` function begins executing once
kernel initialization is complete. The kernel does not pass any arguments
to the function.

The following code outlines a trivial :cpp:func:`main()` function.
The function used by a real application can be as complex as needed.

.. code-block:: c

    void main(void)
    {
        /* initialize a semaphore */
	...

	/* register an ISR that gives the semaphore */
	...

	/* monitor the semaphore forever */
	while (1) {
	    /* wait for the semaphore to be given by the ISR */
	    ...
	    /* do whatever processing is now needed */
	    ...
	}
    }

Suggested Uses
**************

Use the main thread to perform thread-based processing in an application
that only requires a single thread, rather than defining an additional
application-specific thread.

.. _workqueues_v2:

Workqueue Threads
#################

A :dfn:`workqueue` is a kernel object that uses a dedicated thread to process
work items in a first in, first out manner. Each work item is processed by
calling the function specified by the work item. A workqueue is typically
used by an ISR or a high-priority thread to offload non-urgent processing
to a lower-priority thread so it does not impact time-sensitive processing.

Concepts
********

Any number of workqueues can be defined. Each workqueue is referenced by its
memory address.

A workqueue has the following key properties:

* A **queue** of work items that have been added, but not yet processed.

* A **thread** that processes the work items in the queue. The priority of the
  thread is configurable, allowing it to be either cooperative or preemptive
  as required.

A workqueue must be initialized before it can be used. This sets its queue
to empty and spawns the workqueue's thread.

Work Item Lifecycle
===================

Any number of **work items** can be defined. Each work item is referenced
by its memory address.

A work item has the following key properties:

* A **handler function**, which is the function executed by the workqueue's
  thread when the work item is processed. This function accepts a single
  argument, which is the address of the work item itself.

* A **pending flag**, which is used by the kernel to signify that the
  work item is currently a member of a workqueue's queue.

* A **queue link**, which is used by the kernel to link a pending work
  item to the next pending work item in a workqueue's queue.

A work item must be initialized before it can be used. This records the work
item's handler function and marks it as not pending.

A work item may be **submitted** to a workqueue by an ISR or a thread.
Submitting a work item appends the work item to the workqueue's queue.
Once the workqueue's thread has processed all of the preceding work items
in its queue the thread will remove a pending work item from its queue and
invoke the work item's handler function. Depending on the scheduling priority
of the workqueue's thread, and the work required by other items in the queue,
a pending work item may be processed quickly or it may remain in the queue
for an extended period of time.

A handler function can utilize any kernel API available to threads. However,
operations that are potentially blocking (e.g. taking a semaphore) must be
used with care, since the workqueue cannot process subsequent work items in
its queue until the handler function finishes executing.

The single argument that is passed to a handler function can be ignored if
it is not required. If the handler function requires additional information
about the work it is to perform, the work item can be embedded in a larger
data structure. The handler function can then use the argument value to compute
the address of the enclosing data structure, and thereby obtain access to the
additional information it needs.

A work item is typically initialized once and then submitted to a specific
workqueue whenever work needs to be performed. If an ISR or a thread attempts
to submit a work item that is already pending, the work item is not affected;
the work item remains in its current place in the workqueue's queue, and
the work is only performed once.

A handler function is permitted to re-submit its work item argument
to the workqueue, since the work item is no longer pending at that time.
This allows the handler to execute work in stages, without unduly delaying
the processing of other work items in the workqueue's queue.

.. important::
    A pending work item *must not* be altered until the item has been processed
    by the workqueue thread. This means a work item must not be re-initialized
    while it is pending. Furthermore, any additional information the work item's
    handler function needs to perform its work must not be altered until
    the handler function has finished executing.

Delayed Work
============

An ISR or a thread may need to schedule a work item that is to be processed
only after a specified period of time, rather than immediately. This can be
done by submitting a **delayed work item** to a workqueue, rather than a
standard work item.

A delayed work item is a standard work item that has the following added
properties:

* A **delay** specifying the time interval to wait before the work item
  is actually submitted to a workqueue's queue.

* A **workqueue indicator** that identifies the workqueue the work item
  is to be submitted to.

A delayed work item is initialized and submitted to a workqueue in a similar
manner to a standard work item, although different kernel APIs are used.
When the submit request is made the kernel initiates a timeout mechanism
that is triggered after the specified delay has elapsed. Once the timeout
occurs the kernel submits the delayed work item to the specified workqueue,
where it remains pending until it is processed in the standard manner.

An ISR or a thread may **cancel** a delayed work item it has submitted,
providing the work item's timeout is still counting down. The work item's
timeout is aborted and the specified work is not performed.

Attempting to cancel a delayed work item once its timeout has expired has
no effect on the work item; the work item remains pending in the workqueue's
queue, unless the work item has already been removed and processed by the
workqueue's thread. Consequently, once a work item's timeout has expired
the work item is always processed by the workqueue and cannot be canceled.

System Workqueue
================

The kernel defines a workqueue known as the *system workqueue*, which is
available to any application or kernel code that requires workqueue support.
The system workqueue is optional, and only exists if the application makes
use of it.

.. important::
    Additional workqueues should only be defined when it is not possible
    to submit new work items to the system workqueue, since each new workqueue
    incurs a significant cost in memory footprint. A new workqueue can be
    justified if it is not possible for its work items to co-exist with
    existing system workqueue work items without an unacceptable impact;
    for example, if the new work items perform blocking operations that
    would delay other system workqueue processing to an unacceptable degree.

Implementation
**************

Defining a Workqueue
====================

A workqueue is defined using a variable of type :c:type:`struct k_work_q`.
The workqueue is initialized by defining the stack area used by its thread
and then calling :cpp:func:`k_work_q_start()`. The stack area must be defined
using :c:macro:`K_THREAD_STACK_DEFINE` to ensure it is properly set up in
memory.

The following code defines and initializes a workqueue.

.. code-block:: c

    #define MY_STACK_SIZE 512
    #define MY_PRIORITY 5

    K_THREAD_STACK_DEFINE(my_stack_area, MY_STACK_SIZE);

    struct k_work_q my_work_q;

    k_work_q_start(&my_work_q, my_stack_area,
                   K_THREAD_STACK_SIZEOF(my_stack_area), MY_PRIORITY);

Submitting a Work Item
======================

A work item is defined using a variable of type :c:type:`struct k_work`.
It must then be initialized by calling :cpp:func:`k_work_init()`.

An initialized work item can be submitted to the system workqueue by
calling :cpp:func:`k_work_submit()`, or to a specified workqueue by
calling :cpp:func:`k_work_submit_to_queue()`.

The following code demonstrates how an ISR can offload the printing
of error messages to the system workqueue. Note that if the ISR attempts
to resubmit the work item while it is still pending, the work item is left
unchanged and the associated error message will not be printed.

.. code-block:: c

    struct device_info {
        struct k_work work;
        char name[16]
    } my_device;

    void my_isr(void *arg)
    {
        ...
        if (error detected) {
            k_work_submit(&my_device.work);
	}
	...
    }

    void print_error(struct k_work *item)
    {
        struct device_info *the_device =
            CONTAINER_OF(item, struct device_info, work);
        printk("Got error on device %s\n", the_device->name);
    }

    /* initialize name info for a device */
    strcpy(my_device.name, "FOO_dev");

    /* initialize work item for printing device's error messages */
    k_work_init(&my_device.work, print_error);

    /* install my_isr() as interrupt handler for the device (not shown) */
    ...

Submitting a Delayed Work Item
==============================

A delayed work item is defined using a variable of type
:c:type:`struct k_delayed_work`. It must then be initialized by calling
:cpp:func:`k_delayed_work_init()`.

An initialized delayed work item can be submitted to the system workqueue by
calling :cpp:func:`k_delayed_work_submit()`, or to a specified workqueue by
calling :cpp:func:`k_delayed_work_submit_to_queue()`. A delayed work item
that has been submitted but not yet consumed by its workqueue can be canceled
by calling :cpp:func:`k_delayed_work_cancel()`.

Suggested Uses
**************

Use the system workqueue to defer complex interrupt-related processing
from an ISR to a cooperative thread. This allows the interrupt-related
processing to be done promptly without compromising the system's ability
to respond to subsequent interrupts, and does not require the application
to define an additional thread to do the processing.

Configuration Options
#####################

Related configuration options:

* :option:`CONFIG_SYSTEM_WORKQUEUE_STACK_SIZE`
* :option:`CONFIG_SYSTEM_WORKQUEUE_PRIORITY`
* :option:`CONFIG_MAIN_THREAD_PRIORITY`
* :option:`CONFIG_MAIN_STACK_SIZE`
* :option:`CONFIG_IDLE_STACK_SIZE`
* :option:`CONFIG_THREAD_CUSTOM_DATA`
* :option:`CONFIG_NUM_COOP_PRIORITIES`
* :option:`CONFIG_NUM_PREEMPT_PRIORITIES`
* :option:`CONFIG_TIMESLICING`
* :option:`CONFIG_TIMESLICE_SIZE`
* :option:`CONFIG_TIMESLICE_PRIORITY`
* :option:`CONFIG_USERSPACE`



API Reference
#############

.. doxygengroup:: thread_apis
   :project: Zephyr
