# Porting LLaMA Factory to HPE Private Cloud AI (PCAI)

## About LLaMA Factory

LLaMA Factory is an open-source, unified framework for efficiently fine-tuning over 100 large language models (LLMs) and vision-language models (VLMs). Presented at ACL 2024, it is one of the most popular LLM fine-tuning tools in the community with 67.4k [GitHub](https://github.com/hiyouga/LlamaFactory/tree/main) stars (19 Feb 2026).

### Key Features

- **Zero-code fine-tuning** via a Gradio Web UI (LLaMA Board) on port 7860
- **OpenAI-compatible API server** on port 8000 for inference
- **100+ model support**: LLaMA, Qwen, DeepSeek, Gemma, Mistral, Phi, GLM, and many more
- **Diverse training methods**: full fine-tuning, LoRA, QLoRA (2/3/4/5/6/8-bit), DPO, PPO, KTO, ORPO
- **Advanced optimizations**: FlashAttention-2, Unsloth, DeepSpeed, FSDP, GaLore, vLLM/SGLang inference
- **Multi-modal**: image, video, and audio understanding tasks
- **Docker image**: `hiyouga/llamafactory:latest` (Ubuntu 22.04, CUDA 12.4, PyTorch 2.6.0)


## What Was Built

Since no upstream Helm chart exists for LLaMA Factory, a **brand-new Helm chart** was created from scratch and adapted for the PCAI Import Framework. The chart name is `llamafactory` and the chart version is `0.1.0` while the software version is `0.9.2`.


## Notes:

### 1. Shared Memory Volume (`/dev/shm`) 

PyTorch DataLoader workers and NCCL multi-GPU communication require a large `/dev/shm`. Docker defaults to 64MB which is insufficient. The upstream `docker-compose.yaml` uses `--ipc=host` and `--shm-size 16G`.

**Solution:** Mounted an `emptyDir` with `medium: Memory` and a configurable `shmSizeGi` (default 16Gi). This provides the same effect as `--shm-size` without requiring `hostIPC`.

### 2. Persistent Volume Claims (3x PVCs) 

LLM fine-tuning involves large model downloads (tens to hundreds of GB), custom datasets, and training output checkpoints. Without persistence, all data is lost on pod restart.

**Solution:** Three optional PVCs (enabled by default; set `persistence.<n>.enabled: false` to fall back to `emptyDir` on clusters without a dynamic StorageClass):
- `hf-cache` (200Gi) → `/root/.cache/huggingface` — model weight downloads
- `data` (50Gi) → `/app/custom_data` — custom training datasets
- `output` (100Gi) → `/app/output` — checkpoints and exported models

All three PVCs use Helm `pre-install` hooks so they are created before the main deployment. However, Helm only waits for **Jobs** to complete — not PVCs. So a small hook Job (`pvc-wait-job.yaml`) runs after the PVCs are submitted (hook-weight `0` vs `-5`) and sleeps for 30 seconds, giving the storage provisioner time to bind the volumes. Only after the Job finishes does Helm deploy the pod. The `helm.sh/resource-policy: keep` annotation on the PVCs ensures they survive an uninstallation to prevent data loss.

### 3. Startup Probe with High Failure Threshold

LlamaFactory's Gradio UI can take several minutes to fully initialise, especially if loading model metadata on first start. A default K8s liveness probe would kill the pod before it's ready.

**Solution:** `startupProbe` with `failureThreshold: 60` and `periodSeconds: 10` allows up to 10 minutes for initial startup before Kubernetes declares the pod unhealthy.

### 4. Configurable Mode (`webui` vs `api`) 

LlamaFactory supports two primary modes — the Gradio Web UI for interactive fine-tuning, and an OpenAI-compatible API server for inference. Making this configurable lets operators choose the mode at deploy time.

**Solution:** `values.mode` toggles the container command between `llamafactory-cli webui` and `llamafactory-cli api`, and switches the container port accordingly.


## Credentials & Login

- **LlamaFactory's Gradio WebUI does NOT have built-in authentication.** Anyone with network access to the endpoint can use the fine-tuning interface.
- On PCAI, access is implicitly restricted to authenticated platform users who can reach the Tools & Frameworks endpoint. However, there is no per-user login screen within LlamaFactory itself.
- If additional security is needed, operators can set `GRADIO_AUTH` environment variables (see Gradio docs) via `extraEnv` in `values.yaml` to add basic username/password protection.
- **HuggingFace Token**: To download gated models (e.g., LLaMA, Gemma), set `huggingface.token` in `values.yaml` at deploy time. The chart stores it as a Kubernetes Secret and injects it as `HF_TOKEN`. Alternatively, reference a pre-existing secret with `huggingface.existingSecret` and `huggingface.existingSecretKey`.


## Deployment Steps (Summary)

1. In PCAI console → **Tools & Frameworks** → **Import Framework**
2. Upload `llamafactory-0.1.0.tgz`, select a namespace
3. In the values editor, set `ezua.virtualService.endpoint` to `llamafactory.<your-pcai-domain>`
4. Adjust GPU/memory resources and PVC sizes as needed
5. Click **Deploy**
6. Open the application tile to access the LLaMA Board WebUI

