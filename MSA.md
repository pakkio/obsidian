# MSA

> *A scalable, end-to-end trainable latent-memory framework for 100M-token contexts*

## Concetti
[[Python]]

## Panoramica
# MSA: Memory Sparse Attention
*A scalable, end-to-end trainable latent-memory framework for 100M-token contexts*

[**arXiv**](https://arxiv.org/abs/2603.23516) • [**Zenodo**](https://zenodo.org/records/19103670)

[![Models](https://img.shields.io/badge/HuggingFace-MSA--4B-yellow?logo=huggingface)](https://huggingface.co/EverMind-AI/MSA-4B) [![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT) [![EverMind](https://img.shields.io/badge/EverMind-Organization-blue?logo=github)](https://github.com/EverMind-AI) [![EverOS](https://img.shields.io/badge/EverOS-Repo-green?logo=github)](https://github.com/EverMind-AI/everos)

[中文](./README_ZH.md)

## 📝 Abstract

Long-term memory is essential for general intelligence, yet the **full attention** bottleneck constrains most LLMs' **effective context length** to **128K–1M**. Existing attempts，hybrid linear attention, fixed-size state memory (e.g., RNNs), and external storage like **RAG/agents**，either suffer rapid precision decay and latency growth at extreme scales, lack end-to-end differentiability or dynamic memory maintenance, or require complex pipelines. We present **Memory Sparse Attention (MSA)**: an **end-to-end trainable, scalable sparse** **latent-state memory** framework. Core ideas include:

- **Scalable sparse attention** + **document-wise RoPE** (parallel/global) achieving **near-linear complexity** in both training and inference;
- **KV cache compression** with a **Memory Parallel** inference engine to deliver **100M token** throughput on **2×A800** GPUs;
- **Memory Interleave** for multi-round, multi-hop reasoning across scattered memory segments.

On long-context QA and NIAH (Needle-in-a-Haystack) benchmarks, **MSA** surpasses same-backbone RAG, best-of-breed RAG stacks, and leading long-context models. Across an unprecedented **16K→100M token** range, MSA shows **< 9%** degradation, suggesting a practical path to **decouple memory capacity from reasoning**.

> **Scaling from 16K→100M tokens**: MSA fuses top-k selection with sparse attention to remain end-to-end differentiable while allowing document decoupling at inference. On MS MARCO, MSA sustains **<9%** degradation and exhibits strong extrapolation.
> *Some baseline curves end early due to their context limits.*

![Figure 1: Scaling curve 16K→100M tokens](./assets/fig1_scaling.png)
*Figure 1: MSA scalability under extreme-long contexts*

---

## ✨ Key Contributions

- **Memory-Sparse Attention (MSA)**: an **end-to-end trainable**, **scalable sparse attention** layer with **document-wise RoPE**, realizing **O(L)** complexity and **<9%** degradation from **16K→100M tokens**.
- **KV cache compression + Memory Parallel**: tiered storage (GPU-resident routing keys, CPU content K/V), distributed scoring, and on-demand transfers to enable **100M-token** inference on **2×A800**.
- **Memory Interleave**: adaptive alternating "generative retrieval → context expansion → generation," significantly boosting **multi-hop** reasoning across documents.
- **Comprehensive evaluation**: MSA outperforms same-backbone RAG, best-of-breed RAG pipelines, and top long-context models on long-context QA and NIAH, showing superior **stability** and **accuracy** at scale.

---

## 🧩 Overall Design

### Architecture

MSA integrates **retrieval and generation** into a single differentiable loop. Document latent states (**K/V/Kᵣ**) are **chunk-mean pooled** for compression. A **router projector** computes relevance via cosine similarity (mean-pooled over heads, then token-wise max), selects **Top‑k documents**, then concatenates their **compressed K/V** with the query's local **K/V** for autoregressive decoding. Routing applies only to **upper layers**; lower layers keep **independent document processing** for hierarchical alignment.

- **Parallel (document-wise) RoPE**: Each document resets positions from 0, preventing position drift between **train-short** and **infer-long**, enabling 64k trainin

## Tech Stack
- Linguaggi: Python (28)
- File totali (scan): 89
- Estensioni principali: .py=28, (noext)=26, .json=8, .png=5, .md=4, .sh=3, .bat=3, .txt=2
- Remote: `https://github.com/EverMind-AI/MSA.git`

## Struttura (top-level)
```
QUICK_START.md
README.md
README_ZH.md
assets
ckpt
paper
requirements.txt
scripts
src
test_smoke.py
```

## Documentazione presente nel repo
- `QUICK_START.md`
- `README.md`
- `README_ZH.md`
- `ckpt/MSA-4B/README.md`

## Manifest
### requirements.txt
```
tqdm
openai
torch==2.6
torchvision==0.21.0
transformers==4.51.3
liger_kernel==0.5.10
accelerate==1.0.1
```

## Progetti correlati
_nessuno_

## Link
- Repository: https://github.com/EverMind-AI/MSA.git
- Path locale: `/home/pakkio/w/MSA`
