## 符号约定

关键词“MUST”（必须）、“MUST NOT”（禁止）、“REQUIRED”（必填）、“SHALL”（应）、“SHALL NOT”（不应）、“SHOULD”（建议）、“SHOULD NOT”（不建议）、“RECOMMENDED”（推荐）、“NOT RECOMMENDED”（不推荐）和“MAY”（可以）的解释遵循 [RFC 2119](http://tools.ietf.org/html/rfc2119)（Bradner, S.，《RFC中用于指示需求级别的关键词》，BCP 14，RFC 2119，1997年3月）。

关键词“unspecified”（未指定）、“undefined”（未定义）和“implementation-defined”（实现定义）的解释遵循 [C99标准的基本原理](http://www.open-std.org/jtc1/sc22/wg14/www/C99RationaleV5.10.pdf#page=18)。

若某实现未满足其所要实现的协议中任何一项“MUST”“REQUIRED”或“SHALL”要求，则该实现不具合规性。
若某实现满足其所要实现的协议中所有“MUST”“REQUIRED”和“SHALL”要求，则该实现具备合规性。


## 术语

| 术语 | 定义 |
|-------------------|--------------------------------------------------|
| Volume（卷） | 通过CSI在共同管理的容器内可用的存储单元。 |
| Block Volume（块卷） | 在容器内显示为块设备的卷。 |
| Mounted Volume（已挂载卷） | 使用指定文件系统挂载并在容器内显示为目录的卷。 |
| CO | 容器编排系统，通过CSI服务RPC与插件通信。 |
| SP | 存储提供商，即CSI插件实现的供应商。 |
| RPC | [远程过程调用](https://en.wikipedia.org/wiki/Remote_procedure_call)。 |
| Node（节点） | 用户工作负载运行的主机，从插件的角度可通过节点ID唯一标识。 |
| Plugin（插件） | 即“插件实现”，是实现CSI服务的gRPC端点。 |
| Plugin Supervisor（插件管理器） | 管理插件生命周期的进程，可为CO。 |
| Workload（工作负载） | 由CO调度的“工作”原子单元。可以是一个容器或一组容器。 |


## 目标

定义行业标准的“容器存储接口”（CSI），使存储供应商（SP）能够开发一次插件，即可在多个容器编排（CO）系统中使用。


### MVP 目标

容器存储接口（CSI）将：

* 使SP开发者能够编写一个符合CSI标准的插件，该插件能在所有实现CSI的CO中“正常工作”。
* 定义支持以下功能的API（RPCs）：
  * 卷的动态创建和删除。
  * 卷与节点的连接或断开连接。
  * 卷在节点上的挂载或卸载。
  * 块卷和可挂载卷的使用。
  * 本地存储提供商（如device mapper、lvm）。
  * 快照的创建和删除（快照的源是一个卷）。
  * 从快照创建新卷（从快照还原，即擦除原始卷中的数据并替换为快照中的数据，不在此范围内）。
* 定义插件协议建议：
  * 描述管理器配置插件的过程。
  * 容器部署注意事项（`CAP_SYS_ADMIN`、挂载命名空间等）。


### MVP 非目标

容器存储接口（CSI）明确不定义、不提供或不规定：

* 插件管理器管理插件生命周期的特定机制，包括：
  * 如何维护状态（如已连接、已挂载等）。
  * 如何部署、安装、升级、卸载、监控插件，或在插件意外终止时重启。
* 表示“存储等级”（又名“存储类”）的一级消息结构/字段。
* 协议级别的认证和授权。
* 插件的打包方式。
* POSIX合规性：CSI不保证所提供的卷是符合POSIX标准的文件系统。合规性由插件实现（及其所依赖的任何后端存储系统）决定。CSI不应阻碍插件管理器或CO以符合POSIX标准的方式与插件管理的卷进行交互。

### 架构

本规范的主要关注点是容器编排系统（CO）与插件（Plugin）之间的**协议**。

应确保能够为各种部署架构提供跨CO兼容的插件。CO应能够处理集中式插件和无头（headless）插件，以及拆分组件式插件和统一组件式插件。以下图示展示了其中几种可能的架构。

```
                             CO "主" 主机
+-------------------------------------------+
|                                           |
|  +------------+           +------------+  |
|  |    CO      |   gRPC    | 控制器插件  |  |
|  |            +----------->            |  |
|  +------------+           +------------+  |
|                                           |
+-------------------------------------------+

图1：插件在集群的所有节点上运行：CO主主机上有一个集中式的控制器插件，所有CO节点上均有节点插件。
```

```
                            CO "节点" 主机(s)
+-------------------------------------------+
|                                           |
|  +------------+           +------------+  |
|  |    CO      |   gRPC    | 控制器插件  |  |
|  |            +--+-------->            |  |
|  +------------+  |        +------------+  |
|                  |                        |
|                  |                        |
|                  |        +------------+  |
|                  |        |  节点插件   |  |
|                  +-------->            |  |
|                           +------------+  |
|                                           |
+-------------------------------------------+

图2：无头插件部署，仅CO节点主机运行插件。独立的拆分组件式插件分别提供控制器服务和节点服务。
```

```
                            CO "节点" 主机(s)
+-------------------------------------------+
|                                           |
|  +------------+           +------------+  |
|  |    CO      |   gRPC    | 控制器      |  |
|  |            +----------->  节点插件   |  |
|  +------------+           |            |  |
|                           +------------+  |
|                                           |
+-------------------------------------------+

图3：无头插件部署，仅CO节点主机运行插件。一个统一的插件组件同时提供控制器服务和节点服务。
```

```
                            CO "节点" 主机(s)
+-------------------------------------------+
|                                           |
|  +------------+           +------------+  |
|  |    CO      |   gRPC    |  节点插件   |  |
|  |            +----------->            |  |
|  +------------+           +------------+  |
|                                           |
+-------------------------------------------+

图4：无头插件部署，仅CO节点主机运行插件。一个仅含节点功能的插件组件只提供节点服务。其GetPluginCapabilities RPC不会报告CONTROLLER_SERVICE（控制器服务）能力。
```

### Volume Lifecycle

```
   CreateVolume +------------+ DeleteVolume
 +------------->|  CREATED   +--------------+
 |              +---+----^---+              |
 |       Controller |    | Controller       v
+++         Publish |    | Unpublish       +++
|X|          Volume |    | Volume          | |
+-+             +---v----+---+             +-+
                | NODE_READY |
                +---+----^---+
               Node |    | Node
            Publish |    | Unpublish
             Volume |    | Volume
                +---v----+---+
                | PUBLISHED  |
                +------------+

Figure 5: The lifecycle of a dynamically provisioned volume, from
creation to destruction.
```

```
   CreateVolume +------------+ DeleteVolume
 +------------->|  CREATED   +--------------+
 |              +---+----^---+              |
 |       Controller |    | Controller       v
+++         Publish |    | Unpublish       +++
|X|          Volume |    | Volume          | |
+-+             +---v----+---+             +-+
                | NODE_READY |
                +---+----^---+
               Node |    | Node
              Stage |    | Unstage
             Volume |    | Volume
                +---v----+---+
                |  VOL_READY |
                +---+----^---+
               Node |    | Node
            Publish |    | Unpublish
             Volume |    | Volume
                +---v----+---+
                | PUBLISHED  |
                +------------+

Figure 6: The lifecycle of a dynamically provisioned volume, from
creation to destruction, when the Node Plugin advertises the
STAGE_UNSTAGE_VOLUME capability.
```

```
    Controller                  Controller
       Publish                  Unpublish
        Volume  +------------+  Volume
 +------------->+ NODE_READY +--------------+
 |              +---+----^---+              |
 |             Node |    | Node             v
+++         Publish |    | Unpublish       +++
|X| <-+      Volume |    | Volume          | |
+++   |         +---v----+---+             +-+
 |    |         | PUBLISHED  |
 |    |         +------------+
 +----+
   Validate
   Volume
   Capabilities

Figure 7: The lifecycle of a pre-provisioned volume that requires
controller to publish to a node (`ControllerPublishVolume`) prior to
publishing on the node (`NodePublishVolume`).
```

```
       +-+  +-+
       |X|  | |
       +++  +^+
        |    |
   Node |    | Node
Publish |    | Unpublish
 Volume |    | Volume
    +---v----+---+
    | PUBLISHED  |
    +------------+

Figure 8: Plugins MAY forego other lifecycle steps by contraindicating
them via the capabilities API. Interactions with the volumes of such
plugins is reduced to `NodePublishVolume` and `NodeUnpublishVolume`
calls.
```



上述图表展示了容器编排系统（CO）通过本规范中定义的 API 管理卷生命周期的一般流程。

插件应实现接口的所有 RPC：控制器插件应实现Controller（控制器）服务的所有 RPC。不支持的 RPC 应返回适当的错误码（例如CALL_NOT_IMPLEMENTED（调用未实现））。

插件能力的完整列表记录在ControllerGetCapabilities（控制器获取能力）和NodeGetCapabilities（节点获取能力）RPC 中。

## 容器存储接口（CSI）

本节描述容器编排系统（CO）与插件（Plugin）之间的接口。


### RPC 接口

CO 通过 RPC 与插件进行交互。

每个存储提供商（SP）必须提供：
- **节点插件（Node Plugin）**：一个提供 CSI RPC 服务的 gRPC 端点，必须运行在 SP 提供的卷将被发布到的节点上。
- **控制器插件（Controller Plugin）**：一个提供 CSI RPC 服务的 gRPC 端点，可以运行在任意位置。
- 在某些情况下，单个 gRPC 端点可提供所有 CSI RPC 服务（参见[架构](#architecture)中的图 3）。

```protobuf
syntax = "proto3";
package csi.v1;

import "google/protobuf/descriptor.proto";
import "google/protobuf/timestamp.proto";
import "google/protobuf/wrappers.proto";

option go_package = 
  "github.com/container-storage-interface/spec/lib/go/csi";

extend google.protobuf.EnumOptions {
  // 表示此枚举为可选，且属于实验性 API，可能在次要版本间被弃用并最终移除
  bool alpha_enum = 1060;
}
extend google.protobuf.EnumValueOptions {
  // 表示此枚举值为可选，且属于实验性 API，可能在次要版本间被弃用并最终移除
  bool alpha_enum_value = 1060;
}
extend google.protobuf.FieldOptions {
  // 表示此字段可能包含敏感信息，必须按敏感信息处理（例如不记录日志）
  bool csi_secret = 1059;

  // 表示此字段为可选，且属于实验性 API，可能在次要版本间被弃用并最终移除
  bool alpha_field = 1060;
}
extend google.protobuf.MessageOptions {
  // 表示此消息为可选，且属于实验性 API，可能在次要版本间被弃用并最终移除
  bool alpha_message = 1060;
}
extend google.protobuf.MethodOptions {
  // 表示此方法为可选，且属于实验性 API，可能在次要版本间被弃用并最终移除
  bool alpha_method = 1060;
}
extend google.protobuf.ServiceOptions {
  // 表示此服务为可选，且属于实验性 API，可能在次要版本间被弃用并最终移除
  bool alpha_service = 1060;
}
```


RPC 分为三组：
- **身份服务（Identity Service）**：节点插件和控制器插件都必须实现这组 RPC。
- **控制器服务（Controller Service）**：控制器插件必须实现这组 RPC。
- **节点服务（Node Service）**：节点插件必须实现这组 RPC。


```protobuf
service Identity {
  rpc GetPluginInfo(GetPluginInfoRequest)
    returns (GetPluginInfoResponse) {}

  rpc GetPluginCapabilities(GetPluginCapabilitiesRequest)
    returns (GetPluginCapabilitiesResponse) {}

  rpc Probe (ProbeRequest)
    returns (ProbeResponse) {}
}

service Controller {
  rpc CreateVolume (CreateVolumeRequest)
    returns (CreateVolumeResponse) {}

  rpc DeleteVolume (DeleteVolumeRequest)
    returns (DeleteVolumeResponse) {}

  rpc ControllerPublishVolume (ControllerPublishVolumeRequest)
    returns (ControllerPublishVolumeResponse) {}

  rpc ControllerUnpublishVolume (ControllerUnpublishVolumeRequest)
    returns (ControllerUnpublishVolumeResponse) {}

  rpc ValidateVolumeCapabilities (ValidateVolumeCapabilitiesRequest)
    returns (ValidateVolumeCapabilitiesResponse) {}

  rpc ListVolumes (ListVolumesRequest)
    returns (ListVolumesResponse) {}

  rpc GetCapacity (GetCapacityRequest)
    returns (GetCapacityResponse) {}

  rpc ControllerGetCapabilities (ControllerGetCapabilitiesRequest)
    returns (ControllerGetCapabilitiesResponse) {}

  rpc CreateSnapshot (CreateSnapshotRequest)
    returns (CreateSnapshotResponse) {}

  rpc DeleteSnapshot (DeleteSnapshotRequest)
    returns (DeleteSnapshotResponse) {}

  rpc ListSnapshots (ListSnapshotsRequest)
    returns (ListSnapshotsResponse) {}

  rpc GetSnapshot (GetSnapshotRequest)
    returns (GetSnapshotResponse) {
        option (alpha_method) = true;
    }

  rpc ControllerExpandVolume (ControllerExpandVolumeRequest)
    returns (ControllerExpandVolumeResponse) {}

  rpc ControllerGetVolume (ControllerGetVolumeRequest)
    returns (ControllerGetVolumeResponse) {
        option (alpha_method) = true;
    }

  rpc ControllerModifyVolume (ControllerModifyVolumeRequest)
    returns (ControllerModifyVolumeResponse) {
        option (alpha_method) = true;
    }
}

service GroupController {
  rpc GroupControllerGetCapabilities (
        GroupControllerGetCapabilitiesRequest)
    returns (GroupControllerGetCapabilitiesResponse) {}

  rpc CreateVolumeGroupSnapshot(CreateVolumeGroupSnapshotRequest)
    returns (CreateVolumeGroupSnapshotResponse) {
    }

  rpc DeleteVolumeGroupSnapshot(DeleteVolumeGroupSnapshotRequest)
    returns (DeleteVolumeGroupSnapshotResponse) {
    }

  rpc GetVolumeGroupSnapshot(
        GetVolumeGroupSnapshotRequest)
    returns (GetVolumeGroupSnapshotResponse) {
    }
}

service SnapshotMetadata {
  option (alpha_service) = true;

  rpc GetMetadataAllocated(GetMetadataAllocatedRequest)
    returns (stream GetMetadataAllocatedResponse) {}

  rpc GetMetadataDelta(GetMetadataDeltaRequest)
    returns (stream GetMetadataDeltaResponse) {}
}

service Node {
  rpc NodeStageVolume (NodeStageVolumeRequest)
    returns (NodeStageVolumeResponse) {}

  rpc NodeUnstageVolume (NodeUnstageVolumeRequest)
    returns (NodeUnstageVolumeResponse) {}

  rpc NodePublishVolume (NodePublishVolumeRequest)
    returns (NodePublishVolumeResponse) {}

  rpc NodeUnpublishVolume (NodeUnpublishVolumeRequest)
    returns (NodeUnpublishVolumeResponse) {}

  rpc NodeGetVolumeStats (NodeGetVolumeStatsRequest)
    returns (NodeGetVolumeStatsResponse) {}


  rpc NodeExpandVolume(NodeExpandVolumeRequest)
    returns (NodeExpandVolumeResponse) {}


  rpc NodeGetCapabilities (NodeGetCapabilitiesRequest)
    returns (NodeGetCapabilitiesResponse) {}

  rpc NodeGetInfo (NodeGetInfoRequest)
    returns (NodeGetInfoResponse) {}
}
```

#### 并发处理

通常，集群编排器（CO）负责确保同一时间内针对单个卷的“正在执行中”调用不超过一个。然而，在某些情况下，CO可能会丢失状态（例如CO崩溃并重启时），并可能针对同一卷同时发起多个调用。插件应尽可能优雅地处理这种情况。在这种情况下，插件可返回`ABORTED`错误码（详情参见[错误机制](#error-scheme)部分）。


#### 字段要求

除非另有说明，本文档中规定的要求适用于本规范所定义的所有protobuf消息类型的字段，无一例外。违反这些要求可能导致RPC消息数据与所有CO、插件和/或CSI中间件实现不兼容。


##### 大小限制

CSI为不同类型的字段定义了通用大小限制（见下表）。特定字段的通用大小限制可通过在该字段的描述中指定不同的大小限制来覆盖。除非另有规定，字段不得超过此处记录的限制。这些限制适用于CO和插件生成的消息。

| 大小 | 字段类型 |
|------------|---------------------|
| 128字节 | string |
| 4 KiB | map<string, string> |


##### `REQUIRED`（必填）与`OPTIONAL`（可选）

- 标记为`REQUIRED`的字段必须被指定，但需遵循任何特定于RPC的说明；此类说明应尽可能少。
- 标记为`REQUIRED`的`repeated`（重复）或`map`（映射）字段必须至少包含1个元素。
- 标记为`OPTIONAL`的字段可以被指定，且规范应明确定义此类字段的默认零值的预期行为。

根据[proto3](https://developers.google.com/protocol-buffers/docs/proto3#default)的规定，即使是必填的标量字段，如果未被指定也会使用默认值，且任何设置为默认值的字段在网络传输时都不会被序列化。


#### 超时处理

本规范中定义的任何RPC都可能超时，且可能被重试。CO可选择其愿意等待调用的最长时间、重试间隔以及重试次数（这些值无需在插件和CO之间协商）。

幂等性要求确保具有相同字段的重试调用在重试时能接续之前的进度。取消调用的唯一方式是发起一个“否定”调用（如果存在的话）。例如，发起`ControllerUnpublishVolume`调用来取消 pending 的`ControllerPublishVolume`操作等。

在某些情况下，CO可能无法取消pending的操作，因为它需要依赖pending操作的结果才能执行“否定”调用。例如，如果`CreateVolume`调用始终未完成，CO可能没有`volume_id`来调用`DeleteVolume`。


### 错误机制

本规范中定义的所有CSI API调用必须返回[标准gRPC状态](https://github.com/grpc/grpc/blob/master/src/proto/grpc/status/status.proto)。大多数gRPC库都提供了设置和读取状态字段的辅助方法。

状态`code`（代码）必须包含一个[规范错误码](https://github.com/grpc/grpc-go/blob/master/codes/codes.go)。CO必须处理所有有效的错误码。每个RPC都定义了一组gRPC错误码，当遇到特定条件时，插件必须返回这些错误码。此外，如果遇到以下定义的条件，插件也必须返回相关的gRPC错误码。

| 条件 | gRPC代码 | 描述 | 恢复行为 |
|-----------|-----------|-------------|-------------------|
| 缺少必填字段 | 3 INVALID_ARGUMENT | 表示请求中缺少必填字段。`status.message`字段中可提供更易读的信息。 | 调用方必须在重试前通过添加缺失的必填字段来修正请求。 |
| 请求中存在无效或不支持的字段 | 3 INVALID_ARGUMENT | 表示该字段中的一个或多个字段要么不被插件允许，要么具有无效值。gRPC的`status.message`字段中可提供更易读的信息。 | 调用方必须在重试前修正该字段。 |
| 权限被拒绝 | 7 PERMISSION_DENIED | 插件能够从RPC中的密钥推导出或以其他方式推断出身份，但该身份无权调用此RPC。 | 系统管理员应确保授予了必要的权限，之后调用方可重试该RPC。 |
| 卷存在未完成操作 | 10 ABORTED | 表示指定的卷已有一个未完成的操作。通常，集群编排器（CO）负责确保同一时间内针对单个卷的“正在执行中”调用不超过一个。然而，在某些情况下，CO可能会丢失状态（例如CO崩溃并重启时），并可能针对同一卷同时发起多个调用。插件应尽可能优雅地处理这种情况，并可返回此错误码以拒绝次要调用。 | 调用方应确保指定的卷没有其他未完成的调用，然后使用指数退避策略重试。 |
| 调用未实现 | 12 UNIMPLEMENTED | 被调用的RPC未由插件实现，或在插件当前的操作模式下被禁用。 | 调用方不得重试。调用方可调用`GetPluginCapabilities`、`ControllerGetCapabilities`或`NodeGetCapabilities`来发现插件的能力。 |
| 未通过身份验证 | 16 UNAUTHENTICATED | 被调用的RPC未携带有效的身份验证密钥。 | 调用方应要么修正RPC中提供的密钥，要么以其他方式更新这些密钥，使其能通过插件对该RPC的身份验证，之后调用方可重试该RPC。 |

当状态`code`不是`OK`时，状态`message`（消息）必须包含人类可读的错误描述。CO可能会将此字符串展示给终端用户。

状态`details`（详情）必须为空。未来，本规范可能要求当状态`code`不是`OK`时，`details`返回可机器解析的protobuf消息，以使CO能够实现更智能的错误处理和故障解决。


#### 密钥要求

插件可能需要密钥来完成RPC请求。密钥是一个字符串到字符串的映射，其中键标识密钥的名称（例如“username”或“password”），值包含密钥数据（例如“bob”或“abc123”）。

每个键必须由字母数字字符、'-'、'_'或'.'组成。每个值必须包含有效的字符串。存储提供商（SP）可选择通过使用二进制到文本的编码方案（如base64）来接受二进制（非字符串）数据。

SP应在文档中说明所需密钥的键和值的要求。CO应允许传递所需的密钥。CO可能会将相同的密钥传递给所有RPC，因此SP期望的所有唯一密钥的键在所有CSI操作中必须是唯一的。

此类信息属于敏感信息，CO必须妥善处理（例如不记录日志等）。

### 身份服务 RPC

身份服务 RPC 允许容器编排系统（CO）查询插件的能力、健康状态和其他元数据。

成功场景下的大致流程如下（为简洁起见，协议缓冲区以 YAML 格式展示）：

1. CO 通过身份 RPC 查询元数据。
```
   # CO --(GetPluginInfo)--> 插件
   请求：
   响应：
      name: org.foo.whizbang.super-plugin
      vendor_version: blue-green
      manifest:
        baz: qaz
```

2. CO 查询插件的可用能力。
```
   # CO --(GetPluginCapabilities)--> 插件
   请求：
   响应：
     capabilities:
       - service:
           type: CONTROLLER_SERVICE
```

3. CO 查询插件的就绪状态。
```
   # CO --(Probe)--> 插件
   请求：
   响应：{}
```


#### `GetPluginInfo`
```protobuf
message GetPluginInfoRequest {
  // 故意为空。
}

message GetPluginInfoResponse {
  // 名称必须遵循域名表示格式
  // (https://tools.ietf.org/html/rfc1035#section-2.3.1)。
  // 它应包含插件的宿主公司名称和插件名称，以最大程度减少冲突的可能性。
  // 名称长度必须不超过 63 个字符，以字母数字字符（[a-z0-9A-Z]）开头和结尾，中间可包含连字符（-）、点（.）和字母数字字符。
  // 此字段为必填项。
  string name = 1;

  // 此字段为必填项。CO 对此字段的值不做解析。
  string vendor_version = 2;

  // 此字段为可选字段。CO 对此字段的值不做解析。
  map<string, string> manifest = 3;
}
```

##### GetPluginInfo 错误
如果插件无法成功完成 GetPluginInfo 调用，它必须在 gRPC 状态中返回非 ok 的 gRPC 代码。


#### `GetPluginCapabilities`
这个必填的 RPC 允许 CO 查询插件“整体”的支持能力：它是插件软件所有实例的所有能力的总和，适用于插件的预期部署场景。

同一版本（参见 `GetPluginInfoResponse` 的 `vendor_version`）的插件的所有实例，无论（a）实例部署在集群的哪个位置，以及（b）实例提供哪些 RPC，都必须返回相同的能力集。

```protobuf
message GetPluginCapabilitiesRequest {
  // 故意为空。
}

message GetPluginCapabilitiesResponse {
  // 控制器服务支持的所有能力。此字段为可选字段。
  repeated PluginCapability capabilities = 1;
}

// 指定插件的一项能力。
message PluginCapability {
  message Service {
    enum Type {
      UNKNOWN = 0;
      
      // CONTROLLER_SERVICE 表示插件提供控制器服务的 RPC。
      // 插件应提供此能力。在极少数情况下，某些插件可能希望完全省略控制器服务的实现，但这不应成为常见情况。
      // 此能力的存在决定了 CO 是否会尝试调用必需的控制器服务 RPC，以及由 ControllerGetCapabilities 指示的特定 RPC。
      CONTROLLER_SERVICE = 1;

      // VOLUME_ACCESSIBILITY_CONSTRAINTS 表示此插件的卷可能无法被集群中的所有节点平等访问。
      // CO 必须使用 CreateVolumeRequest 返回的拓扑信息以及 NodeGetInfo 返回的拓扑信息，确保在调度工作负载时，给定的卷可从给定的节点访问。
      VOLUME_ACCESSIBILITY_CONSTRAINTS = 2;

      // GROUP_CONTROLLER_SERVICE 表示插件提供用于操作卷组的 RPC。插件可提供此能力。
      // 此能力的存在决定了 CO 是否会尝试调用必需的 GroupController 服务 RPC，以及由 GroupControllerGetCapabilities 指示的特定 RPC。
      GROUP_CONTROLLER_SERVICE = 3;

      // SNAPSHOT_METADATA_SERVICE 表示插件提供 RPC，用于检索单个快照的已分配块的元数据，或同一块卷的一对快照之间的已更改块的元数据。
      // 此能力的存在决定了 CO 是否会尝试调用可选的 SnapshotMetadata 服务 RPC。
      SNAPSHOT_METADATA_SERVICE = 4 [(alpha_enum_value) = true];
    }
    Type type = 1;
  }

  message VolumeExpansion {
    enum Type {
      UNKNOWN = 0;

      // ONLINE 表示卷在发布到节点时可以扩展。
      // 当插件实现此能力时，它必须实现 EXPAND_VOLUME 控制器能力、EXPAND_VOLUME 节点能力，或两者都实现。
      // 当插件支持 ONLINE 卷扩展并且还具有 EXPAND_VOLUME 控制器能力时，插件必须支持当前已发布并在节点上可用的卷的扩展。
      // 当插件支持 ONLINE 卷扩展并且还具有 EXPAND_VOLUME 节点能力时，插件可支持通过 NodeExpandVolume 扩展节点发布的卷。
      //
      // 示例 1：对于共享文件系统卷（如 GlusterFs），插件可设置 ONLINE 卷扩展能力，并实现 ControllerExpandVolume，但不实现 NodeExpandVolume。
      //
      // 示例 2：对于块存储卷类型（如 EBS），插件可设置 ONLINE 卷扩展能力，并同时实现 ControllerExpandVolume 和 NodeExpandVolume。
      //
      // 示例 3：对于仅支持在节点上扩展卷的插件，插件可设置 ONLINE 卷扩展能力，并实现 NodeExpandVolume，但不实现 ControllerExpandVolume。
      ONLINE = 1;

      // OFFLINE 表示当前已发布并在节点上可用的卷不应通过 ControllerExpandVolume 扩展。
      // 当插件支持 OFFLINE 卷扩展时，它必须实现 EXPAND_VOLUME 控制器能力，或者同时实现 EXPAND_VOLUME 控制器能力和 EXPAND_VOLUME 节点能力。
      //
      // 示例 1：对于不支持“节点连接”（即控制器发布）的卷扩展的块存储卷类型（如 Azure Disk），插件可指示 OFFLINE 卷扩展支持，并同时实现 ControllerExpandVolume 和 NodeExpandVolume。
      OFFLINE = 2;
    }
    Type type = 1;
  }

  oneof type {
    // 插件支持的服务。
    Service service = 1;
    VolumeExpansion volume_expansion = 2;
  }
}
```

##### GetPluginCapabilities 错误
如果插件无法成功完成 GetPluginCapabilities 调用，它必须在 gRPC 状态中返回非 ok 的 gRPC 代码。


#### `Probe`
插件必须实现此 RPC 调用。

Probe RPC 的主要用途是验证插件是否处于健康且就绪的状态。如果通过非成功响应报告不健康状态，CO 可采取措施以试图使插件恢复到健康状态。这些措施可包括但不限于：
- 重启插件容器；
- 通知插件管理器。

插件可验证它是否具有正确的配置、设备、依赖项和驱动程序以运行，若验证成功，则返回成功。

CO 可在任何时候调用此 RPC。CO 可多次调用此方法，但需了解插件的实现可能并非简单，且重复调用可能会产生开销。

存储提供商（SP）应记录关于特定插件实现此 RPC 的指导和已知限制。例如，SP 可记录其 Probe 实现应被调用的最大频率。

```protobuf
message ProbeRequest {
  // 故意为空。
}

message ProbeResponse {
  // Readiness 允许插件向 CO 报告其初始化状态。
  // 某些插件的初始化可能耗时较长，CO 区分以下情况很重要：
  //
  // 1) 插件处于不健康状态，可能需要重启。在这种情况下，应返回 gRPC 错误代码。
  // 2) 插件仍在初始化中，但在其他方面完全健康。在这种情况下，应返回成功响应，且 readiness 值为 false。
  //    对插件的控制器和/或节点服务的调用可能会因初始化状态不完整而失败。
  // 3) 插件已完成初始化，准备好处理对其控制器和/或节点服务的调用。返回成功响应，且 readiness 值为 true。
  //
  // 此字段为可选字段。如果不存在，调用方应假定插件处于就绪状态，并正在接受对其控制器和/或节点服务的调用（根据插件报告的能力）。
  .google.protobuf.BoolValue ready = 1;
}
```

##### Probe 错误
如果插件无法成功完成 Probe 调用，它必须在 gRPC 状态中返回非 ok 的 gRPC 代码。

如果遇到以下定义的条件，插件必须返回指定的 gRPC 错误代码。CO 在遇到 gRPC 错误代码时，必须实现指定的错误恢复行为。

| 条件 | gRPC 代码 | 描述 | 恢复行为 |
|-----------|-----------|-------------|-------------------|
| 插件不健康 | 9 FAILED_PRECONDITION | 表示插件未处于健康/就绪状态。 | 调用方应假定插件不健康，且未来的 RPC 可能因此失败。 |
| 缺少必需的依赖项 | 9 FAILED_PRECONDITION | 表示插件缺少一个或多个必需的依赖项。 | 调用方必须假定插件不健康。 |

### 控制器服务 RPC

#### `CreateVolume`

如果控制器插件具有`CREATE_DELETE_VOLUME`控制器能力，则必须实现此 RPC 调用。CO 将调用此 RPC，代表用户配置新卷（作为块设备或已挂载的文件系统使用）。

此操作必须具备幂等性。如果与指定卷`name`对应的卷已存在、可从`accessibility_requirements`访问，且与`CreateVolumeRequest`中指定的`capacity_range`、`volume_capabilities`、`parameters`和`mutable_parameters`兼容，则插件必须返回`0 OK`及对应的`CreateVolumeResponse`。

`parameters`字段应包含在创建时指定的不透明卷属性。`mutable_parameters`字段应包含在创建时定义但可在卷的生命周期内通过后续`ControllerModifyVolume` RPC 更改的不透明卷属性。`mutable_parameters`中指定的值必须优先于`parameters`中的值。

插件可创建 3 种类型的卷：
- 空卷：当插件支持`CREATE_DELETE_VOLUME`可选能力时。
- 基于现有快照：当插件支持`CREATE_DELETE_VOLUME`和`CREATE_DELETE_SNAPSHOT`可选能力时。
- 基于现有卷：当插件支持克隆，并报告可选能力`CREATE_DELETE_VOLUME`和`CLONE_VOLUME`时。

如果 CO 请求从现有快照或卷创建卷，且请求的卷大小大于原始快照（或克隆卷）的大小，插件可通过`OUT_OF_RANGE`错误拒绝该调用，或者必须提供一个卷，当通过`NodePublish`调用呈现给工作负载时，该卷既具有请求的（更大）大小，又包含来自快照（或原始卷）的数据。明确地说，如果卷具有`VolumeCapability`访问类型`MountVolume`，且需要调整文件系统大小以提供请求的容量，则插件有责任在`NodePublish`调用时（或之前）调整新创建卷的文件系统大小。

```protobuf
message CreateVolumeRequest {
  // 存储空间的建议名称。此字段为必填项。
  // 它有两个用途：
  // 1) 幂等性 - 此名称由 CO 生成以实现幂等性。插件应确保对相同名称的多个`CreateVolume`调用不会导致对应于该名称的多个存储被配置。如果插件无法保证幂等性，CO 的错误恢复逻辑可能会导致多个（未使用的）卷被配置。
  //    发生错误时，CO 必须按照“CreateVolume 错误”部分中定义的恢复行为处理 gRPC 错误代码。
  //    CO 负责清理它所配置但不再需要的卷。如果在`CreateVolume`调用失败时，CO 不确定是否已配置卷，CO 可使用相同的名称再次调用`CreateVolume`，以确保卷存在并检索卷的`volume_id`（除非“CreateVolume 错误”另有禁止）。
  // 2) 建议名称 - 某些存储系统允许调用者指定用于引用新配置存储的标识符。如果存储系统支持此功能，它可选择使用此名称作为新卷的标识符。
  // 允许任何符合长度限制的 Unicode 字符串，但不包含以下禁用字符：
  // U+0000-U+0008、U+000B、U+000C、U+000E-U+001F、U+007F-U+009F。
  // （这些是除常用空白字符外的控制字符。）
  string name = 1;

  // 此字段为可选字段。这允许 CO 指定要配置的卷的容量要求。如果未指定，插件可选择实现定义的容量范围。如果指定，即使从源创建卷，也必须始终遵守，这可能会迫使某些后端在创建卷后内部扩展卷。
  CapacityRange capacity_range = 2;

  // 已配置卷必须具备的能力。存储提供商（SP）必须配置满足此列表中所有能力的卷。否则，SP 必须返回适当的 gRPC 错误代码。
  // 插件必须假定 CO 可使用此列表中任何能力的已配置卷。
  // 例如，CO 可指定两个卷能力：一个具有访问模式 SINGLE_NODE_WRITER，另一个具有访问模式 MULTI_NODE_READER_ONLY。在这种情况下，SP 必须验证已配置的卷可在任一模式下使用。
  // 这也使 CO 能够进行早期验证：如果 SP 不支持任何指定的卷能力，调用必须返回适当的 gRPC 错误代码。
  // 此字段为必填项。
  repeated VolumeCapability volume_capabilities = 3;

  // 作为不透明键值对传入的特定于插件的创建时参数。此字段为可选字段。插件负责解析和验证这些参数。CO 将视这些参数为不透明的。
  map<string, string> parameters = 4;

  // 插件完成卷创建请求所需的密钥。此字段为可选字段。有关如何使用此字段，请参阅“密钥要求”部分。
  map<string, string> secrets = 5 [(csi_secret) = true];

  // 如果指定，新卷将用来自此源的数据预填充。此字段为可选字段。
  VolumeContentSource volume_content_source = 6;

  // 指定已配置卷必须可从哪些位置（区域、可用区、机架等）访问。
  // SP 应在文档中说明拓扑可访问性信息的要求。CO 仅应指定 SP 支持的拓扑可访问性信息。
  // 此字段为可选字段。
  // 除非 SP 具有 VOLUME_ACCESSIBILITY_CONSTRAINTS 插件能力，否则不得指定此字段。
  // 如果未指定此字段且 SP 具有 VOLUME_ACCESSIBILITY_CONSTRAINTS 插件能力，SP 可选择已配置卷可访问的位置。
  TopologyRequirement accessibility_requirements = 7;

  // 作为不透明键值对传入的特定于插件的创建时参数。这些 mutable_parameters 可在卷的生命周期内通过后续`ControllerModifyVolume` RPC 更改。此字段为可选字段。
  // 插件负责解析和验证这些参数。CO 将视这些参数为不透明的。
  // 插件必须将这些参数视为优先于 parameters 字段。
  // 除非 SP 具有 MODIFY_VOLUME 插件能力，否则不得指定此字段。
  map<string, string> mutable_parameters = 8 [(alpha_field) = true];
}

// 指定卷将从哪个源创建。必须指定其中一个类型字段。
message VolumeContentSource {
  message SnapshotSource {
    // 包含现有源快照的身份信息。此字段为必填项。如果插件支持 CREATE_DELETE_SNAPSHOT 能力，则必须支持从快照创建卷。
    string snapshot_id = 1;
  }

  message VolumeSource {
    // 包含现有源卷的身份信息。此字段为必填项。报告 CLONE_VOLUME 能力的插件必须支持从另一个卷创建卷。
    string volume_id = 1;
  }

  oneof type {
    SnapshotSource snapshot = 1;
    VolumeSource volume = 2;
  }
}

message CreateVolumeResponse {
  // 包含与 CO 相关的新创建卷的所有属性，以及插件唯一标识该卷所需的信息。此字段为必填项。
  Volume volume = 1;
}

// 指定卷的能力。
message VolumeCapability {
  // 指示卷将通过块设备 API 访问。
  message BlockVolume {
    // 目前故意为空。
  }

  // 指示卷将通过文件系统 API 访问。
  message MountVolume {
    // 文件系统类型。此字段为可选字段。空字符串等同于未指定的字段值。
    string fs_type = 1;

    // 可用于卷的挂载选项。此字段为可选字段。`mount_flags` 可能包含敏感信息。因此，CO 和插件不得向不受信任的实体泄露此信息。此重复字段的总大小不得超过 4 KiB。
    repeated string mount_flags = 2;

    // 如果 SP 具有 VOLUME_MOUNT_GROUP 节点能力且 CO 提供此字段，则 SP 必须确保将 volume_mount_group 参数作为组标识符传递给底层操作系统的挂载系统调用，同时要理解可用的挂载调用参数和/或挂载实现可能因操作系统而异。
    // 此外，写入底层文件系统的新文件和/或目录条目应具有这样的权限标记，除非被工作负载修改，否则它们都可由所述挂载组标识符读写。
    // 这是一个可选字段。
    string volume_mount_group = 3;
  }

  // 指定卷的访问方式。
  message AccessMode {
    enum Mode {
      UNKNOWN = 0;

      // 在任何给定时间，只能在单个节点上以读写方式发布一次。
      SINGLE_NODE_WRITER = 1;

      // 在任何给定时间，只能在单个节点上以只读方式发布一次。
      SINGLE_NODE_READER_ONLY = 2;

      // 可同时在多个节点上以只读方式发布。
      MULTI_NODE_READER_ONLY = 3;

      // 可同时在多个节点上发布。只有其中一个节点可用作读写节点，其余节点为只读。
      MULTI_NODE_SINGLE_WRITER = 4;

      // 可同时在多个节点上以读写方式发布。
      MULTI_NODE_MULTI_WRITER = 5;

      // 在任何给定时间，只能在单个节点上的单个工作负载上以读写方式发布一次。对于使用实验性 SINGLE_NODE_MULTI_WRITER 能力的 CO，应使用此模式代替 SINGLE_NODE_WRITER。
      SINGLE_NODE_SINGLE_WRITER = 6 [(alpha_enum_value) = true];

      // 可同时在单个节点上的多个工作负载上以读写方式发布。对于使用实验性 SINGLE_NODE_MULTI_WRITER 能力的 CO，应使用此模式代替 SINGLE_NODE_WRITER。
      SINGLE_NODE_MULTI_WRITER = 7 [(alpha_enum_value) = true];
    }

    // 此字段为必填项。
    Mode mode = 1;
  }

  // 指定将使用什么 API 访问卷。必须指定以下字段之一。
  oneof access_type {
    BlockVolume block = 1;
    MountVolume mount = 2;
  }

  // 此字段为必填项。
  AccessMode access_mode = 3;
}

// 存储空间的容量（以字节为单位）。要指定精确大小，`required_bytes` 和 `limit_bytes` 应设置为相同值。必须至少指定其中一个字段。
message CapacityRange {
  // 卷必须至少有这么大。此字段为可选字段。值为 0 等同于未指定的字段值。此字段的值不得为负数。
  int64 required_bytes = 1;

  // 卷不得大于此值。此字段为可选字段。值为 0 等同于未指定的字段值。此字段的值不得为负数。
  int64 limit_bytes = 2;
}

// 特定卷的信息。
message Volume {
  // 卷的容量（以字节为单位）。此字段为可选字段。如果未设置（值为 0），表示卷的容量未知（例如，NFS 共享）。此字段的值不得为负数。
  int64 capacity_bytes = 1;

  // 插件生成的此卷的标识符。此字段为必填项。
  // 此字段必须包含足够的信息以唯一标识此特定卷与该插件支持的所有其他卷。
  // CO 将在后续调用中使用此字段引用此卷。
  // SP 不负责多个 SP 之间 volume_id 的全局唯一性。
  string volume_id = 2;

  // 卷的不透明静态属性。SP 可使用此字段确保后续的卷验证和发布调用具有上下文信息。
  // 此字段的内容对 CO 是不透明的。
  // 此字段的内容不得更改。
  // 此字段的内容可供 CO 安全缓存。
  // 此字段的内容不应包含敏感信息。
  // 此字段的内容不应用于唯一标识卷。仅 `volume_id` 应足以标识卷。
  // 由 `volume_id` 唯一标识的卷应始终报告相同的 volume_context。
  // 此字段为可选字段，存在时必须传递给卷验证和发布调用。
  map<string, string> volume_context = 3;

  // 如果指定，表示卷非空，并已用来自指定源的数据预填充。此字段为可选字段。
  VolumeContentSource content_source = 4;

  // 指定已配置卷可从哪些位置（区域、可用区、机架等）访问。
  // 返回此字段的插件还必须设置 VOLUME_ACCESSIBILITY_CONSTRAINTS 插件能力。
  // SP 可指定多个拓扑，以指示卷可从多个位置访问。
  // CO 可使用此信息以及 NodeGetInfo 返回的拓扑信息，确保在调度工作负载时，给定的卷可从给定的节点访问。
  // 此字段为可选字段。如果未指定，CO 可假定该卷可从集群中的所有节点平等访问，并可在任何可用节点上调度引用该卷的工作负载。
  //
  // 示例 1：
  //   accessible_topology = {"region": "R1", "zone": "Z2"}
  // 表示卷仅可从“区域”“R1”和“可用区”“Z2”访问。
  //
  // 示例 2：
  //   accessible_topology =
  //     {"region": "R1", "zone": "Z2"},
  //     {"region": "R1", "zone": "Z3"}
  // 表示卷可从“区域”“R1”中的“可用区”“Z2”和“可用区”“Z3”访问。
  repeated Topology accessible_topology = 5;
}

message TopologyRequirement {
  // 指定已配置卷必须可从其访问的拓扑列表。
  // 此字段为可选字段。如果指定了 TopologyRequirement，则必须指定 requisite或 preferred 或两者。
  //
  // 如果指定了 requisite，已配置卷必须可从至少一个 requisite拓扑访问。
  //
  // 假设
  //   x = 已配置卷可访问的拓扑数量
  //   n =  requisite 拓扑的数量
  // CO 必须确保 n >= 1。SP 必须确保 x >= 1
  // 如果 x==n，则 SP 必须使已配置卷可用于 requisite 拓扑列表中的所有拓扑。如果无法做到，SP 必须使 CreateVolume 调用失败。
  // 例如，如果卷应可从单个可用区访问，且 requisite =
  //   {"region": "R1", "zone": "Z2"}
  // 则已配置卷必须可从“区域”“R1”和“可用区”“Z2”访问。
  // 同样，如果卷应可从两个可用区访问，且 requisite =
  //   {"region": "R1", "zone": "Z2"},
  //   {"region": "R1", "zone": "Z3"}
  // 则已配置卷必须可从“区域”“R1”以及“可用区”“Z2”和“可用区”“Z3”访问。
  //
  // 如果 x<n，则 SP 应从 requisite 拓扑列表中选择 x 个唯一拓扑。如果无法做到，SP 必须使 CreateVolume 调用失败。
  // 例如，如果卷应可从单个可用区访问，且 requisite =
  //   {"region": "R1", "zone": "Z2"},
  //   {"region": "R1", "zone": "Z3"}
  // 则 SP 可选择使已配置卷在“区域”“R1”中的“可用区”“Z2”或“可用区”“Z3”中可用。
  // 同样，如果卷应可从两个可用区访问，且 requisite =
  //   {"region": "R1", "zone": "Z2"},
  //   {"region": "R1", "zone": "Z3"},
  //   {"region": "R1", "zone": "Z4"}
  // 则已配置卷必须可从任意两个唯一拓扑的组合访问：例如“R1/Z2”和“R1/Z3”，或“R1/Z2”和“R1/Z4”，或“R1/Z3”和“R1/Z4”。
  //
  // 如果 x>n，则 SP 必须使已配置卷可从 requisite 拓扑列表中的所有拓扑访问，并可从所有可能的拓扑列表中选择其余 x-n 个唯一拓扑。如果无法做到，SP 必须使 CreateVolume 调用失败。
  // 例如，如果卷应可从两个可用区访问，且 requisite =
  //   {"region": "R1", "zone": "Z2"}
  // 则已配置卷必须可从“区域”“R1”和“可用区”“Z2”访问，SP 可独立选择第二个可用区，例如“R1/Z4”。
  repeated Topology requisite = 1;

  // 指定 CO 希望卷配置在的拓扑列表。
  //
  // 此字段为可选字段。如果指定了 TopologyRequirement，则必须指定 requisite或 preferred 或两者。
  //
  // SP 必须尝试按从第一个到最后一个的顺序使用首选拓扑使已配置卷可用。
  //
  // 如果指定了 requisite，首选列表中的所有拓扑必须也存在于 requisite 拓扑列表中。
  //
  // 如果 SP 无法使已配置卷从任何首选拓扑可用，SP 可从 requisite 拓扑列表中选择一个拓扑。
  // 如果未指定 requisite 拓扑列表，则 SP 可从所有可能的拓扑列表中选择。
  // 如果指定了 requisite 拓扑列表，且 SP 无法使已配置卷从任何 requisite 拓扑可用，它必须使 CreateVolume 调用失败。
  //
  // Example 1:
  // Given a volume should be accessible from a single zone, and
  // requisite =
  //   {"region": "R1", "zone": "Z2"},
  //   {"region": "R1", "zone": "Z3"}
  // preferred =
  //   {"region": "R1", "zone": "Z3"}
  // then the SP SHOULD first attempt to make the provisioned volume
  // available from "zone" "Z3" in the "region" "R1" and fall back to
  // "zone" "Z2" in the "region" "R1" if that is not possible.
  //
  // Example 2:
  // Given a volume should be accessible from a single zone, and
  // requisite =
  //   {"region": "R1", "zone": "Z2"},
  //   {"region": "R1", "zone": "Z3"},
  //   {"region": "R1", "zone": "Z4"},
  //   {"region": "R1", "zone": "Z5"}
  // preferred =
  //   {"region": "R1", "zone": "Z4"},
  //   {"region": "R1", "zone": "Z2"}
  // then the SP SHOULD first attempt to make the provisioned volume
  // accessible from "zone" "Z4" in the "region" "R1" and fall back to
  // "zone" "Z2" in the "region" "R1" if that is not possible. If that
  // is not possible, the SP may choose between either the "zone"
  // "Z3" or "Z5" in the "region" "R1".
  //
  // Example 3:
  // Given a volume should be accessible from TWO zones (because an
  // opaque parameter in CreateVolumeRequest, for example, specifies
  // the volume is accessible from two zones, aka synchronously
  // replicated), and
  // requisite =
  //   {"region": "R1", "zone": "Z2"},
  //   {"region": "R1", "zone": "Z3"},
  //   {"region": "R1", "zone": "Z4"},
  //   {"region": "R1", "zone": "Z5"}
  // preferred =
  //   {"region": "R1", "zone": "Z5"},
  //   {"region": "R1", "zone": "Z3"}
  // then the SP SHOULD first attempt to make the provisioned volume
  // accessible from the combination of the two "zones" "Z5" and "Z3" in
  // the "region" "R1". If that's not possible, it should fall back to
  // a combination of "Z5" and other possibilities from the list of
  // requisite. If that's not possible, it should fall back  to a
  // combination of "Z3" and other possibilities from the list of
  // requisite. If that's not possible, it should fall back  to a
  // combination of other possibilities from the list of requisite.
  repeated Topology preferred = 2;
}

// Topology is a map of topological domains to topological segments.
// A topological domain is a sub-division of a cluster, like "region",
// "zone", "rack", etc.
// A topological segment is a specific instance of a topological domain,
// like "zone3", "rack3", etc.
// For example {"com.company/zone": "Z1", "com.company/rack": "R3"}
// Valid keys have two segments: an OPTIONAL prefix and name, separated
// by a slash (/), for example: "com.company.example/zone".
// The key name segment is REQUIRED. The prefix is OPTIONAL.
// The key name MUST be 63 characters or less, begin and end with an
// alphanumeric character ([a-z0-9A-Z]), and contain only dashes (-),
// underscores (_), dots (.), or alphanumerics in between, for example
// "zone".
// The key prefix MUST be 63 characters or less, begin and end with a
// lower-case alphanumeric character ([a-z0-9]), contain only
// dashes (-), dots (.), or lower-case alphanumerics in between, and
// follow domain name notation format
// (https://tools.ietf.org/html/rfc1035#section-2.3.1).
// The key prefix SHOULD include the plugin's host company name and/or
// the plugin name, to minimize the possibility of collisions with keys
// from other plugins.
// If a key prefix is specified, it MUST be identical across all
// topology keys returned by the SP (across all RPCs).
// Keys MUST be case-insensitive. Meaning the keys "Zone" and "zone"
// MUST not both exist.
// Each value (topological segment) MUST contain 1 or more strings.
// Each string MUST be 63 characters or less and begin and end with an
// alphanumeric character with '-', '_', '.', or alphanumerics in
// between.
message Topology {
  map<string, string> segments = 1;
}
```

