CUDA Stream Compaction
======================

**University of Pennsylvania, CIS 565: GPU Programming and Architecture, Project 2**

* David Liao
* Tested on: Windows 7 Professional, Intel(R) Xeon(R) CPU E5-1630 v4 @ 3.70 GHz 3.70 GHz, GTX 1070 8192MB (SIG Lab)
* Also Tested on: Windows 7 Professional, Intel(R) i7 4770 CPU @ 3.4GHz, Quadro K600 (SIG Lab)

### Analysis

![chart](images/chart.png)
#### Overall
All across the board, it appears that the CPU implementation lags behind all GPU implementations. The naive implementation trails behind the CPU implementation and the efficient GPU implementation beats the naive algorithm. Out of all 4 implementations, the Thrust implementation appears to be the fastest. It scales much better than the rest when given much larger arrays.

#### CPU
The cpu algorithm is a serialized sum iterated across the array. It is limited in computation by the fact that it is serial in nature.

#### Naive Implementation
The naive implementation is the initial parallelized scan that does not take into account the access-sensitive nature of its computation. Thus it suffers from mostly the fact that it has to deal with two memory buffers (to swap between). It does however beat the Work-Efficient algorithm on smaller data-sets. This might be due to the fact that less kernal invocations are called, thus less memory transfers between CPU and GPU occurs and more global memory accesses. 

#### Work-Efficient Implementation
This is an upgraded algorithm that allows in place computation without the extra memory buffer. This helps speed things up greatly. The initial implementation launched n threads for every iteration of the loop when the vast majority of them in the first few iterations will not be doing anything. This caused the Work-Efficient implementation to actually perform worse than the naive implementation. Instead, a smarter implementation would only launch as many threads as needed for that iteration. This is calculated using a bitshift counter to indicate the number of threads to launch in the kernel invocation. I could not get timing improvements for this as anything past 1 << 13 using the inefficient approach would crash the machine (Quadro K600) I was working on. But after I implemented the hack for launching the correct number of threads, everything ran smoothly.

#### Thrust
Thrust is the fastest implementation out of all 4. From NSight, it appears to used shared memory and is very efficient as compared to using global memory in our case which is very expensive. A quick search through thrust's git repo shows that they have multiple versions of scan that works on partitioned memory per warp or block. They also have an implementation working on global memory. Algorithm-wise, they appear to be using a similar up/down sweep algorithm. They also seem to use pick their block size very carefully based on empirical testing. Here's a [link](https://github.com/thrust/thrust/blob/2ef13096187b40a35a71451d09e49b14074b0859/thrust/system/cuda/detail/scan.inl) to their repo.

### Nsight Results
#### Thrust
Row 1: Accumulate Tiles
Row 2: Exclusive Scan
Row 3: Downsweep
![thrust](images/nsight_thrust.PNG)
#### Work Efficient Implementation
![work efficient](images/nsight_efficient.PNG)

#### Discussion
This is obtained on a sample input of 2^15 sized array.
Here we see thrust explicitly using dynamic shared memory. It has very efficient timings and high occupancy for Accumulate Tiles (as well as low register usage), but lower occupancy (and higher register usage) for the other two invocations. I think this is because it manages to schedule the sweeping functions in a way such that it only needs to launch a single kernel. It uses one launch for every operation as compared to the Work Efficient implementation where we see it launching 128, 64, 32, etc. kernels for each iteration of the loop. There's a lot of latency in-between the kernel launches. Despite this, each kernel invocation has a perfect occupancy percentage (yay! this means that our counting hack worked) and very low register usage. However, because it is sequential in launching the downsweep iterations and then the upsweep iterations, we take a hit in runtime. This on top shared memory usage makes thrust very fast. If we can maintain our high occupancy and low register usage, but manage to squeeze all of the kernal launches into a single one, we would be in great shape.

### Test Results
![tests](images/testspassing.png)
