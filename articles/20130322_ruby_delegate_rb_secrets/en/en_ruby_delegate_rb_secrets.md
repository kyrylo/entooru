Ruby Delegate.Rb Secrets
========================

| Project: | [Entooru](https://www.github.com/kyrylo/entooru/)
|:---------|:-----------------------------------------------------------------
| Author:  | Jim Gay <jim@saturnflyer.com>
| Date:    | March 21, 2013
| URI:     | [http://www.saturnflyer.com/blog/jim/2013/03/21/ruby-delegate-rb-secrets/][0]


You’ve seen SimpleDelegator in action and used it a bit yourself. The `delegate`
library is more than just a fancy `method_missing` wrapper.

Easy Wrappers
-------------

First and foremost, SimpleDelegator is a fancy `method_missing` wrapper. I know
I said the library was more than that, just bear with me.

Here’s some sample code:

```ruby
jim = Person.new # some object

class Displayer < SimpleDelegator
  def name_with_location
    "#{__getobj__.name} of #{__getobj__.city}"
  end
end

displayer = Displayer.new(jim)

puts displayer.name_with_location #=> "Jim of Some City"
```

That `Displayer` class initializes with an object and automatically sets it as
`@delegate_sd_obj`. You’ll also get both a `__getobj__` and a `__setobj__`
method to handle the assignment of the `@delegate_sd_obj`.

You may want to alias those methods so they won’t be so ugly when you use them:
`alias_method :__getobj__, :object`.

### Method Missing

Here’s an expanded view of how it handles `method_missing`:

```ruby
target = self.__getobj__ # Get the target object
if target.respond_to?(the_missing_method)
  target.__send__(the_missing_method, *arguments, &block)
else
  super
end
```

The actual code is a bit more compact than that, but it’s that simple.
SimpleDelegator is so simple, in fact, that you can create your own
implementation just like this:

```ruby
class MyWrapper
  def initialize(target)
    @target = target
  end
  attr_reader :target

  def method_missing(method_name, *args, &block)
    target.respond_to?(method_name) ? target.__send__(method_name, *args, &block) : super
  end
end
```

That’s not everything, but if all you need is simple use of `method_missing`,
this is how it works.

SimpleDelegator Methods
-----------------------

SimpleDelegator adds some convenient ways to see what methods are available. For
example, if we have our `jim` object wrapped by `displayer`, what can we do with
it? Well if we call `displayer.methods` we’ll get back a unique collection of
both the object’s and wrapper’s methods.

Here’s what it does:

```ruby
def methods(all=true)
  __getobj__.methods(all) | super
end
```

It defines the `methods` method and uses the union method “|” from Array to make
a unique collection. The object’s methods are combined with those of the wrapper.

```ruby
['a','b'] | ['c','b'] #=> ['a','b','c']
```

The same behavior is implemented for `public_methods` and `protected_methods`
but **not** `private_methods`. Private methods are private, so you probably
shouldn’t be accessing those from the outside anyway.

Why does it do this? Don’t we want to know that the main object and the
SimpleDelegator object have methods of the same name?

Not really.

From the outside all we care to know is what messages we can send to an object.
If both your main object and your wrapper have methods of the same name, the
wrapper will intercept the message and handle it. What you choose to do inside
your wrapper is up to you, but all these `methods` lists need to provide is that
the wrapper can receive any of those messages.

Handling Clone And Dup
----------------------

SimpleDelegator will also prepare clones and dups for your target object.

```ruby
def initialize_clone(obj) # :nodoc:
  self.__setobj__(obj.__getobj__.clone)
end
def initialize_dup(obj) # :nodoc:
  self.__setobj__(obj.__getobj__.dup)
end
```

Read Jon Leighton’s post about [`initialize_clone`, `initialize_dup` and
`initialize_copy` in Ruby][1] for more details about _when_ those methods are
called.

Making Your Own SimpleDelegator
-------------------------------

SimpleDelegator actually inherits almost all of this from Delegator. In fact,
the only changes that SimpleDelegator makes is 2 convenience methods.

```ruby
class SimpleDelegator < Delegator
  def __getobj__
    @delegate_sd_obj
  end
  def __setobj__(obj)
    raise ArgumentError, "cannot delegate to self" if self.equal?(obj)
    @delegate_sd_obj = obj
  end
end
```

Subtracting all the comments around what those methods mean, that’s the entirety
of the class definition as it is in the standard library. If you prefer to use
your own and call it SuperFantasticDelegator, you only need to make these same
getter and setter methods and you’ve got all that you need to replace
SimpleDelegator.

Keep in mind, however, that the `__setobj__` method has some protection in there
against setting the target object to the wrapper itself. You’ll need to do that
too unless you want to get stuck in an endless `method_missing` loop.

Using DelegateClass
-------------------

The delegate library also provides a method called DelegateClass which returns
a new class.

Here’s how you might use it:

```ruby
class Tempfile < DelegateClass(File)
  def initialize(basename, tmpdir=Dir::tmpdir)
    @tmpfile = File.open(tmpname, File::RDWR|File::CREAT|File::EXCL, 0600)
    super(@tmpfile)
  end

  # more methods here...
end
```

This creates a `Tempfile` class that has all the methods defined on `File` but
it automatically sets up the message forwarding with `method_missing`.

Inside the `DelegateClass` method it creates a new class with
`klass = Class.new(Delegator)`.

Then it gathers a collection of methods to define on this new class.

```ruby
methods = superclass.instance_methods
methods -= ::Delegator.public_api
methods -= [:to_s,:inspect,:=~,:!~,:===]
```

It gets the `instance_methods` from the superclass and subtracts and methods
already in the `Delegator.public_api` (which is just the
`public_instance_methods`). Then it removes some special string and comparison
methods (probably because you’ll want to control these yourself and not have
any surprises).

Next it opens up the `klass` that it created and defines all the leftover methods.

```ruby
klass.module_eval do
  def __getobj__  # :nodoc:
    @delegate_dc_obj
  end
  def __setobj__(obj)  # :nodoc:
    raise ArgumentError, "cannot delegate to self" if self.equal?(obj)
    @delegate_dc_obj = obj
  end
  methods.each do |method|
    define_method(method, Delegator.delegating_block(method))
  end
end
```

The code is sure to define the `__getobj__` and `__setobj__` methods so that it
will behave like SimpleDelegator. Remember, it’s copying methods from `Delegator`
which doesn’t define `__getobj__` or `__setobj__`.

What’s interesting here is that it’s using `Delegator.delegating_block(method)`
to create each of the methods. That `delegating_block` returns a lambda that is
used as the block for the method definition. As it defines each of those methods
in the `methods` collection, it creates a forwarding call to the target object.
Here’s the equivalent of what each of those methods will do:

```ruby
target = self.__getobj__
target.__send__(method_name, *arguments, &block)
```

For every method that it gathers to define on this new `DelegateClass` it
forwards the message to the target object as defined by `__getobj__`. Pay close
attention to that. Remember that I pointed out how you can make your own
SimpleDelegator and create your own getter and setter methods? Well
DelegateClass creates methods that expect `__getobj__` specifically. So if you
want to use `DelegateClass` but don’t want to use that method explicitly, you’ll
need to rely on `alias_method` to name it something else. All of your
automatically defined methods rely on `__getobj__`.

Lastly, before returning the `klass`, the `public_instance_methods` and
`protected_instance_methods` are defined. There’s some interesting things going
on in those method definitions, but I’ll keep the explanation simple for now.

This Tempfile class that we created is actually exactly how the standard
library’s Tempfile is defined.

If you’re not familiar with it, you can use it like this:

```ruby
require 'tempfile'
file = Tempfile.new('foo')
# then do whatever you need with a tempfile
```

If you dive into that library you’ll see:

```ruby
class Tempfile < DelegateClass(File)
```

The `tempfile` library relies on the `delegate` library, but not in the way that
you might find in the wild. Often I see developers using only the SimpleDelegator
class, but as you can see there’s a handful of other ways to make use of
`delegate` to handle message forwarding for you.

[0]: http://www.saturnflyer.com/blog/jim/2013/03/21/ruby-delegate-rb-secrets/
[1]: http://www.jonathanleighton.com/articles/2011/initialize_clone-initialize_dup-and-initialize_copy-in-ruby/
