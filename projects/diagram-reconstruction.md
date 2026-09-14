---
layout: default
title: Diagram Reconstruction
permalink: /projects/diagram-reconstruction/
---

# Diagram Reconstruction

## Overview

An AI-assisted experiment for reconstructing diagrams from images into
editable formats such as draw.io and Mermaid.

The project explores how vision models and agentic workflows can be used
to recover both the semantic structure and visual layout of an existing
diagram.

---

## Why

Converting an image of a diagram into an editable diagram sounds like
a straightforward vision-to-code problem.

In practice, there are two different challenges:

1. Understanding the diagram semantically.
2. Reconstructing its visual structure and layout.

An early implementation showed that an AI model could often recover
nodes and relationships correctly, while the resulting editable diagram
could still look significantly different from the original.

This became an experiment in designing a more reliable AI engineering
workflow.

---

## What I Tried

The initial workflow was approximately:

```text
Image
  ↓
Gemini
  ↓
Mermaid / draw.io
  ↓
Headless draw.io rendering
  ↓
Validation
```

For larger diagrams, I experimented with:
```text
Large image
  ↓
Split into smaller regions
  ↓
Analyze each region
  ↓
Generate individual diagrams
  ↓
Validate
  ↓
Merge
```
This improved some local recognition problems but introduced new
challenges around global context and layout consistency.

## Current Architecture
The direction I am exploring is:
```text
                     Original Image
                           │
                           ▼
                  Global Visual Analysis
                           │
                           ▼
                    Region Detection
                           │
             ┌─────────────┼─────────────┐
             ▼             ▼             ▼
        Local Analysis Local Analysis Local Analysis
             └─────────────┼─────────────┘
                           ▼
                    Diagram IR
                           │
                           ▼
                  Global Reconciliation
                           │
                           ▼
                    Layout Engine
                           │
                           ▼
                  draw.io Generator
                           │
                           ▼
                    Headless Render
                           │
                           ▼
                     Validation
```
The key architectural idea is to introduce an intermediate
representation rather than generating the target format directly from
the LLM.

### Diagram IR
The intermediate representation describes the diagram independently
of the target format.

```json
For example:
{
  "nodes": [
    {
      "id": "service-1",
      "type": "service",
      "label": "Order Service",
      "geometry": {
        "x": 200,
        "y": 150,
        "width": 180,
        "height": 80
      }
    }
  ],
  "edges": [
    {
      "source": "service-1",
      "target": "database-1",
      "relationship": "writes"
    }
  ]
}
```
This allows the same semantic representation to potentially support
multiple output formats:
```text
                 Diagram IR
                /    |     \
               /     |      \
          draw.io  Mermaid   SVG
```

## Key Engineering Lessons
1. Semantic correctness is different from visual correctness

A generated diagram can contain the correct nodes and edges while still
having a substantially different layout.

2. Local correctness does not guarantee global correctness

Splitting a large image can improve local recognition, but independent
processing can lose global relationships and spatial context.

3. Merge the model, not the finished diagrams

Rather than generating several independent draw.io diagrams and merging
them, a better approach is to merge the intermediate representations
and perform global layout afterwards.

4. LLMs should not own the entire pipeline

LLMs are useful for visual interpretation, semantic classification and
reasoning.

Deterministic software is better suited for layout constraints,
serialization, merging and validation.

5. Validation needs multiple layers

A useful evaluation strategy should distinguish:

Semantic accuracy
Structural accuracy
Geometric accuracy
Visual similarity

Headless rendering is useful, but visual comparison should not be the
only validation mechanism.

## Technology
Current experiments include:

Gemini
Antigravity
Mermaid
draw.io
Headless draw.io
Structured intermediate representations
Automated rendering and validation

## Status

Experimental

The current implementation is still being redesigned around the
intermediate-representation approach.

The main focus is moving from an image-to-format generation pipeline
towards a more explicit perception → representation → layout →
rendering architecture.

## Related Writing
What a Diagram Reconstruction Experiment Taught Me About AI Engineering

