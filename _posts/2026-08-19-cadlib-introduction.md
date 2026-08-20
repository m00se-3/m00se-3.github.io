---
layout: base
title: CADLIB - Building a Safe Wrapper for a Legacy API
catagory: "cadlib"
---

## {{ page.title }}

In my days working as a Junior Civil3D Technician, there were always slow seasons during the winter months. One of these seasons presented 
me with an opportunity to test my ability to write software in a professional environment using real-life constraints. This post covers the background of
that project and the philosophy of CADLIB.

## The Need

I had learned the company's workflow for my department pretty well. However, there were bottlenecks that made good candidates for optimization. First, 
there were several places where basic data entry tasks were done manually. These could be automated. Second, many of our frequently used workflows could be 
done in fewer steps with greater accuracy. I was able to test the theory behind the second point using an AutoLISP routine. My coworkers and supervisor were impressed, so I was given permission to experiment further during the slow winter months.

I wanted to take things to the next level and, potentially, overhaul much of the department's work for greater accuracy and speed. This would ease some of the training 
difficulties we had and allow the company to take on more work.

## The Issues

AutoCAD's ObjectARX library was just the thing I needed to make my ambitions happen. I could create a shared library in C++ and link it to AutoCAD at runtime. 
I would have access to more of Civil3D's functionality than I knew what to do with.

There were several things that stood in the way, though:

The first was **time**. I was still needed to do my actual job, so the time I could spend on this project was going to be limited. This is what ultimately led to the project haulting later.

The second, **safety and future proofing**. The ObjectARX library is very object-oriented, uses raw pointers everywhere, and uses numeric codes for many things. It 
was a memory safety nightmare and was very error-prone for the inexperienced programmer. I needed to bring order to the chaos so that future coding would run smoothly.

I needed a wrapper library that would accomplish the following things **without compromising performance or flexibility**:

- No null pointer dereferencing, out of bounds errors, or other undefined behavior.
- An API that is easy to use correctly, no swapping parameters, no raw strings when not needed, no switching over integers.
- Produce algorithms that are easy to reason about, minimize raw for loops for example.
- Efficiently handle errors, no unnecessary if statements.

The posts that follow will go over these in greater detail.
