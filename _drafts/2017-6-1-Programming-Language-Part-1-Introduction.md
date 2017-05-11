---
layout: post
title: Programming Language - Part 1 - Introduction
---

In this series we are going to attempt to create our own programming language - sounds 
simple enough right?

You may think, why would you want to create your own language, what motivations do I have, 
or what problems will it solve that aren't done already in other languages? The answer to this is simple, to learn - I'm certainly not expecting this to become the next big language!

At the time of starting this project I will be starting from near-zero knowledge, that is you 
could sum up what I know so far as the following:

1. Have some syntax (keeping it simple to start is usually good advice!)
2. Tokenize the source text into tokens
3. Create an AST from the tokens
4. Declaration and type checking etc
5. Code optimisation / generation
6. Output any errors during the various phases

As far as implementation goes, I'll be figuring out 99% of this as I go, very different to the 
usual web applications I do for my day job to say the least! 

It's probably worth pointing out at this point that I'm a self-taught developer (aren't we 
all really?), and by that I've had no formal education in CS or relevant field, I've learnt 
everything as I go - but this is also a hobby and passion, not just a job!

Initially I'm going to be implementing this language in C#, mostly because it's the main language 
I use daily - you may notice some syntax familiarity if you use C# yourself.

## High-level goals

The high level goals of the language will be to use a C#-like syntax which will allow compilation 
to web assembly (WASM) and/or native (via LLVM).

This may sound like some lofty goals straight off the bat - and you'd be right, but at this point, 
why not? As the project progresses I'm positive that the goals will change, so right now why 
not be ambitious?

## Sample syntax

Here is some example syntax of what I'm envisioning - at least currently:

```
import SomeLib;

module ExampleModule
{
    public class ExampleClass
    {
        private int _intField;

        // C# style properties - including expression bodies
        public int IntProperty { get; set; }

        public int ReturnInput(int value)
        {
            return value; 
        }
        public int Add(int val1, int val2)
        {
            return val1 + val2;
        }

        public constructor(int a)
        {
            _intField = a;
        }
    }

    public string ModuleLevelMethod(string input)
    {
        return string + input;
    }
}
``` 

Those familar with C#, Typescript or Java-like languages should find the syntax familiar with few 
minor differences, the main one being module level method declarations - I envision this 
replacing the need for static methods / classes at this stage.

