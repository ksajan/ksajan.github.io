+++
title = "How we process 10,000 live video streams for a state transportation department"
date = 2025-01-10T00:00:00-05:00
draft = false
summary = "Real-time computer vision across a statewide CCTV network: multi-object tracking, cross-camera re-ID, and pipelines that survive bad bandwidth and edge hardware."
tags = ["computer-vision", "tracking", "distributed-systems", "optimization", "ops"]
categories = ["engineering"]
authors = ["Sajan Kumar"]
cover = ""
+++

A statewide CCTV network does not look like the clean video in a computer vision paper. The cameras were bought over many years from different vendors. Lighting changes through the day and with the weather. Bandwidth is limited and drops without warning. And when something fails, the system has to fail in a predictable way, because people rely on it to see what is happening on the road.

We built a pipeline that handles about 10,000 live streams at once for a state transportation department. Most of the hard parts were not about the models. They were about keeping everything around the models running.

## Split the pipeline so each part can fail alone

The first version treated a stream as one unit of work: pull the video, detect, track, emit events. That works for a demo. At this scale, one slow stage holds up everything behind it, and one bad camera can take a worker down with it.

So we split the pipeline into separate stages: ingest, decode, detector, tracker and event router. Each one scales on its own. Decoding is cheap but constant. Detection is expensive and comes in bursts. Treating them as one unit meant sizing everything for the worst case.

Failures got the same treatment. Dead-letter queues catch work that cannot be processed, and circuit breakers stop a failing camera from retrying itself into a storm. A broken camera should be one missing feed, not an outage.

## Put the right model in the right place

Not every frame needs a heavy model, and not every location can run one. Lightweight detectors run on edge nodes close to the cameras. Heavier models run at regional hubs. That kept latency predictable, so we were not waiting on a busy central cluster for every frame.

We also added a routing step that decides which frames need heavy processing at all. It combines a CNN with an LLM. Most frames in a traffic feed are boring, and skipping the heavy path for those was one of the biggest wins. Frame processing latency dropped by about 40%.

## Following a vehicle across cameras

Tracking inside one camera is a known problem. Following the same vehicle from one camera to the next is harder, because the angle, the lighting and the resolution all change. We used transformer-based embeddings for cross-camera re-identification and cached them in a vector store keyed by time and region. That way a match only has to search the cameras that could plausibly have seen the same vehicle, not the whole state.

## Getting the GPUs to do work

Profiling showed the GPUs sitting idle more than they should. Batching and CUDA stream tuning brought utilization above 80%. Quantization and mixed precision cut cost further, but only where they did not hurt accuracy. Night footage and bad weather are the cases that matter most for safety, and they are exactly where a compressed model loses accuracy first. So we checked mAP on those cases before each change, not just on the average.

## What it changed

The system runs real-time safety analytics and automated incident detection across the network. Manual monitoring costs dropped by about 60%, and planners now have a much richer dataset for infrastructure decisions.

The part I would emphasize to anyone building something similar is observability. Traces, metrics and tagged example frames meant teams without ML experience could operate the pipeline and understand why it did what it did. A system like this lives or dies on whether the people running it at 2am can tell what is wrong.
