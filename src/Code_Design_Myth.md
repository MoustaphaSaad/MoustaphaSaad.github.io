# Code Design Myth

12, Mar, 2019

## The Myth

You go into your average CS school to learn that code design is important and you have to get it right to be a good software engineer.
You also should do a design phase where you take the requirements as an input and you "Design" the solution maybe using UML then you have a documentation of the design, etc..., If only real world was this trivial.

Spoiler alert, this shit doesn't work.

## What is good code?

Good code is the easiest to change. the definition of good code advances on a daily basis, maybe we'll discover the flaws of our ideas tomorrow/next month/next year. So you have to prepared and that by making your code easy to change without breaking the whole thing down.

Good code should have as little corner cases as possible, it also should be easy to understand given that i put the appropriate effort into understanding it.

Good code should normalize the skill level of the maintainer, a reasonably skilled developer should be as good as a 10-year veteran in maintaining the code.

Given the above points, good code should be simple to humans and it has the upside of being simple for the machine also so as a side effect you get a efficient code in return. Don't misunderstand that you seek performance, write simple enough code and it will be efficient by definition. Note that in case of choosing between performance and simplicity always favor simplicity and if performance is really necessary then separate your code into different layers that build upon each other.

For example let's consider the case where you are tasked with building an **easy to use**, **efficient** job queue since this is the latest thing i did. I divided it into 4 layers

- Layer 1: you are able to create a worker and jobs to its queue and it will consume jobs off the queue
- Layer 2: you are able to create a group of worker with round-robin style of job scheduling + job stealing. Note that until now the code is as simple as it gets and no dynamic allocation is done while spawning new jobs. and jobs are just function pointers of this signature `void func(void* arg1, void* arg2)`
- Layer 3: you are able schedule any C++ callable object as a job like lambda, function pointer, function objects, etc..., but at the cost of maybe doing dynamic allocation depending on the size
- Layer 4: you just declare the computation and let the underlying job system do its magic. using the simple `map` and `reduce` functions and they are responsible of dividing the jobs on multiple threads and collecting the result back

Using this separation you will enable ease of use in all the code and efficiency in areas where performance is important areas.

## True Design

Design in general is the process of taking decisions to do some task.
Decisions happen on multiple levels, in the process of writing code you take decisions like

- Level 0: deciding which keys to press on the keyboard
- Level 1: deciding variable names, function names, code style, indentation
- Level 2: deciding which (language features/variable ownership/usage code) you'll use to do the solve the problem at the moment
- Level 3: making an algorithm to solve the problem
- Level 4: making sense of how this piece will fit with other parts of the project

Note that each level influence the above layers, for example choosing a different language feature may lead you to a different algorithm altogether, Also algorithm may influence the lifetime/ownership of some code since it will dictate how the code should flow, etc...

Many developers have the misconception that design only happen on level 4 or maybe 3 and they ignore other lower layers, and that's wrong, every time you write struct, function signature, allocate memory, for loop, etc.. you are taking a design decision. and it's even more important to get right than other higher layer decisions.

A Good design should be invisible to the eye, it should be so easy you don't give it a second thought. First you should get familiar with the problem you are trying to solve. Don't spend much time on designing if you are stuck then drop this task for now and work on something else, maybe after that you'll know how to solve the problem.

So if you ever find yourself "Designing" a thing you are doing it wrong.