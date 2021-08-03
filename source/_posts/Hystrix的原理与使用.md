---
title:  Hystrix 的原理与使用
date: 2021-06-04 18:00:16
tags:
- 微服务
categories: 
- Hystrix
cover: /photo/Hystrix.jpeg

---

## Hystrix 的原理与使用

## 前言

分布式系统环境中，服务间的依赖非常常见，一个业务调用通常依赖多个基础服务。 当进行同步调用时，当库存服务不可用时，订单服务请求线程被阻塞，当有大批量请求调用库存服务时，最终可能导致整个订单服务资源耗尽，无法继续对外提供服务。并且这种不可用可能沿请求调用链向上传递，这种现象被称为**雪崩效应**。

## 服务雪崩效应的定义

服务雪崩效应是一种因 **服务提供者** 的不可用导致 **服务调用者** 的不可用,并将不可用 **逐渐放大** 的过程.如果所示:

![在这里插入图片描述](https://img-blog.csdnimg.cn/20210617100222585.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80NDY5OTM0OA==,size_16,color_FFFFFF,t_70#pic_center)


上图中, A为服务提供者, B为A的服务调用者, C和D是B的服务调用者. 当A的不可用,引起B的不可用,并将不可用逐渐放大C和D时, 服务雪崩就形成了.

## 服务雪崩效应形成的原因

我把服务雪崩的参与者简化为 **服务提供者** 和 **服务调用者**, 并将服务雪崩产生的过程分为以下三个阶段来分析形成的原因:

 - 服务提供者不可用
 - 重试加大流量
 - 服务调用者不可用

![在这里插入图片描述](https://img-blog.csdnimg.cn/20210617100238304.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80NDY5OTM0OA==,size_16,color_FFFFFF,t_70#pic_center)


服务雪崩的每个阶段都可能由不同的原因造成, 比如造成 **服务不可用** 的原因有:

- 硬件故障：如服务器宕机，机房断电，光纤被挖断等。
- 程序Bug：如程序逻辑导致内存泄漏，JVM长时间FullGC等。
- 缓存击穿
- 用户大量请求：如异常流量，重试加大流量等。

硬件故障可能为硬件损坏造成的服务器主机宕机, 网络硬件故障造成的服务提供者的不可访问.
缓存击穿一般发生在缓存应用重启, 所有缓存被清空时,以及短时间内大量缓存失效时. 大量的缓存不命中, 使请求直击后端,造成服务提供者超负荷运行,引起服务不可用.
在秒杀和大促开始前,如果准备不充分,用户发起大量请求也会造成服务提供者的不可用.

而形成 **重试加大流量** 的原因有:

- 用户重试
- 代码逻辑重试

在服务提供者不可用后, 用户由于忍受不了界面上长时间的等待,而不断刷新页面甚至提交表单.
服务调用端的会存在大量服务异常后的重试逻辑.
这些重试都会进一步加大请求流量.

最后, **服务调用者不可用** 产生的主要原因是:

- 同步等待造成的资源耗尽

当服务调用者使用 **同步调用** 时, 会产生大量的等待线程占用系统资源. 一旦线程资源被耗尽,服务调用者提供的服务也将处于不可用状态, 于是服务雪崩效应产生了.

## 服务雪崩的应对策略

针对造成服务雪崩的不同原因, 可以使用不同的应对策略:

1. 流量控制
2. 改进缓存模式
3. 服务自动扩容
4. 服务调用者降级服务

**流量控制** 的具体措施包括:

- 网关限流
- 用户交互限流
- 关闭重试

因为Nginx的高性能, 目前一线互联网公司大量采用Nginx+Lua的网关进行流量控制, 由此而来的OpenResty也越来越热门.

用户交互限流的具体措施有: 1. 采用加载动画,提高用户的忍耐等待时间. 2. 提交按钮添加强制等待时间机制.

**改进缓存模式** 的措施包括:

- 缓存预加载
- 缓存异步加载

**服务自动扩容** 的措施主要有:

- AWS的auto scaling

**服务调用者降级服务** 的措施包括:

- 资源隔离
- 对依赖服务进行分类
- 不可用服务的调用快速失败

资源隔离通常指不同服务调用采用不同的线程池；

我们根据具体业务,将依赖服务分为: 强依赖和弱依赖. 强依赖服务不可用会导致当前业务中止,而弱依赖服务的不可用不会导致当前业务的中止.

不可用服务的调用快速失败一般通过 **超时机制**结合**熔断器** 和熔断后的 **降级方法** 来实现.

## 使用Hystrix预防服务雪崩

**Hystrix** [hɪst'rɪks]的中文含义是豪猪, 因其背上长满了刺,而拥有自我保护能力. Netflix的 **Hystrix** 是一个帮助解决分布式系统交互时超时处理和容错的类库, 它同样拥有保护系统的能力.

Hystrix设计目标：

- 对来自依赖的延迟和故障进行防护和控制——这些依赖通常都是通过网络访问的
- 阻止故障的连锁反应
- 快速失败并迅速恢复
- 回退并优雅降级
- 提供近实时的监控与告警

Hystrix的设计原则包括:

- 资源隔离
- 熔断器
- 命令模式
- 降级

#### 资源隔离

货船为了进行防止漏水和火灾的扩散,会将货仓分隔为多个, 如下图所示:

![在这里插入图片描述](https://img-blog.csdnimg.cn/20210617100303506.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80NDY5OTM0OA==,size_16,color_FFFFFF,t_70#pic_center)


这种资源隔离减少风险的方式被称为:Bulkheads(舱壁隔离模式).
Hystrix将同样的模式运用到了服务调用者上.

在一个高度服务化的系统中,我们实现的一个业务逻辑通常会依赖多个服务,比如:
商品详情展示服务会依赖商品服务, 价格服务, 商品评论服务. 如图所示:

![在这里插入图片描述](https://img-blog.csdnimg.cn/20210617100319684.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80NDY5OTM0OA==,size_16,color_FFFFFF,t_70#pic_center)


调用三个依赖服务会共享商品详情服务的线程池. 如果其中的商品评论服务不可用, 就会出现线程池里所有线程都因等待响应而被阻塞, 从而造成服务雪崩. 如图所示:

![在这里插入图片描述](https://img-blog.csdnimg.cn/20210617100332698.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80NDY5OTM0OA==,size_16,color_FFFFFF,t_70#pic_center)


Hystrix通过将每个依赖服务分配独立的线程池进行资源隔离, 从而避免服务雪崩.
如下图所示, 当商品评论服务不可用时, 即使商品服务独立分配的20个线程全部处于同步等待状态,也不会影响其他依赖服务的调用.

![在这里插入图片描述](https://img-blog.csdnimg.cn/20210617100346285.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80NDY5OTM0OA==,size_16,color_FFFFFF,t_70#pic_center)


资源隔离主要指对线程的隔离。Hystrix提供了两种线程隔离方式：线程池和信号量。

### **线程隔离-线程池**

Hystrix通过命令模式对发送请求的对象和执行请求的对象进行解耦，将不同类型的业务请求封装为对应的命令请求。如商品详情服务查询商品，查询商品请求->商品Command；商品服务查询价格，查询价格请求->价格Command。并且为每个类型的Command配置一个线程池，当第一次创建Command时，根据配置创建一个线程池，并放入ConcurrentHashMap；后续查询商品的请求创建Command时，将会重用已创建的线程池。通过将发送请求线程与执行请求的线程分离，可有效防止发生级联故障。当线程池或请求队列饱和时，Hystrix将拒绝服务，使得请求线程可以快速失败，从而避免依赖问题扩散。

**线程池隔离优缺点**

优点：

- 保护应用程序以免受来自依赖故障的影响，指定依赖线程池饱和不会影响应用程序的其余部分。
- 当引入新客户端lib时，即使发生问题，也是在本lib中，并不会影响到其他内容。
- 当依赖从故障恢复正常时，应用程序会立即恢复正常的性能。
- 当应用程序一些配置参数错误时，线程池的运行状况会很快检测到这一点（通过增加错误，延迟，超时，拒绝等），同时可以通过动态属性进行实时纠正错误的参数配置。
- 如果服务的性能有变化，需要实时调整，比如增加或者减少超时时间，更改重试次数，可以通过线程池指标动态属性修改，而且不会影响到其他调用请求。
- 除了隔离优势外，hystrix拥有专门的线程池可提供内置的并发功能，使得可以在同步调用之上构建异步门面（外观模式），为异步编程提供了支持（Hystrix引入了Rxjava异步框架）。

注意：尽管线程池提供了线程隔离，我们的客户端底层代码也必须要有超时设置或响应线程中断，不能无限制的阻塞以致线程池一直饱和。

缺点：

线程池的主要缺点是增加了计算开销。每个命令的执行都在单独的线程完成，增加了排队、调度和上下文切换的开销。因此，要使用Hystrix，就必须接受它带来的开销，以换取它所提供的好处。

通常情况下，线程池引入的开销足够小，不会有重大的成本或性能影响。但对于一些访问延迟极低的服务，如只依赖内存缓存，线程池引入的开销就比较明显了，这时候使用线程池隔离技术就不适合了，我们需要考虑更轻量级的方式，如信号量隔离。

### **线程隔离-信号量**

上面提到了线程池隔离的缺点，当依赖延迟极低的服务时，线程池隔离技术引入的开销超过了它所带来的好处。这时候可以使用信号量隔离技术来代替，通过设置信号量来限制对任何给定依赖的并发调用量。

使用线程池时，发送请求的线程和执行依赖服务的线程不是同一个，而使用信号量时，发送请求的线程和执行依赖服务的线程是同一个，都是发起请求的线程。

由于Hystrix默认使用线程池做线程隔离，使用信号量隔离需要显示地将属性execution.isolation.strategy设置为ExecutionIsolationStrategy.SEMAPHORE，同时配置信号量个数，默认为10。客户端需向依赖服务发起请求时，首先要获取一个信号量才能真正发起调用，由于信号量的数量有限，当并发请求量超过信号量个数时，后续的请求都会直接拒绝，进入fallback流程。

信号量隔离主要是通过控制并发请求量，防止请求线程大面积阻塞，从而达到限流和防止雪崩的目的。

### **线程隔离总结**

线程池和信号量都可以做线程隔离，但各有各的优缺点和支持的场景，对比如下：

|        | 线程切换 | 支持异步 | 支持超时 | 支持熔断 | 限流 | 开销 |
| ------ | -------- | -------- | -------- | -------- | ---- | ---- |
| 信号量 | 否       | 否       | 否       | 是       | 是   | 小   |
| 线程池 | 是       | 是       | 是       | 是       | 是   | 大   |

线程池和信号量都支持熔断和限流。相比线程池，信号量不需要线程切换，因此避免了不必要的开销。但是信号量不支持异步，也不支持超时，也就是说当所请求的服务不可用时，信号量会控制超过限制的请求立即返回，但是已经持有信号量的线程只能等待服务响应或从超时中返回，即可能出现长时间等待。线程池模式下，当超过指定时间未响应的服务，Hystrix会通过响应中断的方式通知线程立即结束并返回。

#### 熔断器模式

现实生活中，可能大家都有注意到家庭电路中通常会安装一个保险盒，当负载过载时，保险盒中的保险丝会自动熔断，以保护电路及家里的各种电器，这就是熔断器的一个常见例子。Hystrix中的熔断器(Circuit Breaker)也是起类似作用，Hystrix在运行过程中会向每个commandKey对应的熔断器报告成功、失败、超时和拒绝的状态，熔断器维护并统计这些数据，并根据这些统计信息来决策熔断开关是否打开。如果打开，熔断后续请求，快速返回。隔一段时间（默认是5s）之后熔断器尝试半开，放入一部分流量请求进来，相当于对依赖服务进行一次健康检查，如果请求成功，熔断器关闭。

熔断器模式定义了熔断器开关相互转换的逻辑:

![在这里插入图片描述](https://img-blog.csdnimg.cn/20210617100414169.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80NDY5OTM0OA==,size_16,color_FFFFFF,t_70#pic_center)


服务的健康状况 = 请求失败数 / 请求总数.
熔断器开关由关闭到打开的状态转换是通过当前服务健康状况和设定阈值比较决定的.

1. 当熔断器开关关闭时, 请求被允许通过熔断器. 如果当前健康状况高于设定阈值, 开关继续保持关闭. 如果当前健康状况低于设定阈值, 开关则切换为打开状态.
2. 当熔断器开关打开时, 请求被禁止通过.
3. 当熔断器开关处于打开状态, 经过一段时间后, 熔断器会自动进入半开状态, 这时熔断器只允许一个请求通过. 当该请求调用成功时, 熔断器恢复到关闭状态. 若该请求失败, 熔断器继续保持打开状态, 接下来的请求被禁止通过.

熔断器的开关能保证服务调用者在调用异常服务时, 快速返回结果, 避免大量的同步等待. 并且熔断器能在一段时间后继续侦测请求执行结果, 提供恢复服务调用的可能.

#### 命令模式

Hystrix使用命令模式(继承HystrixCommand类)来包裹具体的服务调用逻辑(run方法), 并在命令模式中添加了服务调用失败后的降级逻辑(getFallback).
同时我们在Command的构造方法中可以定义当前服务线程池和熔断器的相关参数. 如下代码所示:

```java
public class Service1HystrixCommand extends HystrixCommand<Response> {
  private Service1 service;
  private Request request;

  public Service1HystrixCommand(Service1 service, Request request){
    supper(
      Setter.withGroupKey(HystrixCommandGroupKey.Factory.asKey("ServiceGroup"))
          .andCommandKey(HystrixCommandKey.Factory.asKey("servcie1query"))
          .andThreadPoolKey(HystrixThreadPoolKey.Factory.asKey("service1ThreadPool"))
          .andThreadPoolPropertiesDefaults(HystrixThreadPoolProperties.Setter()
            .withCoreSize(20))//服务线程池数量
          .andCommandPropertiesDefaults(HystrixCommandProperties.Setter()
            .withCircuitBreakerErrorThresholdPercentage(60)//熔断器关闭到打开阈值
            .withCircuitBreakerSleepWindowInMilliseconds(3000)//熔断器打开到关闭的时间窗长度
      ))
      this.service = service;
      this.request = request;
    );
  }

  @Override
  protected Response run(){
    return service1.call(request);
  }

  @Override
  protected Response getFallback(){
    return Response.dummy();
  }
}
```

在使用了Command模式构建了服务对象之后, 服务便拥有了熔断器和线程池的功能.
![在这里插入图片描述](https://img-blog.csdnimg.cn/20210617100436758.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80NDY5OTM0OA==,size_16,color_FFFFFF,t_70#pic_center)


#### Hystrix的内部处理逻辑

下图为Hystrix服务调用的内部逻辑:

![在这里插入图片描述](https://img-blog.csdnimg.cn/2021061710045281.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80NDY5OTM0OA==,size_16,color_FFFFFF,t_70#pic_center)


1. 构建Hystrix的Command对象, 调用执行方法.
2. Hystrix检查当前服务的熔断器开关是否开启, 若开启, 则执行降级服务getFallback方法.
3. 若熔断器开关关闭, 则Hystrix检查当前服务的线程池是否能接收新的请求, 若超过线程池已满, 则执行降级服务getFallback方法.
4. 若线程池接受请求, 则Hystrix开始执行服务调用具体逻辑run方法.
5. 若服务执行失败, 则执行降级服务getFallback方法, 并将执行结果上报Metrics更新服务健康状况.
6. 若服务执行超时, 则执行降级服务getFallback方法, 并将执行结果上报Metrics更新服务健康状况.
7. 若服务执行成功, 返回正常结果.
8. 若服务降级方法getFallback执行成功, 则返回降级结果.
9. 若服务降级方法getFallback执行失败, 则抛出异常.

## **回退降级**

降级，通常指务高峰期，为了保证核心服务正常运行，需要停掉一些不太重要的业务，或者某些服务不可用时，执行备用逻辑从故障服务中快速失败或快速返回，以保障主体业务不受影响。Hystrix提供的降级主要是为了容错，保证当前服务不受依赖服务故障的影响，从而提高服务的健壮性。要支持回退或降级处理，可以重写HystrixCommand的getFallBack方法或HystrixObservableCommand的resumeWithFallback方法。

Hystrix在以下几种情况下会走降级逻辑：

- 执行construct()或run()抛出异常
- 熔断器打开导致命令短路
- 命令的线程池和队列或信号量的容量超额，命令被拒绝
- 命令执行超时

### **降级回退方式**

**Fail Fast 快速失败**

快速失败是最普通的命令执行方法，命令没有重写降级逻辑。 如果命令执行发生任何类型的故障，它将直接抛出异常。

**Fail Silent 无声失败**

指在降级方法中通过返回null，空Map，空List或其他类似的响应来完成。

```java
@Override
protected Integer getFallback() {
   return null;
}
 
@Override
protected List<Integer> getFallback() {
   return Collections.emptyList();
}
 
@Override
protected Observable<Integer> resumeWithFallback() {
   return Observable.empty();
}
```

**Fallback: Static**

指在降级方法中返回静态默认值。 这不会导致服务以“无声失败”的方式被删除，而是导致默认行为发生。如：应用根据命令执行返回true / false执行相应逻辑，但命令执行失败，则默认为true

```java
@Override
protected Boolean getFallback() {
    return true;
}
@Override
protected Observable<Boolean> resumeWithFallback() {
    return Observable.just( true );
}
```

**Fallback: Stubbed**

当命令返回一个包含多个字段的复合对象时，适合以Stubbed 的方式回退。

```java
@Override
protected MissionInfo getFallback() {
   return new MissionInfo("missionName","error");
}
```

**Fallback: Cache via Network**

有时，如果调用依赖服务失败，可以从缓存服务（如redis）中查询旧数据版本。由于又会发起远程调用，所以建议重新封装一个Command，使用不同的ThreadPoolKey，与主线程池进行隔离。

```java
@Override
protected Integer getFallback() {
   return new RedisServiceCommand(redisService).execute();
}
```

**Primary + Secondary with Fallback**

有时系统具有两种行为- 主要和次要，或主要和故障转移。主要和次要逻辑涉及到不同的网络调用和业务逻辑，所以需要将主次逻辑封装在不同的Command中，使用线程池进行隔离。为了实现主从逻辑切换，可以将主次command封装在外观HystrixCommand的run方法中，并结合配置中心设置的开关切换主从逻辑。由于主次逻辑都是经过线程池隔离的HystrixCommand，因此外观HystrixCommand可以使用信号量隔离，而没有必要使用线程池隔离引入不必要的开销。

主次模型的使用场景还是很多的。如当系统升级新功能时，如果新版本的功能出现问题，通过开关控制降级调用旧版本的功能。示例代码如下：

```java
public class CommandFacadeWithPrimarySecondary extends HystrixCommand<String> {
 
    private final static DynamicBooleanProperty usePrimary = DynamicPropertyFactory.getInstance().getBooleanProperty("primarySecondary.usePrimary", true);
 
    private final int id;
 
    public CommandFacadeWithPrimarySecondary(int id) {
        super(Setter
                .withGroupKey(HystrixCommandGroupKey.Factory.asKey("SystemX"))
                .andCommandKey(HystrixCommandKey.Factory.asKey("PrimarySecondaryCommand"))
                .andCommandPropertiesDefaults(
                        // 由于主次command已经使用线程池隔离，Facade Command使用信号量隔离即可
                        HystrixCommandProperties.Setter()
                                .withExecutionIsolationStrategy(ExecutionIsolationStrategy.SEMAPHORE)));
        this.id = id;
    }
 
    @Override
    protected String run() {
        if (usePrimary.get()) {
            return new PrimaryCommand(id).execute();
        } else {
            return new SecondaryCommand(id).execute();
        }
    }
 
    @Override
    protected String getFallback() {
        return "static-fallback-" + id;
    }
 
    @Override
    protected String getCacheKey() {
        return String.valueOf(id);
    }
 
    private static class PrimaryCommand extends HystrixCommand<String> {
 
        private final int id;
 
        private PrimaryCommand(int id) {
            super(Setter
                    .withGroupKey(HystrixCommandGroupKey.Factory.asKey("SystemX"))
                    .andCommandKey(HystrixCommandKey.Factory.asKey("PrimaryCommand"))
                    .andThreadPoolKey(HystrixThreadPoolKey.Factory.asKey("PrimaryCommand"))
                    .andCommandPropertiesDefaults(                          HystrixCommandProperties.Setter().withExecutionTimeoutInMilliseconds(600)));
            this.id = id;
        }
 
        @Override
        protected String run() {
            return "responseFromPrimary-" + id;
        }
 
    }
 
    private static class SecondaryCommand extends HystrixCommand<String> {
 
        private final int id;
 
        private SecondaryCommand(int id) {
            super(Setter
                    .withGroupKey(HystrixCommandGroupKey.Factory.asKey("SystemX"))
                    .andCommandKey(HystrixCommandKey.Factory.asKey("SecondaryCommand"))
                    .andThreadPoolKey(HystrixThreadPoolKey.Factory.asKey("SecondaryCommand"))
                    .andCommandPropertiesDefaults(  HystrixCommandProperties.Setter().withExecutionTimeoutInMilliseconds(100)));
            this.id = id;
        }
 
        @Override
        protected String run() {
            return "responseFromSecondary-" + id;
        }
 
    }
 
    public static class UnitTest {
 
        @Test
        public void testPrimary() {
            HystrixRequestContext context = HystrixRequestContext.initializeContext();
            try {
                ConfigurationManager.getConfigInstance().setProperty("primarySecondary.usePrimary", true);
                assertEquals("responseFromPrimary-20", new CommandFacadeWithPrimarySecondary(20).execute());
            } finally {
                context.shutdown();
                ConfigurationManager.getConfigInstance().clear();
            }
        }
 
        @Test
        public void testSecondary() {
            HystrixRequestContext context = HystrixRequestContext.initializeContext();
            try {
                ConfigurationManager.getConfigInstance().setProperty("primarySecondary.usePrimary", false);
                assertEquals("responseFromSecondary-20", new CommandFacadeWithPrimarySecondary(20).execute());
            } finally {
                context.shutdown();
                ConfigurationManager.getConfigInstance().clear();
            }
        }
    }
}
```

通常情况下，建议重写getFallBack或resumeWithFallback提供自己的备用逻辑，但不建议在回退逻辑中执行任何可能失败的操作。

## Hystrix Metrics的实现

Hystrix的Metrics中保存了当前服务的健康状况, 包括服务调用总次数和服务调用失败次数等. 根据Metrics的计数, 熔断器从而能计算出当前服务的调用失败率, 用来和设定的阈值比较从而决定熔断器的状态切换逻辑. 因此Metrics的实现非常重要.

#### 1.4之前的滑动窗口实现

Hystrix在这些版本中的使用自己定义的滑动窗口数据结构来记录当前时间窗的各种事件(成功,失败,超时,线程池拒绝等)的计数.
事件产生时, 数据结构根据当前时间确定使用旧桶还是创建新桶来计数, 并在桶中对计数器经行修改.
这些修改是多线程并发执行的, 代码中有不少加锁操作,逻辑较为复杂.

![在这里插入图片描述](https://img-blog.csdnimg.cn/20210617100515292.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80NDY5OTM0OA==,size_16,color_FFFFFF,t_70#pic_center)


#### 1.5之后的滑动窗口实现

Hystrix在这些版本中开始使用RxJava的Observable.window()实现滑动窗口.
RxJava的window使用后台线程创建新桶, 避免了并发创建桶的问题.
同时RxJava的单线程无锁特性也保证了计数变更时的线程安全. 从而使代码更加简洁.
以下为我使用RxJava的window方法实现的一个简易滑动窗口Metrics, 短短几行代码便能完成统计功能,足以证明RxJava的强大:

```java
@Test
public void timeWindowTest() throws Exception{
  Observable<Integer> source = Observable.interval(50, TimeUnit.MILLISECONDS).map(i -> RandomUtils.nextInt(2));
  source.window(1, TimeUnit.SECONDS).subscribe(window -> {
    int[] metrics = new int[2];
    window.subscribe(i -> metrics[i]++,
      InternalObservableUtils.ERROR_NOT_IMPLEMENTED,
      () -> System.out.println("窗口Metrics:" + JSON.toJSONString(metrics)));
  });
  TimeUnit.SECONDS.sleep(3);
}
```

## 总结

通过使用Hystrix,我们能方便的防止雪崩效应, 同时使系统具有自动降级和自动恢复服务的效果.
