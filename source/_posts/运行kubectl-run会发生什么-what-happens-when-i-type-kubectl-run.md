---
title: 运行kubectl run会发生什么(what happens when i type kubectl run)
date: 2024-04-17 20:09:04
categories:
- k8s
tags:
copyright:
top:
type:
---
> 原文链接：https://github.com/jamiehannaford/what-happens-when-k8s
>
想像一下当你运行下面的命令时
```
kubectl create deployment nginx --image=nginx --replica=3
```
在一切顺利的情况下，在k8s集群中能够看到生成了3个pod。那么在底层到底发生了什么？本篇文章试图解决一个请求从客户端到kubelet整体的流程，并且也链接了源码。

## 目录
1. kubectl
   - Validation and generators(验证和生成)
   - API groups and version negotiation(API组和版本协商)
   - Client auth(客户端认证)
2. kube-apiserver
   - Authentication（认证）
   - Authorization（鉴权）
   - Admission control（admission控制）
3. etcd
4. Initializers
5. Control loops
   - Deployments controller
   - ReplicaSet controller
   - Informers
   - Scheduler
6. kubelet
   - Pod sync
   - CRI and pause containers
   - CNI and pod networking
   - Inter-host networking
   - Container startup
7. Wrap-up（总结）

## kubectl
### Validation and generators
针对开头提到的命令，在命令行输入回车后，会发生什么呢？
kubectl首先会进行客户端侧的验证。针对非法的命令(比如不支持的资源或镜像不合法)能够快速失败提示，避免发送到kube-apiserver，从而降低k8s负载。
经过验证后，kubectl组装将要发送到api-server的http请求。在k8s中，获取或者改变k8s中的状态都要通过apiserver，apiserver负责与etcd交互。kubectl使用generators来构建http请求，generators是负责序列化的一种抽象（注：可以简单的把generators理解为帮帮助用户构建完成的资源的工具，比如kubectl create中我们只指定了部分的内容，剩下的内容是generators帮助我们生成的）。
kubectl run 命令可以通过使用--generator参数指定运行多种资源，而不仅仅是deployments。如果不使用--generator参数，kubectl可以主动推断要运行的资源。
例如，参数`--restart-policy=Always`会触发deployments，而`--restrart-policy=Never`会触发pods。kubectl也会根据其他的参数确定需要执行的其他动作。比如记录命令(回滚或审计)，或者`--dry-run`
在确定要创建的资源是Deployment后，kubectl将会使用`DeploymnetAppsV1` generator来根据命令行参数构造runtime object。runtime object是k8s resource的通用表示。

### API groups and version negotiation
k8s使用版本API，并且每个版本有自己所属的API groups。API groups意味着一些相似资源的集合。这比使用单调递增的版本号更加灵活。比如deployment的api group是apps，最近的版本是v1。这也是在deployment中写`apiVersion: apps/v1`的原因。
kubectl生成了runtime object后，kubectl开始找到正确的API group和version，并且会组装versioned client。versioned client能够意识到资源的各种REST 语义。这个阶段被称为版本协商，它涉及kubectl扫描远程API上的/apis路径，从而检索所有可能的API组。由于kube-apiserver在此路径上公开了其模式文档(以OpenAPI格式),因此客户端可以很容易执行自己的发现。
为了提高性能，kubectl会将OpenAPI schema缓存到 ~/.kube/cache/discovery 目录中。如果想要看到API 发现的过程，那么可以将这个缓存目录删除，然后运行kubectl命令时带上-v参数。你将会看到查询API versions的所有http请求。
kubectl最后一步是发送http请求。一旦接收到成功的响应后，那么kubectl就会按照期望的格式打印结果。

### Client auth
为了成功发送请求，kubectl需要能够处理认证信息。用户的凭证一般存储在`kubeconfig`文件中，但是有时候也在其他地方。kubectl做了下面的事情来决定用户凭证：
- 如果kubectl中使用了`--kubeconfig`，那么就使用这个参数指定的文件
- 如果当前环境变量中定义了`$KUBECONFIG`，那么使用这个环境变量指定的文件
- 否则使用推荐的主目录，比如`~/.kube`,然后使用此目录中的第一个文件
当kubectl解析了凭证文件后，kubectl会决定当前需要使用的上下文，当前需要连接的集群，与当前用户关联的认证信息。如果kubectl中提供了指定的参数，比如`--username`, 那么这些参数将会覆盖凭证文件的配置。一旦有了这些信息，那么kubectl将会用这些信息去修饰http请求：
- 使用tls.TLSConfig来发送x509证书
- bearer token放到http header中的"Authorization"
- username 和 password通过http basic authentication发送
- OpenID认证流程是由用户事先手动处理的，生成一个类似于Bearer令牌的token，然后发送。

## kube-apiserver
### Authentication(认证，确定用户是谁)
k8s客户端和k8s系统组件与k8s交互主要是通过kube-apiserver来进行，比如获取和存储系统状态。kube-apiserver第一步要做的就是要验证请求的身份信息，这一步叫做认证(authentication).
apiserver如何进行认证呢？首先，apiserver在启动时，它会首先检查启动参数，然后组装一系列的认证器(authenticators)。举个例子，比如`--client-ca-file`传递进来，那么会添加x509认证器；如果提供`--token-auth-file`，那么会添加token认证器。apiserver收到的每个请求都会经过这些认证器，并且会依次进行认证，直到有一个认证器成功，那么请求就通过了。
- x509 handler 将会检查请求的证书是否有效，并且是否在指定的ca文件中
- bearer token handler 将会检查token(http Authorization header)是否在指定的文件中(由`--token-auth-file`指定)
- basic handler仅仅简单的检查http 请求中的basic 认证信息是否与本地状态匹配
如果每个认证器都认证失败了，那么一个组装的错误(每个错误的认证)会返回。如果认证成功，那么在http header中的`Authorization`会移除，并且用户信息会添加到上下文中。这样后续的apiserver处理逻辑(比如authorization和admission controllers)能够直接获取到用户信息。

### Authorization(鉴权，确定用户是否有权限)
经过认证后，apiserver会进行鉴权检查，确定执行的请求动作是否允许。
apiserver处理鉴权跟认证类似。基于启动的参数，apiserver会组装一系列的鉴权器(authorizers), 这些鉴权器会依次检查请求。如果所有的鉴权都没有通过，那么请求会被拒绝并返回给客户端。如果其中某个鉴权器通过，那么请求会继续。
下面展示了一些鉴权器：
- webhook，负责与k8s集群外的http(s) service进行交互的
- ABAC，执行定义在static文件中的鉴权策略
- RBAC，采用RBAC角色来进行鉴权
- Node，确保node clients(比如kubelet)只能够访问托管到自身上的k8s资源。
### Admission control
从apiserver的角度看，它已经确定了请求是谁以及请求是否能够执行。但是对于kubernetes，系统的其他部分对于能够做什么和不能做什么都有自己的想法。这就是admission controller要做的事情。
鉴权侧重于用户是否有权限，而admission controllers是用来确保请求符合期望以及遵守集群规则。amission controller是对象存储到etcd之前的最后一个处理，因此它封装了系统剩余的检查以确定请求不会产生意想不到的后果。
admission controllers的逻辑与authenticators和authorizers类似，但是不同之处是：不像authenticator和authorizers链式调用，如果一个admission controller失败，那么整个admission chain会失败。
amission controllers比较优秀的设计是专注于提升扩展性。每个controller都作为一个插件放置到`plugin/pkg/admission`目录下，每个controller都满足一个比较小的接口定义。然后编译到k8s中。
admission controllers通常分为这几类，资源管理、安全性、默认设置和引用一致性。下面列出了专注于资源管理的admission controller：
- InitialResources：根据过去资源使用情况，设置默认的资源限制
- LimitRanger：设置容器资源的request和limit或者为资源设置资源上界。
- ResourceQuota：确保集群中资源使用量不会超过配额

## etcd
apiserver验证了请求后，下一步apiserver反序列化http请求，从http请求中构建runtime objects(类似于kubectl generators的反过程)，然后将objects存储到底层存储中。将这个过程细化如下。
首先，apiserver在接受到请求后是怎么知道如何处理呢？这个过程比较复杂。我们首先看下apiserver在启动时都会做什么：
1. 当前apiserver启动时，它会创建server chain，是一个组合的调用链。基本上是由多个apiserver组成的。
2. server chain中有个通用的apiserver，是一个默认的实现
3. openAPI schema会填充到apiserver的配置中
4. apiserver遍历所有在openAPI schema中定义的api groups，然后为每个api group都配置一个storage provider，这些storage provider是一个通用的存储抽象。apiserver主要通过与这些storage provider交互来存储和检索数据。
5. 在每个api group中，apiserver会遍历所有的版本，然后为每个http 路径配置rest mapping。这样apiserver就能够将收到的外部请求与对应的处理逻辑关联起来
6. 针对我们上面的例子，apiserver已经注册了post handler。这个handler被委托用来创建资源
到这里，apiserver已经知道有哪些http route存在，并且如何将handler跟route进行映射。让我们现在看下http 请求进来后的处理
1. 如果请求跟某个特定的route匹配上，那么就会交给对应的handler进行处理。如果没有匹配上，那么会采用基于url 路径的方式进行匹配。如果针对这个路径也没有匹配上，那么一个not found handler会被调用，然后返回404错误。
2. 针对我们的例子，有一个注册的路由会调用createHandler方法。它首先解码http请求，执行基础的验证。比如确保http提供的json格式复合对应的版本资源。
3. 触发审计以及最后的admission
4. 专用的storage provider将资源存储到etcd中。etcd中资源的key通常是`<namespace>/<name>`，但是这个格式是可以配置的
5. 创建中的任何错误都会被捕获，最后storage provider会执行get请求来确保资源一定是创建成功的。如果需要，还会调用而外的后处理逻辑。
6. 构造http 响应并返回
通过上面的步骤，我们知道apiserver做了很多的步骤。当前deployment已经成功的在etcd中创建了。但是当前deployment还没有被调度
## Initializers
在object存储到etcd之后，apiserver或者调度器并不能立马看到它。直到一系列的initializers执行完毕后，才会看到这个对象。初始化控制器将资源类型和要执行的逻辑进行关联。如果一个资源没有注册初始化控制器，那么初始化阶段会调用，这个资源立马会被apiserver或调度器看到。
初始化控制器给我们提供了扩展k8s的能力，比如：
- 给pod注入代理sidecar 容器，或者获取特定的注解
- 向特定的命名空间下的pod注入证书
- 阻止创建小于20个字符的secret小于
`initializerConfiguration`允许你声明哪些资源运行哪些initializers。假设我们想要一个在每个pod创建时都运行的initializer，我们的配置如下：
```yaml
apiVersion: admissionregistration.k8s.io/v1alpha1
kind: InitializerConfiguration
metadata:
   name: custom-pod-initializer
initializers:
   - name: podimage.example.com
     rules:
       - apiGroups:
            - ""
         apiVersions:
            - v1
         resources:
            - pods
```
上面的InitializerConfiguration会给每个pod的`metadata.initializers.pending`字段添加`podimage.example.com`。initializer controller已经提前部署好，initializer会扫描每个新建的pod。当发现pod的`metadata.initializers.pending`字段不为空时，initializer controller就会执行。执行完后，会将对应的`podimage.example.com`删除。当所有的初始化器执行完毕并且`pending`字段为空，则此对象被认为已经初始化完成。
可能你已经发现一个问题，如果资源对于apiserver不可见，那么我们这里的初始化控制器怎么处理呢？apiserver暴露了一个`includeUninitialized`查询参数来返回所有的对象，包括那些还未初始化的对象。

## Control loops
### Deployments controller
到现在这个阶段，deployment已经存储到etcd中并且初始化逻辑已经完成了。接下来就要设置资源拓扑。deployment就是replicasets的集合，而replicaSet就是pod的集合。k8s如何从http 请求中去创建这些对象呢？这就是k8s内置controller需要做的内容。
k8s在整个系统中大量使用了"控制器"思想。控制器就是一个异步过程，调谐k8s当前的状态到期望状态。每个控制器有自己的职责，并且由`kube-controller-manager`组件负责并发运行。下面介绍deployment controller
在 deployment存储到etcd并且初始化完成后，那么deployment就能够被kube-apiserver看到。deployment controller能够检测到可用的deployment资源以及对于资源的变更。在我们的例子中，deployment controller通过informer为创建事件注册了一个特定的回调。
当deployment可用时，handler开始执行，将对象添加到内部的工作队列中。当开始处理这个对象时，控制器会检查deployment，发现deployment没有关联的ReplicaSet或者Pod记录。这是通过使用标签选择器来查询kube-apiserver来完成的。有趣的是，这个同步过程是状态未知的：调谐新的记录跟调谐旧的记录是一样的工作方式。
在意识到相关的ReplcaSet和Pod不存在后，deployment controller会开启scaling process来解析状态。创建一个ReplicaSet资源，给它分配标签选择器，然后给定数字1的版本号。ReplicaSet中的PodSpec是从deployment清单中拷贝而来，类似于其他相关的元数据。有些时候，deployment记录在这个过程后也需要对应的更新(比如，设定了截止日期)
更新deployment中的status字段，然后控制器重新进行调谐过程，然后等待deployment达到期望的状态。由于deployment controller仅仅关心创建ReplicaSets,接下来的工作需要由下一个控制器来完成，既ReplicaSet controller。
### ReplicaSet controller
在前面的步骤中，Deployments controller创建了ReplicaSet,但是还没有创建Pods。这就是ReplicaSet 控制器发挥作用的时刻。ReplicaSet controller的主要工作就是监视ReplicaSet的生命周期和对应的Pods资源。类似于大多数其他的控制器，它通过在特定事件上触发handler来实现。
我们感兴趣的事件是创建。当ReplicaSet被创建后。ReplicaSet controller检查新的replicaSet的状态，然后发现当前的状态跟期望的状态不一致。它试图通过增加属于ReplicaSet的Pod数量来达到目标状态。它以谨慎的方式来创建它们，确保ReplicaSet的数量始终匹配。
创建Pods的过程是批量处理的，以`SlowStartInitiaBatchSize`开始，然后每次都以两倍的数量来启动其他的Pods。这样做的目的是确保当pod创建失败时，避免大量的http 请求到apiserver。如果要失败，可以采用对组件影响最小的方式来进行。
k8s通过Owner References(child资源中的一个字段，用来指向父资源的id)来确保对象的引用关系。这样做，不仅能够确保一旦父资源被删除，子资源也会被删除(级联删除)，同时还能够保证父资源不会竞争子资源(想象一下，两个潜在的父资源认为他们拥有同样的子资源)
Owner Reference另一个好处是，它是有状态的。如果控制器重启了，那么重启过程中，不会影响到这些资源，因为资源拓扑是独立于controler的。这种对隔离的关注也渗透到控制器本身的设计中：它们不应该操作它们没有明确拥有的资源。相反，控制器应该只操作它们明确拥有的资源。
然而，有时候会存在"孤儿"资源，比如下面这些情况：
1. 父资源删除，但是子资源没有删除
2. 垃圾回收策略禁止删除子资源
如果发生了这种情况，那么控制器会确保这些“孤儿”资源被新的父资源领养。多个父资源能够竞争性的去领养孤儿资源，但是只有一个能够成功。
### Informers
rbac authorizer 或者 deployment controller需要获取集群状态，从而实现相关逻辑。比如以rbac authorizer为例，当请求进入时，authenticator需要将用户的初始状态保存起来以供后续使用。rbac authorizer后续会使用用户的初始表示来获取在etcd中相关的角色和角色绑定关系。控制器应该如何访问和修改这些资源？k8s中提供了informers来实现。
informer是一种模式，既允许控制器订阅存储时间以及查询资源列表。informer除了提供一种抽象外，它还封装了大量的工作，比如缓存(缓存很重要，不仅可以减少apiserver连接数，同时还可以较少服务端和客户端的序列化)。informer还提供了线程安全的方式来操作资源。
### Scheduler
在所有的控制器都运行后，当前在etcd中会存在一个deployment，一个ReplicaSet以及三个pod资源。此时，pod的状态是`Pending`,因为它们还没有被调度。最后的一个控制器是scheduler，它负责调度pod。
scheduler是作为控制面的一个标准组件，它的运行机制跟其他的控制器类似。它监听事件，然后尝试将状态调整到期望状态。在这个例子中，调调度器筛选出`NodeName`为空的pod，然后尝试将它们调度到合适的节点上。
为了找到合适的节点，scheduler使用了专门的调度算法。默认的调度算法工作如下：
1. scheduler启动时，会注册一系列的默认predicates。这些predicates会检查pod是否满足特定的条件，比如：如果PodSpec中明确指定了CPU或RAM资源，如果Node上的资源不满足，那么此Node就会被过滤掉。
2. 一旦选择了一些符合条件的节点，那么接下来priority函数将会给这些节点打分。比如：为了在集群中使得资源均匀分配，那么节点剩余资源较多的节点会得到更高的分数，最终选择一个分数最高的节点作为目标节点。
当找到合适节点后，scheduler会创建一个Binding Object，其中名称和UID会匹配对应的pod。Binding Object的ObjectReference字段会包含选择的目标节点。通过post请求会发送给apiserver。
当apiserver接受到Binding object后，它会反序列化请求，然后将信息更新到pod对象上：设置pod的NodeName字段，添加相关的注解以及将`PodScheduled`设置为`True`.
一旦scheudler将pod调度到目标节点后，剩下的工作就交与kubelet来完成了。
> 注：定制调度器。通过`--policy-config-file`可以定制predicate和priority函数。我们可以在k8s集群中运行特定的调度器，比如采用deployment运行。如果在PodSpec中指定了`schedulerName`,那么k8s会将调度过程交给这个调度器执行。

## kubelet
### Pod sync
让我们总结下上面的过程：
1. HTTP请求经过认证、鉴权以及admission controller阶段；
2. 一个deployment、一个replicaSet、三个pod持久化到etcd中
3. 一系列初始化器运行
4. 最后，每个pod被调度到合适的节点上
到这个时候，数据仅仅存在于etcd中。接下来，就需要将pod在worker节点上启动。这个是通过kubelet组件来实现的。
kubelet是k8s集群中运行在每个节点上的组件，主要负责管理pod的生命周期。这意味kubelet需要将pod转换成具体的容器、网络等。同时，它也负责处理卷挂载、容器日志、垃圾收集以及其他重要的事情。
另一种考虑kubelet的方式是将它看做是一个控制器。kubectl从kube-apiserver每个20秒钟（可以配置）查询pod信息，挑选出`NodeName`于当前节点名称一致的pod。然后它将pod与本地内部缓存中pod的信息进行比较，如果发现有差异就进行同步。下面来具体看下同步的过程是怎么样的：
1. 如果pod是要进行创建，那么kubelet会注册一些metrics，这些metrics主要给prometheus用于追踪pod的延迟。
2. kubectl生成PodStatus对象，PodStatus代表了Pod的当前状态(Phase)。pod的状态是pod生命周期的更高一级的抽象。比如`Pending`，`Running`,`Succeeded`,`Failed`和`Unknown`。产生这些状态是比较复杂的，让我们深挖一下：
   - 首先，一系列的`PodSyncHandlers`会顺序执行。每个handler都会检查pod是否应该在当前节点上。如果任意的handler判断pod不应该在当前节点上，那么pod的phase将会变成`PodFailed`，并且pod会从此节点上被剔除。比如在Job资源中设置了`activeDeadlineSeconds`，超过了设置的时间，那么pod就会被剔除。
   - 接下来，pod的phase会由初始化容器以及主容器的状态来决定。因为在这里容器还没有启动，这些容器会被归类为等待。具有等待容器的pod的phase都是`Pending`
   - 最后,Pod Condition会由容器的Condition来决定。因为我们的容器当前还没有被容器运行时创建，所以kubelet会将`PodReady` condition设置为False
3. 当PodStatus产生后，PodStatus对象会被发送到Pod的status manager中。该管理器的任务是通过apiserver异步更新etcd记录
4. 接下来，将运行一系列准入处理程序，以确保pod具有正确的安全权限。这包括强制的AppArmor profiles 和 NO_NEW_PRIVS。在这个阶段被拒绝的pod将无限期地处于挂起状态。
5. 如果kubelet在启动时指定了`cgroups-per-qos`参数，那么kubelet将会为pod创建cgroups并且会应用这些资源参数。这是为了实现对pod更好地服务质量(QoS)处理。
6. 给pod创建数据目录。包括pod目录(通常是：`/var/run/kubelet/pods/<podID>`)，卷目录(`<podDir>/volumes`)以及插件目录(`<podDir>/plugins`)
7. 卷管理器(volume manager)等待`Spec.Volumes`中定义的卷准备好。根据挂载卷的类型，有些pod需要等待更长的时间(比如云或NFS卷)。
8. 在`Spec.ImagePullSecrets`中定义的secrets资源，会从apiserver中获取。从而后续会注入到容器中
9. 容器运行时开始启动容器
### CRI and pause containers
到现在，大多数工作已经完成了，容器已经准备好启动了。启动容器的部分叫做容器运行时(Container Runtime，比如`docker`或`rkt`)。
为了增加扩展性，kubelet自从v1.5.0版本以来，提出了CRI(Container Runtime Interface),CRI用于与具体的容器运行时进行交互。总的来说，CRI是一个抽象层。kubelet和容器运行时通过protocal buffers以及对应的gRPC API接口进行交互。CRI带来了很大的好处，如果后续替换下层的容器运行时，核心的k8s代码不需要做任何修改。
我们回到部署容器的过程。当pod启动后，kubelet通过RPC启动`RunPodSandbox`。沙箱是一个CRI术语，用来描述一组容器，在k8s中被称作pod。
在我们的例子中，我们使用了docker。在沙箱中首先创建一个"pause"容器。pause容器主要是为了给pod中其他的容器提供服务，提供必要的资源。这些资源包括linux namespace(IPC、网络、PID等)。如果你不熟悉linux中容器如何工作。我们快速说一下。linux内核有命名空间的概念，命名空间主要是隔离专有的资源，比如CPU、内存。然后将这些资源指定给特定的进程，从而保证资源只能被指定的进程使用。Cgroups主要是给linux提供了资源分配的方式。Docker利用了linux命名空间和Cgroups来实现容器。
pause容器提供了所有的命名空间，并且允许子容器共享他们。作为同一个网络命名空间的一部分，一个好处是同一个pod中的容器可以通过localhost互相引用。pause容器的第二个角色是，PID命名空间如何工作。进程形成了等级树，最顶层的init进程，负责收割死进程。在pause容器创建后，它被检查点指向磁盘，然后启动。
### CNI and pod networking
pod当前具有了基本的内容：pause容器持有所有的命名空间，从而pod内部的容器可以交流。但是网络如何工作以及如何设置？
kubelet将设置网络的工作委托给了CNI(Container Network Interface)插件。CNI的工作方式类似于Container Runtime Interface。简单来说，CNI提供了一种抽象，用于给不同的网络提供者使用不同的网络实现。kubelet与这些注册的CNI插件通过流式json数据进行交流。下面展示了json配置的例子：
```json
{
   "cniVersion": "0.3.1",
   "name": "bridge",
   "type":"bridge",
   "bridge": "cnio0",
   "isGateway": true,
   "isMasq": true,
   "ipam": {
      "type": "host-local",
      "ranges": [
         [{"subnet": "${POD_CIDR}"}]
      ],
      "routes": [{"dst": "0.0.0.0/0"}]
   }
}
```
json也为pod提供了特定的额外的元数据，比如通过`CNI_ARGS`环境变量提供名称和命名空间。
接下来发生的事情与具体的CNI插件有关，我们看看`bridge` CNI插件的工作：
1. 插件首先在root命名空间中设置一个本地的linux bridge，这个bridge为当前host(本机)上的所有容器服务
2. 接下来，插件创建veth pair，其中一端在pause容器，另一端在bridge中。可以将veth pair想象成一个管道，一端连接到容器，另一端连接到root 网络命名空间，从而允许容器和root网络命名空间进行通信。
3. 现在应该给pause容器分配IP地址，并且设置路由。这样做之后pod就有了自己的ip地址。ip地址分配是委托给IPAM，IPAM在json中定义。
   - IPAM插件类似于主要的网络插件：通过二进制调用并且有标准化的接口。每种IPAM插件都必须去顶容器的IP/子网，网络和路由，然后将这些信息返回给CNI插件。最常用的IPAM插件是`host-local`,它从一个预先定义好的地址范围中分配IP地址。它将已经分配的ip地址存储在本地的文件系统中，从而确保在本机上不会重复使用ip地址。
4. kubelet会指定一个内部的DNS服务器ip地址给CNI插件，保证容器的`resolv.conf`文件被正确设置。
一定这个过程完成，CNI插件会返回json数据给kubelet，表明网络设置的结果。
### Inter-host networking
我们现在仅仅说了容器如何同主机进行通信。但是如果pod在不同的主机上，那么pod之间如何通信呢？
这通常是基于一种叫做overlay networking的技术来实现，这是一种跨多个主机动态同步路由的方法。一个流行的overlay netwoker实现是Flannel。Flannel的核心作用是在不同节点之间提供IPv4网络。Flannel不会控制容器如何跟主机通信(这是CNI的职责)，而是负责流量在不同主机间的转发。为了实现这个功能，Flannel为主机选择了一个子网，将这个信息存储到etcd中。它保留集群路由的本地表示，并将传出的数据包封装在UDP数据报中，确保它到达正确的主机。
### Container startup
所有的网络工作都完成了，接下来就是启动工作容器了。
一旦沙箱完成初始化并且仍然是活跃的，那么kubelet将会为其创建工作容器。kubelet首先启动定义在PodSpec中定义的初始化容器，然后启动主容器。主要过程如下：
1. 拉取容器的镜像。PodSpec中定义的secrets用于私有仓库
2. 通过CRI创建容器.填充`ContainerConfig`结构体(包括启动命令、镜像、标签、挂载、环境变量等)，数据来自于PodSpec。然后通过protobufs协议发送给CRI插件。对于Docker来说，docker会反序列化请求，然后填充到自己的config结构体，然后发送给docker的后台进程。在这个过程中，它将一些元数据标签（如容器类型、日志路径、沙盒id）应用到容器。
3. 然后，它向CPU管理器注册容器，这是1.8中的一个新的alpha功能，通过使用UpdateContainerResources CRI方法将容器分配给本地节点上的CPU集。
4. 容器启动
5. 若定义了post-start hook，则执行这些hook。hooks可以是`Exec`（在容器中执行特定的命令）或`Http`（执行一个http请求）。如果PostStart hook需要很长时间运行、或者失败，那么容器将不会到达running状态

## Wrap-up(总结)
到这里，已经产生了三个容器，他们可能运行在一个或者多个工作节点上。所有的网络、卷、secret都已经被kubelet填充，并且通过CRI插件填充到了容器中。