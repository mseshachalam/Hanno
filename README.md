What's Hanno?
=============

Hanno is an asynchronous event loop system for writing real-time applications in PHP. Unlike some similar systems, it is based around generators, avoiding the need for callbacks and greatly simplifying both reading and writing code.

As it uses generators, it requires at least PHP 5.5.

The name comes from the Japanese verb for to react, 反応する (*Han'nō suru*), a reference to the use of reactors and a nod to [ReactPHP](http://reactphp.org/). Because the Japanese word the name came from is pronounced *Han'nō*, you actually ought to pronounce "Hanno" with two separate 'n' sounds like "Han'no", but in practise everyone (including myself) is just going to say "Hanno" (/ˈhæ.noʊ/).

How do I use it?
================

Hanno is a modern, autoloaded library that supports Composer and is available via Packagist. Thus, simply require the `"ajf\Hanno"` package and you're good to go.

Hanno's basic structural piece is a *task*. Tasks in Hanno are simply plain `Generator`s which are iterated over until they complete. The idea behind this is that you use `yield` to yield control to other tasks when you need to wait for something else, meaning you can efficiently do multiple things at once (handle multiple requests for example), without each getting in eachother's way and without needing multiple threads. To take a rather trivial example, the following generator produces a fairly useless task:

```php
function counter() {
    for ($i = 1; $i <= 3; $i++) {
        yield;
        echo "$i\n";
    }
}
```

In order to run multiple tasks at once, Hanno provides a `Reactor` class to handle multiple tasks. For example, here we create a reactor and add two counters to it:

```php
use ajf\Hanno as Hanno;
$reactor = new Hanno\Reactor;
$reactor->addTask(counter());
$reactor->addTask(counter());
$reactor->run();
```

For each step the reactor is run, it shall execute each task, run it for one iteration (i.e. call its `->next()` method), and if it is not finished (i.e., `->valid()` returns true), schedule it to be run next time. Note that you can also run the reactor as a task, thus can in fact run reactors in reactors! For example, we could do this:

```php
$reactor1 = new Hanno\Reactor;
$reactor1->addTask(counter());
$reactor1->addTask(counter());

$reactor2 = new Hanno\Reactor;
$reactor2->addTask(counter());

$ueber_reactor = new Hanno\Reactor;
$ueber_reactor->addTask($reactor1->runAsTask());
$ueber_reactor->addTask($reactor2->runAsTask());
```

You can nest a reactor inside itself, but Bad Things<sup>TM</sup> *will* happen, so don't try to. ;)

So, now we're familiar with the basics of tasks and reactors. But what if we want to do something actually useful? Reactors don't ignore the values yielded by `yield`. By using the special key `'until'`, you can ask a reactor to suspend your task until something is done, and get back the result of that task. That something must be an `Awaitable` class. For example:

```php
function myTask($myAwaitable) {
    yield;
    
    $result = (yield 'until' => $myAwaitable);
    foobar($result);
}
```

But where can you get an awaitable from? Well, an awaitable is typically returned from a function which does something. It's a very simple object containing a task that the reactor will run, and which the reactor can 'listen' to to know when the job is done and what its result was so your task can start running again. One place you can get awaitables from is Hanno's built-in `Stream` class, which wraps PHP streams. Let's say we want to find out our IP address. We could use the http://ip.jsontest.com/ API like this:

```php
function getIPTask() {
    yield;
    
    $stream = new Hanno\Stream("http://ip.jsontest.com/", "r");
    $data = (yield 'until' => $stream->read());
    $data = json_decode($data);
    var_dump($data->ip);
}
$reactor = new Hanno\Reactor;
$reactor->addTask(getIPTask());
$reactor->run();
```

The constructor takes either a pre-existing resource, or a URL and a mode (like you'd give to fopen). Calling the `->read` method with no argument will read all of the stream and give us an awaitable that'll ultimately result in a string. Then we just `json_decode`.

And that was the basics of Hanno. Have fun!