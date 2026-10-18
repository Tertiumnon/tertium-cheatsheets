# AI Concepts

A glossary of terms that come up when working with large language models and other neural networks — model size and efficiency, training, inference, and the surrounding architecture. Aimed at engineers who consume or integrate AI systems rather than train them from scratch.

## Model Size & Efficiency

- **Parameters:** The learned weights of a model (e.g. a "7B model" has ~7 billion parameters). Roughly correlates with capability and cost, not linearly.
- **Quantization:** Reducing the numeric precision used to store a model's weights (and sometimes activations), trading a small amount of accuracy for large reductions in memory and inference cost. See below for detail.
- **Distillation:** Training a smaller "student" model to mimic the outputs of a larger "teacher" model, producing a cheaper model that retains much of the teacher's behavior.
- **Pruning:** Removing weights or whole neurons that contribute little to output, shrinking the model without full retraining.
- **Mixture of Experts (MoE):** An architecture where only a subset of the model's parameters ("experts") activate per input, giving a large total parameter count with a much smaller compute cost per token.

### Quantization in Detail

A model's weights are normally stored as 32-bit or 16-bit floats. Quantization maps them to a lower-precision representation:

| Format | Bits/weight | Relative size | Typical use |
|--------|-------------|----------------|-------------|
| FP32   | 32          | 1x (baseline)  | Training, reference precision |
| FP16 / BF16 | 16     | 0.5x           | Training and inference on GPUs |
| INT8   | 8           | 0.25x          | Inference, mild quality loss |
| INT4 (GPTQ, AWQ) | 4 | 0.125x       | Local/edge inference, larger quality loss |

Weights are quantized by mapping their float range to the target integer range, then dequantized (approximately) at compute time:

```python
# Simplified symmetric int8 quantization of a weight tensor
import numpy as np

def quantize_int8(weights: np.ndarray) -> tuple[np.ndarray, float]:
    scale = np.max(np.abs(weights)) / 127
    quantized = np.round(weights / scale).astype(np.int8)
    return quantized, scale

def dequantize(quantized: np.ndarray, scale: float) -> np.ndarray:
    return quantized.astype(np.float32) * scale

weights = np.array([0.12, -0.87, 0.45, -0.03])
q, scale = quantize_int8(weights)
restored = dequantize(q, scale)
# restored ≈ weights, with small rounding error proportional to scale
```

- **Post-Training Quantization (PTQ):** Quantize an already-trained model directly — fast, some accuracy loss.
- **Quantization-Aware Training (QAT):** Simulate quantization during training so the model adapts to the lower precision — better accuracy, requires retraining.
- **Why it matters:** A 70B-parameter model at FP16 needs ~140GB of memory; the same model at INT4 needs roughly ~35GB, which is the difference between "needs a multi-GPU server" and "runs on a single high-end GPU or high-end laptop."

## Training

- **Pre-training:** Training a model from scratch (or near-scratch) on a large, general corpus to learn language/world structure.
- **Fine-tuning:** Continuing training on a smaller, task-specific dataset to specialize a pre-trained model's behavior.
- **LoRA (Low-Rank Adaptation) / PEFT:** Fine-tuning by adding small trainable low-rank matrices instead of updating all parameters — far cheaper than full fine-tuning, and multiple LoRA adapters can be swapped on top of one base model.
- **RLHF (Reinforcement Learning from Human Feedback):** A post-training step where a model is optimized against human preference judgments (or a reward model trained on them), used to align outputs with what people actually want rather than just what's statistically likely.

## Inference & Generation

- **Tokenization:** Splitting text into subword units (tokens) that the model actually operates on — "tokenization" itself splits into `token` + `ization`, not necessarily whole words.
- **Context Window:** The maximum number of tokens (input + output combined) a model can attend to in a single request. Exceeding it means the oldest content is dropped or the request is rejected.
- **Temperature:** A sampling parameter controlling randomness — near 0 makes output near-deterministic (always the highest-probability token); higher values increase variety and risk of incoherence.
- **Top-p / Top-k Sampling:** Restrict token sampling to the smallest set of tokens whose cumulative probability exceeds `p` (top-p), or to the `k` most likely tokens (top-k), avoiding low-probability, off-topic completions.
- **KV Cache:** Cached key/value attention state from previously generated tokens, reused so each new token doesn't require recomputing attention over the entire prior sequence — the main reason autoregressive generation isn't quadratically slow per token.

## Architecture

- **Transformer:** The neural network architecture behind virtually all modern LLMs, built around self-attention rather than recurrence, which parallelizes across the input sequence during training.
- **Attention:** The mechanism that lets a model weigh how relevant every other token in the input is when computing the representation of a given token.
- **Embeddings:** Dense vector representations of tokens, words, or documents, positioned so that semantically similar items are close together in vector space — the basis for both a model's internal token representations and external semantic search.

## Retrieval & Agents

- **RAG (Retrieval-Augmented Generation):** Retrieving relevant external documents (typically via embedding similarity search) and inserting them into the prompt, so the model answers grounded in specific data instead of relying solely on what it memorized during training.
- **Hallucination:** A model generating fluent, confident output that is factually wrong or unsupported — a structural risk of next-token prediction, not a bug that a bigger model fully eliminates.
- **Prompt Engineering:** Structuring input (instructions, examples, format constraints) to reliably steer a model's output without changing its weights.
- **Agentic / Tool Use:** A model that can call external tools or functions (search, code execution, APIs) and incorporate their results into further reasoning, rather than producing a single one-shot text response.

## Related

- [Testing Strategy](../../testing/testing-strategy.md) — testing considerations for AI-generated code
- [Code Review Guidelines](../../best-practices/code-review/code-review.md) — reviewing AI-generated code
- [Spec-Driven Architecture](../../spec-driven-architecture/spec-driven-architecture.md) — constraining AI agents with specs before generation

## References & Sources

- Vaswani et al. — "Attention Is All You Need" (2017): https://arxiv.org/abs/1706.03762
- Hugging Face — Quantization: https://huggingface.co/docs/transformers/quantization
- Hu et al. — "LoRA: Low-Rank Adaptation of Large Language Models" (2021): https://arxiv.org/abs/2106.09685
- Ouyang et al. — "Training language models to follow instructions with human feedback" (RLHF, 2022): https://arxiv.org/abs/2203.02155
- Lewis et al. — "Retrieval-Augmented Generation for Knowledge-Intensive NLP Tasks" (2020): https://arxiv.org/abs/2005.11401
