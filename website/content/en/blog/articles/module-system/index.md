---
date: 2020-11-23
title: "A module system for Blech"
linkTitle: "Modules for Blech"
description: >
    This article motivates and explains the Blech module system
author: Franz-Josef Grosch
---

The Blech module system supports modular programming for reactive, embedded, safety-critical applications. 
With the languages used in this domain today, we rely on coding conventions, programming idioms and the physical code structure to organize software in a modular way. 
This is difficult and error prone. 
As a consequence today's systems are often more monolithic than modular.

Blech's synchronous paradigm together with the upcoming module system enables and supports modular programming.
It has the following properties: 

- Module implementations encapsulate code.
- Modules are namespaces for code entities.
- Module interfaces are automatically generated from implementations and their import/export declarations.
- Module interfaces hide implementation details.
- The import hierarchy is always a directed acyclic graph, supporting a layered modular software structure.
- Every layer is separately testable and reusable.
- The compiler recursively compiles programs and modules along the dependency hierarchy.
- Optionally importing all implementation details allows for white-box testing.
- Modules can be packaged to libraries - called *boxes*.
- Boxes are namespaces for modules.
- Boxes can hide internal implementation modules.
- The syntax for modules is light-weight.
- Reasoning about the modular structure is easy.
- Modules and boxes map to files and directories.
- All static analysis is designed to work with separate compilation.

The details follow now.

In order to motivate the design of the Blech module system, we will refer to the wikipedia article on [Modular Programming](https://en.wikipedia.org/wiki/Modular_programming) [[1]](#ModularProgramming) and a blog post on [How to write Large Programs](https://medium.com/@olegalexander/how-to-write-large-programs-628c90a70615) [[2]](#LargePrograms) .

Let's start with the definition of modular programming:
> Modular programming is a software design technique that emphasizes separating the functionality of a program into independent, interchangeable modules, such that each contains [...] only one aspect of the desired functionality. [[1]](#ModularProgramming)

Blech supports this design technique by grouping code into modules.

## Module implementation

To keep things simple, every Blech file can be compiled separately and is either a program or a module.
A module file groups closely related Blech entities - constants, types, functions, activities, etc. - into a compilation unit, that can be used (imported) by other modules or programs.
The keyword `module` classifies the content of a Blech file as a module.
Different to modules, a program file requires an `@[EntryPoint]` activity which serves as the main activity of a Blech program. 
A program file cannot be imported by other modules or programs.

Assume the following module implementation in file `ringbuffer.blc`.
In order to make an entity accessible it needs to be exposed.
Here, functions `initialise`, `push`, and `average` are exposed for usage, in any importing module or program.

```blech
module exposes initialise, push, average

const Size: nat8 = 10

struct RingBuffer
    var buffer: [Size]nat32
    var nextIndex: nat8
    var count: nat8
end

/// returns an initialisation value for a ring buffer
function initialise () returns RingBuffer
    return { nextIndex = 0, count = 0 }
end

/// pushes a new value to the ring buffer
/// displaces the "oldest" value if the ring buffer is full
function push (value: nat32) (rb: RingBuffer)
    rb.buffer[rb.nextIndex] = value
    rb.nextIndex = rb.nextIndex + 1
    if rb.count = Size then // ringbuffer ist completely filled
        rb.nextIndex = rb.nextIndex % Size
    else
        rb.count = rb.count + 1
    end
end

/// calculates the average value of all values stored in the ring buffer
function average (rb: RingBuffer) returns nat32
    var idx: nat8 = 0
    var avg: nat32 = 0
    while idx < rb.count do
        avg = avg + rb.buffer[idx]
    end
    return avg / rb.count
end
```

## Module interface

> A module interface expresses the elements that are provided and required by the module. The elements defined in the interface are detectable by other modules. The implementation contains the working code that corresponds to the elements declared in the interface. [[1]](#ModularProgramming) 

A module interface in Blech is called a `signature`.
The signature is generated by the compiler from the module implementation and the `exposes` declaration.
A Blech programmer never writes a signature file.
After compilation of file `ringbuffer.blc` you will find the following module interface in file `ringbuffer.blh`.

```blech
signature

type RingBuffer

/// returns an initialisation value for a ring buffer
function initialise () returns RingBuffer

/// pushes a new value to the ring buffer
/// replaces the "oldest" value if the ring buffer is completely filled
function push (value: nat32) (rb: RingBuffer)

/// calculates the average value of all values stored in the ring buffer
function average (rb: RingBuffer) returns nat32
```

The exposed functions additionally need the type `RingBuffer`.

Since `struct RingBuffer` is not exposed in the module it is made available as an opaque `type RingBuffer` in the signature.
While the implementation remains hidden, the opaque type can be used to define variables and assign values of this type.
This allows to change the internal representation, while any code the uses the module remains unchanged. 
Opaque types are sometimes also called abstract types, and are a simple form of the theoretical concept of existential types.


Signatures carry all information necessary to compile any code that uses this module. 
This enables to package Blech modules into libraries. 
There is no need to deliver module implementations.
It is enough to deliver compiled libraries with:
* signatures, .h- and .c-files, using the Blech compiler or
* signatures, .h-files and target-specific object files, additionally using the target's C compiler.


## Using modules

> When creating a modular system, instead of creating a monolithic application (where the smallest component is the whole), several smaller modules are written separately so when they are composed together, they construct the executable application program. [[1]](#ModularProgramming)

In order to use a module it gets imported by another module or program.
Assume the following use of the `ringbuffer` module from a module implementation in file `slidingaverage.blc`.

```blech
import rb "ringbuffer"

module exposes SlidingAverage 

param Threshold: nat32 = 10000 // Application parameter, can be modified in the binary 

/// Calculates the average of the latest values in every tick
/// Values outside a fixed threshold are ignored.
activity SlidingAverage (value: nat32) (average: nat32)
    var buf: rb.RingBuffer = rb.initialise() 
    repeat
        if value <= Threshold then
            rb.push(value)(buf)
        end
        average = rb.average(buf)
        await true
    end
end
```

This module imports a module from file `ringbuffer.blc` with the local name `rb`.
It implements and exposes a single activity `SlidingAverage`, with a local variable of type `rb.RingBuffer`.
The module completely hides its internal implementation, which uses the abstract type `RingBuffer` and functions `initialise`, `push` and `average` from module `rb`.

The generated signature in file `slidingaverage.blh` is simple. 
The implementations details are completely hidden.

``` blech
signature

/// Calculates the average of the latest values in every tick
activity SlidingAverage (value: nat32) (average: nat32)
```

Using this module in a Blech program file `main.blc` is straight forward.

```blech
import sa "slidingaverage"

@[EntryPoint]
activity SlidingAverage (sensor: nat32) (sensorAverage: nat32)
    run sa.SlidingAverage(sensor)(sensorAverage)    
end
```

## Compiling a program

> Often modules form a directed acyclic graph (DAG); in this case a cyclic dependency between modules is seen as indicating that these should be a single module. In the case where modules do form a DAG they can be arranged as a hierarchy, where the lowest-level modules are independent, depending on no other modules, and higher-level modules depend on lower-level ones. [[1]](#ModularProgramming)

*No dependency cycles between modules* is an important rule for a good system design:
> This design rule will prevent your code from turning into a big ball of mud. 
[...] Dependency cycles are bad because they increase coupling [...] and [...] can’t be tested in isolation. [[2]](#LargePrograms)

In Blech, programs as top-level compilation units and imported modules always form a directed acyclic graph.
This enables automatic dependency management in the compiler.

In the above example, the compilation of program file `main.blc` automatically compiles imported module `slidingaverage` which in turn compiles imported module `ringbuffer`.
Any cycles in the import dependencies will be flagged as an error. 

In general the compilation of any program and any module triggers the compilation of imported modules, if necessary.
A layered and cylce-free module hierarchy is always guaranteed. 

## Testing a module

> A module interface expresses the elements that are provided and required by the module. The elements defined in the interface are detectable by other modules. [[1]](#ModularProgramming)

This an advantage for loosely coupled system design. But it is a disadvantage for white-box testing.

> White-box testing (also known as clear box testing, glass box testing, transparent box testing, and structural testing) is a method of software testing that tests internal structures or workings of an application, as opposed to its functionality (i.e. black-box testing). [[3]](#WhiteBoxTesting)

In order to enable white-box testing, Blech allows to import all the implementation details of a module by using keyword `internal`.
An `internal import` makes all elements in a module detectable - nothing is hidden.

In the following example, it is not enough to use the signature `ringbuffer.blh` to compile the white-box test program.
In fact the compiler needs the source code of the module implementation `ringbuffer.blc` in order to detect the hidden implementation details.

```blech
import n8 "box:base/nat8"  // base library for nat8-related stuff

internal import rb "ringbuffer"

@[EntryPoint]
activity TestPush ()
    var buf: rb.RingBuffer = rb.initialise()
    var i: nat8 = 0
    repeat
        assert buf.nextIndex < rb.Size
        assert buf.nextIndex == i % rb.Size
        assert buf.count >= 0 
        assert buf.count <= rb.Size
        
        rb.push(42)(buf) // the value is irrelevant
        i = i + 1
        
        if i < rb.Size then
            assert buf.count == i
        else
            assert buf.count == rb.Size
        end
        await true
    until i == n8.Max end
end
```

The modules provided by a library cannot be white-box tested, because a packaged library usually does not contain module implementation files.


## Packaging modules into a library

> A particular library is a [...] collection of modules of its own hierarchy, but can in turn be seen as a lower-level module collection [...] to be used by a higher-level program, library, or system. [[1]](#ModularProgramming)

In Blech we call a library a *Blech box* or a *box* for short. 
A box is a collection of modules and programs.
Typically some modules of a box are supposed to be used from those higher levels, while others remain internally hidden and only serve implementation purposes.

By default every module in a box can be imported from a higher-level program or module.
In order to hide a module in a box it can be declared as an `internal module`.

In our running example, we could decide to make the module `ringbuffer` an internal module.
This can be done by declaring the module as `internal module` - the rest remains unchanged.

```blech
internal module exposes initialise, push, average

const Size ...
struct RingBuffer  ...
function initialise () returns RingBuffer ...
function push (value: nat32) (rb: RingBuffer) ...
function average (rb: RingBuffer) returns nat32 ...
```

The signature becomes `internal` as well.

```blech
internal signature

type RingBuffer
function initialise () returns RingBuffer
function push (value: nat32) (rb: RingBuffer)
function average (rb: RingBuffer) returns nat32
```

If we package this module together with module `slidingaverage` from above into a box, 
only the signature for `slidingaverage` should be detectable, the signatures of internal modules are hidden.

In order to write a non-internal module that imports an `internal module`, the details of those imports must not leak through its interface.
For example, the following module `ringbufferaverage` implicitly exposes the opaque type `RingBuffer` from the imported internal module `ringbuffer`.
The internal implementation leaks through its interface and therefore the module becomes itself an `internal module`.

```blech
import rb "ringbuffer"

internal module exposes RingBufferAverage

activity RingBufferAverage (buf: rb.RingBuffer) (average: nat32)
    repeat
        average = rb.average(buf)
        await true
    end
end
```

This means, module `ringbufferaverage` is not accessible outside of its box.
The compiler checks this, and it is an error to omit the the classification `internal` for the module.

The signature of module `ringbufferaverage` looks as follows.

```blech
import rb "ringbuffer"

internal signature

activity RingBufferAverage (buf: rb.RingBuffer) (average: nat32)
```

For pragmatic reasons it might be necessary to circumvent the black-box interface of a module and use a white-box import instead.
This can only be done within the same box, since it requires the source of the module implementation.
As an example, we would like to use `param Threshold` from module `slidingaverage` - which is not exposed - in a new module `slidingaveragewithreset`.

```blech
internal import sa "slidingaverage"

module exposes SlidingAverageWithReset

activity SlidingAverageWithReset (sensor: nat32) (sensorAverage: nat32)
    when sensor > sa.Treshold reset
        run sa.SlidingAverage(sensor)(sensorAverage)
    end
end
```

Since the interface does not leak any implementation details from the white-box import it can be made detectable (non-internal) in the box.

There are two simple rules:
* a module that leaks details of an imported `internal module` in its signature becomes an `internal module`,
* a module that leaks details of an `internal import` in its signature becomes an `internal module`, too.


## Importing inside and outside a box

In order to distinguish between imports inside a box and imports from other boxes, Blech uses an URL-like import path.

Imports inside a box just adress the imported file.

```blech
import m "localmodule"
```

The compiler searches for file `localmodule.blc` to compile it, or uses `localmodule.blh` if nothing has changed and `localmodule.blc` has been compiled before.
Note, that a recompilation of `localmodule.blc` is also necessary, if any of `localmodule`'s imports have changed.

In general, the imported file name for an import inside a box, can be adressed
* relatively to the importing module, or
* absolutely from the top of the box.

Here are some examples for an import inside a box:
```blech
import ma "../module_A"            // relative path, one directory up
import mb "./sub_dir/module_B"     // relative path one directory down
import mb2 "sub_dir/module_B"      // as before with a different local name 
import mc  "/top_level/module_C"   // absolute path inside the box
```

In order to import a module from another box, we need a prefix to address a box, the name of the box and the module inside the box.
The import declaration has the following form

```blech
import m "box:library/module"
```
This tells the compiler to import a module `module` from box `box:library` and name it `m` locally. 
The compiler searches for the signature file `module.blh` in the box named `library`.

When importing from another box, the compiler prevents:
* the import of an `internal module` from the box, and
* the `internal import` of a detectable module from the box.

Such imports are flagged as an error, even if module implementations and `internal signature`s are part of the box.
This is helpful, when developing different boxes at the same time.
The detectability between boxes is the same during development and after deployment.

Since `internal imports` from other boxes are not allowed, there is no need to deliver the module's implementation file `module.blc` with the box `library`.
Since imports of `internal modules` of a box are also forbidden, there is also no need to deliver `internal signature`s.

This brings us to the last question: How are Blech files organized on the file system?

## Organizing Blech files

Blech programs, libraries and systems are organized in boxes.

When developing a box, the program and module implementation files are placed beneath a top-level directory - the Blech project.
The top-level directory can have sub-directories that contain further implementation files.
The file structure corresponds to the import paths used in the import declaration.

Note, that the choice of directory and file names is restricted for simpler name mangling. 
The compiler checks this.

The file structure of the Blech source code translates to the file structure of compilation artefacts: signatures, .h-files and .c-files.
It is possible to work on more than one box at a time, you just need 2 or more Blech projects in independent directories.

## Tricks of trade

As you might have noticed, every imported entity is qualified by the local module name in the examples.
Sometimes you don't want qualification and address an entity directly. 
The `import` declaration optionally `exposes` selected entities for this purpose.

In rare cases, you might want to `expose` everything in a module, or import everything without qualification.

There are shortcuts for these purposes:

1. Make selected entities directly accessible
```blech
import rb "ringbuffer" exposes initialise, push
```

2. Make all entities directly accessible
```blech
import rb "ringbuffer" exposes ...
```

3. Create an import dependency but do not use it right now in the current status of development
```blech
import _ "ringbuffer" // no name for qualified access
```

4. Omit the local module name
```blech
import _ "ringbuffer" exposes RingBuffer, initialise, push
```

5. Expose everything in a module
```blech
module exposes ...  // no information hiding
```

Use these "tricks of trade" wisely, and only if necessary. Keep in mind, Blech implements a rigid no-shadowing strategy.
If two imports expose the same name, the second will create an error because it shadows the first. 

## Modules are not generic

> Modules lie on a spectrum from high-level (specific) to low-level (generic). The highest level module contains the entry point of the program, whereas the lowest level modules are usually generic libraries. [...] Stability increases at lower levels. [...] Reusability increases at lower levels. Low-level modules should be generic libraries so that they can be reused in other projects. [[2]](#LargePrograms)

Of course it is a natural wish, to have the `ringbuffer` module parameterized by its size (here: `const Size`) and its element type (here: `nat32`). 

But, in Blech we keep the module system simple. 
Instead of having generic modules and generating code for every monomorphised instance, we decided to cope with generics on another language level, similar to interfaces or traits in other languages.
Then, modules also might contain generic types and generic interface implementations.

The concept of type-safe generics is in an early stage, and different to other languages follows our embedded requirements.
It will definitely need some more Blech releases before we can adress generics.


## Better software through modules

> [...] Modular designed systems, if built correctly, [are] far more reusable than a traditional monolithic design, since all (or many) of these modules may then be reused (without change) in other projects. This also facilitates the "breaking down" of projects into several smaller projects. Theoretically, a modularized software project will be more easily assembled by large teams, since no team members are creating the whole system, or even need to know about the system as a whole. They can focus just on the assigned smaller task (this, it is claimed, counters the key assumption of The Mythical Man Month, making it actually possible to add more developers to a late software project without making it later still). [[1]](#ModularProgramming)

The Blech module system supports better software design and improved software qualities:

1. It allows to organize code by using different files for different aspects without reverting to the archaic method of `include`d header files. 

1. Modules and programs are the units of separate compilation. All static analysis in the compiler is designed to work with separate compilation. To our knowledge Blech is the first synchronous language to support separate compilation for causality analysis.
 
1. It allows to package compilation units to libraries - boxes of modules and programs.

1. It is designed with a minimal set of syntactic overhead. All import and export information is visible in the head of a module implementation. External visibility is not scattered all over the file.

1. It prevents the pollution of a single global name spaces. Actually there is no global name space that would force the programmer to carefully choose names that are visible everywhere. Modules are namespaces for types and code, boxes are namespaces for modules.

1. There is no need to separate source code into an interface and an implementation part. The source code of module implementations contains all necessary information, interfaces are generated by the compiler.

1. Modules simplify the design of components that are self-contained: independent, and with a single, well-defined purpose. A major enabler to do this are Blech's `activity`s which hold state between time steps locally instead of using global variables. The famous qualities *high cohesion* and *low coupling* are actively supported by the language.

1. Module interfaces support API-orientation. Modules can be developed, tested and even verified in parallel. Implementation changes are easily possible if interfaces are kept small. Especially opaque types enable this. 

1. The compiler enforces a directed acyclic dependency graph. This supports the design of layered architecture without dependency cycles between modules and boxes. Upcalls can be realized with callbacks without introducing cyclic dependencies. Unfortunately, you will have to wait for a later Blech release to support callback references.

1. Because of the directed acyclic graph structure, modules can be "cut off" and tested in isolation. To test a given module you only need all of its "upstream" dependencies.

1. The module dependency graph can easily be visualized and is independent from the code organisation on the file system.

1. The ability to white-box test a given module allows to separate test code from the module implementation although a module might have an interface that hides many of the implementation details. In many languages white-box testing requires reflection which is not appropriate for embedded code. To our knowledge this feature is unique to Blech.


We hope to release Blech with modules before the end of the year. Stay tuned.

## References

[1] <a name="ModularProgramming">[Modular Programming](https://en.wikipedia.org/wiki/Modular_programming), en.wikipedia.org</a>

[2] <a name=LargePrograms> [How to write large programs](https://medium.com/@olegalexander/how-to-write-large-programs-628c90a70615), Oleg Alexander </a>

[3] <a name=WhiteBoxTesting> [White-box Testing](https://en.wikipedia.org/wiki/White-box_testing), en.wikipedia.org </a>



<!-- # Relevant links

[C++ Modules Might Be Dead-on-Arrival](https://vector-of-bool.github.io/2019/01/27/modules-doa.html)
[C++20: The advantages of modules](https://www.modernescpp.com/index.php/cpp20-modules)
[Low Coupling, High Cohesion](https://medium.com/clarityhub/low-coupling-high-cohesion-3610e35ac4a6)
[How To Write Large Programs](https://medium.com/@olegalexander/how-to-write-large-programs-628c90a70615)
# Steinbruch


> Often modules form a directed acyclic graph (DAG); in this case a cyclic dependency between modules is seen as indicating that these should be a single module. In the case where modules do form a DAG they can be arranged as a hierarchy, where the lowest-level modules are independent, depending on no other modules, and higher-level modules depend on lower-level ones. A particular program or library is a top-level module of its own hierarchy, but can in turn be seen as a lower-level module of a higher-level program, library, or system. 

```blech
module exposes initialise, push, average

const Size: nat8 = 10

struct RingBuffer
    var buffer: [Size]nat32
    var nextIndex: nat8
    var count: nat8
end

function initialise () returns RingBuffer
    return { nextIndex = 0, count = 0 }
end

function push (value: nat32) (rb: RingBuffer)
    rb.buffer[rb.nextIndex] = value
    rb.nextIndex = rb.nextIndex + 1
    if rb.count = Size then // ringbuffer ist completely filled
        rb.nextIndex = rb.nextIndex % Size
    else
        rb.count = rb.count + 1
    end
end

function average (rb: RingBuffer) returns nat32
    var idx: nat8 = 0
    var avg: nat32 = 0
    while idx < rb.count do
        avg = avg + rb.buffer[idx]
    end
    return avg / rb.count
end
```

```blech
signature

type RingBuffer

function initialise () returns RingBuffer
function push (value: nat32) (rb: RingBuffer)
function average (rb: RingBuffer) returns nat32
```


```blech
import rb "ringbuffer"

module exposes SlidingAverage 

activity SlidingAverage (value: nat32) (average: nat32)
    var buf: rb.RingBuffer = rb.initialise() 
    repeat
        rb.push(value)(buf)
        average = rb.average(buf)
        await true
    end
end
```

``` blech
signature

activity SlidingAverage (value: nat32) (average: nat32)
```

```blech
import sa "slidingaverage"

@[EntryPoint]
activity SlidingAverage (sensor: nat32) (sensorAverage: nat32)
    run sa.SlidingAverage(sensor)(sensorAverage)    
end
```


Bad design.

``` blech
import rb "ringbuffer"

module exposes SlidingAverage 

activity SlidingAverage (value: nat32) 
                        (ringbuffer: rb.RingBuffer, average: nat32)
    repeat
        rb.push(value)(ringBuffer)
        average = rb.average(ringBuffer)
        await true
    end
end
```

``` blech
import rb "ringbuffer"

signature

activity SlidingAverage (value: nat32) 
                        (ringbuffer: rb.RingBuffer, average: nat32)
```

```blech
import r "ringbuffer"
import s "slidingaverage"

@[EntryPoint]
activity SlidingAverage (sensor: nat32) (sensorAverage: nat32)
    var ringBuffer: r.RingBuffer = r.initialise()
    run s.SlidingAverage(sensor)(ringBuffer, sensorAverage)    
end
```



### Using internal import to access internal details.

A module that uses an `internal import` becomes itself an `internal module` if it leaks details of the `internal import` in its interface. 
Assume the following module in file `countobserver.blc`-
``` blech
internal import rb "ringbuffer"

internal module exposes MaxCount, count

const MaxCount: nat8 = rb.Size

function count (buf: rb.RingBuffer) returns nat8
    return buf.count
end
```

It is a compiler error not to classify the module as `internal`.

It is also not possible to create a module interface that is independent of the module implementation file  `ringbuffer.blc`.
Compilation completely relies on implementation files.
Therefore we never create the following a signature file.

``` blech
internal import rb "ringbuffer"

signature
const MaxCount: nat8 = rb.Size
function count (buf: rb.RingBuffer) returns nat8
```

On the other hand, we can classify a module as `internal` to prevent exposing it in a package.
This means internal modules are hidden inside a package, because they do not generate a signature.
An `internal import` or the import of an `internal module` always needs the source code of the module implementation file.

In order to create a signature for a module that uses an `internal import` or imports an `internal module`, the details of those imports must not leak through its interface.

``` blech
import rb "ringbuffer"

module exposes ObserveFilledBuffer

const MaxCount: nat8 = rb.Size

function count (buf: rb.RingBuffer) returns nat8
    return buf.count
end

activity ObserveFilledBuffer (value: nat32) (filled: bool)
    var buf: rb.RingBuffer = rb.initialise()
    repeat
        rb.push(value)(buf)
        filled = count(buf) == MaxCount
        await true
    end
end
```

``` blech
signature
activity ObserveFilledBuffer (value: nat32) (filled: bool)
```

 -->