# Intro
I want to thank Pascal Pons for this amazing repo, and the super clear explanations on his blog. 

I've be implementing connect4 in multiple languages over the last 35 years starting in GW-Basic. Most recently I implemented a version in C for this [kaggle competition](https://www.kaggle.com/competitions/connectx/overview) 
My code was a bit slower than Pascals', but more than fast enough to implement a perfect agent in kaggle using a large opening book. 

I wanted to see how long it takes on a modern processor to 'solve' connect4 from scratch. (solve = tell us for each of the 7 opening moves if you win/lose/draw and when in the game). 

As a comparison in 1995 it took [John Tromp](https://tromp.github.io/c4/c4.htm) 40000 hours on Sun and SGI workstations. In this repo I'm going to show it now takes less than a minute on my macbook pro (M1 Max). And because Pascal Pons' code is so clear and simple I'm starting from his solution.

# Step 1. A Baseline
Just run the code as is.

```
% git clone git@github.com:pcnudde/connect4.git
% make
% time echo "" | ./c4solver -a
Unable to load opening book: 7x6.book
 -2 -1 0 1 0 -1 -2
./c4solver -a  402.81s user 0.71s system 99% cpu 6:43.83 total

|  | Duration (sec) |
| --- | --- |
| Baseline | 402 |

