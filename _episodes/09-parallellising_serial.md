---
title: "Serial to Parallel"
teaching: 30
exercises: 30
questions:
- "What is the best way to write a parallel code?"
- "How do I parallelise my serial code?"
objectives:
- "Go through practical steps for going from a serial code to a parallel code"
keypoints:
- "Start from a working serial code"
- "Write a parallel implementation for each function or parallel region"
- "Connect the parallel regions with a minimal amount of communication"
- "Continuously compare with the working serial code"
---

## Going from Serial to Parallel

The examples used in the previous sections were perhaps not the most realistic.
In this section we will look at a more complete code and take it from serial to
parallel a couple of steps.

The exercises and solutions are based on the following code:

~~~
{% include code/poisson.c %}
~~~
{: .language-c .show-c }

~~~
{% include code/poisson.f08 %}
~~~
{: .language-fortran .show-fortran }


### Parallel regions

The first step is to identify parts of the code that
can be written in parallel.
Go through the algorithm and decide for each region if the data can be partitioned for parallel execution,
or if certain tasks can be separated and run in parallel.

Can you find large or time consuming serial regions?
The sum of the serial regions gives the minimum amount of time it will take to run the program.
If the serial parts are a significant part of the algorithm, it may not be possible to write an efficient parallel version.
Can you replace the serial parts with a different, more parallel algorithm?


> ## Parallel Regions
>
> Identify any parallel and serial regions in the code.
> What would be the optimal parallelisation strategy?
>
>> ## Solution
>>
>> The loops over space can be run in parallel.
>> There are parallisable loops in
>> * the setup, when initialising the fields
>> * the calculation of the time step, `unew`
>> * the difference, `unorm`
>> * overwriting the field `u`
>> * writing the files could be done in parallel
>>
>> The best strategy would seem to be parallelising the loops.
>> This will lead to a domain decomposed implementation with the
>> elements of the fields `u` and `rho` divided across the ranks.
>>
>{: .solution}
>
{: .challenge}

### Communication Patterns

If you decide it's worth your time to try to parallelise the problem,
the next step is to decide how the ranks will share the tasks or the data.
This should be done for each region separately, but taking into account the time it would take to reorganise the data if you decide to change the pattern between regions.

The parallelisation strategy is often based on the
underlying problem the algorithm is dealing with.
For example in materials science, it makes sense
to decompose the data into domains by splitting the
physical volume of the material.

Most programs will use the same pattern in every region.
There is a cost to reorganising data, mostly having to do with communicating large blocks between ranks.
This is really only a problem if done in a tight loop, many times per second.

> ## Communications Patterns
>
> How would you divide the data between the ranks?
> When does each rank need data from other ranks?
> Which ranks need data from which other ranks?
>
>> ## Solution
>>
>> Only one of the loops requires data from the other ranks,
>> and these are only nearest neighbours.
>>
>> Parallelising the loops would actually be the same thing as splitting the physical volume.
>> Each iteration of the loop accesses one element
>> in the `rho` and `uner` fields and four elements in
>> the `u` field.
>> The `u` field needs to be communicated if the value
>> is stored on a different rank.
>>
>> There is also a global reduction for calculating `unorm`.
>> Every node needs to know the result.
>>
>{: .solution}
>
{: .challenge}


### Set Up Tests

Write a set of tests for the function you are planning to edit.
The tests should be sufficient to make sure each function performs correctly
in any situation when all tests pass.
You should test all combinations of different types of parameters and preferably use random input.
Here the habit of modular programming is very useful. When the functions are small and have a small amount of possible input types, they are easy to test.

> ## Automated Testing
>
> There are automated unit testing frameworks for almost any programming language.
> Automated testing creatly simplifies the workflow of running your tests and
> verifying that the entire program is correct.
> The program here is still relatively simple, we are testing a single function,
> so writing a main function with a test case is enough.
>
>Check out the
>[cmocka](https://cmocka.org/)
>or 
>[cunit](http://cunit.sourceforge.net/)
>unit testing frameworks.
> {: .show-c}
>
> Check out the
> [fortran-unit-test](https://github.com/dongli/fortran-unit-test)
> unit testing frameworks.
> {: .show-fortran}
>
{: .callout}

> ## Write a Test
>
> Change the main function of the poisson code to check that the poisson_step function
> produces the correct result.
> Test at least the output after 1 step and after 10 steps.
>
> What would you need to check to make sure the function is absolutely correct?
> 
>> ## Solution
>> ~~~
{% include code/poisson_test.c %}
>> ~~~
>>{: .source .language-c}
>{: .solution .show-c }
>
>> ## Solution
>> Here we create a module for the poisson solver, including only the
>> subroutine for performing a single step.
>>~~~
{% include code/poisson_test.f08 %}
>>~~~
>>{: .source .language-fortran}
>{: .solution .show-fortran }
>
{: .challenge}

> ## Make the test work in parallel
>
> Modify the test so that it runs correctly with multiple mpi ranks.
> The tests should succeed when run with a single rank and fail with any
> other number of ranks.
>
> At this point you will need to decide how the data is arranged before the
> function is called.
> Will it be distributed before the function is called or you split and 
> distribute the data each time the function is called?
> In this case the `poisson_step` function is called many times and splitting
> the data up each time would take a lot of time, so we assume it is
> distributed before the function call.
>
> Let's split the outer loop, indexed with j, across the ranks.
> Each rank will have a slice of `GRIDSIZE/n_ranks` points in the `j`
> direction.
> Modify the main function so that it works properly with MPI and
> each rank only initializes it's own data.
> 
>> ## Solution
>> ~~~
{% include code/poisson_test.c %}
>> ~~~
>>{: .source .language-c}
>{: .solution .show-c }
>
>> ## Solution
>>~~~
{% include code/poisson_test2.f08 %}
>>~~~
>>{: .source .language-fortran}
>{: .solution .show-fortran }
>
{: .challenge}

### Write a Parallel Function Thinking About a Single Rank

In the message passing framework, all ranks execute the same code.
When writing a parallel code with MPI, you should think of a single rank.
What does this rank need to to and what information does it need to do it?

Communicate data in the simplest possible way
just after it's created or just before it's needed.
Use blocking or non-blocking communication, which ever you feel is simpler.
If you use non-blocking functions, call wait immediately.

The point is to write a simple code that works correctly.
You can optimise later.

> ## Parallel Execution
>
> First, just implement a single step.
> Write a program that performs the iterations from
> `j=rank*(GRIDSIZE/n_ranks)` to `j=(rank+1)*(GRIDSIZE/n_ranks)`.
> For this, you should not need to communicate the field `u`.
> You will need a reduction.
>
> After this step the first test should succeed and the second test
> should fail.
>
>> ## Solution
>>
>>~~~
{% include code/poisson_mpi_step1.c %}
>>~~~
>>{: .source .language-c}
>{: .solution .show-c}
>
>
>> ## Solution
>>
>>~~~
{% include code/poisson_mpi_step1.f08 %}
>>~~~
>>{: .source .language-fortran}
>{: .solution .show-fortran}
>
{: .challenge}



> ## Communication
> 
> Add in the nearest neighbour communication.
> The second test should then succeed.
>
>> ## Solution
>> Each rank needs to send the values at `u[1]` down to `rank-1` and
>> the values at `u[my_j_max]` to `rank+1`.
>> There needs to be an exception for the first and the last rank.
>>
>>~~~
{% include code/poisson_mpi_step2.c %}
>>~~~
>>{: .source .language-c}
>{: .solution .show-c}
>
>
>> ## Solution
>> Each rank needs to send the values at `u(1)` down to `rank-1` and
>> the values at `u(my_j_max)` to `rank+1`.
>> There needs to be an exeption for the first and the last rank.
>>
>>~~~
{% include code/poisson_mpi_step2.f08 %}
>>~~~
>>{: .source .language-fortran}
>>
>{: .solution .show-fortran}
>
{: .challenge}

### Optimise

Now it's time to think about optimising the code.
Try it with a different numbers of ranks, up to
the maximum number you expect to use.
In the next lesson we will talk about a parallel profiler that can produce
useful information for this.

It's also good to check the speed with just one rank.
In most cases it can be made as fast as a serial code
using the same algorithm.

When making changes to improve performance, keep running the test suite.
There is no reason to write a program that is fast but produces wrong
results.

> ## Strong Scaling
>
> Strong scaling means increasing the number of ranks without
> changing the problem.
> An algorithm with good strong scaling behaviour allows you to
> solve a problem more and more quickly by throwing more cores
> at it.
> There are no real problems where you can achieve perfect strong scaling
> but some do get close.
>
> Modify the main function to call the poisson step 100 times and set `GRIDSIZE=512`.
>
> Try running your program with an increasing number of ranks.
> Measure the time taken using the Unix `time` utility.
> What are the limitations on scaling?
>
>> ## Solution 
>>
>> Exact numbers depend on the machine you're running on, 
>> but with a small number of ranks (up to 4) you should
>> see the time decrease with the number of ranks.
>> At 8 ranks the result is a bit worse and doubling again to 16 has little effect.
>>
>> The figure below shows an example of the scaling with
>> `GRIDSIZE=512` and `GRIDSIZE=2048`.
>> ![A figure showing the result described above for GRIDSIZE=512 and GRIDSIZE=2048]({{ page.root }}{% link fig/poisson_scaling_plot.png %})
>>
>> In the example, which runs on a machine with two 20-core Intel Xeon Scalable CPUs,
>> using 32 ranks actually takes more time.
>> The 32 ranks don't fit on one CPU and communicating between
>> the the two CPUs takes more time, even though they are in the same machine.
>>
>> The communication could be made more efficient.
>> We could use non-blocking communication and do some of the computation
>> while communication is happening.
>>
>{: .solution}
>
{: .challenge}

> ## Weak Scaling
>
> Weak scaling refers to increasing the size of the problem
> while increasing the number of ranks.
> This is much easier than strong scaling and there are several
> algorithms with good weak scaling.
> In fact our algorithm for integrating the Poisson equation
> might well have perfect weak scaling.
>
> Good weak scaling allows you to solve much larger problems
> using HPC systems.
>
> Run the Poisson solver with and increasing number of ranks,
> increasing `GRIDSIZE` only in the `j` direction with the number
> of ranks.
> How does it behave?
>
>> ## Solution
>> 
>> Depending on the machine you're running on, the 
>> runtime should be relatively constant.
>> Runtime increase if you need to use nodes that are
>> further away from each other in the network.
>>
>> The figure below shows an example of the scaling with
>> `GRIDSIZE=128*n_ranks`.
>> This is quickly implemented by settings `my_j_max` to `128`.
>> ![A figure showing a slowly increasing amount of time as a function of n_ranks]({{ page.root }}{% link fig/poisson_scaling_plot_weak.png %})
>>
>> In this case all the ranks are running on the same node with
>> 40 cores. The increase in runtime is probably due to the
>> memory bandwidth of the node being used by a larger number of
>> cores.
>>
>{: .solution}
{: .challenge}

{% include links.md %}

