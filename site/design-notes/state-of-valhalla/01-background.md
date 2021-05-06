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

#### 间接成本

JVM 类型系统包括原始类型（`int`、`long` 等）、类（具有 identity 的异构聚合）和数组（具有 identity 的同构聚合）。这组构造块非常灵活 —— 您可以对任何所需的数据结构进行建模。
不适用可用的原始类型的数据（例如复数、三维点、元组、定点数、字符串等）可以很容易地用对象进行建模。但是，对象是在堆中进行分配的（除非 VM 可以证明它们的作用域足够窄并且 unaliased），需要一个对象头（通常为两个机器 word），并且需要通过内存间接引用来引用。例如，XY 点对象的数组具有以下内存布局：

![XY 点对象数组的布局](xy-points.png){ width=100% }

当 Java 虚拟机在 20 世纪 90 年代初被设计的适合，从内存取回数据的开销与加法等计算操作相当。依赖于当今 CPU 的多级高速缓存以及指令并行性，一次缓存 miss 可能会导致高达相当于上千次计算操作的开销 —— 相对成本大大增加了。因此，JVM 偏好的常用指针的表现形式（涉及数据块之间的很多间接操作）不再是当今硬件上的理想选择。我们目的是给开发人员提供控制权，使得数据布局与现在的硬件性能模型相匹配，从而为开发人员提供简单的方式来实现*平坦*（缓存友好）和*密集*（内存占用友好）的数据布局，又不损失抽象能力和类型安全性。

我们想要的是可以选择这样的布局：

![XY 点对象数组的平面布局](flattened-points.png){ width=60% }

与之前的版本相比，此布局既平坦（无间接寻址），也更密集（没有对象头）。（与对象布局相关的是*调用协定（calling convention）*，表示 JVM 中如何将值从一个方法传递给另一个方法。在缺乏强大的 JVM 优化的情况下，当一个方法把一个 `Point` 传递给另一个方法时，它会传递一个指针，然后被调用方需要解引用该指针以访问该对象的状态。允许更平坦的表示形式也可能会启用更平坦的调用协定，可以通过在寄存器内按值传递组件 `x` 和 `y` 来传递 `Point`。）

Valhalla 项目现在要解决的一个关键问题是：*我们想要编写什么样的代码来获得这种布局？*

想要了解我们的选择，我们需要先了解目前这个不恰当的布局从何而来。根本原因在于 *object identity*：现在的所有对象实例都需要具有 object identity。（在 90 年代初，“一切皆对象”是一句很有吸引力的口头禅，而 identity 的性能开销也并不繁重。）Identity 使得可变性称为可能；为了修改对象的字段，我们必须知道我们修改的是*哪个*对象。即使对于很多避免了可变性的类，也可以通过各种对 identity 敏感的操作来观察到它，包括对象相等性（`==`）、同步、`System::identityHashCode`、弱引用等。结果是，VM 经常必须悲观地保留 identity，以防有人执行 identity 敏感的操作 —— 即使这种情况永远不会发生。因此，是 identity 导致了我们现在指针泛滥的内存布局。

Valhalla 允许一些类 disavow identity。这些类的实例只是它们的状态，因此可以常规地在包含它的对象或数组中展开，并按值在方法间传递。取舍是我们必须放弃一些灵活性 —— 它们必须是不可变的，不能是布局多态的 —— 但作为放弃这些特性的回报，我们可以获得更平坦、更密集的内存布局以及优化的调用协定。在项目进行过程中，这些类的术语发生了变化；它们最初被称为*值类型（value type）*，后来又被称为*内联类（inline class）*，而现在它们被称为*原始类（primitive class）*。这个命名突出了它们是运行时具有原始类型行为的类。Valhalla 的口号是：

> Codes like a class, works like an int.

尽管对可变性和子类方面进行了限制，但原始类依然可以使用适用于类的大多数机制：方法、构造器、字段、封装、超类型、泛型参数、注解等。

每个级别都有原始类的应用。除了显而易见的 —— 将内置的基本类型转为真实的类之外 —— 许多 API 抽象，例如数字、日期、cursor 以及类似 `Optional` 的包装器等也可以自然地建模为 identity-free 类。此外，很多数据结构（例如 `HashMap`）都可以在其实现中使用原始类来提高运行效率。语言编译器也可以将它们用作数值类型、元组或多重返回等功能的编译目标。

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
