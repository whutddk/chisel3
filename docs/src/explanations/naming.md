---
layout: docs
title:  "Naming"
section: "chisel3"
---

Historically, Chisel has had trouble reliably capturing the names of signals. The reasons for this are due to (1)
primarily relying on reflection to find names, (2) using `@chiselName` macro which had unreliable behavior.

Chisel 3.4 introduced a custom Scala compiler plugin which enables reliabe and automatic capturing of signal names, when
they are declared. In addition, this release includes prolific use of a new prefixing API which enables more stable
naming of signals programmatically generated from function calls.

This document explains how naming now works in Chisel for signal and module names. For cookbook examples on how to fix
systemic name-stability issues, please refer to the naming [cookbook](../cookbooks/naming).

### Compiler Plugin

```scala mdoc
// Imports used by the following examples
import chisel3._
import chisel3.experimental.{prefix, noPrefix}
import chisel3.stage.ChiselStage
```

With the release of Chisel 3.4, users should add the following line to their build.sbt settings to get the improved
naming:

```scala
// chiselVersion is the String version (eg. "3.4.0")
addCompilerPlugin("edu.berkeley.cs" % "chisel3-plugin" % chiselVersion cross CrossVersion.full)
```

This plugin will run after the 'typer' phase of the Scala compiler. It looks for any user code which is of the form
`val x = y`, where `x` is of type `chisel3.Data`, `chisel3.MemBase`, or `chisel3.experimental.BaseModule`. For each
line which fits this criteria, it rewrites that line. In the following examples, the commented line is the what the
line above is rewritten to.

If the line is within a bundle declaration or is a module instantiation, it is rewritten to replace the right hand
side with a call to `autoNameRecursively`, which names the signal/module.

```scala mdoc
class MyBundle extends Bundle {
  val foo = Input(UInt(3.W))
  // val foo = autoNameRecursively("foo")(Input(UInt(3.W)))
}
class Example1 extends MultiIOModule {
  val io = IO(new MyBundle())
  // val io = autoNameRecursively("io")(IO(new MyBundle()))
}
```
```scala mdoc:verilog
ChiselStage.emitVerilog(new Example1)
```

Otherwise, it is rewritten to also include the name as a prefix to any signals generated while executing the right-hand-
side of the val declaration:

```scala mdoc
class Example2 extends MultiIOModule {
  val in = IO(Input(UInt(2.W)))
  // val in = autoNameRecursively("in")(prefix("in")(IO(Input(UInt(2.W)))))

  val out = IO(Output(UInt(2.W)))
  // val out = autoNameRecursively("out")(prefix("out")(IO(Output(UInt(2.W)))))

  def inXin() = in * in

  val add = 3.U + inXin()
  // val add = autoNameRecursively("add")(prefix("add")(3.U + inXin()))
  // Note that the intermediate result of the multiplication is prefixed with `add`

  out := add + 1.U
}
```
```scala mdoc:verilog
ChiselStage.emitVerilog(new Example2)
```

Note that the naming also works if the hardware type is nested in an `Option` or a subtype of `Iterable`:

```scala mdoc
class Example3 extends MultiIOModule {
  val in = IO(Input(UInt(2.W)))
  // val in = autoNameRecursively("in")(prefix("in")(IO(Input(UInt(2.W)))))

  val out = IO(Output(UInt()))
  // val out = autoNameRecursively("out")(prefix("out")(IO(Output(UInt(2.W)))))

  def inXin() = in * in

  val opt = Some(3.U + inXin())
  // Note that the intermediate result of the inXin() is prefixed with `opt`:
  // val opt = autoNameRecursively("opt")(prefix("opt")(Some(3.U + inXin())))

  out := opt.get + 1.U
}
```
```scala mdoc:verilog
ChiselStage.emitVerilog(new Example3)
```

There is also a slight variant (`autoNameRecursivelyProduct`) for naming hardware with names provided by an unapply:
```scala mdoc
class UnapplyExample extends MultiIOModule {
  def mkIO() = (IO(Input(UInt(2.W))), IO(Output(UInt())))
  val (in, out) = mkIO()
  // val (in, out) = autoNameRecursivelyProduct(List(Some("in"), Some("out")))(mkIO())

  out := in
}
```
```scala mdoc:verilog
ChiselStage.emitVerilog(new UnapplyExample)
```

Note that the compiler plugin will not insert a prefix in these cases because it is ambiguous what the prefix should be.
Users who desire a prefix are encouraged to provide one as [described below](#prefixing).

### Prefixing

As shown above, the compiler plugin automatically attempts to prefix some of your signals for you. However, you as a
user can also add your own prefixes. This is especially for ECO-type fixes where you need to add some logic to a module
but don't want to influence other names in the module.

In the following example, we prefix additional logic with "ECO", where `Example4` is pre-ECO and `Example5` is post-ECO:

```scala mdoc
class Example4 extends MultiIOModule {
  val in = IO(Input(UInt(2.W)))
  val out = IO(Output(UInt()))

  val add = in + in + in

  out := add + 1.U
}

class Example5 extends MultiIOModule {
  val in = IO(Input(UInt(2.W)))
  val out = IO(Output(UInt()))

  val add = in + in + in

  out := prefix("ECO") { add + 1.U + in }
}
```
```scala mdoc:verilog
ChiselStage.emitVerilog(new Example4)
ChiselStage.emitVerilog(new Example5)
```

Also note that the prefixes append to each other (including the prefix generated by the compiler plugin):

```scala mdoc
class Example6 extends MultiIOModule {
  val in = IO(Input(UInt(2.W)))
  val out = IO(Output(UInt()))

  val add = prefix("foo") { in + in + in }

  out := add
}
```
```scala mdoc:verilog
ChiselStage.emitVerilog(new Example6)
```

Sometimes you may want to disable the prefixing. This might occur if you are writing a library function and
don't want the prefixing behavior. In this case, you can use the `noPrefix` object:

```scala mdoc
class Example7 extends MultiIOModule {
  val in = IO(Input(UInt(2.W)))
  val out = IO(Output(UInt()))

  val add = noPrefix { in + in + in }

  out := add
}
```
```scala mdoc:verilog
ChiselStage.emitVerilog(new Example7)
```

### Suggest a Signal's Name (or the instance name of a Module)

If you want to specify the name of a signal, you can always use the `.suggestName` API. Please note that the suggested
name will still be prefixed (including by the plugin). You can always use the `noPrefix` object to strip this.

```scala mdoc
class Example8 extends MultiIOModule {
  val in = IO(Input(UInt(2.W)))
  val out = IO(Output(UInt()))

  val add = (in + (in + in).suggestName("foo"))

  out := add
}
```
```scala mdoc:verilog
ChiselStage.emitVerilog(new Example8)
```

### Set a Module Name

If you want to specify the module's name (not the instance name of a module), you can always override the `desiredName`
value. Note that you can parameterize the name by the module's parameters. This is an excellent way to make your module
names more stable and is highly recommended to do.

```scala mdoc
class Example9(width: Int) extends MultiIOModule {
  override val desiredName = s"EXAMPLE9WITHWIDTH$width"
  val in = IO(Input(UInt(width.W)))
  val out = IO(Output(UInt()))

  val add = (in + (in + in).suggestName("foo"))

  out := add
}
```
```scala mdoc:verilog
ChiselStage.emitVerilog(new Example9(8))
ChiselStage.emitVerilog(new Example9(1))
```

### Reflection Naming

Regardless of whether the compiler plugin is enabled or not, after Chisel constructs a module, it attempts to name all
members of the Module. This will name all vals which are fields of the module class, but it will not name any
vals in nested functions or scopes.

If the plugin successfully names a signal, the reflection naming will do nothing. We plan to deprecate all reflection
naming in a future Chisel release, but are leaving it to allow the plugin naming to be optional (but recommended).

For example, the signals in the following module are in a nested scope; the plugin successfully names them, but
reflection naming cannot:

```scala mdoc
class Example10 extends MultiIOModule {
  {
    val in = IO(Input(UInt(3.W)))
    val out = IO(Output(UInt()))

    val add = in + in

    out := add
  }
}
```

### @chiselName

This macro is no longer recommended as its functionality is entirely replaced by the compiler plugin. Feel free to
delete from your Chisel designs!