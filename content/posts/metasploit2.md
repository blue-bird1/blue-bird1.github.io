---
title: metasploit python 模块是如何运行
date: 2018-02-02T22:57:52+08:00
lastmod: 2018-02-02T22:57:52+08:00
categories: ["网络安全"]
---
## 开始
metasploit的扩展实现的代码主要在`metasploit-framework/lib/msf/core/modules/external
`
目录结构如下
~~~
├── bridge.rb
├── message.rb
├── python
│   ├── async_timeout
│   │   ├── __init__.py
│   └── metasploit
│       ├── __init__.py
│       ├── __init__.pyc
│       ├── module.py
│       ├── module.pyc
│       ├── probe_scanner.py
├── shim.rb
└── templates
    ├── capture_server.erb
    ├── common_metadata.erb
    ├── dos.erb
    ├── multi_scanner.erb
    └── remote_exploit_cmd_stager.erb

~~~

其中`python`目录的py代码将会在我们运行python模块时加入python路径.这也是为什么我们能导入`metasploit`
`templates`目录则是用于实现将python代码变成模块的代码模板.事实上我们能使用msf对正常模块的功能 如`info`都是靠这些模板实现的.
`message.rb`和`bridge`是与msf jsonrpc通信的一些api.
`shim.rb`则是真正将python代码实现为模块的代码.

这里省略了不重要的细节的`message.rb`代码
~~~
class Msf::Modules::External::Message
 
  def self.from_module(j)                                                                                        
    if j['method']
      m = self.new(j['method'].to_sym)
      m.params = j['params']
      m
    elsif j['response']
      m = self.new(:reply)
      m.params = j['response']
      m.id = j['id']
      m
    end
  end
 
  def initialize(m)
    self.method = m
    self.params = {}
    self.id = Base64.strict_encode64(SecureRandom.random_bytes(16))
  end
 
  def to_json
    params =
      if self.params.respond_to? :to_nested_values
        self.params.to_nested_values
      else
        self.params.to_h
      end
    JSON.generate({jsonrpc: '2.0', id: self.id, method: self.method, params: params})
  end

~~~
这个类实际上是对传递给metasploit的信息的一个封装.
`initialize`是ruby的初始化方法 从这里可以看到它有三个属性`method` `params` `id` 

`from_module`方法则是用于将传递的参数转换成自身
`to_json`方法很明显就是转换成一个可用的json(jsonrpc传递需要json格式)

这里是省略了不重要的细节的`bridge.rb`代码
~~~
require 'msf/core/modules/external/message'

class Msf::Modules::External::Bridge
  
  # 通过jsonrpc运行
  def run(datastore)
    unless self.running
      m = Msf::Modules::External::Message.new(:run)
      m.params = datastore.dup
      send(m)
      self.running = true
    end
  end
	
  # 获取当前状态和恢复run状态  
  def get_status
    if self.running || !self.messages.empty?
      m = receive_notification
      if m.nil?
        close_ios
        self.messages.close
        self.running = false
      end

      return m
    end
  end
  
  # 接收
  def recv(filter_id=nil, timeout=600)
    _, out, err = self.ios
    message = ''
  
  # jsonrpc发送和接受
  def send_receive(message)
    send(message)
    recv(message.id)
  end
  
  # 发送模块元数据
  def describe
    resp = send_receive(Msf::Modules::External::Message.new(:describe))
    close_ios
    resp.params
  end
  	
  # 发送  
  def send(message)
    input, output, err, status = ::Open3.popen3(self.env, self.cmd)
    self.ios = [input, output, err]
    self.wait_thread = status
    case select(nil, [input], nil, 0.1)
    when nil
      raise "Cannot run module #{self.path}"
    when [[], [input], []]
      m = message.to_json
      write_message(input, m)
    else
      raise "Error running module #{self.path}"
    end
  end
  
  # TODO 这里原本有一大段关于网络接受的代码 

# 每个编程语言扩展的具体实现
class Msf::Modules::External::PyBridge < Msf::Modules::External::Bridge
 # 判断是否是py文件
  def self.applies?(module_name)
    module_name.match? /\.py$/
  end
 
  # 初始化 python扩展添加了额外的路径
  def initialize(module_path)
    super
    pythonpath = ENV['PYTHONPATH'] || ''
    self.env = self.env.merge({ 'PYTHONPATH' => pythonpath + File::PATH_SEPARATOR + File.expand_path('../python', __FILE__) })
  end
end


class Msf::Modules::External::Bridge
  # 载入列表 我们可以期待更多的语言可以编写msf模块 如Msf::Modules::External::JsBridge?
  LOADERS = [
    Msf::Modules::External::PyBridge,
    Msf::Modules::External::Bridge
  ]
	
  # 运行模块方法 让载入的bridge都判断是否是自己所属的 
  def self.open(module_path)
    LOADERS.each do |klass|
      return klass.new module_path if klass.applies? module_path
    end
    nil
  end
end
~~~


这里是省略了不重要的细节的`shim.rb`代码
~~~
require 'msf/core/modules/external/bridge'

class Msf::Modules::External::Shim
  # 将bridge返回的数据生成一个模块
  def self.generate(module_path)
    mod = Msf::Modules::External::Bridge.open(module_path)
    return '' unless mod.meta
    # 这里根据模块元数据来选择模板 目前只有3个 元数据获取查看bridge.rb的meta方法 
    case mod.meta['type']
    when 'remote_exploit_cmd_stager'
      remote_exploit_cmd_stager(mod)
    when 'capture_server'
      capture_server(mod)
    when 'dos'
      dos(mod)
    when 'scanner.multi'
      multi_scanner(mod)
    else
      # TODO have a nice load error show up in the logs
      ''
    end
  end
  
  # 返回一个模块 erb是ruby的一个代码模板库
  def self.render_template(name, meta = {})
    template = File.join(File.dirname(__FILE__), 'templates', name)
    ERB.new(File.read(template)).result(binding)
  end

  def self.common_metadata(meta = {})
    render_template('common_metadata.erb', meta)
  end
  
  # 数据转换
  def self.mod_meta_common(mod, meta = {})
    meta[:path]        = mod.path.dump
    meta[:name]        = mod.meta['name'].dump
    meta[:description] = mod.meta['description'].dump
    meta[:authors]     = mod.meta['authors'].map(&:dump).join(",\n          ")

    meta[:options]     = mod.meta['options'].map do |n, o|
      "Opt#{o['type'].camelize}.new(#{n.dump},
        [#{o['required']}, #{o['description'].dump}, #{o['default'].inspect}])"
    end.join(",\n          ")
    meta
  end

 # 渲染膜拜
  def self.dos(mod)
    meta = mod_meta_common(mod)
    meta[:date] = mod.meta['date'].dump
    meta[:references] = mod.meta['references'].map do |r|
      "[#{r['type'].upcase.dump}, #{r['ref'].dump}]"
    end.join(",\n          ")
    render_template('dos.erb', meta)
  end
end

~~~

所以python模块的运行过程其实是这样的
1. class Msf::Modules::External::Shim获取到了模块路径
2. 调用Msf::Modules::External::Bridge.open
3.  在open方法 Msf::Modules::External::PyBridge::applies判断成功(也就是确认了是python模块)
4.  初始化一个Msf::Modules::External::PyBridge并返回
5.  判断元数据类型 假设是dos 则调用dos方法
6.  调用mod_meta_common方法转换元数据 渲染代码模板

我们可以查看`dos.erb`的内容
~~~
require 'msf/core/modules/external/bridge'
require 'msf/core/module/external'

class MetasploitModule < Msf::Auxiliary
  include Msf::Module::External
  include Msf::Auxiliary::Dos

  def initialize
    super({
	  <%= common_metadata meta %>
      'References'  =>
        [
          <%= meta[:references] %>
        ],
      'DisclosureDate' => <%= meta[:date] %>,
      })

      register_options([
        <%= meta[:options] %>
      ])
  end

  def run
    print_status("Starting server...")
    mod = Msf::Modules::External::Bridge.open(<%= meta[:path] %>)
    mod.run(datastore)
    wait_status(mod)
  end
end
~~~
所以事实上python模块的实现就是将python代码中元数据传递到代码模板 然后实际上调用的还是ruby模板 我们的python文件路径将会出现在
~~~
mod = Msf::Modules::External::Bridge.open(<%= meta[:path] %>)
mod.run(datastore)
~~~
最后通过`bridge.run`调用.这种扩展方法不但没有失去对ruby模块的强大支持也没丢失python的灵活性 非常好
