---
title: "基于 Triple 实现 Web 移动端后端全面打通"
linkTitle: "基于 Triple 实现 Web 移动端后端全面打通"
tags: ["apachecon2023", "triple", "protocol"]
date: 2023-10-07
authors: ["陈有为"]
description: 基于 Triple 实现 Web 移动端后端全面打通
---

摘要：本文整理自陌陌研发工程师、Apache Dubbo PMC陈有为在 Community Over Code 2023 大会上的分享，本篇内容主要分为四个部分：

- 一、RPC 协议开发微服务
- 二、全新升级的 Triple 协议
- 三、Triple 协议开发微服务
- 四、Dubbo 为 Triple 协议带来治理能力

## 一、RPC 协议开发微服务

![dubbo-triple-协议](/imgs/blog/2023/8/apachecon-scripts/triple/img.png)

在我们正常开发微服务的时候，传统RPC服务可能在最底层。上层可能是浏览器、移动端、外界的服务器、自己的测试、curl等等。我们可能会通过Tomcat这种外部服务器去组装我们的RPC层，也就是BFF。或者我们没有BFF，我们的RPC就是对外提供服务。但因为浏览器要访问，所以我们需要有一个网关，比如说API sinks或者神域等HTTP网关。

![dubbo-triple-协议](/imgs/blog/2023/8/apachecon-scripts/triple/img_1.png)

上图展示的是我们的流程，但是存在一些问题。

如果我们的服务是非常轻的，我们只需要一个转发层，我们是不是很麻烦。无论是配网关还是起一个webserver去转发，肯定都很麻烦。

此外，RPC服务大部分都是基于二进制的，而二进制正常在本地是没法测试的。因此我们的公司内都可能就会开发一种后台或者中间的Process让我们去测试。但这个的前提是你至少得把它部署到测试环境，所以还是没法在本地测试。

总体来说，这两个问题的易用性比较低，且开发成本相对较高，因为要做一些重复劳动。

## 二、全新升级的 Triple 协议

![dubbo-triple-协议](/imgs/blog/2023/8/apachecon-scripts/triple/img_2.png)

基于上边的两个问题，我们来介绍一下Triple协议。

先来说一下上一代协议，它产出的原因是什么。我们应该都知道Dubbo原来是Dubbo协议，它是基于tcp的，它有一个包。因为它的包的设计，导致了网关无法做一些特殊规则判断、过滤等操作。但也不是绝对的，如果你愿意牺牲性能把包完全解出来，组装回去再透传还是可以做到的，但一般大家都不太能接受。

所以我们就在想能不能把原数据和真正的包分开。现在我们有现成的HTTP，又有一个业界主流的gRPC，所以我们的目标就是兼容gRPC。因为gRPC目前都是用IDL，而IDL有一个问题，尤其在Java侧。因为大家都是写一些接口，定义一些包去实现，这样就会非常麻烦。Go侧就还好，因为大家已经习惯了这种开发模式。

所以我们开发了Triple协议，首先它兼容了gRPC，所以我们能实现和gRPC的完全互通。其次，我们兼容了自己定义接口的方法。虽然会损失一定的性能，但提升了一些易用性。而且RPC一般不是业务的瓶颈，大多数瓶颈还是在DB。

但还有个问题，虽然我们兼容了gRPC，但gRPC是基于TPC的，所以如果前端或者其他第三方系统只有HTTP，它还是接受不了我们的系统。

![dubbo-triple-协议](/imgs/blog/2023/8/apachecon-scripts/triple/img_3.png)

基于此，我们想推出一个全新的Triple协议。为了解决上述的所有问题，我们参考了gRPC、gRPC Web、通用HTTP等多种协议，做到浏览器访问，支持Streaming，还支持同时运行在 HTTP/1、HTTP/2 协议上。因为目前HTTP/3还没有大规模推广，未来也会支持HTTP/3。

最终的设计实现是完全基于HTTP的，且对人类、开发调试友好。我们可以通过简单的浏览器访问或者curl访问，尤其是对unary RPC。此外，我们和gRPC是完全互通的，用HTTP的业务不用担心兼容性的问题，也不用担心签协议的问题。为了稳定性，我们只会采用业界流行的网络库，比如Java的native、Go的基础的net包。

![dubbo-triple-协议](/imgs/blog/2023/8/apachecon-scripts/triple/img_4.png)

虽然Triple协议和gRPC协议都基于HTTP，但gRPC是基于HTTP/2的，而Triple是基于HTTP/1和HTTP/2的。

我们在进入gRPC的同时，我们为了易用性扩展了一些功能。比如请求里我们支持application Json，curl访问，此外上一版的协议，为了支持传统定义接口的方式，我们有一个二次序列化的过程。我们想在这里通过一个特殊的tag来决定我们的body的结构，解决二次序列化的问题。同时这个东西是可以扩展的，理论上HTTP的所有future我们在Triple协议上都可以实现，也可以拓展。

![dubbo-triple-协议](/imgs/blog/2023/8/apachecon-scripts/triple/img_5.png)

用了Triple协议之后，我们的开发流程也发生了改变。如果你不需要进行组装，或者没有外层的代理，可能你的接入流程就是从外部的请求浏览器、对方的服务器、curl、自己测试等直接到了server。

和其他的gRPC的通信也是没有问题的，流程就相当于少了一层。对于大多数用户，如果你不需要这个场景，其实是有很大的好处。

![dubbo-triple-协议](/imgs/blog/2023/8/apachecon-scripts/triple/img_6.png)

Triple协议因为最开始兼容gRPC，那个时候只基于HTTP/2，HTTP/2有Streaming的能力，所以它天然支持Streaming。但这里比较特殊的是，我们新版的协议在HTTP/1也支持了Stream，但仅支持了Server Stream。也就是客户端发一个，服务端发好几个回去，这个HTTP/1的Server Push实现的。

![dubbo-triple-协议](/imgs/blog/2023/8/apachecon-scripts/triple/img_7.png)

Client Stream和Bi Stream就没什么可说的了。但有一个特别的是，在Java侧没有Bi Stream，从编码上就没有，但从实现上是有的。

## 三、Triple 协议开发微服务

![dubbo-triple-协议](/imgs/blog/2023/8/apachecon-scripts/triple/img_8.png)

目前Triple协议比较灵活的支持两种定义方式，分别是IDL定义和直接定义。直接定义支持同步、异步、手写。还有比较极端一点的，比如在自己定义接口的时候用IDL生成probuff的类，我们不定义它的service，只用它的接口也是没问题的，它会自动识别接口使用pb还是不使用pb。

![dubbo-triple-协议](/imgs/blog/2023/8/apachecon-scripts/triple/img_9.png)

Server就是把它的务实现一下。上图是一个例子，我就直接拿了API的组装方式，真正的业务上可能是注解或者XML的方式。

![dubbo-triple-协议](/imgs/blog/2023/8/apachecon-scripts/triple/img_10.png)

因为我们支持了HTTP这个标准的协议，理论上我们的测试就会变得很简单。

因为我们支持gRPC，所以我们可以用gRPC curl去调用我们的服务。但前提是你得有反射服务，然后手动开启一下，它不是默认开启的。然后它就可以通过反射拿到接口的源数据，通过Json转成pb格式发过去。或者我们直接用Application Json的方式直接调过去。这里有一点比较特别的是在HTTP/1下我们也可以用Sream。

另外，因为我们支持HTTP，理论上所有第三方的HTTP客户端都是可以调用的。然后我们的admin也可以进行测试，前提是你得把它注册上。

![dubbo-triple-协议](/imgs/blog/2023/8/apachecon-scripts/triple/img_11.png)

调用端不管是POJO还是IDL，它们都没有本质的区别。

![dubbo-triple-协议](/imgs/blog/2023/8/apachecon-scripts/triple/img_12.png)

现在我们有了Triple协议，但如果这个协议没有承载方也是行不通的。因此我们还得有一个框架，有一些服务治理才是我们的微服务。所以服务治理也是微服务中不可或缺的一部分。

## 四、Dubbo 为 Triple 协议带来治理能力

![dubbo-triple-协议](/imgs/blog/2023/8/apachecon-scripts/triple/img_13.png)

Triple的定位只是Dubbo里的其中一个协议，当然你也可以为了兼容性，用原来的Dubbo协议或者其他的协议。而且我们支持在同一个端口上开多个协议，可以按需选择。

![dubbo-triple-协议](/imgs/blog/2023/8/apachecon-scripts/triple/img_14.png)

同时Dubbo 为 Triple 提供了多语言的实现。目前会在Rust、Go、Java、JS、node、Python这几部分实现官方的实现。这样用户就不用自己根据实验协议的spec去实现了。如果你有一些定制需求，比如内部的一些框架，你根据spec实现也是可以的。

![dubbo-triple-协议](/imgs/blog/2023/8/apachecon-scripts/triple/img_15.png)

Dubbo和服务框架集成的很好，理论上在开发流程中，尤其是在Java侧服务定义、服务治理、服务注册发现等都不用客户来操心，是开箱即用的。

![dubbo-triple-协议](/imgs/blog/2023/8/apachecon-scripts/triple/img_16.png)

Dubbo 提供了丰富的生态，第三方的生态包括Nacos、Zookeeper等等，我们不用创新，直接引入相应的包即可。

![dubbo-triple-协议](/imgs/blog/2023/8/apachecon-scripts/triple/img_17.png)

这是我们使用Triple协议服务注册的例子。上面你可以选Nacos、Zookeeper、K8s，左边是一个Client和一个Server，这么调用。

![dubbo-triple-协议](/imgs/blog/2023/8/apachecon-scripts/triple/img_18.png)

我们在admin上看一下实现。这里提一句，我们的admin也在新版重构，是用Go实现的，大家可以期待一下。

![dubbo-triple-协议](/imgs/blog/2023/8/apachecon-scripts/triple/img_19.png)

我们经常会遇到灰度发布或者流量染色的需求。我们可以从admin上发一个tag下去，把一些实例打上tag，然后这个流量从入口就会挨个下去。
