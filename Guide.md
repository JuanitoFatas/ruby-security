<h1> Ruby Security Reviewer's Guide </h1>
  * Written and maintained by Meder Kydyraliev <[meder.k@gmail.com](mailto:meder.k@gmail.com)>
  * Copyright 2012 Google Inc, rights reserved
  * Released under terms and conditions of the [CC-3.0-BY](http://creativecommons.org/licenses/by/3.0/) license

_Please not that at the moment the guide only covers Ruby, not Rails. For Rails security in general see [Ruby On Rails Security Guide](http://guides.rubyonrails.org/security.html)._

<h1>Who should read this guide?</h1>
<p>This guide is aimed at security reviewers and developers interested in learning more about Ruby and its security properties. It first provides a crash course on the Ruby language, then describes some of the dangers associated with Ruby and their security implications. This guide covers just enough of the basics to bootstrap security reviewers and does NOT provide a comprehensive overview of Ruby. Ruby <code>1.9.3</code> is used throughout this guide unless noted otherwise.</p>
<p>For additional Ruby reading material please see:<br>
<ol><li><a href='http://martinfowler.com/articles/readingRuby.html'>Reading Ruby by M. Fowler</a>
</li><li><a href='http://www.ruby-lang.org/en/documentation/'>http://www.ruby-lang.org/en/documentation/</a>
</li><li><a href='http://www.ruby-doc.org/docs/Tutorial/'>http://www.ruby-doc.org/docs/Tutorial/</a>
</p></li></ol>

<h1> Table of Contents </h1>
<p align='right'>
<i>"Ruby is simple in appearance, but is very complex inside, just like our human body."</i>
</p>
<p align='right'>
<b><i>- Yukihiro "matz" Matsumoto</i></b>
</p>



# A Crash Course Introduction to Ruby #

## Everything is an Object ##
Everything is an Object in Ruby(even `nil`):

```
>> 1337.class
=> Fixnum
>> 1337.even?
=> false
>> 1337.div(10)
=> 133
>> "Ruby".length
=> 4
>> nil.class
=> NilClass
```

and everything can be overridden, but more on that later.

## Variables ##
Variables do not need to be declared and are created whenever you assign a value to them. There are 5 kinds of variables:
  * **Global variables** start with `$`, e.g. <font color='Green'><code>$KUMYS</code></font>
  * **Instance variables** start with `@`, e.g. <font color='Green'><code>@kumys</code></font>
  * **Class variables** (static variables) start with `@@`, e.g. <font color='Green'><code>@@yurt</code></font>
  * **Local variables** start with `_` or a lowercase letter, e.g. <font color='Green'><code>kumys</code>, <code>_kumys</code></font>
  * There are also **class instance variables**, that's right! To read more about them see [Martin Fowler's ClassInstanceVariable](http://martinfowler.com/bliki/ClassInstanceVariable.html), but the basic idea is for each subclass to have its own class variable instead of all subclasses sharing one value.

Constants must start with an uppercase letter and the Ruby interpreter will issue a warning if a constant's value is changed. By convention constants are written in all uppercase characters with underscores separating words.

String interpolation is supported using the `#{...}` construct:

```
puts "Kumys is #{:kumys.length} characters long"
```

If interpolating variables, it is possible to omit curly brackets(`{...}`) if the variable name is qualified, e.g.:
  * `puts "This is my instance variable:`<font color='Green'><code> #@instance_variable</code></font>`"`
  * `puts "This is my global variable:`<font color='Green'><code> #$GLOBAL_VARIABLE</code></font>`"`
  * `puts "This is my class variable:`<font color='Green'><code> #@@class_variable</code></font>`"`

Please note that the examples above use double quotes. Code in single quoted strings is not be interpolated.

## Ruby Symbols ##
A lot of times you will see colon pre-pended identifiers that may look like variable references but they aren't:
```
result = drinks[:kumys]
```

or
```
return :OK
```

`:kumys` and `:OK` are Ruby [Symbols](http://www.ruby-doc.org/core-1.9.3/Symbol.html), which are essentially interned strings. Only one copy of each Symbol resides in memory and it exists for the duration of program execution. See [13 Ways Of Looking At a Ruby Symbol](http://www.randomhacks.net/articles/2007/01/20/13-ways-of-looking-at-a-ruby-symbol) for more information on symbols. Symbols are usually used to implement enumerations or as constants or identifiers. Any String can be converted to Symbol using `.to_sym()` or `.intern()` methods and each `Symbol` can be converted back to `String` using `.to_s()` method. Alternatively, prefixing string literal with `:` will convert the literal to `Symbol`.

_Security implications: Note that there is always only one version of each symbol stored in memory and symbols live there until the program's termination. If the program is converting user-controlled strings into symbols (e.g. HTTP parameters) then an attacker can carry out a denial of service by continuously supplying unique random strings and getting them converted to symbols. Since each unique symbol persists in memory, this will eventually exhaust the program’s memory._


## Blocks, Procs and Lambdas ##
You may have seen code similar to this:

```
foo.each do |element|
  do_something_with(element)
end
```

or

```
foo.each { |element| do_something_with(element) }
```

and wondered what it does. It's actually pretty simple: in both examples above we are passing a chunk of code called a block to a method called `each`, which in turn calls `yield` method to invoke the supplied block. The value between pipes is an argument declaration for our block (multiple arguments are separated by commas) and it is supplied to the `yield` call to get passed to our block. Here's what the `each` method might look like:

```
def each 
  index = 0
  while index < length
    yield arr[index]
    index += 1
  end
end
```

as you can see, our block is invoked using a call to `yield` for each element of the array, and the argument passed to `yield` is passed to our block. In fact that's how Ruby wants you to iterate over various collections. For example, if you want to iterate over keys and values of a `Hash` your block will have 2 arguments, key and value:

```
hash.each do |key, value|
  puts "Key=#{key} Value=#{value}"
end
```


You may have noticed that block isn't specified as an argument anywhere and `yield` is the only way to know that method is expecting a block (or if [block\_given?](http://www.ruby-doc.org/core-1.9.3/Kernel.html#method-i-block_given-3F) is called to check if block was supplied). There is also a way to get the supplied block as an argument:


```
def new_each(&code)
  index = 0
  while index < length
    code.call(arr[index])
    index += 1
  end
end
```


by annotating the last argument with an `&`, we are converting the supplied block to a [Proc](http://www.ruby-doc.org/core-1.9.3/Proc.html), which is essentially a block that you can reference explicitly. Instead of calling `yield` we are now calling the block/proc directly, which makes the code more readable and allows us to pass that block around.


Procs can be saved and passed around just like normal variables:

```
my_proc = Proc.new do |i|
  puts "Got #{i}"
end

my_collect.new_each(my_proc)
```


In addition to Procs there lambdas:

```
my_lambda = lambda { |msg| puts "Lambda says: #{msg}" }
my_lambda.call("Hola!")
my_lambda.call("Hi!", "Would you like some kumys?")
```

The output of the above will be:

```
Lambda says: Hola!
ArgumentError: wrong number of arguments (2 for 1)
```

Lambdas look almost identical to `Proc`s with the following differences:
  * They are declared using the `lambda` keyword instead of using `Proc.new`
  * Unlike Procs, which do not care about the number of arguments (e.g. invoking Proc with incorrect number of arguments will set the missing arguments to `nil` and ignore the rest), lambdas do check the number of arguments and throw `ArgumentError` if number of arguments is incorrect.


See [Understanding Ruby Blocks, Procs and Lambdas](http://www.robertsosinski.com/2008/12/21/understanding-ruby-blocks-procs-and-lambdas/) for more info.

## Classes, Modules & Mixins ##
### Classes ###
Class definition is very straightforward with the `initialize` method being the constructor:

```
class Kumys
  # our constructor
  def initialize(is_the_best)
    @is_the_best = is_the_best
  end

  # Instance method, 'return' keyword is optional.
  # By Ruby coding conventions, methods ending with ?
  # should return boolean
  def is_the_best?
    @is_the_best
  end
end


Kumys.new(true) # creates new instance of our Kumys class
```

It's very common to use the following helper methods to automatically generate instance variable getters/setters:
  * `attr_reader` - generates getter
  * `attr_writer` - generate setter
  * `attr_accessor` - generates setter and getter

```
class Kumys
  attr_accessor :is_the_best
end

k = Kumys.new(true)
k.is_the_best = true
k.is_the_best # returns true
```


### Methods ###
Ruby supports both default argument values and variable-length arguments.

```
def my_method(arg1, arg2 = "Default value")
   ...
end
```

The above code defines 2 arguments and assigns the specified default value to the 2nd argument if it's not supplied during invocation:

```
my_method(1)                   # arg2 is assigned default value
my_method(1, "Another Value")
```

Keep in mind that parentheses are optional during invocation, so the above could be also written as:

```
my_method 1
my_method 1, "Another Value"
```

unfortunately what that means is that if a method doesn't accept any arguments it can be written as:

```
method_with_no_args
```

which makes it hard to tell if it's a local variable reference or a method invocation without looking at the definition.


Variable-length arguments are indicated using a `*` in front of the parameter declaration and results in variable-length arguments bundled into an `Array`:

```
def cook(dish_name, *ingredients) 
  ingredients.each do |ingredient|
    prepare(ingredient)
  end
  ...
end

cook('plov', 'rice', 'carrots', 'onions', 'lamb')
```

all arguments after the 1st argument are passed in the `ingredients` array.


Sometimes it's necessary to expand an `Array` to pass each element as a separate parameter, which is achieved by prefixing the array with `*`:

```
def cook(dish_name, *ingredients) 
  ...
  cut(*ingredients)
  ...
end
```

The `cut` method above receives each element as a separate argument instead of getting an array as an argument.

Another common Ruby pattern is to pass in a `Hash` containing options, e.g.:

```
def cook(dish_name, options = {}) 
  if options[:fry]
    fry options[:ingredients]
  elsif options[:boil]
    boil options[:ingredients]
  end
end

cook('fried rice',
          :fry => true,
          :ingredients => ['rice', 'egg', 'prawns',
                           'peas' 'spring onions])
```

In the above code we've defined `options` argument into which all of the arguments following the `dish_name` argument will be collected. See [Ruby's More About Methods](http://ruby-doc.org/docs/ProgrammingRuby/html/tut_methods.html) for more info.

### Inheritance ###
Inheritance is expressed in a very straightforward way:

```
class SubClass < SuperClass
  def overriden_method(one, two)
    # 'super' without argument will pass all the arguments
    # of the current method to the superclass method
    result = super
    ...
    result
  end
end
```

`SubClass` inherits from `SuperClass` in the above example and overrides the `override_method` method. Class methods covered below are also inherited.

### Class methods ###
Class methods are pretty much what other languages (e.g. Java) call static methods, or at least they are the closest thing to static methods that Ruby has (_they are actually instance methods of a class object on which they are defined._). There are several ways of defining them:


1st variant:
```
class Kumys
  @@total_litres_produced = 0

  def Kumys.make_more(n_of_litres)
    @@total_litres_produced += n_of_litres
    ...
  end
end
```

2nd variant:
```
class Kumys
  @@total_litres_produced = 0

  def self.make_more(n_of_litres)
    @@total_litres_produced += n_of_litres
    ...
  end
end
```

3rd variant:
```
class Kumys
  @@total_litres_produced = 0

  class << self
    def make_more(n_of_litres)
      @@total_litres_produced += n_of_litres
      ...
    end
  end
end
```

All of the above do the same thing and define a class variable `@@total_litres_produced` and a class method called `make_more`, which can be invoked as follows:

```
Kumys.make_more(100)
```

If you are interested in all the gory details about `self` read [Metaprogramming in Ruby: It's All About the Self](http://yehudakatz.com/2009/11/15/metaprogramming-in-ruby-its-all-about-the-self/).


### Modules ###
Modules group methods, constants and class variables. Their definition looks very similar to class definitions except the `class` keyword is replaced with `module`. Unlike classes, modules can't be instantiated or subclassed. There are 2 main use cases for modules: namespaces and mixins.

Using modules as namespaces to avoid name collisions:

```
module MyProduct
  module MySubSystem
    def self.do_something
       ...
    end
  end
end
```

which can be invoked:

```
MyProduct::MySubSystem.do_something
```

There are a couple of things to know about the above code:
  * we've defined a class method in the module (notice the `self` keyword?)
  * you can nest modules and define classes within modules

Ruby doesn't support multiple inheritance, but mixins allow you to mix a module into a class, which essentially achieves the same effect.

```
module UsefulStuff
  def useful
    @useful ||= Useful.new
  end
  ...
end

class SomeClass
  include UsefulStuff

  def do_something
    useful.be_useful
    ...
  end
end
```

In the above code all of the instance methods of `UsefulStuff` are now part of the `SomeClass` class. Whenever `SomeClass` calls `useful` for the first time, a new instance variable `@useful` is created and assigned the value of new `Useful` instance. Note that `||=` is a Ruby idiom, which can be read as "`assign if nil`".


There 2 ways to mixin a module: using [include](http://ruby-doc.org/core-1.9.3/Module.html#method-i-include) or [extend](http://ruby-doc.org/core-1.9.3/Object.html#method-i-extend). And this is where it may get confusing(based on [this](http://stackoverflow.com/questions/156362/what-is-the-difference-between-include-and-extend-in-ruby)):
  * `include` - mixes in module's methods as instance methods in the containing class, so if class `Klazz` `include`s module `Mod`, then all instances of `Klazz` have access to `Mod`'s methods as instance methods
  * `extend` - adds the specified module's methods and constants to the target's metaclass, so `Klazz.extend(Mod)` adds `Mod`'s methods to `Klazz` as class methods and `obj.extend(Mod)` adds `Mod`'s methods as instance methods to `obj`'s metaclass (i.e. other instances of `obj.class` do not have those methods).

Confused yet? Welcome to the world of metaprogramming!


### Method Lookup Algorithm ###
If you are wondering how conflicting methods are handled, here's the algorithm for how Ruby looks up methods (taken from [here](http://blog.rubybestpractices.com/posts/gregory/030-issue-1-method-lookup.html)):
  1. Methods defined in the singleton/metaclass
  1. Modules mixed into singleton/metaclass in reverse order of inclusion
  1. Methods defined by the object's class
  1. Modules included by the object's class in reverse order of inclusion
  1. Methods defined by the object's superclass

The process goes on until `BasicObject` is reached. See Singleton/Eigenclass/Metaclass section for more information on singleton/metaclass.


See [Lookup Algorithm](http://ruby.runpaint.org/methods#lookup-algorithm), [Ruby's method lookup path](http://blog.rubybestpractices.com/posts/gregory/030-issue-1-method-lookup.html) and [the Ruby Method Lookup Flow diagram](http://phrogz.net/RubyLibs/RubyMethodLookupFlow.pdf) for more info on method lookup.

## Tools ##
_This section needs to be expanded_

### IRB ###
`I`nteractive `R`u`B`y gives you an interactive Ruby shell where you can evaluate arbitrary Ruby. It's very useful for quick behaviour verification:

```
$ irb
>> "foo".public_methods
=> [:upcase!, :rpartition, :old_format, :to_str, :ascii_only?, :lstrip, :upto, :getbyte, :lines, :encoding, :sum,
:crypt, :scan, :partition, :reverse!, :==, :clear, :=~, :force_encoding, :each_byte, :tr!, :squeeze!, :inspect, :to_c,
:chop, :next!, :casecmp, :rstrip, :start_with?, :split, :to_f, :succ!, :<, :[]=, :slice!, :valid_encoding?, :center,
:slice, :reverse, :insert, :sub, :tr_s!, :>, :upcase, :next, :strip!, :squeeze, :dump, :count, :sub!, :bytesize, :hash,
:lstrip!, :to_sym, :end_with?, :<=, :replace, :hex, :strip, :capitalize!, :bytes, :setbyte, :chop!, :length, :each_line,
:swapcase, :[], :encode, :include?, :chomp!, :<<, :intern, :gsub, :encode!, :succ, :capitalize, :each_codepoint, :oct,
:delete!, :+, :initialize_copy, :chomp, :rindex, :to_i, :<=>, :eql?, :tr_s, :match, :unpack, :index, :chars, :rstrip!,
:*, :codepoints, :delete, :each_char, :gsub!, :chr, :to_r, :to_s, :rjust, :%, :>=, :empty?, :size, :swapcase!, :concat,
:ord, :tr, :ljust, :downcase, :downcase!, :between?, :__jtrap, :methods, :define_singleton_method, :freeze, :untrust,
:extend, :nil?, :object_id, :protected_methods, :trust, :method, :tainted?, :__id__, :is_a?,
:instance_variable_defined?, :instance_variable_get, :tap, :frozen?, :singleton_class, :respond_to?,
:instance_variable_set, :===, :public_method, :untaint, :respond_to_missing?, :clone, :display, :send, :to_enum,
:private_methods, :enum_for, :singleton_methods, :untrusted?, :type, :public_send, :kind_of?, :dup, :instance_of?,
:taint, :class, :instance_variables, :!~, :public_methods, :instance_exec, :__send__, :instance_eval, :equal?, :!, :!=]
>> "foo".length
=> 3
```


# The Dangerous Ruby Stuff #
This section describes potential Ruby-specific, security-related issues, features and constructs. Keep in mind that while generic security bugs, such as lack of access control, are not be covered in this document, they can still be present in Ruby applications and need to be checked for.

## Developers can override any method! ##
Any method in any class can be overridden in Ruby, e.g. you can easily change the logic behind the `+` method of an `Integer` or override any `String` method. It's also possible to add new methods to core classes, e.g. `String`. Both of the above are usually referred to as "monkey patching" and are heavily used by the Ruby community. That means that even if you are looking at a method that you think will be run, in reality it may not be the case. The fact that, unlike Java, Ruby doesn't enforce any file/directory structure makes it difficult to spot this kind of issues (e.g. `grep`ping will be ineffective in some cases).

Let's look at the following imaginary example, assume the following is in the `money_transfer.rb` file:

```
module MyApp
    class MoneyTransfer
      def transfer(from, to, amount)
        ...
        check_authorization(actor, from, amount)
        ...
      end
    end
end
```

after you've verified the above code for correctness, someone else finds a bug in the audited application that allows an attacker to transfer money from arbitrary accounts. You are confident that you saw the authorization check in the code, but after an investigation you realize that there was another file called `new_money_transfer.rb` in a different directory, which had authorization check refactored out:

```
module MyApp
    class MoneyTransfer
      def transfer(from, to, amount)
        ...
      end
    end
end
```

the above code effectively overrides the original method and you need to look out for that when performing security audits.

Sometimes you may also see code like:

```
input = "this is just a string"
...
input.encrypt(@key)
```


which calls `encrypt()` method on a `String`. However, since `String` documentation doesn't have any references to `encrypt()`, then what you are looking at is a library or application code monkey patching `encrypt` method, either into the `String` class or into object's metaclass (see Metaprogramming section for more info). You are most likely to find a reference to `String` class somewhere e.g.:

```
class String
  def encrypt(key)
    …
  end
end
```

**!Ruby Hacker Tip!** _During security review you could potentially override Ruby-provided hooks (e.g. `inherited` or `extended`) to get more insights into the events taking place at runtime._

## `send()` - invoking methods dynamically ##
One of the widely used features in Ruby is dynamic method invocation using `send()`. The following examples illustrate how [send()](http://ruby-doc.org/core-1.9.3/Object.html#method-i-send) works. The first argument to `send()` is the name of the method to invoke and the rest of the argument is passed to that method as arguments:

```
>> "kumys".send(:length)
=> 5
>> "kumys".send("length")
=> 5
```

the above code dynamically invokes the `length` method on the string "`kumys`". As you can see above, `send()` accepts both strings("`length`") and symbols(`:length`).

```
>> "kumys".send("concat", " rules!")
=> "kumys rules!"
```

the code above invokes the `concat` method and supplies it with an argument ("` rules!`").

```
>> "kumys".send("send", "send", :send, "length")
=> 5
```

the above code invokes `send` 4 times with the last `send` invoking the `length` method.

It is also possible to test whether an object has a method defined by invoking the [respond\_to?](http://ruby-doc.org/core-1.9.3/Object.html#method-i-respond_to-3F) method:

```
>> "foo".respond_to?("got_kumys?")
=> false
>> "foo".respond_to?("length")
=> true
>> "foo".respond_to?("eval")
=> false
>> "foo".respond_to?("eval", true) # true forces private methods lookup
=> true
```

As you can see, `send` can be used to implement dynamic method dispatching.

Another frequent use case is to dynamically assign attributes:

```
class Person
  attr_accessor :name, :age, :address
end
```

in the above case the following code can be used to assign attributes:

```
p = Person.new
p.send("name=", "Foo Baroff")
p.send("address=", "Bazistan")
```

as you've already realized it can also be done dynamically based on user input(e.g. HTTP parameters):

```
p = Person.new()
params.each do |key, value|
  p.send("#{key}=", value)
end
```

the above code appends `=` to the user-supplied value, thus guaranteeing that only methods whose names end with `=` can be invoked. For example, supplying `foo=bar` in HTTP parameters results in the following code executed:

```
p.send("foo=", "bar")
```

and that's, for example, exactly how Rails assigns values to model attributes.

`send()` can invoke both public and _private_ methods.

At this point you are probably realizing that invoking arbitrary methods on an object based on user input is probably a bad idea, though in the example above it is only possible to invoke methods that end with `=`. You may wonder if there are any sensitive methods that end with `=` and the answer would depend on the object `send` is being invoked on.

**!Ruby Hacker Tip!** _To list all public and private methods, use `public_methods` and `private_methods` methods respectively:_

```
>> "foo".private_methods.sort
=> [:Array, :Complex, :Float, :Integer, :Rational, :String, :__callee__, :__method__, :_exec_internal, :`, :abort,
:at_exit, :autoload, :autoload?, :binding, :block_given?, :callcc, :caller, :catch, :default_src_encoding, :eval, :exec,
:exit, :exit!, :fail, :fork, :format, :gem, :gem_original_require, :getc, :gets, :global_variables, :initialize,
:irb_binding, :iterator?, :lambda, :load, :local_variables, :loop, :method_missing, :open, :p, :print, :printf, :proc,
:putc, :puts, :raise, :rand, :readline, :readlines, :remove_instance_variable, :require, :require_relative, :select,
:set_trace_func, :singleton_method_added, :singleton_method_removed, :singleton_method_undefined, :sleep, :spawn,
:sprintf, :srand, :syscall, :system, :test, :throw, :trace_var, :trap, :untrace_var, :warn] 

>> "foo".public_methods.sort
=> [:!, :!=, :!~, :%, :*, :+, :<, :<<, :<=, :<=>, :==, :===, :=~, :>, :>=, :[], :[]=, :__id__, :__jtrap, :__send__,
:ascii_only?, :between?, :bytes, :bytesize, :capitalize, :capitalize!, :casecmp, :center, :chars, :chomp, :chomp!,
:chop, :chop!, :chr, :class, :clear, :clone, :codepoints, :concat, :count, :crypt, :define_singleton_method, :delete,
:delete!, :display, :downcase, :downcase!, :dump, :dup, :each_byte, :each_char, :each_codepoint, :each_line, :empty?,
:encode, :encode!, :encoding, :end_with?, :enum_for, :eql?, :equal?, :extend, :force_encoding, :freeze, :frozen?,
:getbyte, :gsub, :gsub!, :hash, :hex, :include?, :index, :initialize_copy, :insert, :inspect, :instance_eval,
:instance_exec, :instance_of?, :instance_variable_defined?, :instance_variable_get, :instance_variable_set,
:instance_variables, :intern, :is_a?, :kind_of?, :length, :lines, :ljust, :lstrip, :lstrip!, :match, :method, :methods,
:next, :next!, :nil?, :object_id, :oct, :old_format, :ord, :partition, :private_methods, :protected_methods,
:public_method, :public_methods, :public_send, :replace, :respond_to?, :respond_to_missing?, :reverse, :reverse!,
:rindex, :rjust, :rpartition, :rstrip, :rstrip!, :scan, :send, :setbyte, :singleton_class, :singleton_methods, :size,
:slice, :slice!, :split, :squeeze, :squeeze!, :start_with?, :strip, :strip!, :sub, :sub!, :succ, :succ!, :sum,
:swapcase, :swapcase!, :taint, :tainted?, :tap, :to_c, :to_enum, :to_f, :to_i, :to_r, :to_s, :to_str, :to_sym, :tr,
:tr!, :tr_s, :tr_s!, :trust, :type, :unpack, :untaint, :untrust, :untrusted?, :upcase, :upcase!, :upto,
:valid_encoding?]
>>
```

So if the above send code was invoked on an instance of a `String`, we would be able to execute the following methods:

```
==
[]=
<=
>=
===
!=
```

**!Ruby Hacker Tip!** _You may wonder if it's possible to neutralize the `=` in the above send code by injecting special characters (e.g. NUL) in front of it. Unfortunately it appears to not be possible, but you are more than welcome to try it out on various Ruby implementations._

Attentive readers may have noticed that the array of private methods above contains some interesting entries:

```
exec
system
eval
throw
puts
exit
fork
syscall
```

and may wonder why are these methods part of the `String` class? Remember that everything in Ruby is an object? That means that it's impossible to have a standalone function because it must belong to a class or a module. At the same time developers want to be able to invoke certain methods without qualifying them with a module name thus simulating standalone functions, e.g.:

```
...
def my_method
  puts "Hello World!"
end
```

`puts` isn't defined in our class, but it's part of every object and the way it is accomplished in Ruby is by mixing in [Kernel](http://ruby-doc.org/core-1.9.3/Kernel.html) module into [Object](http://ruby-doc.org/core-1.9.3/Object.html) class, which means that all `Kernel` methods (e.g. `exec`, `eval`, `syscall`) become private methods of every single object in Ruby. Thus the following is possible:

```
>> 1.send(:exit)
```

which will terminate Ruby runtime, or

```
>> "Hello pwnage!".send("exec", "mkdir /tmp/PWNED")
```

which will execute the `mkdir /tmp/PWNED` command, or the following, which achieves the same effect, without replacing the current process:

```
>> "hello".send("eval", "`mkdir /tmp/PWNED`")
```

As you can see, `send` can be extremely dangerous if its first argument is attacker controlled and can result in remote code execution if the user controls the first two arguments (method name and argument).

There's a public variant of `send` called [public\_send](http://ruby-doc.org/core-1.9.3/Object.html#method-i-public_send), which will only invoke public methods (remember that `eval`, `exec`, `syscall`, etc are all private), but does that mean it's safe to use it? Unfortunately, the answer is _NO_, because `send` itself is a public method and hence it's possible to do the following:

```
>> "hello".public_send("send", "eval", "`mkdir /tmp/PWNED`")
```

`public_send` invokes `send`, which in turn invokes `eval`.


Security reviewers should pay extra attention to all `__send__()`, `send()` and `public_send()` method calls and make sure that an attacker does not control the first argument:
  * if attacker controls ONLY the first argument then it's possible to at least invoke `exit` method thus shutting down the application
  * if attacker controls first and second arguments then the application is vulnerable to remote code execution since the attacker can invoke a number of methods to evaluate her code (e.g. `eval`)

Additionally, if the first argument is partially controlled (e.g. user input is prefixed or suffixed with a constant) then the reviewer should ensure that no unintended method can be invoked.


## `method_missing()` - implementing functionality on the fly ##
Ruby provides a hook to be invoked whenever a nonexistent method is invoked on an object. Consider the following code:

```
class ClassWithNoMethods
  def method_missing(sym, *args, &block)
    puts "#{sym} method was called"
  end
end
```

let's examine its behaviour:

```
>> c = ClassWithNoMethods.new
 => #<ClassWithNoMethods:0x1b6956f> 
>> c.got_kumys?
got_kumys? method was called
```

as you can see, since the method didn't exist, `method_missing` hook was executed. So what can this functionality be used for? The most popular use case is to dynamically simulate methods based on method names being invoked. A great example of that is the way ActiveRecord implements its finder methods (_simplified code_):

```
module ActiveRecord
  class Base

    def method_missing(method_id, *arguments, &block)
      case method_id.to_s
        when /^find_(all_|last_)?by_([_a-zA-Z]\w*)$/
          finder = :last if $1 == 'last_'
          finder = :all if $1 == 'all_'
          names = $2
        when /^find_by_([_a-zA-Z]\w*)\!$/
          bang = true
          names = $1
        when /^find_or_create_by_([_a-zA-Z]\w*)\!$/
          bang = true
          instantiator = :create
          names = $1
        when /^find_or_(initialize|create)_by_([_a-zA-Z]\w*)$/
          instantiator = $1 == 'initialize' ? :new : :create
          names = $2
        else
          return nil
        end
      ...
    end
  end
end
```

the above code enables users to query their models based on any attribute by executing methods with the right prefix, e.g.:

```
User.find_by_email("user@domain.com")
User.find_all_by_first_name("Vasya")
User.find_or_create_by_email("user@domain.com")
```

all of which will be dispatched to the `method_missing` hook since methods don't exist, which in turn parses the name of the method being invoked and carries out appropriate action.

Another use case for `method_missing` hook is delegation, when only some methods are defined and the rest are delegated to another object. See [this example on GitHub](https://github.com/rails/rails/blob/master/actionpack/lib/action_dispatch/middleware/cookies.rb) for an example.


`method_missing` should be considered together with `send()` especially in cases where the first argument is partially controlled since it is possible to trigger the `method_missing` hook and get it to carry out an interesting action. Let's consider the following contrived example as an illustration of a potential issue:

```
def set_fill(fill_type, argument) 
  send("fill_#{fill_type}", argument)
end

def set_stroke(stroke_type, argument)
  send("stroke_#{stroke_type}", argument)
end

def method_missing(id, *args, &block)
 case(id.to_s)
 when /^fill_and_stroke_(.*)/
   send($1,*args,&block); fill_and_stroke
 when /^stroke_(.*)/
   send($1,*args,&block); stroke
 when /^fill_(.*)/
   send($1,*args,&block); fill
 else
   super
 end
end
```

In the above `method_missing` code the method name(`id`) is matched against regular expressions with groups, and matching group(`$1`) is then used as a method name to invoke via `send()`. Let's assume that arguments come from HTTP parameters (they could also come from a user supplied document) and requesting `/fill?color=red` would result in the following code being run:

```
http_params.each do |name, value|
  set_fill(name, value)
end
```


It's hopefully clear at this point that HTTP parameter name effectively controls which method is invoked and the HTTP parameter value is its argument. To gain full remote code execution the attacker would have to submit:

```
/fill?eval=`mkdir /tmp/PWND`
```

which would result in the following methods execution sequence:

```
set_fill("eval", "`mkdir /tmp/PWND`")
send("fill_eval", "`mkdir /tmp/PWND`")
method_missing("fill_eval", "`mkdir /tmp/PWND`", nil)
send("eval", "`mkdir /tmp/PWND`")
```

While the example above is a bit contrived, it is intended to demonstrate the kind of issues that can arise from the combination of `send()` and `method_missing()`. During security reviews all `method_missing()` implementations should be audited for insecure patterns.


## Metaprogramming ##
One of the reasons why a lot of people love but also hate Ruby is metaprogramming. The subject of metaprogramming is a vast one and covering it in a small section is tricky, but it doesn't hurt to try.

First of all you may wonder why would anyone want to dynamically create classes or define methods? Laziness! Metaprogramming allows users of the dynamic code to avoid writing a lot of boilerplate or complex code and instead have that code automagically generated. Earlier you've seen the `attr_accessor` method, which will generate setters and getters for supplied attributes. That's a very good example of metaprogramming, which makes code more readable, as long as you aren't concerned with implementation details and (un)fortunately, being a security reviewer, you are!

Auditing source code that heavily uses metaprogramming (e.g. Rails) can be a tricky exercise and this section aims to equip you with the necessary knowledge to do so effectively.


### Metaprogramming Basics ###
Let's look at the following code:

```
GREETINGS = {'english' => 'Hello!',
             'spanish' => '¡Hola!'}

class Greeting
  GREETINGS.each do |language, greeting|
      define_method("greet_in_#{language}") do
        puts greeting
      end
  end
end

g = Greeting.new
g.greet_in_english # outputs Hello!
g.greet_in_spanish # outputs ¡Hola!
```

The code above dynamically creates new methods on the `Greeting` class based on the values of the `GREETING` hash. Another way of writing this would be using [class\_eval](http://www.ruby-doc.org/core-1.9.3/Module.html#method-i-class_eval) (instead of the [define\_method](http://www.ruby-doc.org/core-1.9.3/Module.html#method-i-define_method)), which evaluates a string or a block in class context (`Greeting` class in this case):

```
class Greeting
  GREETINGS.each do |language, greeting|
    class_eval <<-EOT, __FILE__, __LINE__
      def greet_in_#{language}
        puts "#{greeting}"
      end
    EOT
  end
end
```

Such dynamic method generation can also be implemented as a method, which eventually allows for "DSL"(Domain Specific Language) tricks, i.e. when the code looks like data and/or reads like text. The above could be rewritten as:

```
module GreetingGenerator
  def greet(language, greeting)
    define_method("greet_in_#{language}") do
      puts greeting
    end
  end
end

class Greeting
  extend GreetingGenerator

  greet "english", "Hello!"
  greet "spanish", "¡Hola!"
end
```

This kind of approach is used heavily in Rails, in particular in ActiveRecord models to implement a lot of magic tricks, which are one of the reasons for Rails' popularity:

```
class Customer < ActiveRecord::Base
  has_many :orders, :dependent => :destroy
end
 
class Order < ActiveRecord::Base
  belongs_to :customer
end
```

`has_many` and `belongs_to` will dynamically add methods to the corresponding classes behind the scenes.


### Singleton/Eigenclass/Metaclass/Object-specific Class ###
A [lot](http://www.klankboomklang.com/2007/10/05/the-metaclass/) [has](http://yehudakatz.com/2009/11/15/metaprogramming-in-ruby-its-all-about-the-self/) [been](http://www.madebydna.com/all/code/2011/06/24/eigenclasses-demystified.html) [written](http://ola-bini.blogspot.com.au/2006/09/ruby-singleton-class.html) on this subject so you are advised to read up on that, but the easiest (in the author's humble opinion) way to understand the concept of a metaclass (which also known as singleton or Eigenclass) is to remember that whenever you create an instance of a class that newly created object will contain its own methods table, while still maintaining a reference to the method table of the class it was created from. And the object's own methods table can be modified (e.g. methods added) without affecting the original class. To illustrate let's consider the following example:

```
>> my_str = "Got kumys?"
>> def my_str.do_something
>>   puts "doing stuff..."
>> end
>> my_str.do_something
doing stuff...
>> another_str = "Got Ruby?"
>> another_str.do_something
NoMethodError: undefined method `do_something' for "Got Ruby?":String
```

What happened? We've defined `do_something` method on `my_str` _metaclass_ and since `another_str` is a new `String` class instance, these changes will not affect it, since it has its own metaclass. Another way of writing the above would be:

```
my_str = "Got kumys?"
class << my_str
  def do_something
    puts "doing stuff..."
  end
end
```

or

```
my_str = "Got kumys?"
my_str.instance_eval do
  def do_something
    puts "doing stuff..."
  end
end
```

That's about all you need to know about metaclasses! But if you must go deeper, classes you define in Ruby are instances of the `Class` class and hence all methods that you define on a class are actually defined on a metaclass!

What if you wanted to define a method on the `String` class itself, instead of defining it on a metaclass?

```
my_str = "Got kumys?"

my_str.class.class_eval do
  def do_something
    puts "doing stuff..."
  end
end

another_str = "Got Ruby?"
another_str.do_something  # prints "doing stuff..."
```

In the code above `class_eval` evaluates supplied block in the context of the `class` (`String`) and adds the `do_something` instance method, which is available to all instances of `String`.

Metaprogramming can make source code auditing significantly harder; having a good understanding of some of the common techniques used in the Ruby community can potentially make the process less frustrating.

## Files ##
Just like with other languages, care should be taken when carrying out file operations with paths containing user-controlled data to avoid path traversal and arbitrary file reading/writing/execution vulnerabilities.

Ruby provides a [variety of file APIs](http://www.ruby-doc.org/core-1.9.3/File.html) to resolve relative paths, etc.

One interesting discrepancy in the way MRI 1.8 is behaving in comparison to MRI 1.9 and JRuby is `NUL` character handling:
  * Ruby 1.8 (MRI):
```
$ ruby -e 'p File.new("/etc/hosts\0").path'
-e:1:in `initialize': string contains null byte (ArgumentError)
```

  * Ruby 1.9.3 (MRI):
```
$ ruby -e 'p File.new("/etc/hosts\0").path'
"/etc/hosts\u0000"
```

  * JRuby 1.6.7.2 ([JRUBY-6247](https://jira.codehaus.org/browse/JRUBY-6247)):
```
$ ruby -e 'p File.new("/etc/hosts\0").path'
"/etc/hosts\u0000"
```

as you can see, `NUL` character injection works in Ruby 1.9 and JRuby so if an attacker controls only the middle part of the path (e.g. the file extension is appended to user input) it is be possible to use `NUL` to neutralize everything that follows it and carry out arbitrary file reading/writing attacks depending on vulnerable code. The following code displays the first line of `/etc/passwd` in 1.9 and JRuby:

```
p File.new("/etc/passwd\0Got Kumys?").gets
```

## Regular Expressions ##
A very common mistake that has been [documented](http://guides.rubyonrails.org/security.html#regular-expressions) before is the difference in what `^` and `$` mean in Ruby compared to other languages. Consider the following code:
```
input = get_http_paramter('locale')

if (input =~ /^[\w]+$/) # only allow word characters
  // valid input
  display_file_contents(File.new(input))
else
  # invalid input
  return INVALID
end
```

while it appears to be correct since regular expression is anchored, it turns out that in Ruby `^` represents **line** beginning and `$` **line** end. So _any_ line in the `input` matching the regular expression results in it returning `true`.

To exploit the code above, the attacker would have to submit the following HTTP parameter:
```
locale=/etc/passwd%00%0ainnocent
```

which when decoded by the web server will look like:
```
/etc/passwd\0
innocent
```

the second line (`innocent`) matches the regular expression and `File` happily stops processing value of `input` after hitting `NUL`(`\0`). This results in `display_file_contents` displaying `/etc/passwd` instead of `innocent` (_note that we are relying on the `NUL` behaviour documented above)._

To achieve the desired effect in the above code anchors need to be changed to `\A` and `\z` respectively:
```
input = get_http_paramter('locale')

if (input =~ /\A[\w]+\z/) # only allow word characters
  // valid input
  display_file_contents(File.new(input))
else
  # invalid input
  return INVALID
end
```


## Good ol' shell injection ##
Ruby provides a variety of methods for developers to execute shell commands:
  * Backticks (e.g. ``````ping www.google.com``````), just like a lot of other languages. If any part of the string is attacker-controlled, it results in shell injection.
  * An interesting variant of the above, which achieves the same effect is `%x`. All of the examples below do the same thing:
```
%x( ping www.google.com )
%x[ ping www.google.com ]
%x{ ping www.google.com }
%x< ping www.google.com >
%x/ ping www.google.com /
%x| ping www.google.com | 
```
> as you can tell, any delimiter character can be used after `%x`.

  * `IO.popen()` whose first argument can either be a `String` or an `Array` of `String`s:
    * If the argument is a `String` then it contains the full command including arguments and any unescaped user controlled values in that string results in shell injection:
```
IO.popen("ls -al /tmp; mkdir /tmp/PWNED") do |io|
  puts io.read
end
```
    * If the argument is an `Array` then the shell is bypassed and the array the the `argv` of the subprocess:
```
IO.popen(['ls', '-al', '/tmp && mkdir /tmp/PWNED']) do |io|
  puts io.read
end
```
> > Obviously, the `Array` variant should be the one used since it removes the need to perform shell escaping.
  * All methods defined in the [Open3](http://www.ruby-doc.org/stdlib-1.9.3/libdoc/open3/rdoc/Open3.html) module (e.g. `popen3`, `pipeline`, `capture3`, etc).
  * `Kernel.exec()` replaces the current process by running the specified command. The method has somewhat complex rules:
    * If one `String` is given, then the argument is shell expanded before execution. Controlling any part of that argument results in shell injection.
    * If the first argument is a 2-element Array, then the 1st element is the program to execute and the 2nd element is the `argv[0]`, which shows up in the process listing:
```
Kernel.exec(['sleep', 'kumys'], '10')
```
> > will execute `sleep` and pass `10` as a command line argument with the process showing up as '`kumys`' in the process listing. No shell processing is done on the arguments.
> > _Interestingly JRuby's behaviour is incorrect and isn't compliant with the documentation and the above code will result in:_
```
Errno::ENOENT: No such file or directory - kumys
```
> > Swapping the elements around results in execution, however, unexpectedly arguments are passed through shell in JRuby:
```
Kernel.exec(['a', 'sleep;` mkdir /tmp/PWNED`'])
```
> > whereas in MRI the following error is returned (after swapping arguments):
```
Kernel.exec(['sleep;` mkdir /tmp/PWNED`', 'a'])
Errno::ENOENT: No such file or directory - sleep;` mkdir /tmp/PWNED`
```
    * If two or more String arguments are passed then the first is used as the command name and the rest are passed as parameters without shell processing.
  * `Kernel.system()` behaves the same way as `Kernel.exec()`: if a single `String` is passed and part of it is user controlled it results in shell injection.
  * `Kernel.open()`, surprisingly, supports command execution if the path starts with `|`, e.g.:
```
open("|date") do |cmd|
  print cmd.gets
end
```

> will execute `date` command and print its output. If any part of the `open()` argument is controlled by an attacker and is not escaped it results in shell injection, e.g.:
```
open ("|ls -al; mkdir /tmp/PWNED")
```
> Additionally if the `open()` call was intended to open a file and not execute commands and the start of the string is user-controlled, it should be possible to get it to execute commands by supplying `|` as the first character.

> Keep in mind that every object in Ruby includes the `Kernel` module, which means that in the code you will see calls to `Kernel`'s methods without the module qualifier, e.g.:
```
open("|date")
system("ls -al")
```

As demonstrated above, if possible, it's strongly recommended to use an array or multiple strings to ensure no shell processing occurs. If, however, it's not possible then [Shellwords.escape()](http://www.ruby-doc.org/stdlib-1.9.3/libdoc/shellwords/rdoc/Shellwords.html#method-c-escape) should be used on user-controlled values.

Auditing all calls to invoke external commands should always be carried out during security reviews.


## Serialization/Marshaling ##
Ruby allows you to marshal objects to a byte stream and back using [Marshal.dump()](http://www.ruby-doc.org/core-1.9.3/Marshal.html#method-c-dump) and [Marshal.load()](http://www.ruby-doc.org/core-1.9.3/Marshal.html#method-c-load) respectively. Only data (no code) is marshaled. Besides DoS bugs (e.g. array allocations based on user-supplied values), the author isn't aware of any serious vulnerabilities associated with unmarshaling untrusted data (_what that really means is that this area needs more research_), of course with the exception of application-specific vulnerabilities (e.g. trusting deserialized values in privileged operations).

One area that needs further investigation is support for module extension:

```
module Innocent
  def action
    puts "In innocent..."
  end
end

module Evil
  def action
    puts "In evil..."
  end
end

Marshal.load("\004\be:\tEvilm\rInnocent")
```

The above call is an equivalent of:
```
Evil.extend_object(Innocent)
```

and calling `Innocent.action` returns "`In evil...`". However, no exploitable condition was identified.

Overall, to be on the safe side, applications shouldn't deserialize untrusted data if possible. Security reviewers should check applications for applications-specific serialization bugs.

# References #
See HTML links scattered in this document.

# Acknowledgements #
The following people have assisted in reviewing this guide and providing valuable feedback:
  * Ilya Grigorik
  * Patrick Toomey
  * Felix Gröbert
  * Alexander Konstantinou
  * Dmitry Ratnikov
  * Michael Muller