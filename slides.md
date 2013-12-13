# Basics of threaded programming in Ruby



<h3>Who am I?</h3>
<h2>Gergő Sulymosi</h2>
<p>
  chief git blame analyst at [DiNa](http://digitalnatives.hu)<br>
  Github/twitter @trekdemo
</p>



# What is a thread?


> In computer science, a thread of execution is the smallest sequence of
> programmed instructions that can be managed independently by an operating
> system scheduler.
>
> -- Wikipedia


``` ruby
Thread.main
Thread.current
Thread.new {}
```

Note:
  By default, your code is always running a thread. Even if you don't create any
  new threads, there's always at least one: the main thread.

  Major difference between regular and main thread:
  When the main thread exits, all other threads are immediately terminated and
  the Ruby process exits.


# What's in it for me?


## Cheaper concurrency
* Cheaper in memory usage
* Less overhead than spawning a process

Note:
  forking processes could be sparing -- copy-on-write
  mark-and-sweep before 2.0 not CoW friendly
  bitmap marking in 2.0 CoW friendly -- you can use this perk


## Code organization

* Handle signals (like ruby)
* Isolate task, functional units
* [More...](http://www.jstorimer.com/blogs/workingwithcode/7766063-threads-not-just-for-optimizations)

Note:
  organizing code (handling signals, etc.)
  http://www.jstorimer.com/blogs/workingwithcode/7766063-threads-not-just-for-optimizations

  In ruby there are 2 threads after you launch the interpreter.



# Concurrency<br> vs.<br> Parallelism


Concurrency and parallelism are two related but distinct concepts.

**Concurrency**
: Task A and task B both need to happen independently of each
other. A starts running, and then B starts before A is finished, but they
not necessarily going to be executed simultaniusly.

**Parallelism**
: Task A and task B both running in the same time, simultaniusly.

Note:
  Concurrency: they don't have to run simultaniusly
  Concurrency: same things, but they run simultaniusly


<pre>
Concurrent
       >---->    >---->>        Task A
>----->      >-->      >----->> Task B

Paralell
       >-------->>              Task A
>---------------------------->> Task B
</pre>



# Threading with different ruby implementations


## Rubinius, JRuby and MacRuby...
...have real native OS threads and paralell execution.


## MRI
* < 1.9 [Green threads](http://en.wikipedia.org/wiki/Green_threads)
* \> 1.9 Native OS threads and fibers GIL/GVL

Note:
  Green threads managed by the VM, there is no real thread behind them
  Fibers are not real threads, their management is done by the developer



# What is GIL <span style="text-transform:lowercase;">a.k.a.</span> GVL <span style="text-transform:lowercase;">a.k.a.</span> global lock?
* The GIL is a global lock around the execution of Ruby code.
* The biggest implication is that Ruby code will never run in parallel on MRI.


``` ruby
require 'digest/md5'

3.times.map do
  Thread.new do
    Digest::MD5.hexdigest(rand)
  end
end.each(&:value)
```

Note:
  This computation will run simultaniusly in 3 threads with jruby, rubinius
  and concurrently on mri.


## What's the reason for it?

1. To protect MRI internals from race conditions
2. To facilitate the C extension API
3. To reduce the likelihood of race conditions in your Ruby code

Note:
  A race condition involves two threads racing to perform an operation on some
  shared state.


## What is possible with MRI?
``` ruby
require 'open-uri'

3.times.map do
  Thread.new do
    open('http://digitalnatives.hu')
  end
end.each(&:value)
```


## How others solved this issue?
* **Rubinius** have fine-grained locks around the internal data structures, and
helps gem (C extension) developers to avoid race conditions.

* **JRuby**, does not support C extensions, you can use java based extension
instead.

Note:
  Rubinius, for instance, replaced their GIL with about [50 fine-grained
  locks](http://www.jstorimer.com/2013/03/26/brian-shirai-threads.html). Without
  a GIL, its internals are still protected, but because it spends less time in
  these locks, more of your Ruby code can run in parallel.



# Thread-safe code


1. Don't allow concurrent modification,
1. or protect concurrent modification!


    @results ||= "some value"

Note:
  * Anything where there is only one shared instance is a global


## Shorthand for

    if @result.nil?
      temp = "some value"
      @result = temp
    end


## What happens if my code is not threadsafe?
* best case: nothing
* worst case:
    * corrupt data in memory/database
    * unpredictable results

Note:
  The biggest problem cused when data modification is not threadsafe. In that
  case, you will have corrupt data. If your application is heavily based on data,
  you're fucked. Your app will give you unpredictable results, mostly under high
  load. And you won't be able to track these bugs in dev environment.


## How can I write threadsafe code?
* Avoid mutating globals (constants, the AST, class variables/methods)
* Create multiple objects instead a shared one
* Use Thread-locals
* Use resource pools
* Avoid lazy-loading
* Prefer data structures over mutexes
* Use immutable objects

Note:
  AST: It's the DOM of your code
  Lazy-loading: require not thread-safe, deadlocks
  Avoid data structures: hide the low level mutex management with data
  structures
  Immutable objects: objects that can't be modified after the initialization


## Avoid or protect concurrent data modification


### Mutexes

    shared_mutex_accross_threads = Mutex.new
    mutex.lock
    mutex.unlock
    mutex.synchronize {}

Note:
  The name 'mutex' is shorthand for 'mutual exclusion.' If you wrap some section
  of your code with a mutex, you guarantee that no two threads can enter that
  section at the same time.


* Must be shared between the threads
* It has performance tradeoffs (mostly on Rubinius, JRuby)
* You should choose the place of lock carefully
* You should lock the smallest possible part of the code
* Avoid deadlocks

Note:
  A deadlock occurs when one thread is blocked waiting for a resource from
  another thread (like blocking on a mutex), while this other thread is itself
  blocked waiting for a resource.


### ConditionVariable
These can be used to communicate between threads to synchronize resource access.

* Thread A produces items into an array
* Thread B waits for the items (conditional variable)
* When Th. A signals through the cond. var.
* Th. B can access the new item from the array


``` ruby
require 'thread'

array = []
mutex = Mutex.new
cond_var = ConditionVariable.new

Thread.new do
  10.times do
    mutex.synchronize do
      sleep(rand)
      array << rand
      cond_var.signal
    end
  end
end
Thread.new do
  10.times do
    mutex.synchronize do
      cond_var.wait(mutex) while array.empty?
      puts "Processing #{array.shift}"
    end
  end
end
```

Note:
  A ConditionVariable can be used to signal one (or many) threads when some
  event happens, or some state changes, whereas mutexes are a means of
  synchronizing access to resources.


### Queue
The only thread-safe data structure from standard lib. Can be implemented with
an Array, a Mutex and a ConditionVariable.

* `#push` and `#pop`
* blocks on `#pop`

Note:
  You might be thinking: "With all of the great concurrency support available to
  Java on the JVM, surely the JRuby Array and Hash are thread-safe?"

  They're not.  For the exact reason mentioned above, using a thread-safe data
  structure in a single-threaded context would reduce performance.


# How many threads do I need?

Note:
  * It depends
  * There is no golden number
  * You have to measure


## CPU-bound vs IO-bound

Note:
  * With MRI, there is no difference between threaded and non-threaded code,
    because the GIL locks the execution if it's a CPU-bound task.
  * But if you have IO heavy operations, you can do them in paralell.



# Further topics...
* Actor model - Celluloid
* Lock-less CAS (compare and set)
* Pools



## Thank you!
# Any questions?

