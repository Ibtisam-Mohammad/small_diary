---
title: "Running Fara-7B on RTX 3060"
tags:
  - ML
  - Windows
  - WSL
  - Fara-7B
---

# How to Run Microsoft Fara-7B Agent on Windows (RTX 3060)

In this guide, I document how I ran Microsoft's [Fara-7B](https://github.com/microsoft/fara) computer-use agent on a my strange PC config. The issue was running a Vision-Language Model on a 12GB VRAM card (RTX 3060) and a my Pentium CPU (Intel Pentium Gold :/). I did it just as a trial run to see the potential of such models locally.

## 🛠 The Hardware Setup

* **GPU:** NVIDIA RTX 3060 (12GB VRAM)
* **CPU:** Intel Pentium Gold (Too slow for compiling or CPU inference)
* **OS:** Windows 10/11 with WSL2 (Ubuntu)

## 📦 The Strategy

Had to load the model and llama.cpp on separate os':

1. **Server:** `llama.cpp` running natively on **Windows** to fully utilize the GPU (CUDA) for inference.
2. **Client:** The `fara` Python agent running in **WSL (Linux)** to handle browser automation (Playwright).

---

## Step 1: Prepare the Model (Quantization)

The original Fara-7B model (~15GB) is too big for a 12GB card. I used **GGUF** quantized versions to fit it into VRAM.

**Repositories Used:**

* **Model:** [`mradermacher/Fara-7B-GGUF`](https://huggingface.co/mradermacher/Fara-7B-GGUF) (I targeted `Q6_K` for quality or `Q4_K_M` for speed).
* **Vision Projector:** A compatible `.mmproj` file (required for the AI to "see").

**Downloads (via WSL):**

```bash
# Create a folder on your Windows drive (mounted in WSL)
mkdir -p /mnt/i/AI/fara/model_weights
cd /mnt/i/AI/fara/model_weights

# Install downloader
pip install huggingface_hub

# Download Model (Q6_K recommended, ~6.4GB)
huggingface-cli download mradermacher/Fara-7B-GGUF Fara-7B.Q6_K.gguf --local-dir .

# Download Vision Projector (Critical for screenshots)
huggingface-cli download mradermacher/Fara-7B-GGUF Fara-7B.mmproj-f16.gguf --local-dir .
```

---

## Step 2: Set Up the AI Server (Windows)

I used the pre-built **Windows CUDA** binaries for `llama.cpp` to avoid 4+ hour compilation times on the Pentium CPU.

**1. Download Binaries:**
From [llama.cpp releases](https://github.com/ggml-org/llama.cpp/releases), I needed **two** zip files extracted into the same folder (e.g., `I:\AI\llama`):

1. `llama-bXXXX-bin-win-cuda-12.4-x64.zip` (The executables like `llama-server.exe`)
2. `cudart-llama-bin-win-cuda-12.4-x64.zip` (The DLLs like `cublas64_12.dll`)

**2. The Launch Command:**
Run this in **PowerShell**. Note the **`-c 8192`** flag, which was critical to prevent "Context Limit" crashes during web browsing.

```powershell
# Navigate to your llama folder
cd I:\AI\llama-b7170-bin-win-cuda-12.4-x64

# Start the server
.\llama-server.exe `
  -m "I:\AI\fara\model_weights\Fara-7B.Q6_K.gguf" `
  --mmproj "I:\AI\fara\model_weights\Fara-7B.mmproj-f16.gguf" `
  --port 8081 `
  --host 0.0.0.0 `
  -ngl 99 `
  -c 8192
```

---

## Step 3: Set Up the Agent Client (WSL)

The Fara agent code runs best in Linux.

**1. Installation:**

```bash
git clone https://github.com/microsoft/fara.git
cd fara
python3 -m venv .venv
source .venv/bin/activate

# Install package
pip install -e .

# Install Browser Engine
playwright install
sudo playwright install-deps  # Critical for WSL!
```

**2. Networking Fix (Connecting WSL to Windows):**
Since WSL is a virtual machine, it cannot access `localhost` of the Windows host directly (unless using Mirrored mode). I found the Windows IP:

```bash
# Run inside WSL to find Windows Host IP
ip route show | grep -i default | awk '{ print $3}'
# Output example: 172.25.160.1
```

**3. Agent Configuration:**
Created `endpoint_configs/windows_server.json`:

```json
{
  "model": "Fara-7B",
  "base_url": "http://172.25.160.1:8081/v1",
  "api_key": "sk-placeholder",
  "model_type": "chat",
  "encoding_format": "base64"
}
```

---

## Step 4: Execution

With the Server running in PowerShell, I launched the Agent in WSL:

```bash
# Using fara-cli (installed by pip install -e .)
fara-cli \
  --task "Go to bing.com and find the population of Canada" \
  --start_page "https://www.bing.com" \
  --endpoint_config endpoint_configs/windows_server.json
```

It worked and was using my browser. The speed was fine. But need to check the accuracy of the results.
