# System Architecture Design Document (Version 1.0)
## Linvest: Linux Investigation Framework

## Architecture Overview
The Linvest (Linux Investigation Framework) architecture is designed around the principles of decoupling, modularity, and determinism. It follows a pipeline-driven approach where data collection, normalization, correlation, detection, and reporting are strictly separated into distinct subsystems. This ensures that the framework remains extensible and that the integrity of evidence is maintained throughout its lifecycle. The architecture is explicitly designed to support Debian-based distributions (including Kali Linux and Ubuntu) and prioritizes standalone execution to minimize the footprint on potentially compromised target systems.

## High-Level Architecture Diagram
```text
                            +-------------------------------------------+
                            |            Linvest Core Orchestrator      |
                            +-------------------------------------------+
                                   |                      |
    +------------------------------+                      +------------------------------+
    |                                                                                    |
+---v-------------------+    +-----------------------+    +-----------------------+  +---v-------------------+
| Configuration Manager |    | Plugin Manager        |    | Error & Logging       |  | State & Context       |
+-----------------------+    +-----------------------+    +-----------------------+  +-----------------------+
                                   |
                                   v
                      +-----------------------------+
                      |   Collector Framework       |
                      |  (Disk, Memory, Network)    |
                      +-----------------------------+
                                   | Raw Artifacts
                                   v
                      +-----------------------------+
                      |   Normalization Engine      |
                      +-----------------------------+
                                   | Unified Schema
                                   v
                      +-----------------------------+
                      |   Correlation Engine        |
                      +-----------------------------+
                                   | Correlated Graph
                                   v
                      +-----------------------------+
                      |   Rule & Detection Engine   |
                      +-----------------------------+
                                   | Indicators of Compromise (IoCs)
                                   v
             +-----------------------------------------------+
             |                                               |
+------------v------------+                     +------------v------------+
|  AI Interpretation      | (Optional)          |  Report Generation      |
|  Engine                 |-------------------->|  Pipeline               |
+-------------------------+                     +-------------------------+
                                                             |
                                                             v
                                                  +-----------------------+
                                                  | Final Output (JSON/MD)|
                                                  +-----------------------+
```

## Core Components
1.  **Core Orchestrator**: The central controller governing the lifecycle of the execution run.
2.  **Configuration Manager**: Parses, validates, and supplies operational parameters.
3.  **Collector Framework**: The execution environment for individual data collection plugins.
4.  **Normalization Engine**: Transforms heterogeneous raw data into a strictly typed, unified schema.
5.  **Correlation Engine**: Cross-references normalized data points to build a comprehensive state graph.
6.  **Rule & Detection Engine**: Applies deterministic logic against the correlated graph to identify anomalies.
7.  **AI Interpretation Engine**: An isolated, optional module that provides narrative context to deterministic findings.
8.  **Report Generation Pipeline**: Formats the final state graph and detected anomalies into actionable outputs.
9.  **State & Context Manager**: Maintains the runtime context, ensuring data is passed efficiently between pipeline stages.
10. **Error & Logging Manager**: Handles internal operational telemetry and robust error recovery.

## Responsibilities of Every Component
*   **Core Orchestrator**: Bootstraps the framework, initializes managers, and sequentially invokes pipeline stages. It is strictly procedural and handles no data manipulation itself.
*   **Configuration Manager**: Ingests YAML/JSON configurations and CLI arguments, overriding defaults securely.
*   **Collector Framework**: Sandboxes and schedules individual plugins, enforcing timeout constraints and resource limits to prevent system degradation.
*   **Normalization Engine**: Maps OS-specific structures (e.g., `/proc` parsing, `ss` output) into standard Linvest schemas (e.g., `NetworkConnection`, `ProcessNode`).
*   **Correlation Engine**: Discovers relationships (e.g., matching a `NetworkConnection`'s PID to a `ProcessNode` and its parent `ProcessNode`).
*   **Rule & Detection Engine**: Ingests the correlated graph and applies YAML-based detection rules (e.g., "Alert if `ProcessNode` running from `/tmp` has a listening `NetworkConnection`").
*   **AI Interpretation Engine**: Ingests purely the flagged anomalies and outputs a human-readable threat narrative. It **does not** collect evidence or define rules.
*   **Report Generation Pipeline**: Consolidates raw evidence, matched rules, and AI narratives into the final deliverable (e.g., JSON for SIEM ingestion, Markdown for human reading).

## Execution Lifecycle
1.  **Bootstrap Phase**: The Core Orchestrator initializes, loading the Configuration Manager and Error & Logging Manager.
2.  **Plugin Discovery**: The Plugin Manager scans the designated plugin directories, validating signatures and compatibility.
3.  **Collection Phase**: The Collector Framework dispatches authorized plugins concurrently or sequentially based on dependency graphs.
4.  **Data Processing Phase**: Raw data is pipelined through the Normalization Engine and then the Correlation Engine.
5.  **Detection Phase**: The Rule & Detection Engine evaluates the correlated data graph against active rulesets.
6.  **Enrichment Phase**: If enabled, the AI Interpretation Engine processes the positive hits.
7.  **Finalization Phase**: The Report Generation Pipeline constructs the final output and writes it to disk. The State & Context Manager performs safe cleanup.

## Internal Data Flow
The data flow is strictly unidirectional down the pipeline.
1.  Plugins yield `RawArtifact` objects.
2.  The Normalization Engine consumes `RawArtifact` and yields `NormalizedEntity`.
3.  The Correlation Engine consumes lists of `NormalizedEntity` and constructs an `EvidenceGraph`.
4.  The Rule Engine consumes the `EvidenceGraph` and yields `DetectionEvent` objects.
5.  The AI Engine consumes `DetectionEvent` objects and yields `NarrativeContext`.
6.  The Report Pipeline consumes `EvidenceGraph`, `DetectionEvent`, and `NarrativeContext` to produce the final output.

## Component Interaction
Components interact strictly through defined, stable interfaces managed by the State & Context Manager. Direct dependency between pipeline stages is prohibited. For example, the Collector Framework has no awareness of the Rule Engine. Communication occurs via in-memory message passing or bounded queues, ensuring that a failure in one stage (e.g., a specific collector timing out) can be gracefully handled without crashing the pipeline.

## Collector Framework Architecture
The Collector Framework utilizes a supervisor model.
*   **Supervisor**: Spawns and monitors execution threads/processes for plugins.
*   **Resource Monitor**: Tracks CPU and memory consumption of active plugins, terminating those that exceed configured thresholds (e.g., to prevent a rogue collector from crashing a fragile target system).
*   **Data Sink**: A unified interface where plugins asynchronously push their collected raw artifacts.

## Plugin Architecture
Plugins are the lifeblood of data collection.
*   **Interface Contract**: Every plugin must implement a strict `ICollector` interface defining `Initialize()`, `Collect()`, and `Teardown()`.
*   **Isolation**: Plugins execute in isolated contexts to prevent a panic in one plugin from bringing down the entire framework.
*   **Metadata**: Plugins must supply metadata declaring their capabilities, required permissions (e.g., CAP_SYS_ADMIN), and expected output format.

## Rule Engine Placement and Responsibilities
The Rule Engine sits strictly *after* data correlation and *before* AI interpretation.
*   **Responsibility**: Execute boolean logic and pattern matching against the `EvidenceGraph`.
*   **Determinism**: It relies solely on predefined, observable indicators. It does not employ heuristic guessing or probabilistic models. A rule either matches the evidence, or it does not.

## AI Engine Placement and Responsibilities
The AI Interpretation Engine is placed at the very end of the analysis pipeline and is entirely optional.
*   **Responsibility**: To summarize and provide context to the deterministic findings of the Rule Engine.
*   **Constraint**: It cannot alter the `EvidenceGraph`, it cannot generate new `DetectionEvents`, and it cannot initiate data collection. It is strictly an advanced formatter and summarizer for human analysts.

## Report Generation Pipeline
The pipeline supports multiple interchangeable sinks:
*   **JSON Sink**: For programmatic ingestion by SIEMs, SOARs, or external databases. Contains the full `EvidenceGraph`.
*   **Markdown/HTML Sink**: For human-readable incident response reports, highlighting `DetectionEvents` and AI narratives.
*   **Archive Sink**: Compresses the raw artifacts and reports into a secure, encrypted ZIP/tarball for exfiltration from the target environment.

## Configuration Management
Configuration relies on a hierarchical resolution strategy:
1.  **Hardcoded Defaults**: Base operational state.
2.  **Environment Variables**: Useful for containerized or orchestrated deployments.
3.  **Local Configuration File**: e.g., `linvest.yaml`.
4.  **Command-Line Interface (CLI) Arguments**: Highest precedence overrides.
The Configuration Manager produces an immutable configuration object passed to all subsystems during initialization.

## Error Handling Strategy
*   **Fail-Safe**: If a non-critical module (e.g., an individual collector or the AI Engine) fails, the framework logs the error, safely terminates the module, and continues execution.
*   **Fail-Fast**: If a critical module (e.g., the Core Orchestrator or Configuration Manager) fails during initialization, the framework halts immediately with a clear, standard error exit code to prevent unpredictable behavior.
*   **Graceful Degradation**: If system resources are heavily constrained, the Collector Framework will automatically disable heavy plugins (e.g., full filesystem hashing) to maintain operational stability.

## Logging Strategy
*   **Internal Telemetry**: Distinct from forensic evidence collection. Telemetry logs the internal state of the framework (e.g., "Plugin X started", "Rule Y evaluated").
*   **Verbosity Levels**: Supports standard TRACE, DEBUG, INFO, WARN, ERROR, and FATAL levels.
*   **Destination**: Telemetry is written to `stderr` or a dedicated local log file (`linvest_run.log`) to avoid contaminating the standard output which may be reserved for the JSON report payload.

## Future Scalability Considerations
*   **Concurrency Models**: The architecture is designed to support a transition from local multi-threading to distributed execution (e.g., gRPC-based remote collectors) without altering the core pipeline logic.
*   **Streaming Analytics**: The Normalization and Correlation engines are structured to eventually support continuous streaming data rather than batch-only processing, paving the way for long-running agent deployments if required by future iterations.
