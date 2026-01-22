# Self‑Declarative Pipelines: A Palantir Foundry Perspective (with Foundry‑Native Examples)

## Collab
1. [Ankita Hatibaruah](https://github.com/Ahb98), [LinkedIn](http://linkedin.com/in/ankita-hatibaruah-bb2a62218)
2. [Pavithra Ananthakrishnan](https://github.com/Pavi-245), [LinkedIn](https://www.linkedin.com/in/pavithra-ananthakrishnan-552416244/)
3. [Sree Bhavya Kanduri](https://github.com/sreebhavya10), [LinkedIn](https://www.linkedin.com/in/kanduri-sree-bhavya-4001a6246?utm_source=share&utm_campaign=share_via&utm_content=profile&utm_medium=android_app)

## TL;DR
Spark 4.1’s Spark Declarative Pipelines (SDP) popularize a dataset‑centric approach, declare the datasets and queries; let the engine plan and run the graph. In Palantir Foundry, you build the same kind of pipelines using Transforms (@transform, @transform_df, @incremental) that materialize datasets and are orchestrated by Pipelines with scheduling, lineage, and governance built‑in. 

## Overview
Data engineering is moving from imperative, job‑centric orchestration to dataset‑centric systems. SDP formalizes this in core Spark: you specify what datasets should exist and how they’re derived; Spark plans the graph, retries, and incremental updates for you. Foundry embodies the same philosophy: you declare Transforms that read inputs and write outputs (datasets); Pipelines schedule execution; and Lineage tracks the DAG across your enterprise.

## What SDP Is (and how it maps)
- SDP introduces flows, datasets (streaming tables, materialized views, temporary views), and a pipeline spec to run.
- Foundry maps naturally:
- SDP flow → Foundry Transform reading Inputs and writing one or more Outputs.
	- SDP materialized view → Persisted transform output dataset (the canonical way to “materialize” in Foundry).
	- SDP temporary view → Intermediate dataset (or a Foundry View if you want read‑time union/dedup without storing new files).
	- SDP graph execution → Foundry Pipelines + schedules + Lineage to monitor dependencies and impact.
	- For incremental behavior, Foundry provides the @incremental() decorator.

## Why this matters (especially for Foundry)
Foundry emphasizes dataset lineage, governed modeling (Ontology), incremental recomputation, and separation between what data represents and how it’s computed. The dataset centric style you learn from SDP aligns naturally with Foundry’s core apps and best practices,only here you also get enterprise‑grade lineage, access control, checks, and auditing as first‑class.

## The Dataflow Graph (in Foundry)
In Foundry, the DAG is explicit in Data Lineage: transforms (nodes) consume input datasets and produce output datasets; lineage updates as you build. You can explore ancestor/descendant relationships, see schemas and last build times, and color or snapshot parts of the graph for collaboration.

## Key Concepts (SDP → Foundry)
	-Transforms (Foundry) = flows (SDP): a unit of compute that reads Inputs and writes Outputs, defined with @transform / @transform_df. Use multiple outputs when needed.
	-Datasets (Foundry) = persisted outputs. Use Views when you want a union or primary‑key dedup at read time without writing new files.
	-Pipelines (Foundry) = orchestration + schedules, with native lineage and governance.
	-Incremental: add @incremental() when inputs and logic meet constraints so only new data is processed.

## Pipeline Project (Foundry)
	-Author Python transforms in a Code Repository.
	-Declare Inputs/Outputs in decorators; write a DataFrame to each Output.
	-Register transforms in a Pipeline, set schedules, and monitor Lineage.
