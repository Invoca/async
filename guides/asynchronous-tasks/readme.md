# Asynchronous Tasks

This guide explains how asynchronous tasks work and how to use them.

## Overview

Tasks are the smallest unit of sequential code execution in {ruby Async}. Tasks can create other tasks, and Async tracks the parent-child relationship between tasks. When a parent task is stopped, it will also stop all its children tasks. The outer event loop generally has one root task.

```mermaid
graph LR
    EL[Event Loop] --> WS
    WS[Web Server Task] --> R1[Request 1 Task]
    WS --> R2[Request 2 Task]

    R1 --> Q1[Database Query Task]
    R1 --> H1[HTTP Client Request Task]

    R2 --> H2[HTTP Client Request Task]
    R2 --> H3[HTTP Client Request Task]
```

## Task Lifecycle

Tasks represent units of work which are executed according to the following state transition diagram:

```mermaid
stateDiagram-v2
    [*] --> initialized : Task.new
    initialized --> running : run
    
    running --> failed : unhandled exception
    running --> complete : user code finished
    running --> stopped : stop

    initialized --> stopped : stop
    
    failed --> [*]
    complete --> [*]
    stopped --> [*]
```

Tasks are created in the `initialized` state, and are run by the event loop. During the execution, a task can either `complete` successfully, become `failed` with an unhandled exception, or be explicitly `stopped`. In all of these cases, you can wait for a task to complete by using {ruby Async::Task#wait}.

1. In the case the task successfully completed, the result will be whatever value was generated by the last expression in the task.
2. In the case the task failed with an unhandled exception, waiting on the task will re-raise the exception.
3. In the case the task was stopped, the result will be `nil`.

## Starting A Task

At any point in your program, you can start the reactor and a root task using the {ruby Kernel::Async} method:

```ruby
Async do
	3.times do |i|
		sleep 1
		puts "Hello World #{i}"
	end
end
```

This program prints "Hello World" 3 times. Before printing, it sleeps for 1 second. The total execution time is 3 seconds because the program executes sequentially.

By using a nested task, we can ensure that each iteration of the loop creates a new task which runs concurrently.

```ruby
Async do
	3.times do |i|
		Async do
			sleep 1
			puts "Hello World #{i}"
		end
	end
end
```

Instead of taking 3 seconds, this program takes 1 second in total. The main loop executes rapidly creating 3 child tasks, and then each child task sleeps for 1 second before printing "Hello World".

```mermaid
graph LR
	EL[Event Loop] --> TT[Initial Task]
	
	TT --> H0[Hello World 0 Task]
	TT --> H1[Hello World 1 Task]
	TT --> H2[Hello World 2 Task]
```

By constructing your program correctly, it's easy to implement concurrent map-reduce:

```ruby
Async do
	# Map (create several concurrent tasks)
	users_size = Async{User.size}
	posts_size = Async{Post.size}
	
	# Reduce (wait for and merge the results)
	average = posts_size.wait / users_size.wait
	puts "#{users_size.wait} users created #{ruby Average} posts on average."
end
```

### Performance Considerations

Task creation and execution has been heavily optimised. Do not trade program complexity to avoid creating tasks; the cost will almost always exceed the gain.

Do consider using correct concurrency primatives like {ruby Async::Semaphore}, {ruby Async::Barrier}, etc, to ensure your program is well-behaved in the presence of large inputs (i.e. don't create an unbounded number of tasks).

## Starting a Limited Number of Tasks

When processing potentially unbounded data, you may want to limit the concurrency using {ruby Async::Semaphore}.

```ruby
Async do
	# Create a semaphore with a limit of 2:
	semaphore = Async::Semaphore.new(2)
	
	file.each_line do |line|
		semaphore.async do
			# Only two tasks at most will be allowed to execute concurrently:
			process(line)
		end
	end
end
```

## Waiting for Tasks

Waiting for a single task is trivial: simply invoke {ruby Async::Task#wait}. To wait for multiple tasks, you may either {ruby Async::Task#wait} on each in turn, or you may want to use a {ruby Async::Barrier}. You can use {ruby Async::Barrier#async} to create multiple child tasks, and wait for them all to complete using {ruby Async::Barrier#wait}.

```ruby
barrier = Async::Barrier.new

Async do
	jobs.each do |job|
		barrier.async do
			# ... process job ...
		end
	end
	
	# Wait for all jobs to complete:
	barrier.wait
end
```

### Waiting for the First N Tasks

Occasionally, you may need to just wait for the first task (or first several tasks) to complete. You can use a combination of {ruby Async::Waiter} and {ruby Async::Barrier} for controlling this:

```ruby
waiter = Async::Waiter.new(parent: barrier)

Async do
	jobs.each do |job|
		waiter.async do
			# ... process job ...
		end
	end
	
	# Wait for the first two jobs to complete:
	done = waiter.wait(2)
	
	# You may use the barrier to stop the remaining jobs
	barrier.stop
end
```

### Combining a Barrier with a Semaphore

{ruby Async::Barrier} and {ruby Async::Semaphore} are designed to be compatible with each other, and with other tasks that nest `#async` invocations. There are other similar situations where you may want to pass in a parent task, e.g. {ruby Async::IO::Endpoint#bind}.

~~~ ruby
barrier = Async::Barrier.new
semaphore = Async::Semaphore.new(2, parent: barrier)

jobs.each do |job|
	semaphore.async(parent: barrier) do
		# ... process job ...
	end
end

# Wait until all jobs are done:
barrier.wait
~~~

## Stopping a Task

When a task completes execution, it will enter the `complete` state (or the `failed` state if it raises an unhandled exception).

There are various situations where you may want to stop a task ({ruby Async::Task#stop}) before it completes. The most common case is shutting down a server, but other important situations exist, e.g. you may fan out multiple (10s, 100s) of requests, wait for a subset to complete (e.g. the first 5 or all those that complete within a given deadline), and then stop (terminate/cancel) the remaining operations.

Using the above program as an example, we can 

```ruby
Async do
	tasks = 3.times.map do |i|
		Async do
			sleep 1
			puts "Hello World #{i}"
		end
	end
	
	# Stop all the above tasks:
	tasks.each(&:stop)
end
```

### Stopping all Tasks held in a Barrier

To stop (terminate/cancel) all the tasks held in a barrier:

```ruby
barrier = Async::Barrier.new

Async do
	tasks = 3.times.map do |i|
		barrier.async do
			sleep 1
			puts "Hello World #{i}"
		end
	end
	
	barrier.stop
end
```

If you're letting individual tasks held by a barrier raise unhandled exceptions, be sure to call ({ruby Async::Barrier#stop}) to stop the remaining tasks:

```ruby
barrier = Async::Barrier.new

Async do
	tasks = 3.times.map do |i|
		barrier.async do
			sleep 1
			puts "Hello World #{i}"
		end
	end
	
	begin
		barrier.wait
	ensure
		barrier.stop
	end
end
```

## Resource Management

In order to ensure your resources are cleaned up correctly, make sure you wrap resources appropriately, e.g.:

~~~ ruby
Async do
	begin
		socket = connect(remote_address) # May raise Async::Stop

		socket.write(...) # May raise Async::Stop
		socket.read(...) # May raise Async::Stop
	ensure
		socket.close if socket
	end
end
~~~

As tasks run synchronously until they yield back to the reactor, you can guarantee this model works correctly. While in theory `IO#autoclose` allows you to automatically close file descriptors when they go out of scope via the GC, it may produce unpredictable behavour (exhaustion of file descriptors, flushing data at odd times), so it's not recommended.

## Exception Handling

{ruby Async::Task} captures and logs exceptions. All unhandled exceptions will cause the enclosing task to enter the `:failed` state. Non-`StandardError` exceptions are re-raised immediately and will generally cause the reactor to fail. This ensures that exceptions will always be visible and cause the program to fail appropriately.

~~~ ruby
require 'async'

task = Async do
	# Exception will be logged and task will be failed.
	raise "Boom"
end

puts task.status # failed
puts task.wait # raises RuntimeError: Boom
~~~

### Propagating Exceptions

If a task has finished due to an exception, calling `Task#wait` will re-raise the exception.

~~~ ruby
require 'async'

Async do
	task = Async do
		raise "Boom"
	end
	
	begin
		task.wait # Re-raises above exception.
	rescue
		puts "It went #{$!}!"
	end
end
~~~

## Timeouts

You can wrap asynchronous operations in a timeout. This allows you to put an upper bound on how long the enclosed code will run vs. potentially blocking indefinitely. If the enclosed code hasn't completed by the timeout, it will be interrupted with an {ruby Async::TimeoutError} exception.

~~~ ruby
require 'async'

Async do |task|
	task.with_timeout(1) do
		sleep 100
	rescue Async::TimeoutError
		puts "I timed out 99 seconds early!"
	end
end
~~~

### Periodic Timers

Sometimes you need to do some recurring work in a loop. Often it's best to measure the periodic delay end-to-start, so that your process always takes a break between iterations and doesn't risk spending 100% of its time on the periodic work. In this case, simply call {ruby sleep} between iterations:

~~~ ruby
require 'async'

Async do |task|
	loop do
		puts Time.now
		# ... process job ...
		sleep 1
	end
end
~~~

## Reactor Lifecycle

Generally, the outer reactor will not exit until all tasks complete. This is informed by {ruby Async::Task#finished?} which checks if the current node has completed execution, which also includes all children. However, there is one exception to this rule: tasks flagged as being `transient` ({ruby Async::Node#transient?}).

### Transient Tasks

Tasks which are flagged as `transient` do not behave like normal tasks.

```ruby
@pruner = Async(transient: true) do
	loop do
	
		sleep 1
		prune_connection_pool
	end
end
```

1. They are not considered by {ruby Async::Task#finished?}, so they will not keep the reactor alive. Instead, they are stopped (with a {ruby Async::Stop} exception) when all other (non-transient) tasks are finished.
2. As soon as a parent task is finished, any transient child tasks will be moved up to be children of the parent's parent. This ensures that they never keep a sub-tree alive.
3. Similarly, if you `stop` a task, any transient child tasks will be moved up the tree as above rather than being stopped.

The purpose of transient tasks is when a task is an implementation detail of an object or instance, rather than a concurrency process. Some examples of transient tasks:

- A task which is reading or writing data on behalf of a stateful connection object, e.g. HTTP/2 frame reader, Redis cache invalidation, etc.
- A task which is monitoring and maintaining a connection pool, pruning unused connections or possibly ensuring those connections are periodically checked for activity (ping/pong, etc).
- A background worker or batch processing job which is independent of any specific operation, and is lazily created.
- A cache system which needs periodic expiration / revalidation of data/values.

Bearing in mind, in all of the above cases, you may need to validate that the background task hasn't been stopped, e.g.

```ruby
require 'async'
require 'thread/local' # thread-local gem.

class TimeCache
	extend Thread::Local # defines `instance` class method that lazy-creates a separate instance per thread
	
	def initialize
		@current_time = nil
	end
	
	def current_time_string
		refresh!
		
		return @current_time
	end
	
	private
	
	def refresh!
		@refresh ||= Async(transient: true) do
			loop do
				@current_time = Time.now.to_s
				sleep(1)
			end
		ensure
			# When the reactor terminates all tasks, this will be invoked:
			@refresh = nil
		end
	end
end

Async do
	# If you are handling 1000s of requests per second, it can be an advantage to cache the current time string.
	p TimeCache.instance.current_time_string
end
```

Upon existing the top level async block, the {ruby @refresh} task will be set to `nil`. Bear in mind, you should not share these resources across threads; doing so would need some form of mutual exclusion.
