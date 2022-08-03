# Intro
I want to thank Pascal Pons for this amazing repo, and the super clear explanations on his blog. 

I've be implementing connect4 in multiple languages over the last 35 years starting in GW-Basic. Most recently I implemented a version in C for this [kaggle competition](https://www.kaggle.com/competitions/connectx/overview) 
My code was a bit slower than Pascals', but more than fast enough to implement a perfect agent in kaggle using a large opening book. 

I wanted to see how long it takes on a modern processor to 'solve' connect4 from scratch. (solve = tell us for each of the 7 opening moves if you win/lose/draw and when in the game). 

As a comparison in 1995 it took [John Tromp](https://tromp.github.io/c4/c4.htm) 40000 hours on Sun and SGI workstations. In this repo I'm going to show it now takes less than a minute on my macbook pro (M1 Max). And because Pascal Pons' code is so clear and simple I'm starting from his solution.

# Setup
All benchmarks will be run on my 2021 macbook pro with an M1 Max processor. (8 performance cores, 2 efficiency cores and 64GB memory)

# Step 1. A Baseline
Just run the code as is.

```
% git clone git@github.com:pcnudde/connect4.git
% make
% time echo "" | ./c4solver -a
Unable to load opening book: 7x6.book
 -2 -1 0 1 0 -1 -2
./c4solver -a  402.81s user 0.71s system 99% cpu 6:43.83 total
```

| Experiment | Duration (sec) |
| --- | --- |
| Baseline | 402 |

# Step 2. The obvious. Let's make it multithreaded.
Given that the the analysis code processes all 7 positions independently and we have 8 cores we should get a nice speedup.
C++ futures/async makes making this multithreaded pretty easy.
We need to make sure we keep multiple transposition tables, one per thread. Both because the table is not threadsafe, but even if it were it is likely much faster to have separate ones.


| Experiment | Duration (sec) | Speedup
| --- | --- | --- |
| Baseline | 402 | | 
| Multithreaded | 112 | 350% |

Almost a 4x increase using 8 threads. We did not expect 8x as the slowest of the 7 moves (the winning middle move) takes 90 sec to compute alone, and there is relatively high memory bandwith to the transposition table.

# Step 3. Let's play with the memory.
We have more memory, so let's see if things improve if in increase or decrease. It is not clear what will be better, for this algorithm I have seen on some CPUs that smaller tables that fit in cache are better that larger tables in main memory. But the M1 has very good memory bandwidth so lets see. 

In our case using the maximum memory helps. So with 40GB we improve with another 170% to get almost at 1 min.

| Experiment | Memory (GB) | Duration (sec) | Speedup
| --- | --- | --- | --- |
| Baseline | | 402 | | 
| Multithreaded |  0.650 | 112 | 350% |
| Multithreaded (small) | 0.04 | 240 | |
| Multithreaded (big) | 40 | 65 | 170% |



 



