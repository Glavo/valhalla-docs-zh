# State of Valhalla

#### 第一部分：通往 Valhalla 的道路
#### Brian Goetz, Apr 2021

::: sidebar
目录：

1. [通往 Valhalla 的道路](01-background.html)
2. [统一 Java 类型系统](02-object-model.html)
3. [JVM 模型](03-vm-model.html)

:::

## 背景

[_Valhalla 项目_][valhalla]于 2014 年启动，目标是为基于 JVM 的语言带来更灵活的扁平化数据类型，以便恢复编程模型与现代硬件性能特征之间的一致性.（某些方面来说，它开始的更早；Java 的设计者希望在语言的初始版本中包含值类型。）最初，Valhalla 是根据我们为实现这一目标而预期增加的功能来描述的：*原始类（primitive class）*（在迭代过程中曾被称为内联类和值类型）和*特化泛型（specialized generic）*。在项目的初始阶段，我们主要关注于理解语言和 JVM 如何发展以支持这些特性，以及迁移兼容性对用户代码的影响。

虽然可以逐渐发布这些特性，但是在提交实现之前，必须对它们如何协同工作有一个一致的设想。我们现在相信，我们有一条连贯的途径通过原始类来增强 Java 语言和虚拟机，将现有的原始类型以及基于值的类（value-based class）迁移至原始类，让原始类与基于擦除实现的泛型之间进行干净的互操作，并将现有的泛型类迁移至特化泛型。这组文档总结了该路径，将会分阶段交付。其中前两个在在 [JEP 401][jep401] 和 [JEP 402][jep402] 中所描述（这两个 JEP 的中文翻译请参见 [JEP 401][jep401-zh] 和 [JEP 402][jep402-zh]）。（如果您想与我们开始时的设计相比较，请参见我们的[初始文档][values0]。）

但是，Valhalla 项目不仅仅是关于这些它将提供的特性，或者关于改进性能；它有更具雄心的 agendum 来*统一 Java 类型系统* —— 统一原始类型与类，并且允许泛型处理任何类型。

#### The costs of indirection

The JVM type system includes primitive types (`int`, `long`, etc.), classes
(heterogeneous aggregates with identity), and arrays (homogeneous aggregates
with identity).  This set of building blocks is flexible -- you can model any
data structure you need to.  Data that does not fit neatly into the available
primitive types (such as complex numbers, 3D points, tuples, decimal values,
strings, etc.), can be easily modeled with objects.  However, objects are
allocated in the heap (unless the VM can prove they are sufficiently narrowly
scoped and unaliased), require an object header (generally two machine words),
and must be referred to via a memory indirection.  For example, an array of XY
point objects has the following memory layout:

![Layout of XY points](xy-points.png){ width=100% }

When the Java Virtual Machine was being designed in the early 1990s, the cost of
a memory fetch was comparable in magnitude to computational operations such as
addition.  With the multi-level memory caches and instruction-level parallelism
of today's CPUs, a single cache miss may cost as much as 1000 arithmetic issue
slots -- a huge increase in relative cost.  As a result, the pointer-rich
representation favored by the JVM, which involves many indirections between
small islands of data, is no longer an ideal match for today's hardware.  We aim
to give developers the control to match data layouts with the performance model
of today's hardware, providing Java developers with an easier path to _flat_
(cache-efficient) and _dense_ (memory-efficient) data layouts without
compromising abstraction or type safety.  

What we would like is to have the option to get a layout more like this:

![Flattened layout of XY points](flattened-points.png){ width=60% }

This layout is both flatter (no indirections) and denser (no headers) than the
previous version.  (Related to object layout is _calling convention_, which is
how the JVM passes values from one method to another.  In the absence of heroic
JVM optimizations, when a method passes a `Point` to another, it passes a
pointer, and then the callee must dereference the pointer to access the object's
state.  Enabling a flatter representation is also likely to enable a flatter
calling convention, where a `Point` can be passed by passing the `x` and `y`
components by value, in registers.)

One of the key questions that Project Valhalla addresses itself to is: _what
code would we want to write to get this layout?_

To understand our options, we must first understand where the current
unfortunate layout comes from.  The root cause here is _object identity_: all
object instances today must have an object identity.  (In the early 90s,
"everything is an Object" was an attractive mantra, and the performance costs of
identity were not onerous.)  Identity enables mutability; in order to mutate a
field of an object, we must know _which_ object we are trying to modify.  And
even for the many classes that eschew mutability, identity can
be observed by various identity-sensitive operations, including object
equality (`==`), synchronization, `System::identityHashCode`, weak references,
etc.  The result is that the VM frequently must pessimistically preserve
identity just in case someone might eventually perform an identity-sensitive
operation -- even if that is never going to happen.  Thus, identity leads to the
pointer-rich memory layout we have today.

Valhalla enables some classes to disavow identity.  Instances of these classes
are just their state, and therefore can be routinely flattened into enclosing
objects or arrays, and passed by value between methods.  The trade-off is that
we have to give up some flexibility -- they must be immutable and cannot be
layout-polymorphic -- but in return for giving up these characteristics, we are
be rewarded with a flatter and denser memory layout and optimized calling
conventions.  The terminology for these classes has changed during the course of
the project; while they were originally called _value types_ and then _inline
classes_,  such classes are now called _primitive classes_.  This naming
highlights the fact that they are classes with the runtime behavior of
primitives.  The slogan for Valhalla is:

> Codes like a class, works like an int.

Despite the restrictions on mutability and subclassing, primitive classes can
use most mechanisms available to classes: methods, constructors, fields,
encapsulation, supertypes, type variables, annotations, etc.

There are applications for primitive classes at every level.  Aside from the
obvious -- turning the built-in primitive types into real classes -- many API
abstractions, such as numerics, dates, cursors, and wrappers like `Optional`,
can be naturally modeled as identity-free classes.  Additionally, many data
structures, such as `HashMap`, can profitably use primitive classes in their
implementations to improve efficiency.  And language compilers can use them as a
compilation target for features like built-in numeric types, tuples, or multiple
return.

## What about generics?

One of the early compromises of Java Generics is that generic type variables can
only be instantiated with reference types, not primitive types.  This is both
inconvenient (we have to say `List<Integer>` when we mean `List<int>`) and
expensive (boxing has performance overhead.)  With eight primitive types,
this restriction is something we learned to live with, but if we can write our
own flattenable data types like our `Point` above, having an `ArrayList<Point>`
not be backed by a flattened array of `Point` seems to defeat, well, the point.

Parametric polymorphism always entails tradeoffs between code footprint,
abstraction, and specificity, and different languages have chosen different
tradeoffs.  At one end of the spectrum, C++ creates a specialized class for each
instantiation of a template, and different specializations have no type-system
relationship with each other. Such a _heterogeneous translation_ provides a high
degree of specificity, to the point where expressions such as `a+b` can be
interpreted relative to the behavior of `+` on the instantiated types of `a` and
`b`, but entails a large code footprint as well as a loss of abstraction --
there is no type that is the equivalent of `Foo<?>` in Java.

At the other end of the spectrum, we have Java's current erased implementation
which produces one class for all reference instantiations and no support for
primitive instantiations.  Such a _homogeneous translation_ yields a high degree
of reuse, since there is only one class and one object layout for all
(reference) instantiations.  It carries the restriction that we can only range over types
that have a common runtime representation, which in Java is the set of reference
types.  This restriction has its roots deep in the design of the JVM; there
are different bytecodes for operations on reference vs primitive values.

While most developers have a certain degree of distaste for erasure, this
approach has a powerful advantage that we could not have gotten any other way:
_gradual migration compatibility_.  This is the ability to compatibly evolve a
class from non-generic to generic, without breaking existing sources or binary
class files, and leaving clients and subclasses with the flexibility to migrate
immediately, later, or never.  Offering users generics, but at the cost of
throwing away all their libraries, would have been a bad trade in 2004, when
Java already had a large and vibrant installed base -- and would be a worse
trade today.  (See [In defense of erasure](erasure) for more detail on the
forces that led to the current design of Java generics.)

Our goal today is even more ambitious than it was in 2004: to extend generics so
that we can instantiate them over primitive classes with specialized
(heterogeneous) layouts, while retaining gradual migration capability.
Further, migration compatibility in the abstract is not enough; we want to
actually migrate the libraries we have, including Collections and Streams.

This goal will be achieved through a combination of i) allowing primitive class
instances to be treated as lightweight Objects; ii) supporting interoperation of
primitive class types with existing erasure-based generics; and iii) developing
optimized heterogeneous _species_ in the JVM for cases in which enough type
information is available at run time.

## A brief history

Project Valhalla has ambitious goals, and its intrusion is both deep and broad,
affecting the `class` file format, JVM, language, libraries, and user model.  In
2014, James Gosling described it as "six Ph.D theses, knotted together." Since
then, we've built six distinct prototypes, each aimed at a understanding a
separate aspect of the problem.

The first three prototypes explored the challenges of generics specialized
directly to primitives, and operated by bytecode rewriting.  The first ("Model
1") was primarily aimed at the mechanics of specialization, identifying what
type behavior needed to be retained by the compiler and acted on by the
specializer.  The second ("Model 2") explored how we might represent wildcards
(and by extension, bridge methods) in a specializable generic model, and started
to look at the challenges of migrating existing libraries.  The third ("Model
3") consolidated what we'd learned, building a sensible classfile format that
could be used to represent specialized generics.  (For a brief tour of some of
the challenges and lessons learned from each of these experiments, see [this
talk][adventures] ([slides here][adventures-slides]).

The results of these experiments were a mixed bag.  On the one hand, it worked
-- it was possible to write specializable generic classes and run them on an
only-slightly-modified JVM.  On the other hand, roadblocks abounded; existing
classes were full of assumptions that were going to make them hard to migrate,
and there were a number of issues for which we did not yet have a good answer.

We then worked the problem from the other direction, with the "Minimal Value
Types" prototype, whose goal was to prove that we could implement flat and dense
layouts in the VM.  The turning point came with the prototype known as "L World"
(so called because it lets primitive classes share the `L` carrier with object
references).  In the earliest explorations, we assumed a VM model where
primitive classes were more like today's primitives -- with separate type
descriptors, bytecodes and top types -- in part because it seemed too daunting
at the time to unify references and primitives under one set of type
descriptors, bytecodes, and types.  L-world gave us this unification, which
addressed a significant number of the challenges we encountered in the early
rounds of prototypes, and leading to a true unification of primitives with
classes.  (The mantra for this part of the work could be described as "`Object`
is the new `Any`.")

## Moving forward

We intend to divide delivery of Project Valhalla into two broad phases:
primitive classes first, followed by specialized generics.  (These may be
further divided into delivery milestones.)  The first phase will focus on
support for primitive classes in the Java language and virtual machine, and
migrating the existing primitive types to primitive classes.  This phase will
also lay the groundwork for the use of primitive classes in the JDK, and even
migrating some existing value-based classes (such as `Optional` or
`LocalDateTime`), and is described by JEPs [401](jep401) and [402](jep402).

The second phase will focus on generics, extending the generic type system to
support instantiation with primitive classes and extending the JVM to support
specialized layouts.


[valhalla]: http://openjdk.java.net/projects/valhalla
[values0]: http://cr.openjdk.java.net/~jrose/values/values-0.html
[adventures]: https://www.youtube.com/watch?v=TkpcuL1t1lY
[adventures-slides]: http://cr.openjdk.java.net/~briangoetz/valhalla/Adventures%20in%20Parametric%20Polymorphism.pdf
[model3]: http://cr.openjdk.java.net/~briangoetz/valhalla/eg-attachments/model3-01.html
[jep401]: https://openjdk.java.net/jeps/401
[jep402]: https://openjdk.java.net/jeps/402
[jep401-zh]: https://glavo.site/translate/2021/03/06/primitive-objects/
[jep402-zh]: https://glavo.site/translate/2021/03/06/unify-the-basic-primitives-with-objects/
[erasure]: ../in-defense-of-erasure.md
