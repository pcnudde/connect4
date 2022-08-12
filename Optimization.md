## Intro
I want to thank Pascal Pons for this amazing repo, and the super clear explanations on his blog. 

I've be implementing connect4 in multiple languages over the last 35 years starting in GW-Basic. Most recently I implemented a version in C for this [kaggle competition](https://www.kaggle.com/competitions/connectx/overview) 
My code was a bit slower than Pascals', but more than fast enough to implement a perfect agent in kaggle using a large opening book. 

I wanted to see how long it takes on a modern processor to 'solve' connect4 from scratch. (solve = tell us for each of the 7 opening moves if you win/lose/draw and when in the game). 

As a comparison in 1995 it took [John Tromp](https://tromp.github.io/c4/c4.htm) 40000 hours on Sun and SGI workstations. In this repo I'm going to show it now takes less than a minute on my macbook pro (M1 Max). And because Pascal Pons' code is so clear and simple I'm starting from his solution.

## Setup
All benchmarks are run on my 2021 macbook pro with an M1 Max processor. (8 performance cores, 2 efficiency cores and 64GB memory)

## Step 1. A Baseline
Just run the code as is.

```
% git clone https://github.com/PascalPons/connect4.git
% make
% time echo "" | ./c4solver -a
Unable to load opening book: 7x6.book
 -2 -1 0 1 0 -1 -2
./c4solver -a  402.81s user 0.71s system 99% cpu 6:43.83 total
```

| Experiment | Duration (sec) |
| --- | --- |
| Baseline | 403 |

This is already an amazingly fast starting point.

## Step 2. The obvious: Let's make it multithreaded.
Given that the the analysis code processes all 7 positions independently and that we have 8 cores, we should get a nice speedup.
C++ futures/async feature makes making this multithreaded [pretty easy](https://github.com/pcnudde/connect4/commit/0c16303d792054acfc39e71f69bad816897544a0#diff-f0068c48beb298a9e249776ef8175a18e8958896b0be6c81a3cf73f450a35350).
We need to make sure we keep multiple transposition tables, one per thread. Both because the table is not threadsafe, but even if it were it is much faster to have separate ones.


| Experiment | Duration (sec) | Speedup
| --- | --- | --- |
| Baseline | 403 | | 
| Multithreaded | 112 | 3.5x |

Almost a 4x increase using 7 threads. We did not expect 7x as the slowest of the 7 moves (the winning middle move) takes 90 sec to compute alone, and there is relatively high memory bandwidth to the transposition tables.

## Step 3. Let's play with the memory.
We have more memory, so let's see if things improve if in increase or decrease the size of the transposition table. It is not clear ahead of time what will be better. For this algorithm, I have seen on some CPUs that smaller tables that fit in cache are better than larger tables in main memory. But the M1 has very good memory bandwidth. 

In our case using the [maximum memory](https://github.com/pcnudde/connect4/commit/6e49b82747b4c307f5f86a06608c45dad7f3cd49#diff-304adfca6c3c2f30bfaabb24f52c8ef03ce929e5b6232cf449431ec6fae90cc8) helps. So with 40GB, we improve with another 70% to get almost to 1 min.

| Experiment | Memory (GB) | Duration (sec) | Speedup
| --- | --- | --- | --- |
| Baseline | | 403 | | 
| Multithreaded |  0.650 | 112 | 3.5x |
| Multithreaded (small table) | 0.04 | 240 | |
| Multithreaded (big table) | 40 | 65 | 70% |

## Step 4. What about some SIMD?
This is not the obvious application for SIMD, but the key function that gets called most often, _compute_winning_position_, is very heavy on bit operations, and if we can do 128 bit in parallel instead of 64 there might be some benefit. The M1 processor supports ARM NEON instruction set with 128 bit. Instead of coding the function completely in assembly, I decided just to use Neon [intrinsics](https://developer.arm.com/documentation/102467/0100/Why-Neon-Intrinsics-). This enables me to [stay in C++](https://github.com/pcnudde/connect4/commit/7b9a9331b2eca1536c68320fc4a03286d9833e4e#diff-2217d624822064f23bf5930540d7716ed3f1f7d6c43ed89d0a454cc83fa67cbf) and not having to do manual register allocation while getting most of the benefit.

| Experiment | Memory (GB) | Duration (sec) | Speedup
| --- | --- | --- | --- |
| Baseline | | 402 | | 
| Multithreaded |  0.650 | 112 | 3.5x |
| Multithreaded (big table) | 40 | 65 | 70% |
| Neon SIMD | 40 | 57 | 14% | 

We are now below the magical 1 min mark. While the new function actually is twice as fast as the old one (I timed it separately), the overall benefit is only 14%. In general, this would not be worth it, but here we are going for speed.

## Step 5. Smaller things.
Now it gets harder as the low hanging fruit is gone. I tried various things:

* use -Ofast optimization: no difference
* replace popcount with a [built in optimized version](https://github.com/pcnudde/connect4/commit/038153b685a86c3064232047235ac0cbbaf2d922#diff-2217d624822064f23bf5930540d7716ed3f1f7d6c43ed89d0a454cc83fa67cbf). less than 0.5 sec improvement
* switch from gcc to clang: less than 0.5 sec improvement

There probably are some more things to do, but I don't think there is that much more optimization possible. 
My guess is that a totally different approach is needed to go faster. Don't do an optimized negamax, but employ the GPU to evaluate in parallel many more positions at a higher speed. Maybe this is a good time to learn some CUDA or Metal programming :). I wonder if 10 seconds is possible!


| Experiment | Memory (GB) | Duration (sec) | Speedup
| --- | --- | --- | --- |
| Baseline | | 402 | | 
| Multithreaded |  0.650 | 112 | 3.5x |
| Multithreaded (big table) | 40 | 65 | 70% |
| Neon SIMD | 40 | 57 | 14% | 
| Final| 40 | 56 | 1% | 


## Step 6. Feedback from Kaggle and final optimizations.
I posted this on kaggle and got some new ideas from Tony Robinson:

* Exploit symmetry in opening position. We only have to evaluate 4 positions and not all 7. 3 second gain. But we can probably leverage more threads later in the compute as we have 8 cores.
* [MTD(f)](https://people.csail.mit.edu/plaat/mtdf.html) 

I also found a few more improvements:
* Not resetting the transposition table to zero. 1 second gain
* Add some extra multithreading in the negamax to use more cores. 8 more seconds


| Experiment | Memory (GB) | Duration (sec) | Speedup
| --- | --- | --- | --- |
| Baseline | | 402 | | 
| Multithreaded |  0.650 | 112 | 3.5x |
| Multithreaded (big table) | 40 | 65 | 70% |
| Neon SIMD | 40 | 57 | 14% | 
| Final| 40 | 56 | 1% | 
| Symmetry | 25 | 53 | 5% | 
| MTD(f) | 25 | 50 | 6% | 
| Final | 25 | 41 | 20% | 




## Conclusion.
It was fun to see how fast this algorithm could run on a modern CPU. One minute is a pretty nice result.

```
time echo "" | ./c4solver -a
 -2 -1 0 1 0 -1 -2
./c4solver -a  257.02s user 4.77s system 465% cpu 56.271 total
```

The complete code can be found [here](https://github.com/pcnudde/connect4/tree/optimized).

