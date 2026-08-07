# Evidence Collection Design Document (Version 1.0)
## Linvest: Linux Investigation Framework

## Evidence Collection Goals
The primary objective of the Linvest Evidence Collection layer is to acquire high-fidelity system state data deterministically, reproducibly, and with minimal operational footprint. The framework must operate effectively in potentially hostile, air-gapped, or degraded environments (specifically targeting Debian-based distributions like Kali Linux and Ubuntu), ensuring that all acquired artifacts are immutable, verifiable, and structured for downstream analysis.

## Evidence Categories
To facilitate structured investigation, evidence is segmented into logical domains:
*   **Identity & Access**: User accounts, groups, SSH keys, sudoers configurations, and authentication records.
*   **Persistence Mechanisms**: `cron` jobs, `systemd` timers, init scripts, `rc.local`, and shell profiles.
*   **Network State**: Active connections, listening sockets, routing tables, DNS configurations, and firewall rules.
*   **Processes & Execution**: Running processes, parent-child relationships, command-line arguments, memory maps, and loaded libraries.
*   **Filesystem & Artifacts**: Hidden files, SUID/SGID binaries, modified timestamps, temporary directories (`/tmp`, `/var/tmp`), and deleted-but-open files.
*   **Kernel & Modules**: Loaded kernel modules (LKM), kernel parameters (`sysctl`), and `dmesg` ring buffer.
*   **System Logs**: Authentication logs (`auth.log`), system logs (`syslog`, `journald`), and application-specific logs.
*   **Services & Daemons**: Active `systemd` units, enabled services, and service configurations.
*   **Containers & Virtualization**: Docker/Podman container states, namespaces, and cgroups.
*   **Hardening & Configuration**: AppArmor/SELinux statuses, PAM configurations, and SSH daemon settings.

## Collector Taxonomy
Collectors are categorized by their interaction method:
*   **API/Syscall Collectors**: Interact directly with the kernel or `/proc` and `/sys` filesystems (e.g., parsing `/proc/[pid]/environ`). These are the preferred, most resilient sources.
*   **File Parsing Collectors**: Read and parse static configuration or log files directly (e.g., reading `/etc/passwd`).
*   **Native Binary Collectors**: Execute built-in system utilities (e.g., `netstat`, `ss`, `ps`) and capture `stdout`. These are used as a fallback when direct API collection is impractical or highly complex.

## Evidence Types and Examples
To ensure data purity, Linvest strictly distinguishes between data phases:
1.  **Raw Artifacts**: Unmodified bytes acquired from the system. *(Example: The raw string output of `/bin/ss -tuln` or the raw byte contents of `/etc/passwd`.)*
2.  **Parsed Evidence**: Raw artifacts tokenized into structured, intermediate formats. *(Example: An array of strings representing the delimited fields of a single passwd entry.)*
3.  **Normalized Entities**: Parsed evidence mapped to Linvest's strict internal schema. *(Example: A `NetworkConnection` object with strongly typed fields like `LocalPort (int)`, `RemoteIP (IPAddress)`, and `Protocol (Enum)`.)*

## Collector Responsibilities
*   **Targeted Acquisition**: Gather only the specific artifact requested without broad, unfocused scanning.
*   **Timeboxing**: Enforce strict execution timeouts to prevent hanging on blocking I/O or stalled system calls.
*   **Metadata Generation**: Record acquisition metadata, including execution time, command used (if applicable), and collector version.
*   **Immutability**: Read data passively. Under no circumstances should a collector alter the system state or modify target files.

## Parser Responsibilities
*   **Ingestion**: Accept raw artifacts from the collector safely.
*   **Tokenization**: Split and structure text while handling edge cases like malformed lines, unexpected encodings, or missing delimiters.
*   **Validation**: Verify that the parsed data conforms to expected patterns (e.g., ensuring an IP address string is valid) before passing it to the Normalization Engine.

## Normalization Requirements
The normalization phase translates parsed evidence into a unified, framework-wide schema required by the Correlation Engine.
*   **Type Safety**: All fields must be strongly typed (e.g., timestamps as UTC objects, integers as numeric types).
*   **Cross-Referenceability**: Entities must possess unique identifiers (e.g., combining PID and Start Time) to allow the Correlation Engine to link disparate data points (e.g., mapping an active network connection to the process that opened it).
*   **Graceful Degradation**: If an artifact lacks a specific field due to OS variations (e.g., differences between Kali Linux and older Ubuntu LTS releases), the normalized entity must safely mark the field as `null` rather than failing the entire entity construction.

## Priority and Execution Order
Collectors execute based on volatility, strictly adhering to the Order of Volatility (OOV) to preserve forensic integrity:
1.  **High Volatility**: Memory-mapped areas, active network connections, running process states.
2.  **Medium Volatility**: Kernel routing tables, ARP caches, temporary files (`/tmp`, `/dev/shm`).
3.  **Low Volatility**: Static configuration files (`/etc`), persistent logs (`/var/log`), binary attributes.

Dependency graphs dictate execution order; for example, the Process Collector must finish before the Network Collector attempts to map listening sockets to parent PIDs.

## Safe Collection Constraints
*   **Minimal System Impact**: Collectors must avoid heavy disk I/O (like full-disk hashing) unless explicitly requested via configuration. Memory usage must remain tightly bounded.
*   **Reproducibility**: Repeated execution on a frozen system state must yield identical raw artifacts.
*   **Offline Operation**: No collector may attempt external network communication (e.g., querying threat intelligence APIs for a hash). All collection relies strictly on local artifacts.

## Handling Missing Permissions or Missing Commands
*   **Privilege Checks**: Collectors must verify capabilities (e.g., `EUID == 0` or specific `CAP_*` flags) prior to execution.
*   **Missing Binaries**: If a Native Binary Collector cannot locate its target (e.g., `ss` is uninstalled), it must gracefully exit, log a warning, and yield an empty artifact set, rather than crashing the pipeline.
*   **File Access Denied**: Attempting to read a restricted file (e.g., `/etc/shadow`) without permissions must be caught, logged, and bypassed smoothly.

## Handling Potentially Compromised Hosts
Because Linvest operates on potentially hostile systems (e.g., environments with active rootkits or masked processes):
*   **Direct API Preference**: Prefer reading `/proc` over executing `/bin/ps`, as user-space binaries may be trojaned or hooked via `LD_PRELOAD`.
*   **Sanitization**: All raw artifacts must be treated as untrusted, hostile input. Parsers must defend against buffer overflows, injection attempts, and malformed strings engineered to crash the investigator tool.

## Output Format Expectations
Collectors do not write to disk directly. They return data structures to the in-memory Data Sink managed by the Collector Framework. Serialization and persistence are solely the responsibility of the downstream Report Generation Pipeline.

## Extensibility for Future Collectors
The Collector Framework relies on a standardized, decoupled interface (`ICollector`). Adding a new collector requires only implementing the required initialization, collection, and teardown contracts, followed by registration in the Plugin Manager. The Core Orchestrator requires no modification to support new evidence sources.

## What Linvest Deliberately Does Not Collect in Version 1
*   **Full Memory Dumps**: Linvest will inspect `/proc` and memory maps for active state, but acquiring multi-gigabyte raw RAM dumps is out of scope for the V1 orchestration engine.
*   **Full Disk Images (dd)**: Linvest performs live logical investigation, not bit-for-bit block device imaging.
*   **Network Packet Captures (PCAP)**: Linvest captures static network state (listening ports, active sockets, routes) but does not hook into network interfaces to capture transit payloads.
