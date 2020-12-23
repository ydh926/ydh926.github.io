**注意:** 自适应限流功能可以对接envoy社区支持的限流插件`envoy.filters.http.local_ratelimit`，也可以对接网易自研插件`com.netease.local_flow_control`。envoy社区的限流插件暂不支持HeaderMatch的配置，使用`com.netease.local_flow_control`插件前需确认envoy二进制中是否包含该插件。      

### 用法介绍

使用Slime的自适应限流功能需在ConfigMap中开启Ratlimit模块：
```json
{
  "ratelimit": {
    "enable": true
  }
  //...
}
```
使用限流功能非常简单，只需为目标服务定义通么的SmartLimite资源即可，如下所示：

```yaml
apiVersion: microservice.netease.com/v1alpha1
kind: SmartLimiter
metadata:
  name: test-svc
  namespace: default
spec:
  descriptors:
  - action:
      quota: "3"
      unit: 1
    condition: "true"

```

这份配置可以限制带有`user=Bob`的头的请求对test-svc服务中实例的访问，限流次数为每秒3次。将配置提交之后，该服务下实例的状态信息以及限流信息会显示在`status`中，如下。

```yaml
apiVersion: microservice.netease.com/v1alpha1
kind: SmartLimiter
metadata:
  name: test-svc
  namespace: default
spec:
  descriptors:
  - action:
      quota: "3"
      unit: 1
    condition: "true"
    match:
    - exact_match: user
      invert_match: false
      name: Bob
status:
  endPointStatus:
    cpu: "398293"        # 业务容器和sidecar容器占用CPU之和 
    cpu_max: "286793"    # CPU占用最大的容器
    memory: "68022"      # 业务容器和sidecar容器内存占用之和  
    memory_max: "55236"  # 内存占用最大的容器
    pod: "1"
  ratelimitStatus:
  - action:
      fill_interval:
        seconds: 1
      quota: "3"
```

### 基于监控的自适应限流

获得监控信息之后，可以将监控信息条目配置到`condition`，例如希望cpu超过300m时触发限流，可以进行如下配置：

```yaml
apiVersion: microservice.netease.com/v1alpha1
kind: SmartLimiter
metadata:
  name: test-svc
  namespace: default
spec:
  descriptors:
  - action:
      quota: "3"
      unit: 1
    condition: {cpu}>300000 # cpu的单位为ns，首先会根据endPointStatus中cpu的值将算式渲染为398293>300000
    match:
    - exact_match: user
      invert_match: false
      name: Bob
status:
  endPointStatus:
    cpu: "398293"        # 业务容器和sidecar容器占用CPU之和 
    cpu_max: "286793"    # CPU占用最大的容器
    memory: "68022"      # 业务容器和sidecar容器内存占用之和  
    memory_max: "55236"  # 内存占用最大的容器
    pod: "1"
  ratelimitStatus:
  - action:
      fill_interval:
        seconds: 1
      quota: "3"
```

condition中的算式会根据endPointStatus的条目进行渲染，渲染后的算式若计算结果为true，则会触发限流。

---

### 服务限流

由于缺乏全局配额管理组件，我们无法做到精确的服务限流，但是在假定负载均衡理想的情况下，实例限流数=服务限流数/实例个数。假定test-svc的服务限流数为3，那么可以将quota字段配置为3/{pod}，在服务发生扩容时，可以在限流状态栏中看到实例限流数的变化。

```yaml
apiVersion: microservice.netease.com/v1alpha1
kind: SmartLimiter
metadata:
  name: test-svc
  namespace: default
spec:
  descriptors:
  - action:
      quota: "3/{pod}" # 算式会根据endPointStatus中pod值渲染为3/3
      unit: 1
    condition: {cpu}>300000 
    match:
    - exact_match: user
      invert_match: false
      name: Bob
status:
  endPointStatus:
    cpu: "xxxxx"        
    cpu_max: "xxxx"    
    memory: "xxx"       
    memory_max: "xx" 
    pod: "3" # test-svc的endpoint扩容成了3
  ratelimitStatus:
  - action:
      fill_interval:
        seconds: 1
      quota: "1" 显然，3/3=1
```