# IPEX-LLM on Aurora (Intel GPU)

**Date**: 2026-04-30
**Platform**: Aurora (Intel GPU Max / Ponte Vecchio, XPU)
**Conda env**: `ipex-llm` (Python 3.10)

---

## Installed packages

| Package | Version | Source |
|---|---|---|
| `torch` | 2.8.0a0+gitba56102 | Aurora frameworks wheelhouse |
| `intel-extension-for-pytorch` | 2.8.10+git09505bb | Aurora frameworks wheelhouse |
| `pytorch-triton-xpu` | 3.4.0+gitae324eea | Aurora frameworks wheelhouse |
| `ipex-llm` | 2.2.0 | PyPI (`--no-deps`) |
| `transformers` | 4.37.0 | PyPI |
| `accelerate` | 0.23.0 | PyPI |
| `tokenizers` | 0.15.2 | PyPI |
| `sentencepiece` | 0.2.1 | PyPI |

---

## Installation procedure

```bash
# 1. Create conda env with Python 3.10 (matches Aurora prebuilt wheels)
conda create -n ipex-llm python=3.10 -y

# 2. Install Aurora-optimized PyTorch + IPEX (from node-local wheelhouse)
WHEELHOUSE=/opt/aurora/25.190.0/frameworks/aurora_frameworks-2025.2.0/wheelhouse
pip install \
    ${WHEELHOUSE}/torch-2.8.0a0+gitba56102-cp310-cp310-linux_x86_64.whl \
    ${WHEELHOUSE}/intel_extension_for_pytorch-2.8.10+git09505bb-cp310-cp310-linux_x86_64.whl \
    ${WHEELHOUSE}/pytorch_triton_xpu-3.4.0+gitae324eea-cp310-cp310-manylinux_2_27_x86_64.manylinux_2_28_x86_64.whl \
    --find-links=${WHEELHOUSE}

# 3. Install ipex-llm without its bundled torch/ipex (they'd downgrade to 2.1.10)
pip install ipex-llm --no-deps

# 4. Install ipex-llm's non-torch dependencies
pip install \
    transformers==4.37.0 \
    sentencepiece \
    tokenizers==0.15.2 \
    accelerate==0.23.0 \
    tabulate
```

**Why `--no-deps` for ipex-llm**: `ipex-llm[xpu]` pins `torch==2.1.0a0` and
`intel-extension-for-pytorch==2.1.10+xpu`, which would downgrade the
Aurora-optimized 2.8.10 build. Using `--no-deps` keeps the newer Aurora wheels.
The `ipex-llm` 2.2.0 library itself is PyTorch-version-agnostic.

**Proxy required** (on Aurora login/compute nodes):
```bash
export http_proxy=http://proxy.alcf.anl.gov:3128
export https_proxy=http://proxy.alcf.anl.gov:3128
```

---

## Verification

```python
import torch
import intel_extension_for_pytorch as ipex
import ipex_llm

print('torch:', torch.__version__)    # 2.8.0a0+gitba56102
print('ipex:', ipex.__version__)      # 2.8.10+git09505bb
# XPU only available on compute nodes with GPU allocation
print('XPU available:', torch.xpu.is_available())
```

XPU returns `False` on login nodes (no GPU allocation). On a `gpu_hack_large`
compute node with `select=1:ngpus=6`, `torch.xpu.is_available()` should return `True`.

---

## Usage example (on compute node)

```python
from ipex_llm.transformers import AutoModelForCausalLM
from transformers import AutoTokenizer

model_path = "meta-llama/Llama-2-7b-chat-hf"  # or local path

# Load with INT4 quantization for Intel GPU
model = AutoModelForCausalLM.from_pretrained(
    model_path,
    load_in_4bit=True,
    optimize_model=True,
    trust_remote_code=True,
)
model = model.to('xpu')

tokenizer = AutoTokenizer.from_pretrained(model_path)
inputs = tokenizer("Tell me about neural circuits:", return_tensors="pt").to('xpu')
outputs = model.generate(**inputs, max_new_tokens=100)
print(tokenizer.decode(outputs[0]))
```

---

## Notes

- **Python 3.10 required**: Aurora wheelhouse wheels are built for `cp310` only
- **Benign warning**: IPEX overrides some XPU operators in PyTorch (expected)
- **transformers 4.37.0**: pinned by ipex-llm 2.2.0; may need updating for newer models
- **HuggingFace models**: see `concept-clio-core-gpu-memory.md` and `matsci_models_huggingface.md`
  for Aurora-relevant materials science LLMs

---

## Related pages

- [clio-core CTE: GPU memory vs CPU DRAM](concept-clio-core-gpu-memory.md)
- [clio-core CTE buffering for MiV optimization](perf-clio-core.md)
