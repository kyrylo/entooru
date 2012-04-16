Ruby Float quirks
=================

| Project: | [Entooru](https://www.github.com/kyrylo/entooru/)
|:---------|:-----------------------------------------------------------------
| Author:  | Clemens Helm <clemens@rails-troubles.com>
| Date:    | January 10, 2012
| URI:     | [http://www.rails-troubles.com/2011/12/ruby-float-quirks.html][0]


Working with Float values sometimes really leads to unexpected behavior. Take
the following example:

``` ruby
def greater_than_sum?(float1, float2)
  difference = float1 - float2
  difference + float2 > float1
end
```

You would expect this method to return `false` all the time. But try calling
`greater_than_sum?(214.04, 74.79)` and it will return `true`!

The problem
-----------

Open up an irb shell and type `139.25 + 74.79`, then you’ll see the reason for
this quirk: The result is `214.04000000000002`, which is apparently slightly
greater than `214.04`.

Float numbers cannot store decimal numbers precisely. The reason is that Float
is a binary number format and always needs to convert from decimal to binary and
vice versa.

As you probably know, not all numbers can be represented in decimal format. When
you calculate `1/3`, the result is `0.33333333333333…`. An endless chain of 3s.

The same holds true for binary numbers. When a decimal number is converted to
binary format, the resulting binary number might be endless. But since we don’t
have endless memory on our computers to represent the number, the computer cuts
off that number at some point. This produces a rounding error and that’s exactly
what we have to deal with here.

The solution
------------

In our previous example you can add a second condition to check if a number is
within a certain interval around another number, called _delta_:

``` ruby
def greater_than_sum?(float1, float2)
  delta = 0.0001
  difference = float1 - float2
  difference + float2 > float1 && difference + float2 - float1 < delta
end
```

What you choose as delta is pretty much up to you. It depends on the precision
you need in your comparisons and the expected error: Unfortunately the
calculation error increases the more calculations you perform on a number.

Decimal solution
----------------

If you need an exact comparison of 2 decimal values, you might wanna go with
[BigDecimal][1]. All calculations on decimal numbers are much more accurate than
with Floats.

So I just use BigDecimal instead of Float from now on!

Of course you can do that, but there is one big disadvantage: Calculations on
BigDecimals are considerably slower. The floating point logic that Floats use is
implemented directly in hardware on your computer’s processor already. This
makes it blazingly fast. BigDecimals on the other hand are just a software
implementation to cope with Float’s issues with no direct hardware support. More
on this on [Wikipedia][2].

So whenever it is possible use Floats – that’s why they are default. When you
need precise decimal numbers, e.g., because you are working with currencies, go
for BigDecimal.

If you want to dig a little deeper check out [What Every Computer Scientist
Should Know About Floating-Point Arithmetic][3]. Enjoy ;)

[0]: http://www.rails-troubles.com/2011/12/ruby-float-quirks.html
[1]: http://www.ruby-doc.org/stdlib-1.9.3/libdoc/bigdecimal/rdoc/BigDecimal.html
[2]: http://en.wikipedia.org/wiki/Arbitrary-precision_arithmetic#Implementation_issues
[3]: http://docs.oracle.com/cd/E19957-01/806-3568/ncg_goldberg.html
