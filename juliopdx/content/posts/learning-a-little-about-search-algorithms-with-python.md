---
title: "Learning a Little About Search Algorithms With Python"
date: 2022-01-12T15:12:30-08:00
draft: false
toc: true
images:
  - /blog/images/markus-spiske.jpg
tags:
  - Python
---

## Introduction

Hello and thank you for tuning in to another episode. First off, I was recently notified that this blog made it to the Cisco IT Blog Awards final! Pretty wild to think about, but thank you to all those involved. This one is definitely going to be different and out of my comfort zone and I appreciate you sticking with me.

I have been working through the Introduction to Computer Science course by CS50x (Harvard). The course does an incredible job of walking learners through the very basics of programming with Scratch and the C language. The course eventually shifts over to Python for the remainder of the course. While taking the course, you learn about arrays, algorithms, memory, and more. David Malan and team really do an incredible job and try to make the barrier of entry as low as possible (free and includes a web based VScode editor).

In week three we learn a bit about algorithms. This was easily one of my favorite weeks. The class walks through different search and sorting algorithms. These are then compared performance wise. This post will go over two search algorithms; linear and binary. I will use some subnets from the 10.0.0.0/8 network for our examples.

## Creating the Data

I decided to create a big list of subnets with some Python. The code below will produce three files that can be used for testing.

`ip_create.py`

```python
"""Small script to create test files"""
import ipaddress
import random

net = ipaddress.ip_network("10.0.0.0/8").subnets(new_prefix=24)

networks = [str(n.network_address) for n in net]

# Ordered list of subnets
with open("ordered_ip.txt", "w", encoding="utf-8") as file:
    for net in networks:
        file.write(f"{net}\n")

# Reversed list of subnets
networks.reverse()
with open("reverse_ip.txt", "w", encoding="utf-8") as file:
    for net in networks:
        file.write(f"{net}\n")

# Random list of subnets
random.shuffle(networks)
with open("random_ip.txt", "w", encoding="utf-8") as file:
    for net in networks:
        file.write(f"{net}\n")
```

`ordered_ip.txt`

```text
10.0.0.0
10.0.1.0
10.0.2.0
10.0.3.0
10.0.4.0
10.0.5.0
10.0.6.0
10.0.7.0
10.0.8.0
10.0.9.0
...
```

`random_ip.txt`

```text
10.44.148.0
10.47.64.0
10.166.151.0
10.139.81.0
10.237.230.0
10.139.3.0
10.179.165.0
10.96.37.0
10.26.247.0
10.12.255.0
...
```

`reverse_ip.txt`

```text
10.255.255.0
10.255.254.0
10.255.253.0
10.255.252.0
10.255.251.0
10.255.250.0
10.255.249.0
10.255.248.0
10.255.247.0
...
```

## Linear Search

Linear search is fairly simple in nature. We take a list or array and check one by one until we find our desired value.

```python
Value: 10 20 30 40 50 60 70
Index:  0  1  2  3  4  5  6
```

For example, using the data above. If we wanted to check for the value of 70, we would first have to check 10 then 20 then 30 and so on. This means it would take us “n” steps. Think of “n” as the size of the array. In our example the array is of size 7. Worst case scenario would be 7 checks. This can then be referenced as “O(n)” or O of n. Now what about the best case scenario? Well, if we are looking for the 10 in this array, then it only requires one step. This is known as “Ω(1)” or Omega of one.

## Binary Search

Binary search works a bit differently than linear search. Binary search does have one additional requirement, the array must be sorted. If not, the algorithm breaks. Binary search works by continuously making the size of the array smaller. Essentially making the search size smaller on each iteration.

```python
# First Check
Value: 10 20 30 40 50 60 70
Index:  0  1  2  3  4  5  6
Start: 0
End: 6
Midpoint: 3
```

Same array as above but with additional information. We have the start of the array at 0 and the end of the array at 6. The midpoint is found by (start + end) / 2. In this case we are looking for 70. The midpoint(40) is lower than our target(70) so the start will shift to midpoint + 1.

```python
# Second Check
Value: 50 60 70
Index:  4  5  6
Start: 4
End: 6
Midpoint: 5
```

On the second check we are searching through a smaller piece of the array. Midpoint is found by (6 + 4) / 2 = 5. The target is higher than our midpoint of 70. So again we will shift the start to midpoint (5) + 1.

```python
# Third Check
Value: 70
Index:  6
Start: 6
End: 6
Midpoint: 6

# Value is Found
```

On this third check the midpoint is equal to our target, the value is found! This will also work in the opposite, but we will decrease the end to (midpoint – 1). Whats the big deal? In binary search, when the size of the array increases, the amount of steps will not grow as large as linear search. Making the algorithm much more efficient. Best case scenario, on the initial midpoint we find our target. This is similar to linear search, Ω(1). What about worst case? That is known as O(log n). Long story short, much faster.

## Comparing With Subnets

We will be working with subnets as our data. To make things more simple, I will convert these to their decimal equivalent. What do I mean by this? Please see example below!

```python
>>> from ipaddress import ip_address
>>> int(ip_address("0.0.0.1"))
1
>>> int(ip_address("0.0.1.0"))
256
>>>
```

When I run the comparisons, in reality they will be comparing integer values vs strings of subnets.

### Linear Seach in Code

```python
with open(sys.argv[1], "r", encoding="utf-8") as file:
    data = [int(ip_address(line.strip())) for line in file.readlines()]

my_target = int(
    (ip_address(input("What /24 prefix in 10.0.0.0/8 are you looking for? ")))
)

for prefix in data:
    time.sleep(DELAY)
    if prefix == my_target:
        print(f"Found: {ip_address(my_target)} with linear search")
        break
```

Initially we open one of the text files and use some list comprehension to create our list of integers that represent different subnets. Then we ask the user to enter a subnet, for example, “10.80.40.0”. We will then compare each prefix or subnet against our target. If we find it, we will convert the integer back to a cleaner format. You may have caught the sleep setting in the code. Well it turns out computers are really fast, so we will add some artificial delay. This is used in both search algorithms.

### Binary Search in Code

```python
def bin_search(some_array: list, target: int):
    """Simple binary search function"""
    array = sorted(some_array)
    start = 0
    end = len(array) - 1
    found = False
    while not found and start <= end:
        time.sleep(DELAY)
        middle = int((start + end) / 2)
        if int(array[middle]) == target:
            found = True
            print(f"Found: {ip_address(target)} with binary search")
        elif int(array[middle]) > target:
            end = middle - 1
        else:
            start = middle + 1
```

I wrapped this in a function and set it to accept two parameters, some array and a target. We will first sort the array since that is required in binary search. We then set our start and end values. Once that is done, we will run a while loop and keep making the search smaller and smaller until the target is found. Lets check out some examples!

## Examples

```python
# Best case scenario for Linear
❯ python algo.py ordered_ip.txt
What /24 prefix in 10.0.0.0/8 are you looking for? 10.0.0.0
Found: 10.0.0.0 with linear search
Elapsed time: 0.0801 seconds
Found: 10.0.0.0 with binary search
Elapsed time: 0.0045 seconds
```

```python
# Worst case scenario for Linear
# Binary did not increase much at all!
❯ python algo.py ordered_ip.txt
What /24 prefix in 10.0.0.0/8 are you looking for? 10.255.255.0
Found: 10.255.255.0 with linear search
Elapsed time: 11.0045 seconds
Found: 10.255.255.0 with binary search
Elapsed time: 0.0050 seconds
```

```python
# Random list of subnets
❯ python algo.py random_ip.txt
What /24 prefix in 10.0.0.0/8 are you looking for? 10.80.40.0
Found: 10.80.40.0 with linear search
Elapsed time: 10.6104 seconds
Found: 10.80.40.0 with binary search
Elapsed time: 0.0159 seconds
```

## Outro and Links

Thank you all for reading this far, it really means a lot and please go vote for your favorite IT blogs.

- [Featured Image by Markus Spiske](https://unsplash.com/photos/uPXs5Vx5bIg)
- [CS50x Introduction to Computer Science](https://cs50.harvard.edu/x/2022/)
- [GitHub Repository](https://github.com/JulioPDX/ne-algo-search)
- [Cisco IT Blog Awards](https://www.cisco.com/c/en/us/training-events/events-webinars/influencer-hub/blog-awards.html)
