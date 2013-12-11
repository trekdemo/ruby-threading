# Basics of threaded programming in Ruby



<h3>Who am I?</h3>
<h2>Gergő Sulymosi</h2>
<p>
  chief git blame analyst at DiNa<br>
  Github/twitter @trekdemo
</p>



# What is a thread?


> In computer science, a thread of execution is the smallest sequence of
> programmed instructions that can be managed independently by an operating
> system scheduler.
>
> -- Wikipedia



# What's in it for me?


## Cheaper concurrency
* Cheaper in memory usage
* Less overhead than spawning a process

Note: memory management (process vs thread - copy vs sharing)


## Code organization

* Handle signals (like ruby)
* Isolate task, functional units
* [More...](http://www.jstorimer.com/blogs/workingwithcode/7766063-threads-not-just-for-optimizations)

Note:
organizing code (handling signals, etc.)
http://www.jstorimer.com/blogs/workingwithcode/7766063-threads-not-just-for-optimizations



# Concurrency<br> vs.<br>  Parallelism


Concurrency and parallelism are two related but distinct concepts.

**Concurrency**
: Task A and task B both need to happen independently of each
other. A starts running, and then B starts before A is finished, but they
not necessarily going to be executed simultaniusly.

**Parallelism**
: Task A and task B both running in the same time, simultaniusly.


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



# What is GIL a.k.a. GVL a.k.a. global lock?
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


## What's the reason for it?

1. To protect MRI internals from race conditions
2. To facilitate the C extension API
3. To reduce the likelihood of race conditions in your Ruby code


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

Note:
}}} images/fightclubpicture.jpg
* Anything where there is only one shared instance is a global


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


### Avoid or protect concurrent data modification


### Mutexes


### Queue, ConditionalVariable



# How many threads do I need?


## CPU-bound vs IO-bound
