# 03 - Wire Protocol

FAP-1 wire protocol 是 Protobuf（proto3）。任何 TransportPlugin 都承载 Envelope 消息。

## 1. 顶层消息

```proto
syntax = "proto3";
package fap.v1;

import "google/protobuf/timestamp.proto";
import "google/protobuf/struct.proto";

message Envelope {
  Header header = 1;
  reserved 2 to 9;

  oneof body {
    ClientHello client_hello = 10;
    ServerHello server_hello = 11;
    reserved 12 to 19;

    AuthInit auth_init = 20;
    AuthAck auth_ack = 21;
    AuthReject auth_reject = 22;
    AuthRevoke auth_revoke = 23;
    reserved 24 to 29;

    CapabilityBind capability_bind = 30;
    CapabilityAck capability_ack = 31;
    reserved 32 to 39;

    InvokeRequest invoke_request = 40;
    InvokeAck invoke_ack = 41;
    InvokeResult invoke_result = 42;
    InvokeCancel invoke_cancel = 43;
    reserved 44 to 49;

    BatchInvokeRequest batch_invoke_request = 50;
    BatchInvokeAck batch_invoke_ack = 51;
    BatchInvokeResult batch_invoke_result = 52;
    reserved 53 to 59;

    ProgressEvent progress_event = 60;
    Problem problem = 61;
    reserved 62 to 69;

    DataOpen data_open = 70;
    DataChunk data_chunk = 71;
    DataCommit data_commit = 72;
    DataAck data_ack = 73;
    ObjectLeaseRenew object_lease_renew = 74;
    ObjectLeaseRenewAck object_lease_renew_ack = 75;
    reserved 76 to 79;

    NodeHello node_hello = 80;
    RegisterCapabilities register_capabilities = 81;
    RemoteInvoke remote_invoke = 82;
    NodeHealth node_health = 83;
    reserved 84 to 89;

    ResumeRequest resume_request = 90;
    ResumeAck resume_ack = 91;
    reserved 92 to 99;

    Receipt receipt = 100;
    ReceiptFinalized receipt_finalized = 101;
    reserved 102 to 109;

    Close close = 110;
    reserved 111 to 119;

    FlowCredit flow_credit = 120;
    DataPause data_pause = 121;
    DataResume data_resume = 122;

    google.protobuf.Any extension_body = 900;
  }

  reserved 200 to 255;
  repeated Signature signatures = 256;
}
```

## 2. Header

```proto
message Header {
  string protocol_version = 1;
  string min_protocol_version = 2;

  uint32 wire_major = 4;
  uint32 wire_minor = 5;

  string message_id = 6;
  string thread_id = 7;
  string session_id = 8;

  uint64 sequence_no = 9;
  string idempotency_key = 10;

  string source_agent_id = 11;
  string target_agent_id = 12;

  google.protobuf.Timestamp created_at = 13;
  google.protobuf.Timestamp expires_at = 14;

  string trace_id = 15;
  string parent_message_id = 16;

  bytes payload_sha256 = 17;
  bytes prev_hash = 18;

  Priority priority = 19;

  map<string, string> extensions = 100;
}

enum Priority {
  PRIORITY_UNSPECIFIED = 0;
  PRIORITY_CONTROL = 1;
  PRIORITY_AUDIT = 2;
  PRIORITY_DATA = 3;
}
```

## 3. Signature

```proto
message Signature {
  string alg = 1;
  string kid = 2;
  bytes value = 3;
}
```

签名字节规则见 [signing-canonical.md](./signing-canonical.md)。

## 4. Hello / Auth

```proto
message ClientHello {
  string client_agent_id = 1;
  string client_version = 2;

  string protocol_version = 3;
  VersionRange supported_protocol_range = 4;

  repeated string supported_transports = 5;
  repeated ProfileVersion supported_profiles = 6;
  repeated AuthMode supported_auth_modes = 7;

  uint32 max_control_frame_bytes = 8;
  uint32 max_data_chunk_bytes = 9;
  uint32 max_concurrent_streams = 10;

  bytes client_nonce = 11;
}

message ServerHello {
  string server_agent_id = 1;
  string server_version = 2;

  string selected_protocol_version = 3;
  repeated ProfileVersion selected_profiles = 4;
  repeated string rejected_profiles = 5;

  AuthMode selected_auth_mode = 6;
  string session_id = 7;

  bytes server_nonce = 8;
  bytes handshake_transcript_hash = 9;

  repeated Capability advertised_capabilities = 10;
  repeated CompatibilityWarning compatibility_warnings = 11;

  bytes version_commitment = 12;
}

message VersionRange {
  string min = 1;
  string max = 2;
}

message ProfileVersion {
  string id = 1;
  string version = 2;
}

message CompatibilityWarning {
  string code = 1;
  string message = 2;
  string affected_feature = 3;
}

enum AuthMode {
  AUTH_MODE_UNSPECIFIED = 0;
  AUTH_MODE_MTLS_JWT = 1;
  AUTH_MODE_DID_VC = 2;
  AUTH_MODE_CUSTOM = 99;
}

message AuthInit {
  AuthMode mode = 1;
  bytes credential = 2;
  map<string, string> metadata = 3;
}

message AuthAck {
  string subject = 1;
  repeated string granted_scopes = 2;
  repeated string granted_capabilities = 3;
  google.protobuf.Timestamp expires_at = 4;
  string auth_context_id = 5;
}

message AuthReject {
  string code = 1;
  string message = 2;
  bool retryable = 3;
  string required_version = 4;
  string current_version = 5;
}

message AuthRevoke {
  string session_id = 1;
  string mandate_id = 2;
  string reason = 3;
  uint32 retry_after_ms = 4;
  RevokeScope scope = 5;
}

enum RevokeScope {
  REVOKE_SCOPE_UNSPECIFIED = 0;
  REVOKE_MANDATE = 1;
  REVOKE_SESSION = 2;
  REVOKE_SUBJECT = 3;
}
```

## 5. Invoke

```proto
message InvokeRequest {
  string invocation_id = 1;
  string capability_id = 2;

  google.protobuf.Struct input = 3;
  repeated ObjectRef input_objects = 4;

  bool stream_result = 5;
  uint32 timeout_ms = 6;

  RiskLevel declared_risk_level = 7;
  string mandate_id = 8;

  repeated string depends_on = 30;
  map<string, string> options = 100;
}

message InvokeAck {
  string invocation_id = 1;
  string status = 2;
  uint32 estimated_queue_ms = 3;
}

message InvokeResult {
  string invocation_id = 1;
  string status = 2;

  google.protobuf.Struct output = 3;
  repeated ObjectRef output_objects = 4;

  ReceiptRef receipt = 10;
}

message InvokeCancel {
  string invocation_id = 1;
  string reason = 2;
}

message BatchInvokeRequest {
  string batch_id = 1;
  repeated InvokeRequest requests = 2;
  BatchPolicy policy = 3;
}

message BatchPolicy {
  uint32 max_parallelism = 1;
  bool fail_fast = 2;
  BatchOrderPolicy order = 3;
  uint32 timeout_ms = 4;
}

enum BatchOrderPolicy {
  BATCH_ORDER_UNSPECIFIED = 0;
  BATCH_ORDER_UNORDERED = 1;
  BATCH_ORDER_RESULT_PRESERVED = 2;
  BATCH_ORDER_SEQUENTIAL = 3;
  BATCH_ORDER_DAG = 4;
}

message BatchInvokeAck {
  string batch_id = 1;
  string status = 2;
  uint32 accepted_count = 3;
}

message BatchInvokeResult {
  string batch_id = 1;
  repeated InvokeResult results = 2;
  ReceiptRef batch_receipt = 3;
}
```

## 6. Data Plane

见 07-data-plane.md。

## 7. Capability / Receipt

见 08-capability-model.md 与 11-receipt-audit.md。

## 8. 校验顺序

```text
1. Decode Protobuf
2. 校验 wire_major/wire_minor
3. 校验 message_id 格式
4. 校验 expires_at ≤ session.max_ttl
5. 校验 Replay Cache
6. 校验 session_id
7. 校验 payload_sha256
8. 校验 prev_hash 指向链内
9. 校验 signatures（按 signing-canonical）
10. 分发 handler
```
