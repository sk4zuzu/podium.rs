---
authors: Michał Opala (sk4zuzu@gmail.com)
state: draft
---

# RFD 0 - Non-HA Remote Process Executor

## What

Design and implementation of a secure client/server protocol for remote linux process execution.

## Why

To research and understand what are the challenges in designing your own "docker-like" software (in Rust).

## Details

### Efficient client / server architecture (GRPC)

For the `GRPC / Protobuf` handling [tonic](https://github.com/hyperium/tonic/blob/master/examples/helloworld-tutorial.md) library can be used. It is `tokio`-based (fully async).

Basic ".proto" file for client and server may look like this:

```protobuf
syntax = "proto3";
package podium;

service Starter {
  rpc Start (StartRequest) returns (StartResponse);
}

message StartRequest {
  string cmd = 1;
  float cpu = 2;
  int64 mem = 3;
  int64 io = 4;
}

message StartSuccess {
  int64 id = 1;
}

message StartFailure {
  int64 id = 1;
  string msg = 2;
}

message StartResponse {
  oneof response {
    StartSuccess success = 1;
    StartFailure failure = 2;
  }
}

service Stopper {
  rpc Stop (StopRequest) returns (StopResponse);
}

message StopRequest {
  int64 id = 1;
}

message StopSuccess {
  int64 id = 1;
}

message StopFailure {
  int64 id = 1;
  string msg = 2;
}

message StopResponse {
  oneof response {
    StopSuccess success = 1;
    StopFailure failure = 2;
  }
}

service Status {
  rpc Status (StatusRequest) returns (StatusResponse);
}

message StatusRequest {
  int64 id = 1;
}

enum State {
  RUNNING = 0;
  COMPLETED = 1;
  FAILED = 2;
}

message StatusSuccess {
  int64 id = 1;
  State state = 2;
}

message StatusFailure {
  int64 id = 1;
  string msg = 2;
}

message StatusResponse {
  oneof response {
    StatusSuccess success = 1;
    StatusFailure failure = 2;
  }
}

service Logger {
  rpc Log (LogRequest) returns (stream LogResponse);
}

message LogRequest {
  int64 id = 1;
}

message LogSuccess {
  int64 id = 1;
  string stdout = 2;
  string stderr = 3;
}

message LogFailure {
  int64 id = 1;
  string msg = 2;
}

message LogResponse {
  oneof response {
    LogSuccess success = 1;
    LogFailure failure = 2;
  }
}
```

### Authorization and user management based on mutual TLS certificates

An example of mTLS authorization in [tonic](https://github.com/hyperium/tonic) is documented [here](https://github.com/hyperium/tonic/tree/master/examples/src/tls_client_auth).

User management part can be achieved similarly to Kubernetes client certificates by just using the `CommonName` as a username.

### Ability to connect to a job and stream (tail) its output logs

For simplicity a job can be represented as a folder (similarily to `/proc` filesystem) `jobs/<id>/`.

Inside such folder, there are 4 files: 1 file containing pid of the running process, 1 file containing status of the job, 2 files containing stdout and stderr outputs (logs).

```shell
$ find jobs/
jobs/
jobs/1
jobs/1/pid
jobs/1/state
jobs/1/stdout
jobs/1/stderr
jobs/2
jobs/2/pid
jobs/2/state
jobs/2/stdout
jobs/2/stderr
```

To stream the logs the `Logger` service is used:

```protobuf
service Logger {
  rpc Log (LogRequest) returns (stream LogResponse);
}
```

Logs could be read (and "tailed") from stdout and stderr files just like [tailf](https://github.com/rudimeier/util-linux/blob/62e905f09269b9163b19cd2b0712b139ec2709c3/text-utils/tailf.c) did.

### Cgroups-based resource limiting

For the `cgroupsv2` management the [cgroups-fs](https://github.com/kata-containers/cgroups-rs) project can be used.

### Kernel-namespace-based process isolation

A special "wrapper" can be introduced (executed by `tokio` as a new process).

The "wrapper" process uses [nix::sched::clone](https://docs.rs/nix/0.20.0/nix/sched/fn.clone.html) to create the user command in new `Pid`, `Mount` and `Net` namespaces.

To make `Net` namespace fully operational, a `bridge`, a `veth` interface, a `lo` interface and a `default route` all need to be configured.

Here is an example of executing a process in the `Pid` and `Mount` namespaces together:

```rust
use anyhow::Result;

use nix::mount::{mount, MsFlags};
use nix::sched::{clone, CloneFlags};
use nix::sys::wait::waitpid;
use nix::unistd::{chroot, execvp};
use std::env::set_current_dir;
use std::ffi::CString;

fn ps_ax() -> Result<()> {
    mount(
        Some("none"),
        "/",
        None::<&str>,
        MsFlags::MS_REC | MsFlags::MS_PRIVATE,
        None::<&str>,
    )?;

    mount(
        Some("/"),
        "/tmp/ps_ax",
        None::<&str>,
        MsFlags::MS_BIND,
        None::<&str>,
    )?;

    chroot("/tmp/ps_ax")?;
    set_current_dir("/")?;

    mount(
        Some("proc"),
        "/proc",
        Some("proc"),
        MsFlags::empty(),
        None::<&str>,
    )?;

    let ps = CString::new("/nix/store/vr0hjrb6g1vi5fc8iiyzh3jla5qnws7y-procps-3.3.16/bin/ps")?;
    execvp(
        &ps,
        &vec![ps.clone(), CString::new("ax")?],
    )?;

    Ok(())
}

fn main() -> Result<()> {
    let f = Box::new(|| {
        if let Ok(_) = ps_ax() {0} else {-1}
    });

    let mut stack = [0u8; 4096];
    clone(f, &mut stack, CloneFlags::CLONE_NEWNS | CloneFlags::CLONE_NEWPID, Some(17))?;

    let status = waitpid(None, None)?;
    println!("{:?}", status);

    Ok(())
}
```

Output:

```
  PID TTY      STAT   TIME COMMAND
    1 ?        R+     0:00 /nix/store/vr0hjrb6g1vi5fc8iiyzh3jla5qnws7y-procps-3.3.16/bin/ps ax
Exited(Pid(6339), 0)
```
