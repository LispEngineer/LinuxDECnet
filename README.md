# LinuxDECnet
DECnet for Linux updated to run on latest Linux kernel (After DECnet code was removed from kernel 6.1)

---

## About this copy

This is a **patched fork** of John Forecast's LinuxDECnet:

* **Upstream / original source:** https://github.com/JohnForecast/LinuxDECnet
* **Forked at:** commit `227ce48`
* All credit for DECnet on modern Linux belongs to John Forecast. This fork
  exists only to carry the module-unload fix described below, and is intended
  to be offered back upstream.

### Fix carried here: `decnet3.ko` could not be safely unloaded

`rmmod decnet3` kernel-panicked a workstation (Arch Linux, kernel
`7.2.6-arch2-1`) on 18-SEP-2026. `dnet_exit()` cleaned up almost nothing that
`dnet_init()` registered, leaving the kernel holding pointers into freed
module memory:

* `dev_add_pack()`, `sock_register()`, `proto_register()` and
  `register_netdevice_notifier()` had **no** matching unregister calls.
* `dn_next_cleanup()`, `dn_node_cleanup()` and `dn_unregister_sysctl()` were
  written but **never called**; `dn_sock_exit()` was an empty function.
* Three timers were left armed. All three re-arm themselves from inside their
  own handlers, so they need `timer_shutdown_sync()` rather than
  `del_timer_sync()` -- the latter can return just after a handler running on
  another CPU has re-armed the timer.
* Two `INIT_WORK()` items, scheduled from those timers, were never cancelled.
* procfs entries created under `init_net.proc_net` were removed with a `NULL`
  parent, which looks in `/proc` rather than `/proc/net` and removes nothing.
* The loopback device reference taken with `dev_hold()` was never released.

In practice the module panicked the machine as soon as anything touched one of
the stale registrations: `ip link delete` walking `netdev_chain` into the
unregistered notifier, a timer firing in softirq context, or simply reading
`/proc/net/ptype`.

Changed files: `kernel/dnet.c`, `kernel/dnet_dev.c`, `kernel/dnet_next.c`,
`kernel/dnet_node.c`, `kernel/dnet_sock.c`.

Verified in a throwaway QEMU/KVM VM booting the host's own kernel image, with
the harness validated against the unpatched module first (which panics in
`ptype_seq_show`) before testing the patched one -- all checks pass, including
three load/unload cycles and the exact sequence that killed the host. Full
write-up, including why a smaller fix would have been worse than none:
`DECNET_MODULE_UNLOAD_CRASH_AND_FIX.md` in the `vms-exploration` project.

One upstream TODO is deliberately left in place: individual entries chained
off `dn_node_db` are still not flushed on unload. That is a bounded memory
leak rather than a crash -- `dn_node_entry` holds no timer, work item or other
kernel registration -- and flushing it properly needs entry lifetime rules the
module does not currently document.

---

README.DECnet

  - Detailed instructions for downloading and installing DECnet on a Linux system.
  
  
BuildAndInstall.sh

  - Shell script which automates most of the process of installing DECnet on a Linux system:
  
  1. Create a working directory on the target system and make that the current directory
  
  2. Copy BuildAndInstall.sh to this directory and make it executable
  
  3. Execute the shell script, answer the questions and wait for a new kernel and DECnet utilities to be built
     (It may take a long time depending on the system configuration)

  4. If BuildAndInstall.sh detects that your installation uses systemd it will create 3 service entries;
     one to change the MAC address of the Ethernet/Wireless LAN interface, the second to load the decnet modules and start it running
     and the third to start the phone daemon running.

  6. If your installation does not use systemd, there are some mechansisms described in the RaspbianDECnet repository about how to get DECnet started (the module name has changed for this respository - "decnet3")
