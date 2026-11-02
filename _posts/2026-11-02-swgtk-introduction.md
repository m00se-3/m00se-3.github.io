---
layout: post
title: "SWGTK - A Project for Learning Game Engine and Library Design."
tag: "swgtk"
---

## Introduction

Ever since I was a kid, I wanted to know how games were made. That was one of the motivating factors of me getting my Associates Degree in 2014. Even 
though life has not allowed me to land that dream as a career, as of this time writing, I still love the idea of making games!

However, unlike most game developers, I'm not creatively inclined. I'm an engineer at heart. Every time I try to pick up a game engine and start a 
project it feels empty, it feels overkill. Finding the right engine feels like trying to choose from a large aisle of cleaning products, each one missing
something crucial.

Combining my love for games and my desire to create a program that feels like a well oiled machine has led me down the route of writing game engines. Having 
little experience making games myself, though, makes that more difficult since I don't know all the common game design patterns. After years of experimenting
and watching how others approach making games on the internet, I feel this problem is soon to be solved.

So here is my intoduction to SWGTK, my latest and greatest effort to create a reusable game framework.

## A Game Engine from What?

Though I'm not shy about writing code at any level, there are practical limits to how deep it is necessary to go to write my own engine. Cross-platform multi
media libraries are a great place to start from, since OS level programming would only distract from the goal of making games. I've tried most of the standard 
ones, [SFML](https://www.sfml-dev.org), [GLFW](https://www.glfw.org), and [SDL](https://www.libsdl.org). Ultimately, I fell in love with SDL. It gives me
all the tools I need without compromising flexibility. It's also **very** widely cross-platform. It has first class support on Steam, my favorite gaming platform. 

In addition, with SDL3, the library is more powerful than ever. I now have the ability to create rendering backends, with or without shader support, using the
very familiar SDL-like way. But the best part is, it now has a 3D capable generic GPU pipeline. Since I have some - but not very much - GPU programming experience,
having an API like this is astronomically helpful!

## What is SWGTK Made to Do?

One thing I wanted to accomplish with SWGTK is to make the API friendly for people who love to code and want to be in control of their game's infrastructure. 
I've seen how other frameworks, like [Raylib](https://www.raylib.com), accomplish this in C. I wanted to take this idea and lean into C++'s features and strengths.
What *exactly* that looks like has gone through many changes over the years, as I will explain in future posts.
