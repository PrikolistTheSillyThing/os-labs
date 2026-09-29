# Operating Systems Lab Report: Observing the OS at Work
---

## 1. Introduction

An operating system sits between user programs and the hardware, deciding who gets what and when. In lecture this was mostly theory; in this lab, that manager is observed directly.

The lab was carried out in a terminal on the provided Linux virtual machine. It covers the four core jobs of any OS:

- **Files:** how the OS organizes, stores, and protects data
- **Processes:** how running programs are created, scheduled, and tracked
- **Memory:** how the OS allocates and monitors RAM
- **Devices:** how hardware is exposed to programs and users

For each area, the report gives the commands used and, more importantly, **what they showed**: the output, what it says about how the OS behaves, and any surprises along the way.

**Environment:** Ubuntu 26.04, VirtualBox, kernel `7.0.0-34-generic`.
