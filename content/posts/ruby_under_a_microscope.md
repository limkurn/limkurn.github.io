---
title: "Ruby_under_a_microscope"
date: 2022-04-20T23:07:11+08:00
draft: true
description: "笔记回顾"
Summary: "Ruby原理剖析笔记"
---

# **ruby under a microscope**

### 分词与语法分析

- ripper库可以对代码字符串进行分词：

  ```ruby 
  require 'ripper'
  
  require 'pp'
  
  code = <<STR
   10.times do |n|
  	puts n
  end
  STR
  
  puts code
  
  pp Ripper.lex(code)
  ```

  

- 在运行程序前ruby 通过将parse.y文件用BISON生成语法分析代码parse.c，然后将分词后的代码进行解析。

- ruby 语法分析使用[LALR parse](https://en.wikipedia.org/wiki/LALR_parser)算法来进行语法分析

- ruby -y 运行程序能够看到内部每次语法分析状态跳转的debug信息

- 在parse.y文件中定义了很多的ruby 语法规则

- ruby代码在解析器创建一系列的节点或内部表示ruby代码的临时数据结构后运行。创建的节点保存在抽象语法树上（AST）

- 查看语法解析内容可以用ripper的sexp方法

  ```ruby
  require 'ripper'
  require 'pp'
  
  code = <<STR
  	10.times do |n|
  		puts n
  	end
  STR
  puts code
  
  pp Ripper.sexp(code)
  ```

### 编译

- 编译意味着将代码从一种编程语言转化为另一种

- ruby在1.9以后才有编译的，以前版本的ruby不包含编译器

- The only differences between YARV and the JVM are the following:

  > Ruby doesn’t expose the compiler to you as a separate tool. Instead,
  >
  > it automatically compiles your Ruby code into bytecode instructions
  >
  > internally. Ruby never compiles your Ruby code all the way to machine
  >
  > language. As you can see in Figure 2-2, Ruby interprets the bytecode
  >
  > instruc- tions. The JVM, on the other hand, can compile some of the
  >
  > bytecode instructions all the way into machine language using its
  >
  > “hotspot” or just-in-time ( JIT) compiler.

- In Ruby all functions are actually methods. That is, functions are always associated with a Ruby class; there is always a receiver. Inside of Ruby, however, Ruby’s parser and compiler distinguish between functions and methods: Method calls have an explicit receiver, while function calls assume the receiver is the current value of self .

- 一种查看ruby编译代码的方法是使用RubyVM::InstructionSequence object：

  ```ruby
  code = <<END
  	puts 2+2
  END
  
  puts RubyVM::InstructionSequence.compile(code).disasm
  ```

- Ruby uses the following labels to describe different types of method and block arguments:

> Arg: A standard method or block argument
>
> Rest: An array of unnamed arguments that are passed together using a splat ( * ) operator
>
> Post: A standard argument that appears after the splat array
>
> Block: A Ruby proc object that is passed using the & operator
>
> Opt=i: A parameter defined with a default value. The integer value i is an index into a table that stores the actual default value. This table is stored along with the YARV snippet but not in the local table itself.

## **执行代码**

- YARV is not just a stack machine—it’s a double-stack machine! It has to track the arguments and return values not only for its own internal instructions but also for your Ruby program.
- 堆栈与堆栈帧指针
- Whenever you make a method call, Ruby sets aside some space on the YARV stack for any local variables declared inside the method you are calling.Ruby knows how many variables you are using by consulting the local table created for each method during the compilation step（堆栈中加了一个environment pointer的指针来检查当前堆栈局部变量的位置）
- Method Arguments Are Treated Like Local Variables
- dynamic variable~~~
- Ruby considers the top-level and inner-block scope to be the same with regard to special variables.

## **控制流与方法调度**

- 切换作用域YARV使用throw指令，但是break redo next等一些指令使用catch table来实现（block throws an exception at the YARV instruction level using a catch table, or a table of pointers that may be attached to any YARV code snippet.）

- YARV方法调度使用send指令 这是ruby中最复杂也最厉害的一个方法，方法调度的流程： call send , method lookup, method dispatch, YARV executes tagret method

- ruby中将方法分类为11种，一般的都是ISEQ，但是为了加速方法调度，还有10种。：

  > ISEQ A normal method that you write using Ruby code, this is the
  >
  > most common method type. ISEQ stands for instruction sequence. CFUNC
  >
  > Using C code included directly inside the Ruby executable, these are
  >
  > the methods that Ruby implements rather than you. CFUNC stands for C
  >
  > function. ATTRSET A method of this type is created by the attr_writer
  >
  > method. ATTRSET stands for attribute set. IVAR Ruby uses this method
  >
  > type when you call attr_reader . IVAR stands for instance variable.
  >
  > BMETHOD Ruby uses this method type when you call define_method and
  >
  > pass in a proc object. Because the method is represented internally by
  >
  > a proc, Ruby needs to handle this method type in a special way. ZSUPER
  >
  > Ruby uses this method type when you set a method to be public or
  >
  > private in a particular class or module when it was actually defined
  >
  > in some superclass. This method is not commonly used. UNDEF Ruby uses
  >
  > this method type internally when it needs to remove a method from a
  >
  > class. Also, if you remove a method using undef_method , Ruby creates
  >
  > a new method of the same name using the UNDEF method type.
  >
  > NOTIMPLEMENTED Like UNDEF, Ruby uses this method type to mark certain
  >
  > methods as not implemented. This is necessary, for example, when you
  >
  > run Ruby on a platform that doesn’t support a particular operating
  >
  > system call. OPTIMIZED Ruby speeds up some important methods using
  >
  > this type, like the Kernel#send method. MISSING Ruby uses this method
  >
  > type if you ask for a method object from a module or class using
  >
  > Kernel#method and the method is missing. REFINED Ruby uses this
  >
  > method type in its implementation of refinements, a new feature
  >
  > introduced in version 2.0.

- 当方法调用是根据参数不同，YARV可能会做一些额外的操作：

  > Block arguments When you use the & operator in an argument list,
  >
  > Ruby needs to convert the provided block into a proc object.
  >
  > Optional arguments Ruby adds additional code to the target method
  >
  > when you use an optional argument with a default value. This code
  >
  > sets the default value into the argument. When you later call a
  >
  > method with an optional argument, YARV resets the program counter or
  >
  > PC register to skip this added code when a value is provided. Splat
  >
  > argument array For these, YARV creates a new array object and
  >
  > collects the provided argument values into it. (See the array [3, 4,
  >
  > 5] in Listing 4-6.) Standard and post arguments Because these are
  >
  > simple values, YARV has no additional work to do.

## **对象与类**

- Ruby saves each of your custom objects in a C structure called RObject
- Every Ruby object is the combination of a class pointer and an array of instance variables.
- Robejct
- generic obejects(custom classes use different c structure)
- As a performance optimization,Integer value Flags Ruby saves small integers, symbols, and a few other simple values with VALUE out any structure at all
- Rclass
- A Ruby class is a Ruby object that also contains method definitions,attribute names, a superclass pointer, and a constants table.

## **方法与常量查找**

- ruby不允许在module中直接创建对象，换个方式说，在module中不能call new方法因为new是Class的方法而不是module
- ruby中不能给module指定superclass，class中引入module用include
- ruby的内部实现将module当做class处理，但是module的内部结构比class少了一些东西
- A Ruby module is a Ruby object that also contains method definitions, a superclass pointer, and a constants table.
- include与extend的Rclass分别变成被引入的superclass和子类
- prepend
- Ruby uses the class tree to find the methods that your code (and Ruby’s own internal code) calls. Similarly, Ruby uses both the lexical scope tree and the superclass hierarchy to find constants that your code refers to.

## **hash table**

- 当用11来作为hash算法的基数时，如果在11位上分布的实体密度太高，ruby会做优化（rehash）将基数增大

- hash函数的运行方式：

  > When you call hash , Ruby finds the default implementation in the
  >
  > Object class. You can override this if you want to. The C code used by
  >
  > the Object class’s implementation of the hash method gets the C
  >
  > pointer value for the target object—that is, the actual memory address
  >
  > of that object’s RValue structure. This is essen- tially a unique ID
  >
  > for that object. Ruby passes the pointer value through a complex C
  >
  > function (the hash function), which scrambles the bits in the value,
  >
  > producing a pseudo­ random integer in a repeatable way.

- 在2.0以后ruby对于小于６个的hash不再创建st*table*entry,而是直接将key和value直接存入一个数组

## ruby是如何借鉴lisp的

- Using each is slower because internally the Range#each method has to call or yield to the block each time around the loop. This involves a fairly large amount of work.

- We’ve seen the stack before. This is where Ruby saves local variables,return values, and arguments for each of the methods in your program.Values on the stack are valid only for as long as that method is running.When a method returns, YARV deletes its stack frame and all the values inside it.

- Ruby uses the heap to save information that you might need for a while, even after a particular method returns. Each value in the heap remains valid for as long as there is a reference to it. Once a value is no longer

  referred to by any variable or object in your program, Ruby’s garbage collection system deletes it, freeing its memory for other uses.

- The stack and heap differ in one other important aspect. Ruby saves only references to data on the stack—that is, the VALUE pointers. For simple integer values, symbols, and constants such as nil , true , or false , the refer-

  ence is the actual value. However, for all other data types, the VALUE is a pointer to a C structure containing the actual data, such as RObject . If only the VALUE references go on the stack, Ruby still save the structures In the heap.

## **元编程**

- 当使用self.method的方式定义方法的时候，实际上是将方法添加到类的元类中，元类是一种内部实现，在ruby编程中隐藏起来了，singleton_class 方法可查看
- 定义一连串的类方法，可以直接class << self的方式打开一个新的作用域
- A singleton class is a special hidden class that Ruby creates internally to hold methods defined only for a particular object.
- A metaclass is a singleton class in the case when that object is itself a class.
- Ruby 2.0’s refinements feature gave us the ability to define methods and add them to a class later if we wish.
- the current version of Ruby, 2.0,allows you to use refinements only in the top-level scope,
- When we call a method on a receiver, Ruby always sets self to be the receiver.
- Inside a class or module scope, self will always be set to that class or module. Ruby creates a new lexical scope when you use the class or module keywords and sets the class for that scope to the new class or module.
- Inside a method (including a class method), Ruby will set self to the receiver of that method call.
- pass a string to eval , and Ruby immediately parses, compiles, and executes the code.
- Ruby evaluates the new code string in the same context from where you called eval
- eval接受一个binding参数，表明eval()不使用现在的上下文

```ruby
def get_binding
	a = 2
	b = 3
	binding
end

eval("puts a+b", get_binding)
```

- The instance_eval method is similar to eval , except that it evaluates the given string in the context of the receiver, or the object we call it on.

- The instance_eval method also creates a new singleton class and sets it as the

  class for a new lexical scope

- define_method (private method in module should use send to use)

## **垃圾回收机制**



- Garbage Collectors Solve Three Problems:

> They allocate memory for use by new objects.
>
> They identify which objects your program is no longer using.
>
> They reclaim memory from unused objects.

* 几种垃圾回收算法

## **jruby**

- jvm讲ruby代码转换为java代码执行，而不是二进制指令
- string 存法不同
- copy on right fot string

## **rubinius**

- c++
- array的存储方式不同
- The Rubinius kernel implements many built-in Ruby classes, such as Array , using Ruby code. This innovative design provides a window into Ruby internals—you can use Rubinius to learn how Ruby works internally without having to know C or Java. You can learn how Ruby implements strings, arrays, or other classes simply by reading the Ruby source code in the Rubinius kernel. Rubinius isn’t just a Ruby implementation; it’s a valuable learning resource for the Ruby community.
