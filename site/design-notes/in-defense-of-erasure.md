# 背景：我们是如何得到现在的泛型的
#### （或者，我们如何学会不再担心，拥抱擦除）
#### Brian Goetz, June 2020

在我们讨论泛型的发展方向之前，我们首先要讨论它们现在的样子，以及它们是如何发展到现在这样的。 This document will focus primarily on how we arrived at the generics we have now, and why, as a means of setting the groundwork for how the generics we have now will influence the "better" generics we are trying to build.

我们在这里特别强调，擦除实际上是在 2004 年向 Java 添加泛型明智且务实的选择 —— 而导致我们选择使用擦除进行翻译的很多力量可能至今仍在发挥作用。

## 擦除

向任何开发者询问有关 Java 泛型的信息，您都可能会对*擦除（erasure）*感到愤怒（though often uninformed）。擦除可能是 Java 中最广泛、最深的被误解的概念。

擦除既不是特定于 Java 的，也不是特定于泛型的；它是一种普遍存在的，而且经常是必要的工具，用于将代码从一个层次转换到更低层次（例如把 Java 源代码编译到字节码，或者把 C 源代码编译为本机代码时）。这是因为随着我们从高级语言向下到中间表示，再到本机代码，再到硬件时，较低层次提供的类型抽象几乎总是比较高层次提供的类型抽象更简单和更弱 —— 的确如此。（我们不想把虚拟分派的语义引入 X86 指令集中，也不希望在寄存器里模拟 Java 的基本类型集。）擦除是一种将一个较高层次的 richer 类型映射到较低层次的 less rich 类型（理想情况下是在较高层次上执行了完备的类型检查后）的技术，也是编译器每天都要做的事情。

例如，Java 字节码集包含用于在栈和局部变量集之间移动 `int` 值（`iload`、`istore`），以及对 `int` 执行算数运算（例如 `iadd`、`imul` 等）的指令。对于 `float`（例如 `fload`、`fstore`、`fmul` 等），`long`（`lload`、`lstore`、`lmul`），`double`（`dload`、`dstore`、`dmul`）以及对象引用（`aload`、`astore`）也有类似的指令。但是没有针对 `byte`、`short`、`char` 以及 `boolean` 的此类指令 —— 因为编译器会把这些类型擦除为 `int`，并使用 `int` 的移动和算数指令。在字节码设计中，这是一种实用的设计取舍；它降低了指令集的复杂度，从而提高运行时的效率。Java 语言的很多其他特性（例如受检异常、方法重载、枚举、确定性赋值分析、嵌套类、通过 lambda 或 局部类捕获局部变量等）都是“语言虚构”的（language fictions） ） —— 它们会在 Java 编译器被检查，但翻译为类文件时会被擦除。

类似地，将 C 编译为本机代码时，有符号和无符号的 int 都会被擦除到通用寄存器中（没有单独的有符号或无符号寄存器），并且 `const` 变量也会被存储到可变的寄存器与内存位置中。我们完全不认为这种擦除很奇怪。

#### 同构翻译 vs 异构翻译

在具有参数化多态的语言中，翻译泛型类型又两种常见方法 —— *同构（homogeneous）*与*异构（heterogeneous）*翻译。在同构翻译中，泛型类 `Foo<T>` 会被翻译为单一 artifact，类似 `Foo.class`（泛型方法也是如此）。在异构翻译中，泛型类型以及方法（`Foo<String>`、`Foo<Integer>`）的每个实例均被视为独立的实体，并生成处理的 artifact。例如，`C++` 使用了异构翻译：模板的不同实例是完全不同的类型，具有不同的语义，会生成不同的代码。类型 `vector<int>` 和 `vector<float>` 都是独立的类型。一方面，这对类型安全性（在展开后可以分别对每个实例进行类型检查）以及生成的代码的质量（因为可以对每个实例分别进行优化）非常有效。另一方面，这意味着代码会有更大的占用（因为 `vector<int>` 与 `vector<float>` 有单独的代码），而且我们不能谈论“某个类型的 vector”（类似 Java 泛型通配符的功能），因为每个实例都是完全不相关的类型。（作为代码膨胀的空间成本的一个较为极端的展示，Scala 尝试提供了一个 `@specialized` 注解，它会让编译器为所有原始类型生成特化的版本。这听起来很酷，但这会让生成的类激增 $9^n$ 倍，其中 $n$ 是类中被特化的类型变量的数量，因此可以很轻松的从几行代码生成一个 100MB 大的 JAR 文件。）

同构翻译与异构翻译之间的抉择涉及到语言的设计者一直在做的各种权衡。异构翻译提供了更多的类型特化性，代价是更大的静态和动态空间占用，以及更少的运行时共享 —— 这些都会影响性能。同构翻译更适合抽象参数化类型族，譬如 Java 的通配符和 C# 的声明处类型变异（这两者都是 `C++` 缺少的，`vector<int>` 和 `vector<float>` 之间没有任何共同之处）。有关翻译策略的更多信息，请参见[*这篇 influential paper*](http://pizzacompiler.sourceforge.net/doc/pizza-translation.pdf)。

#### Java 中的擦除泛型

Java 使用同构翻译来翻译泛型。泛型在编译时进行类型检查，但生成字节码时，类似 `List<String>` 的泛型会被擦除到 `List`，而像 `<T extends Object>` 这样的类型变量会被擦除到它们的边界（本例中为 `Object`）。

如果我们有：

```
class Box<T> {
    private T t;

    public Box(T t) { this.t = t; }

    public Box<T> copy() { return new Box<>(t); }

    public T t() { return t; }
}
```

`javac` 编译器会生成一个类文件 `Box.class`，它是 `Box` 所有实例化的实现 —— 包括通配符（`Box<?>`）和 raw type（`Box`）。字段、方法和超类型描述符都被擦除；类型变量被擦除至其边界，泛型类型被擦除到其首部（`List<String>` 被擦除到 `List`），擦除后的结果如下所示：

```
class Box {
    private Object t;

    public Box(Object t) { this.t = t; }

    public Box copy() { return new Box(t); }

    public Object t() { return t; }
}
```

泛型签名被保留（在属性 `Signature` 中），以便编译器在读取类文件时可以看到泛型签名，但 JVM 中在链接时仅使用擦除后的描述符。这种转换方案意味着在类文件级别上，`Box<T>` 的*布局*和 *API* 都会被擦除。在使用处会发生相同的事情：对 `Box<String>` 的因被擦除为 `Box`，并在使用点插入了到 `String` 的强制转换。

#### 为什么？有哪些其他选择？

正是在这一点上，我们很想发出嘘声，并宣称这些选择显然是愚蠢的和懒惰的，或者说擦除是一种肮脏的手段。毕竟，为什么编译器要丢弃完整的类型信息？

为了更好的理解这个问题，我们还应该问：如果我们要具化这种类型信息，我们希望用它们做什么？以及相关的成本是什么？我们可以设想使用具化类型参数信息的几个不同方式：

  - **反射。**对于某些人来说，“具化泛型”仅仅意味着可以查询一个列表是什么样的列表，无论是用 `instanceof` 这样的语言工具或对类型变量进行模式匹配，还是使用反射库查询类型参数。
  - **布局或 API 特化。**在具有原始类型或内联类的语言中，最好能将 `Pair<int, int>` 的布局展平以容纳两个 `int`，而不是存储两个对装箱对象的引用。
  - **运行时类型检查。**当用户尝试将 `Integer` 放入 `List<String>` 中时（例如通过 `List` 这样的 raw 引用），这会导致堆污染，最好能够捕获这一点并在导致堆污染的地方失败，而不是（可能）在其后某个地方合成的强制转换处检测它。

这三种可能性（反射、特化与类型检查）虽然不是互斥的，但他们针对的是不同的目标（分别为对程序员的便捷性、性能和安全性）—— 并有不同的含义和成本。虽然说“我们想要具化”很容易，但如果我们进行更深入的研究，我们会发现其中重要的部分、相对成本和收益方面存在重大分歧。

为了了解为什么擦除是明智和务实的选择，我们还必须了解当时的目标、优先事项、限制与解决方案。

#### 目标：逐步迁移的兼容性

Java 泛型采用了一个具有雄心的目标：

> 必须能够以二进制兼容并且源代码兼容的方式将现有的非泛型类演变为泛型类。

以 `ArrayList` 为例，这意味着现有的用户和子类可以继续编译，而无需针对泛型化的 `ArrayList<T>` 进行修改，并且现有的类文件能够继续链接至泛型化的 `ArrayList<T>` 的方法。支持这一点意味着泛型类的用户和子类可以选择立即、稍后还是不进行泛型化，并且可以独立于其他用户或子类的维护者选择怎么做。

如果没有这个要求，泛型化一个类需要一个“flag day”，其中所有用户和子类不修改也需要至少进行一次重新编译 —— 所有事情要一次性完成。对于 `ArrayList` 这样的核心类来说，这本质上是要求立即重新编译世界上所有的 Java 代码（或者永远留在 Java 1.4 上）。这样跨越整个 Java 生态的“flag day”是不可能的，所以我们需要一个泛型类型系统，它允许核心平台类（以及流行的第三方库）能够进行泛型化，而不需要用户了解它们的泛型。（更糟糕的是，世界上所有的java代码不能一次性完成泛型化，所以这将不是一个“flag day”，而是很多个）。

这种兼容性的问题用另一种话来说： 我们不能接受去孤立那些可以被泛型化的代码，或者让开发者在泛型化与他们已经投入开发的现有代码中作取舍。我们希望的是，使泛型化成为一个具有兼容性的操作，使得开发者可以保留对现有代码开发的投入的有效性的同时，选择是否泛型化。

这种对”flag day”的厌恶来源于java设计的本质：独立编译，动态链接。独立编译意味着每个源文件将编译成一个或多个类文件，而不是多个源文件编译为一个统合件。动态链接意味着在运行时，多个类文件的引用将是基于符号信息链接在一起的；如果类 `C` 调用D中的方法`void m(int x)`，那么`C`的类文件中将会记录调用方法的方法名和描述符，并在链接时在`D`中查找刚才记录的方法名和描述符，如果两项查找都符合的话，那么调用方就被链接上了。
这听起来很花功夫，但是独立编译，动态链接会带给java一个无与伦比的优势 —— 你可以在类路径下编译`C`和运行`C`时用不同版本的`D`文件（只要你不对`D`进行任何二进制不兼容的更改）。

> The pervasive commitment to dynamic linkage is what allows us to simply drop
a new JAR on the class path to update to a new version of a dependency, without
having to recompile anything.  We do this so often we don't even notice -- but
if this stopped working, it would indeed be noticed.

At the time generics were introduced into Java, there was already a lot of Java
code in the world, and their classfiles were full of references to APIs like
`java.util.ArrayList`.  If we couldn't generify these APIs compatibly, then we
would have to have written _new_ APIs to replace them, and worse, all of the
client code of the old APIs would be stuck with an untenable choice -- either
stay on 1.4 forever, or rewrite them to use the new APIs, simultaneously
(including not only the application code, but all third-party libraries on which
the application depends.)  This would have devalued almost all the Java code in
existence at the time.  

C# made the opposite choice -- to update their VM, and invalidate their existing
libraries and all the user code that dependend on it.  They could do this at the
time because there was comparatively little C# code in the world; Java didn't
have this option at the time.

One consequence of this choice, though, is that it will be an expected
occurrence that a generic class will simultaneously have both generic and
non-generic clients or subclasses.  This is a boon to the software development
process, but it has potential consequences for type safety under such mixed
usage.

#### Heap pollution

Erasing in this manner, and supporting interoperability between generic and
non-generic clients, creates the possibility of _heap pollution_ -- that what is
stored in the box has a runtime type that is not compatible with the
compile-time type that was expected.  When a client uses a `Box<String>`, casts
are inserted whenever a `T` would be assigned to a `String`, so that heap
pollution is detected at the point where data transitions from the world of type
variables (the implementation of `Box`) to the world of concrete types.  In the
presence of heap pollution, these casts can fail.

Heap pollution can come from when non-generic code uses generic classes, or when
we use unchecked casts or raw types to forge a reference to a variable of the
wrong generic type.  (When we used unchecked casts or raw types, the compiler
warns us that heap pollution might result.)  For example:

```
Box<String> bs = new Box<>("hi!");   // safe
Box<?> bq = bs;                      // safe, via subtyping
Box<Integer> bi = (Box<Integer>) bq; // unchecked cast -- warning issued
Integer i = bi.get();                // ClassCastException in synthetic cast to Integer
```

The sin in this code is the unchecked cast from `Box<?>` to `Box<Integer>`;  we
have to take the developer at their word that the specified `box` really is a
`Box<Integer>`.  But the heap pollution is not caught right away; only when we
try to _use_ the `String` that was in the box as an `Integer`, do we detect that
something went wrong.  Under the translation we have, if we cast our box to
`Box<Integer>` and back to `Box<String>` before we used it as a `Box<String>`,
nothing bad happens (for better or worse.)  Under a heterogeneous translation,
`Box<String>` and `Box<Integer>` would have different runtime types, and this
cast would fail.

The language actually provides quite a strong safety guarantee for generics, as
long as we follow the rules:

> If a program compiles with no unchecked or raw warnings, the synthetic casts
inserted by the compiler will never fail.

In other words, _heap pollution can only occur when we are interoperating with
non-generic code_ or when we _lie to the compiler_.  At the point where the heap
pollution is discovered, we get a clean exception telling us what type was
expected and what type was actually found.

#### Context: Ecosystem of JVM implementations and languages

The design choices surrounding generics were also influenced by the structure of
the ecosystem of JVM implementations and of languages running on the JVM.  While
to most developers "Java" is a monolithic entity, in fact the Java Language and
the Java Virtual Machine (JVM) are separate entities, each with their own
specification.  The Java compiler produces classfiles for the JVM (whose format
and semantics are laid out in the Java Virtual Machine Specification), but the
JVM will happy run any valid classfile, regardless of what source language it
originally came from.  By some counts, there are over 200 languages that use the
JVM as compilation target, some of which have a lot in common with the Java
language (e.g., Scala, Kotlin) and others which are very different languages
(e.g., JRuby, Jython, Jaskell.)

One reason the JVM has been so successful as a compilation target, even for
languages quite different from Java, is that it provides a fairly abstract model
for computing, with limited influence from the Java language.  The abstraction
layer between the language and virtual machine was not only useful for
stimulating an ecosystem of other languages that run on the JVM,  but also an
ecosystem of _independent implementations_ of the JVM.  While the market today
has consolidated substantially, at the time that generics were added to Java,
there were over a dozen commercially viable implementations of the JVM.
Reifying generics would mean that not only would we need to enhance the language
to support generics, but also the JVM.

While it might have been technically possible to add generic support to the JVM
at the time, not only would it have been a significant engineering investment
requiring substantial coordination and agreement between the many implementors,
the ecosystem of languages on the JVM might also have had an opinion about
reified generics.  If, for example, the interpretation of reification included
type checking at runtime, would Scala (with its declaration-site variance) be
happy to have the JVM enforce Java's (invariant) generic subtyping rules?

#### Erasure was the pragmatic compromise

Taken together, these constraints (both technical and ecosystem) acted as a
powerful force to push us towards a homogeneous translation strategy where
generic type information is erased at compile time.  To summarize, forces that
pushed us towards this decision include:

  - **Runtime costs.**  A heterogeneous translation entails all sorts of runtime
    costs: greater static and dynamic footprint, greater class-loading costs,
    greater JIT costs and code cache pressure, etc.  This might have put
    developers in a position where they had to choose between type-safety and
    performance.
  - **Migration compatibility.**  There was no known translation scheme at the
    time that would have allowed a migration to reified generics to be source-
    and binary-compatible, creating flag days and invalidating developer's
    considerable investment in their existing code.
  - **Runtime costs, bonus edition.**  If reification is interpreted as
    _checking_ types at runtime (just as stores into Java's covariant arrays are
    dynamically checked), this would have a significant runtime impact, as the
    JVM would have to perform generic subtyping checks at runtime on every field
    or array element store, using the language's generic type system.  (This
    might sound easy and cheap when the type is something simple like
    `List<String>`, but can quickly get expensive when they are something like
    `Map<? extends List<?  super Foo>>, ? super Set<? extends Bar>>`.  (In fact,
    later research cast doubt on the [decidability of generic
    subtyping](https://www.cis.upenn.edu/~bcpierce/papers/variance.pdf)).
  - **JVM ecosystem.**  Getting a dozen JVM vendors to agree on if, and how,
    type information would be reified at runtime was a highly questionable
    proposition.
  - **Delivery pragmatics.**  Even if it were possible to get a dozen  JVM
    vendors to agree on a scheme that could actually work, it would have greatly
    increased the complexity, timeline, and risk of an already substantial and
    risky effort.
  - **Language ecosystem.**  Languages like Scala might not have been happy to
    have Java's invariant generics burned into the semantics of the JVM.
    Agreeing on a set of acceptable cross-language semantics for generics in the
    JVM would again have increased the complexity, timetable, and risk of an
    already substantial and risky effort.
  - **Users would have to deal with erasure (and therefore heap pollution)
    anyway.**  Even if type information could be preserved at runtime, there
    would always be dusty classfiles that were compiled before the class was
    generified, so there would still be the possibility that any given
    `ArrayList` in the heap had no type information attached, with the attendant
    risk of heap pollution.
  - **Certain useful idioms would have been inexpressible.**  Existing generic
    code will occasionally resort to unchecked casts when it knows something
    about runtime types that the compiler does not, and there is no easy way to
    express it in the generic type system; many of these techniques would have
    been impossible with reified generics, meaning that they would have to have
    been expressed in a different, and often far more expensive, way.

It is clear that the costs and risks would have been substantial; what would
have been the benefits?  Earlier, we cited three possible benefits of
reification: reflection, layout specialization, and run-time type checking.  The
above arguments largely rule out the possibility we would have gotten run time
type checking (runtime cost, undecidability risk, ecosystem risk, and the
existence of erased instances).  

Surely it would be nice to be able to ask a `List` what its element type is (and
maybe it could have answered, but maybe not) -- this is clearly of nonzero
benefit.  It is just that the costs and benefits were out of line by several
orders of magnitude.  (Another cost of the chosen translation strategy is that
primitives can not be supported as type parameters; instead of `List<int>`, we
have to use `List<Integer>`.)

The common misconception that erasure is "a dirty hack" generally stems from a
lack of awareness of what the true costs of the alternative would have been,
both in engineering effort, time to market, delivery risk, performance,
ecosystem impact, and programmer convenience given the large volume of Java code
already written and the diverse ecosystem of both JVM implementations and
languages running on the JVM.
