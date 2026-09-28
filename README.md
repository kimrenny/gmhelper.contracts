# GMHelper Contracts

This repository contains the canonical, authoritative Protobuf (gRPC) contracts for the GMHelper ecosystem.

## Workspace Layout Convention

The GMHelper ecosystem relies on a multi-repository workspace layout where all repositories are checked out as siblings in the same root folder:

```text
<workspace_root>/
├── gmhelper.contracts/        # Canonical protobuf contract repository
│   └── solution_hub.proto
├── gmhelper-api/              # .NET Backend Monolith
│   └── MatHelper.Protos/
├── gmhelper-hub-api/          # Go AI Solver Microservice
│   └── proto/
└── gmhelper-web/              # Angular Frontend
```

---

## Canonical Service & RPC Definition

[`solution_hub.proto`](./solution_hub.proto):

```protobuf
syntax = "proto3";

package solutionhub;

option csharp_namespace = "SolutionHub";
option go_package = "gmhelper.solution-hub/proto;proto";

service SolutionHub {
  rpc SolveProblem (SolveProblemRequest) returns (SolveProblemResponse);
}

message SolveProblemRequest {
  string task_id = 1;
  string problem_type = 2;
  string payload = 3;
  string user_id = 4;
}

message SolveProblemResponse {
  string task_id = 1;
  string status = 2;
  string result = 3;
  bool success = 4;
}
```

---

## Consumer Integration & Build Workflow

### 1. `gmhelper-api` (.NET)
* **Integration**: `MatHelper.Protos/MatHelper.Protos.csproj` references `..\..\gmhelper.contracts\solution_hub.proto` directly via MSBuild `<Protobuf>` linking.
* **Build Requirement**: `gmhelper.contracts` must be present as a sibling directory at build time. `dotnet build` uses `Grpc.Tools` to generate C# client stubs on the fly without checking generated files into git.
* **CI/CD Note**: In isolated CI runners (e.g. GitHub Actions), checkout `gmhelper.contracts` as a sibling directory before running `dotnet build`.

### 2. `gmhelper-hub-api` (Go)
* **Integration**: `proto/solution_hub.pb.go` and `proto/solution_hub_grpc.pb.go` are derived stubs generated from `../gmhelper.contracts/solution_hub.proto`.
* **Build Requirement**: `gmhelper-hub-api` checks in its generated Go protobuf files. As a result, `gmhelper-hub-api` can be cloned, built, tested, and containerized independently in Docker/CI without requiring `protoc` or `gmhelper.contracts` at build time.
* **Regeneration**: Whenever `solution_hub.proto` is updated, regenerate Go stubs by running:
  ```bash
  go generate ./...
  ```
  *(Requires `gmhelper.contracts` in the sibling directory, `protoc`, `protoc-gen-go`, and `protoc-gen-go-grpc`)*.

---

## Contract Update Lifecycle
1. Modify `solution_hub.proto` in `gmhelper.contracts`.
2. In `gmhelper-hub-api`, run `go generate ./...` to regenerate Go stubs.
3. In `gmhelper-api`, run `dotnet build` (automatically compiles latest proto).
4. Run test suites in both consumer repositories:
   * `dotnet test MatHelper.sln`
   * `go test ./...`
