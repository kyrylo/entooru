Ruby core classes aren't thread safe
====================================

| Project: | [Entooru](https://www.github.com/kyrylo/entooru/)
|:---------|:-----------------------------------------------------------------
| Author:  | Jesse Storimer <jesse@jstorimer.com>
| Date:    | February 25, 2013
| URI:     | [http://www.jstorimer.com/newsletter/ruby-core-classes-arent-thread-safe.html][0]


A few days I shared a quote on Twitter:

> Ruby Arrays aren't thread-safe, [...] MRI's threading implementation
> accidentally makes them thread-safe.

It saw a bunch of retweets and discussion, but I daresay that the nuances of
that statement were lost on a lot of developers. Heck, a few months ago, I
wouldn't have really known what that meant. It sounds like something bad. Does
this mean I shouldn't use Arrays? Or that I should only use MRI? Or that I
shouldn't use threads? **What the heck does this thread safety issue really mean
for my code?**

That's a big question, and I think the answer lies in education. If we don't
understand what thread-safe code is, or the nuances of MRI's threading
implementation, then that tweet doesn't do much more than spread FUD. So I want
to make sure we all understand what this is about.

Let's dig in with an example.

Here's a very simple inventory tracking class:

```ruby
class Inventory
  def initialize(stock_levels)
    @stock = stock_levels
  end

  def decrease(item)
    @stock[item] -= 1
  end

  def [](item)
    @stock[item]
  end
end
```

It's trivially simple. You give it your stock levels as a Hash, and then ask it
to decrease the inventory as sales are made.

```ruby
inventory = Inventory.new(:tshirt => 200,
                          :pants => 340,
                          :hats => 4000)

inventory.decrease(:hats)
inventory.decrease(:pants)
```

Now let me ask you this: is this thread-safe? How do you know? What tips you
off?

Let's exercise this code to find out. I have 4000 hats, so if I decrease the
inventory 4000 times, I should have 0 inventory left.

```ruby
4000.times do
  inventory.decrease(:hats)
end

puts inventory[:hats]
```

Running this code on MRI 1.9.3, the answer is, predictably, 0. But now our
e-commerce checkout process is deemed too slow. Your co-worker decides that
each checkout needs to be wrapped in a thread so work can be parallelized.

Now let's exercise the code again, with threads this time.

```ruby
threads = Array.new
40.times do
  threads << Thread.new do
    100.times do
      @inventory.decrease(:hats)
    end
  end
end

threads.each(&:join)
puts @inventory[:hats]
```

This time we spawn 40 threads that each decrease the inventory 100 times. This
adds up to 4000, so that the result should be 0 again. There's also a bit more
code in this example because we need to keep track of each thread, calling `join`
on it to make sure it has finished execution. Again I ask, is this code
thread-safe?

Here's the result of running this code against recent versions of MRI JRuby, and
Rubinius.

```
MRI 1.9.3    :  0
JRuby 1.7.2  :  486
RBX 2.0.0rc1 :  2
```

The code appears to be thread-safe when run against MRI, but it's exposed as _not_
thread-safe when run against other Ruby implementations. This is what the tweet
alluded to. Our experience with MRI may lead us to think that the core classes
are thread-safe, but this is an 'accident' of MRI that really shows itself as an
accident when you run the same code on other runtimes.

_Note:_ if you run this code yourself you'll almost certainly get different
non-zero results. This is the nature of the problem. It's unpredictable and
non-deterministic.

So what do we do? Well, there are general rules that produce thread-safe code in
any language.

**If we want our code to be thread-safe, concurrent mutations need to be
synchronized.** And I mean, _all_ concurrent mutations need to be synchronized. This
is really the main rule to writing thread-safe code. This may sound complex, but
I'll show you exactly how to do it in a minute.

Just to drive the point home, the most common way that thread safety is
threatened is when multiple threads share variables and try to modify them at
the same time. In Ruby, this is most obvious when there's a shared global
variable or class variable that different threads try to mutate, but it can just
as easily happen with shared instance variables or local variables when multiple
threads are in play.

**I highly recommend you read the [4 rules to safe concurrency in Ruby][1] (or any
other platform)**. Following those rules will make your life much easier as a
programmer.

So how do we make this `Inventory` code thread-safe? And why does it appear to be
thread-safe in MRI? First, let's make it thread-safe. The easiest solution here
is to use the Mutex class to synchronize access to the `@inventory`.

```ruby
threads = Array.new
lock = Mutex.new

400.times do
  threads << Thread.new do
    lock.synchronize do
      10.times do
        @inventory.decrease(:hats)
      end
    end
  end
end

threads.each(&:join)
puts @inventory[:hats]
```

Running this code against all three implementation again, I'm met with the
expected result.

```
MRI 1.9.3    :  0
JRuby 1.7.2  :  0
RBX 2.0.0rc1 :  0
```

We've just gone through a bunch of new code and skipped over talking about some
of the nuances I alluded to earlier. Let's start with the code.

The concept of a 'mutex' is not specific to Ruby. Just like threads, mutexes are
a mechanism provided by your OS kernel. 'Mutex' is short-hand for mutual
exclusion. A mutex allows you to synchronize access to some section of code. In
other words, **one and only one thread can execute the code that a mutex
synchronizes at any given time**. This is why it's also called a lock. When one
thread is decreasing the inventory, it has a lock on that bit of code. Other
threads must wait until that thread unlocks the mutex before they can have their
turn.

Mutexes protect access to shared state. You saw what happened to the inventory
stock levels when we didn't use a mutex: the data was corrupted. Stick to the 4
rules I mentioned above to avoid this. Forgetting to use a lock somewhere may
cause your data to be corrupted, which is a very sad thing.

So why did MRI produce the correct result without the mutex?

I'll give you a minute to think about it. The answer is an infamous set of
initials in the Ruby community... ding, ding, ding, you got it! The GVL. Also
known as the GIL or the Global Lock.

**The global lock is a feature of MRI that basically wraps a big mutex around all
of your code**. That's right, even if you're using multiple threads on a
multi-core CPU, if your code is running on MRI it will not run in parallel.
There are some very specific caveats that MRI makes for concurrent IO. If you
have one thread that's waiting on IO (ie. waiting for a response from the DB or
a web request), MRI will allow another thread to run in parallel. But in all
other cases, your code is wrapped with a big lock.

JRuby and Rubinius (2.0 and higher) don't have this global lock. This explains
why they produced incorrect results in the example above. Rather than a global
lock, key parts of the runtimes are protected with fine-grained locks so that
their internals are thread-safe. But this allows your code to be truly parallel.
And as we saw, **truly parallel code is great, but also allows for us to shoot
ourselves in the foot if we do the wrong thing**, but isn't that the case with
pretty much everything we do?

One more thing, if you benchmark our two solutions against each other, you'll
find that the multi-threaded version with a mutex is actually slower than the
single-threaded version. This is mostly a result of the contrived example we're
using, but it warrants pointing out. By definition, code inside a mutex cannot
be run in parallel, since only one thread can execute it at any given time. In
this example, all of the code was inside a mutex, so the extra threads just
added overhead, rather than making things faster. It's good to remember that
throwing more threads at a problem doesn't make it automatically faster.

Putting our contrived example in the real world, each thread would have more
responsibility related to the e-commerce checkout process. It would need to
collect payment, fulfill the shipment, notify someone via SMS, _as well_ as
decreasing the inventory. In that case, I'm more confident that a multi-threaded
approach would indeed win the benchmark.

**There's so much more to say on this topic, which is why it's the subject of my
next book**. Stay tuned and I'll have more info for you soon. _[Update: [the website][2]
is up with more info.]_

If you learned something from this email, or it left you with more unanswered
questions, hit reply and let me know. Replies go right to my personal inbox and
I always answer them.

Until next time,<br/>
Jesse<br/>
[http://workingwithcode.com][3]

[0]: http://www.jstorimer.com/newsletter/ruby-core-classes-arent-thread-safe.html
[1]: https://github.com/jruby/jruby/wiki/Concurrency-in-jruby#wiki-concurrency_basics]
[2]: http://workingwithrubythreads.com/
[3]: http://workingwithcode.com
