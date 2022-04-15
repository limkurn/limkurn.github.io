---
title: "Ruby_web_server"
date: 2022-04-16T00:37:06+08:00
draft: true
description: "笔记回顾"
Summary: "ruby 常见web server的源码阅读"
---

#写在开头
时隔许久，开始写这一篇看看web server的源码读书笔记，其实现在在ruby-china.org上有一些帖子在解释对应的web server,自己可以去搜，从前面看rack源码的时候可以学到从开始调用的地方去追寻内部的工作机制，即使不能一窥全貌，但是还是比较容易了解主干的逻辑。所以我先去看看webrick看（这个相对容易点~~），读者可以自行看一些 [ruby-china的这篇帖子](https://ruby-china.org/topics/15102) 。但是我还是想自己去看一看，不过思路都是从rack的handler入手去一步步探寻。

##webrick

1. 在前面说过rack在最后调用捕获到的server调用run方法跑起来。

```
 server.run wrapped_app, options, &blk
	
```

2. 我们去看handler里的run方法

```
	def self.run(app, options={})
        environment  = ENV['RACK_ENV'] || 'development'
        default_host = environment == 'development' ? 'localhost' : '0.0.0.0'

        options[:BindAddress] = options.delete(:Host) || default_host
        options[:Port] ||= 8080
        options[:OutputBufferSize] = 5
        @server = ::WEBrick::HTTPServer.new(options)
        @server.mount "/", Rack::Handler::WEBrick, app
        yield @server  if block_given?
        @server.start
     end

```

这里从顶层作用域去查找WEBrick并调用了在里面定义的HTTPServer的new方法。然后我们跳到里面去

3. 跳过来我们发现HTTPServe的new

```
  class HTTPServer < ::WEBrick::GenericServer
  end
```

我们会发现HTTPServer类继承自GenericServer，然后进去看看就发现ruby-china的davidqhr在帖子里为什么会去预先说SizedQueue的一些东西，以及特别提示

4. 看看GenericServer的初始化方法

```
	attr_reader :status, :config, :logger, :tokens, :listeners

    def initialize(config={}, default=Config::General)
      @config = default.dup.update(config)
      @status = :Stop
      @config[:Logger] ||= Log::new
      @logger = @config[:Logger]

      @tokens = SizedQueue.new(@config[:MaxClients])
      @config[:MaxClients].times{ @tokens.push(nil) }

      webrickv = WEBrick::VERSION
      rubyv = "#{RUBY_VERSION} (#{RUBY_RELEASE_DATE}) [#{RUBY_PLATFORM}]"
      @logger.info("WEBrick #{webrickv}")
      @logger.info("ruby #{rubyv}")

      @listeners = []
      unless @config[:DoNotListen]
        if @config[:Listen]
          warn(":Listen option is deprecated; use GenericServer#listen")
        end
        listen(@config[:BindAddress], @config[:Port])
        if @config[:Port] == 0
          @config[:Port] = @listeners[0].addr[1]
        end
      end
    end

```

ruby-china的davidqhr提到有两个重要参数tokens和listeners 分别代表最大监听数以及监听连接的scoket数组. 对于调用的SizedQueue davidqhr也阐述了其原理即：这是一个线程安全的队列，有大小限制。队列为空的时候，pop操作的线程会被阻塞。队列满的时候，push操作的线程会被阻塞。不过我们还是去它定义的地方看看是为什么。

5. 用rubymine 跳到SizeQueue的定义处才发现是ruby自带的。其定义

> This class represents queues of specified size capacity. The push operation may be blocked if the capacity is full.

6. 所以直接跳回到HTTPServer类

```
	def initialize(config={}, default=Config::HTTP)
      super(config, default)
      @http_version = HTTPVersion::convert(@config[:HTTPVersion])

      @mount_tab = MountTable.new
      if @config[:DocumentRoot]
        mount("/", HTTPServlet::FileHandler, @config[:DocumentRoot],
              @config[:DocumentRootOptions])
      end

      unless @config[:AccessLog]
        @config[:AccessLog] = [
          [ $stderr, AccessLog::COMMON_LOG_FORMAT ],
          [ $stderr, AccessLog::REFERER_LOG_FORMAT ]
        ]
      end

      @virtual_hosts = Array.new
    end
```

HTTPServer.new(options) 调用时讲rack里处理好的一些配置信息传递到webrick里，然后webrick结合自身的一些默认配置设置配置信息。这里调用了mount方法 我比较疑惑，因为在rack handler里也调用了。

7. mount先不看 待会儿回头来看，能看懂多少算多少。继续rack handler的代码。@server调用start方法，但是HTTPServer 没有定义是继承自GenericServer的方法。

```
	def start(&block)
      raise ServerError, "already started." if @status != :Stop
      server_type = @config[:ServerType] || SimpleServer

      server_type.start{
        @logger.info \
          "#{self.class}#start: pid=#{$$} port=#{@config[:Port]}"
        call_callback(:StartCallback)

        thgroup = ThreadGroup.new
        @status = :Running
        while @status == :Running
          begin
            if svrs = IO.select(@listeners, nil, nil, 2.0)
              svrs[0].each{|svr|
                @tokens.pop          # blocks while no token is there.
                if sock = accept_client(svr)
                  sock.do_not_reverse_lookup = config[:DoNotReverseLookup]
                  th = start_thread(sock, &block)
                  th[:WEBrickThread] = true
                  thgroup.add(th)
                else
                  @tokens.push(nil)
                end
              }
            end
          rescue Errno::EBADF, IOError => ex
            # if the listening socket was closed in GenericServer#shutdown,
            # IO::select raise it.
          rescue Exception => ex
            msg = "#{ex.class}: #{ex.message}\n\t#{ex.backtrace[0]}"
            @logger.error msg
          end
        end

        @logger.info "going to shutdown ..."
        thgroup.list.each{|th| th.join if th[:WEBrickThread] }
        call_callback(:StopCallback)
        @logger.info "#{self.class}#start done."
        @status = :Stop
      }
    end
```

前面就是判断是否启动了，如果已经启动了报错，然后根据指定的server类型 或者默认的SimpleServer(简单的接受一个block来运行),然后使用ThreadGroup启动一个线程组

> ThreadGroup provides a means of keeping track of a number of threads as a group.

> A given Thread object can only belong to one ThreadGroup at a time; adding a thread to a new group will remove it from any previous group.

> Newly created threads belong to the same group as the thread from which they were created.

只要@status为Running  循环就会一直执行，2.0S内会监听是否有IO对象可读， 否者过2.0s就会返回nil 然后进入while的下一次循环。如果有可读对象 就往下执行。@token#pop的时候 看注释如果没有token会阻塞这应该是SiezedQueue的pop的策略问题(得去补补tread的知识，有点没看懂pop方法的定义)

```
  def pop(*args)
    retval = super
    @mutex.synchronize {
      if @que.length < @max
        begin
          t = @queue_wait.shift
          t.wakeup if t
        rescue ThreadError
          retry
        end
      end
    }
    retval
  end
```

根据davidqhr 的解释：

> @tokens的作用类似信号量，初始化server的时候，会把@tokens用nil填充满，只有能从@token获取到信号的时候，才可以创建线程，获取不到信号的时候，会阻塞主线程，以此控制并发数量

但是具体的业务是start_thread里去操作的。跳进去看看

8. 看看方法的定义

```
  	def start_thread(sock, &block)
      Thread.start{
        begin
          Thread.current[:WEBrickSocket] = sock
          begin
            addr = sock.peeraddr
            @logger.debug "accept: #{addr[3]}:#{addr[1]}"
          rescue SocketError
            @logger.debug "accept: <address unknown>"
            raise
          end
          call_callback(:AcceptCallback, sock)
          block ? block.call(sock) : run(sock)
        rescue Errno::ENOTCONN
          @logger.debug "Errno::ENOTCONN raised"
        rescue ServerError => ex
          msg = "#{ex.class}: #{ex.message}\n\t#{ex.backtrace[0]}"
          @logger.error msg
        rescue Exception => ex
          @logger.error ex
        ensure
          @tokens.push(nil)
          Thread.current[:WEBrickSocket] = nil
          if addr
            @logger.debug "close: #{addr[3]}:#{addr[1]}"
          else
            @logger.debug "close: <address unknown>"
          end
          sock.close
        end
      }
    end
```

我查了下资料

```
Thread.current[:WEBrickSocket] = sock
```

这一句应该是将sock 存在当前线程的WEBrickSocket里，确保线程安全。
后面执行的逻辑直接跳到run方法里去了。

9. 老长的run方法~~~

```
	def run(sock)
      while true
        res = HTTPResponse.new(@config)
        req = HTTPRequest.new(@config)
        server = self
        begin
          timeout = @config[:RequestTimeout]
          while timeout > 0
            break if IO.select([sock], nil, nil, 0.5)
            timeout = 0 if @status != :Running
            timeout -= 0.5
          end
          raise HTTPStatus::EOFError if timeout <= 0
          raise HTTPStatus::EOFError if sock.eof?
          req.parse(sock)
          res.request_method = req.request_method
          res.request_uri = req.request_uri
          res.request_http_version = req.http_version
          res.keep_alive = req.keep_alive?
          server = lookup_server(req) || self
          if callback = server[:RequestCallback]
            callback.call(req, res)
          elsif callback = server[:RequestHandler]
            msg = ":RequestHandler is deprecated, please use :RequestCallback"
            @logger.warn(msg)
            callback.call(req, res)
          end
          server.service(req, res)
        rescue HTTPStatus::EOFError, HTTPStatus::RequestTimeout => ex
          res.set_error(ex)
        rescue HTTPStatus::Error => ex
          @logger.error(ex.message)
          res.set_error(ex)
        rescue HTTPStatus::Status => ex
          res.status = ex.code
        rescue StandardError => ex
          @logger.error(ex)
          res.set_error(ex, true)
        ensure
          if req.request_line
            if req.keep_alive? && res.keep_alive?
              req.fixup()
            end
            res.send_response(sock)
            server.access_log(@config, req, res)
          end
        end
        break if @http_version < "1.1"
        break unless req.keep_alive?
        break unless res.keep_alive?
      end
    end
```

这里先构建请求和返回的对象，然后设置根据传来的timeout属性设置timeout时间（减去IO.select的时间）后面req调用parse方法根据sock的信息去设置req的一些信息，然后查找server 再调用server对应的service 方法，再执行res.send_response(sock)返回结果。最后看看是不是要保持长连接决定是不是调出while循环。


10. lookup_server

```
	 # Finds the appropriate virtual host to handle +req+

    def lookup_server(req)
      @virtual_hosts.find{|s|
        (s[:BindAddress].nil? || req.addr[3] == s[:BindAddress]) &&
        (s[:Port].nil?        || req.port == s[:Port])           &&
        ((s[:ServerName].nil?  || req.host == s[:ServerName]) ||
         (!s[:ServerAlias].nil? && s[:ServerAlias].find{|h| h === req.host}))
      }
    end
```

其实注释已经很明白了~~~~ 所以继续看service方法

11.  service

```
	def service(req, res)
      if req.unparsed_uri == "*"
        if req.request_method == "OPTIONS"
          do_OPTIONS(req, res)
          raise HTTPStatus::OK
        end
        raise HTTPStatus::NotFound, "`#{req.unparsed_uri}' not found."
      end

      servlet, options, script_name, path_info = search_servlet(req.path)
      raise HTTPStatus::NotFound, "`#{req.path}' not found." unless servlet
      req.script_name = script_name
      req.path_info = path_info
      si = servlet.get_instance(self, *options)
      @logger.debug(format("%s is invoked.", si.class.name))
      si.service(req, res)
    end
```

search_servlet 根据请求的路径求查找存在@mount_tab变量里的对应路径的servlet

```
 # Finds a servlet for +path+

    def search_servlet(path)
      script_name, path_info = @mount_tab.scan(path)
      servlet, options = @mount_tab[script_name]
      if servlet
        [ servlet, options, script_name, path_info ]
      end
    end
```

其实@mount_tab是一个MountTable的实例 本质上是一个hash变量

```
   class MountTable
      def initialize
        @tab = Hash.new
        compile
      end

      def [](dir)
        dir = normalize(dir)
        @tab[dir]
      end

      def []=(dir, val)
        dir = normalize(dir)
        @tab[dir] = val
        compile
        val
      end

      def delete(dir)
        dir = normalize(dir)
        res = @tab.delete(dir)
        compile
        res
      end

      def scan(path)
        @scanner =~ path
        [ $&, $' ]
      end

      private

      def compile
        k = @tab.keys
        k.sort!
        k.reverse!
        k.collect!{|path| Regexp.escape(path) }
        @scanner = Regexp.new("^(" + k.join("|") +")(?=/|$)")
      end

      def normalize(dir)
        ret = dir ? dir.dup : ""
        ret.sub!(%r|/+$|, "")
        ret
      end
    end
```

每次调用mount和unmount的都是都在对hash做存储和删除操作，只不过在操作后会有一些操作以便后面根据path搜索对应的servlet.
而后调用get_instance方法根据传入的server和option创建一个新的servlet， 英语不太好翻译不过来，只能自行领会了~~

>Factory for servlet instances that will handle a request from server using options from the mount point. By default a new servlet instance is created for every call.

然后新的实例调用service方法去将req里对应的do_method 在对应地方掉起执行，如果如果方法不被允许会报MethodNotAllowed的错误。
最后res.send_response(sock)将do_method处理后的body header等信息传回给客户端。请求完成。

### WEBrick里的servlet

1.什么是一个servlet
我查了下wiki，servlet应该是起源于java里的一个概念，主要功能在于交互式的浏览和修改数据。Servlet可以响应任何类型的请求，但绝大多数情况下Servlet只用来扩展基于HTTP协议的Web服务器。然后我google了下看到[这篇帖子](https://www.igvita.com/2007/02/13/building-dynamic-webrick-servers-in-ruby/)后，个人理解在webrick里的servlet 其实是继承自HTTPServlet::AbstractServlet 的一个子类，在子类中根据自己的需要去自定义了do_GET do_POST等方法。在使用时，不同的path挂在不同的servlet的时候，调用service 就能把req和res传到自定的do_GET或者do_POST中处理。

2. Rack::Handler::WEBrick

其实Rack::Handler::WEBrick本身就是继承自::WEBrick::HTTPServlet::AbstractServlet的一个子类，在执行run方法时

```
  @server.mount "/", Rack::Handler::WEBrick, app

```

请求就会用Rack::Handler::WEBrick去处理，Rack::Handler::WEBrick还复写了service方法， do_GET这些方法就没再拓展，直接

```
  status, headers, body = @app.call(env)

```

最后将body head等写入res就搞定一切~~有大神的做参考就是叼~~~ thx  david


## eventmachine

thin我看了源码貌似用了很多eventmachine的东西，所以干脆直接看看eventmachine就好了。

1. eventmachine 是基于[反应堆模式](https://en.wikipedia.org/wiki/Reactor_pattern)的一个事件驱动IO实现.

> EventMachine is designed to simultaneously meet two key needs:
> Extremely high scalability, performance and stability for the most demanding production environments.
> An API that eliminates the complexities of high-performance threaded network programming, allowing engineers to concentrate on their application logic.

2. 相关的学习资料还是比较多的。

  * eventmachine里自带了一些例子脚本，同时有[文档](https://github.com/eventmachine/eventmachine/blob/master/docs/GettingStarted.md)帮助理解，如何用eventmachine启动一个server、如何用它来做一个聊天室同时了解EventMachine::Connection类定义的一些api。
  * 本身的wiki里还有一些文档带我们概览一些用法同时整理了一些和[教程](https://github.com/eventmachine/eventmachine/wiki/Tutorials)。不过感觉弄懂代码里自带的150行聊天室教程，很多都不用看了。
  * 还有人翻译了一一篇[入门教程](http://lunae.cc/play-with-eventmachine)，原链接在blog里说了这篇教程提到了下定时器，而且简短，可以看看。
  * Aman Gupta在rubyconf上做过一个[演讲](https://www.youtube.com/watch?v=GEd3KQEGj0Q)专门说eventmachine的，强烈推荐观看。
  * rubychina的坛子里有人发布一个[帖子](https://ruby-china.org/topics/13565)，然后评论异常精彩~~干活多多。

3. eventmachine的注释好棒！...可以自己去翻看一下，我们先跳到thin。设计到eventmachine的再说。

## thin 

1. 还是从rack的handler开始

```
module Rack
  module Handler
    class Thin
      def self.run(app, options={})
        environment  = ENV['RACK_ENV'] || 'development'
        default_host = environment == 'development' ? 'localhost' : '0.0.0.0'

        host = options.delete(:Host) || default_host
        port = options.delete(:Port) || 8080
        args = [host, port, app, options]
        # Thin versions below 0.8.0 do not support additional options
        args.pop if ::Thin::VERSION::MAJOR < 1 && ::Thin::VERSION::MINOR < 8
        server = ::Thin::Server.new(*args)
        yield server if block_given?
        server.start
      end

      def self.valid_options
        environment  = ENV['RACK_ENV'] || 'development'
        default_host = environment == 'development' ? 'localhost' : '0.0.0.0'

        {
          "Host=HOST" => "Hostname to listen on (default: #{default_host})",
          "Port=PORT" => "Port to listen on (default: 8080)",
        }
      end
    end
  end
end
```

所以我们去从Thin::Server的new和start方法开始

2. Server的初始化方法

```
	def initialize(*args, &block)
      host, port, options = DEFAULT_HOST, DEFAULT_PORT, {}
      
      # Guess each parameter by its type so they can be
      # received in any order.
      args.each do |arg|
        case arg
        when Fixnum, /^\d+$/ then port    = arg.to_i
        when String          then host    = arg
        when Hash            then options = arg
        else
          @app = arg if arg.respond_to?(:call)
        end
      end
      
      # Set tag if needed
      self.tag = options[:tag]

      # Try to intelligently select which backend to use.
      @backend = select_backend(host, port, options)
      
      load_cgi_multipart_eof_fix
      
      @backend.server = self
      
      # Set defaults
      @backend.maximum_connections            = DEFAULT_MAXIMUM_CONNECTIONS
      @backend.maximum_persistent_connections = DEFAULT_MAXIMUM_PERSISTENT_CONNECTIONS
      @backend.timeout                        = options[:timeout] || DEFAULT_TIMEOUT
      
      # Allow using Rack builder as a block
      @app = Rack::Builder.new(&block).to_app if block
      
      # If in debug mode, wrap in logger adapter
      @app = Rack::CommonLogger.new(@app) if Logging.debug?
      
      @setup_signals = options[:signals] != false
    end
```

加上本身的注释还是比较好懂的，判断传递参数的类型去设置host port option 几个参数，根据选择到的backend 设置backend的一些属性，这些属性方法是在Server类开头利用Forwardable进行委托的，后面的只是如果传了block 就用Rack::Builder处理下赋给@app。如果是debug模式再用Rack::CommonLogger包装一遍@app。

3. 再看一下start方法

```
	def start
      raise ArgumentError, 'app required' unless @app
      
      log_info  "Thin web server (v#{VERSION::STRING} codename #{VERSION::CODENAME})"
      log_debug "Debugging ON"
      trace     "Tracing ON"
      
      log_info "Maximum connections set to #{@backend.maximum_connections}"
      log_info "Listening on #{@backend}, CTRL+C to stop"

      @backend.start { setup_signals if @setup_signals }
    end
```

这里其实最重要的最后一句。这里backend调用了自身的start方法。thin的backend 其实感觉和webrick的servlet有点像。thin本身定义了一个backend的base类，同时有TcpServer UnixServer 以及SwiftiplyClient，同时我们可以自己去拓展backend类，然后像注释中的例子这样调用：

```
 Thin::Server.start('galaxy://faraway', 1345, app, :backend => Thin::Backends::MyFancyBackend)

```

注释中还举了几个例子

```
  # == TCP server
  # Create a new TCP server on bound to <tt>host:port</tt> by specifiying +host+
  # and +port+ as the first 2 arguments.
  #
  Thin::Server.start('0.0.0.0', 3000, app)
  #
  # == UNIX domain server
  # Create a new UNIX domain socket bound to +socket+ file by specifiying a filename
  # as the first argument. Eg.: /tmp/thin.sock. If the first argument contains a <tt>/</tt>
  # it will be assumed to be a UNIX socket. 
  #
  Thin::Server.start('/tmp/thin.sock', app)
```

看到这几个例子我们就能够看懂initialize中的select_backend方法是如何工作的了

4. select_backend

```
	def select_backend(host, port, options)
        case
        when options.has_key?(:backend)
          raise ArgumentError, ":backend must be a class" unless options[:backend].is_a?(Class)
          options[:backend].new(host, port, options)
        when options.has_key?(:swiftiply)
          Backends::SwiftiplyClient.new(host, port, options)
        when host.include?('/')
          Backends::UnixServer.new(host)
        else
          Backends::TcpServer.new(host, port)
        end
      end
```

如果我们自己指定backend类，select_backend会去判断是否有这个backend存在 如果存在就建立一个实例。然后判断是不是要使用swiftiply（以及web后端的集群代理，才疏学浅，以前我没听说过~~~）最后根据传递的host是不是一个本地路径来判断是启动一个TcpServer实例还是UnixServer实例。

5. 弄明白了这个我们就可以去看backend类了，由于thin是基于eventmachine的base类中提供一些方法去设置eventmachine的线程池大小，以及在初始化的时候设置一些基本属性例如@connections(从stop中可知里面报错了长连接对象) 以及默认的超时时间等。前面调用@backend.start 其实是从base类继承而来

```
	def start
        @stopping = false
        starter   = proc do
          connect
          yield if block_given?
          @running = true
        end
        
        # Allow for early run up of eventmachine.
        if EventMachine.reactor_running?
          starter.call
        else
          @started_reactor = true
          EventMachine.run(&starter)
        end
     end
      
```

而在每个特定的backend中自定义了connect方法,例如TcpServer调用EventMachine里的方法启动一个tcp server:

```
def connect
   @signature = EventMachine.start_server(@host, @port, Connection,&method(:initialize_connection))
end
```

这里的Connnection类其实是EventMachine::Connection的子类，然后自定义了post_init receive_data以及unbind等方法，如果认真看EventMachine的东西很好懂。

再如UnixServer#connect方法

```
     def connect
        at_exit { remove_socket_file } # In case it crashes
        old_umask = File.umask(0)
        begin
          EventMachine.start_unix_domain_server(@socket, UnixConnection, &method(:initialize_connection))
          # HACK EventMachine.start_unix_domain_server doesn't return the connection signature
          #      so we have to go in the internal stuff to find it.
        @signature = EventMachine.instance_eval{@acceptors.keys.first}
        ensure
          File.umask(old_umask)
        end
      end
```

也是调用EventMachine的start_unix_domain_server去启动对应的unix server

6. 最后Thin是如何终止server的运行呢

```
      # Stop of the backend from accepting new connections.
      def stop
        @running  = false
        @stopping = true
        
        # Do not accept anymore connection
        disconnect
        # Close idle persistent connections
        @connections.each_value { |connection| connection.close_connection if connection.idle? }
        stop! if @connections.empty?
      end
      
      # Force stop of the backend NOW, too bad for the current connections.
      def stop!
        @running  = false
        @stopping = false
        
        EventMachine.stop if @started_reactor && EventMachine.reactor_running?
        @connections.each_value { |connection| connection.close_connection }
        close
      end
```

一般情况是先设置running和stopping属性 然后调用disconnect 管理server 连接不再接受任何连接再关闭所以长连接，等到全部连接清空后再调用stop！如果EventMachine的reactor还在跑就调用EventMachine.stop。不过也可以只调用stop！关闭server，只是比较暴力...

7. Thin是如何接受信号的？在Thin::Server初始化的时候会判断选项时候有signals

```
      @setup_signals = options[:signals] != false

```

后面backend的start的时候会根据这个参数来决定是否运行方法

```
   def setup_signals
        # Queue up signals so they are processed in non-trap context
        # using a EM timer.
        @signal_queue ||= []

        %w( INT TERM ).each do |signal|
          trap(signal) { @signal_queue.push signal }
        end
        # *nix only signals
        %w( QUIT HUP USR1 ).each do |signal|
          trap(signal) { @signal_queue.push signal }
        end unless Thin.win?

        # Signals are processed at one second intervals.
        @signal_timer ||= EM.add_periodic_timer(1) { handle_signals }
      end

      def handle_signals
        case @signal_queue.shift
        when 'INT'
          stop!
        when 'TERM', 'QUIT'
          stop
        when 'HUP'
          restart
        when 'USR1'
          reopen_log
        end
        EM.next_tick { handle_signals } unless @signal_queue.empty?
      end
```

如果执行setup_signals，后面所有接受的信号会保存到@signal_queue变量中 然后调用EventMachine的add_periodic_timer方法设置一个定时器 不断从@signal_queue抛出信号去执行对应的操作，或者@signal_queue为空进入下一次定时调用。

8. Thin还有CLI模式，基本都由Runner Command 以及controllers下面定义的东西来完成有兴趣可以看看。
9. 其实还有个东西值得一看但是我的C语言很差，没看懂就是Thin::HttpParser。

## 尾声

最后推荐大家去看坛子里有人推荐的一个视频[Rails Conf 2013 From Rails to the Web Server to the Browser by David Padilla](https://www.youtube.com/watch?v=A0M4BwYrYiA)，我看了下，讲的可能没那么细，但是帮助我们贯穿整个webserver rack 以及rackapp 是如何工作的。下一篇回去看sinatra的源码...刚吧嘚！！！！！
