---
title:  "Scala Closure's True Self"
categories: scala
---

When I write spark program, I often deal with transformation functions. These functions need to be serialized to enable parallelization. Scala uses lamda closure as syntax suger to write transformation functions. But it really frustrates me when the closure cannot be serialized which causes runtime failure of spark program.

## Example code snippet
In order to figure out the true self of ***scala closure***, take the following code snippet as an [example](http://stackoverflow.com/questions/2670350/how-are-scala-closures-transformed-to-java-objects).
{% highlight scala %}
class Close {
  var n = 5
  def method(i: Int) = i+n
  def function = (i: Int) => i+5
  def closure = (i: Int) => i+n
  def mixed(m: Int) = (i: Int) => i+m
}
{% endhighlight %}

## Methods to dig into the secret
There is something we should know when debuging scala program. Sometimes we need to find out the java code it transform to or even byte code that is compiled to.

 * **source code -> class**:
{% highlight bash %}
    $ scalac Class.scala
{% endhighlight %}
 * **source code -> class (with syntax tree parsed info printed)**:
{% highlight bash %}
    $ scalac -Xprint:all Class.scala
{% endhighlight %}
 * **class -> decompiled signature**:
{% highlight bash %}
    $ scalap / javap Class
{% endhighlight %}
 * **class -> byte code**:
{% highlight bash %}
    $ javap -c Class
{% endhighlight %}

## Parsed syntax tree reveals the truth
The syntax tree of the example class is as follows:
{% highlight scala %}
package <empty> {
  class Close extends Object {
    private[this] var n: Int = _;
    <accessor> def n(): Int = Close.this.n;
    <accessor> def n_=(x$1: Int): Unit = Close.this.n = x$1;
    def method(i: Int): Int = i.+(Close.this.n());
    def function(): Function1 = (new <$anon: Function1>(Close.this): Function1);
    def closure(): Function1 = (new <$anon: Function1>(Close.this): Function1);
    def mixed(m: Int): Function1 = (new <$anon: Function1>(Close.this, m): Function1);
    def <init>(): Close = {
      Close.super.<init>();
      Close.this.n = 5;
      ()
    }
  };
  @SerialVersionUID(value = 0) final <synthetic> class anonfun$function$1 extends scala.runtime.AbstractFunction1$mcII$sp with Serializable {
    final def apply(i: Int): Int = anonfun$function$1.this.apply$mcII$sp(i);
    <specialized> def apply$mcII$sp(i: Int): Int = i.+(5);
    final <bridge> <artifact> def apply(v1: Object): Object = scala.Int.box(anonfun$function$1.this.apply(scala.Int.unbox(v1)));
    def <init>($outer: Close): <$anon: Function1> = {
      anonfun$function$1.super.<init>();
      ()
    }
  };
  @SerialVersionUID(value = 0) final <synthetic> class anonfun$closure$1 extends scala.runtime.AbstractFunction1$mcII$sp with Serializable {
    final def apply(i: Int): Int = anonfun$closure$1.this.apply$mcII$sp(i);
    <specialized> def apply$mcII$sp(i: Int): Int = i.+(anonfun$closure$1.this.$outer.n());
    <synthetic> <paramaccessor> <artifact> private[this] val $outer: Close = _;
    final <bridge> <artifact> def apply(v1: Object): Object = scala.Int.box(anonfun$closure$1.this.apply(scala.Int.unbox(v1)));
    def <init>($outer: Close): <$anon: Function1> = {
      if ($outer.eq(null))
        throw null
      else
        anonfun$closure$1.this.$outer = $outer;
      anonfun$closure$1.super.<init>();
      ()
    }
  };
  @SerialVersionUID(value = 0) final <synthetic> class anonfun$mixed$1 extends scala.runtime.AbstractFunction1$mcII$sp with Serializable {
    final def apply(i: Int): Int = anonfun$mixed$1.this.apply$mcII$sp(i);
    <specialized> def apply$mcII$sp(i: Int): Int = i.+(anonfun$mixed$1.this.m$1);
    final <bridge> <artifact> def apply(v1: Object): Object = scala.Int.box(anonfun$mixed$1.this.apply(scala.Int.unbox(v1)));
    <synthetic> <paramaccessor> private[this] val m$1: Int = _;
    def <init>($outer: Close, m$1: Int): <$anon: Function1> = {
      anonfun$mixed$1.this.m$1 = m$1;
      anonfun$mixed$1.super.<init>();
      ()
    }
  }
}
{% endhighlight %}

So closure is transformed to an inner class with an *apply* method. For closure, the outer class object is passed into the closure inner class when constructed if any class member variable is referenced. So the closure is serializable only if the outer class is serializable. Otherwise, if a closure depends only on some local variables, then only these variables are passed into the closure when constructed. So one way to avoid serializing all members of outer class is to copy the necessary member variables to local ones.

What if a closure depends on some variables of its outer ***object***. See,

{% highlight scala %}
object Close {
  var n = 5
  def method(i: Int) = i+n
  def function = (i: Int) => i+5
  def closure = (i: Int) => i+n
  def mixed(m: Int) = (i: Int) => i+m
}
{% endhighlight %}

It is transformed to
{% highlight scala %}
package <empty> {
  object Close extends Object {
    private[this] var n: Int = _;
    <accessor> def n(): Int = Close.this.n;
    <accessor> def n_=(x$1: Int): Unit = Close.this.n = x$1;
    def method(i: Int): Int = i.+(Close.this.n());
    def function(): Function1 = (new <$anon: Function1>(): Function1);
    def closure(): Function1 = (new <$anon: Function1>(): Function1);
    def mixed(m: Int): Function1 = (new <$anon: Function1>(m): Function1);
    def <init>(): Close.type = {
      Close.super.<init>();
      Close.this.n = 5;
      ()
    }
  };
  @SerialVersionUID(value = 0) final <synthetic> class anonfun$function$1 extends scala.runtime.AbstractFunction1$mcII$sp with Serializable {
    final def apply(i: Int): Int = anonfun$function$1.this.apply$mcII$sp(i);
    <specialized> def apply$mcII$sp(i: Int): Int = i.+(5);
    final <bridge> <artifact> def apply(v1: Object): Object = scala.Int.box(anonfun$function$1.this.apply(scala.Int.unbox(v1)));
    def <init>(): <$anon: Function1> = {
      anonfun$function$1.super.<init>();
      ()
    }
  };
  @SerialVersionUID(value = 0) final <synthetic> class anonfun$closure$1 extends scala.runtime.AbstractFunction1$mcII$sp with Serializable {
    final def apply(i: Int): Int = anonfun$closure$1.this.apply$mcII$sp(i);
    <specialized> def apply$mcII$sp(i: Int): Int = i.+(Close.this.n());
    final <bridge> <artifact> def apply(v1: Object): Object = scala.Int.box(anonfun$closure$1.this.apply(scala.Int.unbox(v1)));
    def <init>(): <$anon: Function1> = {
      anonfun$closure$1.super.<init>();
      ()
    }
  };
  @SerialVersionUID(value = 0) final <synthetic> class anonfun$mixed$1 extends scala.runtime.AbstractFunction1$mcII$sp with Serializable {
    final def apply(i: Int): Int = anonfun$mixed$1.this.apply$mcII$sp(i);
    <specialized> def apply$mcII$sp(i: Int): Int = i.+(anonfun$mixed$1.this.m$1);
    final <bridge> <artifact> def apply(v1: Object): Object = scala.Int.box(anonfun$mixed$1.this.apply(scala.Int.unbox(v1)));
    <synthetic> <paramaccessor> private[this] val m$1: Int = _;
    def <init>(m$1: Int): <$anon: Function1> = {
      anonfun$mixed$1.this.m$1 = m$1;
      anonfun$mixed$1.super.<init>();
      ()
    }
  }
}
{% endhighlight %}

The outer object is not passed by. The object's member variable is used in the closure.


Keywords:  scala 闭包，序列化