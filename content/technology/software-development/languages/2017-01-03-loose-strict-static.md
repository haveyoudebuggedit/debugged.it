---
slug: loose-strict-static
guid: https://pasztor.at/blog/loose-strict-static/
authors:
- janos
categories:
- Technology
date: "2017-01-03T00:00:00Z"
publishDate: "2017-01-03T00:00:00Z"
summary: There seems to be a great deal of confusion in the programming world what
  loose, weak, strict, static and duck typing actually mean. Let's go through it and
  clear a few things up!
tags:
- Software Development
title: The loose, the strict and the static typing
---

If you're reading this, you have probably written a few programs yourself, maybe in languages in loosely typed 
languages like Ruby and PHP, or strictly typed languages like Java or C, C++. But just what is the difference between
them?

## Typing

When you are getting started in programming, you may find typing difficult. For a beginner this piece of code makes 
perfect sense:

```c
int i = 1;
printf("The number is: " + i);
```

However, if you compile this in C, you'll get a warning: 

> warning: adding 'int' to a string does not append to the string

Despite the warning, it will actually compile, but the result will be quite unexpected. Due to how C works, both sides
of the addition will be interpreted as numbers and used as a memory address.

Running a similar code in Ruby will yield us a clear error:

```ruby
i = 1;
//Error: no implicit conversion of Fixnum into String (TypeError)
puts "The number is: " + i;
```

On the other hand, we can do things like this in PHP:

```php
$i = 1;
//Dot is the string concat operator in PHP
print("The number is: " . $i);
```

What's going on here? Let me explain.

### Performance

Almost all modern programming languages have a concept of data types. You can have integers (whole numbers), float 
(decimal numbers with a floating point), strings (text types), etc, so the language knows how to deal with them. But 
why? 

To understand that we have to go back to the high school IT. Remember, the computer stores everything in binary 
numbers. Zeroes and ones. These individual binary numbers are called bits. 0 or 1 is one bit, 00, 01, 10 and 11 are 
two bits, etc.

Now, these bits aren't just floating around inside the computer. Modern computers organize data into blocks of 8 
bits, called a byte. (It used to be 7 for a time.)

That raises the question: how much data can you store on one byte? For one bit, you can have 0 or 1. Two possible 
values. How about 2 bits? 00, 01, 10 and 11. Four values. If you continue the math, you'll end up discovering that 
one byte (8 bits) can hold 256 different values, or 2<sup>8</sup>.

Our modern CPUs are 32 and 64 bits, which means that they can process up to 4 and 8 bytes of data in one instruction.
Everything above that has to be split into multiple instructions.

So far so good? Great. However, these are just numbers. To store text, we need to encode it as numbers. There are 
many different forms of encoding text as numeric values, but the oldest form is
[ASCII](https://en.wikipedia.org/wiki/ASCII). The conversion table contains the letters of the US alphabet, numbers, 
control characters, etc. For example, the letter A is encoded as 65, B is 66, etc. The number 0 is 48, 1 is 49, etc.

So if we encode the number 123456789 as a string, we end up with 9 bytes of data. 9 bytes or 72 bits. Remember, our 
CPU can't deal with that, so we end up with very inefficient code. A more useful representation would be not encoding
the numbers into a string at all, but convert it to binary directly. 123456789 encoded as binary only takes up 26 
bits so that it can be dealt with in one instruction. Not to mention the fact that decoding the data from a string 
representation, then performing an arithmetic instruction and then re-encoding it is a LOT of instructions.

You can hopefully see where this is leading: our high-level programming language needs to know what the data
type is to run at a reasonable speed.

### Making sense of data

Let's set the performance issue aside for just a second and imagine we're storing everything in strings. What would 
this piece of code mean:

```
"123" + "456"
```

Intuitively you'd say the programming language should just add the two numbers, which would yield `579`. But what if 
we wanted to concatenate it to give us `123456`? We'd need a different operator. We'd need to tell the compiler that 
we want these two pieces treated as strings rather than numbers.

In other words, even if we store everything as strings, we can't escape the fact that the compiler needs to know how 
the data should be treated.

## Loose vs. strict typing

Now that we have established that we need types let's take a look at how languages treat types. Taking our number 
example again, what happens if we add two a string and a number?

```
"123" + 456
```

Some languages will give you an error outright because they'll tell you that these are incompatible types and cannot 
be added together. These languages are called strictly or strongly typed. They don't have internal type juggling rules 
so you need to state explicitly what data type you are using.

Other languages will allow you to mix types. These are called loosely or weakly typed. Depending on the languages type 
rules, you'll end up with different results. PHP will recognize that there's a number in the string and treat 
the whole thing as a numeric addition. To achieve a string concatenation, you'll have to use the string 
concatenation operator (dot). JavaScript will treat the addition as a string concatenation and convert the second 
number to a string.

Depending on the language you are programming in, these type juggling rules, make a varying degree of sense. PHP and 
Javascript, for example, are notorious when it comes to these implicit type conversions. One particularly annoying 
example in PHP:

```php
//yields int(579)
var_dump("123a" + 456);
```

While I won't say that weakly typed languages are bad, you have to keep the type juggling in mind when working with
them. They are easier for a beginner to get into, but most weakly typed languages hold various pitfalls when it comes to
type juggling. In the example above, I would expect PHP to throw an error as that particular string should not be
converted to an integer.

## Dynamic vs. static typing

All right, so we have a typing system for our code, but when do we get the error about using 
incompatible types?

There are two approaches to this. Languages like Python, Ruby, and PHP (when you switch it to strict typing) give you 
an error when you run the code, and the actual invalid operation is attempted. We call these languages dynamic, or 
*dynamically typed languages*. These languages usually also allow you to change the type of a variable mid-run.

Other languages like Java or C/C++, on the other hand, give you an error when you compile the code. All execution paths
are analyzed, and invalid type assignments are complained about immediately. These are called *statically typed 
languages*.

Both have their strengths and weaknesses. Dynamic languages make it easier to write and read programs because you 
don't have to litter your code with type declarations. Also, usually these languages are script languages, so there is
no compiling involved in coding, which makes the whole process faster.

{{% tip %}}
**Please note:** dynamic typing does NOT mean that the language is weakly typed! You can have a strongly and 
dynamically typed language!
{{% /tip %}}

Statically typed languages, on the other hand, can catch many bugs at compile time. This is something you can only 
achieve by using code analyzers in the case of dynamic languages (if it is possible at all).

### Duck typing

Many people think that languages like Python or Ruby are actually weakly typed, which is not true. The confusion 
stems from the concept of *duck typing*. That, in turn, comes from the
[old saying](https://en.wikipedia.org/wiki/Duck_test):

> If it looks like a duck, swims like a duck, and quacks like a duck, then it probably is a duck.

In other words, the type check is applied based on *behavior* rather than the type itself. So this would work perfectly:

```ruby
class Duck
  def quack
    puts "Quaaaak"
  end
end

class Person
  def quack
    puts "I'm a duck!"
  end
end

def makeTheDuckQuack(duck)
  duck.quack
end

makeTheDuckQuack Person.new
```

Ruby will behave nicely as long as the object passed to it has a `quack` method. While this behavior makes it easy to
write code, it makes detecting potential bugs with a static code analyzer harder because you don't have a 
declaration of types anywhere, they are evaluated at runtime.

## Which is better?

Remember, the purpose of a programming language is to make it easier for us to describe our wishes to the computer. 
The only valid measurement of how good a programming language is *fitness for purpose*. If you are writing a 
forum software, you probably don't care that much about type. For a finance application, on the other hand, data types 
are essential because you don't want to do financial calculations with floating point numbers. But that's a topic for
 another day.

You *can*, of course, write tests to work around the missing code analysis, but at some point, it may become too 
tedious. For PHP specifically, I would recommend enabling strict typing because the type juggling modes make no sense 
at all and lead bugs through silent errors.
