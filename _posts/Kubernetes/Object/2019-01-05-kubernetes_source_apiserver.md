---

layout: post
title: Kubernetes源码分析——apiserver
category: 技术
tags: Kubernetes
keywords: kubernetes 源码分析 apiserver

---

## 简介

* TOC
{:toc}

apiserver 核心职责
1. 提供Kubernetes API
2. 代理集群组件，比如Kubernetes dashboard、流式日志、`kubectl exec` 会话

## 声明式API

1. 命令式命令行操作，比如直接 `kubectl run`
2. 命令式配置文件操作，比如先`kubectl create -f xx.yaml` 再 `kubectl replace -f xx.yaml` 
3. 声明式API 操作，比如`kubectl apply -f xx.yaml`。**命令式api 描述和执行 是一体的，声明式api 则需要额外的 执行器**（下文叫Controller） sync desired state 和 real state。

声明式API 有以下优势

1. 实现层的逻辑不同。kube-apiserver 在响应命令式请求（比如，kubectl replace）的时候，一次只能处理一个写请求，否则会有产生冲突的可能。而对于声明式请求（比如，kubectl apply），一次能处理多个写操作，并且具备 Merge 能力。
2. 如果xx.yaml 不变，可以任意多次、同一时间并发 执行apply 操作。
3. “声明式 API”允许有多个 API 写端，以 PATCH 的方式对 API 对象进行修改，而无需关心本地原始 YAML文件的内容。例如lstio 会自动向每一个pod 写入 envoy 容器配置（用户无感知），如果xx.yaml 是一个 xx.sh 则该效果很难实现。

[火得一塌糊涂的kubernetes有哪些值得初学者学习的？](https://mp.weixin.qq.com/s/iI5vpK5bVkKmdbf9sbAGWw)在分布式系统中，任何组件都可能随时出现故障。当组件恢复时，需要弄清楚要做什么，使用命令式 API 时，处理起来就很棘手。但是使用声明式 API ，组件只需查看 API 服务器的当前状态，即可确定它需要执行的操作。《阿里巴巴云原生实践15讲》 称之为：**面向终态**自动化。

## k8s api 术语

1. Kind, 表示实体的类型。直接对应一个Golang的类型，会持久化存储在etcd 中
2. API group, 在逻辑上相关的一组 Kind 集合。如 Job 和 ScheduledJob 都在 batch API group 里。
3. Version, 标示 API group 的版本更新， API group 会有多个版本 (version)。v1alpha1: 初次引入 ==> v1beta1: 升级改进 ==> v1: 开发完成毕业
4. Resource, 通常是小写的复数词（例如，pods），用于标识一组 HTTP 端点（路径），来对外暴露 CURD 操作。

![](/public/upload/kubernetes/k8s_rest_api.png)

每个 Kind 和 Resource 都存在于一个APIGroupVersion 下，分别通过 GroupVersionKind 和 GroupVersionResource 标识。关联GVK 到GVR （资源存储与http path）的映射过程称作 REST mapping。PS： 这在代码命名上有非常直接的体现

## 分层架构

[apiserver分析-路由注册管理](https://mp.weixin.qq.com/s/pto9_I5PWDWw1S0lklVrlQ)

![](/public/upload/kubernetes/apiserver_overview.png)

适合从下到上看。不考虑鉴权等，先解决一个Kind 的crudw，多个Kind （比如/api/v1/pods, /api/v1/services）汇聚成一个APIGroupVersion，多个APIGroupVersion（比如/api/v1, /apis/batch/v1, /apis/extensions/v1） 汇聚为 一个GenericAPIServer 即api server。

### go-restful框架

API Server使用了go-restful框架，按照go-restful的原理，包含以下的组件
1. Container: 一个Container包含多个WebService
2. WebService: 一个WebService包含多条route
3. Route: 一条route包含一个method(GET、POST、DELETE，WATCHLIST等)，一条具体的path以及一个响应的handler

```go
ws := new(restful.WebService)
ws.Path("/users").
  Consumes(restful.MIME_XML, restful.MIME_JSON).
  Produces(restful.MIME_JSON, restful.MIME_XML)
ws.Route(ws.GET("/{user-id}").To(u.findUser).
  Doc("get a user").
  Param(ws.PathParameter("user-id", "identifier of the user").DataType("string")).
  Writes(User{}))    
...
func (u UserResource) findUser(request *restful.Request, response *restful.Response) {
  id := request.PathParameter("user-id")
  ...
}
```
### 存储层

位于 `k8s.io/apiserver/pkg/storage` 下

```go
// k8s.io/apiserver/pkg/storage/interface.go
type Interface interface {
    Versioner() Versioner
    Create(ctx context.Context, key string, obj, out runtime.Object, ttl uint64) error
    Delete(ctx context.Context, key string, out runtime.Object, preconditions *Preconditions) error
    Watch(ctx context.Context, key string, resourceVersion string, p SelectionPredicate) (watch.Interface, error)
    Get(ctx context.Context, key string, resourceVersion string, objPtr runtime.Object, ignoreNotFound bool) error
    List(ctx context.Context, key string, resourceVersion string, p SelectionPredicate, listObj runtime.Object) error
    ...
}
```

封装了对etcd 的操作，还提供了一个cache 以减少对etcd 的访问压力。在Storage这一层，并不能感知到k8s资源对象之类的内容，纯粹的存储逻辑。

### registry 层

实现各种资源对象的存储逻辑

1. `kubernetes/pkg/registry`负责k8s内置的资源对象存储逻辑实现
2. `k8s.io/apiextensions-apiserver/pkg/registry`负责crd和cr资源对象存储逻辑实现

```
k8s.io/apiserver/pkg/registry
    /generic
        /regisry
            /store.go       // 对storage 层封装，定义 Store struct
k8s.io/kubernetes/pkg/registry/core
    /pod
        /storage
            /storage.go     // 定义了 PodStorage struct，使用了Store struct
    /service
    /node
    /rest
        /storage_core.go
```

registry这一层比较分散，k8s在不同的目录下按照k8s的api组的管理方式完成各自资源对象存储逻辑的编写，主要就是定义各自的结构体，然后和Store结构体进行一次组合。

```go
type PodStorage struct {
	Pod                 *REST
	Log                 *podrest.LogREST
	Exec                *podrest.ExecREST
	...
}
type REST struct {
	*genericregistry.Store
	proxyTransport http.RoundTripper
}
func (c LegacyRESTStorageProvider) NewLegacyRESTStorage(restOptionsGetter generic.RESTOptionsGetter) (LegacyRESTStorage, genericapiserver.APIGroupInfo, error) {
    ...
    // 关联路径 与各资源对象的关系
    restStorageMap := map[string]rest.Storage{
		"pods":             podStorage.Pod,
		"pods/attach":      podStorage.Attach,
		"pods/status":      podStorage.Status,
		"pods/log":         podStorage.Log,
		"pods/exec":        podStorage.Exec,
		"pods/portforward": podStorage.PortForward,
		"pods/proxy":       podStorage.Proxy,
		"pods/binding":     podStorage.Binding,
		"bindings":         podStorage.LegacyBinding,
    }
}
```

### endpoint 层

位于 k8s.io/apiserver/pkg/endpoints 包下。根据Registry层返回的路径与存储逻辑的关联关系，完成服务器上路由的注册。

```go
// k8s.io/apiserver/pkg/endpoints/installer.go
type APIInstaller struct {
	group             *APIGroupVersion
	prefix            string // Path prefix where API resources are to be registered.
	minRequestTimeout time.Duration
}
// 一个Resource 下的 所有处理函数 都注册到 restful.WebService 中了
func (a *APIInstaller) registerResourceHandlers(path string, storage rest.Storage, ws *restful.WebService) (*metav1.APIResource, error) {
    // 遍历所有操作，完成路由注册
    for _, action := range actions {
        ...
        switch action.Verb {
            case "GET": // Get a resource.
                ...
                route := ws.GET(action.Path).To(handler).
                    Doc(doc).
                    Param(ws.QueryParameter("pretty", "If 'true', then the output is pretty printed.")).
                    Returns(http.StatusOK, "OK", producedObject).
                    Writes(producedObject)
                ...
                routes = append(routes, route)
            case ...
        }
        ...
        for _, route := range routes {
			...
			ws.Route(route)
		}
    }
}
func (a *APIInstaller) Install() ([]metav1.APIResource, *restful.WebService, []error) {
	ws := a.newWebService()
	...
	for _, path := range paths {
		apiResource, err := a.registerResourceHandlers(path, a.group.Storage[path], ws)
		apiResources = append(apiResources, *apiResource)
	}
	return apiResources, ws, errors
}
type APIGroupVersion struct {
	Storage map[string]rest.Storage // 对应上文的restStorageMap 
    Root string
}
// 一个APIGroupVersion 下的所有Resource处理函数 都注册到 restful.Container 中了
func (g *APIGroupVersion) InstallREST(container *restful.Container) error {
	prefix := path.Join(g.Root, g.GroupVersion.Group, g.GroupVersion.Version)
	installer := &APIInstaller{
		group:             g,
		prefix:            prefix,
		minRequestTimeout: g.MinRequestTimeout,
	}
	apiResources, ws, registrationErrors := installer.Install()
	...
	container.Add(ws)
	return utilerrors.NewAggregate(registrationErrors)
}
```

同时在Endpoints还应该负责路径级别的操作：比如：到指定类型的认证授权，路径的调用统计，路径上的操作审计等。这部分内容通常在endpoints模块下的fileters内实现，这就是一层在http.Handler外做了一层装饰器，便于对请求进行拦截处理。

### server 层

Server模块对外提供服务器能力。主要包括调用调用Endpoints中APIInstaller完成路由注册，同时为apiserver的扩展做好服务器层面的支撑（主要是APIService这种形式扩展）

```go
// 注册所有 apiGroupVersion 的处理函数 到restful.Container 中
func (s *GenericAPIServer) installAPIResources(apiPrefix string, apiGroupInfo *APIGroupInfo, openAPIModels openapiproto.Models) error {
	for _, groupVersion := range apiGroupInfo.PrioritizedVersions {
		apiGroupVersion := s.getAPIGroupVersion(apiGroupInfo, groupVersion, apiPrefix)
		if err := apiGroupVersion.InstallREST(s.Handler.GoRestfulContainer); err != nil {
			...
		}
	}
	return nil
}
```

除了路由注册到服务器的核心内容外，server模块还提供了如下内容：
1. 路由层级的日志记录（在httplog模块）
2. 健康检查的路由（healthz模块）
3. 服务器级别的过滤器（filters模块），如，cors,请求数，压缩，超时等过滤器，
4. server级别的路由（routes模块），如监控，swagger，openapi，监控等。

## 拦截api请求

1. Admission Controller
2. Initializers
3. webhooks, If you’re not planning to modify the object and intercepting just to read the object, [webhooks](https://kubernetes.io/docs/reference/access-authn-authz/extensible-admission-controllers/#external-admission-webhooks) might be a faster and leaner alternative to get notified about the objects. Make sure to check out [this example](https://github.com/caesarxuchao/example-webhook-admission-controller) of a webhook-based admission controller.

### Admission Controller

准入控制器是kubernetes 的API Server上的一个链式Filter，它根据一定的规则决定是否允许当前的请求生效，并且有可能会改写资源声明。比如

1. enforcing all container images to come from a particular registry, and prevent other images from being deployed in pods. 
2. applying pre-create checks
3. setting up default values for missing fields.

The problem with admission controllers are:

1. **They’re compiled into Kubernetes**: If what you’re looking for is missing, you need to fork Kubernetes, write the admission plugin and keep maintaining a fork yourself.
2. You need to enable each admission plugin by passing its name to --admission-control flag of kube-apiserver. In many cases, this means redeploying a cluster.
3. Some managed cluster providers may not let you customize API server flags, therefore you may not be able to enable all the admission controllers available in the source code.

### Initializers

[How Kubernetes Initializers work](https://ahmet.im/blog/initializers/)

Initializers are not part of Kubernetes source tree, or compiled into it; you need to write a controller yourself.

**When you intercept Kubernetes objects before they are created, the possibilities are endless**: You can mutate the objects in any way you like, or prevent the objects from being created.Here are some ideas for initializers, each enforce a particular policy in your cluster:

1. Inject a proxy sidecar container to the pod if it has port 80, or has a particular annotation.
2. Inject a volume with test certificates to all pods in the test namespace automatically.
3. If a Secret is shorter than 20 characters (probably a password), prevent its creation.

Anatomy of Initialization

1. Configure which resource types need initialization
2. API server will assign initializers to the new resources
3. You will write a controller to watch for the resources
4. Wait for your turn to modify the resource
5. Finish modifying, yield to the next initializer
6. No more initializers, resource ready to be realized. When Kubernetes API server sees that the object has no more pending initializers, it considers the object “initialized”. **Now the Kubernetes scheduler and other controllers can see the fully initialized object and make use of them**.

从中可以看到，为啥Admission Controller 干活要改源码，Initializers 不用呢？ 因为干活要改源码，Initializer 只是给待处理资源加上了标记`metadata.initalizers.pending=InitializerName`，需要相应的Controller 打辅助。

[示例](https://github.com/kelseyhightower/kubernetes-initializer-tutorial)

## etcd: Kubernetes’ brain

**Every component in Kubernetes (the API server, the scheduler, the kubelet, the controller manager, whatever) is stateless**. All of the state is stored in a key-value store called etcd, and communication between components often happens via etcd.

For example! Let’s say you want to run a container on Machine X. You do not ask the kubelet on that Machine X to run a container. That is not the Kubernetes way! Instead, this happens:

1. you write into etcd, “This pod should run on Machine X”. (technically you never write to etcd directly, you do that through the API server, but we’ll get there later)
2. the kublet on Machine X looks at etcd and thinks, “omg!! it says that pod should be running and I’m not running it! I will start right now!!”

When I understood that basically everything in Kubernetes works by watching etcd for stuff it has to do, doing it, and then writing the new state back into etcd, Kubernetes made a lot more sense to me.

[Reasons Kubernetes is cool](https://jvns.ca/blog/2017/10/05/reasons-kubernetes-is-cool/)Because all the components don’t keep any state in memory(stateless), you can just restart them at any time and that can help mitigate a variety of bugs.The only stateful thing you have to operate is etcd

k8s 在 etcd中的存在

```go
/registry/minions
/registry/minions/192.168.56.102    # 列出该节点的信息，包括其cpu和memory能力
/registry/minions/192.168.56.103
/registry/controllers
/registry/controllers/default
/registry/controllers/default/apache2-controller	# 跟创建该controller时信息大致相同，分为desireState和currentState
/registry/controllers/default/heapster-controller
/registry/pods
/registry/pods/default
/registry/pods/default/128e1719-c726-11e4-91cd-08002782f91d   	# 跟创建该pod时信息大致相同，分为desireState和currentState
/registry/pods/default/128e7391-c726-11e4-91cd-08002782f91d
/registry/pods/default/f111c8f7-c726-11e4-91cd-08002782f91d
/registry/nodes
/registry/nodes/192.168.56.102
/registry/nodes/192.168.56.102/boundpods	# 列出在该主机上运行pod的信息，镜像名，可以使用的环境变量之类，这个可能随着pod的迁移而改变
/registry/nodes/192.168.56.103
/registry/nodes/192.168.56.103/boundpods
/registry/events
/registry/events/default
/registry/events/default/704d54bf-c707-11e4-91cd-08002782f91d.13ca18d9af8857a8		# 记录操作，比如将某个pod部署到了某个node上
/registry/events/default/f1ff6226-c6db-11e4-91cd-08002782f91d.13ca07dc57711845
/registry/services
/registry/services/specs
/registry/services/specs/default
/registry/services/specs/default/monitoring-grafana		#  基本跟创建信息大致一致，但包含serviceip
/registry/services/specs/default/kubernetes
/registry/services/specs/default/kubernetes-ro
/registry/services/specs/default/monitoring-influxdb
/registry/services/endpoints
/registry/services/endpoints/default
/registry/services/endpoints/default/monitoring-grafana	  	# 终端（traffic在这里被处理），和某一个serviceId相同，包含了service对应的几个pod的ip，这个可能经常变。
/registry/services/endpoints/default/kubernetes
/registry/services/endpoints/default/kubernetes-ro
/registry/services/endpoints/default/monitoring-influxdb
```
## create pod

![](/public/upload/kubernetes/apiserver_create_pod.png)

