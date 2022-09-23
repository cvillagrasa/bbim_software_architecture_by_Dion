# BlenderBIM Software Architecture explanation by Dion Moult

The following is an extract of the [OSArch Live Chat](https://osarch.org/chat/) on 23rd September, 2022.

It consists of explanations from main BlenderBIM developer [Dion Moult](https://github.com/Moult) to incomer BlenderBIM 
developer (myself). Hopefully, having the following invaluable information as a complement of the 
[BlenderBIM Developer Documentation](https://blenderbim.org/docs/devs/hello_world.html) can help incoming devs with
the same questions.

# OSArch Live Chat extract
## What is the primary purpose of Core?

The primary purpose of the core is to break down large complex software functionality into simple, short, 
english-reading functions.

## But why is there Core, Tool and core.tool? Why not just write well-organized unified code and its corresponding tests? Do we really need to type function blueprints three times in Python?

Indeed, it feels like "extra work", but it's worth it in the long term.

Let's say you started writing a new function and decided to do it "all in one function". You'd create a Blender 
operator class, and write a long (say 50+ lines of code) function to do the task.

That function would query bpy.context.scene, it might query blender objects, it might manipulate IFC data, and some 
general vector math for example.

Firstly, if that function were long, you'd break it up into small functions anyway, right? so instead of one long 
function, you'd have 5 small functions, and you'd give those functions meaningful, english-like names that are 
descriptive, like "move_object_axis" or "edit_pset" and so on.

However, there's still a problem, all those 5 functions are in your one blender operator class. what if another 
operator wants to do something similar, and can reuse a function or two? following the DRY principle, you'd want to 
share the functions.

## Sure, I wouldn't place the functions in the operator class.

So then you'd start taking out those functions and putting them into another class/module/whatever. So now you have say:
- 1 main entry function in the operator that describes the overall flow of that operator
- 4 "shared functions" in a shared tool class

## Yes, up to now that's what I’d do

Cool, and that has the added side effect that you can now write 5 simple tests for 5 functions instead of many complex 
tests for many permutations for 1 large function.

So far, we've recreated the logic of the "simple operator" + tool + tool tests.

But there's still a few problems. Blender operators rely on Blender state to be called. which means if you want to 
write a test for the Blender operator, you need to 1) launch Blender 2) setup scene state 3) make sure context is 
right...

Also, if hackerX wants to write his own script and reuse portions of bbim's functionality, he's need to call 
`bpy.ops.whatever_function()`, which typically will read from Blender props, so first he'd need to do things like set 
Blender props, select objects, and only then do bpy.ops.whatever_function, make sure poll state is true...

Which is also known as "automating the UI" which is a very slow, inflexible approach to writing scripts. Also, 
if hackerX wants to write his own script and reuse portions of bbim's functionality, he's need to call 
`bpy.ops.whatever_function()`

## Ok, but why doesn't hackerX just use code from Tool?

He can, but don't forget Tool is a collection of 4 small functions, he'd need to know the right way to call each 
individual function in tool and in the right order.

The operator "entry" function handles all that for him, and also if we fix something in the future in the operator 
function, he automatically gets the benefit.

Don't forget that operators are not "simple". when you edit an objects material, should it recreate the wall's geometry, 
recalcaulte its quantities? update opening positions? change cost value? change scheduling duration? think of how many 
"tool calls" there are.

That's why most people approaching Blender scripting just call `bpy.ops.foo() :)

Another problem of the Blender operator is that your tests also have the same problem.

## But if hackerX wants exactly the same as the existing operator does, then there's nothing to do. If he wants to make something else with certain overlapping functionality, then he can reuse the parts of Tool as necessary. I don't see how an operator fix outside of Tool functions would benefit him.

"There's nothing to do" <-- no, there's a lot to do, he needs to set the right props and scene state.

e.g. if he wants to call `bpy.ops.add_pset()` he needs to manually select an object in the scene, then he needs to find 
the right props for the pset dropdown, then he needs to manually trigger the prop refresh code, and only then call the 
add_pset... then he doesn't even get a "pset" element returned... since operators cannot return anything.

So then he needs to ensure the object is selected again, then call `tool.ifc.get_entity`, then reverse engineer 
`util.element.get_psets() to "find" the pset he may or may not have created and find its id, then 
`tool.Ifc.get().by_id()` to get the pset.

It ends up being many, many lines of code just to manage blender's UX, when ideally it should be just 
`pset = add_pset(obj, "MyPset").

## Ok, I'm starting to understand something 😂

It's not the only problem though. Not only does hackerX have problems, you also have problems because you need to write 
tests for the operator function.

So all the same problems hackerX has, you also have, but you have an extra problem because your tests now require 
Blender to run, and that make sthings 100x slower to run tests (yes i've measured the test run times, and it is so 
slow it cannot scale to test all of BlenderBIM's functionality in the future).

And a second problem not only is it slow to run tests, it's slow to run code itself. what if your operator code wants 
to run another operator? e.g. you want to call `bpy.ops.bar()` in your `bpy.ops.foo()`? you have hackerX's problems all 
over again, plus you get a speed penalty (and a significant penalty, I've measured it and it resulted in very noticeable 
slowdowns for the user)
calling a blender operator is very expensive.

To give you an idea, tests run 10-20 times slower, and In the case where `bpy.ops.bim.copy_class(obj="my object name")` 
is called as a suboperator, I counted about 3 seconds to copy 40 objects (note: I've commented out the expensive 
add_representation subsuboperator - to purely test the suboperator call). If I instead call `copy_class(obj)` as a plain 
Python function, not a Blender operator, 3 seconds drops to basically instant. 

This speed boost is perhaps also made more extreme because operators can only pass simple primitives, so the operator 
call requires an additional object name lookup.

So a function that previously took 3 seconds now is instant, that's a crazy speed up.

## Ok, so if you test Tool, everything's blazingly fast, got it

No, if you separate operator from a core function, things get blazingly fast and you solve all of hackerX's problems 
and your test problems too, and suboperator call problems too.

## But Core functions usually contain calls to operators, don't they?

Nope, core functions call tools, which may call other core functions, but it should never call another operator 
(no other BlenderBIM operator, that is, sometimes a tool will call a regular Blender `bpy.ops` because there is no 
alternative)

## I've just looked it up, they contain calls to ifc.run, I got confused there...

Yes, `ifc.run is a tool function.

## Which has strings as an argument, and my head explodes 😅😂

Hehehe we'll get there, there's a logic to all of this, and lots of lessons learned.

So now we've covered the logic of changing from "1 operator + 4 tool functions" to "1 operator calling 1 core function 
calling 4 tool functions"

And absolutely, when writing a small blender addon all of this is overkill you're right, but when you're dealing with 
40 modules of occasionally interdependent logic...

Well, hopefully you understand the 1op + 4tools to 1op+1core+4tools switch logic so far, with the speed and hackerX 
problems.

## Yes, just with the minor detail of why core functions have the Tool module as an argument. Are there real scenarios where there can be more than one module? Couldn't it be made as a module-level choice? and remove that arg?

Yep, we'll get there don't worry.

So the next issue is once you've separated your op and core, you still need to write a test for your core function.

But there are two issues with writing tests for your core function:

Firstly, you want your tests to be fast, yes, omitting the blender operator call is already a speed boost, but at the 
same time your core still depends on tools, and tools still depend on bpy data, and so to run tests, you need to fire 
up Blender which is slooooow (you need to also clear fresh blender sessions for every test).

Secondly, you already have separate tests for all your 4 tool functions, what's the point of your test for the core 
function? you don't want to write the same test twice, and deal with all the nitty gritty details of the tool tests.

Ideally you want to write a "high level" test just of logic flow, that is super fast and doesn't depend on Blender, 
because the only thing the core does is high level logic, that's the only thing you want to test.

Both of these problems can be solved by making the core function not call the tool classes directly, but instead be fed 
the tool classes via an argument. This technique is known as "Dependency Injection". It means the core is no longer 
dependant on the tool, the tool can be created from "anywhere"

With this trick (which is actually a famous design pattern used almost everywhere), it means you can write a test that 
sends in a "fake tool"... an "impostor" that pretends it’s a tool, but instead is only there to check wheter the core 
is performing the intended logic flow properly

That way your tests can run without Blender at all, and even without all the details of whatever the tool needs 
(e.g. even without IFC).

The "impostor" comes from a class known as "Prophecy":
https://github.com/IfcOpenShell/IfcOpenShell/blob/v0.7.0/src/blenderbim/test/core/bootstrap.py#L234

You might already be aware of "mocks", "stubs" and "spies" from unit testing libraries 
(in any language, not just Python).

This "Prophecy" is a variation of a mock object, inspired by my history working with ruby-style PHP code which I find 
much, much more expressive and easy to write than traditional mock objects.

It's described a bit here:
https://blenderbim.org/docs/devs/running_tests.html#core-tests

## Ok, I'll study testing a bit more, because it seems that at least in this case, it’s driving the code structure.

Indeed, code readability and dev maintainability should drive code structure - they are critical for a software 
project to scale and last.

You will notice as BlenderBIM has grown in complexity, stability has been increasing, and yet code functions are 
staying small, and dev time for new features relatively constant as opposed to growing exponentially, that is the 
evidence that the structure works.

Ignoring the tests, BlenderBIM itself is almost 50,000 lines of code! (90k if considering tests).

Think of the alternative, if your core code called `tool.Foo.bar()` directly which needed to access `bpy.something`, 
then it becomes impossible to run your test without making blender run too and managing even a tiny bit of Blender 
state (at the very least, clearing the blender session before each test, and setting a new ifc file object).

Similarly if your core code called `tool.Foo.bar()` which called `ifcopenshell.something`, then to run a single test 
you'd need not only to setup an ifc model, you'd need to make sure it had exactly the right data in it too.

## I think one of the only doubts I have left is about core.tool. Is it really necessary? isn't it the same as to look at the function outliner from the tool module?

Ah yeah the core tool is critical :)

So now we have 1op + 1core + 4tool + 1testcore + 4testtools.

In Python they use duck typing so in theory you don't need core tool at all, and yes if you have a fancy IDE you can 
just list all functions in the tool module.

If you don't have a fancy IDE, that's one purpose of core tool, that it does offer a nice listing of functions and 
constantly makes you think "how can i better organise my tools" as you code
or "is there already a tool that does X that i can reuse".

But it serves a deeper purpose, tied into how the core tests work with the mock (prophecies)
let's imagine core tool didn't exist. and you wanted to write a test for a core function to call `tool.Foo()`. how do 
you know that `tool.Foo()` exists? you won't, since you've only got a mock object
so in theory you could lie to yourself and write a green passing core test that `tool.Foo()` is called when 
`tool.Foo` never exists.

Alternatively, what if `tool.Foo.bar()` does exist, and you use it in multiple core functions, but then one day you 
decide to refactor and clean up and rename it to `tool.Foo.baz()`. how will you know which core functions you need to 
update?

Or, if you decide to delete `tool.Foo.bar()`, then how do you know which core functions will break because 
`tool.Foo.bar()` no longer exists?

Or another scenario, if you change the function signature of `tool.Foo.bar` to have different arguments, 
which core functions will break?

You won't know the answer to these questions with your core tests, because your core tests are using mock prophecy 
objects only.

You _will_ know the answer to these questions in your full integration tests, but full integration tests are very, 
very, veeeeery slow to run, and you may not have tests for every single permutation of user behaviour because they 
are so slow, so coverage is not complete.

The solution is to create `core.tool` ... effectively an "Interface" class in other more strongly typed languages, 
then base the mock objects off these Interface classes.

So if any tool gets renamed, deleted, signature changed, the prophecy based on the `core.tool` interface will instantly 
recognise a change has been made and cause the correlating core tests to fail.

Similarly, if you implement a tool that doesn't follow the interface, it comes up as an error, so if you start 
delegating work between different developers (E.g. some work on tool, some work on ui, some work on core and api), 
then that makes sure you can do code integration correctly.