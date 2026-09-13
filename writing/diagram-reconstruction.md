---
layout: default
title: What a Diagram Reconstruction Experiment Taught Me About AI Engineering
---

# What a Diagram Reconstruction Experiment Taught Me About AI Engineering

Recently, I worked on a small but surprisingly challenging project: turning diagram images into editable formats such as Mermaid and draw.io.

The goal sounded simple:

> Give an AI model an image of a diagram, and get back an editable diagram.

I used Gemini and Antigravity to experiment with the workflow. I also introduced smaller image regions, generated diagrams for those regions independently, merged the results, and used headless draw.io to validate the generated files.

It worked, at least partially.

The generated diagrams often captured the important nodes and connections. But when I opened the resulting draw.io file, the diagram frequently looked quite different from the original image.

More importantly, as I kept improving the implementation, the solution started to feel increasingly complicated and difficult to reason about.

That experience led me to a more interesting question:

> Was the implementation becoming messy because the code needed more refinement, or because I had chosen the wrong abstraction for the problem?

I think it was mostly the latter.

## The First Lesson: Understanding a Diagram Is Not the Same as Reconstructing It

The first surprising observation was that the AI could often understand the semantics of the diagram reasonably well.

For example, given a diagram like:

```text
        ┌─────────────┐
        │   Service   │
        └──────┬──────┘
               │
        ┌──────┴──────┐
        ↓             ↓
   ┌────────┐    ┌────────┐
   │   DB   │    │ Queue  │
   └────────┘    └────────┘
```

The model might correctly identify:

```text
Service → DB
Service → Queue
```

From a semantic perspective, this is a success.

But an editable diagram has another dimension: geometry.

The original diagram also contains information such as:

- where each node is positioned
- how far apart nodes are
- which nodes are aligned
- which elements are grouped together
- where containers begin and end
- how edges are routed
- where labels are placed
- what visual hierarchy exists

So there are really two different goals:

```text
Semantic fidelity
+
Visual fidelity
```

I initially treated them as one problem.

That was one of the fundamental mistakes.

---

# The Second Lesson: Image → Target Format Is Too Large an Abstraction

My initial mental model was approximately:

```text
Image
  ↓
Gemini
  ↓
draw.io / Mermaid
```

It is attractive because it is simple.

But the model is being asked to solve several different problems at once:

```text
Visual understanding
       +
Semantic extraction
       +
Graph reconstruction
       +
Layout reconstruction
       +
Style reconstruction
       +
Target-format generation
```

These are not the same problem.

In particular, generating valid draw.io XML is not the same thing as understanding the diagram.

Once I recognized this, the architecture became much clearer.

A better abstraction is:

```text
Image
  ↓
Visual / Semantic Analysis
  ↓
Intermediate Representation
  ↓
Layout / Reconstruction
  ↓
Target Format
```

The missing piece is the intermediate representation.

---

# The Importance of an Intermediate Representation

Instead of asking the model to directly generate draw.io XML, I would now have the model produce a diagram-specific intermediate representation.

For example:

```json
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

The important idea is that this representation is independent of draw.io.

It describes the diagram rather than the serialization format.

Then we can have:

```text
                 Diagram IR
                /    |     \
               /     |      \
          draw.io  Mermaid   SVG
```

This separation gives each component a much clearer responsibility.

The AI model can focus on understanding the diagram.

The layout engine can focus on geometry.

The renderer can focus on producing the target format.

The validation layer can focus on checking correctness.

This is much easier to reason about than having an agent manipulate XML directly.

---

# The Problem With Splitting the Image

One of the approaches I experimented with was splitting a large diagram into smaller regions.

This was motivated by a reasonable observation:

> A model may perform better when it doesn't have to understand a huge and complicated diagram in one pass.

So the workflow became something like:

```text
Large Image
    ↓
Split into regions
    ↓
Analyze each region
    ↓
Generate individual diagrams
    ↓
Validate with headless draw.io
    ↓
Merge them
```

This improved some local recognition problems.

But it introduced another problem: loss of global context.

Suppose the original diagram contains:

```text
                 System A
                    │
          ┌─────────┴─────────┐
          ↓                   ↓
      Service A           Service B
          │                   │
          └─────────┬─────────┘
                    ↓
                   DB
```

If I independently process different regions, each local result may be correct.

But when those results are merged, the global relationships may no longer be preserved.

This is an important distinction:

> Local correctness does not guarantee global correctness.

The problem is similar to distributed systems: each component can make a locally reasonable decision while the overall system becomes inconsistent.

---

# The Better Pattern: Global → Local → Global

I now think the right approach is not simply:

```text
Split → Process → Merge
```

but:

```text
Global analysis
      ↓
Region identification
      ↓
Local analysis
      ↓
Global reconciliation
```

The first global pass should establish the overall structure:

- diagram type
- major regions
- containers
- approximate layout
- major relationships
- global hierarchy

Then local analysis can focus on the details.

Finally, the local results should be reconciled back into a single global model.

The key difference is that we merge **semantic representations**, rather than merging already-generated diagrams.

In other words:

```text
Wrong abstraction:

small diagram A
small diagram B
small diagram C
        ↓
      merge
        ↓
large diagram
```

A better abstraction is:

```text
local semantic models
        ↓
global Diagram IR
        ↓
global layout
        ↓
one diagram
```

That distinction makes a significant difference.

---

# Another Lesson: Validation Should Be Layered

I also experimented with using headless draw.io as part of the validation process.

This was useful because it gave me a concrete way to render the generated diagram and inspect the result.

However, I eventually realized that "does the rendered image look correct?" is too coarse a validation question.

There are several different kinds of correctness.

## 1. Semantic validation

Are the nodes correct?

Are the labels correct?

Are the relationships correct?

For example:

```text
Expected nodes: 37
Generated nodes: 35
```

This is already a useful signal.

## 2. Structural validation

Are the containers correct?

Are the groups correct?

Are the hierarchy and cross-region relationships preserved?

## 3. Geometric validation

Are nodes approximately where they should be?

Are nodes aligned?

Are relative distances preserved?

For example:

```text
Original:

A -------- B

Generated:

A ------------------------------ B
```

The graph is technically correct, but the geometry is not.

## 4. Visual validation

Finally, render the generated diagram and compare it with the original image.

This is where headless draw.io becomes useful.

The important insight is:

> Rendering validation should be one layer of validation, not the entire validation strategy.

---

# Why My Implementation Started to Feel Messy

Looking back, the messiness was a symptom.

As problems appeared, it was tempting to add more special cases:

```text
if image_is_large:
    split()

if region_is_complex:
    split_again()

if validation_fails:
    regenerate()

if edge_is_wrong:
    repair_edge()

if nodes_overlap:
    move_nodes()

if rendering_is_different:
    ask_the_model_again()
```

Each individual change seemed reasonable.

But collectively, the system was turning into a collection of patches.

This usually happens when there is no strong intermediate abstraction.

Without an intermediate representation, every component starts communicating through implementation details:

```text
LLM → XML
XML → renderer
renderer → image
image → LLM
LLM → XML again
```

The system becomes difficult to reason about because there is no stable model of what the application actually believes the diagram is.

The diagram itself should be the first-class object.

---

# What Should the LLM Do?

This project also changed my thinking about agentic AI.

My initial instinct was to let the model do as much as possible.

But that is not necessarily the best architecture.

The model is well suited to tasks such as:

```text
Identify
Classify
Interpret
Infer
Explain
Resolve ambiguity
```

Deterministic code is better suited to:

```text
Calculate coordinates
Apply constraints
Merge models
Validate schemas
Serialize XML
Generate files
```

Specialized tools are better suited to:

```text
OCR
Rendering
Image processing
Format validation
```

So instead of:

```text
LLM does everything
```

the architecture should look more like:

```text
                 ┌──────────────┐
                 │ Vision Model │
                 └──────┬───────┘
                        ↓
                  Diagram IR
                        ↓
          ┌─────────────┴─────────────┐
          ↓                           ↓
   LLM reasoning              Deterministic code
          ↓                           ↓
 semantic interpretation       layout / validation
          └─────────────┬─────────────┘
                        ↓
                    Renderer
```

This is a much more useful mental model for building agentic systems.

---

# The Bigger AI Engineering Lesson

What looked initially like a small "AI conversion" problem turned out to be a good example of a broader engineering principle:

> LLMs are powerful reasoning components, but they should not automatically become the architecture.

An agent is not necessarily better because it performs more of the workflow.

The engineering challenge is deciding:

> Which parts of the problem benefit from probabilistic reasoning, and which parts should remain deterministic?

For this particular problem:

```text
Visual understanding       → AI
Semantic interpretation    → AI
Ambiguous relationships    → AI
Global reconciliation      → AI + deterministic checks
Geometry                   → deterministic algorithms
Layout                     → layout engine
Serialization              → deterministic code
Rendering                  → specialized tool
Validation                 → deterministic + AI
```

That separation makes the system both easier to debug and easier to improve.

---

# What I Would Build Next

If I restarted this project, I would not start by improving the prompt.

I would start by defining the Diagram IR.

Then I would build the system incrementally:

```text
Phase 1
Image → Diagram IR
```

Focus only on semantic correctness.

```text
Phase 2
Diagram IR → deterministic draw.io
```

Make serialization reliable.

```text
Phase 3
Image geometry → Diagram IR
```

Add bounding boxes, alignment and spatial relationships.

```text
Phase 4
Diagram IR → layout engine
```

Separate layout from AI reasoning.

```text
Phase 5
Global + local analysis
```

Introduce region-level processing without losing global context.

```text
Phase 6
Multi-level evaluation
```

Measure:

```text
semantic accuracy
structural accuracy
geometric accuracy
visual similarity
```

Only after these foundations are stable would I add more sophisticated agentic workflows.

---

# From "Make It Work" to "Make the System Understandable"

The most useful outcome of this experiment was not the diagram converter itself.

It was realizing why the implementation felt wrong.

The problem was not simply that Gemini sometimes generated bad diagrams.

The deeper issue was that I had initially treated a complex reconstruction problem as a single LLM generation task.

Once the problem is decomposed into:

```text
Perception
    ↓
Semantic model
    ↓
Structural model
    ↓
Geometry
    ↓
Layout
    ↓
Rendering
    ↓
Validation
```

the engineering boundaries become much clearer.

And that is probably the more transferable lesson.

When building AI systems, especially agentic systems, a useful question is not:

> "What can the model do?"

A better question is:

> "What should the model do, and what should the rest of the system do?"

That distinction is where a prototype starts becoming an engineering system.

...
