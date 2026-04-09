---
name: gpu-ml-tools
description: GPU monitoring and ML environment quick reference
tags: [gpu, cuda, python, ml, local]
---

# GPU & ML Tools

## GPU Monitoring

| Command | Action |
|---------|--------|
| `gpu` | `nvidia-smi` (one-shot) |
| `gpuw` | `watch -n 1 nvidia-smi` (live) |

## Python Environment

| Alias | Expands To |
|-------|-----------|
| `py` | `python` |
| `ipy` | `ipython` |
| `pip` | `uv pip` (fast package installer) |
| `mm` | `micromamba` |
| `sa` | `source activate` |

## Key Environment Variables

```bash
CUDA_PATH=/usr/local/cuda-12.6
HF_HOME=$DATA/../huggingface
PYTHONBREAKPOINT=ipdb.set_trace
IPYTHON_EDITING_MODE=vi
OLLAMA_HOST=127.0.0.1:11434
```

## Data Paths

| Variable | Path |
|----------|------|
| `$DATA` | `/mnt/d5179bee.../data` |
| `$HF_HOME` | Hugging Face cache |
| `$DOLT_ROOT_PATH` | `~/ollama/dolt-data` |

## Micromamba Quick Commands

```bash
mm create -n myenv python=3.11
mm activate myenv
mm install pytorch torchvision -c pytorch
mm env list
```

## Vast.ai GPU (via workspace-job)

```bash
vast search offers "gpu_name==RTX_5090 dph_total<=0.40"
vast create instance <offer_id> --image pytorch/pytorch:2.5.1-cuda12.8.1
vast ssh <instance_id>
```