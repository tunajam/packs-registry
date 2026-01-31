# gRPC Patterns

Best practices for gRPC service design, protobuf definitions, streaming, and error handling.

## When to Apply

- Designing gRPC services and APIs
- Writing protobuf definitions
- Implementing streaming RPCs
- Handling errors and status codes
- Setting up interceptors/middleware
- Optimizing performance

## Protobuf Design

### File Structure
```protobuf
// api/v1/user.proto
syntax = "proto3";

package api.v1;

option go_package = "github.com/myorg/myapp/gen/api/v1;apiv1";

import "google/protobuf/timestamp.proto";
import "google/protobuf/empty.proto";

// Service definition
service UserService {
  // Unary RPC
  rpc GetUser(GetUserRequest) returns (GetUserResponse);
  rpc CreateUser(CreateUserRequest) returns (CreateUserResponse);
  rpc UpdateUser(UpdateUserRequest) returns (UpdateUserResponse);
  rpc DeleteUser(DeleteUserRequest) returns (google.protobuf.Empty);
  
  // Server streaming
  rpc ListUsers(ListUsersRequest) returns (stream User);
  
  // Client streaming
  rpc BatchCreateUsers(stream CreateUserRequest) returns (BatchCreateUsersResponse);
  
  // Bidirectional streaming
  rpc SyncUsers(stream SyncRequest) returns (stream SyncResponse);
}

// Messages
message User {
  string id = 1;
  string email = 2;
  string name = 3;
  UserStatus status = 4;
  google.protobuf.Timestamp created_at = 5;
  google.protobuf.Timestamp updated_at = 6;
}

enum UserStatus {
  USER_STATUS_UNSPECIFIED = 0;
  USER_STATUS_ACTIVE = 1;
  USER_STATUS_INACTIVE = 2;
  USER_STATUS_SUSPENDED = 3;
}

message GetUserRequest {
  string id = 1;
}

message GetUserResponse {
  User user = 1;
}

message CreateUserRequest {
  string email = 1;
  string name = 2;
}

message CreateUserResponse {
  User user = 1;
}
```

### Naming Conventions
```protobuf
// Package: lowercase with dots
package mycompany.myservice.v1;

// Messages: PascalCase
message UserAccount {}

// Fields: snake_case
string user_name = 1;

// Enums: SCREAMING_SNAKE_CASE with prefix
enum Status {
  STATUS_UNSPECIFIED = 0;  // Always have zero value
  STATUS_ACTIVE = 1;
}

// Services: PascalCase with "Service" suffix
service UserService {}

// RPCs: PascalCase verb phrases
rpc GetUser(GetUserRequest) returns (GetUserResponse);
rpc CreateUser(CreateUserRequest) returns (CreateUserResponse);
rpc ListUsers(ListUsersRequest) returns (ListUsersResponse);
```

### Field Numbering Rules
```protobuf
message Example {
  // 1-15: Single byte encoding (use for frequent fields)
  string id = 1;
  string name = 2;
  
  // 16-2047: Two byte encoding
  string description = 16;
  
  // Reserved: Never reuse deleted field numbers
  reserved 3, 4, 100 to 200;
  reserved "old_field_name";
}
```

## Service Patterns

### Request/Response Wrappers
```protobuf
// Good: Wrapped messages (extensible)
message GetUserRequest {
  string id = 1;
  FieldMask field_mask = 2;  // Can add fields later
}

message GetUserResponse {
  User user = 1;
}

// Bad: Unwrapped (can't add metadata)
rpc GetUser(string) returns (User);  // Don't do this
```

### Pagination
```protobuf
message ListUsersRequest {
  int32 page_size = 1;
  string page_token = 2;  // Opaque cursor
  string filter = 3;       // Optional filter
  string order_by = 4;     // Optional ordering
}

message ListUsersResponse {
  repeated User users = 1;
  string next_page_token = 2;  // Empty if no more pages
  int32 total_size = 3;        // Optional total count
}
```

### Partial Updates (Field Masks)
```protobuf
import "google/protobuf/field_mask.proto";

message UpdateUserRequest {
  User user = 1;
  google.protobuf.FieldMask update_mask = 2;
}

// Usage: update_mask = "name,email"
// Only specified fields are updated
```

## Go Implementation

### Server
```go
package main

import (
    "context"
    "log"
    "net"

    "google.golang.org/grpc"
    "google.golang.org/grpc/codes"
    "google.golang.org/grpc/status"
    
    pb "github.com/myorg/myapp/gen/api/v1"
)

type userServer struct {
    pb.UnimplementedUserServiceServer
    users map[string]*pb.User
}

func (s *userServer) GetUser(ctx context.Context, req *pb.GetUserRequest) (*pb.GetUserResponse, error) {
    user, ok := s.users[req.Id]
    if !ok {
        return nil, status.Errorf(codes.NotFound, "user %s not found", req.Id)
    }
    return &pb.GetUserResponse{User: user}, nil
}

func (s *userServer) CreateUser(ctx context.Context, req *pb.CreateUserRequest) (*pb.CreateUserResponse, error) {
    if req.Email == "" {
        return nil, status.Error(codes.InvalidArgument, "email is required")
    }
    
    user := &pb.User{
        Id:    generateID(),
        Email: req.Email,
        Name:  req.Name,
    }
    s.users[user.Id] = user
    
    return &pb.CreateUserResponse{User: user}, nil
}

func main() {
    lis, err := net.Listen("tcp", ":50051")
    if err != nil {
        log.Fatalf("failed to listen: %v", err)
    }
    
    grpcServer := grpc.NewServer()
    pb.RegisterUserServiceServer(grpcServer, &userServer{
        users: make(map[string]*pb.User),
    })
    
    log.Println("Server listening on :50051")
    if err := grpcServer.Serve(lis); err != nil {
        log.Fatalf("failed to serve: %v", err)
    }
}
```

### Client
```go
package main

import (
    "context"
    "log"
    "time"

    "google.golang.org/grpc"
    "google.golang.org/grpc/credentials/insecure"
    
    pb "github.com/myorg/myapp/gen/api/v1"
)

func main() {
    conn, err := grpc.Dial("localhost:50051",
        grpc.WithTransportCredentials(insecure.NewCredentials()),
    )
    if err != nil {
        log.Fatalf("failed to connect: %v", err)
    }
    defer conn.Close()
    
    client := pb.NewUserServiceClient(conn)
    
    ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
    defer cancel()
    
    // Create user
    createResp, err := client.CreateUser(ctx, &pb.CreateUserRequest{
        Email: "user@example.com",
        Name:  "John Doe",
    })
    if err != nil {
        log.Fatalf("CreateUser failed: %v", err)
    }
    log.Printf("Created user: %v", createResp.User)
    
    // Get user
    getResp, err := client.GetUser(ctx, &pb.GetUserRequest{
        Id: createResp.User.Id,
    })
    if err != nil {
        log.Fatalf("GetUser failed: %v", err)
    }
    log.Printf("Got user: %v", getResp.User)
}
```

## Streaming

### Server Streaming
```go
// Server
func (s *userServer) ListUsers(req *pb.ListUsersRequest, stream pb.UserService_ListUsersServer) error {
    for _, user := range s.users {
        if err := stream.Send(user); err != nil {
            return err
        }
    }
    return nil
}

// Client
stream, err := client.ListUsers(ctx, &pb.ListUsersRequest{})
if err != nil {
    return err
}

for {
    user, err := stream.Recv()
    if err == io.EOF {
        break
    }
    if err != nil {
        return err
    }
    log.Printf("Received user: %v", user)
}
```

### Client Streaming
```go
// Server
func (s *userServer) BatchCreateUsers(stream pb.UserService_BatchCreateUsersServer) error {
    var created int32
    
    for {
        req, err := stream.Recv()
        if err == io.EOF {
            return stream.SendAndClose(&pb.BatchCreateUsersResponse{
                CreatedCount: created,
            })
        }
        if err != nil {
            return err
        }
        
        // Create user
        s.createUser(req)
        created++
    }
}

// Client
stream, err := client.BatchCreateUsers(ctx)
if err != nil {
    return err
}

for _, user := range users {
    if err := stream.Send(&pb.CreateUserRequest{
        Email: user.Email,
        Name:  user.Name,
    }); err != nil {
        return err
    }
}

resp, err := stream.CloseAndRecv()
if err != nil {
    return err
}
log.Printf("Created %d users", resp.CreatedCount)
```

### Bidirectional Streaming
```go
// Server
func (s *userServer) SyncUsers(stream pb.UserService_SyncUsersServer) error {
    for {
        req, err := stream.Recv()
        if err == io.EOF {
            return nil
        }
        if err != nil {
            return err
        }
        
        // Process and respond
        resp := processSyncRequest(req)
        if err := stream.Send(resp); err != nil {
            return err
        }
    }
}
```

## Error Handling

### Status Codes
| Code | When to Use |
|------|-------------|
| `OK` | Success |
| `INVALID_ARGUMENT` | Client provided invalid input |
| `NOT_FOUND` | Resource doesn't exist |
| `ALREADY_EXISTS` | Resource already exists |
| `PERMISSION_DENIED` | No permission for operation |
| `UNAUTHENTICATED` | Missing/invalid credentials |
| `RESOURCE_EXHAUSTED` | Rate limit, quota exceeded |
| `FAILED_PRECONDITION` | Operation rejected (state issue) |
| `UNAVAILABLE` | Service temporarily unavailable |
| `INTERNAL` | Internal server error |
| `UNIMPLEMENTED` | Method not implemented |

### Error Details
```go
import (
    "google.golang.org/genproto/googleapis/rpc/errdetails"
    "google.golang.org/grpc/status"
)

func validateUser(req *pb.CreateUserRequest) error {
    var violations []*errdetails.BadRequest_FieldViolation
    
    if req.Email == "" {
        violations = append(violations, &errdetails.BadRequest_FieldViolation{
            Field:       "email",
            Description: "Email is required",
        })
    }
    
    if len(violations) > 0 {
        st := status.New(codes.InvalidArgument, "validation failed")
        st, _ = st.WithDetails(&errdetails.BadRequest{
            FieldViolations: violations,
        })
        return st.Err()
    }
    
    return nil
}

// Client handling
if err != nil {
    st := status.Convert(err)
    for _, detail := range st.Details() {
        if badReq, ok := detail.(*errdetails.BadRequest); ok {
            for _, violation := range badReq.FieldViolations {
                log.Printf("Field %s: %s", violation.Field, violation.Description)
            }
        }
    }
}
```

## Interceptors (Middleware)

### Unary Server Interceptor
```go
func loggingInterceptor(
    ctx context.Context,
    req interface{},
    info *grpc.UnaryServerInfo,
    handler grpc.UnaryHandler,
) (interface{}, error) {
    start := time.Now()
    
    resp, err := handler(ctx, req)
    
    log.Printf("method=%s duration=%s error=%v",
        info.FullMethod,
        time.Since(start),
        err,
    )
    
    return resp, err
}

// Usage
grpcServer := grpc.NewServer(
    grpc.UnaryInterceptor(loggingInterceptor),
)
```

### Chain Multiple Interceptors
```go
import "github.com/grpc-ecosystem/go-grpc-middleware"

grpcServer := grpc.NewServer(
    grpc.UnaryInterceptor(grpc_middleware.ChainUnaryServer(
        loggingInterceptor,
        authInterceptor,
        recoveryInterceptor,
    )),
)
```

### Stream Interceptor
```go
func streamLoggingInterceptor(
    srv interface{},
    ss grpc.ServerStream,
    info *grpc.StreamServerInfo,
    handler grpc.StreamHandler,
) error {
    log.Printf("Stream started: %s", info.FullMethod)
    err := handler(srv, ss)
    log.Printf("Stream ended: %s error=%v", info.FullMethod, err)
    return err
}

grpcServer := grpc.NewServer(
    grpc.StreamInterceptor(streamLoggingInterceptor),
)
```

## Metadata (Headers)

### Server Reading Metadata
```go
import "google.golang.org/grpc/metadata"

func (s *userServer) GetUser(ctx context.Context, req *pb.GetUserRequest) (*pb.GetUserResponse, error) {
    md, ok := metadata.FromIncomingContext(ctx)
    if ok {
        if values := md.Get("authorization"); len(values) > 0 {
            token := values[0]
            // Validate token
        }
    }
    // ...
}
```

### Client Sending Metadata
```go
ctx := metadata.AppendToOutgoingContext(ctx,
    "authorization", "Bearer "+token,
    "x-request-id", requestID,
)

resp, err := client.GetUser(ctx, req)
```

## Performance

### Connection Pooling
```go
// Client-side: reuse connections
conn, _ := grpc.Dial(address,
    grpc.WithDefaultServiceConfig(`{"loadBalancingPolicy":"round_robin"}`),
)
// Reuse conn for multiple clients/requests
```

### Keepalive
```go
// Server
grpcServer := grpc.NewServer(
    grpc.KeepaliveParams(keepalive.ServerParameters{
        MaxConnectionIdle:     15 * time.Second,
        MaxConnectionAge:      30 * time.Second,
        MaxConnectionAgeGrace: 5 * time.Second,
        Time:                  5 * time.Second,
        Timeout:               1 * time.Second,
    }),
)

// Client
conn, _ := grpc.Dial(address,
    grpc.WithKeepaliveParams(keepalive.ClientParameters{
        Time:                10 * time.Second,
        Timeout:             3 * time.Second,
        PermitWithoutStream: true,
    }),
)
```

### Compression
```go
import "google.golang.org/grpc/encoding/gzip"

// Client
resp, err := client.GetUser(ctx, req, grpc.UseCompressor(gzip.Name))

// Server (automatic decompression)
// No special config needed
```

## Code Generation

### buf.yaml
```yaml
version: v1
name: buf.build/myorg/myapp
deps:
  - buf.build/googleapis/googleapis
breaking:
  use:
    - FILE
lint:
  use:
    - DEFAULT
```

### buf.gen.yaml
```yaml
version: v1
plugins:
  - plugin: buf.build/protocolbuffers/go
    out: gen
    opt: paths=source_relative
  - plugin: buf.build/grpc/go
    out: gen
    opt: paths=source_relative
```

```bash
# Generate code
buf generate

# Lint protos
buf lint

# Check breaking changes
buf breaking --against '.git#branch=main'
```

## References

- [gRPC Documentation](https://grpc.io/docs/)
- [Protocol Buffers Style Guide](https://protobuf.dev/programming-guides/style/)
- [Google API Design Guide](https://cloud.google.com/apis/design)
- [Buf Documentation](https://buf.build/docs/)
