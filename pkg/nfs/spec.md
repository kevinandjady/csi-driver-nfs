# Container Storage Interface (CSI)

Authors:

* Jie Yu <<jie@mesosphere.io>> (@jieyu)
* Saad Ali <<saadali@google.com>> (@saad-ali)
* James DeFelice <<james@mesosphere.io>> (@jdef)
* <container-storage-interface-working-group@googlegroups.com>

## Notational Conventions

The keywords "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" are to be interpreted as described in [RFC 2119](http://tools.ietf.org/html/rfc2119) (Bradner, S., "Key words for use in RFCs to Indicate Requirement Levels", BCP 14, RFC 2119, March 1997).

The key words "unspecified", "undefined", and "implementation-defined" are to be interpreted as described in the [rationale for the C99 standard](http://www.open-std.org/jtc1/sc22/wg14/www/C99RationaleV5.10.pdf#page=18).

An implementation is not compliant if it fails to satisfy one or more of the MUST, REQUIRED, or SHALL requirements for the protocols it implements.
An implementation is compliant if it satisfies all the MUST, REQUIRED, and SHALL requirements for the protocols it implements.

## Terminology

| Term              | Definition                                       |
|-------------------|--------------------------------------------------|
| Volume            | A unit of storage that will be made available inside of a CO-managed container, via the CSI.                          |
| Block Volume      | A volume that will appear as a block device inside the container.                                                     |
| Mounted Volume    | A volume that will be mounted using the specified file system and appear as a directory inside the container.         |
| CO                | Container Orchestration system, communicates with Plugins using CSI service RPCs.                                     |
| SP                | Storage Provider, the vendor of a CSI plugin implementation.                                                          |
| RPC               | [Remote Procedure Call](https://en.wikipedia.org/wiki/Remote_procedure_call).                                         |
| Node              | A host where the user workload will be running, uniquely identifiable from the perspective of a Plugin by a node ID.  |
| Plugin            | Aka “plugin implementation”, a gRPC endpoint that implements the CSI Services.                                        |
| Plugin Supervisor | Process that governs the lifecycle of a Plugin, MAY be the CO.                                                        |
| Workload          | The atomic unit of "work" scheduled by a CO. This MAY be a container or a collection of containers.                   |

## Objective

To define an industry standard “Container Storage Interface” (CSI) that will enable storage vendors (SP) to develop a plugin once and have it work across a number of container orchestration (CO) systems.

### Goals in MVP

The Container Storage Interface (CSI) will

* Enable SP authors to write one CSI compliant Plugin that “just works” across all COs that implement CSI.
* Define API (RPCs) that enable:
  * Dynamic provisioning and deprovisioning of a volume.
  * Attaching or detaching a volume from a node.
  * Mounting/unmounting a volume from a node.
  * Consumption of both block and mountable volumes.
  * Local storage providers (e.g., device mapper, lvm).
  * Creating and deleting a snapshot (source of the snapshot is a volume).
  * Provisioning a new volume from a snapshot (reverting snapshot, where data in the original volume is erased and replaced with data in the snapshot, is out of scope).
* Define plugin protocol RECOMMENDATIONS.
  * Describe a process by which a Supervisor configures a Plugin.
  * Container deployment considerations (`CAP_SYS_ADMIN`, mount namespace, etc.).

### Non-Goals in MVP

The Container Storage Interface (CSI) explicitly will not define, provide, or dictate:

* Specific mechanisms by which a Plugin Supervisor manages the lifecycle of a Plugin, including:
  * How to maintain state (e.g. what is attached, mounted, etc.).
  * How to deploy, install, upgrade, uninstall, monitor, or respawn (in case of unexpected termination) Plugins.
* A first class message structure/field to represent "grades of storage" (aka "storage class").
* Protocol-level authentication and authorization.
* Packaging of a Plugin.
* POSIX compliance: CSI provides no guarantee that volumes provided are POSIX compliant filesystems.
  Compliance is determined by the Plugin implementation (and any backend storage system(s) upon which it depends).
  CSI SHALL NOT obstruct a Plugin Supervisor or CO from interacting with Plugin-managed volumes in a POSIX-compliant manner.

## Solution Overview

This specification defines an interface along with the minimum operational and packaging recommendations for a storage provider (SP) to implement a CSI compatible plugin.
The interface declares the RPCs that a plugin MUST expose: this is the **primary focus** of the CSI specification.
Any operational and packaging recommendations offer additional guidance to promote cross-CO compatibility.

### Architecture

The primary focus of this specification is on the **protocol** between a CO and a Plugin.
It SHOULD be possible to ship cross-CO compatible Plugins for a variety of deployment architectures.
A CO SHOULD be equipped to handle both centralized and headless plugins, as well as split-component and unified plugins.
Several of these possibilities are illustrated in the following figures.

```
                             CO "Master" Host
+-------------------------------------------+
|                                           |
|  +------------+           +------------+  |
|  |     CO     |   gRPC    | Controller |  |
|  |            +----------->   Plugin   |  |
|  +------------+           +------------+  |
|                                           |
+-------------------------------------------+

                            CO "Node" Host(s)
+-------------------------------------------+
|                                           |
|  +------------+           +------------+  |
|  |     CO     |   gRPC    |    Node    |  |
|  |            +----------->   Plugin   |  |
|  +------------+           +------------+  |
|                                           |
+-------------------------------------------+

Figure 1: The Plugin runs on all nodes in the cluster: a centralized
Controller Plugin is available on the CO master host and the Node
Plugin is available on all of the CO Nodes.
```

```
                            CO "Node" Host(s)
+-------------------------------------------+
|                                           |
|  +------------+           +------------+  |
|  |     CO     |   gRPC    | Controller |  |
|  |            +--+-------->   Plugin   |  |
|  +------------+  |        +------------+  |
|                  |                        |
|                  |                        |
|                  |        +------------+  |
|                  |        |    Node    |  |
|                  +-------->   Plugin   |  |
|                           +------------+  |
|                                           |
+-------------------------------------------+

Figure 2: Headless Plugin deployment, only the CO Node hosts run
Plugins. Separate, split-component Plugins supply the Controller
Service and the Node Service respectively.
```

```
                            CO "Node" Host(s)
+-------------------------------------------+
|                                           |
|  +------------+           +------------+  |
|  |     CO     |   gRPC    | Controller |  |
|  |            +----------->    Node    |  |
|  +------------+           |   Plugin   |  |
|                           +------------+  |
|                                           |
+-------------------------------------------+

Figure 3: Headless Plugin deployment, only the CO Node hosts run
Plugins. A unified Plugin component supplies both the Controller
Service and Node Service.
```

```
                            CO "Node" Host(s)
+-------------------------------------------+
|                                           |
|  +------------+           +------------+  |
|  |     CO     |   gRPC    |    Node    |  |
|  |            +----------->   Plugin   |  |
|  +------------+           +------------+  |
|                                           |
+-------------------------------------------+

Figure 4: Headless Plugin deployment, only the CO Node hosts run
Plugins. A Node-only Plugin component supplies only the Node Service.
Its GetPluginCapabilities RPC does not report the CONTROLLER_SERVICE
capability.
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

The above diagrams illustrate a general expectation with respect to how a CO MAY manage the lifecycle of a volume via the API presented in this specification.
Plugins SHOULD expose all RPCs for an interface: Controller plugins SHOULD implement all RPCs for the `Controller` service.
Unsupported RPCs SHOULD return an appropriate error code that indicates such (e.g. `CALL_NOT_IMPLEMENTED`).
The full list of plugin capabilities is documented in the `ControllerGetCapabilities` and `NodeGetCapabilities` RPCs.

## Container Storage Interface

This section describes the interface between COs and Plugins.

### RPC Interface

A CO interacts with an Plugin through RPCs.
Each SP MUST provide:

* **Node Plugin**: A gRPC endpoint serving CSI RPCs that MUST be run on the Node whereupon an SP-provisioned volume will be published.
* **Controller Plugin**: A gRPC endpoint serving CSI RPCs that MAY be run anywhere.
* In some circumstances a single gRPC endpoint MAY serve all CSI RPCs (see Figure 3 in [Architecture](#architecture)).

```protobuf
syntax = "proto3";
package csi.v1;

import "google/protobuf/descriptor.proto";
import "google/protobuf/timestamp.proto";
import "google/protobuf/wrappers.proto";

option go_package = 
  "github.com/container-storage-interface/spec/lib/go/csi";

extend google.protobuf.EnumOptions {
  // Indicates that this enum is OPTIONAL and part of an experimental
  // API that may be deprecated and eventually removed between minor
  // releases.
  bool alpha_enum = 1060;
}
extend google.protobuf.EnumValueOptions {
  // Indicates that this enum value is OPTIONAL and part of an
  // experimental API that may be deprecated and eventually removed
  // between minor releases.
  bool alpha_enum_value = 1060;
}
extend google.protobuf.FieldOptions {
  // Indicates that a field MAY contain information that is sensitive
  // and MUST be treated as such (e.g. not logged).
  bool csi_secret = 1059;

  // Indicates that this field is OPTIONAL and part of an experimental
  // API that may be deprecated and eventually removed between minor
  // releases.
  bool alpha_field = 1060;
}
extend google.protobuf.MessageOptions {
  // Indicates that this message is OPTIONAL and part of an experimental
  // API that may be deprecated and eventually removed between minor
  // releases.
  bool alpha_message = 1060;
}
extend google.protobuf.MethodOptions {
  // Indicates that this method is OPTIONAL and part of an experimental
  // API that may be deprecated and eventually removed between minor
  // releases.
  bool alpha_method = 1060;
}
extend google.protobuf.ServiceOptions {
  // Indicates that this service is OPTIONAL and part of an experimental
  // API that may be deprecated and eventually removed between minor
  // releases.
  bool alpha_service = 1060;
}
```

There are three sets of RPCs:

* **Identity Service**: Both the Node Plugin and the Controller Plugin MUST implement this sets of RPCs.
* **Controller Service**: The Controller Plugin MUST implement this sets of RPCs.
* **Node Service**: The Node Plugin MUST implement this sets of RPCs.

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

#### Concurrency

In general the Cluster Orchestrator (CO) is responsible for ensuring that there is no more than one call “in-flight” per volume at a given time.
However, in some circumstances, the CO MAY lose state (for example when the CO crashes and restarts), and MAY issue multiple calls simultaneously for the same volume.
The plugin SHOULD handle this as gracefully as possible.
The error code `ABORTED` MAY be returned by the plugin in this case (see the [Error Scheme](#error-scheme) section for details).

#### Field Requirements

The requirements documented herein apply equally and without exception, unless otherwise noted, for the fields of all protobuf message types defined by this specification.
Violation of these requirements MAY result in RPC message data that is not compatible with all CO, Plugin, and/or CSI middleware implementations.

##### Size Limits

CSI defines general size limits for fields of various types (see table below).
The general size limit for a particular field MAY be overridden by specifying a different size limit in said field's description.
Unless otherwise specified, fields SHALL NOT exceed the limits documented here.
These limits apply for messages generated by both COs and plugins.

| Size       | Field Type          |
|------------|---------------------|
| 128 bytes  | string              |
| 4 KiB      | map<string, string> |

##### `REQUIRED` vs. `OPTIONAL`

* A field noted as `REQUIRED` MUST be specified, subject to any per-RPC caveats; caveats SHOULD be rare.
* A `repeated` or `map` field listed as `REQUIRED` MUST contain at least 1 element.
* A field noted as `OPTIONAL` MAY be specified and the specification SHALL clearly define expected behavior for the default, zero-value of such fields.

Scalar fields, even REQUIRED ones, will be defaulted if not specified and any field set to the default value will not be serialized over the wire as per [proto3](https://developers.google.com/protocol-buffers/docs/proto3#default).

#### Timeouts

Any of the RPCs defined in this spec MAY timeout and MAY be retried.
The CO MAY choose the maximum time it is willing to wait for a call, how long it waits between retries, and how many time it retries (these values are not negotiated between plugin and CO).

Idempotency requirements ensure that a retried call with the same fields continues where it left off when retried.
The only way to cancel a call is to issue a "negation" call if one exists.
For example, issue a `ControllerUnpublishVolume` call to cancel a pending `ControllerPublishVolume` operation, etc.
In some cases, a CO MAY NOT be able to cancel a pending operation because it depends on the result of the pending operation in order to execute the "negation" call.
For example, if a `CreateVolume` call never completes then a CO MAY NOT have the `volume_id` to call `DeleteVolume` with.

### Error Scheme

All CSI API calls defined in this spec MUST return a [standard gRPC status](https://github.com/grpc/grpc/blob/master/src/proto/grpc/status/status.proto).
Most gRPC libraries provide helper methods to set and read the status fields.

The status `code` MUST contain a [canonical error code](https://github.com/grpc/grpc-go/blob/master/codes/codes.go). COs MUST handle all valid error codes. Each RPC defines a set of gRPC error codes that MUST be returned by the plugin when specified conditions are encountered. In addition to those, if the conditions defined below are encountered, the plugin MUST return the associated gRPC error code.

| Condition | gRPC Code | Description | Recovery Behavior |
|-----------|-----------|-------------|-------------------|
| Missing required field | 3 INVALID_ARGUMENT | Indicates that a required field is missing from the request. More human-readable information MAY be provided in the `status.message` field. | Caller MUST fix the request by adding the missing required field before retrying. |
| Invalid or unsupported field in the request | 3 INVALID_ARGUMENT | Indicates that the one or more fields in this field is either not allowed by the Plugin or has an invalid value. More human-readable information MAY be provided in the gRPC `status.message` field. | Caller MUST fix the field before retrying. |
| Permission denied | 7 PERMISSION_DENIED | The Plugin is able to derive or otherwise infer an identity from the secrets present within an RPC, but that identity does not have permission to invoke the RPC. | System administrator SHOULD ensure that requisite permissions are granted, after which point the caller MAY retry the attempted RPC. |
| Operation pending for volume | 10 ABORTED | Indicates that there is already an operation pending for the specified volume. In general the Cluster Orchestrator (CO) is responsible for ensuring that there is no more than one call "in-flight" per volume at a given time. However, in some circumstances, the CO MAY lose state (for example when the CO crashes and restarts), and MAY issue multiple calls simultaneously for the same volume. The Plugin, SHOULD handle this as gracefully as possible, and MAY return this error code to reject secondary calls. | Caller SHOULD ensure that there are no other calls pending for the specified volume, and then retry with exponential back off. |
| Call not implemented | 12 UNIMPLEMENTED | The invoked RPC is not implemented by the Plugin or disabled in the Plugin's current mode of operation. | Caller MUST NOT retry. Caller MAY call `GetPluginCapabilities`, `ControllerGetCapabilities`, or `NodeGetCapabilities` to discover Plugin capabilities. |
| Not authenticated | 16 UNAUTHENTICATED | The invoked RPC does not carry secrets that are valid for authentication. | Caller SHALL either fix the secrets provided in the RPC, or otherwise regalvanize said secrets such that they will pass authentication by the Plugin for the attempted RPC, after which point the caller MAY retry the attempted RPC. |

The status `message` MUST contain a human readable description of error, if the status `code` is not `OK`.
This string MAY be surfaced by CO to end users.

The status `details` MUST be empty. In the future, this spec MAY require `details` to return a machine-parsable protobuf message if the status `code` is not `OK` to enable CO's to implement smarter error handling and fault resolution.

### Secrets Requirements

Secrets MAY be required by plugin to complete a RPC request.
A secret is a string to string map where the key identifies the name of the secret (e.g. "username" or "password"), and the value contains the secret data (e.g. "bob" or "abc123").
Each key MUST consist of alphanumeric characters, '-', '_' or '.'.
Each value MUST contain a valid string.
An SP MAY choose to accept binary (non-string) data by using a binary-to-text encoding scheme, like base64.
An SP SHALL advertise the requirements for required secret keys and values in documentation.
CO SHALL permit passing through the required secrets.
A CO MAY pass the same secrets to all RPCs, therefore the keys for all unique secrets that an SP expects MUST be unique across all CSI operations.
This information is sensitive and MUST be treated as such (not logged, etc.) by the CO.

### Identity Service RPC

Identity service RPCs allow a CO to query a plugin for capabilities, health, and other metadata.
The general flow of the success case MAY be as follows (protos illustrated in YAML for brevity):

1. CO queries metadata via Identity RPC.

```
   # CO --(GetPluginInfo)--> Plugin
   request:
   response:
      name: org.foo.whizbang.super-plugin
      vendor_version: blue-green
      manifest:
        baz: qaz
```

2. CO queries available capabilities of the plugin.

```
   # CO --(GetPluginCapabilities)--> Plugin
   request:
   response:
     capabilities:
       - service:
           type: CONTROLLER_SERVICE
```

3. CO queries the readiness of the plugin.

```
   # CO --(Probe)--> Plugin
   request:
   response: {}
```

#### `GetPluginInfo`

```protobuf
message GetPluginInfoRequest {
  // Intentionally empty.
}

message GetPluginInfoResponse {
  // The name MUST follow domain name notation format
  // (https://tools.ietf.org/html/rfc1035#section-2.3.1). It SHOULD
  // include the plugin's host company name and the plugin name,
  // to minimize the possibility of collisions. It MUST be 63
  // characters or less, beginning and ending with an alphanumeric
  // character ([a-z0-9A-Z]) with dashes (-), dots (.), and
  // alphanumerics between. This field is REQUIRED.
  string name = 1;

  // This field is REQUIRED. Value of this field is opaque to the CO.
  string vendor_version = 2;

  // This field is OPTIONAL. Values are opaque to the CO.
  map<string, string> manifest = 3;
}
```

##### GetPluginInfo Errors

If the plugin is unable to complete the GetPluginInfo call successfully, it MUST return a non-ok gRPC code in the gRPC status.

#### `GetPluginCapabilities`

This REQUIRED RPC allows the CO to query the supported capabilities of the Plugin "as a whole": it is the grand sum of all capabilities of all instances of the Plugin software, as it is intended to be deployed.
All instances of the same version (see `vendor_version` of `GetPluginInfoResponse`) of the Plugin SHALL return the same set of capabilities, regardless of both: (a) where instances are deployed on the cluster as well as; (b) which RPCs an instance is serving.

```protobuf
message GetPluginCapabilitiesRequest {
  // Intentionally empty.
}

message GetPluginCapabilitiesResponse {
  // All the capabilities that the controller service supports. This
  // field is OPTIONAL.
  repeated PluginCapability capabilities = 1;
}

// Specifies a capability of the plugin.
message PluginCapability {
  message Service {
    enum Type {
      UNKNOWN = 0;
      // CONTROLLER_SERVICE indicates that the Plugin provides RPCs for
      // the ControllerService. Plugins SHOULD provide this capability.
      // In rare cases certain plugins MAY wish to omit the
      // ControllerService entirely from their implementation, but such
      // SHOULD NOT be the common case.
      // The presence of this capability determines whether the CO will
      // attempt to invoke the REQUIRED ControllerService RPCs, as well
      // as specific RPCs as indicated by ControllerGetCapabilities.
      CONTROLLER_SERVICE = 1;

      // VOLUME_ACCESSIBILITY_CONSTRAINTS indicates that the volumes for
      // this plugin MAY NOT be equally accessible by all nodes in the
      // cluster. The CO MUST use the topology information returned by
      // CreateVolumeRequest along with the topology information
      // returned by NodeGetInfo to ensure that a given volume is
      // accessible from a given node when scheduling workloads.
      VOLUME_ACCESSIBILITY_CONSTRAINTS = 2;

      // GROUP_CONTROLLER_SERVICE indicates that the Plugin provides
      // RPCs for operating on groups of volumes. Plugins MAY provide
      // this capability.
      // The presence of this capability determines whether the CO will
      // attempt to invoke the REQUIRED GroupController service RPCs, as
      // well as specific RPCs as indicated by
      // GroupControllerGetCapabilities.
      GROUP_CONTROLLER_SERVICE = 3;

      // SNAPSHOT_METADATA_SERVICE indicates that the Plugin provides
      // RPCs to retrieve metadata on the allocated blocks of a single
      // snapshot, or the changed blocks between a pair of snapshots of
      // the same block volume.
      // The presence of this capability determines whether the CO will
      // attempt to invoke the OPTIONAL SnapshotMetadata service RPCs.
      SNAPSHOT_METADATA_SERVICE = 4 [(alpha_enum_value) = true];
    }
    Type type = 1;
  }

  message VolumeExpansion {
    enum Type {
      UNKNOWN = 0;

      // ONLINE indicates that volumes may be expanded when published to
      // a node. When a Plugin implements this capability it MUST
      // implement either the EXPAND_VOLUME controller capability or the
      // EXPAND_VOLUME node capability or both. When a plugin supports
      // ONLINE volume expansion and also has the EXPAND_VOLUME
      // controller capability then the plugin MUST support expansion of
      // volumes currently published and available on a node. When a
      // plugin supports ONLINE volume expansion and also has the
      // EXPAND_VOLUME node capability then the plugin MAY support
      // expansion of node-published volume via NodeExpandVolume.
      //
      // Example 1: Given a shared filesystem volume (e.g. GlusterFs),
      //   the Plugin may set the ONLINE volume expansion capability and
      //   implement ControllerExpandVolume but not NodeExpandVolume.
      //
      // Example 2: Given a block storage volume type (e.g. EBS), the
      //   Plugin may set the ONLINE volume expansion capability and
      //   implement both ControllerExpandVolume and NodeExpandVolume.
      //
      // Example 3: Given a Plugin that supports volume expansion only
      //   upon a node, the Plugin may set the ONLINE volume
      //   expansion capability and implement NodeExpandVolume but not
      //   ControllerExpandVolume.
      ONLINE = 1;

      // OFFLINE indicates that volumes currently published and
      // available on a node SHALL NOT be expanded via
      // ControllerExpandVolume. When a plugin supports OFFLINE volume
      // expansion it MUST implement either the EXPAND_VOLUME controller
      // capability or both the EXPAND_VOLUME controller capability and
      // the EXPAND_VOLUME node capability.
      //
      // Example 1: Given a block storage volume type (e.g. Azure Disk)
      //   that does not support expansion of "node-attached" (i.e.
      //   controller-published) volumes, the Plugin may indicate
      //   OFFLINE volume expansion support and implement both
      //   ControllerExpandVolume and NodeExpandVolume.
      OFFLINE = 2;
    }
    Type type = 1;
  }

  oneof type {
    // Service that the plugin supports.
    Service service = 1;
    VolumeExpansion volume_expansion = 2;
  }
}
```

##### GetPluginCapabilities Errors

If the plugin is unable to complete the GetPluginCapabilities call successfully, it MUST return a non-ok gRPC code in the gRPC status.

#### `Probe`

A Plugin MUST implement this RPC call.
The primary utility of the Probe RPC is to verify that the plugin is in a healthy and ready state.
If an unhealthy state is reported, via a non-success response, a CO MAY take action with the intent to bring the plugin to a healthy state.
Such actions MAY include, but SHALL NOT be limited to, the following:

* Restarting the plugin container, or
* Notifying the plugin supervisor.

The Plugin MAY verify that it has the right configurations, devices, dependencies and drivers in order to run and return a success if the validation succeeds.
The CO MAY invoke this RPC at any time.
A CO MAY invoke this call multiple times with the understanding that a plugin's implementation MAY NOT be trivial and there MAY be overhead incurred by such repeated calls.
The SP SHALL document guidance and known limitations regarding a particular Plugin's implementation of this RPC.
For example, the SP MAY document the maximum frequency at which its Probe implementation SHOULD be called.

```protobuf
message ProbeRequest {
  // Intentionally empty.
}

message ProbeResponse {
  // Readiness allows a plugin to report its initialization status back
  // to the CO. Initialization for some plugins MAY be time consuming
  // and it is important for a CO to distinguish between the following
  // cases:
  //
  // 1) The plugin is in an unhealthy state and MAY need restarting. In
  //    this case a gRPC error code SHALL be returned.
  // 2) The plugin is still initializing, but is otherwise perfectly
  //    healthy. In this case a successful response SHALL be returned
  //    with a readiness value of `false`. Calls to the plugin's
  //    Controller and/or Node services MAY fail due to an incomplete
  //    initialization state.
  // 3) The plugin has finished initializing and is ready to service
  //    calls to its Controller and/or Node services. A successful
  //    response is returned with a readiness value of `true`.
  //
  // This field is OPTIONAL. If not present, the caller SHALL assume
  // that the plugin is in a ready state and is accepting calls to its
  // Controller and/or Node services (according to the plugin's reported
  // capabilities).
  .google.protobuf.BoolValue ready = 1;
}
```

##### Probe Errors

If the plugin is unable to complete the Probe call successfully, it MUST return a non-ok gRPC code in the gRPC status.
If the conditions defined below are encountered, the plugin MUST return the specified gRPC error code.
The CO MUST implement the specified error recovery behavior when it encounters the gRPC error code.

| Condition | gRPC Code | Description | Recovery Behavior |
|-----------|-----------|-------------|-------------------|
| Plugin not healthy | 9 FAILED_PRECONDITION | Indicates that the plugin is not in a healthy/ready state. | Caller SHOULD assume the plugin is not healthy and that future RPCs MAY fail because of this condition. |
| Missing required dependency | 9 FAILED_PRECONDITION | Indicates that the plugin is missing one or more required dependency. | Caller MUST assume the plugin is not healthy. |


### Controller Service RPC

#### `CreateVolume`

A Controller Plugin MUST implement this RPC call if it has `CREATE_DELETE_VOLUME` controller capability.
This RPC will be called by the CO to provision a new volume on behalf of a user (to be consumed as either a block device or a mounted filesystem).

This operation MUST be idempotent.
If a volume corresponding to the specified volume `name` already exists, is accessible from `accessibility_requirements`, and is compatible with the specified `capacity_range`, `volume_capabilities`, `parameters` and `mutable_parameters` in the `CreateVolumeRequest`, the Plugin MUST reply `0 OK` with the corresponding `CreateVolumeResponse`.

The `parameters` field SHALL contain opaque volume attributes to be specified at creation time. 
The `mutable_parameters` field SHALL contain opaque volume attributes that are defined at creation time but MAY also be changed during the lifetime of the volume via a subsequent `ControllerModifyVolume` RPC. 
Values specified in `mutable_parameters` MUST take precedence over the values from `parameters`.

Plugins MAY create 3 types of volumes:

- Empty volumes. When plugin supports `CREATE_DELETE_VOLUME` OPTIONAL capability.
- From an existing snapshot. When plugin supports `CREATE_DELETE_VOLUME` and `CREATE_DELETE_SNAPSHOT` OPTIONAL capabilities.
- From an existing volume. When plugin supports cloning, and reports the OPTIONAL capabilities `CREATE_DELETE_VOLUME` and `CLONE_VOLUME`.

If CO requests a volume to be created from existing snapshot or volume and the requested size of the volume is larger than the original snapshotted (or cloned volume), the Plugin can either refuse such a call with `OUT_OF_RANGE` error or MUST provide a volume that, when presented to a workload by `NodePublish` call, has both the requested (larger) size and contains data from the snapshot (or original volume).
Explicitly, it's the responsibility of the Plugin to resize the filesystem of the newly created volume at (or before) the `NodePublish` call, if the volume has `VolumeCapability` access type `MountVolume` and the filesystem resize is required in order to provision the requested capacity.

```protobuf
message CreateVolumeRequest {
  // The suggested name for the storage space. This field is REQUIRED.
  // It serves two purposes:
  // 1) Idempotency - This name is generated by the CO to achieve
  //    idempotency.  The Plugin SHOULD ensure that multiple
  //    `CreateVolume` calls for the same name do not result in more
  //    than one piece of storage provisioned corresponding to that
  //    name. If a Plugin is unable to enforce idempotency, the CO's
  //    error recovery logic could result in multiple (unused) volumes
  //    being provisioned.
  //    In the case of error, the CO MUST handle the gRPC error codes
  //    per the recovery behavior defined in the "CreateVolume Errors"
  //    section below.
  //    The CO is responsible for cleaning up volumes it provisioned
  //    that it no longer needs. If the CO is uncertain whether a volume
  //    was provisioned or not when a `CreateVolume` call fails, the CO
  //    MAY call `CreateVolume` again, with the same name, to ensure the
  //    volume exists and to retrieve the volume's `volume_id` (unless
  //    otherwise prohibited by "CreateVolume Errors").
  // 2) Suggested name - Some storage systems allow callers to specify
  //    an identifier by which to refer to the newly provisioned
  //    storage. If a storage system supports this, it can optionally
  //    use this name as the identifier for the new volume.
  // Any Unicode string that conforms to the length limit is allowed
  // except those containing the following banned characters:
  // U+0000-U+0008, U+000B, U+000C, U+000E-U+001F, U+007F-U+009F.
  // (These are control characters other than commonly used whitespace.)
  string name = 1;

  // This field is OPTIONAL. This allows the CO to specify the capacity
  // requirement of the volume to be provisioned. If not specified, the
  // Plugin MAY choose an implementation-defined capacity range. If
  // specified it MUST always be honored, even when creating volumes
  // from a source; which MAY force some backends to internally extend
  // the volume after creating it.
  CapacityRange capacity_range = 2;

  // The capabilities that the provisioned volume MUST have. SP MUST
  // provision a volume that will satisfy ALL of the capabilities
  // specified in this list. Otherwise SP MUST return the appropriate
  // gRPC error code.
  // The Plugin MUST assume that the CO MAY use the provisioned volume
  // with ANY of the capabilities specified in this list.
  // For example, a CO MAY specify two volume capabilities: one with
  // access mode SINGLE_NODE_WRITER and another with access mode
  // MULTI_NODE_READER_ONLY. In this case, the SP MUST verify that the
  // provisioned volume can be used in either mode.
  // This also enables the CO to do early validation: If ANY of the
  // specified volume capabilities are not supported by the SP, the call
  // MUST return the appropriate gRPC error code.
  // This field is REQUIRED.
  repeated VolumeCapability volume_capabilities = 3;

  // Plugin specific creation-time parameters passed in as opaque 
  // key-value pairs. This field is OPTIONAL. The Plugin is responsible 
  // for parsing and validating these parameters. COs will treat 
  // these as opaque.
  map<string, string> parameters = 4;

  // Secrets required by plugin to complete volume creation request.
  // This field is OPTIONAL. Refer to the `Secrets Requirements`
  // section on how to use this field.
  map<string, string> secrets = 5 [(csi_secret) = true];

  // If specified, the new volume will be pre-populated with data from
  // this source. This field is OPTIONAL.
  VolumeContentSource volume_content_source = 6;

  // Specifies where (regions, zones, racks, etc.) the provisioned
  // volume MUST be accessible from.
  // An SP SHALL advertise the requirements for topological
  // accessibility information in documentation. COs SHALL only specify
  // topological accessibility information supported by the SP.
  // This field is OPTIONAL.
  // This field SHALL NOT be specified unless the SP has the
  // VOLUME_ACCESSIBILITY_CONSTRAINTS plugin capability.
  // If this field is not specified and the SP has the
  // VOLUME_ACCESSIBILITY_CONSTRAINTS plugin capability, the SP MAY
  // choose where the provisioned volume is accessible from.
  TopologyRequirement accessibility_requirements = 7;

  // Plugin specific creation-time parameters passed in as opaque 
  // key-value pairs. These mutable_parameteres MAY also be
  // changed during the lifetime of the volume via a subsequent
  // `ControllerModifyVolume` RPC. This field is OPTIONAL.
  // The Plugin is responsible for parsing and validating these
  // parameters. COs will treat these as opaque.
  
  // Plugins MUST treat these 
  // as if they take precedence over the parameters field.
  // This field SHALL NOT be specified unless the SP has the
  // MODIFY_VOLUME plugin capability.
  map<string, string> mutable_parameters = 8 [(alpha_field) = true];
}

// Specifies what source the volume will be created from. One of the
// type fields MUST be specified.
message VolumeContentSource {
  message SnapshotSource {
    // Contains identity information for the existing source snapshot.
    // This field is REQUIRED. Plugin is REQUIRED to support creating
    // volume from snapshot if it supports the capability
    // CREATE_DELETE_SNAPSHOT.
    string snapshot_id = 1;
  }

  message VolumeSource {
    // Contains identity information for the existing source volume.
    // This field is REQUIRED. Plugins reporting CLONE_VOLUME
    // capability MUST support creating a volume from another volume.
    string volume_id = 1;
  }

  oneof type {
    SnapshotSource snapshot = 1;
    VolumeSource volume = 2;
  }
}

message CreateVolumeResponse {
  // Contains all attributes of the newly created volume that are
  // relevant to the CO along with information required by the Plugin
  // to uniquely identify the volume. This field is REQUIRED.
  Volume volume = 1;
}

// Specify a capability of a volume.
message VolumeCapability {
  // Indicate that the volume will be accessed via the block device API.
  message BlockVolume {
    // Intentionally empty, for now.
  }

  // Indicate that the volume will be accessed via the filesystem API.
  message MountVolume {
    // The filesystem type. This field is OPTIONAL.
    // An empty string is equal to an unspecified field value.
    string fs_type = 1;

    // The mount options that can be used for the volume. This field is
    // OPTIONAL. `mount_flags` MAY contain sensitive information.
    // Therefore, the CO and the Plugin MUST NOT leak this information
    // to untrusted entities. The total size of this repeated field
    // SHALL NOT exceed 4 KiB.
    repeated string mount_flags = 2;

    // If SP has VOLUME_MOUNT_GROUP node capability and CO provides
    // this field then SP MUST ensure that the volume_mount_group
    // parameter is passed as the group identifier to the underlying
    // operating system mount system call, with the understanding
    // that the set of available mount call parameters and/or
    // mount implementations may vary across operating systems.
    // Additionally, new file and/or directory entries written to
    // the underlying filesystem SHOULD be permission-labeled in such a
    // manner, unless otherwise modified by a workload, that they are
    // both readable and writable by said mount group identifier.
    // This is an OPTIONAL field.
    string volume_mount_group = 3;
  }

  // Specify how a volume can be accessed.
  message AccessMode {
    enum Mode {
      UNKNOWN = 0;

      // Can only be published once as read/write on a single node, at
      // any given time.
      SINGLE_NODE_WRITER = 1;

      // Can only be published once as readonly on a single node, at
      // any given time.
      SINGLE_NODE_READER_ONLY = 2;

      // Can be published as readonly at multiple nodes simultaneously.
      MULTI_NODE_READER_ONLY = 3;

      // Can be published at multiple nodes simultaneously. Only one of
      // the node can be used as read/write. The rest will be readonly.
      MULTI_NODE_SINGLE_WRITER = 4;

      // Can be published as read/write at multiple nodes
      // simultaneously.
      MULTI_NODE_MULTI_WRITER = 5;

      // Can only be published once as read/write at a single workload
      // on a single node, at any given time. SHOULD be used instead of
      // SINGLE_NODE_WRITER for COs using the experimental
      // SINGLE_NODE_MULTI_WRITER capability.
      SINGLE_NODE_SINGLE_WRITER = 6 [(alpha_enum_value) = true];

      // Can be published as read/write at multiple workloads on a
      // single node simultaneously. SHOULD be used instead of
      // SINGLE_NODE_WRITER for COs using the experimental
      // SINGLE_NODE_MULTI_WRITER capability.
      SINGLE_NODE_MULTI_WRITER = 7 [(alpha_enum_value) = true];
    }

    // This field is REQUIRED.
    Mode mode = 1;
  }

  // Specifies what API the volume will be accessed using. One of the
  // following fields MUST be specified.
  oneof access_type {
    BlockVolume block = 1;
    MountVolume mount = 2;
  }

  // This is a REQUIRED field.
  AccessMode access_mode = 3;
}

// The capacity of the storage space in bytes. To specify an exact size,
// `required_bytes` and `limit_bytes` SHALL be set to the same value. At
// least one of the these fields MUST be specified.
message CapacityRange {
  // Volume MUST be at least this big. This field is OPTIONAL.
  // A value of 0 is equal to an unspecified field value.
  // The value of this field MUST NOT be negative.
  int64 required_bytes = 1;

  // Volume MUST not be bigger than this. This field is OPTIONAL.
  // A value of 0 is equal to an unspecified field value.
  // The value of this field MUST NOT be negative.
  int64 limit_bytes = 2;
}

// Information about a specific volume.
message Volume {
  // The capacity of the volume in bytes. This field is OPTIONAL. If not
  // set (value of 0), it indicates that the capacity of the volume is
  // unknown (e.g., NFS share).
  // The value of this field MUST NOT be negative.
  int64 capacity_bytes = 1;

  // The identifier for this volume, generated by the plugin.
  // This field is REQUIRED.
  // This field MUST contain enough information to uniquely identify
  // this specific volume vs all other volumes supported by this plugin.
  // This field SHALL be used by the CO in subsequent calls to refer to
  // this volume.
  // The SP is NOT responsible for global uniqueness of volume_id across
  // multiple SPs.
  string volume_id = 2;

  // Opaque static properties of the volume. SP MAY use this field to
  // ensure subsequent volume validation and publishing calls have
  // contextual information.
  // The contents of this field SHALL be opaque to a CO.
  // The contents of this field SHALL NOT be mutable.
  // The contents of this field SHALL be safe for the CO to cache.
  // The contents of this field SHOULD NOT contain sensitive
  // information.
  // The contents of this field SHOULD NOT be used for uniquely
  // identifying a volume. The `volume_id` alone SHOULD be sufficient to
  // identify the volume.
  // A volume uniquely identified by `volume_id` SHALL always report the
  // same volume_context.
  // This field is OPTIONAL and when present MUST be passed to volume
  // validation and publishing calls.
  map<string, string> volume_context = 3;

  // If specified, indicates that the volume is not empty and is
  // pre-populated with data from the specified source.
  // This field is OPTIONAL.
  VolumeContentSource content_source = 4;

  // Specifies where (regions, zones, racks, etc.) the provisioned
  // volume is accessible from.
  // A plugin that returns this field MUST also set the
  // VOLUME_ACCESSIBILITY_CONSTRAINTS plugin capability.
  // An SP MAY specify multiple topologies to indicate the volume is
  // accessible from multiple locations.
  // COs MAY use this information along with the topology information
  // returned by NodeGetInfo to ensure that a given volume is accessible
  // from a given node when scheduling workloads.
  // This field is OPTIONAL. If it is not specified, the CO MAY assume
  // the volume is equally accessible from all nodes in the cluster and
  // MAY schedule workloads referencing the volume on any available
  // node.
  //
  // Example 1:
  //   accessible_topology = {"region": "R1", "zone": "Z2"}
  // Indicates a volume accessible only from the "region" "R1" and the
  // "zone" "Z2".
  //
  // Example 2:
  //   accessible_topology =
  //     {"region": "R1", "zone": "Z2"},
  //     {"region": "R1", "zone": "Z3"}
  // Indicates a volume accessible from both "zone" "Z2" and "zone" "Z3"
  // in the "region" "R1".
  repeated Topology accessible_topology = 5;
}

message TopologyRequirement {
  // Specifies the list of topologies the provisioned volume MUST be
  // accessible from.
  // This field is OPTIONAL. If TopologyRequirement is specified either
  // requisite or preferred or both MUST be specified.
  //
  // If requisite is specified, the provisioned volume MUST be
  // accessible from at least one of the requisite topologies.
  //
  // Given
  //   x = number of topologies provisioned volume is accessible from
  //   n = number of requisite topologies
  // The CO MUST ensure n >= 1. The SP MUST ensure x >= 1
  // If x==n, then the SP MUST make the provisioned volume available to
  // all topologies from the list of requisite topologies. If it is
  // unable to do so, the SP MUST fail the CreateVolume call.
  // For example, if a volume should be accessible from a single zone,
  // and requisite =
  //   {"region": "R1", "zone": "Z2"}
  // then the provisioned volume MUST be accessible from the "region"
  // "R1" and the "zone" "Z2".
  // Similarly, if a volume should be accessible from two zones, and
  // requisite =
  //   {"region": "R1", "zone": "Z2"},
  //   {"region": "R1", "zone": "Z3"}
  // then the provisioned volume MUST be accessible from the "region"
  // "R1" and both "zone" "Z2" and "zone" "Z3".
  //
  // If x<n, then the SP SHALL choose x unique topologies from the list
  // of requisite topologies. If it is unable to do so, the SP MUST fail
  // the CreateVolume call.
  // For example, if a volume should be accessible from a single zone,
  // and requisite =
  //   {"region": "R1", "zone": "Z2"},
  //   {"region": "R1", "zone": "Z3"}
  // then the SP may choose to make the provisioned volume available in
  // either the "zone" "Z2" or the "zone" "Z3" in the "region" "R1".
  // Similarly, if a volume should be accessible from two zones, and
  // requisite =
  //   {"region": "R1", "zone": "Z2"},
  //   {"region": "R1", "zone": "Z3"},
  //   {"region": "R1", "zone": "Z4"}
  // then the provisioned volume MUST be accessible from any combination
  // of two unique topologies: e.g. "R1/Z2" and "R1/Z3", or "R1/Z2" and
  //  "R1/Z4", or "R1/Z3" and "R1/Z4".
  //
  // If x>n, then the SP MUST make the provisioned volume available from
  // all topologies from the list of requisite topologies and MAY choose
  // the remaining x-n unique topologies from the list of all possible
  // topologies. If it is unable to do so, the SP MUST fail the
  // CreateVolume call.
  // For example, if a volume should be accessible from two zones, and
  // requisite =
  //   {"region": "R1", "zone": "Z2"}
  // then the provisioned volume MUST be accessible from the "region"
  // "R1" and the "zone" "Z2" and the SP may select the second zone
  // independently, e.g. "R1/Z4".
  repeated Topology requisite = 1;

  // Specifies the list of topologies the CO would prefer the volume to
  // be provisioned in.
  //
  // This field is OPTIONAL. If TopologyRequirement is specified either
  // requisite or preferred or both MUST be specified.
  //
  // An SP MUST attempt to make the provisioned volume available using
  // the preferred topologies in order from first to last.
  //
  // If requisite is specified, all topologies in preferred list MUST
  // also be present in the list of requisite topologies.
  //
  // If the SP is unable to to make the provisioned volume available
  // from any of the preferred topologies, the SP MAY choose a topology
  // from the list of requisite topologies.
  // If the list of requisite topologies is not specified, then the SP
  // MAY choose from the list of all possible topologies.
  // If the list of requisite topologies is specified and the SP is
  // unable to to make the provisioned volume available from any of the
  // requisite topologies it MUST fail the CreateVolume call.
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

##### CreateVolume Errors

If the plugin is unable to complete the CreateVolume call successfully, it MUST return a non-ok gRPC code in the gRPC status.
If the conditions defined below are encountered, the plugin MUST return the specified gRPC error code.
The CO MUST implement the specified error recovery behavior when it encounters the gRPC error code.

| Condition | gRPC Code | Description | Recovery Behavior |
|-----------|-----------|-------------|-------------------|
| Source incompatible or not supported | 3 INVALID_ARGUMENT | Besides the general cases, this code MUST also be used to indicate when plugin supporting CREATE_DELETE_VOLUME cannot create a volume from the requested source (`SnapshotSource` or `VolumeSource`). Failure MAY be caused by not supporting the source (CO SHOULD NOT have provided that source) or incompatibility between `parameters` from the source and the ones requested for the new volume. More human-readable information SHOULD be provided in the gRPC `status.message` field if the problem is the source. | On source related issues, caller MUST use different parameters, a different source, or no source at all. |
| Source does not exist | 5 NOT_FOUND | Indicates that the specified source does not exist. | Caller MUST verify that the `volume_content_source` is correct, the source is accessible, and has not been deleted before retrying with exponential back off. |
| Volume already exists but is incompatible | 6 ALREADY_EXISTS | Indicates that a volume corresponding to the specified volume `name` already exists but is incompatible with the specified `capacity_range`, `volume_capabilities`, `parameters`, `accessibility_requirements` or `volume_content_source`. | Caller MUST fix the arguments or use a different `name` before retrying. |
| Unable to provision in `accessible_topology` | 8 RESOURCE_EXHAUSTED | Indicates that although the `accessible_topology` field is valid, a new volume can not be provisioned with the specified topology constraints. More human-readable information MAY be provided in the gRPC `status.message` field. | Caller MUST ensure that whatever is preventing volumes from being provisioned in the specified location (e.g. quota issues) is addressed before retrying with exponential backoff. |
| Unsupported `capacity_range` | 11 OUT_OF_RANGE | Indicates that the capacity range is not allowed by the Plugin, for example when trying to create a volume smaller than the source snapshot or the Plugin does not support creating volumes larger than the source snapshot or source volume. More human-readable information MAY be provided in the gRPC `status.message` field. | Caller MUST fix the capacity range before retrying. |


#### `DeleteVolume`

A Controller Plugin MUST implement this RPC call if it has `CREATE_DELETE_VOLUME` capability.
This RPC will be called by the CO to deprovision a volume.

This operation MUST be idempotent.
If a volume corresponding to the specified `volume_id` does not exist or the artifacts associated with the volume do not exist anymore, the Plugin MUST reply `0 OK`.

CSI plugins SHOULD treat volumes independent from their snapshots.

If the Controller Plugin supports deleting a volume without affecting its existing snapshots, then these snapshots MUST still be fully operational and acceptable as sources for new volumes as well as appear on `ListSnapshot` calls once the volume has been deleted.

When a Controller Plugin does not support deleting a volume without affecting its existing snapshots, then the volume MUST NOT be altered in any way by the request and the operation must return the `FAILED_PRECONDITION` error code and MAY include meaningful human-readable information in the `status.message` field.

```protobuf
message DeleteVolumeRequest {
  // The ID of the volume to be deprovisioned.
  // This field is REQUIRED.
  string volume_id = 1;

  // Secrets required by plugin to complete volume deletion request.
  // This field is OPTIONAL. Refer to the `Secrets Requirements`
  // section on how to use this field.
  map<string, string> secrets = 2 [(csi_secret) = true];
}

message DeleteVolumeResponse {
  // Intentionally empty.
}
```

##### DeleteVolume Errors

If the plugin is unable to complete the DeleteVolume call successfully, it MUST return a non-ok gRPC code in the gRPC status.
If the conditions defined below are encountered, the plugin MUST return the specified gRPC error code.
The CO MUST implement the specified error recovery behavior when it encounters the gRPC error code.

| Condition | gRPC Code | Description | Recovery Behavior |
|-----------|-----------|-------------|-------------------|
| Volume in use | 9 FAILED_PRECONDITION | Indicates that the volume corresponding to the specified `volume_id` could not be deleted because it is in use by another resource or has snapshots and the plugin doesn't treat them as independent entities. | Caller SHOULD ensure that there are no other resources using the volume and that it has no snapshots, and then retry with exponential back off. |


#### `ControllerPublishVolume`

A Controller Plugin MUST implement this RPC call if it has `PUBLISH_UNPUBLISH_VOLUME` controller capability.
This RPC will be called by the CO when it wants to place a workload that uses the volume onto a node.
The Plugin SHOULD perform the work that is necessary for making the volume available on the given node.
The Plugin MUST NOT assume that this RPC will be executed on the node where the volume will be used.

This operation MUST be idempotent.
If the volume corresponding to the `volume_id` has already been published at the node corresponding to the `node_id`, and is compatible with the specified `volume_capability` and `readonly` flag, the Plugin MUST reply `0 OK`.

If the operation failed or the CO does not know if the operation has failed or not, it MAY choose to call `ControllerPublishVolume` again or choose to call `ControllerUnpublishVolume`.

The CO MAY call this RPC for publishing a volume to multiple nodes if the volume has `MULTI_NODE` capability (i.e., `MULTI_NODE_READER_ONLY`, `MULTI_NODE_SINGLE_WRITER` or `MULTI_NODE_MULTI_WRITER`).

```protobuf
message ControllerPublishVolumeRequest {
  // The ID of the volume to be used on a node.
  // This field is REQUIRED.
  string volume_id = 1;

  // The ID of the node. This field is REQUIRED. The CO SHALL set this
  // field to match the node ID returned by `NodeGetInfo`.
  string node_id = 2;

  // Volume capability describing how the CO intends to use this volume.
  // SP MUST ensure the CO can use the published volume as described.
  // Otherwise SP MUST return the appropriate gRPC error code.
  // This is a REQUIRED field.
  VolumeCapability volume_capability = 3;

  // Indicates SP MUST publish the volume in readonly mode.
  // CO MUST set this field to false if SP does not have the
  // PUBLISH_READONLY controller capability.
  // This is a REQUIRED field.
  bool readonly = 4;

  // Secrets required by plugin to complete controller publish volume
  // request. This field is OPTIONAL. Refer to the
  // `Secrets Requirements` section on how to use this field.
  map<string, string> secrets = 5 [(csi_secret) = true];

  // Volume context as returned by SP in
  // CreateVolumeResponse.Volume.volume_context.
  // This field is OPTIONAL and MUST match the volume_context of the
  // volume identified by `volume_id`.
  map<string, string> volume_context = 6;
}

message ControllerPublishVolumeResponse {
  // Opaque static publish properties of the volume. SP MAY use this
  // field to ensure subsequent `NodeStageVolume` or `NodePublishVolume`
  // calls calls have contextual information.
  // The contents of this field SHALL be opaque to a CO.
  // The contents of this field SHALL NOT be mutable.
  // The contents of this field SHALL be safe for the CO to cache.
  // The contents of this field SHOULD NOT contain sensitive
  // information.
  // The contents of this field SHOULD NOT be used for uniquely
  // identifying a volume. The `volume_id` alone SHOULD be sufficient to
  // identify the volume.
  // This field is OPTIONAL and when present MUST be passed to
  // subsequent `NodeStageVolume` or `NodePublishVolume` calls
  map<string, string> publish_context = 1;
}
```

##### ControllerPublishVolume Errors

If the plugin is unable to complete the ControllerPublishVolume call successfully, it MUST return a non-ok gRPC code in the gRPC status.
If the conditions defined below are encountered, the plugin MUST return the specified gRPC error code.
The CO MUST implement the specified error recovery behavior when it encounters the gRPC error code.

| Condition | gRPC Code | Description | Recovery Behavior |
|-----------|-----------|-------------|-------------------|
| Volume does not exist | 5 NOT_FOUND | Indicates that a volume corresponding to the specified `volume_id` does not exist. | Caller MUST verify that the `volume_id` is correct and that the volume is accessible and has not been deleted before retrying with exponential back off. |
| Node does not exist | 5 NOT_FOUND | Indicates that a node corresponding to the specified `node_id` does not exist. | Caller MUST verify that the `node_id` is correct and that the node is available and has not been terminated or deleted before retrying with exponential backoff. |
| Volume published but is incompatible | 6 ALREADY_EXISTS | Indicates that a volume corresponding to the specified `volume_id` has already been published at the node corresponding to the specified `node_id` but is incompatible with the specified `volume_capability` or `readonly` flag . | Caller MUST fix the arguments before retrying. |
| Volume published to another node | 9 FAILED_PRECONDITION | Indicates that a volume corresponding to the specified `volume_id` has already been published at another node and does not have MULTI_NODE volume capability. If this error code is returned, the Plugin SHOULD specify the `node_id` of the node at which the volume is published as part of the gRPC `status.message`. | Caller SHOULD ensure the specified volume is not published at any other node before retrying with exponential back off. |
| Max volumes attached | 8 RESOURCE_EXHAUSTED | Indicates that the maximum supported number of volumes that can be attached to the specified node are already attached. Therefore, this operation will fail until at least one of the existing attached volumes is detached from the node. | Caller MUST ensure that the number of volumes already attached to the node is less then the maximum supported number of volumes before retrying with exponential backoff. |

#### `ControllerUnpublishVolume`

Controller Plugin MUST implement this RPC call if it has `PUBLISH_UNPUBLISH_VOLUME` controller capability.
This RPC is a reverse operation of `ControllerPublishVolume`.
It MUST be called after all `NodeUnstageVolume` and `NodeUnpublishVolume` on the volume are called and succeed.
The Plugin SHOULD perform the work that is necessary for making the volume ready to be consumed by a different node.
The Plugin MUST NOT assume that this RPC will be executed on the node where the volume was previously used.

This RPC is typically called by the CO when the workload using the volume is being moved to a different node, or all the workload using the volume on a node has finished.

This operation MUST be idempotent.
If the volume corresponding to the `volume_id` is not attached to the node corresponding to the `node_id`, the Plugin MUST reply `0 OK`.
If the volume corresponding to the `volume_id` or the node corresponding to `node_id` cannot be found by the Plugin and the volume can be safely regarded as ControllerUnpublished from the node, the plugin SHOULD return `0 OK`.
If this operation failed, or the CO does not know if the operation failed or not, it can choose to call `ControllerUnpublishVolume` again.

```protobuf
message ControllerUnpublishVolumeRequest {
  // The ID of the volume. This field is REQUIRED.
  string volume_id = 1;

  // The ID of the node. This field is OPTIONAL. The CO SHOULD set this
  // field to match the node ID returned by `NodeGetInfo` or leave it
  // unset. If the value is set, the SP MUST unpublish the volume from
  // the specified node. If the value is unset, the SP MUST unpublish
  // the volume from all nodes it is published to.
  string node_id = 2;

  // Secrets required by plugin to complete controller unpublish volume
  // request. This SHOULD be the same secrets passed to the
  // ControllerPublishVolume call for the specified volume.
  // This field is OPTIONAL. Refer to the `Secrets Requirements`
  // section on how to use this field.
  map<string, string> secrets = 3 [(csi_secret) = true];
}

message ControllerUnpublishVolumeResponse {
  // Intentionally empty.
}
```

##### ControllerUnpublishVolume Errors

If the plugin is unable to complete the ControllerUnpublishVolume call successfully, it MUST return a non-ok gRPC code in the gRPC status.
If the conditions defined below are encountered, the plugin MUST return the specified gRPC error code.
The CO MUST implement the specified error recovery behavior when it encounters the gRPC error code.

| Condition | gRPC Code | Description | Recovery Behavior |
|-----------|-----------|-------------|-------------------|
| Volume does not exist and volume not assumed ControllerUnpublished from node | 5 NOT_FOUND | Indicates that a volume corresponding to the specified `volume_id` does not exist and is not assumed to be ControllerUnpublished from node corresponding to the specified `node_id`. | Caller SHOULD verify that the `volume_id` is correct and that the volume is accessible and has not been deleted before retrying with exponential back off. |
| Node does not exist and volume not assumed ControllerUnpublished from node  | 5 NOT_FOUND | Indicates that a node corresponding to the specified `node_id` does not exist and the volume corresponding to the specified `volume_id` is not assumed to be ControllerUnpublished from node. | Caller SHOULD verify that the `node_id` is correct and that the node is available and has not been terminated or deleted before retrying with exponential backoff. |


#### `ValidateVolumeCapabilities`

A Controller Plugin MUST implement this RPC call.
This RPC will be called by the CO to check if a pre-provisioned volume has all the capabilities that the CO wants.
This RPC call SHALL return `confirmed` only if all the volume capabilities specified in the request are supported (see caveat below).
This operation MUST be idempotent.

NOTE: Older plugins will parse but likely not "process" newer fields that MAY be present in capability-validation messages (and sub-messages) sent by a CO that is communicating using a newer, backwards-compatible version of the CSI protobufs.
Therefore, the CO SHALL reconcile successful capability-validation responses by comparing the validated capabilities with those that it had originally requested.

```protobuf
message ValidateVolumeCapabilitiesRequest {
  // The ID of the volume to check. This field is REQUIRED.
  string volume_id = 1;

  // Volume context as returned by SP in
  // CreateVolumeResponse.Volume.volume_context.
  // This field is OPTIONAL and MUST match the volume_context of the
  // volume identified by `volume_id`.
  map<string, string> volume_context = 2;

  // The capabilities that the CO wants to check for the volume. This
  // call SHALL return "confirmed" only if all the volume capabilities
  // specified below are supported. This field is REQUIRED.
  repeated VolumeCapability volume_capabilities = 3;

  // See CreateVolumeRequest.parameters.
  // This field is OPTIONAL.
  map<string, string> parameters = 4;

  // Secrets required by plugin to complete volume validation request.
  // This field is OPTIONAL. Refer to the `Secrets Requirements`
  // section on how to use this field.
  map<string, string> secrets = 5 [(csi_secret) = true];

  // See CreateVolumeRequest.mutable_parameters.
  // This field is OPTIONAL.
  map<string, string> mutable_parameters = 6 [(alpha_field) = true];
}

message ValidateVolumeCapabilitiesResponse {
  message Confirmed {
    // Volume context validated by the plugin.
    // This field is OPTIONAL.
    map<string, string> volume_context = 1;

    // Volume capabilities supported by the plugin.
    // This field is REQUIRED.
    repeated VolumeCapability volume_capabilities = 2;

    // The volume creation parameters validated by the plugin.
    // This field is OPTIONAL.
    map<string, string> parameters = 3;

    // The volume creation mutable_parameters validated by the plugin.
    // This field is OPTIONAL.
    map<string, string> mutable_parameters = 4 [(alpha_field) = true];
  }

  // Confirmed indicates to the CO the set of capabilities that the
  // plugin has validated. This field SHALL only be set to a non-empty
  // value for successful validation responses.
  // For successful validation responses, the CO SHALL compare the
  // fields of this message to the originally requested capabilities in
  // order to guard against an older plugin reporting "valid" for newer
  // capability fields that it does not yet understand.
  // This field is OPTIONAL.
  Confirmed confirmed = 1;

  // Message to the CO if `confirmed` above is empty. This field is
  // OPTIONAL.
  // An empty string is equal to an unspecified field value.
  string message = 2;
}
```

##### ValidateVolumeCapabilities Errors

If the plugin is unable to complete the ValidateVolumeCapabilities call successfully, it MUST return a non-ok gRPC code in the gRPC status.
If the conditions defined below are encountered, the plugin MUST return the specified gRPC error code.
The CO MUST implement the specified error recovery behavior when it encounters the gRPC error code.

| Condition | gRPC Code | Description | Recovery Behavior |
|-----------|-----------|-------------|-------------------|
| Volume does not exist | 5 NOT_FOUND | Indicates that a volume corresponding to the specified `volume_id` does not exist. | Caller MUST verify that the `volume_id` is correct and that the volume is accessible and has not been deleted before retrying with exponential back off. |


#### `ListVolumes`

A Controller Plugin MUST implement this RPC call if it has `LIST_VOLUMES` capability.
The Plugin SHALL return the information about all the volumes that it knows about.
If volumes are created and/or deleted while the CO is concurrently paging through `ListVolumes` results then it is possible that the CO MAY either witness duplicate volumes in the list, not witness existing volumes, or both.
The CO SHALL NOT expect a consistent "view" of all volumes when paging through the volume list via multiple calls to `ListVolumes`.

```protobuf
message ListVolumesRequest {
  // If specified (non-zero value), the Plugin MUST NOT return more
  // entries than this number in the response. If the actual number of
  // entries is more than this number, the Plugin MUST set `next_token`
  // in the response which can be used to get the next page of entries
  // in the subsequent `ListVolumes` call. This field is OPTIONAL. If
  // not specified (zero value), it means there is no restriction on the
  // number of entries that can be returned.
  // The value of this field MUST NOT be negative.
  int32 max_entries = 1;

  // A token to specify where to start paginating. Set this field to
  // `next_token` returned by a previous `ListVolumes` call to get the
  // next page of entries. This field is OPTIONAL.
  // An empty string is equal to an unspecified field value.
  string starting_token = 2;
}

message ListVolumesResponse {
  message VolumeStatus{
    // A list of all `node_id` of nodes that the volume in this entry
    // is controller published on.
    // This field is OPTIONAL. If it is not specified and the SP has
    // the LIST_VOLUMES_PUBLISHED_NODES controller capability, the CO
    // MAY assume the volume is not controller published to any nodes.
    // If the field is not specified and the SP does not have the
    // LIST_VOLUMES_PUBLISHED_NODES controller capability, the CO MUST
    // not interpret this field.
    // published_node_ids MAY include nodes not published to or
    // reported by the SP. The CO MUST be resilient to that.
    repeated string published_node_ids = 1;

    // Information about the current condition of the volume.
    // This field is OPTIONAL.
    // This field MUST be specified if the
    // VOLUME_CONDITION controller capability is supported.
    VolumeCondition volume_condition = 2 [(alpha_field) = true];
  }

  message Entry {
    // This field is REQUIRED
    Volume volume = 1;

    // This field is OPTIONAL. This field MUST be specified if the
    // LIST_VOLUMES_PUBLISHED_NODES controller capability is
    // supported.
    VolumeStatus status = 2;
  }

  repeated Entry entries = 1;

  // This token allows you to get the next page of entries for
  // `ListVolumes` request. If the number of entries is larger than
  // `max_entries`, use the `next_token` as a value for the
  // `starting_token` field in the next `ListVolumes` request. This
  // field is OPTIONAL.
  // An empty string is equal to an unspecified field value.
  string next_token = 2;
}
```

##### ListVolumes Errors

If the plugin is unable to complete the ListVolumes call successfully, it MUST return a non-ok gRPC code in the gRPC status.
If the conditions defined below are encountered, the plugin MUST return the specified gRPC error code.
The CO MUST implement the specified error recovery behavior when it encounters the gRPC error code.

| Condition | gRPC Code | Description | Recovery Behavior |
|-----------|-----------|-------------|-------------------|
| Invalid `starting_token` | 10 ABORTED | Indicates that `starting_token` is not valid. | Caller SHOULD start the `ListVolumes` operation again with an empty `starting_token`. |

#### `ControllerGetVolume`

**ALPHA FEATURE**

This optional RPC MAY be called by the CO to fetch current information about a volume.

A Controller Plugin MUST implement this `ControllerGetVolume` RPC call if it has `GET_VOLUME` capability.

A Controller Plugin MUST provide a non-empty `volume_condition` field in `ControllerGetVolumeResponse` if it has `VOLUME_CONDITION` capability.

`ControllerGetVolumeResponse` should contain current information of a volume if it exists.
If the volume does not exist any more, `ControllerGetVolume` should return gRPC error code `NOT_FOUND`.

```protobuf
message ControllerGetVolumeRequest {
  option (alpha_message) = true;

  // The ID of the volume to fetch current volume information for.
  // This field is REQUIRED.
  string volume_id = 1;
}

message ControllerGetVolumeResponse {
  option (alpha_message) = true;

  message VolumeStatus{
    // A list of all the `node_id` of nodes that this volume is
    // controller published on.
    // This field is OPTIONAL.
    // This field MUST be specified if the LIST_VOLUMES_PUBLISHED_NODES
    // controller capability is supported.
    // published_node_ids MAY include nodes not published to or
    // reported by the SP. The CO MUST be resilient to that.
    repeated string published_node_ids = 1;

    // Information about the current condition of the volume.
    // This field is OPTIONAL.
    // This field MUST be specified if the
    // VOLUME_CONDITION controller capability is supported.
    VolumeCondition volume_condition = 2;
  }

  // This field is REQUIRED
  Volume volume = 1;

  // This field is REQUIRED.
  VolumeStatus status = 2;
}
```

##### ControllerGetVolume Errors

If the plugin is unable to complete the ControllerGetVolume call successfully, it MUST return a non-ok gRPC code in the gRPC status.
If the conditions defined below are encountered, the plugin MUST return the specified gRPC error code.
The CO MUST implement the specified error recovery behavior when it encounters the gRPC error code.

| Condition | gRPC Code | Description | Recovery Behavior |
|-----------|-----------|-------------|-------------------|
| Volume does not exist | 5 NOT_FOUND | Indicates that a volume corresponding to the specified `volume_id` does not exist. | Caller MUST verify that the `volume_id` is correct and that the volume is accessible and has not been deleted before retrying with exponential back off. |


#### `ControllerModifyVolume`

A Controller plugin MUST implement this RPC call if the plugin has the MODIFY_VOLUME controller capability.
This RPC allows the CO to change mutable key attributes of a volume. 

This operation MUST be idempotent. 
The new mutable parameters in ControllerModifyVolume can be different from the existing mutable parameters. 


```protobuf
message ControllerModifyVolumeRequest {
  option (alpha_message) = true;

  // Contains identity information for the existing volume.
  // This field is REQUIRED.
  string volume_id = 1;

  // Secrets required by plugin to complete modify volume request.
  // This field is OPTIONAL. Refer to the `Secrets Requirements`
  // section on how to use this field.
  map<string, string> secrets = 2 [(csi_secret) = true];

  // Plugin specific volume attributes to mutate, passed in as 
  // opaque key-value pairs. 
  // This field is REQUIRED. The Plugin is responsible for
  // parsing and validating these parameters. COs will treat these
  // as opaque. The CO SHOULD specify the intended values of all mutable
  // parameters it intends to modify. SPs MUST NOT modify volumes based
  // on the absence of keys, only keys that are specified should result
  // in modifications to the volume.
  map<string, string> mutable_parameters = 3;
}

message ControllerModifyVolumeResponse {
  option (alpha_message) = true;
}

```

##### ControllerModifyVolume Errors

| Condition | gRPC Code | Description | Recovery Behavior |
|-----------|-----------|-------------|-------------------|
| Parameters not supported | 3 INVALID_ARGUMENT | Indicates that the CO has specified mutable parameters not supported by the volume. | Caller MAY verify mutable parameters. |
| Exceeds capabilities | 3 INVALID_ARGUMENT | Indicates that the CO has specified capabilities not supported by the volume. | Caller MAY verify volume capabilities by calling ValidateVolumeCapabilities and retry with matching capabilities. |
| Volume does not exist | 5 NOT_FOUND | Indicates that a volume corresponding to the specified volume_id does not exist. | Caller MUST verify that the volume_id is correct and that the volume is accessible and has not been deleted before retrying with exponential back off. |

#### `GetCapacity`

A Controller Plugin MUST implement this RPC call if it has `GET_CAPACITY` controller capability.
The RPC allows the CO to query the capacity of the storage pool from which the controller provisions volumes.

```protobuf
message GetCapacityRequest {
  // If specified, the Plugin SHALL report the capacity of the storage
  // that can be used to provision volumes that satisfy ALL of the
  // specified `volume_capabilities`. These are the same
  // `volume_capabilities` the CO will use in `CreateVolumeRequest`.
  // This field is OPTIONAL.
  repeated VolumeCapability volume_capabilities = 1;

  // If specified, the Plugin SHALL report the capacity of the storage
  // that can be used to provision volumes with the given Plugin
  // specific `parameters`. These are the same `parameters` the CO will
  // use in `CreateVolumeRequest`. This field is OPTIONAL.
  map<string, string> parameters = 2;

  // If specified, the Plugin SHALL report the capacity of the storage
  // that can be used to provision volumes that in the specified
  // `accessible_topology`. This is the same as the
  // `accessible_topology` the CO returns in a `CreateVolumeResponse`.
  // This field is OPTIONAL. This field SHALL NOT be set unless the
  // plugin advertises the VOLUME_ACCESSIBILITY_CONSTRAINTS capability.
  Topology accessible_topology = 3;
}

message GetCapacityResponse {
  // The available capacity, in bytes, of the storage that can be used
  // to provision volumes. If `volume_capabilities` or `parameters` is
  // specified in the request, the Plugin SHALL take those into
  // consideration when calculating the available capacity of the
  // storage. This field is REQUIRED.
  // The value of this field MUST NOT be negative.
  int64 available_capacity = 1;

  // The largest size that may be used in a
  // CreateVolumeRequest.capacity_range.required_bytes field
  // to create a volume with the same parameters as those in
  // GetCapacityRequest.
  //
  // If `volume_capabilities` or `parameters` is
  // specified in the request, the Plugin SHALL take those into
  // consideration when calculating the minimum volume size of the
  // storage.
  //
  // This field is OPTIONAL. MUST NOT be negative.
  // The Plugin SHOULD provide a value for this field if it has
  // a maximum size for individual volumes and leave it unset
  // otherwise. COs MAY use it to make decision about
  // where to create volumes.
  google.protobuf.Int64Value maximum_volume_size = 2;

  // The smallest size that may be used in a
  // CreateVolumeRequest.capacity_range.limit_bytes field
  // to create a volume with the same parameters as those in
  // GetCapacityRequest.
  //
  // If `volume_capabilities` or `parameters` is
  // specified in the request, the Plugin SHALL take those into
  // consideration when calculating the maximum volume size of the
  // storage.
  //
  // This field is OPTIONAL. MUST NOT be negative.
  // The Plugin SHOULD provide a value for this field if it has
  // a minimum size for individual volumes and leave it unset
  // otherwise. COs MAY use it to make decision about
  // where to create volumes.
  google.protobuf.Int64Value minimum_volume_size = 3
    [(alpha_field) = true];
}
```

##### GetCapacity Errors

If the plugin is unable to complete the GetCapacity call successfully, it MUST return a non-ok gRPC code in the gRPC status.

#### `ControllerGetCapabilities`

A Controller Plugin MUST implement this RPC call. This RPC allows the CO to check the supported capabilities of controller service provided by the Plugin.

```protobuf
message ControllerGetCapabilitiesRequest {
  // Intentionally empty.
}

message ControllerGetCapabilitiesResponse {
  // All the capabilities that the controller service supports. This
  // field is OPTIONAL.
  repeated ControllerServiceCapability capabilities = 1;
}

// Specifies a capability of the controller service.
message ControllerServiceCapability {
  message RPC {
    enum Type {
      UNKNOWN = 0;
      CREATE_DELETE_VOLUME = 1;
      PUBLISH_UNPUBLISH_VOLUME = 2;
      LIST_VOLUMES = 3;
      GET_CAPACITY = 4;
      // Currently the only way to consume a snapshot is to create
      // a volume from it. Therefore plugins supporting
      // CREATE_DELETE_SNAPSHOT MUST support creating volume from
      // snapshot.
      CREATE_DELETE_SNAPSHOT = 5;
      LIST_SNAPSHOTS = 6;

      // Plugins supporting volume cloning at the storage level MAY
      // report this capability. The source volume MUST be managed by
      // the same plugin. Not all volume sources and parameters
      // combinations MAY work.
      CLONE_VOLUME = 7;

      // Indicates the SP supports ControllerPublishVolume.readonly
      // field.
      PUBLISH_READONLY = 8;

      // See VolumeExpansion for details.
      EXPAND_VOLUME = 9;

      // Indicates the SP supports the
      // ListVolumesResponse.entry.published_node_ids field and the
      // ControllerGetVolumeResponse.published_node_ids field.
      // The SP MUST also support PUBLISH_UNPUBLISH_VOLUME.
      LIST_VOLUMES_PUBLISHED_NODES = 10;

      // Indicates that the Controller service can report volume
      // conditions.
      // An SP MAY implement `VolumeCondition` in only the Controller
      // Plugin, only the Node Plugin, or both.
      // If `VolumeCondition` is implemented in both the Controller and
      // Node Plugins, it SHALL report from different perspectives.
      // If for some reason Controller and Node Plugins report
      // misaligned volume conditions, CO SHALL assume the worst case
      // is the truth.
      // Note that, for alpha, `VolumeCondition` is intended be
      // informative for humans only, not for automation.
      VOLUME_CONDITION = 11 [(alpha_enum_value) = true];

      // Indicates the SP supports the ControllerGetVolume RPC.
      // This enables COs to, for example, fetch per volume
      // condition after a volume is provisioned.
      GET_VOLUME = 12 [(alpha_enum_value) = true];

      // Indicates the SP supports the SINGLE_NODE_SINGLE_WRITER and/or
      // SINGLE_NODE_MULTI_WRITER access modes.
      // These access modes are intended to replace the
      // SINGLE_NODE_WRITER access mode to clarify the number of writers
      // for a volume on a single node. Plugins MUST accept and allow
      // use of the SINGLE_NODE_WRITER access mode when either
      // SINGLE_NODE_SINGLE_WRITER and/or SINGLE_NODE_MULTI_WRITER are
      // supported, in order to permit older COs to continue working.
      SINGLE_NODE_MULTI_WRITER = 13 [(alpha_enum_value) = true];

      // Indicates the SP supports modifying volume with mutable
      // parameters. See ControllerModifyVolume for details.
      MODIFY_VOLUME = 14 [(alpha_enum_value) = true];

      // Indicates the SP supports the GetSnapshot RPC.
      // This enables COs to fetch an existing snapshot.
      GET_SNAPSHOT = 15 [(alpha_enum_value) = true];
    }

    Type type = 1;
  }

  oneof type {
    // RPC that the controller supports.
    RPC rpc = 1;
  }
}
```

##### ControllerGetCapabilities Errors

If the plugin is unable to complete the ControllerGetCapabilities call successfully, it MUST return a non-ok gRPC code in the gRPC status.

#### `CreateSnapshot`

A Controller Plugin MUST implement this RPC call if it has `CREATE_DELETE_SNAPSHOT` controller capability.
This RPC will be called by the CO to create a new snapshot from a source volume on behalf of a user.

This operation MUST be idempotent.
If a snapshot corresponding to the specified snapshot `name` is successfully cut and ready to use (meaning it MAY be specified as a `volume_content_source` in a `CreateVolumeRequest`), the Plugin MUST reply `0 OK` with the corresponding `CreateSnapshotResponse`.

If an error occurs before a snapshot is cut, `CreateSnapshot` SHOULD return a corresponding gRPC error code that reflects the error condition.

For plugins that supports snapshot post processing such as uploading, `CreateSnapshot` SHOULD return `0 OK` and `ready_to_use` SHOULD be set to `false` after the snapshot is cut but still being processed.
CO SHOULD then reissue the same `CreateSnapshotRequest` periodically until boolean `ready_to_use` flips to `true` indicating the snapshot has been "processed" and is ready to use to create new volumes.
If an error occurs during the process, `CreateSnapshot` SHOULD return a corresponding gRPC error code that reflects the error condition.

A snapshot MAY be used as the source to provision a new volume.
A CreateVolumeRequest message MAY specify an OPTIONAL source snapshot parameter.
Reverting a snapshot, where data in the original volume is erased and replaced with data in the snapshot, is an advanced functionality not every storage system can support and therefore is currently out of scope.

##### The ready_to_use Parameter

Some SPs MAY "process" the snapshot after the snapshot is cut, for example, maybe uploading the snapshot somewhere after the snapshot is cut.
The post-cut process MAY be a long process that could take hours.
The CO MAY freeze the application using the source volume before taking the snapshot.
The purpose of `freeze` is to ensure the application data is in consistent state.
When `freeze` is performed, the container is paused and the application is also paused.
When `thaw` is performed, the container and the application start running again.
During the snapshot processing phase, since the snapshot is already cut, a `thaw` operation can be performed so application can start running without waiting for the process to complete.
The `ready_to_use` parameter of the snapshot will become `true` after the process is complete.

For SPs that do not do additional processing after cut, the `ready_to_use` parameter SHOULD be `true` after the snapshot is cut.
`thaw` can be done when the `ready_to_use` parameter is `true` in this case.

The `ready_to_use` parameter provides guidance to the CO on when it can "thaw" the application in the process of snapshotting.
If the cloud provider or storage system needs to process the snapshot after the snapshot is cut, the `ready_to_use` parameter returned by CreateSnapshot SHALL be `false`.
CO MAY continue to call CreateSnapshot while waiting for the process to complete until `ready_to_use` becomes `true`.
Note that CreateSnapshot no longer blocks after the snapshot is cut.

A gRPC error code SHALL be returned if an error occurs during any stage of the snapshotting process.
A CO SHOULD explicitly delete snapshots when an error occurs.

Based on this information, CO can issue repeated (idempotent) calls to CreateSnapshot, monitor the response, and make decisions.
Note that CreateSnapshot is a synchronous call and it MUST block until the snapshot is cut.

```protobuf
message CreateSnapshotRequest {
  // The ID of the source volume to be snapshotted.
  // This field is REQUIRED.
  string source_volume_id = 1;

  // The suggested name for the snapshot. This field is REQUIRED for
  // idempotency.
  // Any Unicode string that conforms to the length limit is allowed
  // except those containing the following banned characters:
  // U+0000-U+0008, U+000B, U+000C, U+000E-U+001F, U+007F-U+009F.
  // (These are control characters other than commonly used whitespace.)
  string name = 2;

  // Secrets required by plugin to complete snapshot creation request.
  // This field is OPTIONAL. Refer to the `Secrets Requirements`
  // section on how to use this field.
  map<string, string> secrets = 3 [(csi_secret) = true];

  // Plugin specific parameters passed in as opaque key-value pairs.
  // This field is OPTIONAL. The Plugin is responsible for parsing and
  // validating these parameters. COs will treat these as opaque.
  // Use cases for opaque parameters:
  // - Specify a policy to automatically clean up the snapshot.
  // - Specify an expiration date for the snapshot.
  // - Specify whether the snapshot is readonly or read/write.
  // - Specify if the snapshot should be replicated to some place.
  // - Specify primary or secondary for replication systems that
  //   support snapshotting only on primary.
  map<string, string> parameters = 4;
}

message CreateSnapshotResponse {
  // Contains all attributes of the newly created snapshot that are
  // relevant to the CO along with information required by the Plugin
  // to uniquely identify the snapshot. This field is REQUIRED.
  Snapshot snapshot = 1;
}

// Information about a specific snapshot.
message Snapshot {
  // This is the complete size of the snapshot in bytes. The purpose of
  // this field is to give CO guidance on how much space is needed to
  // create a volume from this snapshot. The size of the volume MUST NOT
  // be less than the size of the source snapshot. This field is
  // OPTIONAL. If this field is not set, it indicates that this size is
  // unknown. The value of this field MUST NOT be negative and a size of
  // zero means it is unspecified.
  int64 size_bytes = 1;

  // The identifier for this snapshot, generated by the plugin.
  // This field is REQUIRED.
  // This field MUST contain enough information to uniquely identify
  // this specific snapshot vs all other snapshots supported by this
  // plugin.
  // This field SHALL be used by the CO in subsequent calls to refer to
  // this snapshot.
  // The SP is NOT responsible for global uniqueness of snapshot_id
  // across multiple SPs.
  string snapshot_id = 2;

  // Identity information for the source volume. Note that creating a
  // snapshot from a snapshot is not supported here so the source has to
  // be a volume. This field is REQUIRED.
  string source_volume_id = 3;

  // Timestamp when the point-in-time snapshot is taken on the storage
  // system. This field is REQUIRED.
  .google.protobuf.Timestamp creation_time = 4;

  // Indicates if a snapshot is ready to use as a
  // `volume_content_source` in a `CreateVolumeRequest`. The default
  // value is false. This field is REQUIRED.
  bool ready_to_use = 5;

  // The ID of the volume group snapshot that this snapshot is part of.
  // It uniquely identifies the group snapshot on the storage system.
  // This field is OPTIONAL.
  // If this snapshot is a member of a volume group snapshot, and it
  // MUST NOT be deleted as a stand alone snapshot, then the SP
  // MUST provide the ID of the volume group snapshot in this field.
  // If provided, CO MUST use this field in subsequent volume group
  // snapshot operations to indicate that this snapshot is part of the
  // specified group snapshot.
  // If not provided, CO SHALL treat the snapshot as independent,
  // and SP SHALL allow it to be deleted separately.
  // If this message is inside a VolumeGroupSnapshot message, the value
  // MUST be the same as the group_snapshot_id in that message.
  string group_snapshot_id = 6;
}
```

##### CreateSnapshot Errors

If the plugin is unable to complete the CreateSnapshot call successfully, it MUST return a non-ok gRPC code in the gRPC status.
If the conditions defined below are encountered, the plugin MUST return the specified gRPC error code.
The CO MUST implement the specified error recovery behavior when it encounters the gRPC error code.

| Condition | gRPC Code | Description | Recovery Behavior |
|-----------|-----------|-------------|-------------------|
| Snapshot already exists but is incompatible | 6 ALREADY_EXISTS | Indicates that a snapshot corresponding to the specified snapshot `name` already exists but is incompatible with the specified `volume_id`. | Caller MUST fix the arguments or use a different `name` before retrying. |
| Operation pending for snapshot | 10 ABORTED | Indicates that there is already an operation pending for the specified snapshot. In general the Cluster Orchestrator (CO) is responsible for ensuring that there is no more than one call "in-flight" per snapshot at a given time. However, in some circumstances, the CO MAY lose state (for example when the CO crashes and restarts), and MAY issue multiple calls simultaneously for the same snapshot. The Plugin, SHOULD handle this as gracefully as possible, and MAY return this error code to reject secondary calls. | Caller SHOULD ensure that there are no other calls pending for the specified snapshot, and then retry with exponential back off. |
| Not enough space to create snapshot | 13 RESOURCE_EXHAUSTED | There is not enough space on the storage system to handle the create snapshot request. | Caller SHOULD fail this request. Future calls to CreateSnapshot MAY succeed if space is freed up. |


#### `DeleteSnapshot`

A Controller Plugin MUST implement this RPC call if it has `CREATE_DELETE_SNAPSHOT` capability.
This RPC will be called by the CO to delete a snapshot.

This operation MUST be idempotent.
If a snapshot corresponding to the specified `snapshot_id` does not exist or the artifacts associated with the snapshot do not exist anymore, the Plugin MUST reply `0 OK`.

The CO SHALL NOT call this RPC with a snapshot for which SP provided a non-empty group_snapshot_id field at creation time.
A snapshot for which SP provided a non-empty group_snapshot_id indicates a snapshot that CAN NOT be deleted stand alone.
The SP MAY refuse to delete such snapshots with this RPC call and return an error instead.
For such snapshots SP MUST delete the entire snapshot group via a DeleteVolumeGroupSnapshotRequest call.

```protobuf
message DeleteSnapshotRequest {
  // The ID of the snapshot to be deleted.
  // This field is REQUIRED.
  string snapshot_id = 1;

  // Secrets required by plugin to complete snapshot deletion request.
  // This field is OPTIONAL. Refer to the `Secrets Requirements`
  // section on how to use this field.
  map<string, string> secrets = 2 [(csi_secret) = true];
}

message DeleteSnapshotResponse {}
```

##### DeleteSnapshot Errors

If the plugin is unable to complete the DeleteSnapshot call successfully, it MUST return a non-ok gRPC code in the gRPC status.
If the conditions defined below are encountered, the plugin MUST return the specified gRPC error code.
The CO MUST implement the specified error recovery behavior when it encounters the gRPC error code.

| Condition | gRPC Code | Description | Recovery Behavior |
|-----------|-----------|-------------|-------------------|
| Snapshot is part of a group | 3 INVALID_ARGUMENT | Indicates that the snapshot corresponding to the specified `snapshot_id` could not be deleted because it is part of a group snapshot and CAN NOT be deleted stand alone. | Caller SHOULD stop calling DeleteSnapshot and call DeleteVolumeGroupSnapshot instead. |
| Snapshot in use | 9 FAILED_PRECONDITION | Indicates that the snapshot corresponding to the specified `snapshot_id` could not be deleted because it is in use by another resource. | Caller SHOULD ensure that there are no other resources using the snapshot, and then retry with exponential back off. |
| Operation pending for snapshot | 10 ABORTED | Indicates that there is already an operation pending for the specified snapshot. In general the Cluster Orchestrator (CO) is responsible for ensuring that there is no more than one call "in-flight" per snapshot at a given time. However, in some circumstances, the CO MAY lose state (for example when the CO crashes and restarts), and MAY issue multiple calls simultaneously for the same snapshot. The Plugin, SHOULD handle this as gracefully as possible, and MAY return this error code to reject secondary calls. | Caller SHOULD ensure that there are no other calls pending for the specified snapshot, and then retry with exponential back off. |


#### `ListSnapshots`

A Controller Plugin MUST implement this RPC call if it has `LIST_SNAPSHOTS` capability.
The Plugin SHALL return the information about all snapshots on the storage system within the given parameters regardless of how they were created.
`ListSnapshots` SHALL NOT list a snapshot that is being created but has not been cut successfully yet.
If snapshots are created and/or deleted while the CO is concurrently paging through `ListSnapshots` results then it is possible that the CO MAY either witness duplicate snapshots in the list, not witness existing snapshots, or both.
The CO SHALL NOT expect a consistent "view" of all snapshots when paging through the snapshot list via multiple calls to `ListSnapshots`.

```protobuf
// List all snapshots on the storage system regardless of how they were
// created.
message ListSnapshotsRequest {
  // If specified (non-zero value), the Plugin MUST NOT return more
  // entries than this number in the response. If the actual number of
  // entries is more than this number, the Plugin MUST set `next_token`
  // in the response which can be used to get the next page of entries
  // in the subsequent `ListSnapshots` call. This field is OPTIONAL. If
  // not specified (zero value), it means there is no restriction on the
  // number of entries that can be returned.
  // The value of this field MUST NOT be negative.
  int32 max_entries = 1;

  // A token to specify where to start paginating. Set this field to
  // `next_token` returned by a previous `ListSnapshots` call to get the
  // next page of entries. This field is OPTIONAL.
  // An empty string is equal to an unspecified field value.
  string starting_token = 2;

  // Identity information for the source volume. This field is OPTIONAL.
  // It can be used to list snapshots by volume.
  string source_volume_id = 3;

  // Identity information for a specific snapshot. This field is
  // OPTIONAL. It can be used to list only a specific snapshot.
  // ListSnapshots will return with current snapshot information
  // and will not block if the snapshot is being processed after
  // it is cut.
  string snapshot_id = 4;

  // Secrets required by plugin to complete ListSnapshot request.
  // This field is OPTIONAL. Refer to the `Secrets Requirements`
  // section on how to use this field.
  map<string, string> secrets = 5 [(csi_secret) = true];
}

message ListSnapshotsResponse {
  message Entry {
    Snapshot snapshot = 1;
  }

  repeated Entry entries = 1;

  // This token allows you to get the next page of entries for
  // `ListSnapshots` request. If the number of entries is larger than
  // `max_entries`, use the `next_token` as a value for the
  // `starting_token` field in the next `ListSnapshots` request. This
  // field is OPTIONAL.
  // An empty string is equal to an unspecified field value.
  string next_token = 2;
}
```

##### ListSnapshots Errors

If the plugin is unable to complete the ListSnapshots call successfully, it MUST return a non-ok gRPC code in the gRPC status.
If the conditions defined below are encountered, the plugin MUST return the specified gRPC error code.
The CO MUST implement the specified error recovery behavior when it encounters the gRPC error code.

| Condition | gRPC Code | Description | Recovery Behavior |
|-----------|-----------|-------------|-------------------|
| Invalid `starting_token` | 10 ABORTED | Indicates that `starting_token` is not valid. | Caller SHOULD start the `ListSnapshots` operation again with an empty `starting_token`. |

#### `GetSnapshot`

**ALPHA FEATURE**

This optional RPC MAY be called by the CO to fetch current information about a snapshot.

A Controller Plugin MUST implement this `GetSnapshot` RPC call if it has `GET_SNAPSHOT` capability.

An SP that supports Group Controller Service and the CREATE_DELETE_GET_VOLUME_GROUP_SNAPSHOT capability SHOULD implement the `GetSnapshot` RPC in order to retrieve information for individual snapshots that are part of a group snapshot.

`GetSnapshotResponse` should contain current information of a snapshot if it exists.
If the snapshot does not exist any more, `GetSnapshot` should return gRPC error code `NOT_FOUND`. Snapshots which were recently created and not yet ready to use must be retrievable with this RPC.

```protobuf
message GetSnapshotRequest {
  option (alpha_message) = true;

  // The ID of the snapshot to fetch current snapshot information for.
  // This field is REQUIRED.
  string snapshot_id = 1;

  // Secrets required by plugin to complete GetSnapshot request.
  // This field is OPTIONAL. Refer to the `Secrets Requirements`
  // section on how to use this field.
  map<string, string> secrets = 2 [(csi_secret) = true];
}

message GetSnapshotResponse {
  option (alpha_message) = true;

  // This field is REQUIRED
  Snapshot snapshot = 1;
}
```

##### GetSnapshot Errors

If the plugin is unable to complete the GetSnapshot call successfully, it MUST return a non-ok gRPC code in the gRPC status.
If the conditions defined below are encountered, the plugin MUST return the specified gRPC error code.
The CO MUST implement the specified error recovery behavior when it encounters the gRPC error code.

| Condition | gRPC Code | Description | Recovery Behavior |
|-----------|-----------|-------------|-------------------|
| Snapshot does not exist | 5 NOT_FOUND | Indicates that a snapshot corresponding to the specified `snapshot_id` does not exist. | Caller MUST verify that the `snapshot_id` is correct and that the snapshot is accessible and has not been deleted before retrying with exponential back off. |

#### `ControllerExpandVolume`

A Controller plugin MUST implement this RPC call if plugin has `EXPAND_VOLUME` controller capability.
This RPC allows the CO to expand the size of a volume.

This operation MUST be idempotent.
If a volume corresponding to the specified volume ID is already larger than or equal to the target capacity of the expansion request, the plugin SHOULD reply 0 OK.

This call MAY be made by the CO during any time in the lifecycle of the volume after creation if plugin has `VolumeExpansion.ONLINE` capability.
If plugin has `EXPAND_VOLUME` node capability, then `NodeExpandVolume` MUST be called after successful `ControllerExpandVolume` and `node_expansion_required` in `ControllerExpandVolumeResponse` is `true`.

If specified, the `volume_capability` in `ControllerExpandVolumeRequest` should be same as what CO would pass in `ControllerPublishVolumeRequest`.

If the plugin has only `VolumeExpansion.OFFLINE` expansion capability and volume is currently published or available on a node then `ControllerExpandVolume` MUST be called ONLY after either:
- The plugin has controller `PUBLISH_UNPUBLISH_VOLUME` capability and `ControllerUnpublishVolume` has been invoked successfully.

OR ELSE

- The plugin does NOT have controller `PUBLISH_UNPUBLISH_VOLUME` capability, the plugin has node `STAGE_UNSTAGE_VOLUME` capability, and `NodeUnstageVolume` has been completed successfully.

OR ELSE

- The plugin does NOT have controller `PUBLISH_UNPUBLISH_VOLUME` capability, nor node `STAGE_UNSTAGE_VOLUME` capability, and `NodeUnpublishVolume` has completed successfully.

Examples:
* Offline Volume Expansion:
  Given an ElasticSearch process that runs on Azure Disk and needs more space.
  - The administrator takes the Elasticsearch server offline by stopping the workload and CO calls `ControllerUnpublishVolume`.
  - The administrator requests more space for the volume from CO.
  - The CO in turn first makes `ControllerExpandVolume` RPC call which results in requesting more space from Azure cloud provider for volume ID that was being used by ElasticSearch.
  - Once `ControllerExpandVolume` is completed and successful, the CO will inform administrator about it and administrator will resume the ElasticSearch workload.
  - On the node where the ElasticSearch workload is scheduled, the CO calls `NodeExpandVolume` after calling `NodeStageVolume`.
  - Calling `NodeExpandVolume` on volume results in expanding the underlying file system and added space becomes available to workload when it starts up.
* Online Volume Expansion:
  Given a Mysql server running on Openstack Cinder and needs more space.
  - The administrator requests more space for volume from the CO.
  - The CO in turn first makes `ControllerExpandVolume` RPC call which results in requesting more space from Openstack Cinder for given volume.
  - On the node where the mysql workload is running, the CO calls `NodeExpandVolume` while volume is in-use using the path where the volume is staged.
  - Calling `NodeExpandVolume` on volume results in expanding the underlying file system and added space automatically becomes available to mysql workload without any downtime.


```protobuf
message ControllerExpandVolumeRequest {
  // The ID of the volume to expand. This field is REQUIRED.
  string volume_id = 1;

  // This allows CO to specify the capacity requirements of the volume
  // after expansion. This field is REQUIRED.
  CapacityRange capacity_range = 2;

  // Secrets required by the plugin for expanding the volume.
  // This field is OPTIONAL.
  map<string, string> secrets = 3 [(csi_secret) = true];

  // Volume capability describing how the CO intends to use this volume.
  // This allows SP to determine if volume is being used as a block
  // device or mounted file system. For example - if volume is
  // being used as a block device - the SP MAY set
  // node_expansion_required to false in ControllerExpandVolumeResponse
  // to skip invocation of NodeExpandVolume on the node by the CO.
  // This is an OPTIONAL field.
  VolumeCapability volume_capability = 4;
}

message ControllerExpandVolumeResponse {
  // Capacity of volume after expansion. This field is REQUIRED.
  int64 capacity_bytes = 1;

  // Whether node expansion is required for the volume. When true
  // the CO MUST make NodeExpandVolume RPC call on the node. This field
  // is REQUIRED.
  bool node_expansion_required = 2;
}
```

##### ControllerExpandVolume Errors

| Condition | gRPC Code | Description | Recovery Behavior |
|-----------|-----------|-------------|-------------------|
| Exceeds capabilities | 3 INVALID_ARGUMENT | Indicates that the CO has specified capabilities not supported by the volume. | Caller MAY verify volume capabilities by calling ValidateVolumeCapabilities and retry with matching capabilities. |
| Volume does not exist | 5 NOT_FOUND | Indicates that a volume corresponding to the specified volume_id does not exist. | Caller MUST verify that the volume_id is correct and that the volume is accessible and has not been deleted before retrying with exponential back off. |
| Volume in use | 9 FAILED_PRECONDITION | Indicates that the volume corresponding to the specified `volume_id` could not be expanded because it is currently published on a node but the plugin does not have ONLINE expansion capability. | Caller SHOULD ensure that volume is not published and retry with exponential back off. |
| Unsupported `capacity_range` | 11 OUT_OF_RANGE | Indicates that the capacity range is not allowed by the Plugin. More human-readable information MAY be provided in the gRPC `status.message` field. | Caller MUST fix the capacity range before retrying. |

#### RPC Interactions

##### `CreateVolume`, `DeleteVolume`, `ListVolumes`

It is worth noting that the plugin-generated `volume_id` is a REQUIRED field for the `DeleteVolume` RPC, as opposed to the CO-generated volume `name` that is REQUIRED for the `CreateVolume` RPC: these fields MAY NOT contain the same value.
If a `CreateVolume` operation times out, leaving the CO without an ID with which to reference a volume, and the CO *also* decides that it no longer needs/wants the volume in question then the CO MAY choose one of the following paths:

1. Replay the `CreateVolume` RPC that timed out; upon success execute `DeleteVolume` using the known volume ID (from the response to `CreateVolume`).
2. Execute the `ListVolumes` RPC to possibly obtain a volume ID that may be used to execute a `DeleteVolume` RPC; upon success execute `DeleteVolume`.
3. The CO takes no further action regarding the timed out RPC, a volume is possibly leaked and the operator/user is expected to clean up.

It is NOT REQUIRED for a controller plugin to implement the `LIST_VOLUMES` capability if it supports the `CREATE_DELETE_VOLUME` capability: the onus is upon the CO to take into consideration the full range of plugin capabilities before deciding how to proceed in the above scenario.

##### `CreateSnapshot`, `DeleteSnapshot`, `ListSnapshots`

The plugin-generated `snapshot_id` is a REQUIRED field for the `DeleteSnapshot` RPC, as opposed to the CO-generated snapshot `name` that is REQUIRED for the `CreateSnapshot` RPC.
A `CreateSnapshot` operation SHOULD return with a `snapshot_id` when the snapshot is cut successfully.
If a `CreateSnapshot` operation times out before the snapshot is cut, leaving the CO without an ID with which to reference a snapshot, and the CO also decides that it no longer needs/wants the snapshot in question then the CO MAY choose one of the following paths:

1. Execute the `ListSnapshots` RPC to possibly obtain a snapshot ID that may be used to execute a `DeleteSnapshot` RPC; upon success execute `DeleteSnapshot`.
2. The CO takes no further action regarding the timed out RPC, a snapshot is possibly leaked and the operator/user is expected to clean up.

It is NOT REQUIRED for a controller plugin to implement the `LIST_SNAPSHOTS` capability if it supports the `CREATE_DELETE_SNAPSHOT` capability: the onus is upon the CO to take into consideration the full range of plugin capabilities before deciding how to proceed in the above scenario.

ListSnapshots SHALL return with current information regarding the snapshots on the storage system.
When processing is complete, the `ready_to_use` parameter of the snapshot from ListSnapshots SHALL become `true`.
The downside of calling ListSnapshots is that ListSnapshots will not return a gRPC error code if an error occurs during the processing. So calling CreateSnapshot repeatedly is the preferred way to check if the processing is complete.

### Node Service RPC

#### `NodeStageVolume`

A Node Plugin MUST implement this RPC call if it has `STAGE_UNSTAGE_VOLUME` node capability.

This RPC is called by the CO prior to the volume being consumed by any workloads on the node by `NodePublishVolume`.
The Plugin SHALL assume that this RPC will be executed on the node where the volume will be used.
This RPC SHOULD be called by the CO when a workload that wants to use the specified volume is placed (scheduled) on the specified node for the first time or for the first time since a `NodeUnstageVolume` call for the specified volume was called and returned success on that node.

If the corresponding Controller Plugin has `PUBLISH_UNPUBLISH_VOLUME` controller capability and the Node Plugin has `STAGE_UNSTAGE_VOLUME` capability, then the CO MUST guarantee that this RPC is called after `ControllerPublishVolume` is called for the given volume on the given node and returns a success.
The CO MUST guarantee that this RPC is called and returns a success before any `NodePublishVolume` is called for the given volume on the given node.

This operation MUST be idempotent.
If the volume corresponding to the `volume_id` is already staged to the `staging_target_path`, and is identical to the specified `volume_capability` the Plugin MUST reply `0 OK`.

If this RPC failed, or the CO does not know if it failed or not, it MAY choose to call `NodeStageVolume` again, or choose to call `NodeUnstageVolume`.

```protobuf
message NodeStageVolumeRequest {
  // The ID of the volume to publish. This field is REQUIRED.
  string volume_id = 1;

  // The CO SHALL set this field to the value returned by
  // `ControllerPublishVolume` if the corresponding Controller Plugin
  // has `PUBLISH_UNPUBLISH_VOLUME` controller capability, and SHALL be
  // left unset if the corresponding Controller Plugin does not have
  // this capability. This is an OPTIONAL field.
  map<string, string> publish_context = 2;

  // The path to which the volume MAY be staged. It MUST be an
  // absolute path in the root filesystem of the process serving this
  // request, and MUST be a directory. The CO SHALL ensure that there
  // is only one `staging_target_path` per volume. The CO SHALL ensure
  // that the path is directory and that the process serving the
  // request has `read` and `write` permission to that directory. The
  // CO SHALL be responsible for creating the directory if it does not
  // exist.
  // This is a REQUIRED field.
  // This field overrides the general CSI size limit.
  // SP SHOULD support the maximum path length allowed by the operating
  // system/filesystem, but, at a minimum, SP MUST accept a max path
  // length of at least 128 bytes.
  string staging_target_path = 3;

  // Volume capability describing how the CO intends to use this volume.
  // SP MUST ensure the CO can use the staged volume as described.
  // Otherwise SP MUST return the appropriate gRPC error code.
  // This is a REQUIRED field.
  VolumeCapability volume_capability = 4;

  // Secrets required by plugin to complete node stage volume request.
  // This field is OPTIONAL. Refer to the `Secrets Requirements`
  // section on how to use this field.
  map<string, string> secrets = 5 [(csi_secret) = true];

  // Volume context as returned by SP in
  // CreateVolumeResponse.Volume.volume_context.
  // This field is OPTIONAL and MUST match the volume_context of the
  // volume identified by `volume_id`.
  map<string, string> volume_context = 6;
}

message NodeStageVolumeResponse {
  // Intentionally empty.
}
```

#### NodeStageVolume Errors

If the plugin is unable to complete the NodeStageVolume call successfully, it MUST return a non-ok gRPC code in the gRPC status.
If the conditions defined below are encountered, the plugin MUST return the specified gRPC error code.
The CO MUST implement the specified error recovery behavior when it encounters the gRPC error code.

| Condition | gRPC Code | Description | Recovery Behavior |
|-----------|-----------|-------------|-------------------|
| Volume does not exist | 5 NOT_FOUND | Indicates that a volume corresponding to the specified `volume_id` does not exist. | Caller MUST verify that the `volume_id` is correct and that the volume is accessible and has not been deleted before retrying with exponential back off. |
| Volume published but is incompatible | 6 ALREADY_EXISTS | Indicates that a volume corresponding to the specified `volume_id` has already been published at the specified `staging_target_path` but is incompatible with the specified `volume_capability` flag. | Caller MUST fix the arguments before retrying. |
| Exceeds capabilities | 9 FAILED_PRECONDITION | Indicates that the CO has specified capabilities not supported by the volume. | Caller MAY choose to call `ValidateVolumeCapabilities` to validate the volume capabilities, or wait for the volume to be unpublished on the node. |

#### `NodeUnstageVolume`

A Node Plugin MUST implement this RPC call if it has `STAGE_UNSTAGE_VOLUME` node capability.

This RPC is a reverse operation of `NodeStageVolume`.
This RPC MUST undo the work by the corresponding `NodeStageVolume`.
This RPC SHALL be called by the CO once for each `staging_target_path` that was successfully setup via `NodeStageVolume`.

If the corresponding Controller Plugin has `PUBLISH_UNPUBLISH_VOLUME` controller capability and the Node Plugin has `STAGE_UNSTAGE_VOLUME` capability, the CO MUST guarantee that this RPC is called and returns success before calling `ControllerUnpublishVolume` for the given node and the given volume.
The CO MUST guarantee that this RPC is called after all `NodeUnpublishVolume` have been called and returned success for the given volume on the given node.

The Plugin SHALL assume that this RPC will be executed on the node where the volume is being used.

This RPC MAY be called by the CO when the workload using the volume is being moved to a different node, or all the workloads using the volume on a node have finished.

This operation MUST be idempotent.
If the volume corresponding to the `volume_id` is not staged to the `staging_target_path`,  the Plugin MUST reply `0 OK`.

If this RPC failed, or the CO does not know if it failed or not, it MAY choose to call `NodeUnstageVolume` again.

```protobuf
message NodeUnstageVolumeRequest {
  // The ID of the volume. This field is REQUIRED.
  string volume_id = 1;

  // The path at which the volume was staged. It MUST be an absolute
  // path in the root filesystem of the process serving this request.
  // This is a REQUIRED field.
  // This field overrides the general CSI size limit.
  // SP SHOULD support the maximum path length allowed by the operating
  // system/filesystem, but, at a minimum, SP MUST accept a max path
  // length of at least 128 bytes.
  string staging_target_path = 2;
}

message NodeUnstageVolumeResponse {
  // Intentionally empty.
}
```

#### NodeUnstageVolume Errors

If the plugin is unable to complete the NodeUnstageVolume call successfully, it MUST return a non-ok gRPC code in the gRPC status.
If the conditions defined below are encountered, the plugin MUST return the specified gRPC error code.
The CO MUST implement the specified error recovery behavior when it encounters the gRPC error code.

| Condition | gRPC Code | Description | Recovery Behavior |
|-----------|-----------|-------------|-------------------|
| Volume does not exist | 5 NOT_FOUND | Indicates that a volume corresponding to the specified `volume_id` does not exist. | Caller MUST verify that the `volume_id` is correct and that the volume is accessible and has not been deleted before retrying with exponential back off. |

#### RPC Interactions and Reference Counting
`NodeStageVolume`, `NodeUnstageVolume`, `NodePublishVolume`, `NodeUnpublishVolume`

The following interaction semantics ARE REQUIRED if the plugin advertises the `STAGE_UNSTAGE_VOLUME` capability.
`NodeStageVolume` MUST be called and return success once per volume per node before any `NodePublishVolume` MAY be called for the volume.
All `NodeUnpublishVolume` MUST be called and return success for a volume before `NodeUnstageVolume` MAY be called for the volume.

Note that this requires that all COs MUST support reference counting of volumes so that if `STAGE_UNSTAGE_VOLUME` is advertised by the SP, the CO MUST fulfill the above interaction semantics.

#### `NodePublishVolume`

This RPC is called by the CO when a workload that wants to use the specified volume is placed (scheduled) on a node.
The Plugin SHALL assume that this RPC will be executed on the node where the volume will be used.

If the corresponding Controller Plugin has `PUBLISH_UNPUBLISH_VOLUME` controller capability, the CO MUST guarantee that this RPC is called after `ControllerPublishVolume` is called for the given volume on the given node and returns a success.

This operation MUST be idempotent.
If the volume corresponding to the `volume_id` has already been published at the specified `target_path`, and is compatible with the specified `volume_capability` and `readonly` flag, the Plugin MUST reply `0 OK`.

If this RPC failed, or the CO does not know if it failed or not, it MAY choose to call `NodePublishVolume` again, or choose to call `NodeUnpublishVolume`.

This RPC MAY be called by the CO multiple times on the same node for the same volume with a possibly different `target_path` and/or other arguments if the volume supports either `MULTI_NODE_...` or `SINGLE_NODE_MULTI_WRITER` access modes (see second table).
The possible `MULTI_NODE_...` access modes are `MULTI_NODE_READER_ONLY`, `MULTI_NODE_SINGLE_WRITER` or `MULTI_NODE_MULTI_WRITER`.
COs SHOULD NOT call `NodePublishVolume` a second time with a different `volume_capability`.
If this happens, the Plugin SHOULD return `FAILED_PRECONDITION`.

The following table shows what the Plugin SHOULD return when receiving a second `NodePublishVolume` on the same volume on the same node:

(**T<sub>n</sub>**: target path of the n<sup>th</sup> `NodePublishVolume`; **P<sub>n</sub>**: other arguments of the n<sup>th</sup> `NodePublishVolume` except `secrets`)

|                    | T1=T2, P1=P2    | T1=T2, P1!=P2  | T1!=T2, P1=P2       | T1!=T2, P1!=P2     |
|--------------------|-----------------|----------------|---------------------|--------------------|
| MULTI_NODE_...     | OK (idempotent) | ALREADY_EXISTS | OK                  | OK                 |
| Non MULTI_NODE_... | OK (idempotent) | ALREADY_EXISTS | FAILED_PRECONDITION | FAILED_PRECONDITION|

NOTE: If the Plugin supports the `SINGLE_NODE_MULTI_WRITER` capability, use the following table instead for what the Plugin SHOULD return when receiving a second `NodePublishVolume` on the same volume on the same node:

|                                       | T1=T2, P1=P2    | T1=T2, P1!=P2  | T1!=T2, P1=P2       | T1!=T2, P1!=P2      |
|---------------------------------------|-----------------|----------------|---------------------|---------------------|
| SINGLE_NODE_SINGLE_WRITER             | OK (idempotent) | ALREADY_EXISTS | FAILED_PRECONDITION | FAILED_PRECONDITION |
| SINGLE_NODE_MULTI_WRITER              | OK (idempotent) | ALREADY_EXISTS | OK                  | OK                  |
| MULTI_NODE_...                        | OK (idempotent) | ALREADY_EXISTS | OK                  | OK                  |
| Non MULTI_NODE_...                    | OK (idempotent) | ALREADY_EXISTS | FAILED_PRECONDITION | FAILED_PRECONDITION |

The `SINGLE_NODE_SINGLE_WRITER` and `SINGLE_NODE_MULTI_WRITER` access modes are intended to replace the `SINGLE_NODE_WRITER` access mode to clarify the number of writers for a volume on a single node.
Plugins MUST accept and allow use of the `SINGLE_NODE_WRITER` access mode (subject to the processing rules above), when either `SINGLE_NODE_SINGLE_WRITER` and/or `SINGLE_NODE_MULTI_WRITER` are supported, in order to permit older COs to continue working.


```protobuf
message NodePublishVolumeRequest {
  // The ID of the volume to publish. This field is REQUIRED.
  string volume_id = 1;

  // The CO SHALL set this field to the value returned by
  // `ControllerPublishVolume` if the corresponding Controller Plugin
  // has `PUBLISH_UNPUBLISH_VOLUME` controller capability, and SHALL be
  // left unset if the corresponding Controller Plugin does not have
  // this capability. This is an OPTIONAL field.
  map<string, string> publish_context = 2;

  // The path to which the volume was staged by `NodeStageVolume`.
  // It MUST be an absolute path in the root filesystem of the process
  // serving this request.
  // It MUST be set if the Node Plugin implements the
  // `STAGE_UNSTAGE_VOLUME` node capability.
  // This is an OPTIONAL field.
  // This field overrides the general CSI size limit.
  // SP SHOULD support the maximum path length allowed by the operating
  // system/filesystem, but, at a minimum, SP MUST accept a max path
  // length of at least 128 bytes.
  string staging_target_path = 3;

  // The path to which the volume will be published. It MUST be an
  // absolute path in the root filesystem of the process serving this
  // request. The CO SHALL ensure uniqueness of target_path per volume.
  // The CO SHALL ensure that the parent directory of this path exists
  // and that the process serving the request has `read` and `write`
  // permissions to that parent directory.
  // For volumes with an access type of block, the SP SHALL place the
  // block device at target_path.
  // For volumes with an access type of mount, the SP SHALL place the
  // mounted directory at target_path.
  // Creation of target_path is the responsibility of the SP.
  // This is a REQUIRED field.
  // This field overrides the general CSI size limit.
  // SP SHOULD support the maximum path length allowed by the operating
  // system/filesystem, but, at a minimum, SP MUST accept a max path
  // length of at least 128 bytes.
  string target_path = 4;

  // Volume capability describing how the CO intends to use this volume.
  // SP MUST ensure the CO can use the published volume as described.
  // Otherwise SP MUST return the appropriate gRPC error code.
  // This is a REQUIRED field.
  VolumeCapability volume_capability = 5;

  // Indicates SP MUST publish the volume in readonly mode.
  // This field is REQUIRED.
  bool readonly = 6;

  // Secrets required by plugin to complete node publish volume request.
  // This field is OPTIONAL. Refer to the `Secrets Requirements`
  // section on how to use this field.
  map<string, string> secrets = 7 [(csi_secret) = true];

  // Volume context as returned by SP in
  // CreateVolumeResponse.Volume.volume_context.
  // This field is OPTIONAL and MUST match the volume_context of the
  // volume identified by `volume_id`.
  map<string, string> volume_context = 8;
}

message NodePublishVolumeResponse {
  // Intentionally empty.
}
```

##### NodePublishVolume Errors

If the plugin is unable to complete the NodePublishVolume call successfully, it MUST return a non-ok gRPC code in the gRPC status.
If the conditions defined below are encountered, the plugin MUST return the specified gRPC error code.
The CO MUST implement the specified error recovery behavior when it encounters the gRPC error code.

| Condition | gRPC Code | Description | Recovery Behavior |
|-----------|-----------|-------------|-------------------|
| Volume does not exist | 5 NOT_FOUND | Indicates that a volume corresponding to the specified `volume_id` does not exist. | Caller MUST verify that the `volume_id` is correct and that the volume is accessible and has not been deleted before retrying with exponential back off. |
| Volume published but is incompatible | 6 ALREADY_EXISTS | Indicates that a volume corresponding to the specified `volume_id` has already been published at the specified `target_path` but is incompatible with the specified `volume_capability` or `readonly` flag. | Caller MUST fix the arguments before retrying. |
| Exceeds capabilities | 9 FAILED_PRECONDITION | Indicates that the CO has specified capabilities not supported by the volume. | Caller MAY choose to call `ValidateVolumeCapabilities` to validate the volume capabilities, or wait for the volume to be unpublished on the node. |
| Staging target path not set | 9 FAILED_PRECONDITION | Indicates that `STAGE_UNSTAGE_VOLUME` capability is set but no `staging_target_path` was set. | Caller MUST make sure call to `NodeStageVolume` is made and returns success before retrying with valid `staging_target_path`. |



#### `NodeUnpublishVolume`

A Node Plugin MUST implement this RPC call.
This RPC is a reverse operation of `NodePublishVolume`.
This RPC MUST undo the work by the corresponding `NodePublishVolume`.
This RPC SHALL be called by the CO at least once for each `target_path` that was successfully setup via `NodePublishVolume`.
If the corresponding Controller Plugin has `PUBLISH_UNPUBLISH_VOLUME` controller capability, the CO SHOULD issue all `NodeUnpublishVolume` (as specified above) before calling `ControllerUnpublishVolume` for the given node and the given volume.
The Plugin SHALL assume that this RPC will be executed on the node where the volume is being used.

This RPC is typically called by the CO when the workload using the volume is being moved to a different node, or all the workload using the volume on a node has finished.

This operation MUST be idempotent.
If this RPC failed, or the CO does not know if it failed or not, it can choose to call `NodeUnpublishVolume` again.

```protobuf
message NodeUnpublishVolumeRequest {
  // The ID of the volume. This field is REQUIRED.
  string volume_id = 1;

  // The path at which the volume was published. It MUST be an absolute
  // path in the root filesystem of the process serving this request.
  // The SP MUST delete the file or directory it created at this path.
  // This is a REQUIRED field.
  // This field overrides the general CSI size limit.
  // SP SHOULD support the maximum path length allowed by the operating
  // system/filesystem, but, at a minimum, SP MUST accept a max path
  // length of at least 128 bytes.
  string target_path = 2;
}

message NodeUnpublishVolumeResponse {
  // Intentionally empty.
}
```

##### NodeUnpublishVolume Errors

If the plugin is unable to complete the NodeUnpublishVolume call successfully, it MUST return a non-ok gRPC code in the gRPC status.
If the conditions defined below are encountered, the plugin MUST return the specified gRPC error code.
The CO MUST implement the specified error recovery behavior when it encounters the gRPC error code.

| Condition | gRPC Code | Description | Recovery Behavior |
|-----------|-----------|-------------|-------------------|
| Volume does not exist | 5 NOT_FOUND | Indicates that a volume corresponding to the specified `volume_id` does not exist. | Caller MUST verify that the `volume_id` is correct and that the volume is accessible and has not been deleted before retrying with exponential back off. |


#### `NodeGetVolumeStats`

A Node plugin MUST implement this RPC call if it has GET_VOLUME_STATS node capability or VOLUME_CONDITION node capability.
`NodeGetVolumeStats` RPC call returns the volume capacity statistics available for the volume.

If the volume is being used in `BlockVolume` mode then `used` and `available` MAY be omitted from `usage` field of `NodeGetVolumeStatsResponse`.
Similarly, inode information MAY be omitted from `NodeGetVolumeStatsResponse` when unavailable.

The `staging_target_path` field is not required, for backwards compatibility, but the CO SHOULD supply it.
Plugins can use this field to determine if `volume_path` is where the volume is published or staged,
and setting this field to non-empty allows plugins to function with less stored state on the node.

```protobuf
message NodeGetVolumeStatsRequest {
  // The ID of the volume. This field is REQUIRED.
  string volume_id = 1;

  // It can be any valid path where volume was previously
  // staged or published.
  // It MUST be an absolute path in the root filesystem of
  // the process serving this request.
  // This is a REQUIRED field.
  // This field overrides the general CSI size limit.
  // SP SHOULD support the maximum path length allowed by the operating
  // system/filesystem, but, at a minimum, SP MUST accept a max path
  // length of at least 128 bytes.
  string volume_path = 2;

  // The path where the volume is staged, if the plugin has the
  // STAGE_UNSTAGE_VOLUME capability, otherwise empty.
  // If not empty, it MUST be an absolute path in the root
  // filesystem of the process serving this request.
  // This field is OPTIONAL.
  // This field overrides the general CSI size limit.
  // SP SHOULD support the maximum path length allowed by the operating
  // system/filesystem, but, at a minimum, SP MUST accept a max path
  // length of at least 128 bytes.
  string staging_target_path = 3;
}

message NodeGetVolumeStatsResponse {
  // This field is OPTIONAL.
  repeated VolumeUsage usage = 1;
  // Information about the current condition of the volume.
  // This field is OPTIONAL.
  // This field MUST be specified if the VOLUME_CONDITION node
  // capability is supported.
  VolumeCondition volume_condition = 2 [(alpha_field) = true];
}

message VolumeUsage {
  enum Unit {
    UNKNOWN = 0;
    BYTES = 1;
    INODES = 2;
  }
  // The available capacity in specified Unit. This field is OPTIONAL.
  // The value of this field MUST NOT be negative.
  int64 available = 1;

  // The total capacity in specified Unit. This field is REQUIRED.
  // The value of this field MUST NOT be negative.
  int64 total = 2;

  // The used capacity in specified Unit. This field is OPTIONAL.
  // The value of this field MUST NOT be negative.
  int64 used = 3;

  // Units by which values are measured. This field is REQUIRED.
  Unit unit = 4;
}

// VolumeCondition represents the current condition of a volume.
message VolumeCondition {
  option (alpha_message) = true;

  // Normal volumes are available for use and operating optimally.
  // An abnormal volume does not meet these criteria.
  // This field is REQUIRED.
  bool abnormal = 1;

  // The message describing the condition of the volume.
  // This field is REQUIRED.
  string message = 2;
}
```

##### NodeGetVolumeStats Errors

If the plugin is unable to complete the `NodeGetVolumeStats` call successfully, it MUST return a non-ok gRPC code in the gRPC status.
If the conditions defined below are encountered, the plugin MUST return the specified gRPC error code.
The CO MUST implement the specified error recovery behavior when it encounters the gRPC error code.


| Condition | gRPC Code | Description | Recovery Behavior |
|-----------|-----------|-------------|-------------------|
| Volume does not exist | 5 NOT_FOUND | Indicates that a volume corresponding to the specified `volume_id` does not exist on specified `volume_path`. | Caller MUST verify that the `volume_id` is correct and that the volume is accessible on specified `volume_path` and has not been deleted before retrying with exponential back off. |

#### `NodeGetCapabilities`

A Node Plugin MUST implement this RPC call.
This RPC allows the CO to check the supported capabilities of node service provided by the Plugin.

```protobuf
message NodeGetCapabilitiesRequest {
  // Intentionally empty.
}

message NodeGetCapabilitiesResponse {
  // All the capabilities that the node service supports. This field
  // is OPTIONAL.
  repeated NodeServiceCapability capabilities = 1;
}

// Specifies a capability of the node service.
message NodeServiceCapability {
  message RPC {
    enum Type {
      UNKNOWN = 0;
      STAGE_UNSTAGE_VOLUME = 1;
      // If Plugin implements GET_VOLUME_STATS capability
      // then it MUST implement NodeGetVolumeStats RPC
      // call for fetching volume statistics.
      GET_VOLUME_STATS = 2;
      // See VolumeExpansion for details.
      EXPAND_VOLUME = 3;
      // Indicates that the Node service can report volume conditions.
      // An SP MAY implement `VolumeCondition` in only the Node
      // Plugin, only the Controller Plugin, or both.
      // If `VolumeCondition` is implemented in both the Node and
      // Controller Plugins, it SHALL report from different
      // perspectives.
      // If for some reason Node and Controller Plugins report
      // misaligned volume conditions, CO SHALL assume the worst case
      // is the truth.
      // Note that, for alpha, `VolumeCondition` is intended to be
      // informative for humans only, not for automation.
      VOLUME_CONDITION = 4 [(alpha_enum_value) = true];

      // Indicates the SP supports the SINGLE_NODE_SINGLE_WRITER and/or
      // SINGLE_NODE_MULTI_WRITER access modes.
      // These access modes are intended to replace the
      // SINGLE_NODE_WRITER access mode to clarify the number of writers
      // for a volume on a single node. Plugins MUST accept and allow
      // use of the SINGLE_NODE_WRITER access mode (subject to the
      // processing rules for NodePublishVolume), when either
      // SINGLE_NODE_SINGLE_WRITER and/or SINGLE_NODE_MULTI_WRITER are
      // supported, in order to permit older COs to continue working.
      SINGLE_NODE_MULTI_WRITER = 5 [(alpha_enum_value) = true];

      // Indicates that Node service supports mounting volumes
      // with provided volume group identifier during node stage
      // or node publish RPC calls.
      VOLUME_MOUNT_GROUP = 6;
    }

    Type type = 1;
  }

  oneof type {
    // RPC that the controller supports.
    RPC rpc = 1;
  }
}
```

##### NodeGetCapabilities Errors

If the plugin is unable to complete the NodeGetCapabilities call successfully, it MUST return a non-ok gRPC code in the gRPC status.

#### `NodeGetInfo`

A Node Plugin MUST implement this RPC call if the plugin has `PUBLISH_UNPUBLISH_VOLUME` controller capability.
The Plugin SHALL assume that this RPC will be executed on the node where the volume will be used.
The CO SHOULD call this RPC for the node at which it wants to place the workload.
The CO MAY call this RPC more than once for a given node.
The SP SHALL NOT expect the CO to call this RPC more than once.
The result of this call will be used by CO in `ControllerPublishVolume`.

```protobuf
message NodeGetInfoRequest {
}

message NodeGetInfoResponse {
  // The identifier of the node as understood by the SP.
  // This field is REQUIRED.
  // This field MUST contain enough information to uniquely identify
  // this specific node vs all other nodes supported by this plugin.
  // This field SHALL be used by the CO in subsequent calls, including
  // `ControllerPublishVolume`, to refer to this node.
  // The SP is NOT responsible for global uniqueness of node_id across
  // multiple SPs.
  // This field overrides the general CSI size limit.
  // The size of this field SHALL NOT exceed 256 bytes. The general
  // CSI size limit, 128 byte, is RECOMMENDED for best backwards
  // compatibility.
  string node_id = 1;

  // Maximum number of volumes that controller can publish to the node.
  // If value is not set or zero CO SHALL decide how many volumes of
  // this type can be published by the controller to the node. The
  // plugin MUST NOT set negative values here.
  // This field is OPTIONAL.
  int64 max_volumes_per_node = 2;

  // Specifies where (regions, zones, racks, etc.) the node is
  // accessible from.
  // A plugin that returns this field MUST also set the
  // VOLUME_ACCESSIBILITY_CONSTRAINTS plugin capability.
  // COs MAY use this information along with the topology information
  // returned in CreateVolumeResponse to ensure that a given volume is
  // accessible from a given node when scheduling workloads.
  // This field is OPTIONAL. If it is not specified, the CO MAY assume
  // the node is not subject to any topological constraint, and MAY
  // schedule workloads that reference any volume V, such that there are
  // no topological constraints declared for V.
  //
  // Example 1:
  //   accessible_topology =
  //     {"region": "R1", "zone": "Z2"}
  // Indicates the node exists within the "region" "R1" and the "zone"
  // "Z2".
  Topology accessible_topology = 3;
}
```

##### NodeGetInfo Errors

If the plugin is unable to complete the NodeGetInfo call successfully, it MUST return a non-ok gRPC code in the gRPC status.
The CO MUST implement the specified error recovery behavior when it encounters the gRPC error code.

#### `NodeExpandVolume`

A Node Plugin MUST implement this RPC call if it has `EXPAND_VOLUME` node capability.
This RPC call allows CO to expand volume on a node.

This operation MUST be idempotent.
If a volume corresponding to the specified volume ID is already larger than or equal to the target capacity of the expansion request, the plugin SHOULD reply 0 OK.

`NodeExpandVolume` ONLY supports expansion of already node-published or node-staged volumes on the given `volume_path`.

If plugin has `STAGE_UNSTAGE_VOLUME` node capability then:
* `NodeExpandVolume` MUST be called after successful `NodeStageVolume`.
* `NodeExpandVolume` MAY be called before or after `NodePublishVolume`.

Otherwise `NodeExpandVolume` MUST be called after successful `NodePublishVolume`.

If a plugin only supports expansion via the `VolumeExpansion.OFFLINE` capability, then the volume MUST first be taken offline and expanded via `ControllerExpandVolume` (see `ControllerExpandVolume` for more details), and then node-staged or node-published before it can be expanded on the node via `NodeExpandVolume`.

The `staging_target_path` field is not required, for backwards compatibility, but the CO SHOULD supply it.
Plugins can use this field to determine if `volume_path` is where the volume is published or staged,
and setting this field to non-empty allows plugins to function with less stored state on the node.

```protobuf
message NodeExpandVolumeRequest {
  // The ID of the volume. This field is REQUIRED.
  string volume_id = 1;

  // The path on which volume is available. This field is REQUIRED.
  // This field overrides the general CSI size limit.
  // SP SHOULD support the maximum path length allowed by the operating
  // system/filesystem, but, at a minimum, SP MUST accept a max path
  // length of at least 128 bytes.
  string volume_path = 2;

  // This allows CO to specify the capacity requirements of the volume
  // after expansion. If capacity_range is omitted then a plugin MAY
  // inspect the file system of the volume to determine the maximum
  // capacity to which the volume can be expanded. In such cases a
  // plugin MAY expand the volume to its maximum capacity.
  // This field is OPTIONAL.
  CapacityRange capacity_range = 3;

  // The path where the volume is staged, if the plugin has the
  // STAGE_UNSTAGE_VOLUME capability, otherwise empty.
  // If not empty, it MUST be an absolute path in the root
  // filesystem of the process serving this request.
  // This field is OPTIONAL.
  // This field overrides the general CSI size limit.
  // SP SHOULD support the maximum path length allowed by the operating
  // system/filesystem, but, at a minimum, SP MUST accept a max path
  // length of at least 128 bytes.
  string staging_target_path = 4;

  // Volume capability describing how the CO intends to use this volume.
  // This allows SP to determine if volume is being used as a block
  // device or mounted file system. For example - if volume is being
  // used as a block device the SP MAY choose to skip expanding the
  // filesystem in NodeExpandVolume implementation but still perform
  // rest of the housekeeping needed for expanding the volume. If
  // volume_capability is omitted the SP MAY determine
  // access_type from given volume_path for the volume and perform
  // node expansion. This is an OPTIONAL field.
  VolumeCapability volume_capability = 5;

  // Secrets required by plugin to complete node expand volume request.
  // This field is OPTIONAL. Refer to the `Secrets Requirements`
  // section on how to use this field.
  map<string, string> secrets = 6
    [(csi_secret) = true, (alpha_field) = true];
}

message NodeExpandVolumeResponse {
  // The capacity of the volume in bytes. This field is OPTIONAL.
  int64 capacity_bytes = 1;
}
```

##### NodeExpandVolume Errors

| Condition             | gRPC code | Description           | Recovery Behavior                 |
|-----------------------|-----------|-----------------------|-----------------------------------|
| Exceeds capabilities | 3 INVALID_ARGUMENT | Indicates that the CO has specified capabilities not supported by the volume. | Caller MAY verify volume capabilities by calling ValidateVolumeCapabilities and retry with matching capabilities. |
| Volume does not exist | 5 NOT_FOUND | Indicates that a volume corresponding to the specified volume_id does not exist. | Caller MUST verify that the volume_id is correct and that the volume is accessible and has not been deleted before retrying with exponential back off. |
| Volume in use | 9 FAILED_PRECONDITION | Indicates that the volume corresponding to the specified `volume_id` could not be expanded because it is node-published or node-staged and the underlying filesystem does not support expansion of published or staged volumes. | Caller MUST NOT retry. |
| Unsupported capacity_range | 11 OUT_OF_RANGE | Indicates that the capacity range is not allowed by the Plugin. More human-readable information MAY be provided in the gRPC `status.message` field. | Caller MUST fix the capacity range before retrying. |

### Group Controller Service RPCs

#### `GroupControllerGetCapabilities`

A Plugin that implements GroupController MUST implement this RPC call.
This RPC allows the CO to check the supported capabilities of group controller service provided by the Plugin.

```protobuf
message GroupControllerGetCapabilitiesRequest {
  // Intentionally empty.
}

message GroupControllerGetCapabilitiesResponse {
  // All the capabilities that the group controller service supports.
  // This field is OPTIONAL.
  repeated GroupControllerServiceCapability capabilities = 1;
}

// Specifies a capability of the group controller service.
message GroupControllerServiceCapability {
  message RPC {
    enum Type {
      UNKNOWN = 0;

      // Indicates that the group controller plugin supports
      // creating, deleting, and getting details of a volume
      // group snapshot.
      CREATE_DELETE_GET_VOLUME_GROUP_SNAPSHOT = 1;
    }

    Type type = 1;
  }

  oneof type {
    // RPC that the controller supports.
    RPC rpc = 1;
  }
}
```

##### GroupControllerGetCapabilities Errors

If the plugin is unable to complete the GroupControllerGetCapabilities call successfully, it MUST return a non-ok gRPC code in the gRPC status.

#### `CreateVolumeGroupSnapshot`

A Group Controller Plugin MUST implement this RPC call if it has `CREATE_DELETE_GET_VOLUME_GROUP_SNAPSHOT` controller capability.
This RPC will be called by the CO to create a new volume group snapshot from a list of source volumes on behalf of a user.

The purpose of this call is to request the creation of a multi-volume group snapshot.
This group snapshot MUST give a write-order consistency guarantee or fail if that's not possible.
That is to say, all the of the volume snapshots in the group MUST be taken at the same point-in-time relative to a stream of write traffic to the specified volumes.
Note that calls to this function MUST be idempotent - the function may be called multiple times for the same name, with the same `source_volume_ids` and `parameters` - the group snapshot MUST only be created once.

If a group snapshot corresponding to the specified group snapshot `name`, `source_volume_ids`, and `parameters` is successfully created (meaning all snapshots associated with the group are successfully cut), the Plugin MUST reply `0 OK` with the corresponding `CreateVolumeGroupSnapshotResponse`.

If an error occurs before a group snapshot is cut, `CreateVolumeGroupSnapshot` SHOULD return a corresponding gRPC error code that reflects the error condition.

For plugins that support snapshot post processing such as uploading, CreateVolumeGroupSnapshot SHOULD return 0 OK and the ready_to_use field SHOULD be set to false after the group snapshot is cut but still being processed.
The CO SHOULD issue the CreateVolumeGroupSnapshotRequest RPC with the same arguments again periodically until the ready_to_use field has a value of true indicating all the snapshots have been "processed" and are ready to use to create new volumes.
If an error occurs during the process for any individual snapshot, CreateVolumeGroupSnapshot SHOULD return a corresponding gRPC error code that reflects the error condition.
The ready_to_use field for each individual snapshot SHOULD have a value of false until the snapshot has been "processed" and is ready to use to create new volumes.

After snapshot creation, any individual snapshot from the group MAY be used as a source to provision a new volume.

In the VolumeGroupSnapshot message, both snapshots and group_snapshot_id are required fields.

If an error occurs before all the individual snapshots are cut when creating a group snapshot of multiple volumes and a group_snapshot_id is not yet available for CO to do clean up, SP MUST return an error, and SP SHOULD also do clean up and make sure no snapshots are leaked.

```protobuf
message CreateVolumeGroupSnapshotRequest {
  // The suggested name for the group snapshot. This field is REQUIRED
  // for idempotency.
  // Any Unicode string that conforms to the length limit is allowed
  // except those containing the following banned characters:
  // U+0000-U+0008, U+000B, U+000C, U+000E-U+001F, U+007F-U+009F.
  // (These are control characters other than commonly used whitespace.)
  string name = 1;

  // volume IDs of the source volumes to be snapshotted together.
  // This field is REQUIRED.
  repeated string source_volume_ids = 2;

  // Secrets required by plugin to complete
  // ControllerCreateVolumeGroupSnapshot request.
  // This field is OPTIONAL. Refer to the `Secrets Requirements`
  // section on how to use this field.
  // The secrets provided in this field SHOULD be the same for 
  // all group snapshot operations on the same group snapshot. 
  map<string, string> secrets = 3 [(csi_secret) = true];

  // Plugin specific parameters passed in as opaque key-value pairs.
  // This field is OPTIONAL. The Plugin is responsible for parsing and
  // validating these parameters. COs will treat these as opaque.
  map<string, string> parameters = 4;
}

message CreateVolumeGroupSnapshotResponse {
  // Contains all attributes of the newly created group snapshot.
  // This field is REQUIRED.
  VolumeGroupSnapshot group_snapshot = 1;
}

message VolumeGroupSnapshot {
  // The identifier for this group snapshot, generated by the plugin.
  // This field MUST contain enough information to uniquely identify
  // this specific snapshot vs all other group snapshots supported by
  // this plugin.
  // This field SHALL be used by the CO in subsequent calls to refer to
  // this group snapshot.
  // The SP is NOT responsible for global uniqueness of
  // group_snapshot_id across multiple SPs.
  // This field is REQUIRED.
  string group_snapshot_id = 1;

  // A list of snapshots belonging to this group.
  // This field is REQUIRED.
  repeated Snapshot snapshots = 2;

  // Timestamp of when the volume group snapshot was taken.
  // This field is REQUIRED.
  .google.protobuf.Timestamp creation_time = 3;

  // Indicates if all individual snapshots in the group snapshot
  // are ready to use as a `volume_content_source` in a
  // `CreateVolumeRequest`. The default value is false.
  // If any snapshot in the list of snapshots in this message have
  // ready_to_use set to false, the SP MUST set this field to false.
  // If all of the snapshots in the list of snapshots in this message
  // have ready_to_use set to true, the SP SHOULD set this field to
  // true.
  // This field is REQUIRED.
  bool ready_to_use = 4;
}
```

##### CreateVolumeGroupSnapshot Errors

If the plugin is unable to complete the CreateVolumeGroupSnapshot call successfully, it MUST return a non-ok gRPC code in the gRPC status.
If the conditions defined below are encountered, the plugin MUST return the specified gRPC error code.
The CO MUST implement the specified error recovery behavior when it encounters the gRPC error code.

| Condition | gRPC Code | Description | Recovery Behavior |
|-----------|-----------|-------------|-------------------|
| Group snapshot already exists but is incompatible | 6 ALREADY_EXISTS | Indicates that a group snapshot corresponding to the specified group snapshot `name` already exists but is incompatible with the specified `source_volume_ids` or `parameters`. | Caller MUST fix the arguments or use a different `name` before retrying. |
| Cannot snapshot multiple volumes together | 9 FAILED_PRECONDITION | Indicates that the specified volumes cannot be snapshotted together because the volumes are not configured properly based on requirements from the SP. | Caller MUST fix the configuration of the volumes so that they meet the requirements for group snapshotting before retrying. |
| Not enough space to create group snapshot | 13 RESOURCE_EXHAUSTED | There is not enough space on the storage system to handle the create group snapshot request. | Future calls to CreateVolumeGroupSnapshot MAY succeed if space is freed up. |

#### `DeleteVolumeGroupSnapshot`

A Controller Plugin MUST implement this RPC call if it has `CREATE_DELETE_GET_VOLUME_GROUP_SNAPSHOT` capability.
This RPC will be called by the CO to delete a volume group snapshot.
This operation will delete a volume group snapshot as well as all individual snapshots that are part of this volume group snapshot.

This operation MUST be idempotent.
If a group snapshot corresponding to the specified `group_snapshot_id` does not exist or the artifacts associated with the group snapshot do not exist anymore, the Plugin MUST reply `0 OK`.

```protobuf
message DeleteVolumeGroupSnapshotRequest {
  // The ID of the group snapshot to be deleted.
  // This field is REQUIRED.
  string group_snapshot_id = 1;

  // A list of snapshot IDs that are part of this group snapshot.
  // If SP does not need to rely on this field to delete the snapshots
  // in the group, it SHOULD check this field and report an error
  // if it has the ability to detect a mismatch.
  // Some SPs require this list to delete the snapshots in the group.
  // If SP needs to use this field to delete the snapshots in the
  // group, it MUST report an error if it has the ability to detect
  // a mismatch.
  // This field is REQUIRED.
  repeated string snapshot_ids = 2;

  // Secrets required by plugin to complete group snapshot deletion
  // request.
  // This field is OPTIONAL. Refer to the `Secrets Requirements`
  // section on how to use this field.
  // The secrets provided in this field SHOULD be the same for
  // all group snapshot operations on the same group snapshot.
  map<string, string> secrets = 3 [(csi_secret) = true];
}

message DeleteVolumeGroupSnapshotResponse {
  // Intentionally empty.
}
```

##### DeleteVolumeGroupSnapshot Errors

If the plugin is unable to complete the DeleteVolumeGroupSnapshot call successfully, it MUST return a non-ok gRPC code in the gRPC status.
If the conditions defined below are encountered, the plugin MUST return the specified gRPC error code.
The CO MUST implement the specified error recovery behavior when it encounters the gRPC error code.

| Condition | gRPC Code | Description | Recovery Behavior |
|-----------|-----------|-------------|-------------------|
| Snapshot list mismatch | 3 INVALID_ARGUMENT | Besides the general cases, this code SHOULD also be used to indicate when plugin supporting CREATE_DELETE_GET_VOLUME_GROUP_SNAPSHOT detects a mismatch in the `snapshot_ids`. | If a mismatch is detected in the `snapshot_ids`, caller SHOULD use different `snapshot_ids`. |
| Volume group snapshot in use | 9 FAILED_PRECONDITION | Indicates that the volume group snapshot corresponding to the specified `group_snapshot_id` could not be deleted because it is in use by another resource. | Caller SHOULD ensure that there are no other resources using the volume group snapshot, and then retry with exponential back off. |

#### `GetVolumeGroupSnapshot`

This optional RPC MAY be called by the CO to fetch current information about a volume group snapshot.

A Controller Plugin MUST implement this `GetVolumeGroupSnapshot` RPC call if it has `CREATE_DELETE_GET_VOLUME_GROUP_SNAPSHOT` capability.

`GetVolumeGroupSnapshotResponse` should contain current information of a volume group snapshot if it exists.
If the volume group snapshot does not exist any more, `GetVolumeGroupSnapshot` should return gRPC error code `NOT_FOUND`.

```protobuf
message GetVolumeGroupSnapshotRequest {
  // The ID of the group snapshot to fetch current group snapshot
  // information for.
  // This field is REQUIRED.
  string group_snapshot_id = 1;

  // A list of snapshot IDs that are part of this group snapshot.
  // If SP does not need to rely on this field to get the snapshots
  // in the group, it SHOULD check this field and report an error
  // if it has the ability to detect a mismatch.
  // Some SPs require this list to get the snapshots in the group.
  // If SP needs to use this field to get the snapshots in the
  // group, it MUST report an error if it has the ability to detect
  // a mismatch.
  // This field is REQUIRED.
  repeated string snapshot_ids = 2;

  // Secrets required by plugin to complete
  // GetVolumeGroupSnapshot request.
  // This field is OPTIONAL. Refer to the `Secrets Requirements`
  // section on how to use this field.
  // The secrets provided in this field SHOULD be the same for
  // all group snapshot operations on the same group snapshot.
  map<string, string> secrets = 3 [(csi_secret) = true];
}

message GetVolumeGroupSnapshotResponse {
  // This field is REQUIRED
  VolumeGroupSnapshot group_snapshot = 1;
}
```

##### GetVolumeGroupSnapshot Errors

If the plugin is unable to complete the GetVolumeGroupSnapshot call successfully, it MUST return a non-ok gRPC code in the gRPC status.
If the conditions defined below are encountered, the plugin MUST return the specified gRPC error code.
The CO MUST implement the specified error recovery behavior when it encounters the gRPC error code.

| Condition | gRPC Code | Description | Recovery Behavior |
|-----------|-----------|-------------|-------------------|
| Snapshot list mismatch | 3 INVALID_ARGUMENT | Besides the general cases, this code SHOULD also be used to indicate when plugin supporting CREATE_DELETE_GET_VOLUME_GROUP_SNAPSHOT detects a mismatch in the `snapshot_ids`. | If a mismatch is detected in the `snapshot_ids`, caller SHOULD use different `snapshot_ids`. |
| Volume group snapshot does not exist | 5 NOT_FOUND | Indicates that a volume group snapshot corresponding to the specified `group_snapshot_id` does not exist. | Caller MUST verify that the `group_snapshot_id` is correct and that the volume group snapshot is accessible and has not been deleted before retrying with exponential back off. |

### Snapshot Metadata Service RPCs

**ALPHA FEATURE**

The Snapshot Metadata service is an optional service that is used to retrieve metadata on the allocated blocks of a single snapshot, or the changed blocks between a pair of snapshots of the same block volume.
Retrieval of the data blocks of a snapshot is not addressed by this service and it is assumed that existing mechanisms to read snapshot data will suffice.

#### Metadata Format

Block volume data ranges are specified by a sequence of `BlockMetadata` tuples.
Tuples in a sequence are in ascending order of `byte_offset`, with no overlap between adjacent tuples.

```protobuf
// BlockMetadata specifies a data range.
message BlockMetadata {
  // This is the zero based byte position in the volume or snapshot,
  // measured from the start of the object.
  // This field is REQUIRED.
  int64 byte_offset = 1;

  // This is the size of the data range.
  // size_bytes MUST be greater than zero.
  // This field is REQUIRED.
  int64 size_bytes = 2;
}
```

The `BlockMetadataType` enumerated type describes how the `size_bytes` fields of `BlockMetadata` tuples may vary in a sequence.
There are two prevalent styles in which data ranges are described:

- The **FIXED_LENGTH** style requires a fixed value for the `size_bytes` field in all tuples in a given sequence.
- The **VARIABLE_LENGTH** style does not constrain the value of the `size_bytes` field in a sequence.

The Snapshot Metadata service permits either style at the discretion of the plugin as long as the style does not change mid-stream in any given RPC.
The style is represented by the following data type:

```protobuf
enum BlockMetadataType {
  UNKNOWN = 0;

  // The FIXED_LENGTH value indicates that data ranges are
  // returned in fixed size blocks.
  FIXED_LENGTH = 1;

  // The VARIABLE_LENGTH value indicates that data ranges
  // are returned in potentially variable sized extents.
  VARIABLE_LENGTH = 2;
}
```

### Special considerations for processing a snapshot metadata gRPC stream

The remote procedure calls of this service return snapshot metadata within a gRPC stream of response messages.
There are some important things for the SP and CO to consider here:

- In normal operation an SP MUST terminate the stream only after ALL metadata is transmitted, using the language specific idiom for terminating a gRPC stream source.
  This results in the CO receiving its language specific notification idiom for the end of a gRPC stream, which provides a **definitive indication that all available metadata has been received**.

- If the SP encounters an error while recovering the metadata it MUST abort transmission of the stream with its language specific error idiom.
  This results in the CO receiving the error in its language specific idiom, which will be different from the language specific idiom for the end of a gRPC stream.

- It is possible that the gRPC stream gets interrupted for arbitrary reasons beyond the control of either the SP or the CO.
  The SP will get an error when writing to the stream and MUST abort its transmission.
  The CO will receive an error in its language specific idiom, which will be different from the language specific idiom for the end of a gRPC stream.

- In all circumstances where the CO receives an error when reading from the gRPC stream it MAY attempt to **continue** the operation by re-sending its request message but with a `starting_offset` adjusted to the NEXT byte position beyond that of any metadata already received.
  The SP MUST always ensure that the `starting_offset` requested be considered in the computation of the data range for the first message in the returned gRPC stream, though the data range of the first message is not required to actually include the `starting_offset` if there is no applicable data between the `starting_offset` and the start of the data range returned by the first message.

#### `GetMetadataAllocated`

The plugin must implement this RPC if it provides the SnapshotMetadata service.

```protobuf
// The GetMetadataAllocatedRequest message is used to solicit metadata
// on the allocated blocks of a snapshot: i.e. this identifies the
// data ranges that have valid data as they were the target of some
// previous write operation on the volume.
message GetMetadataAllocatedRequest {
  // This is the identifier of the snapshot.
  // This field is REQUIRED.
  string snapshot_id = 1;

  // This indicates the zero based starting byte position in the volume
  // snapshot from which the result should be computed.
  // It is intended to be used to continue a previously interrupted
  // call.
  // The CO SHOULD specify this value to be the offset of the byte
  // position immediately after the last byte of the last data range
  // received, if continuing an interrupted operation, or zero if not.
  // The SP MUST ensure that the returned response stream does not
  // contain BlockMetadata tuples that end before the requested
  // starting_offset: i.e. if S is the requested starting_offset, and
  // B0 is block_metadata[0] of the first message in the response
  // stream, then (S < B0.byte_offset + B0.size_bytes) must be true.
  // This field is REQUIRED.
  int64 starting_offset = 2;

  // This is an optional parameter, and if non-zero it specifies the
  // maximum number of tuples to be returned in each
  // GetMetadataAllocatedResponse message returned by the RPC stream.
  // The plugin will determine an appropriate value if 0, and is
  // always free to send less than the requested value.
  // This field is OPTIONAL.
  int32 max_results = 3;

  // Secrets required by plugin to complete the request.
  // This field is OPTIONAL. Refer to the `Secrets Requirements`
  // section on how to use this field.
  map<string, string> secrets = 4 [(csi_secret) = true];
}

// GetMetadataAllocatedResponse messages are returned in a gRPC stream.
// Cumulatively, they provide information on the allocated data
// ranges in the snapshot.
message GetMetadataAllocatedResponse {
  // This specifies the style used in the BlockMetadata sequence.
  // This value must be the same in all such messages returned by
  // the stream.
  // If block_metadata_type is FIXED_LENGTH, then the size_bytes field
  // of each message in the block_metadata list MUST be constant.
  // This field is REQUIRED.
  BlockMetadataType block_metadata_type = 1;

  // This returns the capacity of the underlying volume in bytes.
  // This value must be the same in all such messages returned by
  // the stream.
  // This field is REQUIRED.
  int64 volume_capacity_bytes = 2;

  // This is a list of data range tuples.
  // If the value of max_results in the GetMetadataAllocatedRequest
  // message is greater than zero, then the number of entries in this
  // list MUST be less than or equal to that value.
  // The SP MUST respect the value of starting_offset in the request.
  // The byte_offset fields of adjacent BlockMetadata messages
  // MUST be strictly increasing and messages MUST NOT overlap:
  // i.e. for any two BlockMetadata messages, A and B, if A is returned
  // before B, then (A.byte_offset + A.size_bytes <= B.byte_offset)
  // MUST be true.
  // This MUST also be true if A and B are from block_metadata lists in
  // different GetMetadataAllocatedResponse messages in the gRPC stream.
  // This field is OPTIONAL.
  repeated BlockMetadata block_metadata = 3;
}
```

##### GetMetadataAllocated Errors
If the plugin is unable to complete the `GetMetadataAllocated` call successfully it must return a non-OK gRPC code in the gRPC status.

The following conditions are well defined:

| Condition | gRPC Code | Description | Recovery Behavior |
|-----------|-----------|-------------|-------------------|
| Missing or otherwise invalid argument | 3 INVALID_ARGUMENT | Indicates that a required argument field was not specified or an argument value is invalid | The caller should correct the error and resubmit the call. |
| Invalid `snapshot_id` | 5 NOT_FOUND | Indicates that the snapshot specified was not found. | The caller should re-check that this object exists. |
| Invalid `starting_offset` | 11 OUT_OF_RANGE | The starting offset is negative or exceeds the volume size. | The caller should specify a valid offset. |

#### `GetMetadataDelta`

The plugin must implement this RPC if it provides the SnapshotMetadata service.

```protobuf
// The GetMetadataDeltaRequest message is used to solicit metadata on
// the data ranges that have changed between two snapshots.
message GetMetadataDeltaRequest {
  // This is the identifier of the snapshot against which changes
  // are to be computed.
  // This field is REQUIRED.
  string base_snapshot_id = 1;

  // This is the identifier of a second snapshot in the same volume,
  // created after the base snapshot.
  // This field is REQUIRED.
  string target_snapshot_id = 2;

  // This indicates the zero based starting byte position in the volume
  // snapshot from which the result should be computed.
  // It is intended to be used to continue a previously interrupted
  // call.
  // The CO SHOULD specify this value to be the offset of the byte
  // position immediately after the last byte of the last data range
  // received, if continuing an interrupted operation, or zero if not.
  // The SP MUST ensure that the returned response stream does not
  // contain BlockMetadata tuples that end before the requested
  // starting_offset: i.e. if S is the requested starting_offset, and
  // B0 is block_metadata[0] of the first message in the response
  // stream, then (S < B0.byte_offset + B0.size_bytes) must be true.
  // This field is REQUIRED.
  int64 starting_offset = 3;

  // This is an optional parameter, and if non-zero it specifies the
  // maximum number of tuples to be returned in each
  // GetMetadataDeltaResponse message returned by the RPC stream.
  // The plugin will determine an appropriate value if 0, and is
  // always free to send less than the requested value.
  // This field is OPTIONAL.
  int32 max_results = 4;

  // Secrets required by plugin to complete the request.
  // This field is OPTIONAL. Refer to the `Secrets Requirements`
  // section on how to use this field.
  map<string, string> secrets = 5 [(csi_secret) = true];
}

// GetMetadataDeltaResponse messages are returned in a gRPC stream.
// Cumulatively, they provide information on the data ranges that
// have changed between the base and target snapshots specified
// in the GetMetadataDeltaRequest message.
message GetMetadataDeltaResponse {
  // This specifies the style used in the BlockMetadata sequence.
  // This value must be the same in all such messages returned by
  // the stream.
  // If block_metadata_type is FIXED_LENGTH, then the size_bytes field
  // of each message in the block_metadata list MUST be constant.
  // This field is REQUIRED.
  BlockMetadataType block_metadata_type = 1;

  // This returns the capacity of the underlying volume in bytes.
  // This value must be the same in all such messages returned by
  // the stream.
  // This field is REQUIRED.
  int64 volume_capacity_bytes = 2;

  // This is a list of data range tuples.
  // If the value of max_results in the GetMetadataDeltaRequest message
  // is greater than zero, then the number of entries in this list MUST
  // be less than or equal to that value.
  // The SP MUST respect the value of starting_offset in the request.
  // The byte_offset fields of adjacent BlockMetadata messages
  // MUST be strictly increasing and messages MUST NOT overlap:
  // i.e. for any two BlockMetadata messages, A and B, if A is returned
  // before B, then (A.byte_offset + A.size_bytes <= B.byte_offset)
  // MUST be true.
  // This MUST also be true if A and B are from block_metadata lists in
  // different GetMetadataDeltaResponse messages in the gRPC stream.
  // This field is OPTIONAL.
  repeated BlockMetadata block_metadata = 3;
}
```

##### GetMetadataDelta Errors
If the plugin is unable to complete the `GetMetadataDelta` call successfully it must return a non-OK gRPC code in the gRPC status.

The following conditions are well defined:

| Condition | gRPC Code | Description | Recovery Behavior |
|-----------|-----------|-------------|-------------------|
| Missing or otherwise invalid argument | 3 INVALID_ARGUMENT | Indicates that a required argument field was not specified or an argument value is invalid | The caller should correct the error and resubmit the call. |
| Invalid `base_snapshot_id` or `target_snapshot_id` | 5 NOT_FOUND | Indicates that the snapshots specified were not found. | The caller should re-check that these objects exist. |
| Invalid `starting_offset` | 11 OUT_OF_RANGE | The starting offset is negative or exceeds the volume size. | The caller should specify a valid offset. |

## Protocol

### Connectivity

* A CO SHALL communicate with a Plugin using gRPC to access the `Identity`, and (optionally) the `Controller` and `Node` services.
  * proto3 SHOULD be used with gRPC, as per the [official recommendations](http://www.grpc.io/docs/guides/#protocol-buffer-versions).
  * All Plugins SHALL implement the REQUIRED Identity service RPCs.
    Support for OPTIONAL RPCs is reported by the `ControllerGetCapabilities` and `NodeGetCapabilities` RPC calls.
* The CO SHALL provide the listen-address for the Plugin by way of the `CSI_ENDPOINT` environment variable.
  Plugin components SHALL create, bind, and listen for RPCs on the specified listen address.
  * Only UNIX Domain Sockets MAY be used as endpoints.
    This will likely change in a future version of this specification to support non-UNIX platforms.
* All supported RPC services MUST be available at the listen address of the Plugin.

### Security

* The CO operator and Plugin Supervisor SHOULD take steps to ensure that any and all communication between the CO and Plugin Service are secured according to best practices.
* Communication between a CO and a Plugin SHALL be transported over UNIX Domain Sockets.
  * gRPC is compatible with UNIX Domain Sockets; it is the responsibility of the CO operator and Plugin Supervisor to properly secure access to the Domain Socket using OS filesystem ACLs and/or other OS-specific security context tooling.
  * SP’s supplying stand-alone Plugin controller appliances, or other remote components that are incompatible with UNIX Domain Sockets MUST provide a software component that proxies communication between a UNIX Domain Socket and the remote component(s).
    Proxy components transporting communication over IP networks SHALL be responsible for securing communications over such networks.
* Both the CO and Plugin SHOULD avoid accidental leakage of sensitive information (such as redacting such information from log files).

### Debugging

* Debugging and tracing are supported by external, CSI-independent additions and extensions to gRPC APIs, such as [OpenTracing](https://github.com/grpc-ecosystem/grpc-opentracing).

## Configuration and Operation

### General Configuration

* The `CSI_ENDPOINT` environment variable SHALL be supplied to the Plugin by the Plugin Supervisor.
* An operator SHALL configure the CO to connect to the Plugin via the listen address identified by `CSI_ENDPOINT` variable.
* With exception to sensitive data, Plugin configuration SHOULD be specified by environment variables, whenever possible, instead of by command line flags or bind-mounted/injected files.


#### Plugin Bootstrap Example

* Supervisor -> Plugin: `CSI_ENDPOINT=unix:///path/to/unix/domain/socket.sock`.
* Operator -> CO: use plugin at endpoint `unix:///path/to/unix/domain/socket.sock`.
* CO: monitor `/path/to/unix/domain/socket.sock`.
* Plugin: read `CSI_ENDPOINT`, create UNIX socket at specified path, bind and listen.
* CO: observe that socket now exists, establish connection.
* CO: invoke `GetPluginCapabilities`.

#### Filesystem

* Plugins SHALL NOT specify requirements that include or otherwise reference directories and/or files on the root filesystem of the CO.
* Plugins SHALL NOT create additional files or directories adjacent to the UNIX socket specified by `CSI_ENDPOINT`; violations of this requirement constitute "abuse".
  * The Plugin Supervisor is the ultimate authority of the directory in which the UNIX socket endpoint is created and MAY enforce policies to prevent and/or mitigate abuse of the directory by Plugins.

### Supervised Lifecycle Management

* For Plugins packaged in software form:
  * Plugin Packages SHOULD use a well-documented container image format (e.g., Docker, OCI).
  * The chosen package image format MAY expose configurable Plugin properties as environment variables, unless otherwise indicated in the section below.
    Variables so exposed SHOULD be assigned default values in the image manifest.
  * A Plugin Supervisor MAY programmatically evaluate or otherwise scan a Plugin Package’s image manifest in order to discover configurable environment variables.
  * A Plugin SHALL NOT assume that an operator or Plugin Supervisor will scan an image manifest for environment variables.

#### Environment Variables

* Variables defined by this specification SHALL be identifiable by their `CSI_` name prefix.
* Configuration properties not defined by the CSI specification SHALL NOT use the same `CSI_` name prefix; this prefix is reserved for common configuration properties defined by the CSI specification.
* The Plugin Supervisor SHOULD supply all RECOMMENDED CSI environment variables to a Plugin.
* The Plugin Supervisor SHALL supply all REQUIRED CSI environment variables to a Plugin.

##### `CSI_ENDPOINT`

Network endpoint at which a Plugin SHALL host CSI RPC services. The general format is:

    {scheme}://{authority}{endpoint}

The following address types SHALL be supported by Plugins:

    unix:///path/to/unix/socket.sock

Note: All UNIX endpoints SHALL end with `.sock`. See [gRPC Name Resolution](https://github.com/grpc/grpc/blob/master/doc/naming.md).

This variable is REQUIRED.

#### Operational Recommendations

The Plugin Supervisor expects that a Plugin SHALL act as a long-running service vs. an on-demand, CLI-driven process.

Supervised plugins MAY be isolated and/or resource-bounded.

##### Logging

* Plugins SHOULD generate log messages to ONLY standard output and/or standard error.
  * In this case the Plugin Supervisor SHALL assume responsibility for all log lifecycle management.
* Plugin implementations that deviate from the above recommendation SHALL clearly and unambiguously document the following:
  * Logging configuration flags and/or variables, including working sample configurations.
  * Default log destination(s) (where do the logs go if no configuration is specified?)
  * Log lifecycle management ownership and related guidance (size limits, rate limits, rolling, archiving, expunging, etc.) applicable to the logging mechanism embedded within the Plugin.
* Plugins SHOULD NOT write potentially sensitive data to logs (e.g. secrets).

##### Available Services

* Plugin Packages MAY support all or a subset of CSI services; service combinations MAY be configurable at runtime by the Plugin Supervisor.
  * A plugin MUST know the "mode" in which it is operating (e.g. node, controller, or both).
  * This specification does not dictate the mechanism by which mode of operation MUST be discovered, and instead places that burden upon the SP.
* Misconfigured plugin software SHOULD fail-fast with an OS-appropriate error code.

##### Linux Capabilities

* Plugin Supervisor SHALL guarantee that plugins will have `CAP_SYS_ADMIN` capability on Linux when running on Nodes.
* Plugins SHOULD clearly document any additionally required capabilities and/or security context.

##### Namespaces

* A Plugin SHOULD NOT assume that it is in the same [Linux namespaces](https://en.wikipedia.org/wiki/Linux_namespaces) as the Plugin Supervisor.
  The CO MUST clearly document the [mount propagation](https://www.kernel.org/doc/Documentation/filesystems/sharedsubtree.txt) requirements for Node Plugins and the Plugin Supervisor SHALL satisfy the CO’s requirements.

##### Cgroups Isolation

* A Plugin MAY be constrained by cgroups.
* An operator or Plugin Supervisor MAY configure the devices cgroups subsystem to ensure that a Plugin MAY access requisite devices.
* A Plugin Supervisor MAY define resource limits for a Plugin.

##### Resource Requirements

* SPs SHOULD unambiguously document all of a Plugin’s resource requirements.