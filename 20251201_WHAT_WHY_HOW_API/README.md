# WHAT–WHY–HOW–API

This proposal is an extension to the Linux Kernel Requirements Template work. It
aims to introduce more precise and traceable requirements into the Linux kernel.

As a short recap, the current Requirements Template suggests placing SPDX-\*
fields directly in source code comments, while additional requirement metadata
is stored in separate "sidecar" text files. In this model, one requirement is
split across two locations: some fields are inside the function's comment block,
and the rest are in the corresponding sidecar file.

Further discussion and feedback from the community are welcome.

## Proposal

This document proposes the following changes:

1. **Distill and clarify the semantics of `SPDX-Req-Text`**

The single `SPDX-Req-Text` field currently contains different types of
information mixed together. We propose dividing it into several fields. These
fields would capture four different aspects: WHAT (the requirement statement),
WHY (the rationale or justification), HOW (design or implementation
considerations), API (the interface description).

This separation ensures that each aspect is expressed clearly, reduces mixing of
concepts, and supports more consistent documentation across the codebase. It
also introduces a stable discipline and workflow that Linux kernel developers
can rely on when writing documentation and when working with all
WHAT/WHY/HOW/API aspects of information. The meaning of each field is described
below.

2. **Optional: Store all requirement content inside the source comment**

If WHAT/WHY/HOW/API are clearly separated:

- It becomes possible to reduce or remove the need for sidecar files. All
  requirement content can be stored directly inside the function's comment
  block. This simplifies the workflow by removing the need to mirror the sidecar
  file structure, while still preserving clarity and traceability.

- It also becomes possible to place the less textual or less narrative
  information toward the end of the comment block. A developer can quickly
  locate the required aspect by searching for its field name, knowing that the
  more "noisy" meta-information is placed at the bottom of the comment.

## WHAT/WHY/HOW/API fields

### WHAT (Statement)

This field contains the main functional requirement of the function.

It should describe what the function is expected to do, without describing how
it achieves this result. The "how" part belongs to the HOW field.

### WHY (Rationale)

This field contains the reason for the function's existence. A typical rationale
explains how this function satisfies its parent requirement(s) referenced with
`SPDX-Req-Ref`.

The rationale may also explain why the function has its particular scope and
interface.

### HOW (Implementation details)

This field contains internal design and implementation details. It should
include information that is important for developers working on the function.
Less obvious design choices and potential pitfalls can be listed here.

### API (Interface description)

This field describes the function's interface, including its parameters, return
value, and possible side effects. It is similar to kernel-doc or Doxygen-style
documentation, but without any functional behavior description. Functional
behavior belongs to the WHAT field.

## Fat WHAT vs lean API

WHAT and API both describe externally visible aspects, but their roles are
different:

- WHAT describes the required functional behavior and the observable result that
  the function must provide.
- API describes the interface through which callers use the function. This
  includes how the function is called, which arguments it takes, how the return
  value is handled, and any other interface-level details.

In short: WHAT defines the required functional behavior. API defines the usage
surface, that is, the contract between the function and its callers.

## What is tested

Automated tests are written only against the statements found in the WHAT field.

The HOW, WHY, and API fields are NOT tested by automated tests.

This approach follows testing practices in several regulated industries, where
requirements specifications are tested but interface documents (e.g., Interface
Control Documents) are not.

## Proposed order of the field declaration

- `SPDX-Req-Title:`
- `SPDX-Req-API:`
- `SPDX-Req-What:`
- `SPDX-Req-Why:`
- `SPDX-Req-How:`

Meta information fields follow at the bottom of the block:

- `SPDX-Req-Sys: <SUBSYSTEM>`
- `SPDX-Req-ID: <UUID>`
- `SPDX-Req-HKey: <KEY>`
- `SPDX-Req-Ref: <UUID>` (can be multiple)

The `SPDX-Req-End` marker terminates the declaration. It can be omitted if there
is no additional content in the source comment.

## Why not conventional requirement field titles?

The field WHAT is effectively a statement, WHY is a rationale, and HOW is
implementation or design. The API field could also be named interface. A natural
question is why these fields are not named directly in this way.

The answer in this proposal is that the terms WHAT, WHY, HOW, and API are closer
to the mental model of a developer who is writing code or reviewing
requirements. These words match how developers usually think about the
information they need during design, review, and implementation work.

If required by tools that process requirements, these field names can be easily
mapped to formal categories such as statements or rationales. However, using
these formal names directly is seen as too redundant and too heavy for the
everyday workflow of software developers.

## Examples

### Example 1

```c
/**
 * SPDX-Req-TITLE: read_mem - read from physical memory (/dev/mem).
 *
 * SPDX-Req-API:
 *
 * This function checks if the requested physical memory range is valid
 * and accessible by the user, then it copies data to the input
 * user-space buffer up to the requested number of bytes.
 *
 * @file: struct file associated with /dev/mem.
 * @buf: user-space buffer to copy data to.
 * @count: number of bytes to read.
 * @ppos: pointer to the current file position, representing the physical
 *        address to read from.
 *
 * Context: Process context.
 *
 * Return:
 * * the number of bytes copied to user on success
 * * %-EFAULT - the requested address range is not valid or a fault happened
 *   when copying to user-space (i.e. copy_from_kernel_nofault() failed)
 * * %-EPERM - access to any of the required physical pages is not allowed
 * * %-ENOMEM - out of memory error for auxiliary kernel buffers supporting
 *   the operation of copying content from the physical pages
 *
 * SPDX-Req-WHAT:
 *
 * Function's expectations:
 *
 * 1. This function shall check if the value pointed by ppos exceeds the
 *    maximum addressable physical address;
 *
 * 2. This function shall check if the physical address range to be read
 *    is valid (i.e. it falls within a memory block and if it can be mapped
 *    to the kernel address space);
 *
 * 3. For each memory page falling in the requested physical range
 *    [ppos, ppos + count - 1]:
 *   3.1. this function shall check if user space access is allowed (if
 *        config STRICT_DEVMEM is not set, access is always granted);
 *
 *   3.2. if access is allowed, the memory content from the page range falling
 *        within the requested physical range shall be copied to the user space
 *        buffer;
 *
 *   3.3. zeros shall be copied to the user space buffer (for the page range
 *        falling within the requested physical range):
 *     3.3.1. if access to the memory page is restricted or,
 *     3.2.2. if the current page is page 0 on HW architectures where page 0 is
 *            not mapped.
 *
 * 4. The file position '*ppos' shall be advanced by the number of bytes
 *    successfully copied to user space (including zeros).
 *
 * SPDX-Req-WHY: <Not described by this function>
 *
 * SPDX-Req-HOW: <Not described by this function>
 *
 * SPDX-Req-ID: a89784c55426aec4b8ba345f281a0ec478d43897a0a248618cb140c03c770c75
 * SPDX-Req-HKEY: 6e16917c09ee583de5dc9e8a24a406e75bb229554699a501cfa8efdb308862d7
 */
static ssize_t read_mem(struct file *file, char __user *buf,
			size_t count, loff_t *ppos)
{
}
```

### Example 2

The version below is closer to StrictDoc's naming conventions visually:

- `Req` -> `REQ`
- `HKey` -> `HASH`

```c
/**
 * SPDX-REQ-TITLE: read_mem - read from physical memory (/dev/mem).
 *
 * SPDX-REQ-API:
 *
 * This function checks if the requested physical memory range is valid
 * and accessible by the user, then it copies data to the input
 * user-space buffer up to the requested number of bytes.
 *
 * @file: struct file associated with /dev/mem.
 * @buf: user-space buffer to copy data to.
 * @count: number of bytes to read.
 * @ppos: pointer to the current file position, representing the physical
 *        address to read from.
 *
 * Context: Process context.
 *
 * Return:
 * * the number of bytes copied to user on success
 * * %-EFAULT - the requested address range is not valid or a fault happened
 *   when copying to user-space (i.e. copy_from_kernel_nofault() failed)
 * * %-EPERM - access to any of the required physical pages is not allowed
 * * %-ENOMEM - out of memory error for auxiliary kernel buffers supporting
 *   the operation of copying content from the physical pages
 *
 * SPDX-REQ-WHAT:
 *
 * Function's expectations:
 *
 * 1. This function shall check if the value pointed by ppos exceeds the
 *    maximum addressable physical address;
 *
 * 2. This function shall check if the physical address range to be read
 *    is valid (i.e. it falls within a memory block and if it can be mapped
 *    to the kernel address space);
 *
 * 3. For each memory page falling in the requested physical range
 *    [ppos, ppos + count - 1]:
 *   3.1. this function shall check if user space access is allowed (if
 *        config STRICT_DEVMEM is not set, access is always granted);
 *
 *   3.2. if access is allowed, the memory content from the page range falling
 *        within the requested physical range shall be copied to the user space
 *        buffer;
 *
 *   3.3. zeros shall be copied to the user space buffer (for the page range
 *        falling within the requested physical range):
 *     3.3.1. if access to the memory page is restricted or,
 *     3.2.2. if the current page is page 0 on HW architectures where page 0 is
 *            not mapped.
 *
 * 4. The file position '*ppos' shall be advanced by the number of bytes
 *    successfully copied to user space (including zeros).
 *
 * SPDX-REQ-WHY: <Not described by this function>
 *
 * SPDX-REQ-HOW: <Not described by this function>
 *
 * SPDX-REQ-ID: a89784c55426aec4b8ba345f281a0ec478d43897a0a248618cb140c03c770c75
 * SPDX-REQ-HASH: 6e16917c09ee583de5dc9e8a24a406e75bb229554699a501cfa8efdb308862d7
 */
static ssize_t read_mem(struct file *file, char __user *buf,
			size_t count, loff_t *ppos)
{
}
```

### Example 3

The version below removes SPDX-REQ- entirely. The interesting driver for the
removal is a very close analogy with the `FIXME`, `TODO`, `TBC` keywords known
to many software developers.

```c
/**
 * TITLE: read_mem - read from physical memory (/dev/mem).
 *
 * API:
 *
 * This function checks if the requested physical memory range is valid
 * and accessible by the user, then it copies data to the input
 * user-space buffer up to the requested number of bytes.
 *
 * @file: struct file associated with /dev/mem.
 * @buf: user-space buffer to copy data to.
 * @count: number of bytes to read.
 * @ppos: pointer to the current file position, representing the physical
 *        address to read from.
 *
 * Context: Process context.
 *
 * Return:
 * * the number of bytes copied to user on success
 * * %-EFAULT - the requested address range is not valid or a fault happened
 *   when copying to user-space (i.e. copy_from_kernel_nofault() failed)
 * * %-EPERM - access to any of the required physical pages is not allowed
 * * %-ENOMEM - out of memory error for auxiliary kernel buffers supporting
 *   the operation of copying content from the physical pages
 *
 * WHAT:
 *
 * Function's expectations:
 *
 * 1. This function shall check if the value pointed by ppos exceeds the
 *    maximum addressable physical address;
 *
 * 2. This function shall check if the physical address range to be read
 *    is valid (i.e. it falls within a memory block and if it can be mapped
 *    to the kernel address space);
 *
 * 3. For each memory page falling in the requested physical range
 *    [ppos, ppos + count - 1]:
 *   3.1. this function shall check if user space access is allowed (if
 *        config STRICT_DEVMEM is not set, access is always granted);
 *
 *   3.2. if access is allowed, the memory content from the page range falling
 *        within the requested physical range shall be copied to the user space
 *        buffer;
 *
 *   3.3. zeros shall be copied to the user space buffer (for the page range
 *        falling within the requested physical range):
 *     3.3.1. if access to the memory page is restricted or,
 *     3.2.2. if the current page is page 0 on HW architectures where page 0 is
 *            not mapped.
 *
 * 4. The file position '*ppos' shall be advanced by the number of bytes
 *    successfully copied to user space (including zeros).
 *
 * WHY: <Not described by this function>
 *
 * HOW: <Not described by this function>
 *
 * UUID: a89784c55426aec4b8ba345f281a0ec478d43897a0a248618cb140c03c770c75
 * HASH: 6e16917c09ee583de5dc9e8a24a406e75bb229554699a501cfa8efdb308862d7
 */
static ssize_t read_mem(struct file *file, char __user *buf,
			size_t count, loff_t *ppos)
{
}
```
