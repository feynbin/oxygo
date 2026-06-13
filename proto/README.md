# Protocol Buffers

此目录存放 oxygo 的 gRPC 协议定义源文件（.proto）。

## 服务定义

```protobuf
syntax = "proto3";
package oxygo.v1;

// admin <-> server 控制面
service NodeAgent {
  rpc Register(RegisterRequest) returns (RegisterResponse);
  rpc WatchConfig(WatchRequest) returns (stream ConfigRevision);
  rpc ReportStats(stream NodeStats) returns (ReportAck);
  rpc ExecCommand(CommandRequest) returns (stream CommandChunk);
}
```

## 生成命令

```bash
protoc --go_out=. --go-grpc_out=. proto/oxygo/v1/*.proto
```
