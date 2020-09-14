---
title: Introduction To Virtualization
date: 2020-09-14 17:30:47 +07:00
modified: 2020-09-14 17:30:47 +07:00
tags: [windows-kernel, hevd]
---
### Double Fetch
Now we will exploit the double fetch bug.

This kind of bug happens when the user supplied data is fetched twice, for example, there is a ioctl that recives an array of
chars and it length, if the function will once check the size and second will copy using it, the exact same reference to the variable.
It exposes itself to the double-fetch bug.

This is also called TOC-TOU TimeOfCheck and TimeOfUse, when you are fetching this value you are exposing yourself to that the user will be able to change this data between the check and acatual use thus the vulnerability.

#### Analysing the binary
We are interested in the function `TriggerDoubleFetch`.

| ![could not load photo] (/assets/hevd-writeups/double_fetch_function_analysis.png) |

First of all we are getting some data from the function, for instance, the size of the kernel buffer

Then a check is made, the size of we supplied vs the size of the kernel buffer size in order to prevent overflow

If we passed the check our buffer is copied to kernel buffer using memcpy and the size we sent.

Beacause the function uses our "fetches" the size twice we have a window of oppurtunities to change it's value.

`esi is our passed paramater`
So if we look again at what we can achive, we can get OOB(out of bounds) write on the stack.

#### Explotation

In this section I'll explain how to exploit the bug we saw earliar, I will only use parts of my code, I'll link to the full exploit in the end

Ok, so we want two things two things to happen
1. Change the value before its use and after the check. 
2. Use the OOB write to jump to an address of our choosing - will be done with changing the size were sending

So in theory we will run two threads, one that will repetadly engage with the driver and will trigger a double fetch vulnerability and another to change the value of the size being sent.

Ok, lets create two threads and attach our functions
| ![could not load photo] (/assets/hevd-writeups/double_fetch_create_threads.png) |

Before we will pop a cmd with system we will fail.
At first my VM was with one processor, we must think about our OS resources, for example, how much processors we have? a low number (<4) will give us a hard time when trying to exploit.
It took me quite some time to trigger the exploit using two processors

Also, we need to consider the fact that our threads are not alone in the system and we are not even in the highest priority for our system

Lets change our threads priority
| ![could not load photo] (/assets/hevd-writeups/double_fetch_set_priority.png) |

And a check to verify the amount of processors
| ![could not load photo] (/assets/hevd-writeups/double_fetch_check_processors.png) |

Now we can run the exploit with success
| ![could not load photo] (/assets/hevd-writeups/double_fetch_system.png) |

Another thing we can do is set each of our threads to a different processor, so he will not be competing with the our second thread about the processor resources
| ![could not load photo] (/assets/hevd-writeups/double_fetch_set_processor.png) |

where i represents the location of a bit in a bitmask that represents the processor number (i == 0 -> first processor)

ofcourse, we will set by our machine capabillities or the call will fail.