使用Slime的Http插件管理功能需在ConfigMap中开启Plugin模块：
```json
{
  "plugin": {
    "enable": true
  }
  //...
}
```
### 内建插件
**注意:** envoy的二进制需支持扩展插件

**打开/停用**   

按如下格式配置PluginManager，即可打开内建插件:
```yaml
apiVersion: microservice.netease.com/v1alpha1
kind: PluginManager
metadata:
  name: my-plugin
  namespace: default
spec:
  workload_labels:
    app: my-app
  plugins:
  - enable: true          # 插件开关
    name: {plugin-1}
  # ...
  - enable: true
    name: {plugin-N}
```
其中，{plugin-N}为插件名称，PluginManager中的排序为插件执行顺序。
将enable字段设置为false即可停用插件。

**全局配置**

全局配置对应LDS中的插件配置，按如下格式设置全局配置:
```yaml
apiVersion: microservice.netease.com/v1alpha1
kind: PluginManager
metadata:
  name: my-plugin
  namespace: default
spec:
  workload_labels:
    app: my-app
  plugins:
  - enable: true          # 插件开关
    name: {plugin-1}      # 插件名称
    inline:
      settings:
        {plugin settings} # 插件配置
  # ...
  - enable: true
    name: {plugin-N}
```

**Host/路由级别配置**

按如下格式配置EnvoyPlugin:
```yaml
apiVersion: microservice.netease.com/v1alpha1
kind: EnvoyPlugin
metadata:
  name: project1-abc
  namespace: gateway-system
spec:
  workload_labels:
    app: my-app
  host:                          # 插件的生效范围(host级别)              
  - jmeter.com
  - istio.com
  - 989.mock.qa.netease.com
  - demo.test.com
  - netease.com
  route:                         # 插件的生效范围(route级别), route字段须对应VirtualService中的名称
  - abc
  plugins:
  - name: com.netease.supercache # 插件名称
    settings:                    # 插件配置
      cache_ttls:
        LocalHttpCache:
          default: 60000
      enable_rpx:
        headers:
        - name: :status
          regex_match: 200|
      key_maker:
        exclude_host: false
        ignore_case: true
      low_level_fill: true
```
### 扩展wasm插件

// TODO

### 示例

// TODO

---