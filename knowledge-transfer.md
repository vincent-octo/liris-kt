KNOWLEDGE TRANSFER
==================

by Vincent, on October 2014.

[//]: # (documenter les résultats de ce que j'ai fais)
[//]: # (pré-requis pour permettre la comprehension de ce que j'ai fais par d'autres)


Abstract
--------
This documents is a retrospective of my work at LIRIS. I worked on improving the kTBS, both in term of performance and usability. The performance part was two-fold: evaluate different RDF stores with benchmarking, improve the kTBS core performance with multi-threading. The usability part was on protecting data using authentification and authorization. In this document I will try to explain how I did it, and what are the results. Overall, I think my work helped making the kTBS more production ready.


Table of Contents
-----------------

1. kTBS: Purpose & Composition
 
2. kTBS Performance
  1. Triple-store performance
  2. Locking mechanism
  3. Running the kTBS in multi-process behind Apache

3. Authentication with SSO
  1. What is OpenID Connect? (diff openid, oauth2, etc. for disambiguation, the basics of how it works)
  2. How the kTBS could benefit from it?


kTBS: Purpose & Composition
---------------------------

While the kTBS has a theoretical background, it is used in practical ways to solve different problems.
People tend to use it to collect data and to get some knowledge from the data.

The data collection has a strong focus on temporality: each event is timestamped, we view the data as a timeline of events.
In the kTBS, an event is called an *obsel* (for observed element). An obsel must contain the following information: the time it happened and its type, it can also contain other information.
The timeline that contains series of obsels is called a *trace*.
For organizational purpose, traces are stored in a *base*, and a kTBS manages multiple bases.

In order to get knowledge from the collected data, a user makes requests to the kTBS. For example, a request can filter out some obsel types, or merge obsels, etc. leading to new information.
The kTBS is a bit like a database. A major difference between standard SQL databases and the kTBS is the data format. SQL databases use the concept of tables to store data, whereas the kTBS uses RDF.
In RDF, data is stored as triples to make expressions of the form: subject-predicate-object. This is where the knowledge part of the kTBS comes from. In order to store theses triples we will see later that we need a special piece of software called a *triple store*.

From a more technical point of view, the [main kTBS implementation][gh-ktbs] is in Python. It is split into the kTBS core and the RDFREST interface.
The [kTBS documentation][rtd-ktbs] provides an [abstract API specification][rtd-ktbs-apispec] so developers can implement the kTBS in many languages.

The full kTBS stack looks like this:
```
[ User or Javascript application ]
               ^
               | HTTP
               v
        [ HTTP server ]
               ^
               | Python via WSGI
               v
           [ kTBS ]
               ^
               | Python
               v
          [ RDFlib ]
               ^
               | SPARQL or Python
               v
        [ Triple store ]
```

[gh-ktbs]: https://github.com/ktbs/ktbs
[rtd-ktbs]: kernel-for-trace-based-systems.readthedocs.org/
[rtd-ktbs-apispec]: http://kernel-for-trace-based-systems.readthedocs.org/en/latest/concepts/abstract_api.html


kTBS Performance
----------------


### Triple-store Performance ###

My first task on the kTBS was to analyse its current performance so it can be used later to see if it has improved.

First, we decided to focus on what stores the data: the triple-store. We thought it could be a bottleneck because the kTBS is data intensive, so storing and retrieving data should be as fast as possible. Furthermore, new triple-stores like [Virtuoso][virtuoso] or [Jena][jena] look promising so we wanted to test their performance.

I designed some benchmarks to measure the performance of several triple-stores. The tested triple-stores were:

- RDF stores : [Virtuoso][virtuoso], [Jena][jena] and [4store][4store]
- [Sleepycat][wp-sleepycat]
- SQL databases: MySQL, PostgreSQL, SQLite.

There were 2 benchmarks: one to measure the time to add data into a triple-store, and another one to measure the time to retrieve data from a triple-store. We focused on the later, as it is the more critical for the kTBS.
I will not extend the discussion on these benchmarks here, as I already did so on the [kTBS bench reports][ktbs-reports].

The main result from these benchmarks is that the existing triple-store, Sleepycat, is as good or better than others for what we want to do with the kTBS. We kept Sleepycat as moving to another triple-store would require a significant amount of work for probably no performance improvement. I had some trouble with some RDF stores ([see why][discarding-stores]), namely Jena and 4store, and maybe they could outperform Sleepycat and Virtuoso. But we didn't want to spend time to diagnose, fix, and tweak them, we wanted to benchmark stores that perform well pretty much out of the box.

[//]: # (TODO: lien vers tutoriel benchmark)

[wp-sleepycat]: https://en.wikipedia.org/wiki/Berkeley_DB
[virtuoso]: http://virtuoso.openlinksw.com/
[jena]: https://jena.apache.org/
[4store]: http://4store.org/
[ktbs-reports]: http://ktbs-bench.readthedocs.org/en/latest/reports.html
[discarding-stores]: http://ktbs-bench.readthedocs.org/en/latest/reports/bench_selected_stores.html


### Running the kTBS in parallel ###

The next step for improving the kTBS performance was to tune the kTBS itself. Eventhough the implementation was not easy, the idea is fairly simple: make the kTBS run in several instances to take advantage of the request parallelization.

If the kTBS doesn't run in parallel then it will run the requests sequentially, one after antoher. For example, if a user does a request to get all the obsels on a base that takes 1 min, then another that just want to post a single obsel will have to wait a full minute. But if the kTBS runs in parallel, it will accept multiple requests to be run *at the same time*, leading to a shorter response time. For the above example, the second user request doesn't have to wait for the first request completion to post the obsel.

Ok, so if parallelization is that great, why didn't we use it right away? In order for a program to run well in parallel we have to make sure that their is no side effects when it runs that way. Such side effects can occur when multiple instances of a program write to shared data. If an instance doesn't check that it is the only instance able to write to this shared data, then another instance may write to it without the first one knowing. The first instance then use the shared data value from the second instance, which could lead to bugs.
In order to prevent this kind of problem, we have to make the program [thread safe][wp-thread-safe].

#### Implementing the lock mechanism ####

We will now dive into the implementation of the lock mechanism that attempts to make the kTBS thread-safe. This work is spread across multiple commits and has been [merged][ktbs-lock-commit] into the kTBS development branch.

As I said previously, we had to make the kTBS thread-safe. To do this we must control where side-effects occur: at the change of a resource (either a delete, edit or create).
We use the concept of *named semaphore* to control the side-effects, via the [posix_ipc][posix_ipc-home] python library. It is kind of a counting lock, where the count represents how many threads can interact with some shared data. Typically, we initialize the semaphore counter to 1, meaning that at most one thread can access the shared data at a given time. If a thread access the shared data, it decrements the semaphore counter, bringing it to 0 in our case. If a semaphore counter is 0 and a thread wants to access the protected data, then it has to wait until the counter is more than 0 to interact with the data.

[posix_ipc-home]: http://semanchuk.com/philip/posix_ipc/

The code responsible for the locking mechanism is in the source file [`engine/lock.py`][ktbs-source-lock], and the core method is [`WithLockMixin.lock`](https://github.com/ktbs/ktbs/blob/733b4a7e84515682612981f47714acd1feac5e95/lib/ktbs/engine/lock.py#L68-L116). I will now try to explain how the locking mechanism work by looking at this method step by step.

I will first explain the main work with the semaphore, that is the code in the [else block](https://github.com/ktbs/ktbs/blob/733b4a7e84515682612981f47714acd1feac5e95/lib/ktbs/engine/lock.py#L89-L116) (I intentionnaly left out some code bits, marked with  `# ...`, to focus on the semaphore concept):

```python
semaphore = self._get_semaphore()
try:
    semaphore.acquire(timeout)
    # ...
    try:
        # ...
        yield
    except:
        # ...
        raise
    finally:
        # ...
        semaphore.release()
        semaphore.close()
        # ...
except posix_ipc.BusyError:
    # ...
    raise posix_ipc.BusyError(error_msg)
```

We start by initiliazing the semaphore with
```python
semaphore = self._get_semaphore()
```
Then we use a `try`/`except` statement. The `try` block is where we are going to use the lock, we first acquire the semaphore:
```python
semaphore.acquire(timeout)
```
This means that we decrement the lock count (we bring it from 1 to 0), and are now expected to be the only thread to work on the resource. If we fail to acquire the lock after `timeout` seconds it should be that another thread is busy with the resource. After this failure, a `BusyError` exception is raised and we reach the [`except posix_ipc.BusyError` block](https://github.com/ktbs/ktbs/blob/733b4a7e84515682612981f47714acd1feac5e95/lib/ktbs/engine/lock.py#L112-L116). In this block we essentially re-raise the exception in order to log it.

Let's get back to the normal process and see what happens if everything goes well after we acquire the semaphore. At this point we are safe to run code that has side effect on a resource, because we are the only one changing it. We use a `try`/`except`/`finally` statement. In the [`try` block](https://github.com/ktbs/ktbs/blob/733b4a7e84515682612981f47714acd1feac5e95/lib/ktbs/engine/lock.py#L97-L102), we run the code that must be locked by using the `yield` statement:

```python
    try:
        # ...
        yield
```

If anything goes wrong, we catch the exception in the [`except` block](https://github.com/ktbs/ktbs/blob/733b4a7e84515682612981f47714acd1feac5e95/lib/ktbs/engine/lock.py#L103-L105) and re-raise it:

```python
    except:
        # ...
        raise
```

Last, we make sure to clean after ourselves in the [`finally` block](https://github.com/ktbs/ktbs/blob/733b4a7e84515682612981f47714acd1feac5e95/lib/ktbs/engine/lock.py#L106-L110). The `finally` block is always executed, even if there were failures in the `try` and `except` blocks. This is a good way for us to make sure to release and close the lock:
```python
    finally:
        # ...
        semaphore.release()
        semaphore.close()
```


There are two code blocks that I didn't mention yet :

- The first one is the first [`if` block](https://github.com/ktbs/ktbs/blob/7a1e53cf3e7710c76ecad1b0814220e5e3c8b727/lib/ktbs/engine/lock.py#L85-L86). Its purpose is to set a timeout value for the lock.

- The second one is the second [`if` block](https://github.com/ktbs/ktbs/blob/7a1e53cf3e7710c76ecad1b0814220e5e3c8b727/lib/ktbs/engine/lock.py#L88-L91). We check if the thread that asks for the lock is a re-entering thread. This happens for example when one deletes a trace: we lock the base via [`InBase.delete`](https://github.com/ktbs/ktbs/blob/7a1e53cf3e7710c76ecad1b0814220e5e3c8b727/lib/ktbs/engine/base.py#L135), but the call to [`super()`](https://github.com/ktbs/ktbs/blob/7a1e53cf3e7710c76ecad1b0814220e5e3c8b727/lib/ktbs/engine/base.py#L136) leads to a call to [`edit()`](https://github.com/ktbs/ktbs/blob/7a1e53cf3e7710c76ecad1b0814220e5e3c8b727/lib/ktbs/engine/lock.py#L130) that also wants to lock the base. It is safe to let it bypass the lock at this point because it already locked the ressource upper in the call stack. That what's we do with this `if` block.


#### Using the lock ####
The lock mechanism is provided as a mixin class [`WithLockMixin`][ktbs-source-withlockmixin]. This means that every class that has `WithLockMixin` has a parent will be able to use the locking mechanism. This is the case for the classes [`KtbsRoot`][ktbs-source-ktbsroot] and [`Base`][ktbs-source-base] that inherits `WithLockMixin` as their *first* parent.

As you can see in the `WithLockMixin` class, we have implemented the methods: `edit`, `post_grah`, `delete`. The classes `KtbsRoot` and `Base` already had those methods from other class inheritance. What we wanted to do is wrap these existing methods inside a `lock`. We used the concept of [context][py-doc-context] in Python to do this. The idea is that you have a function, called a *context manager*, that will encapsulate some code and you want to do things before and/or after the code. If we decorate a function with `@contextmanager` then we can run the encapsulated code with a `yield` statement:

```python
@contextmanager
def greetings():
    print('Hello!')
    yield
    print('Goodbye.')

with greetings():
    print 'Why not 42?'
```
will display
```
Hello!
Why not 42?
Goodbye.
```

We did this with the `lock` method. We encapsulate code that may have side effects around a lock:

```python
with lock():
    function_with_side_effects()
    # ...
```

Now, because the `KtbsRoot` and `Base` classes already inherits the methods `edit`, `post_grah`, `delete`, we just have to override them in `WithLockMixin` and put a lock around them. This gives something like:
```python
class WithLockMixin:
    # ...
    def post_graph():
        with self.lock():
            super(WithLockMixin, self).post_graph()
```

This way when `post_graph` is called on a `Base`, it will call the `WithLockMixin` method first. The method will get the lock, run the main code of `post_graph` that is defined in another class that `Base` inherits from, and then close the lock.


#### Running the kTBS in multi-process behind Apache ####

In order to benefit from the multi-process implementation we need a way to run it this way. Apache allows us to do so. In the kTBS documentation on how to [setup Apache][ktbs-tuto] there is the following directive for `mod_wsgi`:

```ApacheConf
<IfModule mod_wsgi.c>
    WSGIScriptAlias /ktbs /home/user/ktbs-env/application.wsgi
    WSGIDaemonProcess myktbs processes=1 threads=1 python-path=/home/user/ktbs-env/ktbs/lib
    WSGIProcessGroup myktbs
</IfModule>
```

You can see on the line that starts with `WSGIDaemonProcess` that we can define the number of processes and/or threads that Apache will use for the WSGI application (the kTBS in this case). You can tweak `processes` and `threads` to have better performance. For example, this is what I am using on my machine:

```ApacheConf
WSGIDaemonProcess myktbs processes=4 threads=1 python-path=/home/user/ktbs-env/ktbs/lib
```


#### Results ####

When comparing a kTBS running on a single process and a kTBS running on 4 processes we typically see a performance improvement by a factor of 2 to 3 (in term of requests per second). Keep in mind that this result depends on:

- what machine you are running on: a server? a laptop that has other tasks to do?

- the queries you are doing on the kTBS: a GET request on a 30k obsels trace will take long on both cases, but if you use multi-processing the following requests won't block so the throughput will be higher.

[wp-thread-safe]: https://en.wikipedia.org/wiki/Thread_safety
[ktbs-lock-commit]: https://github.com/ktbs/ktbs/commit/4be847439e43cda60d7865e5c9e3eb8fe1bc872c
[ktbs-source-ires]: https://github.com/ktbs/ktbs/blob/733b4a7e84515682612981f47714acd1feac5e95/lib/rdfrest/interface.py#L91
[ktbs-source-lock]: https://github.com/ktbs/ktbs/blob/733b4a7e84515682612981f47714acd1feac5e95/lib/ktbs/engine/lock.py
[ktbs-source-withlockmixin]: https://github.com/ktbs/ktbs/blob/733b4a7e84515682612981f47714acd1feac5e95/lib/ktbs/engine/lock.py#L44
[ktbs-source-ktbsroot]: https://github.com/ktbs/ktbs/blob/733b4a7e84515682612981f47714acd1feac5e95/lib/ktbs/engine/ktbs_root.py#L30
[ktbs-source-base]: https://github.com/ktbs/ktbs/blob/733b4a7e84515682612981f47714acd1feac5e95/lib/ktbs/engine/base.py#L30
[ktbs-source-lock]: https://github.com/ktbs/ktbs/blob/733b4a7e84515682612981f47714acd1feac5e95/lib/ktbs/engine/lock.py
[py-doc-context]: https://docs.python.org/2/reference/datamodel.html#context-managers
[ktbs-tuto]: http://kernel-for-trace-based-systems.readthedocs.org/en/latest/tutorials/install.html



Authentication with SSO
-----------------------

### What is OpenID Connect?

### How the kTBS could benefit from it?

