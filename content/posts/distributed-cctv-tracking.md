+++
title = "How we process 10,000 live video streams for a state transportation department"
date = 2025-01-10T00:00:00-05:00
draft = false
summary = "Building real-time computer vision at INDOT scale: multi-object tracking, cross-camera re-ID, and pipelines that survive bad bandwidth and edge hardware."
tags = ["computer-vision", "tracking", "distributed-systems", "optimization", "ops"]
categories = ["engineering"]
authors = ["Sajan Kumar"]
cover = ""
+++

Statewide CCTV networks come with painful constraints: heterogeneous hardware, changing lighting, bandwidth limits, and the need for deterministic failover. Here is how we tackled it.

## Architecture
- Split the pipeline into ingest, decode, detector, tracker, and event router stages, each independently autoscalable.
- Used transformer-based embeddings for cross-camera re-ID and cached them in a vector store keyed by time and region.
- Pushed lightweight detectors to edge nodes while reserving heavier models for regional hubs to keep latency predictable.
- Added dead-letter queues and circuit breakers so individual camera failures did not cascade.

## Performance work
- Profiled GPU utilization across stages; batching and CUDA stream tuning delivered 80%+ utilization.
- Reduced frame processing latency by ~40% with a hybrid LLM+CNN routing step that decides which frames need heavy processing.
- Applied quantization and mixed precision where possible without tanking mAP on night and weather edge cases.

## Outcomes
- Processes 10K+ streams per day with real-time safety analytics and automated incident detection.
- Reduced manual monitoring costs by ~60% and gave planners a richer dataset for infrastructure decisions.
- Clear observability (traces, metrics, and tagged examples) made it easier for non-ML teams to operate the pipeline.
