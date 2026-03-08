---
name: kernel-glossary
description: >
  Generate structured Linux kernel documentation pages for this knowledge base.
user-invocable: true
---

# kernel-glossary

Generate a Linux kernel documentation page following this project's conventions.

## Input

`$ARGUMENTS` or conversation context provides:
- The subsystem (e.g., xHCI, PCIe, ACPI, USB4, DRM)
- The topic name (e.g., "host controller initialization", "MSI-X vectors")
- Optionally, an output directory override

If `$ARGUMENTS` is empty, derive the subsystem and topic from the conversation context.

## Procedure

### 1. Read the template

Before generating any content, read this file relative to `${CLAUDE_SKILL_DIR}`:

- `docs/templates/TEMPLATE-FULL.md` (page structure and section order)

### 2. Determine subsystem and output path

Look up the subsystem in the Subsystem Map (at the end of this file) to find:

- `tag`: the value for the `topics` and `tags` front matter fields
- `dir`: the output directory under `docs/`
- `kernel_paths`: directories in the kernel source tree to search first
- `spec`: specification name(s) for the SPECIFICATIONS section
- `section6_heading`: the heading to use for section 6 (REGISTERS, METHODS, PRIMITIVES, INTERFACES, or omit)

Construct the output path: `${CLAUDE_SKILL_DIR}/docs/<dir>/<topic-slug>.md`

If the output directory does not exist, create it.

### 3. Check for an existing page

Check whether a file already exists at the computed output path (`${CLAUDE_SKILL_DIR}/docs/<dir>/<topic-slug>.md`).

If the file exists, proceed to step 4.

If no file exists, stop and ask the user what to do. State the topic and subsystem you identified, then present these options:

1. Search the local kernel source tree and generate a full page (steps 4-9).
2. Create a minimal stub page now, without searching the kernel tree.
3. Cancel.

Wait for the user to choose before proceeding.

### 4. Search local kernel source code

Search the local kernel source tree (not the web) for relevant code. Use Grep and Glob to find:

- Source files in `kernel_paths` relevant to the topic
- Function definitions (with line numbers) using patterns like `^(static\s+)?\w+.*\bfunction_name\b\s*\(`
- Struct and macro definitions
- Comments referencing specification sections
- Files under `Documentation/` related to the topic

Record exact file paths and line numbers for every function, struct, or macro found.

### 5. Construct GitHub URLs

Use the base URL: `https://github.com/torvalds/linux/blob/v6.19/`

For file references:
```
[`path/to/file.c`](https://github.com/torvalds/linux/blob/v6.19/path/to/file.c)
```

For function references (include line number):
```
[`'\<function_name\>':'path/to/file.c'`](https://github.com/torvalds/linux/blob/v6.19/path/to/file.c#L1234)
```

For kernel documentation files:
```
[`Documentation/subsystem/file.rst`](https://github.com/torvalds/linux/blob/v6.19/Documentation/subsystem/file.rst): brief description
```

### 6. Identify specifications

Check source code comments and headers for references to specification chapters and sections. Map the subsystem to its known specifications using the `spec` field from the Subsystem Map.

Format each entry as: `<spec name>, section <N.N>: <section title>`

If no specification applies, leave the SPECIFICATIONS section present but empty.

### 7. Generate the page

Follow the template structure exactly. The page must contain these sections in order:

1. YAML front matter with `topics` and `tags` (include `"verification-needed"` tag)
2. H1: the topic name (just the name, no extra text)
3. A short summary paragraph with an ASCII diagram if appropriate
4. `## SUMMARY`
5. `## SPECIFICATIONS`
6. `## LINUX KERNEL`
7. `## KERNEL DOCUMENTATION`
8. `## OTHER SOURCES`
9. `## <section6_heading>` (from Subsystem Map; omit entirely if set to "none")
10. `## DETAILS`

### 8. Writing rules (mandatory)

All generated content must follow these rules:

- No em-dashes. Use parentheses instead: "CC (Command Completed)" not "CC --- Command Completed"
- No boldface (`**...**`)
- No negative constructions. Write "It is synchronous" not "It is synchronous, not asynchronous"
- No question-style headings. Write "Enable the Events" not "How Events Are Enabled"
- H1 is always the topic name only
- Every generated or modified page gets the `"verification-needed"` tag (at most one instance)
- Do not add any tags other than `"verification-needed"`
- `Documentation/` references go in KERNEL DOCUMENTATION, never in OTHER SOURCES

### 9. Save the page

Write the completed page to: `${CLAUDE_SKILL_DIR}/docs/<dir>/<topic-slug>.md`

Do not modify `mkdocs.yml`.

Ask before doing actual save.

## Subsystem Map

Each entry maps a subsystem to its tag, output directory, primary kernel source paths, specification name(s), and the heading to use for section 6.

### PCIe

- tag: `pcie`
- dir: `pci`
- kernel_paths: `drivers/pci/`, `include/linux/pci.h`, `include/uapi/linux/pci_regs.h`
- spec: PCI Express Base Specification
- section6_heading: REGISTERS

### xHCI

- tag: `usb` (secondary: `xhci`)
- dir: `xhci`
- kernel_paths: `drivers/usb/host/xhci*`, `include/linux/usb/hcd.h`
- spec: xHCI (eXtensible Host Controller Interface) Specification
- section6_heading: REGISTERS

### USB

- tag: `usb`
- dir: `usb`
- kernel_paths: `drivers/usb/core/`, `drivers/usb/common/`, `include/linux/usb.h`, `include/linux/usb/ch9.h`
- spec: USB 2.0 Specification, USB 3.2 Specification
- section6_heading: REGISTERS

### ACPI

- tag: `acpi`
- dir: `acpi`
- kernel_paths: `drivers/acpi/`, `include/acpi/`, `include/linux/acpi.h`
- spec: ACPI Specification
- section6_heading: METHODS

### USB4

- tag: `usb4`
- dir: `usb4`
- kernel_paths: `drivers/thunderbolt/`, `include/linux/thunderbolt.h`
- spec: USB4 Specification, Thunderbolt 3/4 Specification
- section6_heading: REGISTERS

### DisplayPort

- tag: `display-port`
- dir: `dp`
- kernel_paths: `drivers/gpu/drm/display/drm_dp*`, `include/drm/display/drm_dp*`
- spec: VESA DisplayPort Standard, VESA eDP Standard
- section6_heading: REGISTERS

### DRM

- tag: `graphics`
- dir: `drm`
- kernel_paths: `drivers/gpu/drm/`, `include/drm/`, `include/uapi/drm/`
- spec: (none; refer to DRM subsystem documentation)
- section6_heading: INTERFACES

### Sound

- tag: `sound`
- dir: `sound`
- kernel_paths: `sound/`, `include/sound/`, `include/uapi/sound/`
- spec: Intel High Definition Audio Specification, USB Audio Class Specification
- section6_heading: REGISTERS

### Power Management

- tag: `power-management`
- dir: `pm`
- kernel_paths: `drivers/base/power/`, `kernel/power/`, `include/linux/pm.h`, `include/linux/suspend.h`
- spec: ACPI Specification (power management chapters), PCI PM Specification
- section6_heading: none

### Concurrency

- tag: `concurrency`
- dir: `concurrency`
- kernel_paths: `kernel/locking/`, `include/linux/spinlock.h`, `include/linux/mutex.h`, `include/linux/rwsem.h`
- spec: (none)
- section6_heading: PRIMITIVES

### Drivers

- tag: `drivers`
- dir: `drivers`
- kernel_paths: `drivers/base/`, `include/linux/device.h`, `include/linux/platform_device.h`
- spec: (none)
- section6_heading: INTERFACES

### Debugging

- tag: `debugging`
- dir: `debugging`
- kernel_paths: `kernel/trace/`, `lib/dynamic_debug.c`, `include/linux/ftrace.h`
- spec: (none)
- section6_heading: none

### ARM64

- tag: `arm64`
- dir: `arm64`
- kernel_paths: `arch/arm64/`, `include/asm-generic/`
- spec: Arm Architecture Reference Manual (Arm ARM)
- section6_heading: REGISTERS

### Workflows

- tag: `workflows`
- dir: `workflows`
- kernel_paths: (none; workflow pages describe development processes)
- spec: (none)
- section6_heading: none
