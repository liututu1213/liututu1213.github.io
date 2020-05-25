---
title: Ruby基础
category: Ruby
abbrlink: 6648
date: 2020-01-18 00:56:10
tags:
---


###  1. Ruby 面向对象概览

#### 1.1 基础变量

```ruby
x = 1
y = 1.0

puts x.class
puts y.class
```

打印结果：

```
Integer
Float
```

在Ruby中，变量可以包含Ruby可以理解的任何概念，==在Ruby中所有的元素，包括基本类型都是对象，这个内置类有定义好的方法来执行数字类的操作==，同时也不存在==运算符==的概念，所谓的`1 + 1`，其实只不过是`1. + (1)`的语法糖而已。



####  1.2 Ruby 对象和类

##### 1.2.1 对象

Ruby是一个面向对象的编程语言

```ruby
class Person
	attr_accessor :name, :age, :gender
end

p = Person.new
p.name = "小明"
p.age = 10
p.gender = "woman"

puts "#{ p.name }"
```

`attr_accessor`创建了三个属性(attributes)，其中accessor表示让这些属性可访问，可被修改和设置，除此之外，还有attr_reader`表示只可读，`attr_writer`表示只可写入。



#####  1.2.2 继承

面向对象编程的特点之一就是：继承。

```ruby
class Pet
	attr_accessor :name, :age, :color
end

class Cat <Pet
end

class Dog <Pet
	def bark
		puts "Woof !"
	end
end

class Snake <Pet
	attr_accessor :length
end
```



#### 1.2.3 Kernel模块的方法

Kernel是一个特殊的类，前面使用到的`puts`方法没有完整的类或者对象的前缀，实际上，`puts`是来自Kernel模块的方法，当调用`puts`时，Ruby发现没有指定任何类或对象，就在默认的、预定义的类和模块中查到相同名的方法，在Kernel模块中进行查找，并进行调用，也可以显式地调用`Kernel.puts "hi"`。



### 2. Ruby的构造元素：数据、表达式和流程控制

#### 2.1 数字与表达式

##### 2.1.1 比较运算符和表达式

| 比较运算 |                           含义                           |
| :------: | :------------------------------------------------------: |
|  x >  y  |                           大于                           |
|  x < y   |                           小于                           |
|  x == y  |                           等于                           |
|  x >= y  |                         大于等于                         |
|  x <= y  |                         小于等于                         |
| x <=> y  | 比较，如果x==y则返回0，x大于y则返回1，如果x小于y则返回-1 |
|  x != y  |                          不等于                          |



```ruby
x = 10 
y = 3
puts x.to_f / y.to_f

z = 3.845
puts z.to_i
puts z.round

f = 3.9
puts f.floor

c = 10.1
puts c.ceil
```

打印结果：

```
3.3333333333333335
3
3.85
3
11
```

Ruby中提供函数用于浮点数（Float）和整数（Integer）之间的转换。`to_i`会直接去掉小数点，`round`为四舍五入，`floor`返回比接受者小的最大整数，`ceil`返回比接受者大的最大整数



##### 2.1.2 常量

常量在Ruby中的表达形式是以大写字母开头的变量名。

```ruby
Pi = 3.141592
Pi = 3
```

运行结果：

```
constant.rb:2: warning: already initialized constant Pi
constant.rb:1: warning: previous definition of Pi was here
```

对于常量的值，Ruby提供了完全的控制权，在前面定义的Dog等class 以大写字母开头的名字来引用类，是因为类一旦定义之后，就是程序的固化部分，与常量的性质类似。



#### 2.2 文本与字符串

##### 2.2.1 字面字符串

在Ruby中所有的字符串都是String类的对象，`"hello"`这种构造的字符串称为字面字符串。

把字符串包含在程序中有许多方式，如用引号只适合于单行文本，如果是多行文本则可以使用如下的方式：

```ruby
x = %q{this is string  
of the multi line capabilities}
puts x
```

在上面的例子中，引号被替换成`%q{}`,并且也可以用`<>`、`()`或其他自选的两个分解符代替引号。

##### 2.2.2 字符串表达式

|            表达式            |   含义   |
| :--------------------------: | :------: |
|          "a" + "b"           |   "ab"   |
|          "abc" * 2           | "abcabc" |
| puts "x" > "y"      => false | 大小比较 |



##### 2.2.3 字符串 占位符

```ruby
x = 10
y = 20
puts "#{x} + #{y} = #{x+y}"
```

插写一段重复的字符串，可以使用以下方式：

```ruby
puts "it's a #{"bad " * 5} world"
```



##### 2.2.4 字符串方法

对字符串"Test"调用不同的方法的结果

|        表达式        |        结果         |
| :------------------: | :-----------------: |
|  "Test".capitalize   |        Test         |
| puts "Test".downcase |        Test         |
|     "Test".chop      |         Tes         |
|     "Test".hash      | 3285097822602877765 |
|     "Test".next      |        Tesu         |
|    "Test".reverse    |        tseT         |
|      "Test".sum      |         416         |
|   "Test".swapcase    |        tEST         |
|    "Test".upcase     |        TEST         |



##### 2.2.5 正则表达式与字符操作

```ruby
puts "foobar".sub("bar", "foo")
```

`sub`将第一个参数替换成第二个参数。

```ruby
puts "this is a test string".gsub("i", "")
```

`gsub`方法会对所有匹配文本进行多次替换。

```ruby
x = "this is a test"
puts x.sub(/^../, "Hello")
puts x.sub(/..$/, "Hello")
```

运行结果：

```
Hellois is a test
this is a teHello
```

在正则表达式中`^`，称为锚，表示正则表达式将从字符串中的任一行开头进行匹配，`..`中每一个点表示”任何字符“，整个表达式表达的是某行开始的开头两个字符，`$`为另一种锚，表示行尾。



#### 2.3 数组与列表

##### 2.3.1 基本数组

在Ruby中，可以用数组(array)来表示对象的有序集合。

```ruby
x = [1, 2, 3, 4]
x[2] = "Fish" * 3
x << "word"
x.push(5)
puts x
```

对于数组来说，`<<`就是把数据放到数组末尾的运算符，也可以调用`push`,具有相同的效果，`x.pop`表示一处末尾元素。

```ruby
x = [1, 2, 3, 4]
puts x.join(",")
```

`join`表示把数组元素以指定的字符串连接起来。

```ruby
x = "this is a test"
puts x.split(" ")
```

`split`表示将字符串以特定的字符串，分割成字符串数组。



##### 2.3.2 数组迭代

```ruby
x = [1, "test", 2, 3, 4]
x.each {|element| puts element.to_s + "X"}

y = x.collect {|element| element * 2}
puts y
```

`each` 方法遍历数组的每个元素，`collect`方法对数组进行实时转换。

```ruby
x = [1, 2, 3, 4]
y = [1, 2]
z = x + y
puts x.empty?
puts x.first
puts x.last
puts x.reverse.inspect
```

运行结果：

```
false
1
4
[4, 3, 2, 1]
```

```ruby
x = [1, 2, 3, 4]
y = [1, 2]
z = x + y
puts z.uniq.inspect
```

打印结果：

```
[1, 2, 3, 4]
```

`uniq`这个方法是用来移除重复的元素。

#### 2.4 散列表

数组是对象的集合，散列表亦是。散列表中的对象不在列表给定位置，而是给定一个指向对象的键(key)。

```ruby
dictionary = {'cat' => 'feline animal', 'dog' => 'canine animal'}
puts dictionary
puts dictionary.size
```

运行结果：

```
{"cat"=>"feline animal", "dog"=>"canine animal"}
2
```

##### 2.4.1 散列表的基础方法

获取散列表元素信息

```ruby
x = {'a' => 1, 'b' => 2}
x.each { |key, value|
	puts "#{key} equals #{value}"
}
puts "keys: #{x.keys.inspect} \nvalue: #{x.values.inspect}"
```

删除元素

```ruby
x = {'a' => 1, 'b' => 2}
x.delete("a")
```

 有条件地删除元素

```ruby
x = {'a' => 1, 'b' => 2}
x.delete_if{|key, value| value < 2}
puts x.inspect
```



#### 2.5 流程控制

##### 2.5.1 if与unless

```ruby
age = 10
puts "you are too young to use this system" if age < 18
if age < 18 
	puts "you are too young to use this system" 
end
```

上面这两种方式功能等同。

```ruby
age = 10
puts "so we are going to exit your program now" unless age > 18
```

`unless` 是`if`的反义词。

##### 2.5.2 elsif 与case

```ruby
color = "orange"
if color == "orange"
	puts "i like this color"
elsif color == 'green'
	puts "i not like"
else 
	puts "normal"
end
```

`case`块首先处理表达式，然后找出匹配该表达式的`when`块并执行，如果匹配不到`when`则执行`case`中的`else`块

```ruby
case color
when "orange"
	puts "i like this color"
when "green"
	puts "i not like"
else
	puts "normal"	
end
```

##### 2.5.3 代码块

```ruby
def each_vowel(&code_block)
	%w{a e i o u}.each { |vowel| code_block.call(vowel)}
end

each_vowel { |vowel|
	puts vowel
}
```

`each_vowel`接受代码块作为参数，`&`表示该参数是代码块，`call`执行传递进来的代码块。

```ruby
def each_vowel(&code_block)
	%w{a e i o u}.each { |vowel| yield vowel}
end
```

`yield`会自动检测传递给它的代码块，并将控制权移交给该代码块。

ps：一次只能传递一个代码块，任何方法都不可能接受两个及以上的代码块作为参数，但代码块本身可以接受多个参数。

```ruby
print_param = lambda { |x| puts x}
print_param.call(100)
```

可以用`lambda`方法把代码块储存在变量中，用lambda对象的`call`方法来执行代码块，以及接受传递来的任何参数。



#### 2.6 其他构造元素

##### 2.6.1 日期与时间

Ruby提供了`Time`这个类来处理时间和日期，`Time`内部机制是用微妙值来保存时间，根据UNIX的时间基准：格林尼治标准时间(GMT)/协调世界时(UTC) 1970年元月1日0时0分0秒。

```ruby
puts Time.now
```

打印结果：

```
2020-01-28 19:18:51 +0800
```

`+0800`表示当前时区的时间 

```ruby
puts Time.local("2020", "01", "27", "19", "22", "00")
```

`Time`允许根据当前时区创建一个`Time`对象，从month开始所有的参数都是可选的。

```ruby
puts Time.utc("2020", "01", "27", "19", "22", "00")
```

根据GMT/UTC创建`Time`对象，所需参数与`local`方法类似。

```ruby
seconds = Time.now.to_i
time = Time.at(seconds)

puts time
puts time.year, time.month
```

基准时间和`Time`对象互转，以及获取日期的年、月等相关属性。

| 方法  |         方法的返回值          |
| :---: | :---------------------------: |
| hour  |     表示24小时格式的数字      |
|  min  |            分钟数             |
|  sec  |             秒数              |
| usec  |            微秒数             |
|  day  |          该月第几天           |
| mday  |           与day同义           |
| wday  |         按周计的天数          |
| yday  |         按年计的天数          |
| month |           当年月份            |
| year  |             年份              |
| zone  |     返回时间关联的时区名      |
| utc?  | 根据time判断是否在UTC/GMT时区 |
| gmt?  |        与utc?方法同义         |



### 3. 类、对象和模块

#### 3.1 面向对象的基础知识

##### 3.1.1 局部变量、全局变量、实例变量和类变量

全局变量在程序的任何地方都可以访问，包括在类或对象中，在Ruby中全局变量并不常用，因为它违背了面向对象的思想，在Ruby使用`$`符号来定义全局变量。

```ruby
$x = 10
def basic_mehod
	puts $x
end

basic_mehod
```

实例变量其作用域关联于当前对象，实例变量有一个`@`符号前缀。

类变量的作用域处于整个类中，类变量以`@@`符号为前缀表示。

##### 3.1.2 类方法和对象方法

```ruby
class Square
	def test_method
		puts "from an instance of the class"
	end

	def self.test_method
		puts "from the Square class"
	end
end
```

`self.test_method`为类方法，类方法由`self.`前缀标识，或者也可以使用`Square.test_method`来定义类方法。



##### 3.1.3 继承

Ruby中任何类都可以继承另一个类的功能特性，Ruby只支持单继承。

```ruby
class Person
	def initialize(name)
		@name = name
	end

	def name
		@name
	end
end

class Doctor < Person
	def name
		"Dr. " + super
	end
end
```

`Doctor`类覆写`name`方法，并调用`super`。



##### 3.1.4 覆写现有方法

Ruby是一门动态语言，可以覆写现有的类和方法。

```ruby
class String
	def length
		20
	end
end
puts "test string".length
```

Ruby的一些程序库和扩展（插件）覆写了核心类的方法，以扩展Ruby的通用功能，对于覆写系统提供的方法，还是要小心慎重。



##### 3.1.5 对象方法的反射和发现

反射是指计算机程序在运行和使用中，检视、分析并修改自身的过程，而Ruby把这种反射用到了极致，允许在运行自己的代码时，修改语言自身的大部分功能。在Ruby中，可以查询几乎任何对象定义的方法。

```ruby
puts "test".methods.join(" ")
```

运行结果：

```
encode encode! unpack unpack1 include? % * + count partition to_c sum next casecmp casecmp? insert bytesize match match? succ! <=> next! upto index replace == === chr =~ rindex [] []= byteslice getbyte setbyte clear scrub empty? eql? -@ downcase scrub! dump undump upcase +@ capitalize swapcase upcase! downcase! capitalize! swapcase! hex oct freeze inspect bytes chars codepoints lines reverse reverse! concat split crypt ord length size grapheme_clusters succ start_with? center prepend strip rjust rstrip ljust chomp delete_suffix sub to_str to_sym intern sub! lstrip << to_s to_i to_f gsub! chop! chomp! delete_prefix gsub chop end_with? scan tr strip! lstrip! rstrip! delete_prefix! delete_suffix! delete! tr_s delete squeeze tr! tr_s! each_grapheme_cluster squeeze! each_line each_byte each_char each_codepoint b slice slice! hash encoding force_encoding unicode_normalize valid_encoding? ascii_only? rpartition unicode_normalize! unicode_normalized? to_r between? <= >= clamp < > instance_variable_defined? remove_instance_variable instance_of? kind_of? is_a? tap instance_variable_get instance_variable_set instance_variables singleton_method method public_send define_singleton_method public_method extend to_enum enum_for !~ respond_to? object_id send display nil? class singleton_class clone dup itself yield_self then taint tainted? untaint untrust untrusted? trust frozen? methods singleton_methods protected_methods private_methods public_methods equal? ! __id__ instance_exec != instance_eval __send__
```

对于任何对象，methods方法都返回该对象可用方法的数组。



##### 3.1.6 封装

封装是指对象将其组成数据隐藏在抽象接口之后的能力，封装的基本原理是向类之外尽可能少地减少暴露方法。

```ruby
class Person
	def initialize(name)
		set_name(name)
	end

	def name
		@first_name + ' '+ @last_name
	end

	private
	def set_name(name)
		first_name, last_name = name.split(/\s+/)
		set_first_name(first_name)
		set_last_name(last_name)
	end

	def set_first_name(name)
		@first_name = name
	end

	def set_last_name(name)
		@last_name = name
	end
end
```

`private`关键字的作用就是class之外的代码无法访问，与其他语言的类似。

```ruby
private :set_name, :set_first_name, :set_last_name
```

也可以把`private`当成命令使用，向其传递一组符号参数，表示想把这些符号表示的方法设置为私有。



##### 3.1.7 多态

多态的概念是指编写代码可以同时适用于多种类型和类的对象，如 `+`方法适用于数据相加、字符串相连以及数组合并等，`+`方法具体操作什么，完全依赖于具体类型。

##### 3.1.8 嵌套类

在Ruby中，可以把类放入其他类中，这样的类称为嵌套(nested)类。

```ruby
class Drawing
	def self.get_circle
		Circle.new
	end

	class Line
	end

	class Circle
		def desc
			"this is a circle"
		end
	end
end

d = Drawing.get_circle
c = Drawing::Circle.new

puts d.desc
puts c.desc
```

嵌套类只能以`::`的形式创建。

##### 3.1.8 常量的作用域

```ruby
Pi = 3.141592
class Planet
	Pi = 3.14
	def circle(radius)
		2 * Pi * radius
	end
end
puts Planet::Pi
puts Pi
```

在类中定义的常量，其作用域在类的环境中，通过`Planet::Pi`方式调用。



##### 3.1.9 判断对象的类型

```ruby
puts "hello".is_a? Object
puts "hello".kind_of? Object
puts "hello".instance_of?Object
puts "hello".instance_of? String
```

打印结果：

```
true
true
false
true
```

`is_a?`和`kind_of?`是同义的，`instance_of?`与这个两个不同的是，只在对象是那个类的实例而不是子类时才返回true。



#### 3.2 模块、命名空间和渗入

模块提供了一种结构，用来把Ruby类、方法和常量收集到单独命名和定义的单元中，这样可以避免与现有类、方法和常量发生命名冲突。

##### 3.2.1 命名空间

在number_stuff.rb和letter_stuff.rb分别分别存在`random`方法如下：

```ruby
# number_stuff.rb
def random
	rand(100)
end
```

```ruby
# letter_stuff.rb
def random
	(rand(26) + 65).chr
end
```

遇到如下的场景，加载的两个文件都存在同一个方法`random`:

```ruby
require './number_stuff.rb'
require './letter_stuff.rb'

puts random
```

打印结果:

```
M
```

由于最后一个文件(letter_stuff.rb)被载入，因此结果时使用最后一个版本的`random`方法。

```ruby
module NumberStuff
	def NumberStuff.random
		rand(100)
	end
end

module LetterStuff
	def LetterStuff.random
		(rand(26) + 65).chr
	end
end
```

可以用`module`来解决命名冲突的问题，它提供了命名空间。



##### 3.2.2 渗入

在Ruby中不支持多继承，但在某些情况，需要从不同的类中共享功能，类似swift中的协议，在这种意义上，模块像某种“超级”类，可被包含到其他类中，用模块提供的功能来扩展其他类。

```ruby
module UsefulFeatures
	def class_name
		self.class.to_s
	end
end

class Person
	include UsefulFeatures

end

p = Person.new
puts p.class_name
```

`UsefulFeatures`模块看起来几乎像个“类”，不过模块本身就是组织工具，而不是类，`class_name`方法位于模块中，因此随即被包含在`Person`类中。`include`命名是把模块的内容放到当前作用域中。

```ruby
module A
	def a1
		puts "this is a1"
	end
end

module B
	def b1
		puts "this is b1"
	end
end

class Sample
	include A
	include B
	def s1
		puts "this is Sample"
	end
end

s = Sample.new
s.a1
s.b1
s.s1
```

打印结果：

```
this is a1
this is b1
this is Sample
```



前面提到的`Kernel`模块包含所有的标准命令，在Ruby中无须指定对象或者类，如`load`、`require`、`exit`等，它们是特殊的方法，通过`Kernel`模块，被默认包含所有类（包括主程序作用）中。

除了`Kernel`模块，系统还提供了`Enmerable`、`Comparable`模块，`Enmerable`提供的主要是与迭代瞎相关的功能，如`each`、`sort`、`max`、`min`等，`Comparable`为其他类提供比较运算符。



### 4. 部署Ruby应用和程序库

#### 4.1 Ruby程序发布

#### 4.2 检测Ruby运行环境

#### 4.3 以gem包形式发布Ruby程序库

### 5. Ruby的高级功能

#### 5.1 动态代码执行

#### 5.2 Ruby运行其他程序

### 6. Ruby on Rails: Ruby的杀手级应用

