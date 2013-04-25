sad-single-file
===============

The sad(em based background job) in a single-file

一个文件实现简单的基于Eventmachine的后台任务

```ruby
require "active_support"
require "eventmachine"
require "em-redis"
require "json"

module Sad
	class Config
		class << self
			def namespace=(str)
				@_namespace = str
			end

			def namespace
				@_namespace || 'SadQueue'
			end

			def redis=(opts={})
				opts = {
					:host => 'localhost', 
					:port => 6379,
					:db => 0
					}.update opts
				@_redis = EM::Protocols::Redis.connect :host => opts[:host], :port => opts[:port], :db => opts[:db]
			end

			def redis
				@_redis || (EM::Protocols::Redis.connect)
			end
		end
	end

	module Worker
		def queue_name
			name = if self.respond_to?(:queue)
				self.send :queue
			else
				nil
			end
			[Sad::Config.namespace, name].join ':'
		end

		def enqueue(*args)
			payload = ::Sad::Payload.new(self.to_s, args)
			::Sad::Config.redis.rpush(queue_name, payload.encode)
		end
	end

	class Server
		class << self
			def run(queue)
				fetch([Sad::Config.namespace, queue].join ':')
			end

			def fetch(queue)
				::Sad::Config.redis.blpop(queue, 60){|_, data|
					if data
						STDOUT.puts '-'*15 + data.inspect + '-'*15
						payload = Payload.decode(data)
						EM.defer{perform(payload.klass, payload.args)}
					end
					fetch(queue)
				}
			end

			def perform(klass, args)
				klass.constantize.send :perform, args
			end
		end
	end

	class Payload
		attr_accessor :klass, :args

		def initialize(klass, args)
			@klass = klass
			@args = args
		end

		def encode
			{
				'klass' => @klass,
				'args' => @args
			}.to_json
		end

		def self.decode(json)
			h = JSON.parse(json)
			self.new(h['klass'], h['args'])
		end
	end
end
```

----------

测试类的定义

```ruby
class SadJob
	extend ::Sad::Worker
	
	def self.queue
		'MySadJob'
	end

	def self.perform(*args)
		puts "I'm in sad job perform method."
		puts args
	end
end
```

### 在一个IRB/PRY中执行

```ruby
EM::PeriodicTimer.new(3){
	SadJob.enqueue('this is some args', {:hello => 'code'})	
}
```

### 在另外一个IRB/PRY中执行

```ruby
EM.run{
	Sad::Server.run('MySadJob')
}
```

两个运行环境都要有SadJob这个测试类的定义存在