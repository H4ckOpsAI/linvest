# Project Foundation Document: Linvest (Linux Investigation Framework)

## Product Definition
Linvest is a specialized, open-source orchestration engine that deterministically collects, correlates, and interprets Linux system artifacts to identify indicators of compromise without functioning as an active security control.

## Executive Summary
Linvest (Linux Investigation Framework) is an open-source, host-based investigation framework designed to orchestrate evidence collection and facilitate deterministic analysis on Linux systems. It provides security engineers and incident responders with a unified, transparent mechanism to acquire, correlate, and interpret forensic data. By decoupling evidence aggregation from interpretation, Linvest ensures reproducibility and rigor in post-compromise investigations without functioning as an active, preventative security control.

## Vision Statement
To establish the premier standard for deterministic, reproducible, and transparent Linux host investigations, empowering security professionals to uncover ground truth through uncompromised evidence correlation.

## Problem Statement
Incident response on Linux systems often suffers from fragmented data collection and over-reliance on opaque, proprietary tools. Security teams are frequently required to manually aggregate system state, logs, and artifacts from disparate sources, leading to incomplete analyses, missed threat indicators, and non-reproducible investigation outcomes.

## Why Existing Linux Security Tools Are Not Enough
Current open-source ecosystem tools serve distinct, specialized purposes that do not fully encompass the requirements of a comprehensive investigation framework:

*   **rkhunter & chkrootkit**: These are legacy, signature-based scanners focused strictly on known rootkits. They lack comprehensive system state awareness and cannot perform deep behavioral or chronological correlation.
*   **Lynis**: Designed primarily for system hardening and compliance auditing, Lynis does not facilitate post-breach evidence collection or forensic investigation.
*   **AIDE**: A traditional file integrity monitoring (FIM) tool. While useful for detecting file changes, it lacks context regarding network activity, memory state, or execution history.
*   **OSQuery**: Provides a powerful SQL interface to operating system state, but operates as a generalized query engine rather than an orchestrated investigation framework. It lacks built-in forensic correlation and automated reporting tailored for incident response.

## Proposed Solution
Linvest bridges the gap by acting as a comprehensive orchestration layer for evidence collection and analysis. It deterministically aggregates data across key Linux subsystems (e.g., file systems, process trees, network states, and system logs), correlates this data to establish a timeline of events, and provides a structured output for human review. AI capabilities are strictly constrained to interpreting the collected evidence and generating readable reports, ensuring that the underlying data collection remains deterministic, verifiable, and transparent.

## Mission Statement
To provide a modular, reliable, and evidence-driven framework that accelerates Linux incident response by standardizing data collection and enhancing human analysis through deterministic correlation.

## Engineering Design Goals
*   **Minimal Footprint**: The collection engine must not interfere with or degrade the performance of the target system, maintaining operational stability even in degraded environments.
*   **Decoupled Architecture**: Subsystem data collection (e.g., memory, disk, network) must be strictly isolated from the correlation and interpretation logic.
*   **Auditability**: Every finding or flagged anomaly must trace directly back to the raw, verifiable data source it was generated from.
*   **Resilience against Obfuscation**: The framework should prioritize methods of data retrieval that bypass superficial, user-space tampering where practical and safe.

## Core Engineering Principles
*   **Evidence-Driven Investigation**: All conclusions must be traceable to specific, verifiable system artifacts.
*   **Deterministic Analysis**: Data collection and correlation must produce consistent, reproducible results across identical system states.
*   **Modularity**: The framework must support pluggable collectors and analyzers to adapt to evolving threats and varying environments.
*   **Transparency**: Collection methodologies and correlation logic must be open, understandable, and well-documented.
*   **Reproducibility**: Investigations must be repeatable, yielding the same artifacts and reports when run against the same dataset.
*   **Offline Capability**: The framework must function fully in air-gapped or network-restricted environments, preventing data exfiltration and ensuring operational security during an investigation.

## Target Users
*   Incident Responders (IR)
*   Digital Forensics (DFIR) Analysts
*   Security Operations Center (SOC) Engineers
*   Systems Administrators and Site Reliability Engineers (SRE)
*   Cybersecurity Researchers

## Product Scope

### In Scope
*   Orchestrated collection of volatile and non-volatile system data.
*   Aggregation of logs, process information, network connections, and file metadata.
*   Deterministic correlation of collected artifacts to identify anomalies.
*   Modular architecture allowing custom evidence collectors.
*   Offline execution and localized data processing.
*   AI-assisted interpretation of raw evidence and automated report generation.

### Out of Scope
*   Real-time prevention, blocking, or remediation of threats (Linvest is not an Antivirus or EDR).
*   Active memory modification or process termination.
*   Continuous, background monitoring or alerting.
*   Automated system hardening or configuration management.

## Version 1 Threat Model

### Designed to Detect (Observable Indicators)
*   **Anomalous Processes**: Execution of binaries from non-standard paths, processes with masked names, or unexpected parent-child relationships.
*   **Persistence Mechanisms**: Unauthorized modifications to `cron` jobs, `systemd` services, startup scripts, and `LD_PRELOAD` environment variables.
*   **Suspicious Network Activity**: Unexpected open ports, unusual outbound connections from standard system binaries, or concealed listeners.
*   **File System Discrepancies**: Hidden files, anomalous SUID/SGID binaries, and timestamp inconsistencies.
*   **Account Anomalies**: Unauthorized user creation, unexpected privilege escalations, or SSH key manipulations.

### Intentionally Does Not Detect
*   **In-Memory Only Execution**: Without a memory snapshot or kernel module, sophisticated memory-only payloads (e.g., advanced fileless malware) that leave no disk or network footprint may bypass detection.
*   **Network Packet Payload Inspection**: Linvest is not a packet sniffer or IDS; it observes state, not transit payloads.
*   **Zero-Day Exploits in Real-Time**: It does not block or flag the exploitation process as it occurs, only the post-exploitation artifacts left behind.

## Competitive Positioning
Linvest distinguishes itself by avoiding active defense mechanisms. Unlike EDRs that focus on behavioral blocking, or FIMs that focus on compliance, Linvest focuses exclusively on the investigative phase of incident response. It is a post-event, deep-dive tool that orchestrates data gathering and correlation, providing a single source of truth for investigators without altering the target system state.

## Project Goals
*   Develop a stable, core orchestration engine for data collection.
*   Implement a baseline set of modular collectors for essential Linux subsystems.
*   Establish a deterministic correlation engine for initial anomaly detection.
*   Integrate an optional, localized AI module strictly for report generation and evidence synthesis.
*   Ensure the entire framework can operate without external network dependencies.

## Success Criteria
*   The framework successfully executes and aggregates data across major Linux distributions (e.g., Debian and RHEL families).
*   Modular collectors can be added or removed without altering the core engine.
*   Analysis yields reproducible results across multiple runs on a static system image.
*   AI interpretation can be entirely disabled, leaving a fully functional deterministic framework.
*   Zero external network requests are made by the core framework during execution.

## Measurable Success Metrics for Version 1
*   **Collection Latency**: Data extraction from a baseline Linux installation completes in under 60 seconds.
*   **False Positive Rate**: Baseline system state on a fresh, untampered installation of supported distributions results in zero high-severity alerts.
*   **Correlation Accuracy**: Injection of known, static indicators of compromise (e.g., a dummy cron persistence script) triggers the expected detection logic 100% of the time.
*   **Standalone Execution**: Verification that the core executable and its required modules can be deployed as a single, self-contained binary or archive without requiring compilation on the target host.

## Assumptions
*   The executing user will have appropriate permissions (e.g., root access) to collect necessary system artifacts.
*   Target systems conform to standard Linux hierarchical structures.
*   Investigations may occur on potentially compromised systems; therefore, the framework must minimize reliance on potentially untrusted local binaries where possible.

## Constraints
*   The framework must prioritize minimal system footprint during execution to preserve forensic integrity.
*   It must not require complex, heavy dependencies that are difficult to deploy on legacy or minimal environments.
*   AI features must be designed to run locally or rely on explicitly configured endpoints, prioritizing data privacy.

## Risks
*   **Data Tampering**: Advanced rootkits or kernel-level malware may manipulate the APIs or raw disk data the framework relies upon.
*   **Scope Creep**: The temptation to add active response features could compromise the framework's core investigative identity.
*   **Performance Overhead**: Extensive data collection might inadvertently cause resource exhaustion on fragile production systems.

## Investigation Philosophy
Linvest operates on a strict pipeline designed to separate raw data from subjective interpretation:
1.  **Evidence Collection**: Immutable gathering of system state data across designated subsystems.
2.  **Normalization**: Standardizing disparate data formats (e.g., log entries, `/proc` data) into a unified schema.
3.  **Correlation**: Cross-referencing normalized data (e.g., mapping a suspicious network connection to its originating process and executable file).
4.  **Detection**: Applying deterministic rulesets to identify deviations from known baselines or expected behaviors.
5.  **AI Interpretation (Optional)**: Synthesizing the correlated findings into human-readable narratives, explaining *why* an anomaly might be significant.
6.  **Reporting**: Outputting the final findings along with the verifiable evidence trail.

> [!IMPORTANT]
> **Linvest never declares a system "safe" or "clean."** It only reports the presence or absence of observable indicators of compromise based on the collected evidence. The absence of findings does not guarantee the absence of an adversary.

## Guiding Design Philosophy
"Collect deterministically, correlate transparently, and interpret cautiously." The framework exists to augment the human investigator by providing pristine, orchestrated evidence, never to obscure the investigation behind opaque algorithms or active defense mechanisms.

## Long-Term Product Vision
While Version 1 focuses on deterministic host-based collection and correlation, the long-term roadmap envisions Linvest evolving into a distributed investigation fabric. Future iterations may include:
*   **Centralized Evidence Aggregation**: Allowing multiple independent Linvest agents to push evidence to a central, secure repository for fleet-wide correlation.
*   **Advanced Memory Forensics**: Integration with specialized memory analysis frameworks (e.g., Volatility) for deep introspection.
*   **Automated Threat Hunting**: Expanding the correlation engine to support complex, graph-based queries for proactive threat hunting across large environments.
*   **Cross-Platform Expansion**: While currently Linux-centric, the underlying orchestration and correlation architecture will be designed to eventually accommodate other operating systems.
