## Call Graphs from Syscalls to Functions in `net/`

This document lists functions in the `net` directory that have a call graph back to a syscall, and documents the function call graph to reach them from a syscall entry point. Only functions that are reachable from a syscall are included.

### Example: socket syscall path

#### 1. `sys_socket` (syscall entry, defined via SYSCALL_DEFINE3(socket, ...))
  - in `net/socket.c`
  - Calls: `__sys_socket` (in same file)

**Summary:**
`sys_socket` is the syscall entry point for creating a new socket. It receives arguments directly from userspace (syscall arguments: domain, type, protocol), which are passed to `__sys_socket`.

#### 2. `__sys_socket`
  - in `net/socket.c`
  - Calls: `__sock_create`

**Summary:**
`__sys_socket` implements the core logic of the socket syscall. It takes the syscall arguments (domain, type, protocol) and calls `__sock_create` to create the socket. The arguments originate from the syscall.

#### 3. `__sock_create`
  - in `net/socket.c`
  - Calls: `security_socket_create`, `sock_alloc`, `lookup_net_fam`, `pf->create`, etc.

**Summary:**
`__sock_create` creates a socket structure, performs security checks, allocates resources, and calls protocol family-specific creation routines. The domain, type, and protocol arguments originate from the syscall.

#### 4. `sock_alloc`
  - in `net/socket.c`
  - Allocates a new socket struct and inode

**Summary:**
`sock_alloc` allocates and initializes a new socket structure and associated inode. It is called as part of socket creation. Arguments do not directly originate from the syscall, but are derived from earlier steps.

#### 5. `sock_alloc_file`
  - in `net/socket.c`
  - Called from `__sys_socket_file`, `sock_map_fd`, etc.

**Summary:**
`sock_alloc_file` allocates a file structure for the socket, enabling it to be associated with a file descriptor. Arguments are derived from the socket structure, not directly from the syscall.

#### 6. `sock_map_fd`
  - in `net/socket.c`
  - Called from `__sys_socket` and others

**Summary:**
`sock_map_fd` maps a socket to a file descriptor. It is used to return the socket to userspace via a file descriptor. The returned fd is ultimately provided to the user as a syscall result.

#### 7. `sock_release`
  - in `net/socket.c`
  - Called from many paths, including syscalls

**Summary:**
`sock_release` releases the resources associated with a socket. It is called when a socket is closed or released, either directly or indirectly from a syscall. Arguments are derived from kernel socket structures.

#### 8. `move_addr_to_kernel`, `move_addr_to_user`
  - in `net/socket.c`
  - Used in various socket syscalls (e.g., `sys_bind`, `sys_connect`, etc.)

**Summary:**
`move_addr_to_kernel` copies a socket address from userspace to kernel space. The address argument originates directly from the syscall. `move_addr_to_user` copies a socket address from kernel space to userspace, typically for returning results to the user.

---

### Example: sendmsg syscall path

#### 1. `sys_sendmsg` (syscall entry, defined via SYSCALL_DEFINE3(sendmsg, ...))
  - in `net/socket.c`
  - Calls: `__sys_sendmsg`

**Summary:**
`sys_sendmsg` is the syscall entry point for sending a message on a socket. It receives arguments directly from userspace (sockfd, msg, flags), which are passed to `__sys_sendmsg`.

#### 2. `__sys_sendmsg`
  - in `net/socket.c`
  - Calls: `sockfd_lookup`, `security_socket_sendmsg`, `sock_sendmsg`, etc.

**Summary:**
`__sys_sendmsg` implements the core logic of the sendmsg syscall. It takes the syscall arguments (sockfd, msg, flags), looks up the socket, performs security checks, and calls `sock_sendmsg`. The arguments originate from the syscall.

#### 3. `sockfd_lookup`
  - in `net/socket.c`
  - Calls: `fget`, `sock_from_file`

**Summary:**
`sockfd_lookup` looks up a socket structure given a file descriptor. The file descriptor argument originates from the syscall.

#### 4. `sock_from_file`
  - in `net/socket.c`
  - Returns socket from file struct

**Summary:**
`sock_from_file` extracts the socket structure from a file structure. The file structure is obtained from previous steps (e.g., `fget`), not directly from the syscall.

#### 5. `sock_sendmsg`
  - in `net/socket.c`
  - Calls: `ops->sendmsg` (protocol specific)

**Summary:**
`sock_sendmsg` sends a message on a socket by calling the protocol-specific sendmsg operation. The message and socket arguments are derived from previous steps, with the message originating from the syscall.

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

**Summary:**
`sys_bind` is the syscall entry point for binding a socket to an address. It receives arguments directly from userspace (sockfd, addr, addrlen), which are passed to `__sys_bind`.

#### 2. `__sys_bind`
  - in `net/socket.c`
  - Calls: `sockfd_lookup`, `move_addr_to_kernel`, `security_socket_bind`, `sock->ops->bind`

**Summary:**
`__sys_bind` implements the core logic of the bind syscall. It takes the syscall arguments (sockfd, addr, addrlen), looks up the socket, copies the address from userspace, performs security checks, and calls the protocol-specific bind operation. The arguments originate from the syscall.

#### 3. `move_addr_to_kernel`
  - in `net/socket.c`
  - Copies sockaddr from user

**Summary:**
`move_addr_to_kernel` copies a socket address from userspace to kernel space. The address argument originates directly from the syscall.

#### 4. `sockfd_lookup`/`sock_from_file`
  - in `net/socket.c`
  - Gets socket struct from fd

**Summary:**
`sockfd_lookup` looks up a socket structure given a file descriptor. The file descriptor argument originates from the syscall. `sock_from_file` extracts the socket structure from a file structure, which is derived from previous steps.

#### 5. `sock->ops->bind`
  - protocol-specific bind implementation

**Summary:**
`sock->ops->bind` is the protocol-specific implementation of the bind operation. It binds the socket to an address. The socket and address arguments are derived from previous steps, with the address originating from the syscall.

---

### Example: getsockopt syscall path

#### 1. `sys_getsockopt` (SYSCALL_DEFINE5(getsockopt, ...))
  - in `net/socket.c`
  - Calls: `__sys_getsockopt`

**Summary:**
`sys_getsockopt` is the syscall entry point for getting socket options. It receives arguments directly from userspace (sockfd, level, optname, optval, optlen), which are passed to `__sys_getsockopt`.

#### 2. `__sys_getsockopt`
  - in `net/socket.c`
  - Calls: `sockfd_lookup`, `security_socket_getsockopt`, `sock_getsockopt`

**Summary:**
`__sys_getsockopt` implements the core logic of the getsockopt syscall. It takes the syscall arguments (sockfd, level, optname, optval, optlen), looks up the socket, performs security checks, and calls `sock_getsockopt`. The arguments originate from the syscall.

#### 3. `sock_getsockopt`
  - in `net/socket.c`
  - Calls: `sock->ops->getsockopt`

**Summary:**
`sock_getsockopt` calls the protocol-specific getsockopt operation to retrieve socket options. The socket and option arguments are derived from previous steps, with the option arguments originating from the syscall.

#### 4. `sock->ops->getsockopt`
  - protocol-specific getsockopt implementation

**Summary:**
`sock->ops->getsockopt` is the protocol-specific implementation of the getsockopt operation. It retrieves socket options. The socket and option arguments are derived from previous steps, with the option arguments originating from the syscall.

---

### Note

Many functions in `net/socket.c` are directly or indirectly reachable from syscalls, including:

- `sock_alloc`, `sock_release`, `sock_alloc_file`, `sock_map_fd`, `sockfd_lookup`, `sock_from_file`, `move_addr_to_kernel`, `move_addr_to_user`, `sock_sendmsg`, `sock_recvmsg`, etc.

The protocol-specific implementations (e.g., in `net/ipv4/`, `net/ipv6/`, etc.) are called via function pointers in the `proto_ops` structures.