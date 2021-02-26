---
title: Notes about v8 deoptimization
date: 2021-02-26 17:30:47 +07:00
modified: 2021-02-26 17:30:47 +07:00
tags: [v8, deoptimization, simplified-lowering]
---

Those are my raw notes that i wrote while reading and diving to [this article](https://doar-e.github.io/blog/2020/11/17/modern-attacks-on-the-chrome-browser-optimizations-and-deoptimizations/).

I wanted people to see the questions and thinking process for someone that started this article with zero to little knowledge about ignition source code or the simplified-lowering optimization.

ofcourse if you see a mistake i'd be happy to fix it :)

## Start 

deoptimization input data -> to know what kind of deoptimization is to be done, where are we going back?
what kind of frame should we build, where to deptimize (offset of bytecode) 

deoptimization types:
eager: triggered type guard - code that has invalidated itself
lazy: optimized code that has been invalidated by execution of other code
sofy: normal, the function was optimized too early

dependencies in the context of deopt:
	installing code dependecies on global variable, all code obj that depend on this var will be deopt once it is mutated
	it basicly knows only when we check for the 'if' statement if we changed the global var

the UseInfo class is used to describe a use of an input of a node. => AnyTagged: any - Truncation, tagged - UseInfo
Truncation means how can i make this more specific kind
machine representation is like 'useInfo' but more specific, e.g  'kWord64' insteak of 'Tagged'

ProcessInput -> u can actually see the type for the input params -> UseInfo
SetOutput -> sets the output type for the specified node

what's restriction type? -> maybe what type could be input? like some kind of mechanisem to restrict what input we can get
machineType -> simplifies version of machineRepresentation

for every instr that could cause deopt, we have a block called frameStateDescriptor to give us information about the deopt
now, it is using 'Translation' in order to get info of the output frame, this is
input frame -> output frame.
the StateValueList is a list of `StateValueDescriptor`, those are the inputs of the frameState,
and u can link them to the var's job at the `Translation` .

run with --turbo-profiling and --print-code-verbose in order to view the deopt input data (aka stateValueDescriptor)

{ANY} -> The input representation is undetermined. That is the most generic case.

the bytecode is implicitly loading and storing to the accumulator, the second param is the index for the feedback vector

if u look at the bytecode handlers those that use the value of the accumulator, we can use them to maybe gain 
stronger primitives, e.g 'Add', 'StaKeyedProperty'.

looking at the implementation of the bytecode handler can be a pain in the ass because of all the macros

everything that is not a smi is heap allocated - bigUint is 'heapNumber'

release does'nt have the --print-code flag -> always run tests with debug version
learn more about the csa-builtins - the Label, BIND thing there.

you can control the value of the accumulator by changing the order, 'y + 1n' != '1n + y'.

by what order does the bind branches are checked?

number != bigint -- https://v8.dev/blog/react-cliff

## bug specific
there was a mistake in the translation to the output frame, inside 'AddTranslationForOperand we
check for kInt64 constants and because of 'DeoptMachineTypeOf' of 'BigInt' the machine type returned is 'AnyTagged' which caused
the translation to think that the value is an address instead of raw number.

primitive - we can rematerialize an object from a user-controlled value

so if we have an info leak, we can set the BigInt to be the address of some heap number and leak its value
the limitation though is quite frustrating.

looks like when we try to 'y / 1n' the exploit does not work, should compare bytecode with and without - when the number we divide with is 'typeof bigint'
when i look at the source it looks like it's the same for both, the branch itself is different but in the end they are both loaded with 'LoadHeapNumberValue'
looked at different bytecode handler, next time first look at the --print-bytecode instead of assuming it's the same, the correct handler is 'Add' instead of 'SmiAdd'

## questions
1. why does the number must be in the range of 49 bits?
2. declaration of the 'a' variable?
3. why do we need the try catch? we dont throw an exception, we only deoptimize
4. why do we need the param we get
5. why only the first time of the xpl works? second, etc' returns the address as decimal.

## answers
1. the fraction size is 52 bits in IEEE-754 representation, something else.
   maybe related to the process addr space, only 49 are used?
   we first need to look at the docs, what is this param to 'asUintN', the param is used for setting the max number we can store inside the 'bigint' object, look at the example in the docs
   based on that, doesnt seem to have any special meaning to this number besides setting a limit on the size of the address
   
2. we dont really need it, it's ok to remove.

3. we go to a different branch inside the handler if {lhs} and {rhs} are both 'typeof bigint', 
   we want to get a specific branch where we load the value as a heap number.

4. we dont need, ok to remove.

5. i think we dont deopt in the second time, need to check, yes!
   the reason we deopt is 'Insufficent type feedback for binary operation' when we run it enough times we wont get it again because we will have enough type information
   
   in the second poc we deoptimize because of 'wrong name' (see different deopt reasons in deoptimize-reason.h) we just need to call it with a different param we

## Understand better
the steps of the simplified lowering, such as the truncation, propagation and lowering - read the source file, [simplified-lowering.cc](https://chromium.googlesource.com/external/v8/+/cb1b554a837bb47ec718c1542d462cb2ac2aa0fd/src/compiler/simplified-lowering.cc#35)