---
layout:     post

title:      "使用服务网格和 Envoy Gateway 构建客户端的可用性和弹性"
subtitle:   ""
description: '在讨论可用性和弹性时，我们通常是从基础设施和服务的角度来探讨的。那么有没有办法在不修改服务的情况下在客户端提高服务的“实际感知可用性”呢？'
author: "Zack Butcher Tetrate.io (赵化冰 译)"
date: 2024-04-07
image: "https://images.unsplash.com/photo-1711639270142-e1e181749333?q=80&w=3133&auto=format&fit=crop&ixlib=rb-4.0.3&ixid=M3wxMjA3fDB8MHxwaG90by1wYWdlfHx8fGVufDB8fHx8fA%3D%3D"
published: true
tags:
    - Envoy Gateway
    - Envoy
    - Service Mesh
categories:
    - Tech
showtoc: true
---

## 如何从客户端角度思考服务的可用性和弹性

> 这是一系列关于 Envoy Gateway 价值的文章之一，该网关已经达到了 1.0 版本的发布里程碑，可以投入生产使用。

在讨论可用性和弹性时，我们通常是从基础设施和服务的角度来探讨的。我们很少考虑是否可以在客户端采用某种方法来提高后端服务的“实际感知可用性”（即在客户端测量到的服务的可用性）。这主要是因为我们在大部分情况下都无法控制客户端与服务的交互方式。但实际上我们有办法对客户端和服务之间的交互进行控制，从而提高客户端对服务的“实际感知可用性”。

本文将讨论服务网格如何通过下面这六种方式提高客户端对服务的“感知可用性”，从而增强系统的整体弹性：

1. 客户端负载均衡
2. 重试
3. 超时
4. 断路器
5. 异常检测
6. 限流

本文将依次介绍每种功能及其提供的价值。但需要注意的是，虽然每种功能单独提供了一些好处，但只有它们共同作用，才能为真正地为我们的系统提高可用性。

## 每个服务都有一个“网关”

我们可以在后端服务的入口或 API 网关处控制“客户端”的行为，通过使用网关保护我们的内部系统免受外部发生的任何异常情况的影响。在该网关处，我们可以实现诸如负载均衡、重试、超时、限流等众所周知的模式。随着服务网格的引入，我们在基础设施中为每个服务都都部署了一个“网关” —— 服务网格的边车。边车不仅在服务器端起作用，提供（m）TLS 和策略执行点。它还在客户端（调用者）端提供了重要的功能。由于网格提供了集中式控制，服务所有者可以轻松地为调用其服务的客户端设置默认行为。因此，我们不再只是从基础设施故障域、数据库影响范围等服务的角度来讨论系统的弹性，还可以从客户端如何与服务器通信的角度对此进行讨论。

边车为客户端带来的第一个最重要的功能是客户端负载均衡。边车中有客户端可能要与之通信的每个服务实例的信息，在客户端对外通信时，边车直接在这些服务实例之间进行流量负载均衡。该模式中流量直接从客户端到服务器，而无需通过中央负载均衡器（如F5）这样的中间人。同时，由于我们在客户端处理负载均衡，网格还可以提供一系列其他强大的工具。

## 以三个九的价格获得五个九的高可用能力：一个真实的案例

为了提供一个采用这些工具改善应用程序的感知可用性能力的真实证据，可以看一下我在上一个雇主那里工作时的一个服务。该服务主要方法对外宣称的 SLA 是五个九：在保持 P90 时延为 10ms 以内的同时可用性达到 99.9995%。（习惯于使用高可用性系统的人会意识到实现这种稳定运行时间所需的成本。对于不了解的人，我使用的经验法则是：“从1000美元开始，每增加一个9，就增加一个零”。当然，短时间内也许可以更便宜，但如果要长时间（多年）维持该可用性稳定运行，那么这大概就是要考虑的正确成本。）

与其通过构建昂贵的服务器端的能力来提供这种高可用性，我们可以以较低的成本构建一个“厚客户端”来实现相同的高可用性能力，该“厚客户端”采用服务网格的边车来提供这些能力。使用该客户端的重试、超时、异常检测、断路器——以及一些高级模式，如[请求对冲¹](https://grpc.io/docs/guides/request-hedging/)——我们能够交付一个感知可用性满足五个九目标的系统，而后端本身则只需提供了略高于三个九（99.95％）的可用性。由于我们能够隔离我们的故障域，结合智能的客户端行为，我们可以使请求的失败“看起来”是不相关的。

## 服务网格中的客户端负载均衡：超越组件之和

**客户端负载均衡**意味着客户端知道它们可以与之通信的所有可能的后端，并且可以在和某服务通信时选择该服务的任意一个实例。我们可以使用各种不同的算法来选择负载均衡的服务器端点。在这个基本功能上，我们可以建立我们列表中的其他能力。

### 重试

**重试**有助于减轻瞬时故障的影响。在后端存在不稳定性、不可靠网络、服务器过载和故障等情况下，重试使我们有能力尝试使用不同的后端来处理同一请求，在各个后端的故障是不相关的情况下，重试可能成功。Envoy 会避免将重试请求发送到任何已经失败的后端上——在部署了足够多的后端的情况下，Envoy 会保证重试请求会发送到一个新的后端服务。然而，盲目地重试也会导致问题，下面的两个功能可以帮助解决这个问题。

### 异常检测

**异常检测**是一种被动的服务健康检查，异常检测观察每个服务实例如何响应来自客户端的请求，并标记那些与其他实例相比表现不佳的实例（例如，连续返回错误或反复超时的实例）。如果一个实例持续表现不佳，则将其从活动负载均衡实例池中移除——换句话说，Envoy 将暂时停止向其发送流量。对于异常的实例， Envoy 采用下面的策略判断其状态是否恢复：Envoy 将逐步尝试向先前表现不佳的实例发送流量，以查看它们是否恢复正常。如果恢复正常，则将被放回到活动负载均衡池中，重新参与正常的流量分发。

当我们将异常检测与重试结合在一起时，对于单个请求，我们可以避免向已知的异常实例发送重试；通过汇总分析所有请求，客户端可以了解每个实例的健康状况，并更倾向于向行为良好的实例发送流量。

### 断路器

断路器有助于限制每个客户端到每个后端的最大并发数量——由于存在正在进行的重试，使用断路器来避免由于大量重试而导致的级联故障至关重要！断路器限制并发，包括连接的生存时间、客户端和每个后端服务器之间的最大 TCP 连接数、每个后端服务器上允许的最大 HTTP 请求数量等等。当一个服务实例触发断路器时，它会触发该端点的异常检测，将其移出活动负载均衡池。

因此，当我们将重试、异常检测和断路器三者结合在一起时，我们得到了一个强大的客户端，可以继续将流量转发到正常工作的后端，并避免异常的后端，同时不会因为超载系统而导致其他故障。

### 超时

大量的重试也会对系统产生更重的资源负担——即使服务端失败了，接受和处理请求也会消耗服务端的资源！因此，限制系统的行为非常重要：不仅可以减少用户可能遇到的最坏延迟，还可以避免太多请求导致的资源消耗（当请求超时时，Envoy 可以选择执行关闭到服务器的连接等操作，以停止请求“浪费”的工作）。超时通过限制客户端等待服务器响应的时间（总体和每次重试）来限制消耗的系统资源。一旦超时时间到达，Envoy 将关闭连接并向调用方返回超时错误，释放客户端和服务器端的资源。

### 限流

超时有助于限制我们在任一请求上花费的资源，限流则有助于限制系统中同时存在的请求总数。我们可以将超时和重试看作控制的深度（为任一请求等待/支付的成本），而将限流看作控制的宽度（某一时刻的服务总量）。限流制旨在保护服务器不会被大量并发的客户端压垮，而断路器则有助于保护服务器不会被特定的单一客户端压垮。

可以将限流视为保护服务器的共享资源免受过载的影响，而断路器则保护服务器的每个单个实例免受过载的影响。

Envoy 支持进行本地限流，每个 Envoy 实例跟踪其看到的请求并应用速率限制。这在将 Envoy 作为入口网关使用时是非常有用的。Envoy 也支持“全局”限流，其限流的额度由所有 Envoy 实例共享。全局限流需要部署一个用于存储速率限制的 Redis 实例，以及一个位于其前面的限流服务器。[Tetrate Enterprise Gateway for Envoy (TEG)²](https://tetrate.io/tetrate-enterprise-gateway-for-envoy/)—Tetrate 针对 [Envoy Gateway³](https://gateway.envoyproxy.io/) 的企业版本中提供了全局限流所需的所有组件，并以 helm chart 的方式进行发布。

当服务是无状态服务时，通常情况下，不需要引入速率限制及其所需要的额外机制就能达到很好的效果（无状态服务具有较好的伸缩性，如果已有实例被过多的请求所淹没，可以通过在短时间内简单地增配更多实例来解决）。

## 低成本下的高感知可用性

通过本文的介绍，希望您能清楚地理解为什么我们需要一起讨论所有这些功能，而不只是是单独讨论其中任何一个。完整考虑到您系统中的不同故障模式和资源约束，并构建一套全面的客户端策略——结果是以较低的成本获得显著提高的客户端感知可用性。

## 下一步

[Envoy Gateway (EG)³](https://gateway.envoyproxy.io/) 是由 Envoy 社区驱动的一个项目，旨在简化 Envoy 的使用和操作，使其成为网关的首选。它专注于易用性，让常见用例变得简单，并利用 [Kubernetes Gateway API⁴](https://gateway-api.sigs.k8s.io/) 来管理 Envoy 和对外暴露应用。Tetrate 协助启动了 EG 项目并持续对其进行大量投入。

Tetrate 提供了一款企业级的 Envoy 网关分发产品——Tetrate Enterprise Gateway for Envoy ——您可以立即开始使用。欢迎[查看文档以了解更多信息并试用⁵](https://docs.tetrate.io/envoy-gateway/)。

## 参考链接

1. https://grpc.io/docs/guides/request-hedging
2. https://tetrate.io/tetrate-enterprise-gateway-for-envoy
3. https://gateway.envoyproxy.io
4. https://gateway-api.sigs.k8s.io
5. https://docs.tetrate.io/envoy-gateway
6. https://tetrate.io/blog/client-side-availability-and-resiliency-with-envoy-gateway