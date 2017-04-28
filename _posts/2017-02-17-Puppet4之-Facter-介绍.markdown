---
layout:     post
title:      "Puppet之Facter介绍"
subtitle:   "Puppet4"
date:       2017-02-17 16:25:11
author:     "Luyuan"
header-img: "img/puppet4/puppet-facter.jpg"
catalog: true
tags:
    - Puppet
---


## 1.前言
Facter是Puppet的跨平台系统剖析库，它可以发现和报告每个节点的系统参数，可以包含硬件信息、软件信息、系统状态、网卡信息等其它预定义的信息，同时它可以在manifests清单和puppet代码、Hiera中作为变量使用。用清新脱俗的话来总结它，它是一个收集各种数据的小玩意儿，收集来数据供奉给上面的各“老大”使用。在Puppet4的最新版本中，使用最新的Facter 3.x版本，puppet4中最重要一个特性就是把往前Puppet, Facter, Hiera, MCollective, pxp-agent, root certificates, Ruby, Augeas等包集中到puppet-agent当中。最新版本中Facter使用C++11重构，剔除了原来在puppet2、puppet3版本当中的ruby语言写的Facter.但是自定义Facter写法没有变化，只要你在ruby file中启动require 'facter’.即可
参考：https://pom.nops.cloud/bestpractice/puppet4.html
## 2.正文
## 2.1 一些重要的自带facter
当然在puppet4中系统安装完成后，会自带很多facter参数，比如：aio_agent_version os processorcount productname fqdn 等等参数，如果需要可以通过命令行去查看
facter -p


## 2.2 fact代码存放的位置
puppet4 fact存放的位置
/opt/puppetlabs/puppet/cache/facts.d
/module/lib/ #每个模块lib库当中
可以通过命令打印fact的具体目录位置 puppet config print factpath

## 2.3 如何编写一个facter
Facter的一个典型事实是简单的几个不同元素的组合， 您需要熟悉Ruby才能理解这些示例。 要了解一个明确的介绍。
```ruby
Facter.add(:rubypath) do
  setcode 'which ruby'
  end
```

```ruby
Facter.add(:rubypath) do
  confine :osfamily => "Windows"
  # Windows uses 'where' instead of 'which'
  setcode 'where ruby'
end
```
上述两个例是查看ruby 安装路径，第一个例子只会在Linux系统上执行，而第二例子只有:osfamily = win 的时候才会执行 where ruby 操作。

```ruby
Facter.add(:jruby_installed) do
  confine :kernel do |value|
    value == "Linux"
  end

  setcode do
    # If jruby is present, return true. Otherwise, return false.
    Facter::Core::Execution.which('jruby') != nil
  end
end
```
上述代码通过confine语句，限定只有基于Linux的内核才能使用该Fact值，然后通过Facter的exe的方法执行which命令。confine 的语句作用是仅当它所指定条件满足时才返回后面代码块执行结果。这个值通常基于其它fact的值。并且要注意的是在自定义fact时，要使用ruby符号，也就是在变量前面加冒号。 confine :kernel => :linux

### 2.5 现实中实现一个Facter
由于目前我们当前openstack环境中使用puppet-module来管理我们的配置文件，近期我们同学发现，Openstack中Python服务占用大量内存及CPU的使用量。后来我们发现在puppet-module中代码逻辑直接调用了facter原生变量作为传送值，比如：CPU核心数为48核，那么在nova-api进程就会启动48个进程工作。这样会大量占用系统内存。
代码如下：
```ruby
# [*osapi_compute_workers*]
#   (optional) Number of workers for OpenStack API service
#   Defaults to $::processorcount
#
class nova::api(
  $osapi_compute_workers     = $::processorcount,
  $metadata_workers          = $::processorcount,
  ){
  nova_config {
    'DEFAULT/osapi_compute_workers':     value => $osapi_compute_workers;
    'DEFAULT/metadata_workers':          value => $metadata_workers;
 }
}
```
由于每台服务器配置都不是一样，我们最终决定自定义facter来解决此问题
```ruby
require 'facter'

#This is optimize to system for usage
Facter.add(:anyworker) do
  setcode do
    compare_worker = 8
    real_worker = Facter::Core::Execution.exec("grep -c processor /proc/cpuinfo")
    if real_worker.to_i > 0
      optimize_worker = real_worker.to_i / 2
    end
  result_worker = [optimize_worker,compare_worker].min
  end
end
```
## 3 测试Fact
测试fact方法有很多种，本章是通过aio环境进行测试
###3-1.使用FACTERLIB环境变量
export FACTERLIB="/root/"
facter anyworker

###3-2.使用--custom-dir命令行选项
facter --custom-dir=/root/ anyworker

###3-3.使用debug 进行调试
 facter --custom-dir=/root/ anyworker --debug

# 3 总结
通过facter我们可以获取Puppet资源无法直接使用的变量，通过这些变量我们可以做很多事情，比如角色分类、数值大小比较、系统类似判断，但是facter会消耗部分的系统资源。

## 后续

—— Luyuan 后记于 2017.02.17
