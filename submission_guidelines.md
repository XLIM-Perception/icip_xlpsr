# [ICIP-XLPSR Challenge Docker Submission Guide](https://xlim-perception.github.io/icip_xlpsr/)


This document explains how competitors should extend the provided **base Docker image** for the challenge and how the evaluation system will execute your submission.

> ⚠️ **Important**: The base system, CUDA version, OS, and core environment **must not be modified**. You are only allowed to install additional Python/system packages required for your method.

---

## 🐳 Base Image Policy

The provided base image already contains:
- Ubuntu 22.04
- CUDA 12.4.1 + cuDNN 9 (GPU support)
- Python 3.10
- Core ML & vision libraries (PyTorch 2.10, OpenCV, NumPy, etc.)
- Non-root execution user
- Predefined directories: `/input`, `/output`, `/workspace`

### ❌ You must NOT:
- Change the base image (`FROM ...` line)
- Change CUDA version
- Change OS version
- Switch to a different CUDA image
- Add system daemons (ssh, cron, services, etc.)
- Change user permissions or revert to root

Any submission violating this will be **disqualified**.

---

## ✅ How to  Install Additional Packages and   Incorporate Your Solution  

You may install **only the packages needed for your method** under te `INSTALL SPECIFIC PACKAGES` section of the Dockerfile. Make sure to indicate the version of the packages being installed.

### Python packages (recommended)
Use `pip`:

```dockerfile
# ---------- INSTALL SPECIFIC PACKAGES ----------
RUN pip install --no-cache-dir \
    timm \
    pytorch-lightning \
    segmentation-models-pytorch
```

Or via `requirements.txt`:

```dockerfile
COPY requirements.txt /workspace/requirements.txt
RUN pip install --no-cache-dir -r /workspace/requirements.txt
```

---

### System packages (only if strictly necessary)

```dockerfile
# ---------- INSTALL SPECIFIC PACKAGES ----------
RUN apt-get update && apt-get install -y --no-install-recommends \
    libgeos-dev \
    libgdal-dev \
    && rm -rf /var/lib/apt/lists/*
```

⚠️ Avoid unnecessary system packages

---

### Incorporate Your Solution

Instructions to copy your solution **must be under te `COPY YOUR SOLUTION`** section of the Dockerfile. 
Copy your source and weights into `/workspace/`  and the  `run.sh` entrypoint

```dockerfile

# ---------- COPY YOUR SOLUTION ----------
COPY  src/    /workspace/
COPY  run.sh  /workspace/
```


---

## Directory Structure and Entry Point (MANDATORY)

Your container will have the following structure:

```
/input      # Read-only input data
/output     # Write-only output predictions
/workspace  # Your code
```

You **must not** write outside `/output`.

---

### Entry Point

Your container must provide a script that runs the full pipeline automatically and process all the inputs in /input

```
/workspace/run.sh
```

This script should:
1. Load input data from `/input`
2. Run inference
3. Write outputs to `/output`

---

### Input Interface

During evaluation, the system will mount the `/input` directory. It will contain a `csv` file listing all the available sequences, and the sequences in separate folders. Each folder will contain the frames for each sequence (in `png` format). 

```
/input
```

Example structure:
```
/input
  ├── all_sequences.csv
  │
  ├── seq_001/
  │     ├── img_01.png
  │     ├── img_02.png
  │     ├── img_03.png
  │     └── ...
  │
  ├── seq_002/
  │     ├── img_01.png
  │     ├── img_02.png
  │     ├── img_03.png
  │     └── ...
  │
  └── seq_003/
        ├── img_01.png
        ├── img_02.png
        ├── img_03.png
        └── ...

```

Your code must **only read** from this directory.

---

### Output Interface

Your container must write all results to the `/output` directory. One folder (with the same name as the input) per sequence. Each output folder must contain a super-resolved image `sr.png` and the recognized `text.txt`.

```
/output
```

Example:
```
/output
  ├── seq_001/
  │     ├── sr.png
  │     └── text.txt
  │
  ├── seq_002/
  │     ├── sr.png
  │     └── text.txt
  │
  └── seq_003/
        ├── sr.png
        └── text.txt
```

Only files written in `/output` will be collected and scored.

---

## 🚀 Evaluation Execution Model

Your container will be executed as follows (conceptually):

```bash
docker run \
  --rm \
  --gpus '"device=0"' \
  --network=none \
  --read-only \
  --cap-drop=ALL \
  --security-opt=no-new-privileges \
  --pids-limit=256 \
  --cpus=4 \
  --memory=30g \
  -v input:/input:ro \
  -v output:/output:rw \
  your_image:tag \
  bash /workspace/run.sh
```

⚠️ A common issue is that the `input` and `output` directories in the host computer do not have the necessary permissions for the container to access them. The simplest way to fix this is by setting the permissions of both directories (and subdirectories) to 777 (ie.  `chmod +R 777   input_folder`)


### Security
- No internet access
- No root privileges
- No access to host system
- No persistent storage
- No inter-container communication

---

### Resource Limits

Each submission will be executed with strict limits:
- CPU: **8 cores**
- RAM: **30GB**
- GPU: **RTX 3090 - 24GB**
- Time: max runtime per submission: **1 minute per image**

Submissions exceeding limits will be **terminated and scored as failed**.

---

## ❗ Rules Summary

✔ Install extra Python/system packages if needed  
✔ Use `/input` for reading data  
✔ Use `/output` for writing results  
✔ Provide a single executable pipeline in `/workspace/run.sh`

❌ Do not modify base image  
❌ Do not change CUDA or OS  
❌ Do not use network  
❌ Do not run background services  
❌ Do not write outside `/output`  

---



## 📦 Submission Packaging & Delivery


Participants must provide a **reproducible container submission**.


### Mandatory delivery contents
Each submission must include:

1. **Docker image** (exported archive)
2. **Dockerfile** used to build the image
3. **All files required to rebuild the image**, including:
- `requirements.txt`
- source code (`src`)
- scripts (`run.sh`, etc.)
- model weights **(they will not be downloaded)**


---


### 💾 How to Save Your Docker Image


After building your image locally:


```bash
docker build -t my_submission:latest .
```


Save the image to a file:


```bash
docker save my_submission:latest -o my_submission_image.tar
```


This file (`my_submission_image.tar`) is the **official image artifact**.


---


### Submission Format


Your final submission package must contain the following files, the **submission should be uploaded using the personalized invitation link provided by the organizers** 


```
submission/
├── my_submission_image.tar # Docker image archive
├── Dockerfile # Build recipe
├── run.sh # Entry point
├── requirements.txt # (if used)
├── src/ # Your code
└── README.md # (optional description)
```


---


### Reproducibility Requirement


The organizers must be able to regenerate your image using:


```bash
docker build -t reproduced_image .
```


with no missing dependencies and no external manual steps.


Submissions that are **not reproducible** will be rejected.



---



### Local Testing


Participants are encouraged to test locally the submission:

```bash
docker build -t my_submission .

docker run --rm \
  --gpus all \
  -v ./test_input:/input:ro \
  -v ./test_output:/output:rw \
  my_submission \
  bash /workspace/run.sh
```

---

## Final Notes

This challenge uses a **sandboxed black-box evaluation model**:
- Your container is treated as an isolated executable
- Only input/output behavior matters
- Internal implementation is irrelevant to scoring

Design for **determinism, reproducibility, and robustness**.
