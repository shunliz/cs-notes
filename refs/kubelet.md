作者：packy
链接：https://zhuanlan.zhihu.com/p/106544069
来源：知乎
著作权归作者所有。商业转载请联系作者获得授权，非商业转载请注明出处。



在Kubernetes集群中，每个Node节点（又称Minion）上都会启动一个Kubelet服务进行。该进程用于处理Master节点下发到本节点的任务，管理Pod及Pod中的容器。每个Kubelet进程会在API Server上注册节点自身信息，定期向Master节点汇报节点资源的使用情况，并通过cAdvise监控容器和节点资源。

> 此源码分析基于 K8s 1.14.6

## Kubelet 架构

若想解读 Kubelet 源码，首先我们需要了解 Kubelet 的主要架构及组件。

![img](https://pic3.zhimg.com/v2-c35989d94166425edb0c8ad5c6ac7a7f_720w.jpg?source=d16d100b)![](https://pic3.zhimg.com/80/v2-c35989d94166425edb0c8ad5c6ac7a7f_720w.jpg?source=d16d100b)

## 启动入口

kubelet 的主函数入口在 `cmd/kubelet/kubelet.go`中，启动代码很简洁：

```text
func main() {
    rand.Seed(time.Now().UnixNano())
    // 这里则是启动 kubelet 服务进程，返回一个*cobra.Command
    command := app.NewKubeletCommand(server.SetupSignalHandler())
    logs.InitLogs()
    defer logs.FlushLogs()

    if err := command.Execute(); err != nil {
        fmt.Fprintf(os.Stderr, "%v\n", err)
        os.Exit(1)
    }
}
```

`app.NewKubeletCommand` 函数则是对 kubelet 进行参数化配置并启动 kubelet，然后生成kubelet对象以及该kubelet维护pod所需要用到的服务。主要步骤如下

1. 解析参数，加载当前flag，对参数的合法性进行判断。flag 包含两种：

- kubeletFlags：kubelet -- 后追加的参数 
- kubeletConfig： 通过解析特定配置文件获取 

\2. 构造 cobra.Command 对象，此对象用于执行用户输入的命令行交互。此对象结构体为 

```text
    {
        Use: componentKubelet,
        Long: ...,
        // The Kubelet has special flag parsing requirements to enforce flag precedence rules,
        // so we do all our parsing manually in Run, below.
        // DisableFlagParsing=true provides the full set of flags passed to the kubelet in the
        // `args` arg to Run, without Cobra's interference.
        DisableFlagParsing: true,
        Run: func(cmd *cobra.Command, args []string) {...
        },
    }
```

其中的 `Run` 则是用于具体执行用户命令的函数，这个函数的流程也就是 kubelet的主流程，创建 kubelet 对象，创建各种服务。

## 启动流程

这一部分解析一下 Run 函数。首先我们需要了解两个比较重要的配置结构体`KubeletFlags` `kubeletconfig.KubeletConfiguration`：

启动流程如下：

1. 新建一个watch的功能，主要是用来watch kubelet的配置文件是否改变，如果已经改变，那么就重新load kubelet的配置文件 用的是kubernetes常用到的Controller,也就是Informer的架构，watch ConfigMap对象 

> controller 相关代码在 pkg/kubelet/kubeletconfig/controller.go 中 *//TODO 介绍下 controller informer 机制*

\2. 构造 Kubelet server对象，
kubelet server 对象由 kubelet 各种配置组成，其结构体如下： 

```go
    type KubeletServer struct {
    KubeletFlags
    kubeletconfig.KubeletConfiguration
    }
```

根据 server 创建 `kubeletDeps`对象，它并不会启动任何进程，仅返回适合运行的依赖项或错误。这个是比较重要的部分，其结构体为：

```text
   kubelet.Dependencies{
        Auth:                nil, // default does not enforce auth[nz]
        CAdvisorInterface:   nil, // cadvisor.New launches background processes (bg http.ListenAndServe, and some bg cleaners), not set here
        Cloud:               nil, // cloud provider might start background processes
        ContainerManager:    nil,
        DockerClientConfig:  dockerClientConfig,
        KubeClient:          nil,
        HeartbeatClient:     nil,
        EventClient:         nil,
        Mounter:             mounter,
        Subpather:           subpather,
        OOMAdjuster:         oom.NewOOMAdjuster(),
        OSInterface:         kubecontainer.RealOS{},
        VolumePlugins:       ProbeVolumePlugins(),
        DynamicPluginProber: GetDynamicPluginProber(s.VolumePluginDir, pluginRunner),
        TLSOptions:          tlsOptions}, nil
```

我们可以看到，返回的这些包含底层docker 客户端、挂载器等。其实就是可以供 server 使用的工具包。

\3. 执行 Run 函数，启动 kubelet。注意此 Run 函数并非上文的 Command 中的 Run 

```go
   func Run(s *options.KubeletServer, kubeDeps *kubelet.Dependencies, stopCh <-chan struct{}) error {
        // To help debugging, immediately log version
        klog.Infof("Version: %+v", version.Get())
        // 这里是针对 windows 系统的初始化操作
        if err := initForOS(s.KubeletFlags.WindowsService); err !=  nil {
            return fmt.Errorf("failed OS init: %v", err)
        }
        // 这里启动 kubelet
        if err := run(s, kubeDeps, stopCh); err != nil {
            return fmt.Errorf("failed to run Kubelet: %v", err)
        }
        return nil
    }
```


调用的 `func run` 将会执行以下一系列操作：

1. 通过`SetFromMap`设置 kubelet 的 feature Gate 
2. 验证初始化的 server 
3. 注册端点 `/configz` 
4. 获取并配置各种客户端，包括： 

- kubeclient 
- 事件 client：
  配置 EventRecordQPS EventBrust 参数
  调用 `k8s.io/client-go/kubernetes/typed/core/v1`中的 `NewForConfig` 
- 心跳客户端：
  配置 `QPS` `Timeout`(若开启了`NodeLease` feature，则设置 `NodeLeaseDurationSeconds` 为timeout) 

\5. 构建认证器 `AuthInterface`，调用`BuildAuth` 

```go
    func BuildAuth(nodeName types.NodeName, client clientset.Interface, config kubeletconfig.KubeletConfiguration) (server.AuthInterface, error) {
        // Get clients, if provided
        var (
            tokenClient authenticationclient.TokenReviewInterface
            sarClient   authorizationclient.SubjectAccessReviewInterface
        )
        if client != nil && !reflect.ValueOf(client).IsNil() {
            tokenClient = client.AuthenticationV1beta1().TokenReviews()
            sarClient = client.AuthorizationV1beta1().SubjectAccessReviews()
        }
    
        authenticator, err := BuildAuthn(tokenClient, config.Authentication)
        if err != nil {
            return nil, err
        }
    
        attributes := server.NewNodeAuthorizerAttributesGetter(nodeName)
    
        authorizer, err := BuildAuthz(sarClient, config.Authorization)
        if err != nil {
            return nil, err
        }
    
        return server.NewKubeletAuth(authenticator, attributes, authorizer), nil
    }    
```

\6. 构建`CadvisorInterface`，主要用于监控功能，包括信息有 

```go
    type Interface interface {
        Start() error
        DockerContainer(name string, req *cadvisorapi.ContainerInfoRequest)     (cadvisorapi.ContainerInfo, error)
        ContainerInfo(name string, req *cadvisorapi.ContainerInfoRequest)     (*cadvisorapi.ContainerInfo, error)
        ContainerInfoV2(name string, options cadvisorapiv2.RequestOptions) (map[string]    cadvisorapiv2.ContainerInfo, error)
        SubcontainerInfo(name string, req *cadvisorapi.ContainerInfoRequest) (map[string]    *cadvisorapi.ContainerInfo, error)
        MachineInfo() (*cadvisorapi.MachineInfo, error)

        VersionInfo() (*cadvisorapi.VersionInfo, error)

        // Returns usage information about the filesystem holding container images.
        ImagesFsInfo() (cadvisorapiv2.FsInfo, error)

        // Returns usage information about the root filesystem.
        RootFsInfo() (cadvisorapiv2.FsInfo, error)

        // Get events streamed through passedChannel that fit the request.
        WatchEvents(request *events.Request) (*events.EventChannel, error)

        // Get filesystem information for the filesystem that contains the given file.
        GetDirFsInfo(path string) (cadvisorapiv2.FsInfo, error)
    }
```

\7. 初始化 `ContainerManager`，用于管理运行于节点上的 container，相关组件有： 

```text
    kubeReserved 包含节点的相关资源，包括 cpu memory pid数量等
    SystemReserved 节点资源，支持 cpu memory
    experimentalQOSReserved 即--qos-reserve-requests参数
    devicePluginEnabled（DevicePlugins feature Gate）
```

最终构建的结构体为：

```go
    kubeDeps.ContainerManager, err = cm.NewContainerManager(
            kubeDeps.Mounter,
            kubeDeps.CAdvisorInterface,
            cm.NodeConfig{
                RuntimeCgroupsName:    s.RuntimeCgroups,
                SystemCgroupsName:     s.SystemCgroups,
                KubeletCgroupsName:    s.KubeletCgroups,
                ContainerRuntime:      s.ContainerRuntime,
                CgroupsPerQOS:         s.CgroupsPerQOS,
                CgroupRoot:            s.CgroupRoot,
                CgroupDriver:          s.CgroupDriver,
                KubeletRootDir:        s.RootDirectory,
                ProtectKernelDefaults: s.ProtectKernelDefaults,
                NodeAllocatableConfig: cm.NodeAllocatableConfig{
                    KubeReservedCgroupName:   s.KubeReservedCgroup,
                    SystemReservedCgroupName: s.SystemReservedCgroup,
                    EnforceNodeAllocatable:   sets.NewString(s.EnforceNodeAllocatable...),
                    KubeReserved:             kubeReserved,
                    SystemReserved:           systemReserved,
                    HardEvictionThresholds:   hardEvictionThresholds,
                },
                QOSReserved:                           *experimentalQOSReserved,
                ExperimentalCPUManagerPolicy:          s.CPUManagerPolicy,
                ExperimentalCPUManagerReconcilePeriod: s.CPUManagerReconcilePeriod.Duration,
                ExperimentalPodPidsLimit:              s.PodPidsLimit,
                EnforceCPULimits:                      s.CPUCFSQuota,
                CPUCFSQuotaPeriod:                     s.CPUCFSQuotaPeriod.Duration,
            },
            s.FailSwapOn,
            devicePluginEnabled,
            kubeDeps.Recorder)
```

\9. 进行权限验证，需要以 uid 为 0 的用户启动。 

\10. 调用 `RunKubelet` 启动 kubelet 进程 

### 调用 RunKubelet

RunKubelet 主要流程：

1. 获取主机名 
2. 建立并初始化 event recorder 
3. 获取以下资源(均读取自 `KubeletFlags`)： 

- hostNetworkSources 主要指Kubelet 允许 pod中的某些资源（包括 file，http，api，*） 使用 hostnetwork 
- hostPIDSources 指 Kubelet 允许使用host pid命名空间的Pod源列表。 
- hostIPCSources 指 Kubelet 允许使用 host IPC 资源的 pod 源列表 
- privilegedSources 由以上三个资源配置构成 

\4. 若设置 `runonce` 参数，则只拉取一次容器组配置，并在启动容器组后退出，否则将以 server 形式保持 

- 对于runonce，首先创建所需的目录，监听 pod update 信息，得到 pod 信息后，创建pod 并返回他们的状态 
- 以 server 模式启动，调用`startKubelet`,流程如下： 

\5. 检查 logserver 以及 apiserver 是否可用 

1. 1. 如有 cloud provider 配置， 则启动`cloudResourceSyncManager`，将请求发送给 cloud provider 
   2. 启动 volumeManager，VolumeManager运行一组异步循环，这些循环根据在此节点上调度的Pod来确定需要附加/装入/卸载/分离的卷。 
   3. 调用 `kubelet.syncNodeStatus`同步 node 状态，如果从上次同步起有任何更改 或 经过了足够的时间，它将节点状态同步到主节点，并在必要时先注册kubelet。 
   4. 调用 `kubelet.updateRuntimeUp`，updateRuntimeUp调用容器运行时状态回调，在容器运行时首次出现时初始化依赖于运行时的模块，如果状态检查失败，则返回错误。 如果状态检查确定，则在kubelet runtimeState中更新容器运行时正常运行时间。 
   5. 开启循环，同步 iptables 规则（*但是在源码中无任何操作*） 
   6. 启动一个用于“杀死pod” 的 goroutine，如果尚未使用其他goroutine，则podKiller会从通道(`podKillingCh`)中接收到一个pod，然后启动goroutine杀死他 
   7. 启动 statusManager 和 probeManager （都是无限循环的同步机制），statusManager 与 apiserver 同步pod状态；probeManager 管理并接收 container 探针。 
   8. 启动 runtimeClass manager （**注意这里与替换底层容器相关**） 
       runtimeClass 是 K8s 的一个 api 对象，可以通过定义 runtimeClass 实现 K8s 对接不同的 容器运行时。
   9. 启动 pleg （pod lifecycle event generator），用于生成 pod 相关的 event。 

至此，kubelet 整体的启动流程完毕，进入无限循环中，实时同步不同组件的状态。同时也对端口进行监听，响应 http 请求。