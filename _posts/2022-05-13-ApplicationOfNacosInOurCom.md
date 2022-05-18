---
layout: post
title: "Nacos在我司的应用"
date: 2022-05-13 13:00:00
image: ''
description: 'Nacos在我司的应用及Webman生态的Nacos-client开发'
tags:
- PHP
- 组件
categories:
- Chaz6chez

---


## 前言

我目前所在的部门主要是负责公司的数据相关的内容，可以理解为数据统计，做的工作其实也比较复杂，除了做一些数据统计分析业务之外，需要做一些基础服务的开发；我部门因为内部开发语言并不统一，在这种情况下，项目被动的分成了A\B\C\D等子项目，并没有将项目合并到一个项目中开发，在这种过程中，被动的接受了SOA这样的结构。

A项目是一个任务的调度分配服务，可以理解为一个大型的脚本/定时执行器，有点类似与现在比较流行的serverless函数服务，向A项目中添加一个任务函数或者执行脚本，他就会在合适的时候被触发；由于硬件服务器并不止有一台，数据库也并不只有一台，结合现在容器化思路，这样的配置需要很多，如果仅仅是写在配置文件中，并不能方便运维的统一快速方便的管理，所以我们计划做一个配置中心；因为我们的A\B\C\D等子项目也并不只有一个实例，他们各自是可以横向拓展的主体；在这样的前提下，我们决定引入Nacos/consul等包含了配置管理的服务注册/发现服务（Nacos/consul都是优秀的服务注册/发现服务，选用Nacos是一些额外因素，他们各自有优缺点）。

## PHP

PHP在SOA中扮演了Web业务服务的一个角色，主要是进行一些业务接口的输出，但是我们的服务由于需要高承载量，原本计划是使用自研的Reactor模型的NIO框架，但是考虑到减少心智负担，所以选用了文档及社群更完善Webman作为开发框架。

## 接入Nacos

最初我是使用了 [Tinywan/nacos](https://www.workerman.net/plugin/25) 的插件进行的业务开发，但是我们在使用过程中发现，他的配置监听项是通过Timer创建一个nacos->config->get请求实现的，在Timer间隔期内可能变更了配置，也就是极限状况下存在{Timer interval}的同步延迟，这样并不符合我司的具体情况，我们要求的服务变更可能需要更迅速，因为在一些业务点我们不能有过多时长的错误及业务不通畅，但如果仅仅是将{timer interval}的值缩小至ms，那么又会存在对Nacos服务的过多请求；另外由于我们的业务已经写了有一段时间了，累积了大量的config()调用方式，这时候我们需要考虑怎么样非侵入的改变这一习惯或者着一些代码，于是，我基于 [Tinywan/nacos](https://www.workerman.net/plugin/25) 的思路封装了适合我们的Nacos客户端插件 [Workbunny/webman-nacos](https://www.workerman.net/plugin/50);

## Workbunny/webman-nacos

#### 1. 配置监听

配置监听部分我们需要完成以下三个要求

- 时效性
- 触达深度
- 0侵入

我们在配置中使用yaml文件作为了环境配置替代了原有的.env文件，并且将yaml文件保存在nacos对应的namespace；相当于业务使用config函数的时候，config函数会找到config目录下对应的php文件，PHP文件中又使用yaml函数去调用对应的yaml文件引入对应的值，调用链可以理解为如下：

~~~
config() -> /config/X.php -> yaml() -> /x.yaml
~~~

**这个过程完全可以简化成config()直接找到config目录的对应php文件，将多个php文件保存至nacos对应的namespace下。**

基于上述的过程，我最早使用了Timer + Guzzle异步请求 + nacos长轮询监听 保证 **时效性**，因为存在多个yaml文件，所以需要对多个yaml文件进行监听，如果单纯一个配置开一个进程有点太奢侈，所以我使用了一个进程 + Guzzle异步请求；Nacos监听的长轮询机制你可以理解为如果有消息，就马上返回对应的配置id，如果没消息，就一直阻塞到timeout并且返回一个空字符串；考虑到请求会阻塞，为了不影响该进程内Timer的下一个执行周期，我将Timer的间隔时长和长轮询阻塞时长画上了等号。

~~~
	public function onWorkerStart(Worker $worker)
    {
        $worker->count = 1;

        if($this->configListeners){
            // 拉取配置项文件
            foreach ($this->configListeners as $listener){
                list($dataId, $group, $tenant, $configPath) = $listener;
                if(!file_exists($configPath)){
                    $this->_get($dataId, $group, $tenant, $configPath);
                }
            }
            // 创建定时监听
            Timer::add($this->longPullingInterval, function (){
                $promises = [];
                foreach ($this->configListeners as $listener){
                    list($dataId, $group, $tenant, $configPath) = $listener;
					# 初始化文件
                    if(file_exists($configPath)){
                        $promises[] = $this->client->config->listenerAsync(
                            $dataId,
                            $group,
                            md5(file_get_contents($configPath)),
                            $tenant,
                            $this->longPullingInterval * 1000
                        )->then(function (ResponseInterface $response) use($dataId, $group, $tenant, $configPath){
                            if($response->getStatusCode() === 200){
                                if($response->getBody()->getContents() !== ''){
									# 文件通过nacos get并覆盖写入本地文件
                                    $this->_get($dataId, $group, $tenant, $configPath);
                                }
                            }
                        },function (GuzzleException $exception){
                            Log::channel('error')->error($exception->getMessage(), $exception->getTrace());
                        });
                    }
                }
                if($promises){
                    Utils::settle($promises)->wait();
                }
            });
        }
    }
~~~


第一版完成后我发现了一些问题：
1. 比如config()中已经获取的配置无法刷新，常驻内存的一些数据库连接等没有被触达
2. Timer + Guzzle异步请求实际上在Timer的执行周期内是阻塞的，只是Guzzle在对多个请求可以并发的发起
3. Timer的第一次执行并不能立即执行，导致初次启动时并不能及时获取最新的配置文件

为了解决第一个问题，我在_get方法内加入了对workers的reload
~~~
	protected function _get(string $dataId, string $group, string $tenant, string $path)
    {
        $res = $this->client->config->get($dataId, $group, $tenant);
        if(file_put_contents($path, $res, LOCK_EX)){
            reload($path);
        }
    }
~~~

~~~
	function reload(string $file)
	{
    	Worker::log($file . ' update and reload. ');
    	if(extension_loaded('posix') and extension_loaded('pcntl')){
       		posix_kill(posix_getppid(), SIGUSR1);
    	}else{
        	Worker::reloadAllWorkers();
    	}
	}
~~~

第二个问题我使用了Workerman/http-client的异步http客户端，在使用的过程中还有个 [小插曲](https://www.workerman.net/q/8452) ，由于http-client使用了workerman的event-loop，我的项目是在workerman的on回调生命周期内，所以可以利用event-loop达到无阻塞的请求；

~~~
	public function onWorkerStart(Worker $worker)
    {
        $worker->count = 1;

        if($this->configListeners){
            // 拉取配置项文件
            foreach ($this->configListeners as $listener){
                list($dataId, $group, $tenant, $configPath) = $listener;
                if(!file_exists($configPath)){
                    $this->_get($dataId, $group, $tenant, $configPath);
                }
                $this->timers[$dataId] = Timer::add($this->longPullingInterval,
                    function () use($dataId, $group, $tenant, $configPath){
                        $this->client->config->listenerAsyncUseEventLoop([
                                'dataId' => $dataId,
                                'group' => $group,
                                'contentMD5' => md5(file_get_contents($configPath)),
                                'tenant' => $tenant
                        ], function (Response $response) use($dataId, $group, $tenant, $configPath){
                            if($response->getStatusCode() === 200){
                                if((string)$response->getBody() !== ''){
                                    $this->_get($dataId, $group, $tenant, $configPath);
                                }
                            }
                        }, function (\Exception $exception){
                            Log::channel('error')->error($exception->getMessage(), $exception->getTrace());
                        });
                });
            }
        }
    }
~~~

第三个问题，我基于workerman/timer封装了一个简易的能达到我目的的timer：

~~~
<?php
declare(strict_types=1);

namespace Workbunny\WebmanNacos;

use Workerman\Timer as WorkermanTimer;

/**
 * 定时器
 *
 * @desc 对workerman/timer的封装
 * 1.延迟单此执行
 * 2.立即单次执行
 * 3.延迟循环执行
 *      - 延迟与循环时间不同
 *      - 延迟与循环间隔相同
 * 4.立即循环执行
 * @author chaz6chez
 */
final class Timer {

    /** @var array[] 子定时器 */
    protected static array $_timers = [];

    /**
     * 新增定时器
     * @param float $delay
     * @param float $repeat
     * @param callable $callback
     * @param ...$args
     * @return int|bool
     */
    public static function add(float $delay, float $repeat, callable $callback, ... $args)
    {
        switch (true){
            # 立即循环
            case ($delay === 0.0 and $repeat !== 0.0):
                $callback(...$args);
                return WorkermanTimer::add($repeat, $callback, $args);

            # 延迟执行一次
            case ($delay !== 0.0 and $repeat === 0.0):
                return WorkermanTimer::add($delay, $callback, $args, false);

            # 延迟循环执行，延迟与重复相同
            case ($delay !== 0.0 and $repeat !== 0.0 and $repeat === $delay):
                return WorkermanTimer::add($delay, $callback, $args);

            # 延迟循环执行，延迟与重复不同
            case ($delay !== 0.0 and $repeat !== 0.0 and $repeat !== $delay):
                return $id = WorkermanTimer::add($delay, function(...$args) use(&$id, $repeat, $callback){
                    $callback(...$args);
                    self::$_timers[$id] = WorkermanTimer::add($repeat, $callback, $args);
                }, $args, false);

            # 立即执行
            default:
                $callback(...$args);
                return 0;
        }
    }

    /**
     * 移除定时器
     * @param int $id
     * @return void
     */
    public static function del(int $id): void
    {
        if(
            $id !== 0 and
            isset(self::$_timers[$id]) and
            is_int($timerId = self::$_timers[$id])
        ){
            unset(self::$_timers[$id]);
            WorkermanTimer::del($timerId);
        }
    }

    /**
     * @return void
     */
    public static function delAll(): void
    {
        self::$_timers = [];
        WorkermanTimer::delAll();
    }
}
~~~


## 之后想到什么再补充吧