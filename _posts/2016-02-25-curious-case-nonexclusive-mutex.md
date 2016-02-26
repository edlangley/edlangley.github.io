---
layout: article_page
title: "The curious case of the non-exclusive mutex...on ARM Cortex-A"
date: 2016-02-25 21:25:46
description: "Why a mutex primitive on ARM Cortex-A series processors might not be mutually exclusive"
tags: qnx embedded
---

Here's an interesting embedded software development bug I ended up dealing with, relating to mutual exclusion implementation on ARMv6 and later CPU cores. The project involved yet another board based on an i.MX6Q (Which contains four ARM Cortex-A9 CPUs), running QNX with SMP.

<!--more-->

For this specific board, one of the QNX drivers had been modified in such a way that it then needed to toggle one of the i.MX6 GPIO outputs on a regular basis. Meanwhile, another driver was already using a different GPIO in the same bank.

If a process wishes to set a bit in the GPIO output register, it must read the register to a variable, set the bit in the variable, and write that value back to the register. If another process should also happen to do this for another bit in the same register, at almost the same time, it might read the register before the first process has managed to write it's updated value back. Then the second process to write its value back to the register will clear the bit set by the first process.

Now, on some brands of system-on-a-chip, peripherals such as GPIO or DMA that use value or enable registers, where each bit in a register equates to one output pin level or channel enable, might have separate set and clear registers. That is, there will be the usual register address to read the current register contents, plus an address for a "set" register, where writing a 1 to a bit will only set that bit, i.e. writing a 0 will leave the bit at it's current value. Plus, a "clear" register, where writing 1 to a bit instead clears it to 0, again writing a 0 leaves the bit as-is. Such an arrangement would allow multiple users of the same GPIO bank register to merrily wiggle just their bits up and down all day long without bothering each other at all. Unfortunately the i.MX6, as well equipped as it is, only provides the standard read-modify-write arrangement for its GPIO output registers.

What should have happened at this point was to write another driver (In this case a QNX resource manager) to manage the GPIO controller, which the other drivers would call upon to update GPIO output register bits in a queued fashion.

However for the sake of two drivers each needing just a bit each, in one teensy weensy little 32 bit register, it was instead decided to perform the register updates in the respective drivers, guarding the activity using a pthread_mutex_t, shared between the two drivers. Whilst a less elegant and less future proof approach, this is still a valid method which meant less code to write and so (should have been) quicker to implement.

So here comes the important bit: Where to put that shared mutex? In order for it to be accessible from both driver processes? (For those unacquainted, QNX is fully micro-kernel, so these drivers were effectively user space applications).

On the first pass, the mutex went into the i.MX6 on-chip RAM, at physical address 0x900000:

{% highlight c %}
#define OCRAM_START_ADDR   0x00900000
#define OCRAM_MAPPED_SIZE  0x00000030

....

    shared_mem_ptr = mmap_device_memory(0, OCRAM_MAPPED_SIZE, PROT_READ | PROT_WRITE | PROT_NOCACHE, 0, OCRAM_START_ADDR);
    if(MAP_FAILED == shared_mem_ptr)
    {
        printf("mmap_device_memory for shared_mem_ptr failed");
        return -1;
    }

    mutex_ptr = (pthread_mutex_t *)shared_mem_ptr;

    if(EOK == pthread_mutexattr_init(&mutex_shared_attr))
    {
        if(EOK == pthread_mutexattr_setpshared(&mutex_shared_attr, PTHREAD_PROCESS_SHARED))
        {
            if(EOK != (result = pthread_mutex_init(mutex_ptr, &mutex_shared_attr)))
            {
                printf("pthread_mutex_init fail %d\n", result);
                return -1;
            }
        }
        else
        {
            printf("pthread_mutexattr_setpshared fail\n");
            return -1;
        }
    }
    else
    {
        printf("pthread_mutexattr_init fail\n");
        return -1;
    }
{% endhighlight %}

The significance of this decision was not appreciated at first, and the problem with this soon manifested itself. One of the drivers was using it's GPIO as the top address line to bank two NOR flashes, across which was mounted a single QNX flash filesystem. When the two drivers were ran together, you guessed it, the flash FS would quickly get corrupted.

After chasing these misleading symptoms, around GPIO register write latencies, and the rest of the flash driver, attention eventually came back to that mutex. Soon after that it came clear that the critical sections in both drivers were being entered at once. That is, the mutual exclusion primitive.... was not actually doing any excluding.

# Where's the 'ARM in it?

Traditionally, the explanation of how one process would keep another out of a critical section used to involve the use of a single assembly instruction which performs a test and set on a bit in a specified memory location (The details will vary depending on CPU architecture). The result of the instruction could be checked afterwards, to see if the value was already set or not. If it is a given that an assembly instruction is the smallest possible unit of execution in the processor, then this single concept can be used to implement all the variations of locking needed (Semaphores/mutexes) behind the desired API.

Older ARMs, up until v6, provided a slight variation on this concept on the same basis. The SWP and SWPB instructions atomically swapped a specified register with a specified memory location.

{% highlight asm %}
lock_mutex_swp
    LDR r2, =locked
    SWP r1, r2, [r0]       ; Swap R2 with location [R0], [R0] value placed in R1
    CMP r1, r2             ; Check if memory value was ‘locked’
    BEQ lock_mutex_swp     ; If so, retry immediately
    BX  lr                 ; If not, lock successful, return
    ENDP
{% endhighlight %}

However this technique became ineffective when multi-processor systems started to become popular. SWP is actually only atomic on the CPU it is running on, which for a single processor system is good as it means there is no need to disable interrupts. But on multi-processor systems it does not protect against other CPUs potentially executing the same SWP in parallel.

So in ARMv6, SWP was deprecated in favour of what ARM call exclusive accesses, an example implementation of which is given on this [ARM Infocenter page on exclusive accesses](http://infocenter.arm.com/help/index.jsp?topic=/com.arm.doc.dht0008a/ch01s03s02.html).

Now looking at some of the QNX libc source code from the past, when all of it except the kernel was available to the public, I see that in most normal, non priority ceiling cases, a call to pthread_mutex_lock will lead to entering a function named _mux_smp_cmpxchg.

By inspecting a dump of the QNX 6.6 libc, it appears this is still the case (Assembly explanation comments added by me, not objdump):

<div class="preformatted_console"><pre>
C:\>cd qnx660\target\qnx6\armle-v7\lib

C:\qnx660\target\qnx6\armle-v7\lib>ntoarmv7-objdump -d libc.so

....

0003b7c4 &lt;_mux_smp_cmpxchg>:
   3b7c4:	e1a0c000 	mov	ip, r0                         ; Address of "lock" variable from r0 -> ip
   3b7c8:	e19c0f9f 	ldrex	r0, [ip]                       ; Value of "lock" var -> r0, exclusive monitor active
   3b7cc:	e1300001 	teq	r0, r1                         ; Does lock var match expected "unlocked" value in r1?
   3b7d0:	018c3f92 	strexeq	r3, r2, [ip]                   ; If unlocked: "locked" value in r2 -> lock var
                                                                       ;              If the store exclusive worked, r3=0, else r3=1
   3b7d4:	03330001 	teqeq	r3, #1                         ; If unlocked: Did the store exclusive fail?
   3b7d8:	0afffffa 	beq	3b7c8 &lt;_mux_smp_cmpxchg+0x4>   ; If unlocked, or strex failed: Try ldrex again
   3b7dc:	f57ff05f 	dmb	sy                             ; Ensure the lock var is written to memory before further instructions
   3b7e0:	e12fff1e 	bx	lr                             ; Return
</pre></div>

The locking code in QNX's ARMv7 libc is very similar to that suggested by ARM. The parts of interest in that assembly dump are LDREX and STREX - special versions of the standard load and store instructions which mark a memory location as being monitored, until the CPU core which ran the LDREX runs a STREX back to the same address. The LDREX always performs the load from memory. However if another process was scheduled, or another CPU accessed the memory address, before the STREX executes, then the STREX will fail. The tracking of the lock status is performed below software control by what is called an "Exclusive monitor", a hardware state machine which takes note of the currently locked memory address, if any. There is actually one exclusive monitor per CPU core, known as the "local monitors", and one "global monitor" that sits watching the CPU interconnect bus, tracking the current LDREX/STREX lock in any shared memory regions.

As an aside, the ARM documentation is not explicit in saying that only one LDREX/STREX memory address can be locked at once on each CPU. This is OK because the point of the LDREX/STREX pair is that they are to be used close together to atomically read then write to a "lock" variable in memory, so only one use of the monitor will ever be needed at a time. If the process is pre-empted after the LDREX and before the STREX, when it is scheduled back in, the STREX will fail and set Z, forcing another attempt at LDREX.

So with all this in mind, what was the problem with placing the mutex in the on-chip SRAM then? Well on the [Infocenter page about LDREX and STREX](http://infocenter.arm.com/help/index.jsp?topic=/com.arm.doc.dht0008a/ch01s02s01.html) it says "Load-Exclusive and Store-Exclusive must only access memory regions marked as Normal". I'm not sure what constitutes a Normal memory region in that sense, but possibly not on-chip RAM Vs say, DRAM?

Also, in the next section discussing [exclusive monitors](http://infocenter.arm.com/help/index.jsp?topic=/com.arm.doc.dht0008a/ch01s03s02.html), the documentation then states "Each processor that supports exclusive accesses has a local monitor. Exclusive accesses to memory locations marked as Non-shareable are checked only against this local monitor". Assuming that this means marked as shareable in the MMU page-table mapping entry, the QNX documentation for it's mmap_device_memory api call states that it uses the MAP_SHARED flag by default. So the OCRAM mapping setup in the code snippet above should fit this criteria?

Either way, the code was changed to use what QNX is aware of as shared memory, residing in DRAM:

{% highlight c %}
    int shmFd;

    if((shmFd = shm_open("/shared_mutex_mem", O_RDWR | O_CREAT, 0777 )) < 0)
    {
        /* Set the memory object's size */
        if(-1 != ftruncate(shmFd, sizeof(pthread_mutex_t)))
        {
            if(MAP_FAILED != (shared_mem_ptr = mmap(0, sizeof(pthread_mutex_t), PROT_READ | PROT_WRITE, MAP_SHARED, shmFd, 0)))
            {
                printf("Mapped shared_mem_ptr to 0x%p\n", shared_mem_ptr);
            }
            else
            {
                printf("shared_mem_ptr mmap fail\n");
                return -1;
            }
        }
        else
        {
            printf("ftruncate: %s\n", strerror(errno));
            return -1;
        }
    }
    else
    {
        printf("shm_open fail\n");
        return -1;
    }

    ....
{% endhighlight %}

And the mutex then worked fine and all was well.

So the moral of the story is that if you want to share a mutex on an ARM system, be careful where you put it (And possibly how access to it is mapped) otherwise it may silently but completely fail in it's purpose. In the worst case, on a system in which such a lock is taken infrequently, with a very slim window for a race condition, could fail after many months or even years in the field. If that were the case, it might be an idea to add some test code to work the lock at a higher frequency during development, just to be safe.





