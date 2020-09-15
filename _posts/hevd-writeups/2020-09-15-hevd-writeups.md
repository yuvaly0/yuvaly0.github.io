---
title: HEVD writeups
date: 2020-09-15 17:30:47 +07:00
modified: 2020-09-15 17:30:47 +07:00
tags: [windows, kernel, hevd]
---

## Non Paged Pool Overflow
This bug will occur when writing too much data to a buffer, in this case, the buffer is in the non paged pool.

For example, a function receives a buffer and just copies it to her buffer without checking its length.

#### Analysing the binary
We are interested in the function `TriggerNonPagedPoolOverflow`

![could not load photo](/assets/hevd-writeups/pool_overflow/1_function_analysis.png)
![2_function_analysis](/assets/hevd-writeups/pool_overflow/2_function_analysis.png)

First of all the function creates a chunk using the function [ExAllocatePoolWithTag](https://docs.microsoft.com/en-us/windows-hardware/drivers/ddi/wdm/nf-wdm-exallocatepoolwithtag), the requested chunk is in size of 0x1f8, it's tag will be 'Hack' (important) and it will be allocated in the non paged pool.

We're given the information mentioned above and some on our sent buffer and size and the kernel buffer which is the chunk mentioned earlier.

Next, they use `memcpy` to copy our buffer with our given size (the vulnerability) to their buffer without any size checking.

Lastly, they free the chunk.

#### Explotation
Our goal is to pop a cmd with system permissions

1. Derandomize the pool (get to a predictable state) - heap spray
2. Trigger the overflow and overwrite a shellcode address
3. jump to shellcode and pop a cmd

We cant just the overwrite the buffer because it will mess up the pool structure and cause a BSOD:

![code to cause BSOD visual studio](/assets/hevd-writeups/pool_overflow/3_cause_bsod.png)
![bsod - windbg](/assets/hevd-writeups/pool_overflow/4_bsod_windbg.png)


##### Derandomize the pool
We will use a technique called heap spray.
This technique is used to get the pool to a controlled state, because of its allocator mechanism this is possible.

But with what objects? </br>

We can use the event object, they are each sizeof 0x40, but if we will multiply by 8 will get 0x200 which is the size of our driver allocated chunk 0x1f8 (+ 0x8 for the _POOL_HEADER struct)

![code for heap spray](/assets/hevd-writeups/pool_overflow/5_heap_spray_code.png)
![handles](/assets/hevd-writeups/pool_overflow/6_handles.png)
![predictable heap-windbg](/assets/hevd-writeups/pool_overflow/7_heap_spray_allocations.png)

The idea is that the first wave will derandomize and the second wave will start in a state where the pool is already derandomized.
We could do it in one wave.

##### Trigger Overflow
The `TypeIndex` field in the _OBJECT_HEADER is an index to a table of pointers that will contain pointers to different OBJECT_TYPE types.

![show object header](/assets/hevd-writeups/pool_overflow/8_object_header.png)

Inside this structure, there are a couple of pointers for functions, such as: `OkayToCloseProcedure`, `CloseProcedure`.

![show procedures](/assets/hevd-writeups/pool_overflow/10_procedures.png)

So if we could overwrite the pointer to the table, causing it to point to another index in the table say - the first index (0) is pointed to null address, because we operating on windows 7 we can allocate a null page and simulate the OBJECT_TYPE there giving us the ability to control EIP

![show table](/assets/hevd-writeups/pool_overflow/9_type_index_table.png)

But we cannot just overwrite the object header with random or even some other chunk metadata because its unsafe.
Because we know that we will overwrite an event object we can take one of our known headers, after all, we know all of them are the same (at least until the IndexType offset).
![view raw header data](/assets/hevd-writeups/pool_overflow/11_payload_data_colored.png)

##### Getting system
Now we need to concat all of our previous steps and run the program:
![show system](/assets/hevd-writeups/pool_overflow/12_system.png)

[full source code](https://github.com/yuvaly0/HEVD_Solutions/blob/master/HEVD_Solutions/NonPagedPoolOverflow.cpp)


## Double Fetch
This kind of bug happens when the user-supplied data is fetched twice, for example, there is an ioctl that receives an array of
chars and its length, if the function will check the size and will copy using it (the same reference to the variable).
It will expose itself to the double-fetch bug.

This is also called TOC-TOU, TimeOfCheck and TimeOfUse, when you are fetching this value for the second time you are exposing yourself to the fact that the user will be able to change this data between the check and the use thus the vulnerability.

#### Analysing the binary
We are interested in the function `TriggerDoubleFetch`.

![could not load photo](/assets/hevd-writeups/double_fetch_function_analysis.png)

First of all the function prints for us some data.
Then a check is made, the size that we supplied vs the size of the kernel buffer size to prevent overflow
If we passed the check, our buffer is copied to the kernel buffer using memcpy and the size we sent.
The fact that the function "fetches" the size twice we have a window of opportunities to change its value.
So if we look again at what we can achieve, we can get OOB(out of bounds) write on the stack.

#### Explotation

Ok, so we want two things to happen
1. Change the value before its use and after the check. 
2. Jump to an arbitrary address of our choosing

So we will run two threads, one that will repeatedly engage with the driver and will trigger a double fetch vulnerability and another to change the value of the size being sent.

Ok, let's create two threads and attach our functions
![could not load photo](/assets/hevd-writeups/double_fetch_create_threads.png)

To pop a cmd with system we need to consider something else like our computer resources
At first, my VM was with one processor, we must think about our OS resources, for example, how much processors we have? a low number (<4) will give us a hard time when trying to exploit.
It took me quite some time to trigger the exploit using two processors

Also, we need to consider the fact that our threads are not alone in the system and we are not even in the highest priority for our system

Let's change our threads priority
![could not load photo](/assets/hevd-writeups/double_fetch_set_priority.png) 

And a check to verify the number of processors
![could not load photo](/assets/hevd-writeups/double_fetch_check_processors.png) 

Now we can run the exploit with success
![could not load photo](/assets/hevd-writeups/double_fetch_system.png) 

Another thing we can do is set each of our threads to a different processor, so he will not be competing with our second thread about the processor resources
![could not load photo](/assets/hevd-writeups/double_fetch_set_processor.png) 

where i represents the location of a bit in a bitmask that represents the processor number</br> (i == 0 -> first processor)

of course, we will set by our machine capabilities or the call will fail.


[full source code](https://github.com/yuvaly0/HEVD_Solutions/blob/master/HEVD_Solutions/DoubleFetch.cpp)