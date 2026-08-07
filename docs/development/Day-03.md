# Day 03

## Date
08 August 2026

## Milestone
Evidence Collection Framework

## Objectives
- Define the evidence collection philosophy
- Categorize investigation evidence
- Design collector taxonomy
- Define parsing and normalization pipeline
- Establish execution order
- Define safe collection constraints

## Completed

- Designed evidence collection framework
- Defined evidence categories
- Established collector taxonomy
- Separated raw artifacts, parsed evidence, and normalized entities
- Defined normalization requirements
- Defined order of volatility
- Established collector constraints
- Defined handling of hostile environments
- Documented Version 1 collection boundaries

## Key Decisions

- Evidence before analysis
- /proc preferred over user-space binaries when possible
- Passive collection only
- Offline-first evidence acquisition
- Standardized normalization model
- Modular collector architecture

## Lessons Learned

The quality of Linvest depends more on the quality of evidence than on the sophistication of AI. A deterministic, structured, and reproducible evidence pipeline is the foundation of trustworthy investigation.

## Next Objective

Design the complete Collector Framework, including collector lifecycle, interfaces, registration model, dependency management, execution scheduling, and plugin integration.
