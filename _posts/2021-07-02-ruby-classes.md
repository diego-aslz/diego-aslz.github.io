---
layout: post
title: "Ruby Classes"
category: posts
comments: true
---

<iframe width="560" height="315" src="https://www.youtube.com/embed/ygov3dW0kgU" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe>

Ruby is a dynamic interpreted language and this time I'm gonna talk about how
it handles classes and how dead simple it feels like.

The day I learned what I'm gonna show you here I had my mind blown. Hopefully I'll give you the same feeling.

Let's start with a simple class. Do you know what is `self` in each of these places?

```ruby
self                                              # => main
self.class                                        # => Object

class Person
  self                                            # => Person
  self.class                                      # => Class
end
```

As you can see, when we're in the root level, we're actually inside an `Object` called `main`. Now when
inside the class definition, `self` becomes the class itself, which is an instance of the `Class` class.

Pay attention to this statement: an instance of the `Class` class. That's right. Classes are just plain old objects
of `Class`.

Now let me show you a different way of creating a new class:

```ruby
Product = Class.new
Product                                           # => Product
Product.class                                     # => Class
```

We just create new instances of `Class`. It's that simple.

That constructor can also take an argument, which is the superclass. This is how you
can define inheritance.

```ruby
Apparel = Class.new(Product)
Apparel                                           # => Product
Apparel.class                                     # => Class
Apparel.ancestors                                 # => [Apparel, Product, Object, Kernel, BasicObject]
```

By the way, the `Class#ancestors` method allows you to see all the inheritance chain for a class.

The Ruby classes I've shown you this far are constants, but they don't need to be. Check this out:

```ruby
product = Class.new
product                                           # => #<Class:0x00007f9a48018238>
product.class                                     # => Class
```

Constants in Ruby are pretty much just variables that start with an upper case letter. In this example `product` is
a variable, not a constant, just because it starts with a lower case letter.

Did you notice the difference, though? Because now `product` is a variable and not a constant, it does not get a name
assigned to it. You can see that in the output: you see the memory address, not the class name. That's what
Ruby does for you behind the scenes when you assign a new class to a constant: it defines its name based on the
constant name.

Now let's dive into another Ruby feature: class methods. First an example you're probably very familiar with:

```ruby
class Person
  def self.table_name
    'people'
  end
end
Person.table_name                                 # => "people"
```

Here you have a simple class method defined. Not much going on here, right? But now that you know what `self` means
inside that context, is it more clear for you how that works?

If not, let me show you how we would define that method for the `Product` class, which is created with `Class.new`:

```ruby
def Product.table_name
  'products'
end
Product.table_name                                # => "products"
```

Can you see how these methods are defined? You pass them the object you want to attach the method to.
`self` in the first example is `Person`. In the second example we use `Product`, which is the constant we created
manually.

This doesn't apply only to classes. You can use that to define the so called singleton methods in specific objects.
Check this out:

```ruby
diego = Person.new
def diego.last_name
  'Selzlein'
end

john = Person.new

diego.last_name                                   # => "Selzlein"
john.last_name                                    # =>

# ~> -:55:in `<main>': undefined method `last_name' for #<Person:0x00007fba7f005000> (NoMethodError)
```

As you can see, when you do that only the object for which you defined the method will respond to that method.

And lastly, remember that when in the root level of the Ruby code we were actually inside the `Object` class?
Almost all classes in Ruby inherit from `Object`. And this is why when you define a method in the root level
context it becomes accessible almost everywhere: because you are almost always inside a class that inherits
from the `Object` class:

```ruby
self.class                                        # => Object
def say_hello
  'Hello!'
end
say_hello                                         # => "Hello!"

class Person
  def anything
    say_hello
  end
end
Person.new.anything                               # => "Hello!"
```

Well, that covers it for this episode. Thank you for watching and I hope you enjoyed this snack!
