## Call Graphs from Syscalls to Functions in `net/`

This document lists functions in the `net` directory that have a call graph back to a syscall, and documents the function call graph to reach them from a syscall entry point. Only functions that are reachable from a syscall are included.

### Example: socket syscall path


#### socket syscall path
sys_socket (syscall, args from userspace)
  -> __sys_socket (args from syscall)
    -> __sock_create (args from syscall)
      -> security_socket_create
      -> sock_alloc
      -> lookup_net_fam
      -> pf->create
    -> sock_alloc_file
    -> sock_map_fd
  -> sock_release
  -> move_addr_to_kernel (addr from syscall)
  -> move_addr_to_user

---

### Example: sendmsg syscall path


#### sendmsg syscall path
sys_sendmsg (syscall, args from userspace)
  -> __sys_sendmsg (args from syscall)
    -> sockfd_lookup (fd from syscall)
      -> fget
      -> sock_from_file
    -> security_socket_sendmsg
    -> sock_sendmsg (msg from syscall)
      -> ops->sendmsg

---


### General pattern
sys_* (syscall, args from userspace)
  -> __sys_*
    -> helpers (sockfd_lookup, move_addr_to_kernel, sock_alloc, etc.)
      -> protocol-specific ops (sock->ops->*, proto_ops)

---

### Example: bind syscall path


#### bind syscall path
sys_bind (syscall, args from userspace)
  -> __sys_bind (args from syscall)
    -> sockfd_lookup (fd from syscall)
      -> sock_from_file
    -> move_addr_to_kernel (addr from syscall)
    -> security_socket_bind
    -> sock->ops->bind (addr from syscall)

---

### Example: getsockopt syscall path


#### getsockopt syscall path
sys_getsockopt (syscall, args from userspace)
  -> __sys_getsockopt (args from syscall)
    -> sockfd_lookup (fd from syscall)
    -> security_socket_getsockopt
    -> sock_getsockopt (opt args from syscall)
      -> sock->ops->getsockopt (opt args from syscall)

---

### Note

Many functions in `net/socket.c` are directly or indirectly reachable from syscalls, including:

- `sock_alloc`, `sock_release`, `sock_alloc_file`, `sock_map_fd`, `sockfd_lookup`, `sock_from_file`, `move_addr_to_kernel`, `move_addr_to_user`, `sock_sendmsg`, `sock_recvmsg`, etc.

The protocol-specific implementations (e.g., in `net/ipv4/`, `net/ipv6/`, etc.) are called via function pointers in the `proto_ops` structures.