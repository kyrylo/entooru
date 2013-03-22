Understanding method lookup in Ruby 2.0
=======================================

| Project: | [Entooru](https://www.github.com/kyrylo/entooru/)
|:---------|:-----------------------------------------------------------------------------
| Author:  | Marc-André Lafortune <github_rocks@marc-andre.ca>
| Date:    | March 18, 2013
| URI:     | [http://tech.pro/tutorial/1149/understanding-method-lookup-in-ruby-20][0]


The introduction of `prepend` in Ruby 2.0 is a great opportunity to review how
exactly Ruby deals with method calls.

To understand method lookup it is imperative to master the class hierarchy in
Ruby. I've peppered this article with many code examples; you'll need Ruby 1.9.2
or newer to run most of them yourself. There's one that uses `prepend`; it will
only work in Ruby 2.0.0.

Class hierarchy
---------------

Let's start with a classical example of class-based inheritance:

```ruby
class Animal
  def initialize(name)
    @name = name
  end

  def info
    puts "I'm a #{self.class}."
    puts "My name is '#{@name}'."
  end
end

class Dog < Animal
  def info
    puts "I #{make_noise}."
    super
  end

  def make_noise
    'bark "Woof woof"'
  end
end

lassie = Dog.new "Lassie"
lassie.info
# => I bark "Woof woof".
#    I'm a dog.
#    My name is 'Lassie'.
```

[Fiddle with it!][1]

In this example, `Dog` inherits from `Animal`. We say that `Animal` is the
[superclass][2] of `Dog`:

```ruby
Dog.superclass # => Animal
```

Note that the method `Dog#info` calls `super`. This special keyword executes the
next definition of `info` in the hierarchy, in this case `Animal#info`.

The hierarchy of a class is available with `ancestors`:

```ruby
Dog.ancestors # => [Dog, Animal, Object, Kernel, BasicObject]
```

It's interesting to note that the ancestry doesn't end with `Animal`:

```ruby
Animal.superclass # => Object
```

The declaration `class Animal` was equivalent to writing class `Animal < Object`.

This is why an animal has more methods than just `info` and `make_noise`, in
particular introspection methods like `respond_to?`, `methods`, etc:

```ruby
lassie.respond_to? :upcase # => false
lassie.methods
 # => [:nil?, :===, :=~, :!~, :eql?, :hash, :<=>, :class, :singleton_class, ...]
```

So what about `Kernel` and `BasicObject`? I'll come back to `Kernel` later, and
there's not much to say about `BasicObject` besides the fact that is only has a
very limited number of methods and is the end of the hierarchy for all classes:

```ruby
# BasicObject is the end of the line:
Object.superclass # => BasicObject
BasicObject.superclass # => nil
# It has very few methods:
Object.instance_methods.size # => 54
BasicObject.instance_methods.size # => 8
```

> Classes form a _rooted tree_ with `BasicObject` as the root.

Mixins
------

Although Ruby supports only single inheritance (i.e. a class has only one
`superclass`), it also support [mixins][3]. A mixin is a collection of methods
that can be included in classes. In Ruby, these are instances of the `Module`
class:

```ruby
module Mamal
  def info
    puts "I'm a mamal"
    super
  end
end
Mamal.class # => Module
```

To pull this functionality in our `Dog` class, we can use either `include` or
the new `prepend`. These will insert the module either after or before the class
itself:

```ruby
class Dog
  prepend Mamal
end
lassie = Dog.new "Lassie"
lassie.info
# => I'm a mamal.
#    I bark "Woof woof".
#    I'm a dog.
#    My name is 'Lassie'.
Dog.ancestors # => [Mamal, Dog, Animal, Object, ...]
```

If the module was included instead of prepended, the effect would be similar,
but the order would be different. Can you guess the output and the ancestors?
[Try it here][4].

You can include and prepend as many modules as you want and modules can even be
`include`d and `prepend`ed to other modules<a name="sub1-r"></a><a href="#sub1">¹</a>.
Don't hesitate to call `ancestors` to double check the hierarchy of modules &
classes.

Singleton classes
-----------------

In Ruby, there's just one extra layer to the hierarchy of modules and classes.
Any object can have a special class just for itself that takes precedence over
everything: the _singleton class_.

Here's a simple example building on previous ones:

```ruby
scooby = Dog.new "Scooby-Doo"

class << scooby
  def make_noise
    'howl "Scooby-Dooby-Doo!"'
  end
end
scooby.info
# => I'm a mamal.
#    I howl "Scooby-Dooby-Doo!".
#    I'm a dog.
#    My name is 'Scooby-Doo'.
```

Note how the barking was replaced with Scooby Doo's special howl. This won't
affect any other instances of the `Dog` class.

The `class << scooby` is the special notation to reopen the singleton class of
an object. There are alternative ways to define singleton methods:

```ruby
# equivalent to previous example:
def scooby.make_noise
  'howl "Scooby-Dooby-Doo!"'
end
```

The singleton class is a real `Class` and it can be accessed by calling
`singleton_class`:

```ruby
# Singleton classes have strange names:
scooby.singleton_class # => #<Class:#<Dog:0x00000100a0a8d8>>
# Singleton classes are real classes:
scooby.singleton_class.is_a?(Class) # => true
# We can get a list of its instance methods:
scooby.singleton_class.instance_methods(false) # => [:make_noise]
```

All Ruby objects can have singleton classes<a name="sub2-r"></a><a href="#sub2">²</a>, including classes themselves and yes, even singleton classes.

This sounds a bit crazy... wouldn't that require an infinite number of singleton
classes? In a way, yes, but Ruby will create singleton classes as they are
needed.

Although the previous example used the singleton class of an instance of `Dog`,
it is more often used for classes. Indeed, "[class methods][5]" are actually
methods of the singleton class. For example, `attr_accessor` is an instance
method of the singleton class of `Module`. Ruby on Rails' `ActiveRecord::Base`
has many such methods like `has_many`, `validates_presence_of`, etc. These are
methods of the singleton class of `ActiveRecord::Base`:

```ruby
Module.singleton_class
      .private_instance_methods
      .include?(:attr_accessor) # => true

require 'active_record'
ActiveRecord::Base.singleton_method
                  .instance_methods(false)
                  .size  # => 170
Singleton classes get their names from the fact that they can only have a single instance:

scooby2 = scooby.singleton_class.new
  # => TypeError: can't create instance of singleton class
```

For basically the same reason, you can't directly inherit from a singleton class:

```ruby
class Scoobies < scooby.singleton_class
  # ...
end
# => TypeError: can't make subclass of singleton class
```

On the other hand, singleton classes have a complete class hierachy.

For objects, we have:

```ruby
scooby.singleton_class.superclass == scooby.class == Dog
# => true, as for most objects
```

For classes, Ruby will automatically set the superclass so that different paths
through the superclasses or the singleton classes are equivalent:

```ruby
Dog.singleton_class.superclass == Dog.superclass.singleton_class
# => true, as for any Class
```

This means that `Dog` inherits `Animal`'s instance methods as well as its
singleton methods.

Just to be certain to confuse everybody, I'll finish with a note on `extend`. It
can be seen as a shortcut to include a module in the singleton class of the
receiver<a name="sub3-r"></a><a href="#sub3">³</a>:

```ruby
obj.extend MyModule
# is a shortcut for
class << obj
  include MyModule
end
```

> Ruby's singleton class follow the [eigenclass model][6].

Method lookup and method missing
--------------------------------

Almost done!

The rich ancestor chain that Ruby supports is the basis of all method lookups.

When reaching the last superclass (`BasicObject`), Ruby provides an extra
possibility to handle the call with `method_missing`.

```ruby
lassie.woof # => NoMethodError: undefined method
  # `woof' for #<Dog:0x00000100a197e8 @name="Lassie">

class Animal
  def method_missing(method, *args)
    if make_noise.include? method.to_s
      puts make_noise
    else
      super
    end
  end
end

lassie.woof # => bark "Woof woof!"
scooby.woof # => NoMethodError ...
scooby.howl # => howl "Scooby-Dooby-Doo!"
```

[Fiddle with it!][7]

In this example, we call `super` unless the method name is part of the noise an
animal makes. `super` will go down the ancestors chain until it reaches
`BasicObject#method_missing` which will raise the `NoMethodError`.

Summary
-------

In summary, here's what happens when calling `receiver.message` in Ruby:

* send message down `receiver.singleton_class`' ancestors chain
* then send `method_missing(message)` down that chain

The first methods encountered in this lookup gets executed and its result gets
returned. Any call to `super` resumes the lookup to find the next method.

The ancestors chain for a Module `mod` is:

* the ancestors chain of each prepended module (last module prepended first)
* `mod` itself
* the ancestors chain of each included module (last module included first)
* if `mod` is a class, then the ancestors chain of its superclass.

We can write the `ancestors` method in pseudo Ruby code<a name="sub4-r"></a><a href="#sub4">⁴</a>:

```ruby
class Module
  def ancestors
    prepended_modules.flat_map(&:ancestors) +
    [self] +
    included_modules.flat_map(&:ancestors) +
    is_a?(Class) ? superclass.ancestors : []
  end
end
```

Writing the actual lookup is more tricky, but it would look like:

```ruby
class Object
  def send(method, *args)
    singleton_class.ancestors.each do |mod|
      if mod.defines? method
        execute(method, for: self, arguments: args,
          if_super_called: resume_lookup_at(mod))
      end
    end
    send :method_missing, method, *args
  end

  def method_missing(method, *args)
    # This is the end of the line.
    # If we're here, either no method was defined anywhere in ancestors,
    # and no method called 'method_missing' was defined either except this one,
    # or some methods were found but called `super` when there was
    # no more methods to be found but this one.
    raise NoMethodError, "undefined method `#{method}' for #{self}"
  end
end
```

Technical Footonotes
--------------------

<a name="sub1"></a>
<a href="#sub1-r">¹</a> There are some restrictions to mixins:

* Ruby does not support very well hierarchies where the same module appears more
  than once in the hierarchy. `ancestors` will (usually) only list a module once,
  even if it's included at [different][8] levels in the ancestry. I still hope this
  can change in the future, in particular to [avoid embarrassing][9] bugs.
* Including a submodule in a module `M` won't have any effect for any class that
  already included M, but it will for classes that included `M` afterwards. See
  it in action [in this fiddle][10].
* Also, Ruby does not allow cycles in the ancestors chain.

<a name="sub2"></a>
<a href="#sub2-r">²</a> Ruby actually prohibits access to singleton classes for
very few classes: `Fixnum`, `Symbol` and (since Ruby 2.0) `Bignum` and `Float`:

```ruby
42.singleton_class # => TypeError: can't define singleton
```

The rule of thumb is that _immediates_ can't have singleton classes. Since
`(1 << 42).class` will be `Fixnum` if the platform is 64 bits or `Bignum`
otherwise, it was felt best to treat `Bignum` the same way. The same reasoning
applies to `Float` in Ruby 2.0, since some floats [may be immediates][11].

The only exceptions are `nil`, `true` and `false` who are singletons of their
class: `nil.singleton_class` is `NilClass`.

<a name="sub3"></a>
<a href="#sub3-r">³</a> To be perfectly accurate, `extend` and
`singleton_class.send :include` have the same effect by default, but they will
trigger different callbacks: `extended` vs `included`, `append_features` vs
`extend_object`. If these callbacks are defined differently, then the effect
could be different.

<a name="sub4"></a>
<a href="#sub4-r">⁴</a> Note that, `prepended_modules` [does not exist yet][12].
Also, `singleton_class.ancestors` does not include the singleton class itself,
but this [will change in Ruby 2.1][13].

[0]: http://tech.pro/tutorial/1149/understanding-method-lookup-in-ruby-20
[1]: http://rubyfiddle.com/riddles/9a1d5/2
[2]: http://en.wikipedia.org/wiki/Superclass_%28computer_science%29
[3]: http://en.wikipedia.org/wiki/Mixin
[4]: http://rubyfiddle.com/riddles/c09ef/2
[5]: http://en.wikipedia.org/wiki/Method_%28computer_programming%29#Class_methods
[6]: http://en.wikipedia.org/wiki/Eigenclass_model
[7]: http://rubyfiddle.com/riddles/b33b4
[8]: https://bugs.ruby-lang.org/issues/1586
[9]: https://bugs.ruby-lang.org/issues/7844
[10]: http://rubyfiddle.com/riddles/37fb2
[11]: http://blog.marc-andre.ca/2013/02/23/ruby-2-by-example/#optimizations
[12]: http://bugs.ruby-lang.org/issues/8026
[13]: http://bugs.ruby-lang.org/issues/8035
