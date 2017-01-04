---
layout: post
title: "TensorFlow in Scala with ScalaPy"
date: "2017-01-04"
color: "#ef6c00"
---

This winter break, I started work on a project for controlling robots with neural networks. I knew I wanted to use Scala for implementing the project, because of its static-typing safety and potential for integration into distributed computation pipelines. But I also wanted to use [TensorFlow](https://www.tensorflow.org)'s Python API, due to its user-friendliness and the large number of existing examples to learn from. In order to have the best of both worlds, I created [ScalaPy](https://github.com/shadaj/scalapy), which allowed me to use TensorFlow's Python API from Scala code. In this blog post, we'll go through the basics of how ScalaPy works and then implement a basic TensorFlow example in Scala.

## How ScalaPy works
ScalaPy gives the ability to use Python code from Scala, both through dynamic and static interfaces. It is built upon [Jep](https://github.com/mrj0/jep), which provides access to a CPython interpreter through JNI. Because Jep uses the native interpreter, it is compatible with any Python library and doesn't require special incantations to work with Python libraries that depend on native bindings. Since Jep only provides functions for evaluating Python code in the interpreter, ScalaPy provides a layer on top of these functions to interact with Python libraries in a manner similar to interacting with a Scala or Java library.

At the core of ScalaPy is `py.Object`, which is the base class for interacting with Python values. Whenever a `py.Object` is created, a variable is generated in Python land to hold the value, since Jep does not provide a way to hold onto a Python value directly.

The variable held onto by the `Object` can then be passed to other Python functions and can have methods called on it. Since Python is dynamically typed, `py.Object`s can be casted to `DynamicObject`s, which extend `scala.Dynamic` to provide a dynamic interface to the object's methods and values. Since there can potentially be multiple Jep instances running at the same time, interacting with any `py.Object`s always takes an implicit `Jep` object to execute operations with.

Let's start with a simple example:

```scala
$ sbt -Djava.library.path=./lib/
[info] Set current project to scalapy (in build file:.../scalapy/)
> console
[info] Starting scala interpreter...
[info]
Welcome to Scala 2.12.1 (Java HotSpot(TM) 64-Bit Server VM, Java 1.8.0_112).
Type in expressions for evaluation. Or try :help.

scala> import me.shadaj.scalapy.py
import me.shadaj.scalapy.py

scala> implicit val interpeter = new jep.Jep()
interpeter: jep.Jep = jep.Jep@52028600

scala> val num1 = py.Object("1")
num1: me.shadaj.scalapy.py.Object = 1

scala> val num2 = py.Object("2")
num2: me.shadaj.scalapy.py.Object = 2

scala> num1.asInstanceOf[py.DynamicObject] + num2
res0: me.shadaj.scalapy.py.DynamicObject = 3
```

Here, we created objects by passing Python expressions to evaluated, converted one of the objects to a `DynamicObject` so that we can call `+` on it with the other number. But ScalaPy contains many conversions between Scala and Python types, so we can just run:

```scala
scala> num1.asInstanceOf[py.DynamicObject] + 3
res1: me.shadaj.scalapy.py.DynamicObject = 4
```

and here ScalaPy automatically converts the `3` into a `py.Object`.

To access global methods and values, we can use `py.global`, which is already dynamic:

```scala
scala> py.global.len(Seq(1, 2, 3))
res2: me.shadaj.scalapy.py.Object = 3
```

Two things just happened here: the `Seq(1, 2, 3)` is a Scala sequence which was converted to a Python array, and we called the `len()` method on that the get the length of the array.

If we want to access Python modules, we can use `py.module(moduleName)`, which translates into the result of `import moduleName`.

```scala
scala> val np = py.module("numpy")
np: me.shadaj.scalapy.py.Module = me.shadaj.scalapy.py.Module@1acbc3e3

scala> np.arange(15).reshape(3, 5)
res3: me.shadaj.scalapy.py.DynamicObject =
[[ 0  1  2  3  4]
 [ 5  6  7  8  9]
 [10 11 12 13 14]]
```

Here we loaded up NumPy, we generated a range of numbers from 0 to 14, and reshaped it into a 3x5 matrix.

## TensorFlow with ScalaPy
Now we're ready to start playing with TensorFlow. We'll be implementing the introductory example at [https://www.tensorflow.org/get_started/](https://www.tensorflow.org/get_started/). Let's start by loading up the modules we need:

```scala
implicit val jep = new Jep()

// prep for tensorflow
val sys = py.module("sys")
sys.argv = Array("jep")

val tf = py.module("tensorflow").as[TensorFlow]
val np = py.module("numpy").as[NumPy]
```

Before we load TensorFlow, we have to set up the `sys.argv` property, which TensorFlow expects to be initialized but is not set up by Jep. In addition, you'll see that we placed a `.as[TensorFlow]` and `.as[NumPy]`. This is the way of wrapping a Python object in a static facade in ScalaPy. `TensorFlow` and `NumPy` are Scala classes that extend `ObjectFascade`, the base type for all ScalaPy facades. Although Python objects can be manipulated as dynamic values, static facades help to check your code at compile time to minimize errors during runtime.

Now that TensorFlow is loaded, we can continue by preparing the data to fit a line to:

```scala
val xData = np.random.rand(100).astype(np.float32)
val yData = (xData * 0.1) + 0.3
```

Then, to set up the graph to fit a line, we define variables for the slope and intercept, and prepare the equation for predicting data:

```scala
val W = tf.Variable(tf.random_uniform(Seq(1), -1, 1)) // slope
val b = tf.Variable(tf.zeros(Seq(1))) // intercept
val y = (W * xData) + b // predicted
```

To learn the optimal values for `W` and `b`, we define the loss to minimize and select an optimization strategy:

```scala
val loss = tf.reduce_mean(tf.square(y - yData))
val optimizer = tf.train.GradientDescentOptimizer(0.5)
val train = optimizer.minimize(loss)
```

Finally, we create a TensorFlow session and run 200 iterations of gradient descent:

```scala
val init = tf.global_variables_initializer()

val sess = tf.Session()
sess.run(init)

(0 to 200).foreach { step =>
  sess.run(train)

  if (step % 20 == 0) {
    println(s"$step ${sess.run(W).head} ${sess.run(b).head}")
  }
}
```

And that's it! If you run the program you will see gradient descent converge to values of 0.1 for `W` and 0.3 for `b`! There's a lot going on behind the scenes of ScalaPy to maintain high performance and make interop as seamless as possible, but I'll write a later blog post that covers that.

You can check out [ScalaPy](https://github.com/shadaj/scalapy), [scalapy-numpy](https://github.com/shadaj/scalapy-numpy), and [scalapy-tensorflow](https://github.com/shadaj/scalapy-tensorflow) on GitHub.
