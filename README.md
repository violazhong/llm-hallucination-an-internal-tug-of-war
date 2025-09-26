---
layout: default
---

# LLM Hallucinations: An Internal Tug of War

## TL;DR

I mechanistically investigated a surprising "confidence bias" in Llama-2-13b-chat. My results reveal an internal "tug of war": the model has a strong, effective circuit for representing user uncertainty, but this is frequently overridden by a more powerful, systemic bias to act confidently. This provides a direct, causal explanation for the "incentivized guessing" phenomenon described by OpenAI, suggesting confident hallucination is not a failure to know, but a policy failure where a default to confidence overpowers a more truthful internal state.

## Table of Contents

- [LLM Hallucinations: An Internal Tug of War](#llm-hallucinations-an-internal-tug-of-war)
  - [TL;DR](#tldr)
  - [Table of Contents](#table-of-contents)
  - [The Hunt for a Ghost](#the-hunt-for-a-ghost)
  - [Part 1: The Unexpected Clue - A 40% Failure Rate](#part-1-the-unexpected-clue---a-40-failure-rate)
  - [Part 2: The Investigation - A Temporary Brain Surgery](#part-2-the-investigation---a-temporary-brain-surgery)
  - [Part 3: The "Aha!" Moment - A Shared, Biased Circuit](#part-3-the-aha-moment---a-shared-biased-circuit)
  - [Part 4: Connecting the Dots - The Internal Tug of War](#part-4-connecting-the-dots---the-internal-tug-of-war)
  - [Part 5: The Hunt for a Stable Machine](#part-5-the-hunt-for-a-stable-machine)
  - [The Bigger Picture](#the-bigger-picture)
  - [A Final Thought: The Accidental Detective](#a-final-thought-the-accidental-detective)

## The Hunt for a Ghost

We all know the feeling. You're talking to a chatbot, and it states a complete falsehood with the unwavering confidence of a seasoned expert. This confident hallucination is one of the most critical and dangerous failure modes in modern AI. The big question is, why does it happen?

Concurrent work from OpenAI recently provided a powerful clue: they found that models are effectively trained with incentives that reward plausible guessing over admitting uncertainty. But this is a behavioral explanation. It tells us what the model is optimized for, but not how the machine is physically configured to execute this flawed policy.

I didn't set out to find the ghost in the machine. My project began as a simple investigation into user modeling. But I stumbled upon a bizarre clue—a strange asymmetry in the model's behavior—that led me down a rabbit hole and, I believe, to the doorstep of the mechanism itself. This is the story of that investigation.

## Part 1: The Unexpected Clue - A 40% Failure Rate

My initial goal was to understand how Llama-2-13b-chat-hf adapts to a user's epistemic status. I created a simple experiment: give the model a series of prompts where the user is either "certain" or "uncertain" about a fact, and see if the model mirrors their style.

The results were not what I expected. The model was near-perfect at mirroring certainty, adopting a declarative style 97.5% of the time. But when the user was uncertain, the model only adopted a tentative style 57.5% of the time. In the other 42.5% of cases, it defaulted back to a confident, declarative response.

![Figure 1: Bar chart showing the 97.5% vs 57.5% behavioral asymmetry](assets/llama_exp1.png)

This was the puzzle. This wasn't just noise; it was a statistically significant (p < 0.0001) and highly asymmetric failure. The model wasn't just bad at handling uncertainty; it seemed to have a powerful, intrinsic bias pulling it back towards confidence. Why?

## Part 2: The Investigation - A Temporary Brain Surgery

To find the cause, I needed to look inside the machine. I used a technique called activation patching, which is like performing a temporary, precise brain surgery on the model. It allows us to ask causal questions: "Which part of the model is responsible for this behavior?"

I ran a comparative analysis:

1. **Experiment 2A (The "Certainty Circuit")**: I tried to "restore" a confident style by patching activations from certain prompts into a run on uncertain prompts.
2. **Experiment 2B (The "Uncertainty Circuit")**: I tried to "induce" a tentative style by patching activations from uncertain prompts into a run on certain prompts.

My initial hypothesis was simple: I'd find two different circuits, one for certainty and one for uncertainty, and the certainty one would just be stronger.

I was wrong.

## Part 3: The "Aha!" Moment - A Shared, Biased Circuit

The results were shocking. The "Certainty Circuit" and the "Uncertainty Circuit" were in the exact same place. Both behaviors were driven by a single, shared block of layers in the latter half of the model (layers 20-39).

![Figure 2 & 3: Side-by-side heatmaps from Experiment 2A and 2B, showing the shared locus in layers 20-39.](assets/llama_exp2.png)

This finding was massively validated when I realized this locus (layers 20-39) almost perfectly overlapped with the user-modeling region (layers 20-30) that Chen et al. had independently identified in the same model. This convergence of evidence from different methods and different tasks was a huge signal that we had tapped into a real, fundamental part of the Llama-2 architecture: a central "Confidence Hub."

But this hub was biased. While the "Certainty" pathway was perfect (1.000 restoration score), the "Uncertainty" pathway was measurably weaker (~0.909 score). This provided the direct, causal explanation for the behavioral asymmetry.

## Part 4: Connecting the Dots - The Internal Tug of War

This is where the deepest insight lies. Why would a strong internal signal for uncertainty (~0.909) lead to such a weak behavioral outcome (57.5%)?

It suggests an internal tug of war.

1. One part of the circuit successfully represents the user's uncertainty. It "knows" it should be tentative. The strong patching score is evidence of this.
2. But another, more powerful force—the model's default policy of being a confident assistant, likely installed during RLHF—is constantly pulling in the other direction.

The 57.5% behavioral result is the outcome of this conflict. The mechanism for representing uncertainty wins the tug-of-war just over half the time, but it's frequently overpowered by the model's intrinsic bias to sound confident.

This provides the missing mechanistic piece to the OpenAI puzzle. The "incentivized guessing" they described is not just a high-level policy; it is a physical conflict between circuits in the model. Hallucination is what happens when the "Default Confidence" side of the rope wins the tug-of-war, dragging a state of internal uncertainty into a statement of external confidence.

## Part 5: The Hunt for a Stable Machine

This project was not a straight line. The journey to get these results was a two-day sprint through a minefield of technical failures. It required pivoting from Colab (memory crashes), to RunPod (deep infrastructure issues), to Lambda. It required abandoning high-level libraries like TransformerLens and nnsight when they failed and dropping down to a raw PyTorch implementation to maintain control. This struggle wasn't a distraction; it was a reminder that research is often a gritty, pragmatic fight to get a clean signal from an uncooperative universe.

## The Bigger Picture

This work suggests that confident hallucination is not a mystical, distributed failure. It is the predictable, systematic output of a specific, biased, and now, locatable circuit. This reframes the problem from a vague statistical issue into a tangible engineering one. We now have a target.

The next step, I think is finding evidences that can disconfirm this hypothesis. On one hand, I believe this experiment results and this hypothesus are worth serious consideration, they might be true, on the other hand, we should try our best to stick with objectivity until we're fully convinced we find something real.

## A Final Thought: The Accidental Detective

This project's journey from a simple user-modeling query to a deep dive into hallucination's roots feels like a full circle for me. My initial fascination with this entire domain began with a simple, accidental discovery.

There was a time when I liked to ask LLMs to tell me something that I didn't know. Casually, I started noticing a pattern: the models, just like Sherlock Holmes, were able to efficiently infer a user's demographic, psychological, and even cognitive traits from conversations that were not directly related to those topics. It was this startling, almost magical ability of the model to "read" its user that first convinced me that a rich, complex user model must exist within the machine.

Finding the "Confidence Hub" in this project feels like finding the first real, tangible gear in that mysterious user model. And discovering that this same gear might be the engine behind hallucination reinforces the idea that the path to safer, more aligned AI runs directly through a deep, mechanistic understanding of the models themselves.
