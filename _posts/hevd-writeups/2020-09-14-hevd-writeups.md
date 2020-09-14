---
title: HEVD writeups
date: 2020-09-14 17:30:47 +07:00
modified: 2020-09-14 17:30:47 +07:00
tags: [windows, kernel, hevd]
---
## Double Fetch
This kind of bug happens when the user supplied data is fetched twice, for example, there is a ioctl that recives an array of
chars and its length, if the function will check the size and will copy using it (the exact same reference to the variable).
It will expose itself to the double-fetch bug.

This is also called TOC-TOU, TimeOfCheck and TimeOfUse, when you are fetching this value for the second time you are exposing yourself to the fact that the user will be able to change this data between the check and acatual use thus the vulnerability.

#### Analysing the binary
We are interested in the function `TriggerDoubleFetch`.

![could not load photo](/assets/hevd-writeups/double_fetch_function_analysis.png)

First of all the function prints for us some data.
Then a check is made, the size that we supplied vs the size of the kernel buffer size in order to prevent overflow
If we passed the check, our buffer is copied to kernel buffer using memcpy and the size we sent.
The fact that the function "fetches" the size twice we have a window of oppurtunities to change it's value.
So if we look again at what we can achive, we can get OOB(out of bounds) write on the stack.

#### Explotation

Ok, so we want two things two things to happen
1. Change the value before its use and after the check. 
2. Jump to an arbitrary address of our choosing

So we will run two threads, one that will repeatedly engage with the driver and will trigger a double fetch vulnerability and another to change the value of the size being sent.

Ok, lets create two threads and attach our functions
![could not load photo](/assets/hevd-writeups/double_fetch_create_threads.png)

In order to pop a cmd with system we need to consider something else like our computer resources
At first my VM was with one processor, we must think about our OS resources, for example, how much processors we have? a low number (<4) will give us a hard time when trying to exploit.
It took me quite some time to trigger the exploit using two processors

Also, we need to consider the fact that our threads are not alone in the system and we are not even in the highest priority for our system

Lets change our threads priority
![could not load photo](/assets/hevd-writeups/double_fetch_set_priority.png) 

And a check to verify the amount of processors
![could not load photo](/assets/hevd-writeups/double_fetch_check_processors.png) 

Now we can run the exploit with success
![could not load photo](/assets/hevd-writeups/double_fetch_system.png) 

Another thing we can do is set each of our threads to a different processor, so he will not be competing with the our second thread about the processor resources
![could not load photo](/assets/hevd-writeups/double_fetch_set_processor.png) 

where i represents the location of a bit in a bitmask that represents the processor number</br> (i == 0 -> first processor)

ofcourse, we will set by our machine capabillities or the call will fail.


[full source code](https://github.com/yuvaly0/HEVD_Solutions/blob/master/HEVD_Solutions/DoubleFetch.cpp)