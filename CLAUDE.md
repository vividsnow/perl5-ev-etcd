# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

EV-Etcd is a minimal, high-performance Perl XS module implementing an async etcd v3 client using native gRPC Core C API and EV/libev.

## Build Commands

```bash
perl Makefile.PL
make
make test
prove -lv t/specific_test.t   # single test
make clean
```

## Architecture

### Stack

```
┌─────────────────────────┐
│   EV::Etcd (Perl API)   │
├─────────────────────────┤
│      Etcd.xs (XS)       │
├─────────────────────────┤
│  gRPC Core + protobuf-c │
├─────────────────────────┤
│   libev (event loop)    │
└─────────────────────────┘
```

### Key Components

- **gRPC Core C API**: Native gRPC library for protobuf/HTTP2 communication
- **protobuf-c**: etcd v3 API protobuf definitions compiled to C structs
- **libev integration**: Dedicated thread polls gRPC completion queue, signals main thread via ev_async

### Core Operations

- KV: `get`, `put`, `delete`, `range`, `txn`
- Watch: `watch` (bidirectional streaming)
- Lease: `grant`, `revoke`, `keepalive`

### Async Pattern

```c
// gRPC async call -> completion queue -> pthread -> ev_async -> perl callback
```

A dedicated pthread polls the gRPC completion queue. When operations complete, events are queued and the main thread is signaled via ev_async. The main thread then processes events and invokes Perl callbacks.

### File Structure

```
lib/EV/Etcd.pm      # Perl interface with callback API
Etcd.xs             # XS bindings wrapping gRPC Core calls
etcd_*.{c,h}        # Modular C handlers (kv, watch, lease, auth, etc.)
proto/*.proto       # etcd v3 API protobufs (rpc.proto, kv.proto, lock.proto)
*.pb-c.{c,h}        # generated protobuf-c code
typemap             # Perl/C type mappings
```

## Dependencies

- EV (Perl)
- gRPC Core (libgrpc - C gRPC library)
- protobuf-c (C protobuf compiler and runtime)
- libev

## Protobuf Generation

```bash
protoc --c_out=. proto/rpc.proto proto/kv.proto proto/lock.proto proto/election.proto proto/cluster.proto
```

## Code Patterns

### Helper Macros (etcd_common.h)

The codebase provides several helper macros to reduce boilerplate:

- `SERIALIZE_PROTOBUF_TO_SLICE` - Zero-copy protobuf serialization (used throughout)
- `CALL_ERROR_CALLBACK` - Standard error callback pattern
- `CALL_SIMPLE_ERROR_CALLBACK` - Simple string error callback
- `CALL_SUCCESS_CALLBACK` - Success callback pattern
- `PROCESS_HEADER_ONLY_RESPONSE` - Macro for responses that only have a header
- `BEGIN_RESPONSE_HANDLER` / `UNPACK_RESPONSE` - Response handler boilerplate

### Future Optimization Opportunities

1. **TLS support**: Currently only insecure connections are supported. TLS/mTLS would require adding credential options to `new()`.

2. **Completion queue polling**: Fixed 100ms interval could be made adaptive based on load.
