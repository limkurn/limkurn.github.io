---
title: "Sinatra_source_code"
date: 2022-04-16T00:38:14+08:00
draft: true
description: "笔记回顾"
Summary: "Sinatra 源码阅读"
---

## 写在开头

sinatra是一个很轻量的web框架，同时它也可以当做rack的middleware来用。我在ruby china论坛看到一些很不错的帖子来介绍sinatra，讨论了源码以及设计风格。同时也找到一份解说源码的非常不错的英文资料。所以我想了想还是直接列出来，大家自己去看一下就好，我只对几个对我来说觉得有意思的点理一理。因为那几篇帖子，特别是英文资料说的实在太详细了，而且水平也比我高很多。sinatra版本为1.4.6

## 相关帖子以及资料

如果要好好理解sinatra的源码，我推荐大家认真阅读这几篇帖子。

1. 五星推荐 (sinatra_explained)[https://github.com/zhengjia/sinatra-explained]这个系列文章没写完，但是从启动到middleware、路由分配以及请求和相应结果都做了详细的刨析，值得仔细品读。
2. 讲述源码的流程，亮点在于对unboudmethod的阐述讲解 (阅读Sinatra 源码的感悟)[https://ruby-china.org/topics/23579]
3. 再一篇就是炮哥写的(how sinatra works)[https://ruby-china.org/topics/7921]，但是感觉有点粗粒度.开头的启动流程言简意赅，很湿容易理解。同时说了为什么可以用两种风格写sinatra应用。
4. 最后一个是saito的(sinatra的扩展机制)[https://ruby-china.org/topics/2110], saito我不认识真人，但是据说是一个sinatra迷，他还写了一个叫(simba的gem)[https://github.com/SaitoWu/simba]。

如果详细看过上面的几篇内容，可以不用往下看了。

## 自己感兴趣的点

### 为什么sinatra可以使用两种风格的的写法

#### 模块化风格

先看看这种风格的写法，理解起来比较容易

```
require'sinatra/base'

class Hello < Sinatra::Base

  get "/" do
    "hello world"
  end

  run!
end
```

这种写法的get以及 run！等方法都是定义在Hello的父类Sinatra::Base之中。我们在pry中执行

```
require 'sinatra/base'
Sinatra::Base.methods(false)
```

> [:settings, :routes, :filters, :templates, :errors, :reset!, :extensions, :middleware, :set, :enable, :disable, :error, :not_found, :template, :layout, :inline_templates=, :mime_type, :mime_types, :before, :after, :add_filter, :condition, :public=, :public_dir=, :public_dir, :get, :put, :post, :delete, :head, :options, :patch, :link, :unlink, :helpers, :register, :development?, :production?, :test?, :configure, :use, :quit!, :stop!, :run!, :start!, :running?, :prototype, :new!, :new, :build, :call, :caller_files, :caller_locations, :force_encoding, :environment=, :environment, :environment?, :raise_errors=, :raise_errors, :raise_errors?, :dump_errors=, :dump_errors, :dump_errors?, :show_exceptions=, :show_exceptions, :show_exceptions?, :sessions=, :sessions, :sessions?, :logging=, :logging, :logging?, :protection=, :protection, :protection?, :method_override=, :method_override, :method_override?, :use_code=, :use_code, :use_code?, :default_encoding=, :default_encoding, :default_encoding?, :x_cascade=, :x_cascade, :x_cascade?, :add_charset=, :add_charset, :add_charset?, :session_secret=, :session_secret, :session_secret?, :methodoverride?, :methodoverride=, :run=, :run, :run?, :running_server=, :running_server, :running_server?, :handler_name=, :handler_name, :handler_name?, :traps=, :traps, :traps?, :server=, :server, :server?, :bind=, :bind, :bind?, :port=, :port, :port?, :absolute_redirects=, :absolute_redirects, :absolute_redirects?, :prefixed_redirects=, :prefixed_redirects, :prefixed_redirects?, :empty_path_info=, :empty_path_info, :empty_path_info?, :app_file=, :app_file, :app_file?, :root=, :root, :root?, :views=, :views, :views?, :reload_templates=, :reload_templates, :reload_templates?, :lock=, :lock, :lock?, :threaded=, :threaded, :threaded?, :public_folder=, :public_folder, :public_folder?, :static=, :static, :static?, :static_cache_control=, :static_cache_control, :static_cache_control?]

在这些列出的方法中就可以看到我们刚刚使用的 get和 run！方法的身影。

#### 经典风格

```
require'sinatra'

get '/' do
  "hello world"
end
```

这个就要跳到源码中的main.rb的最后一句

```
extend Sinatra::Delegator
```

```
  # Sinatra delegation mixin. Mixing this module into an object causes all
  # methods to be delegated to the Sinatra::Application class. Used primarily
  # at the top-level.
  
  module Delegator
    def self.delegate(*methods)
      methods.each do |method_name|
        define_method(method_name) do |*args, &block|
          return super(*args, &block) if respond_to? method_name
          Delegator.target.send(method_name, *args, &block)
        end
        private method_name
      end
    end

    delegate :get, :patch, :put, :post, :delete, :head, :options, :link, :unlink,
             :template, :layout, :before, :after, :error, :not_found, :configure,
             :set, :mime_type, :enable, :disable, :use, :development?, :test?,
             :production?, :helpers, :settings, :register

    class << self
      attr_accessor :target
    end

    self.target = Application
  end
```

Delegator类的注释其实写的很清楚这个主要用与顶层作用域，同时会把委托的方法发送到Application中去执行。
在这里 Delegator先定义了一个delegate的类方法，使用define_method定义指定的方法(由于receiver 是self 所以定义的其实是Delegator类方法)，在执行这些方法时，如果引入Delegator模块的对象能响应这个方法 就将参数传到父类的同名方法执行；如果不行就会发送到 Delegator.target的对应方法执行。注意一点就是 在delegate :get 等方法时Delegator.target并没有解析，执行时才会解析。 默认的Delegator.target是Application类。

#### Rack::Builder

main.rb最后的代码是拓展了 Rack::Builder

```
class Rack::Builder
  include Sinatra::Delegator
end

```

所以在诸如

```
app = Rack::Builder.new do
  use Rack::CommonLogger
  use Rack::ShowExceptions
  map "/lobster" do
    use Rack::Lint
    run Rack::Lobster.new
  end
end
```

这种代码的block里就可以去用get 等方法了


### sinatra拓展

写sinatra拓展主要依赖register和helpers两个方法，根据炮哥所说:

> sinatra主要分两个作用域，一个叫DSL (class) Context，另一个叫Request Context。register和helpers分别扩展这两个作用域。(详细的看这里吧)[http://www.sinatrarb.com/extensions.html]

看着两个方法的实现：

```
  # Extend the top-level DSL with the modules provided.
  def self.register(*extensions, &block)
    Delegator.target.register(*extensions, &block)
  end

  # Include the helper modules provided in Sinatra's request context.
  def self.helpers(*extensions, &block)
    Delegator.target.helpers(*extensions, &block)
  end
```

都是交由Delegator.target(Application)来完成的：

```
  class Application < Base
    set :logging, Proc.new { ! test? }
    set :method_override, true
    set :run, Proc.new { ! test? }
    set :session_secret, Proc.new { super() unless development? }
    set :app_file, nil

    def self.register(*extensions, &block) #:nodoc:
      added_methods = extensions.map {|m| m.public_instance_methods }.flatten
      Delegator.delegate(*added_methods)
      super(*extensions, &block)
    end
  end
  
  class Base
    ...
      #Makes the methods defined in the block and in the Modules given
      # in `extensions` available to the handlers and templates
      def helpers(*extensions, &block)
        class_eval(&block)   if block_given?
        include(*extensions) if extensions.any?
      end

      # Register an extension. Alternatively take a block from which an
      # extension will be created and registered on the fly.
      def register(*extensions, &block)
        extensions << Module.new(&block) if block_given?
        @extensions += extensions
        extensions.each do |extension|
          extend extension
          extension.registered(self) if extension.respond_to?(:registered)
        end
      end
    ...
  end
```

#### register

register其实拓展是DSL Context。 Application类的register首先将拓展的所有public_instance_methods利用 Delegator.delegate拓展到Top level后面使用send调用，再利用Base中定义的方法自己Application类extend所有拓展的方法为类方法。

```
    def self.delegate(*methods)
      methods.each do |method_name|
        define_method(method_name) do |*args, &block|
          return super(*args, &block) if respond_to? method_name
          Delegator.target.send(method_name, *args, &block)
        end
        private method_name
      end
    end
```

因为这种操作并没有修改Base类，所以就有了模块的时候手动register 这些拓展使得 Base的子类可以使用这些拓展的方法，所以官方文档的例子也就能理解为什么了。

```
 #Modular style applications must register the extension explicitly in their Sinatra::Base subclasses 
require 'sinatra/base'
require 'sinatra/linkblocker'

class Hello < Sinatra::Base
  register Sinatra::LinkBlocker

  block_links_from 'digg.com'

  get '/' do
    "Hello World"
  end
end

```

这里最主要的是理解 这些拓展的方法没有修改Base类，不然就陷入误区。

#### helpers

helpers 其实修改的是Request Context。诸如此形式：

```
require 'sinatra/base'

module Sinatra
  module HTMLEscapeHelper
    def h(text)
      Rack::Utils.escape_html(text)
    end
  end

  helpers HTMLEscapeHelper
end
```

helpers将extension中的方法拓展Delegator.target中的instance_method，不改变Base类，对应Base的其他子类需要使用这些拓展时 也同理需要手动调用helpers方法，将extension中的方法拓展到当前类。这些东西需要自己去体会。炮哥他们在帖子里一笔带过，但是人家心里明白，我们自己也要做到心里明白才行。


### catch&&throw

还是推荐大家看看(这篇文章)[https://ruby-china.org/topics/23579]分析的比较细致， 这里面对catch throw 做了讲解。同时我也测试了一下 catch throw 即使对于相同的标签是逐层上抛的,这个小脚本试试就知道了。

```
def invoke
  res = catch(:halt) { yield }
  puts "#{res}\n"
end

invoke do
  invoke do
    throw :halt, ["1111"]
  end
end
```

#### 不带返回参数时

根据官网文档

> catch executes its block. If throw is not called, the block executes normally, and catch returns the value of the last expression evaluated.
> If +throw(tag2, val)+ is called, Ruby searches up its stack for a catch block whose tag has the same object_id as tag2. When found, the block stops executing and returns val (or nil if no second argument was given to throw).

所以对于这种代码： 

```
	  catch(:pass) do
        conditions.each { |c| throw :pass if c.bind(self).call == false }
        block ? block[self, values] : yield(self, values)
      end
```

throw了以后 最后整个块就返回nil....


### 为什么Base里定义了Application，还要在main.rb里分开写一部分

pending...

### 其他感悟

在sinatra中写拓展时，register 如何实现类似included 这样的钩子方法; 以及sinatra 如果现实before after 这样的回调方法，如何去执行这些回调的，很开阔思路，后面看看Rails框架的实现，收获应该更大。


### 一个有趣的项目

我给写sinatra的作者写邮件的时候，突然发现一个(有趣的项目)[https://github.com/rkh/almost-sinatra], 主要代码只有几行，可以自己去体会下



