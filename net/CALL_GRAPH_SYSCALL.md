## Call Graphs from Syscalls to Functions in `net/`

This document lists functions in the `net` directory that have a call graph back to a syscall, and documents the function call graph to reach them from a syscall entry point. Only functions that are reachable from a syscall are included.

### Example: socket syscall path

#### 1. `sys_socket` (syscall entry, defined via SYSCALL_DEFINE3(socket, ...))
  - in `net/socket.c`
  - Calls: `__sys_socket` (in same file)

#### 2. `__sys_socket`
  - in `net/socket.c`
  - Calls: `__sock_create`

#### 3. `__sock_create`
  - in `net/socket.c`
  - Calls: `security_socket_create`, `sock_alloc`, `lookup_net_fam`, `pf->create`, etc.

#### 4. `sock_alloc`
  - in `net/socket.c`
  - Allocates a new socket struct and inode

#### 5. `sock_alloc_file`
  - in `net/socket.c`
  - Called from `__sys_socket_file`, `sock_map_fd`, etc.

#### 6. `sock_map_fd`
  - in `net/socket.c`
  - Called from `__sys_socket` and others

#### 7. `sock_release`
  - in `net/socket.c`
  - Called from many paths, including syscalls

#### 8. `move_addr_to_kernel`, `move_addr_to_user`
  - in `net/socket.c`
  - Used in various socket syscalls (e.g., `sys_bind`, `sys_connect`, etc.)

---

### Example: sendmsg syscall path

#### 1. `sys_sendmsg` (syscall entry, defined via SYSCALL_DEFINE3(sendmsg, ...))
  - in `net/socket.c`
  - Calls: `__sys_sendmsg`

#### 2. `__sys_sendmsg`
  - in `net/socket.c`
  - Calls: `sockfd_lookup`, `security_socket_sendmsg`, `sock_sendmsg`, etc.

#### 3. `sockfd_lookup`
  - in `net/socket.c`
  - Calls: `fget`, `sock_from_file`

#### 4. `sock_from_file`
  - in `net/socket.c`
  - Returns socket from file struct

#### 5. `sock_sendmsg`
  - in `net/socket.c`
  - Calls: `ops->sendmsg` (protocol specific)

---

### General pattern

Most socket-related syscalls in the kernel (e.g., socket, bind, connect, sendmsg, recvmsg, getsockopt, setsockopt, shutdown, etc.) have the following call graph pattern:

1. `sys_*` (SYSCALL_DEFINE* in `net/socket.c`)
2. `__sys_*` (internal implementation)
3. Helper functions (e.g., `sockfd_lookup`, `move_addr_to_kernel`, `sock_alloc`, etc.)
4. Protocol-specific operations via `sock->ops->*` or `proto_ops` tables

---

### Example: bind syscall path

#### 1. `sys_bind` (SYSCALL_DEFINE3(bind, ...))
  - in `net/socket.c`
  - Calls: `__sys_bind`

#### 2. `__sys_bind`
  - in `net/socket.c`
  - Calls: `sockfd_lookup`, `move_addr_to_kernel`, `security_socket_bind`, `sock->ops->bind`

#### 3. `move_addr_to_kernel`
  - in `net/socket.c`
  - Copies sockaddr from user

#### 4. `sockfd_lookup`/`sock_from_file`
  - in `net/socket.c`
  - Gets socket struct from fd

#### 5. `sock->ops->bind`
  - protocol-specific bind implementation

---

### Example: getsockopt syscall path

#### 1. `sys_getsockopt` (SYSCALL_DEFINE5(getsockopt, ...))
  - in `net/socket.c`
  - Calls: `__sys_getsockopt`

#### 2. `__sys_getsockopt`
  - in `net/socket.c`
  - Calls: `sockfd_lookup`, `security_socket_getsockopt`, `sock_getsockopt`

#### 3. `sock_getsockopt`
  - in `net/socket.c`
  - Calls: `sock->ops->getsockopt`

#### 4. `sock->ops->getsockopt`
  - protocol-specific getsockopt implementation

---

### Note

Many functions in `net/socket.c` are directly or indirectly reachable from syscalls, including:

- `sock_alloc`, `sock_release`, `sock_alloc_file`, `sock_map_fd`, `sockfd_lookup`, `sock_from_file`, `move_addr_to_kernel`, `move_addr_to_user`, `sock_sendmsg`, `sock_recvmsg`, etc.

The protocol-specific implementations (e.g., in `net/ipv4/`, `net/ipv6/`, etc.) are called via function pointers in the `proto_ops` structures.