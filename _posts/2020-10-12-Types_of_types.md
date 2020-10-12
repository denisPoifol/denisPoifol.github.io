---
layout: post
title:  "Types of types"
subhead: "The types of types you might want to know about before mistyping your types."
date:   2020-10-12 10:00:00 +0100
categories: ["swift"]
---

You can also read this article on [my employer's blog](https://fabernovel.github.io/2020-10-12/types-of-types)

Let's talk about types and memory!  
In Swift a type can either be value type or reference type.

## Value types

A Value type is a type that provides value semantics.
But what is value semantics?  
Value semantics is the behaviour you have when you manipulate `Integer`.
{% highlight swift %}
var x: Int = 0
var y: Int = x
y += 1
print(x, y) // prints 0 1
{% endhighlight %}
The modification of the variable `y` affects only the value stored into the variable `y`. In Swift structs and enums are value types, as you can see from the following example.
{% highlight swift %}
struct MyValueType { 
    var value: Int
}

var x = MyValueType(value: 0)
var y: MyValueType = x
y.value += 1
print(x.value, y.value) // prints 0 1
{% endhighlight %}

## Reference types

You probably saw it coming: reference types are types that provide reference semantics.  
In Swift reference types are classes.
{% highlight swift %}
class MyReferenceType { 
    var value: Int
}

var x = MyReferenceType(value: 0)
var y: MyReferenceType = x
y.value += 1
print(x.value, y.value) // prints 1 1
{% endhighlight %}
The modification applied to the variable `y` will reflect on the variable `x` because they both rely on shared memory.

## When should you use one over the other?

Reference types are classes which means they are stored into the heap. When using them you are manipulating a pointer to the actual data, use them when you want to create a shared mutable state.  
Value types are the actual data and not a reference to the data. Because of that, copies of the same value have independent states. This also means they are safer to use when running code on multiple threads (no concurrent access is possible).  
When creating a value type variable it will be stored inside the stack.

## Are there types that are neither value nor reference type

The notion of value/reference type is actually a bit weird, is the following struct a value type?
{% highlight swift %}
class SomeClass { 
    var value: Int = 0
}

struct SomeWeirdStruct {
    private let someClass = SomeClass()

    var someOtherValue: Int = 0

    var value: Int {
        get { someClass.value }
        set { someClass.value = newValue }
    }
}

var x = SomeWeirdStruct()
var y: SomeWeirdStruct = x
y.value += 1
y.someOtherValue += 1
print(x.value, y.value) // prints 1 1
print(x.someOtherValue, y.someOtherValue) // prints 0 1

{% endhighlight %}
You might argue that modifying `y.value` ends up in a modification of `x.value` and there is a shared mutable state between the two, yet modifying `y.someOtherValue` does not impact `x.someOtherValue`

But I think a better way to see it is that `SomeWeirdStruct` is a value type, but its value is a `SomeClass` pointer and an `Integer`. And even though `value` is declared as a variable it is a *computed* variable which makes it a pair of functions.  
In less modern language you would write it this way :
{% highlight swift %}
struct SomeWeirdStruct {
    private let someClass = SomeClass()

    var someOtherValue: Int = 0

    func getValue() -> Int {
        someClass.value
    }

    func setValue(_ newValue) {
        someClass.value = newValue
    }
}
{% endhighlight %}
So in conclusion a type really is either a value or a reference type in Swift. But keep an open mind because saying `SomeWeirdStruct` is a reference type when it comes to its `value` variable may not be a valid statement but it is an understandable one. And if we reject such a statement, saying `Array` is a value type would only mean it is a struct, yet it is way more interesting to know that it provides value semantics for the elements stored in the array.
