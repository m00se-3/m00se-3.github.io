---
layout: post
title: Cadlib - Building a Safe Wrapper for a Legacy API
catagory: "cadlib"
edited: 08-20-2026
---

In my days working as a Junior Civil3D Technician, there were always slow seasons during the winter months. One of these seasons presented 
me with an opportunity to test my ability to write software in a professional environment using real-life constraints. This post covers the background of
that project and the philosophy of CADLIB.

## The Need

I had learned the company's workflow for my department pretty well. However, there were bottlenecks that made good candidates for optimization. First, 
there were several places where basic data entry tasks were done manually. These could be automated. Second, many of our frequently used workflows could be 
done in fewer steps with greater accuracy. I was able to test the theory behind the second point using an AutoLISP routine. My coworkers and supervisor were impressed, so I was given permission to experiment further during the slow winter months.

I wanted to take things to the next level and, potentially, overhaul much of the department's work for greater accuracy and speed. This would ease some of the training 
difficulties we had and allow the company to take on more work. however, I was still needed to do my actual job, so the time I could spend on this project was going to be 
limited. Since this is the case, cadlib is as of yet an unfinished product. But it is complete enough to use and talk about.

## What is cadlib?

cadlib is a C++ 20 based wrapper library for AutoCAD's ObjectARX SDK. **Specifically, it wraps the version that is compatible with Civil3D 2020**. This
was the Civil3D edition my department used. It is a generic library that introduces modern coding practices to the legacy code. 
The ObjectARX library was just the thing I needed to make my ambitions happen. I could create a shared library in C++ and link it to AutoCAD at runtime. 
I would have access to more of Civil3D's functionality than I knew what to do with.

## The Issues

There were several problems I wanted to solve:

First, ObjectARX is very object-oriented and uses raw pointers everywhere. This meant checking for NULL many times and keeping track of everything
you need to delete. But it didn't stop there. Some objects require the **delete** operator while others required you use special function, depending on the
context. Also, you will be interacting with several objects at a time for any sophisticated transaction. This is a memory nightmare! In addition, the object hierarchy
is **vast** and heavily intertwined. I would need to bring order to the chaos to get anything done efficiently with reasonable guarantees.

Second, ObjectARX uses integer constants for things that really should enums, such as type identifiers for unions. Yes, I said unions. Arbitrary data retreived from 
AutoCAD often came packaged in a massive union! To identify the types you want, ObjectARX provides integer constants, *many* of them. This easily can spell disaster
for accidently passing the wrong code or an invalid code. I would much rather eliminate the possibility of that from runtime use altogether.

All of this means one thing, hard to read code that is fragile and difficult to debug! It would mean endless nested **for** loops over bare iterators, constant NULL 
checks, a tangle of error handling code, and whole lot of *undefined behavior*.

I needed a wrapper library that would accomplish the following things **without compromising performance or flexibility**:

- **Memory and type safe**, no null pointer dereferencing, out of bounds errors, or other undefined behavior.
- An API that is **easy to use correctly**, no swapping parameters, no raw strings when not needed, no switching over integers.
- Produce **algorithms that are easy to reason about**, minimize raw for loops, for example.
- **Efficiently handle errors**, no unnecessary if statements.

The posts that follow will go over these in greater detail.
