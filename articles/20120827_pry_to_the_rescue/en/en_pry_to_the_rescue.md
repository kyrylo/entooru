Pry to the rescue
=================

| Project: | [Entooru](https://www.github.com/kyrylo/entooru/)
|:---------|:--------------------------------------------------------------------
| Author:  | Conrad Irwin <conrad.irwin@gmail.com>
| Date:    | August 27, 2012
| URI:     | [http://cirw.in/blog/pry-to-the-rescue][0]


Introducing: [pry-rescue](https://github.com/ConradIrwin/pry-rescue): super-fast,
painless, debugging for the (ruby) masses.

What does it do?
================

Whenever an unhandled exception happens in your program, pry-rescue opens an interactive
[pry](http://pryrepl.org/) shell right at the point it was raised. Instead of glowering
at a stack-trace spewed out by a dying process, you'll be engaging with your program as
though it were still alive!

The pry console gives you access to the method that raised the exception, you can use it
to inspect the values of variables (no more print statements!), the source code of methods
(no more flapping around with a text editor!), and even move up and down the call stack
(like a real debugger!).

Because the shell opens as though the exception were just about to be raised, you don't
even need to re-run your program from the beginning in order to start debugging. It's
optimized for the "exceptions-happen" workflow that you need when developing code.

Ok, sounds good, how do I use this?
===================================

It's easy: `gem install pry-rescue pry-stack_explorer`, and then run your program using
`rescue foo.rb` instead of `ruby foo.rb`. If `rescue` doesn't float your boat, then just
wrap chunks of your program in `Pry::rescue{ }`. Any exceptions that are unhandled within
that block will be rescued by pry on your behalf. (rack middleware coming soon!)

Whenever pry opens because of an exception you have two choices: hit &lt;ctrl-d&gt; to let
the exception bubble on its way, or do some debugging! If you choose the latter path, then
you can use the full power of pry to fix the problem; and then `try-again` to verify the
fix worked.

Uh… do you have an example?
=============================

Sure! Let's imagine that I wrote some code that looks like this:

<div class="highlight"><pre><code clas="ruby"><span class="bold"><span class="f8"><span class="blink"></span></span><span class="bold">def</span> <span class="f8"><span class="nf">find_capitalized</span></span></span>(a)
   a.select <span class="f8"><span class="blink"></span></span><span class="bold">do</span> |name|
     name.chars.first == name.chars.first.upcase
   <span class="f8"><span class="blink"></span></span><span class="bold">end</span>
 <span class="f8"><span class="blink"></span></span><span class="bold">end</span>
</code></pre></div>

Now I run the code: `rescue rescue.rb`

<div class="highlight"><pre><code clas="ruby"><span class="bold">From:</span> rescue.rb @ line 2 Object#find_capitalized:

    <span class="ef30">2</span>: <span class="f8"><span class="blink"></span></span><span class="bold">def</span> <span class="f8"><span class="blink"><span class="nf">find_capitalized</span></span></span>(a)
    <span class="ef30">3</span>:   a.select <span class="f8"><span class="blink"></span></span><span class="bold">do</span> |name|
 =&gt; <span class="ef30">4</span>:     name.chars.first == name.chars.first.upcase
    <span class="ef30">5</span>:   <span class="f8"><span class="blink"></span></span><span class="bold">end</span>
    <span class="ef30">6</span>: <span class="f8"><span class="blink"></span></span><span class="bold">end</span>

NoMethodError: undefined method `chars' for :direction:Symbol
from rescue.rb:4:in `block in find_capitalized'
[1] pry(main)&gt;
</code></pre></div>

Well, it's gone wrong. But at least I can see why. Let's move `up` the stack and see if we
can find which code is calling this method with symbols:

<div class="highlight"><pre><code clas="ruby">[1] pry(main)&gt; up
<span class="bold">From:</span> rescue.rb @ line 8 Object#extract_people:

     <span class="ef30">8</span>: <span class="f8"><span class="blink"></span></span><span class="bold">def</span> <span class="f8"><span class="blink"><span class="nf">extract_people</span></span></span>(opts)
 =&gt;  <span class="ef30">9</span>:   name_keys = find_capitalized(opts.keys)
    <span class="ef30">10</span>: 
    <span class="ef30">11</span>:   name_keys.each_with_object({}) <span class="f8"><span class="blink"></span></span><span class="bold">do</span> |name, o|
    <span class="ef30">12</span>:     o[name] = opts.delete name
    <span class="ef30">13</span>:   <span class="f8"><span class="blink"></span></span><span class="bold">end</span>
    <span class="ef30">14</span>: <span class="f8"><span class="blink"></span></span><span class="bold">end</span>

[2] pry(main)&gt; opts
=&gt; {<span class="ef161"><span class="ef161">&quot;</span></span><span class="ef161"><span class="ef161">Arthur</span></span><span class="ef161"><span class="ef161">&quot;</span></span><span class="ef161"></span>=&gt;<span class="ef161"><span class="ef161">&quot;</span></span><span class="ef161"><span class="ef161">Dent</span></span><span class="ef161"><span class="ef161">&quot;</span></span><span class="ef161"></span>, <span class="ef90">:direction</span>=&gt;<span class="ef90">:left</span>}
</code></pre></div>

Ok, that seems odd, but fair enough, let's see if there's a better way of implementing
`find_capitalized`:

<div class="highlight"><pre><code clas="ruby">[3] pry(main)&gt; down
[4] pry(main)> name.first
NoMethodError: undefined method `first' for :direction
from (pry):9:in `block in find_capitalized'
[5] pry(main)> name.capitalize
=> <span class="ef90">:Direction<span>
</code></pre></div>

Got it. Let's `edit-method` to fix the code,  and `try-again` to verify the fix worked:

<div class="highlight"><pre><code clas="ruby">[6] pry(main)&gt; edit-method
[7] pry(main)> whereami
<span class="bold">From:</span> rescue.rb @ line 2 Object#find_capitalized:

    <span class="ef30">2</span>: <span class="f8"><span class="blink"></span></span><span class="bold">def</span> <span class="f8"><span class="blink"><span class="nf">find_capitalized</span></span></span>(a)
 =&gt; <span class="ef30">3</span>:   a.select <span class="f8"><span class="blink"></span></span><span class="bold">do</span> |name|
    <span class="ef30">4</span>:     name.capitalize == name
    <span class="ef30">5</span>:   <span class="f8"><span class="blink"></span></span><span class="bold">end</span>
    <span class="ef30">6</span>: <span class="f8"><span class="blink"></span></span><span class="bold">end</span>

[8] pry(main)&gt; try-again
Arthur Dent moves left
</pre></code></div>

<aside>If you want to work through this example yourself, just clone
`https://github.com/ConradIrwin/pry-rescue` and then run `rescue examples/rescue.rb`.</aside>

What else do I need to know?
============================

Well, there are a myriad of pry commands that are useful while debugging: `wtf?` and
`cat --ex` can be used to examine stack traces and the associated code; `cd` and `ls`
can be used to closely examine objects; `$` and `edit-method` can be used to view and edit
source code. The best thing to do is just type `help` once you've installed pry.

The other useful `pry-rescue` command is `cd-cause`. It lets you rewind back the the
previously raised exception. So, if you've rescued one exception, and then raised another
(it happens…) you can jump back to the original cause of the problem.

Apart from that, the only thing you might want to know is that you can use
`Pry::rescue{ }` with exceptions that you handle yourself. Just call `Pry::rescued(e)` in
your rescue block.

So, `gem install pry-rescue` now. If you find problems, please
let me know on [github](https://github.com/ConradIrwin/pry-rescue).

[0]: http://cirw.in/blog/pry-to-the-rescue
