---
layout: doc
---

## ServiceFence

```protobuf
/*
* @Author: yangdihang
* @Date: 2020/5/21
*/

syntax = "proto3";
package netease.microservice.v1alpha1;

option go_package = "yun.netease.com/slime/pkg/apis/microservice/v1alpha1";

// ServiceFence是在社区Sidecar资源之上的一层封装，其主要作用与Sidecar资源相同，可以隔绝服务
// 所不关心的配置，从而提升大规模场景下服务网格的性能。与Sidecar资源不同的是，ServiceFence
// 会根据VirtualService资源的变化调整围栏策略。
// 考虑如下场景：
// apiVersion: networking.istio.io/v1alpha3
// kind: VirtualService
// metadata:
//   # name和namespace需要和所属服务一一对应
//   name: a
//   namespace: test1
// spec:
//   host:
//   - b.test1.svc.cluster.local
//   http:
//   - route:
//     - destination:
//         host: c.test1.svc.cluster.local
//
// 假如此时，Sidecar资源只考虑了a-->b的服务调用，那么如上的路由配置将会出现503,原因是c对于a而言不可达，
// 对于使用者而言配置路由规则时还要考虑Sidecar资源的配置，这显然不合适。ServiceFence会根据
// VirtualService对应的修改Sidecar配置
// 例如：
// apiVersion: microservice.netease.com/v1alpha1
// kind: ServiceFence
// metadata:
//   # name和namespace需要和所属服务一一对应
//   name: a
//   namespace: test1
// spec:
//   host:
//     b.test1.svc.cluster.local:
//       stable: {}
//
// 该配置经过operator计算后，其状态会改变为：
// apiVersion: microservice.netease.com/v1alpha1
// kind: ServiceFence
// metadata:
//   # name和namespace需要和所属服务一一对应
//   name: a
//   namespace: test1
// spec:
//   host:
//     b.test1.svc.cluster.local:
//       stable: {}
// status:
//   domains:
//     b.test1.svc.cluster.local:
//       hosts:
//       - b.test1.svc.cluster.local
//       - c.test1.svc.cluster.local
//       status: ACTIVE
//
// 也可以利用ServiceFence管理隔离配置的生命周期，ServiceFence有三种记录策略：
// 1. stable，稳定的配置，用户手动回收配置
// 2. deadline， 到期回收
// 3. auto， 根据服务掉用情况自动回收

message Timestamp {

    // Represents seconds of UTC time since Unix epoch
    // 1970-01-01T00:00:00Z. Must be from 0001-01-01T00:00:00Z to
    // 9999-12-31T23:59:59Z inclusive.
    int64 seconds = 1;

    // Non-negative fractions of a second at nanosecond resolution. Negative
    // second values with fractions must still have non-negative nanos values
    // that count forward in time. Must be from 0 to 999,999,999
    // inclusive.
    int32 nanos = 2;
}

message ServiceFenceSpec {

    // 访问的服务以及其回收策略
    map<string, RecyclingStrategy> host = 1;
    bool enable = 2;
}
message RecyclingStrategy {

    message Stable {
    }

    message Deadline {
        Timestamp expire = 1;
    }

    message Auto {
        Timestamp duration = 1;
    }
    // 永久存在的配置
    Stable stable = 1;

    // 到期失效
    Deadline deadline = 2;

    // 当最近一次访问时间大于duration时，将其状态改变为失效
    Auto auto = 3;

    Timestamp RecentlyCalled = 4;
}


message Destinations {

    // 记录最近一次调用，当失效策略为auto时，需定期刷新该值
    Timestamp RecentlyCalled = 1;

    // domain对应的上游服务
    repeated string hosts = 2;

    enum Status {
        ACTIVE = 0;
        EXPIRE = 1;
    }
    Status status = 3;
}

message ServiceFenceStatus {
    map<string, Destinations> domains = 1;
    map<string, bool> visitor = 2;
}
```

---

**PluginManager:**

```protobuf
/*
* @Author: yangdihang
* @Date: 2020/5/21
*/

syntax = "proto3";

// PluginManager 用于管理插件，以及插件在全局维度上的配置，enable选项可以启用或禁用某插件
// 示例如下：
// apiVersion: microservice.netease.com/v1alpha1
// kind: PluginManager
// metadata:
//   name: gw-cluster-gateway-proxy
//   namespace: gateway-system
// spec:
//   plugin:
//   - enable: true
//     name: envoy.cors
//   - enable: true
//     name: envoy.fault
//   - enable: true
//     name: com.netease.iprestriction
//   - enable: true
//     name: com.netease.transformation
//   - enable: false
//     name: com.netease.resty
//   - enable: true
//     name: com.netease.metadatahub
//   - enable: true
//     name: com.netease.supercache
//     inline:
//       settings:
//         apis_prefix: testremove
//         used_caches:
//           local: {}
//           redis:
//             general:
//               host: 10.107.16.138
//               port: 6379
//             timeout: 100
//   workloadLabels:
//     gw_cluster: gateway-proxy

import "google/protobuf/struct.proto";

package netease.microservice.v1alpha1;

option go_package = "yun.netease.com/slime/pkg/apis/microservice/v1alpha1";

message PluginManager {
    // Zero or more labels that indicate a specific set of pods/VMs whose
    // proxies should be configured to use these additional filters.  The
    // scope of label search is platform dependent. On Kubernetes, for
    // example, the scope includes pods running in all reachable
    // namespaces. Omitting the selector applies the filter to all proxies in
    // the mesh.
    map<string, string> workload_labels = 1;

    repeated Plugin plugin = 2;

    // Names of gateways where the rule should be applied to. Gateway names
    // at the top of the VirtualService (if any) are overridden. The gateway
    // match is independent of sourceLabels.
    repeated string gateways = 3;
}

message Plugin {
    bool enable = 1;
    string name = 2;
    google.protobuf.Struct settings = 3;
    enum ListenerType {
        Outbound = 0;
        Inbound = 1;
    }
    ListenerType listenerType = 4;
    string type_url = 5;
    oneof plugin_settings {
        Wasm wasm = 6;
        Inline inline = 7;
    }
}

message Wasm {
    string rootID = 1;
    string fileName = 2;
    google.protobuf.Struct settings = 3;
}

message Inline {
    google.protobuf.Struct settings = 1;
}
```

---

## EnvoyPlugin

```protobuf
/*
* @Author: yangdihang
* @Date: 2020/5/21
*/

syntax = "proto3";

// EnvoyPlugin 用于设置网关扩展插件配置，配置生效范围有3个级别
// route级别，对应同一route name下的路由规则
// host级别，对应同一host下的路由规则

// 示例如下：
// apiVersion: microservice.netease.com/v1alpha1
// kind: EnvoyPlugin
// metadata:
//   name: project1-3-458-rewrite
// spec:
//   gateway:
//   - gateway-proxy-1
//   host:
//   - 103.196.65.178
//   plugins:
//   - name: com.netease.rewrite
//     settings:
//       request_transformations:
//       - conditions:
//         - headers:
//           - name: :path
//             regex_match: /aaaaaaa/(.*)
//         transformation_template:
//           extractors:
//             "userpath":
//               header: :path
//               regex: /aaaaaaa/(.*)
//               subgroup: 1
//           headers:
//             :path:
//               text: "/{{userpath}}"
//           parse_body_behavior: DontParse
import "pkg/apis/microservice/v1alpha1/plugin_manager.proto";

package netease.microservice.v1alpha1;

option go_package = "yun.netease.com/slime/pkg/apis/microservice/v1alpha1";

message WorkloadSelector {
    // One or more labels that indicate a specific set of pods/VMs
    // on which the configuration should be applied. The scope of
    // label search is restricted to the configuration namespace in which the
    // the resource is present.
    map<string, string> labels = 1;
}

message EnvoyPlugin {
    map<string, string> workload_labels = 9;

    // route level plugin
    repeated string route = 1;

    // host level plugin
    repeated string host = 2;

    // service level plugin
    repeated string service = 3;

    repeated Plugin plugins = 4;

    // which gateway should use this plugin setting
    repeated string gateway = 5;

    // which user should use this plugin setting
    repeated string user = 6;

    // group setting 用于路由组级别的配置设置，其优先级低于路由级别的配置
    bool isGroupSetting = 7;

    message Listener {
        uint32 port = 1;
        bool outbound = 2;
    }

    // listener level
    repeated Listener listener = 8;
}
```

---

## SmartLimiter

```protobuf
/*
* @Author: yangdihang
* @Date: 2020/5/21
*/

syntax = "proto3";
import "google/protobuf/duration.proto";
package netease.microservice.v1alpha1;

option go_package = "yun.netease.com/slime/pkg/apis/microservice/v1alpha1";

// SmartLimiter 用于配置自适应限流插件，以及插件在全局维度上的配置，enable选项可以启用或禁用某插件
// 示例如下：
// apiVersion: microservice.netease.com/v1alpha1
// kind: SmartLimiter
// metadata:
//   name: a
//   namespace: powerful
// spec:
//   descriptors:
//   - action:
//       fill_interval:
//         seconds: 60
//       quota: "30/{pod}"
//     condition: "true"

message Action {
    string quota = 1;
    google.protobuf.Duration fill_interval = 2;
}

message ActionStatus {
    uint32 quota = 1;
    google.protobuf.Duration fill_interval = 2;
}

message SmartLimitDescriptor {
    string condition = 2;
    Action action = 3;
    repeated HeaderMatcher match = 4;
}

message SmartLimitDescriptorStatus {
    string condition = 2;
    ActionStatus action = 3;
    repeated HeaderMatcher match = 4;
}

message SmartLimiterSpec {
    string domain = 1;
    repeated SmartLimitDescriptor descriptors = 2;
}

message SmartLimiterStatus {
    repeated SmartLimitDescriptor ratelimitStatus = 1;
    map<string, string> endPointStatus = 2;
    message ServiceStatus {
        map<string, string> selector = 1;
        message Listener {
            string name = 1;
            uint32 port = 2;
        }
        repeated Listener listener = 2;
    }
    ServiceStatus serviceStatus = 3;
}

message HeaderMatcher {
    string name = 1;

    string exact_match = 2;
    string regex_match = 3;
    bool present_match = 7;
    string prefix_match = 9;
    string suffix_match = 10;

    bool invert_match = 8;
}
```

