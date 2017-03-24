title: Docker compose使用默认的bridge网络
date: 2017-03-24 17:54:52
tags:
- docker
---

首先compose默认会为当前的compose建立单独的网络，然后所有的服务连接到这个网络。如果需要定义默认连接到自定义的网络，需要这样定义

<!-- more -->

```
networks:
  default:
    # Use a custom driver
    driver: custom-driver-1
```

如果是自定义的外部网络

```
networks:
  default:
    external:
      name: my-pre-existing-network
```

如果外部网络使用的是docker默认的`bridge`网络，会报如下错误

> Network-scoped alias is supported only for containers in user defined networks

原因是compose依赖网络范围的别名，如果使用外部的bridge网络，他仍然会尝试设置别名，但是网络别名只能设置在用户定义的网络上，默认的网络是不能设置别名的，所以报错。可以设置network_mode，来关闭compose内置的alias功能，让他使用网络默认的别名。

但是默认的`bridge`中无法设置别名，也就不存在默认的别名，service之间的网络通信又是依赖于别名的。

所以结论上讲，compose中的service是无法使用默认的`bridge`网络进行通信的，必须使用用户自定义的网络，外部定义或者compose中定义均可。

[1]: https://github.com/docker/compose/issues/3012