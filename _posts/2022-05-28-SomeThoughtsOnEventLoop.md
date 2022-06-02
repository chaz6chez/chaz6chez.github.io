---
layout: post
title: "EventLoop的一些心得"
date: 2022-05-28 22:10:00
image: ''
description: "在自己实现PHP的EventLoop库时的一些心得体会"
tags:
- Eventloop
- PHP
categories:
- Chaz6chez

---


> 🐇 最新更新于2020-05-30

# 前言

最早接触reactor模型的时候，应该是在参与一个叫zanphp项目的时候，他是一个类似swoole的php拓展项目，当然它们之间的故事我就不多说了，也有一些冲突和迷茫；在那个时间段的PHP发展还是很蓬勃向上的，那时候的滴滴、有赞、百度都有很多很多PHP项目，那时候的原生PHP有许多许多的瓶颈，所以国内那时候涌现了很多使用C来为PHP加速的开发者。
随着PHP慢慢发展，PHP的特性越来越丰富，性能也越来越好，而PECL库里的拓展也经历了这么多年的洗礼和冲刷，越来越稳固。现如今更多的PHP开发者围绕着原生PHP做业务，其实我觉得这反而是一个好现象，专业的人做专业的事，更多热爱它的人愿意留下做贡献，社区虽然没有像那段时间一样的向四面八方高速发展，但体现出来的是更有方向感的一种进步。
在这样一种状况下，外加上我接触了比较多其他的语言和项目，激发了我想利用现有的扩展结合原生PHP去做一些看起来厉害的、用起来骚气的一些库或者组件，做一些可能重复造轮子的事儿；当然，一方面是希望尽可能的做一些新轮子，另一方面也是希望能够通过实践，更深的理解某些知识。

# 开始

我打算做的是一个轻量的任务调度服务，原本计划是Golang做开发，在业内大部分人的评价来说，Golang像是一个高级的PHP/高级Python；其实用Golang做一个任务调度服务来说是件比较简单的事儿，而且市面上也有比较多的调度服务，这其实是一个重复轮子的事儿，考虑到这个情况，我思考了一下，打算先用PHP实现一个这样的服务。

通常来说PHP语言都围绕着PHP-FPM来做的开发，毕竟PHP业内最工业化的架构就是LNMP/LAMP，但是这就不符合“轻量”这一特性了，所以我把目光锁定在了workerman、amphp、reactphp上；但为了深入了解这些PHP中优秀的reacto模型框架，我决定自己撸一个event-loop；

# 研究

我分别研究了 [walkor/workerman](https://github.com/walkor/workerman) 、 [reactphp/event-loop](https://github.com/reactphp/event-loop)，发现了一些比较有趣的事儿；

## 为什么workerman的性能会高于reactphp呢？

- 一方面要看这个PHP框架的C含量有多少了，越多性能就越高。
  这样的思维普遍存在在各种语言/框架中，比如PHP的YAF、Phalcon、swoole等，比如Python早期的cpython、numpy等；毕竟C语言作为祖师爷的存在，更贴近系统，对于内存、系统的调用来的更直接，只要编码足够优秀，性能就可以足够优秀。

- 一方面要关注开发的模式
  通常来说，我们选用一款开发框架都是用于开发业务的，这里面会面对和使用各种各样已经存在的轮子，比如PDO、composer组件等，这些组件/拓展功能的存在已经有一定的历史了，他们从过去到现在的发展也大多聚焦在blocking-IO的模式上，也就是同步阻塞的模式，因为这种模式更简单直接，每一行的代码都是顺序进行下去，好掌控，也好排查，这样的开发模式可以让程序员降低不少的心智负担，聚焦在业务上。

我们回过头聚焦 reactphp和workerman；我们大部分的开发模式像上述说的，我们都会聚焦在BIO的模式上，一方面是轮子是这么做的，一方面是这样的开发速度更快更直接；抛开进程不说，如果我们在单进程下使用reactor模型，那么这样的event-loop就会退化成和PHP-FPM一样的阻塞等待的程序，workerman是如何做的呢？多进程，多开了比较多的进程来并行处理业务，也利用了linux的端口复用(SO_REUSEADDR、SO_REUSEPORT）；reactphp利用的是异步编程的方式，尽可能地不阻塞event-loop；那么这里reactphp的代价就是需要为这套编程方式实现许许多多的异步客户端，做很多轮子的工作，这里包含解决回调地狱的 reactphp/promise 等，因为一旦阻塞了event-loop，它便会退化。

除此之外，还有提到的含C量上，以下我会用代码解释这个含C量具体提现在哪儿：

- workerman的event-loop，以ext-event举例

```php
<?php 
/**
 * This file is part of workerman.
 *
 * Licensed under The MIT License
 * For full copyright and license information, please see the MIT-LICENSE.txt
 * Redistributions of files must retain the above copyright notice.
 *
 * @author    有个鬼<42765633@qq.com>
 * @copyright 有个鬼<42765633@qq.com>
 * @link      http://www.workerman.net/
 * @license   http://www.opensource.org/licenses/mit-license.php MIT License
 */
namespace Workerman\Events;

use Workerman\Worker;

/**
 * libevent eventloop
 */
class Event implements EventInterface
{
    /**
     * Event base.
     * @var object
     */
    protected $_eventBase = null;
    
    /**
     * All listeners for read/write event.
     * @var array
     */
    protected $_allEvents = array();
    
    /**
     * Event listeners of signal.
     * @var array
     */
    protected $_eventSignal = array();
    
    /**
     * All timer event listeners.
     * [func, args, event, flag, time_interval]
     * @var array
     */
    protected $_eventTimer = array();

    /**
     * Timer id.
     * @var int
     */
    protected static $_timerId = 1;
    
    /**
     * construct
     * @return void
     */
    public function __construct()
    {
        if (\class_exists('\\\\EventBase', false)) {
            $class_name = '\\\\EventBase';
        } else {
            $class_name = '\EventBase';
        }
        $this->_eventBase = new $class_name();
    }
   
    /**
     * @see EventInterface::add()
     */
    public function add($fd, $flag, $func, $args=array())
    {
        if (\class_exists('\\\\Event', false)) {
            $class_name = '\\\\Event';
        } else {
            $class_name = '\Event';
        }
        switch ($flag) {
            case self::EV_SIGNAL:

                $fd_key = (int)$fd;
                $event = $class_name::signal($this->_eventBase, $fd, $func);
                if (!$event||!$event->add()) {
                    return false;
                }
                $this->_eventSignal[$fd_key] = $event;
                return true;

            case self::EV_TIMER:
            case self::EV_TIMER_ONCE:

                $param = array($func, (array)$args, $flag, $fd, self::$_timerId);
                $event = new $class_name($this->_eventBase, -1, $class_name::TIMEOUT|$class_name::PERSIST, array($this, "timerCallback"), $param);
                if (!$event||!$event->addTimer($fd)) {
                    return false;
                }
                $this->_eventTimer[self::$_timerId] = $event;
                return self::$_timerId++;
                
            default :
                $fd_key = (int)$fd;
                $real_flag = $flag === self::EV_READ ? $class_name::READ | $class_name::PERSIST : $class_name::WRITE | $class_name::PERSIST;
                $event = new $class_name($this->_eventBase, $fd, $real_flag, $func, $fd);
                if (!$event||!$event->add()) {
                    return false;
                }
                $this->_allEvents[$fd_key][$flag] = $event;
                return true;
        }
    }
    
    /**
     * @see Events\EventInterface::del()
     */
    public function del($fd, $flag)
    {
        switch ($flag) {

            case self::EV_READ:
            case self::EV_WRITE:

                $fd_key = (int)$fd;
                if (isset($this->_allEvents[$fd_key][$flag])) {
                    $this->_allEvents[$fd_key][$flag]->del();
                    unset($this->_allEvents[$fd_key][$flag]);
                }
                if (empty($this->_allEvents[$fd_key])) {
                    unset($this->_allEvents[$fd_key]);
                }
                break;

            case  self::EV_SIGNAL:
                $fd_key = (int)$fd;
                if (isset($this->_eventSignal[$fd_key])) {
                    $this->_eventSignal[$fd_key]->del();
                    unset($this->_eventSignal[$fd_key]);
                }
                break;

            case self::EV_TIMER:
            case self::EV_TIMER_ONCE:
                if (isset($this->_eventTimer[$fd])) {
                    $this->_eventTimer[$fd]->del();
                    unset($this->_eventTimer[$fd]);
                }
                break;
        }
        return true;
    }
    
    /**
     * Timer callback.
     * @param int|null $fd
     * @param int $what
     * @param int $timer_id
     */
    public function timerCallback($fd, $what, $param)
    {
        $timer_id = $param[4];
        
        if ($param[2] === self::EV_TIMER_ONCE) {
            $this->_eventTimer[$timer_id]->del();
            unset($this->_eventTimer[$timer_id]);
        }

        try {
            \call_user_func_array($param[0], $param[1]);
        } catch (\Exception $e) {
            Worker::stopAll(250, $e);
        } catch (\Error $e) {
            Worker::stopAll(250, $e);
        }
    }
    
    /**
     * @see Events\EventInterface::clearAllTimer() 
     * @return void
     */
    public function clearAllTimer()
    {
        foreach ($this->_eventTimer as $event) {
            $event->del();
        }
        $this->_eventTimer = array();
    }
     

    /**
     * @see EventInterface::loop()
     */
    public function loop()
    {
        $this->_eventBase->loop();
    }

    /**
     * Destroy loop.
     *
     * @return void
     */
    public function destroy()
    {
        $this->_eventBase->exit();
    }

    /**
     * Get timer count.
     *
     * @return integer
     */
    public function getTimerCount()
    {
        return \count($this->_eventTimer);
    }
}
```
以上的代码中以 loop() 这段代码来说，workerman直接将事件的循环交给了ext-event拓展，因为ext-event底层使用的是libevent库，是一个C语言的事件循环库，你可以理解为把一个循环体交给了C语言。
C语言的循环肯定是比PHP的循环效率更高的，在同等时间单位下，循环的次数肯定是比PHP要高不少的。

- reactphp的event-loop，以ext-event举例

```php
<?php

namespace React\EventLoop;

use BadMethodCallException;
use Event;
use EventBase;
use React\EventLoop\Tick\FutureTickQueue;
use React\EventLoop\Timer\Timer;
use SplObjectStorage;

/**
 * An `ext-event` based event loop.
 *
 * This uses the [`event` PECL extension](https://pecl.php.net/package/event),
 * that provides an interface to `libevent` library.
 * `libevent` itself supports a number of system-specific backends (epoll, kqueue).
 *
 * This loop is known to work with PHP 5.4 through PHP 8+.
 *
 * @link https://pecl.php.net/package/event
 */
final class ExtEventLoop implements LoopInterface
{
    private $eventBase;
    private $futureTickQueue;
    private $timerCallback;
    private $timerEvents;
    private $streamCallback;
    private $readEvents = array();
    private $writeEvents = array();
    private $readListeners = array();
    private $writeListeners = array();
    private $readRefs = array();
    private $writeRefs = array();
    private $running;
    private $signals;
    private $signalEvents = array();

    public function __construct()
    {
        if (!\class_exists('EventBase', false)) {
            throw new BadMethodCallException('Cannot create ExtEventLoop, ext-event extension missing');
        }

        // support arbitrary file descriptors and not just sockets
        // Windows only has limited file descriptor support, so do not require this (will fail otherwise)
        // @link http://www.wangafu.net/~nickm/libevent-book/Ref2_eventbase.html#_setting_up_a_complicated_event_base
        $config = new \EventConfig();
        if (\DIRECTORY_SEPARATOR !== '\\') {
            $config->requireFeatures(\EventConfig::FEATURE_FDS);
        }

        $this->eventBase = new EventBase($config);
        $this->futureTickQueue = new FutureTickQueue();
        $this->timerEvents = new SplObjectStorage();
        $this->signals = new SignalsHandler();

        $this->createTimerCallback();
        $this->createStreamCallback();
    }

    public function __destruct()
    {
        // explicitly clear all references to Event objects to prevent SEGFAULTs on Windows
        foreach ($this->timerEvents as $timer) {
            $this->timerEvents->detach($timer);
        }

        $this->readEvents = array();
        $this->writeEvents = array();
    }

    public function addReadStream($stream, $listener)
    {
        $key = (int) $stream;
        if (isset($this->readListeners[$key])) {
            return;
        }

        $event = new Event($this->eventBase, $stream, Event::PERSIST | Event::READ, $this->streamCallback);
        $event->add();
        $this->readEvents[$key] = $event;
        $this->readListeners[$key] = $listener;

        // ext-event does not increase refcount on stream resources for PHP 7+
        // manually keep track of stream resource to prevent premature garbage collection
        if (\PHP_VERSION_ID >= 70000) {
            $this->readRefs[$key] = $stream;
        }
    }

    public function addWriteStream($stream, $listener)
    {
        $key = (int) $stream;
        if (isset($this->writeListeners[$key])) {
            return;
        }

        $event = new Event($this->eventBase, $stream, Event::PERSIST | Event::WRITE, $this->streamCallback);
        $event->add();
        $this->writeEvents[$key] = $event;
        $this->writeListeners[$key] = $listener;

        // ext-event does not increase refcount on stream resources for PHP 7+
        // manually keep track of stream resource to prevent premature garbage collection
        if (\PHP_VERSION_ID >= 70000) {
            $this->writeRefs[$key] = $stream;
        }
    }

    public function removeReadStream($stream)
    {
        $key = (int) $stream;

        if (isset($this->readEvents[$key])) {
            $this->readEvents[$key]->free();
            unset(
                $this->readEvents[$key],
                $this->readListeners[$key],
                $this->readRefs[$key]
            );
        }
    }

    public function removeWriteStream($stream)
    {
        $key = (int) $stream;

        if (isset($this->writeEvents[$key])) {
            $this->writeEvents[$key]->free();
            unset(
                $this->writeEvents[$key],
                $this->writeListeners[$key],
                $this->writeRefs[$key]
            );
        }
    }

    public function addTimer($interval, $callback)
    {
        $timer = new Timer($interval, $callback, false);

        $this->scheduleTimer($timer);

        return $timer;
    }

    public function addPeriodicTimer($interval, $callback)
    {
        $timer = new Timer($interval, $callback, true);

        $this->scheduleTimer($timer);

        return $timer;
    }

    public function cancelTimer(TimerInterface $timer)
    {
        if ($this->timerEvents->contains($timer)) {
            $this->timerEvents[$timer]->free();
            $this->timerEvents->detach($timer);
        }
    }

    public function futureTick($listener)
    {
        $this->futureTickQueue->add($listener);
    }

    public function addSignal($signal, $listener)
    {
        $this->signals->add($signal, $listener);

        if (!isset($this->signalEvents[$signal])) {
            $this->signalEvents[$signal] = Event::signal($this->eventBase, $signal, array($this->signals, 'call'));
            $this->signalEvents[$signal]->add();
        }
    }

    public function removeSignal($signal, $listener)
    {
        $this->signals->remove($signal, $listener);

        if (isset($this->signalEvents[$signal]) && $this->signals->count($signal) === 0) {
            $this->signalEvents[$signal]->free();
            unset($this->signalEvents[$signal]);
        }
    }

    public function run()
    {
        $this->running = true;

        while ($this->running) {
            $this->futureTickQueue->tick();

            $flags = EventBase::LOOP_ONCE;
            if (!$this->running || !$this->futureTickQueue->isEmpty()) {
                $flags |= EventBase::LOOP_NONBLOCK;
            } elseif (!$this->readEvents && !$this->writeEvents && !$this->timerEvents->count() && $this->signals->isEmpty()) {
                break;
            }

            $this->eventBase->loop($flags);
        }
    }

    public function stop()
    {
        $this->running = false;
    }

    /**
     * Schedule a timer for execution.
     *
     * @param TimerInterface $timer
     */
    private function scheduleTimer(TimerInterface $timer)
    {
        $flags = Event::TIMEOUT;

        if ($timer->isPeriodic()) {
            $flags |= Event::PERSIST;
        }

        $event = new Event($this->eventBase, -1, $flags, $this->timerCallback, $timer);
        $this->timerEvents[$timer] = $event;

        $event->add($timer->getInterval());
    }

    /**
     * Create a callback used as the target of timer events.
     *
     * A reference is kept to the callback for the lifetime of the loop
     * to prevent "Cannot destroy active lambda function" fatal error from
     * the event extension.
     */
    private function createTimerCallback()
    {
        $timers = $this->timerEvents;
        $this->timerCallback = function ($_, $__, $timer) use ($timers) {
            \call_user_func($timer->getCallback(), $timer);

            if (!$timer->isPeriodic() && $timers->contains($timer)) {
                $this->cancelTimer($timer);
            }
        };
    }

    /**
     * Create a callback used as the target of stream events.
     *
     * A reference is kept to the callback for the lifetime of the loop
     * to prevent "Cannot destroy active lambda function" fatal error from
     * the event extension.
     */
    private function createStreamCallback()
    {
        $read =& $this->readListeners;
        $write =& $this->writeListeners;
        $this->streamCallback = function ($stream, $flags) use (&$read, &$write) {
            $key = (int) $stream;

            if (Event::READ === (Event::READ & $flags) && isset($read[$key])) {
                \call_user_func($read[$key], $stream);
            }

            if (Event::WRITE === (Event::WRITE & $flags) && isset($write[$key])) {
                \call_user_func($write[$key], $stream);
            }
        };
    }
}
```

这里 reactphp 的 **run()** 与 workerman 的 **loop()** 不同的是使用了PHP的循环来做的，这可能也是为了更好的掌控循环中的一些事件的优先级，对循环的把控力度更高的缘故。

那么综上所述，workerman 相比较 reactphp 在含C量上是要足很多的，尤其是主要用作常驻功能、事件监听的循环体上，这是最关键的一点，这一点尤其体现在当两者都退化成同步阻塞的业务来说，workerman肯定是比reactphp的性能更高的；但如果使用了reactphp的异步体系的话，这个我没有测试过，但凭经验来说的话，我觉得reactphp可能会更好（没有测试过，大胆猜测以下）；另外swoole是使用C在底层做了很多异步的调度工作，让用户开发更像同步开发，这点就不展开来说了。

# 自行实现event-loop

秉着折腾的心态，我自己实现了一个事件循环库，结合了reacphp和workerman二者，折腾了一下：

**[workbunny/event-loop](https://github.com/workbunny/event-loop)**

在做这个项目的的单元测试的时候，我也遇到了很多坑，有了很多心得，今天就先更新到这，改天再继续说说这些坑和心得。

> 🐇 更新于2020-05-28

## PHP原生loop

使用PHP实现原生loop其实是件比较简单的事儿，这个loop里面需要干的事儿只有三件：
1. 监听系统信号
2. 监听流信息
3. 定时器

1和2分别可以使用PHP相关函数来实现：
1. pcntl_signal_dispatch()，用于监听信号
2. stream_select()，用于获取流数据

那么定时器是如何实现的呢？这里我用的是一个优先队列来保存（**PHP的SPL中有相当多有用的高效的数据结构，推荐大家多关注SPL，也就是PHP标准库**）：

```php
	/** @var SplPriorityQueue 优先队列 */
    protected SplPriorityQueue $_queue;

	/** @inheritDoc */
    public function __construct()
    {
        if(!extension_loaded('pcntl')){
            throw new LoopException('not support: ext-pcntl');
        }
        parent::__construct();

        $this->_queue = new SplPriorityQueue();
        $this->_queue->setExtractFlags(SplPriorityQueue::EXTR_BOTH);
        $this->_readFds = [];
        $this->_writeFds = [];
    }

	/** 执行 */
    protected function _tick(): void
    {
        $count = $this->_queue->count();
        while ($count--){
            $data = $this->_queue->top();
            $runTime = -$data['priority'];
            $timerId = $data['data'];
            /** @var Timer $data */
            if($data = $this->_storage->get($timerId)){
                $repeat = $data->getRepeat();
                $callback = $data->getHandler();
                $timeNow = \hrtime(true) * 1e-9;
                if (($runTime - $timeNow) <= 0) {
                    $this->_queue->extract();
                    \call_user_func($callback);
                    if($repeat !== 0.0){
                        $nextTime = $timeNow + $repeat;
                        $this->_queue->insert($timerId, -$nextTime);
                    }else{
                        $this->delTimer($timerId);
                    }
                }
            }
        }
    }
```

毕竟添加的定时器可能比较多，每个定时器也在当前进程内也会相互阻塞，所以肯定是需要有优先级的，在每个循环周期里，都有一个最接近的定时器，越接近的，优先级越高，具体代码体现在上述 **_tick()** 内；

所以最后我的 **loop()** 实现如下：

```php
	/** @inheritDoc */
    public function loop(): void
    {
        $this->_stopped = false;

        while (!$this->_stopped) {
            if(!$this->_readFds and !$this->_writeFds and !$this->_signals and $this->_storage->isEmpty()){
                break;
            }

            \pcntl_signal_dispatch();

            $writes = $this->_writeFds;
            $reads = $this->_readFds;
            $excepts = [];
            foreach ($writes as $key => $socket) {
                if (!isset($reads[$key]) && @\ftell($socket) === 0) {
                    $excepts[$key] = $socket;
                }
            }

            if($writes or $reads or $excepts){
                try {
                    @stream_select($reads, $writes, $excepts, 0,200000);
                } catch (\Throwable $e) {}

                foreach ($reads as $stream) {
                    $key = (int)$stream;
                    if (isset($this->_reads[$key])) {
                        ($this->_reads[$key])($stream);
                    }
                }

                foreach ($writes as $stream) {
                    $key = (int)$stream;
                    if (isset($this->_writes[$key])) {
                        ($this->_writes[$key])($stream);
                    }
                }
            }

            $this->_tick();
        }
    }
```

**stream_select()** 这里一开始我使用了官方推荐的200000 微秒作为阻塞等待的时长，因为这样可以降低CPU占用率，但是相当于每一次loop都至少会阻塞200000微秒(也就是200ms)，那么loop一周期的时长就太长了，单位秒内loop的次数就大大降低，所以我就想试试如果改成0会如何；结果发现CPU的占用变得很高，虽然PHP内部应该是对其做了处理，但是还是很高。

这是什么原因呢？这个大概需要讲到linux的时间分片算法相关的内容了，这个地方的内容我的另一篇分享里有提到 [趣谈程序演变的过程](https://chaz6chez.cn/chaz6chez/2022/05/18/InterestingTalkAboutEvolutionOfProgram.html)，大概意思就是如果你的这个程序不主动出让CPU，整个系统调度为了方便、高效，那么你的进程分配在CPU-0上，下次继续依然会在CPU-0上，那么就是我们通常听到的“不能利用多核”（当然，如果是单进程，利用不利用多核其实意义不大）；

为了减少CPU占用，可以使用主动出让CPU，每种语言的sleep函数或者方法都是可以出让CPU的，所以我在每个loop周期最后都加入了usleep(0)，这里虽然使用的是0的入参，但是出让CPU依然有效，sleep(0)同理。

```php
	/** @inheritDoc */
    public function loop(): void
    {
        $this->_stopped = false;

        while (!$this->_stopped) {
            if(!$this->_readFds and !$this->_writeFds and !$this->_signals and $this->_storage->isEmpty()){
                break;
            }

            \pcntl_signal_dispatch();

            $writes = $this->_writeFds;
            $reads = $this->_readFds;
            $excepts = [];
            foreach ($writes as $key => $socket) {
                if (!isset($reads[$key]) && @\ftell($socket) === 0) {
                    $excepts[$key] = $socket;
                }
            }

            if($writes or $reads or $excepts){
                try {
                    @stream_select($reads, $writes, $excepts, 0,0);
                } catch (\Throwable $e) {}

                foreach ($reads as $stream) {
                    $key = (int)$stream;
                    if (isset($this->_reads[$key])) {
                        ($this->_reads[$key])($stream);
                    }
                }

                foreach ($writes as $stream) {
                    $key = (int)$stream;
                    if (isset($this->_writes[$key])) {
                        ($this->_writes[$key])($stream);
                    }
                }
            }

            $this->_tick();

            usleep(0);
        }
    }
```

因为这个是原生PHP的loop，所以需要自行考虑循环内的业务逻辑及先后顺序；而其他的如event、ev，我直接使用了他们的loop，对比原生loop来说，会更简单理解一些，这里就不多提了，代码里面都有。


# 测试

在做测试的时候我还是没有那么顺利，也遇到了不少问题，发现了不少之前没有在意的一些东西，先说结论：

1. PHP实现的原生loop要慢挺多的；
2. Event拓展还是最稳健的一个选择；
3. Ev拓展有一点小坑，可能是我固有思维导致的；
4. Swoole/OpenSwoole个人觉得还是不稳当，最好有阅读C源代码的能力；

## PHP实现的原生loop要慢挺多的

这个我没有刻意做一些性能测试，我在测试的时候使用的PHPunit，用例都是一致的（除了Ev和OpenSwoole有几个用例特殊外），单凭运行时候的进度就能肉眼可见的观察出区别，还是挺明显的。

## Event拓展还是最稳健的一个选择

这个结论是因为我做测试的时候这个拓展是全程没有报错，并且很顺利很高效的完成了，对于我来说，心智负担约等于0，我个人是比较推荐使用这个拓展的。

## Ev拓展有一点小坑

这个结论主要是体现在如下：

```php
use EvLoop as BaseEvLoop;

	/** @var BaseEvLoop loop */
    protected BaseEvLoop $_loop;

    /** @inheritDoc */
    public function __construct()
    {
        if(!extension_loaded('ev')){
            throw new LoopException('ext-ev not support');
        }

        parent::__construct();

        $this->_loop = new BaseEvLoop();
    }

	/** @inheritDoc */
    public function addReadStream($stream, Closure $handler): void
    {
        if(is_resource($stream) and !isset($this->_reads[$key = (int)$stream])){
            $event = $this->_loop->io($stream, Ev::READ, $handler);
            $this->_reads[$key] = $event;
            $this->_readFds[$key] = $stream;
        }
    }
```

这是一个我实现的注册读取流回调的方法，最开始我的实现方式不是调用 **$this->_loop->io()** ，而是如下的实现方式：

```php
$event = new EvIo($stream,Ev::READ, $handler);
$event->start();
```

结果并没有生效，事件回调并没有被成功注册进入，也可能是我的使用方式错了。

在这个地方，注册成功的回调，我原本以为会将注册的stream传入我的回调函数，但没想到传入的是一个EvIo对象；如下：

```php
# 我以为
function(resource $stream){}

# 实际上
function(EvIo $stream){}
```

但这个其实还好，我可以通过 **$stream->fd** 获取对应的resource流，但是没想到这里却又有一个坑；我在注册的时候通过 **(int)$stream** 获得的resource id，并且将其id和resource以KV的形式保存在_readFds属性中``` $this->_readFds[$key] = $stream; ``` ；但没有想到，回调传入的EvIo对象中通过fd属性获取的resource id竟然和我注册时候的id不对等，而且每次都不对等，相当于每一次都是新的；这时候我就懵逼了；

但还好，我发现每次传入的EvIo对象是同一个，我索性以EvIo对象做判断，通过 **spl_object_hash()** 获取EvIo对象的id，将注册代码改为了如下：

```php

	/** @inheritDoc */
    public function addReadStream($stream, Closure $handler): void
    {
        if(is_resource($stream) and !isset($this->_reads[$key = (int)$stream])){
            $event = $this->_loop->io($stream, Ev::READ, $handler);
            $this->_reads[$key] = $event;
            $this->_readFds[spl_object_hash($event)] = $stream;
        }
    }
```

那么对应的delReadStream也要做相应的调整：

```php
/**
     * @param resource|EvIo $stream 为了兼容，所以可以传入原resource，
	 * 	也可以传入回调入参EvIo对象
     * @return void
     */
    public function delReadStream($stream): void
    {
        if(is_resource($stream) and isset($this->_reads[$key = (int)$stream])){
            /** @var EvIo $event */
            $event = $this->_reads[$key];
            $event->stop();
            unset(
                $this->_reads[$key],
                $this->_readFds[spl_object_hash($event)]
            );
        }

        if($stream instanceof EvIo and isset($this->_readFds[spl_object_hash($stream)])){
            $stream->stop();
            $key = (int)($this->_readFds[spl_object_hash($stream)]);
            unset(
                $this->_reads[$key],
                $this->_readFds[spl_object_hash($stream)]
            );
        }
    }
```

WriteStream同理，这里就不多说了。

**另外还有一点不是很重要的区别，无延迟定时器在Ev下会比IO更快，你可以理解为每次循环都是最先执行，优先级最高；而原生loop和event都是在IO之后，有点类似于每次循环的结束的时候触发**

## Swoole/OpenSwoole个人觉得还是不稳当，最好有阅读C源代码的能力

Swoole在测试的时候报了挺多错误的，感觉和PHPunit有一些冲突，最明显的是如下这个错误：

```php
PHPUnit\Framework\Exception: PHP Fatal error:  Uncaught Exception: Serialization of 'Closure' is not allowed
```
即便使用PHPunit的 **@backupStaticAttributes** 和 **@backupGlobals** 依然存在；我没有去看源码，个人理解可能是Swoole底层利用了一些全局的静态导致了这个，感兴趣的可以去看看源码。

OpenSwoole反而比Swoole好像要少一些，但也可能是我是用方式不同导致的，但也有一些比较令我意外的地方；在OpenSwoole的event-loop中，我利用了 **Event::defer** 嵌套 **Timer::tick** 做了一些工作，实现了无延迟单次定时器、无延迟循环定时器等：

```php
Event::defer(
	function() use($id, &$timerId, $repeat, $callback){
    	if($timerId = Timer::tick($repeat, $callback))
			$this->_storage->set($id, $timerId);
        }
        $callback();
    }
);
```

在一个测试信号的用例中：

```php
	/** @runInSeparateProcess 测试信号相应 */
    public function testSignalResponse()
    {
        if (
            !function_exists('posix_kill') or
            !function_exists('posix_getpid')
        ) {
            $this->markTestSkipped('Signal test skipped because functions "posix_kill" and "posix_getpid" are missing.');
        }

        $count1 = $count2 = 0;
        $this->loop->addSignal(12, function () use (&$count1) {
            $count1 ++;
            $this->loop->delSignal(12);
            $this->loop->destroy();
        });

        $this->loop->addSignal(10, function () use (&$count2) {
            $count2 ++;
            $this->loop->delSignal(10);
            $this->loop->destroy();
        });

        $this->loop->addTimer(0.0,0.0,function () {
            posix_kill(posix_getpid(), 10);
        });

        # 猜测是因为发送信号会被转为异步的缘故，所以当去除这个timer时，则不满足预期
        # 但是不可以使用Event::defer，这里的无延迟定时器使用的就是Event::defer
//        $this->loop->addTimer(0.0,0.0, function (){});
        $this->loop->addTimer($this->tickTimeout,0.0, function (){});

        $this->loop->loop();


        $this->assertEquals(0, $count1);
        $this->assertEquals(1, $count2);
    }
```

如果上述代码我注释掉 **$this->loop->addTimer($this->tickTimeout,0.0, function (){});** 这行代码，结果便达不到预期；如果使用 **$this->loop->addTimer(0.0,0.0, function (){});** 也同样达不到预期，这里 **$this->loop->addTimer(0.0,0.0, function (){});** 可以等同看作是 **Event::defer**，也就是说必须有一个基于 **Timer::tick** 的方法挂在最后，这样的结果才符合预期；

我的猜测是在Swoole/OpenSwoole环境下，信号的发送和接收是会被转为异步操作，交给了swoole的辅助线程的缘故，因为swoole中还有一条辅助线程用来调度协程，也会包含一个loop，因为这个信号被转为了异步，也就是说当我的loop循环到第二圈的时候，信号还并没有通知到我这里，我因为使用了defer，在第二个循环中就将当前循环**destroy()**了，自然达不到预期，但当我使用了一个**Timer::tick**将我的loop挂起了不止一圈，那么这时候可能就收到了异步的信号通知，那么自然也就触发了我的回调，自然也就符合了预期。

还有一个问题，就是因为swoole没有无延迟的定时器提供，所以我使用了**Event::defer**来充当这么一个定时器，但这里会存在一个问题，无延迟的定时器可能在生产环境中产生不同的业务逻辑，同时存在好几个不同的无延迟定时器，但这里使用了**Event::defer**只能注册一个回调方法，后注册的会覆盖先注册的，所以需要慎用。

这个swoole/openswoole我目前还没有测试完，之后测试了有其他心得的话，会继续更新，毕竟如果能够使用一些协程，也是比较快乐的事儿，继续折腾！

这里继续提一句，我的 **[workbunny/event-loop](https://github.com/workbunny/event-loop) -1.0.0** 是生产可用的，做好了测试覆盖的；
随后 **1.x** 我会加入 **openswoole**，当然我得先把它测试通过才可以；

**欢迎star！PR！issue！**

> 🐇 更新于2020-05-30


# 上述关于OpenSwoole相关的内容我需要纠正一下，在我调整了测试用例后没有复现，这里经过反复测试得出来了几个结论：

1. **Event::tick 可以多次创建不会覆盖，也就是不影响无延迟触发器和无延迟定时器得多次创建**

2. **之前测试的信号和Timer及Event相关的猜测应该是不存在的，是可以正常调用，出现之前所述的问题的原因是在测试用例时并没有注意Swoole在一个循环周期内的事件优先级，这个需要大家关注**

# 更多详细的测试内容可以参看[workbunny/event-loop](https://github.com/workbunny/event-loop)，已经发布1.1.1版本，增加了OpenSwoole的支持！

> 🐇 更新于2020-06-02