# Fixed-gaussian-splatting

A Windows-compatible adaptation of the original [graphdeco-inria/gaussian-splatting](https://github.com/graphdeco-inria/gaussian-splatting) repository.

Tested on: **RTX 4070 Super · Windows 11 · CUDA 11.8 · Python 3.10 · PyTorch 2.0.1**

---

## What was changed

Three files were patched from the original. Everything else is untouched upstream code.

### 1. `train.py` and `render.py` — Windows CUDA DLL fix

On Windows, the compiled CUDA extension modules (`diff_gaussian_rasterization`, `simple_knn`) cannot locate `cudart64_118.dll` through Python's default DLL search path. Added two lines before `import torch` in both files:

```python
import sys
os.add_dll_directory(os.path.join(sys.prefix, 'Lib', 'site-packages', 'torch', 'lib'))
```

`sys.prefix` resolves to the active conda environment root, so this works on any machine without hardcoding a username or install path.

### 2. `scene/dataset_readers.py` — NumPy dtype fix

Line ~259 used `dtype=np.byte` (signed int8) when constructing an RGB image array passed to PIL:

```python
# Before (broken):
image = Image.fromarray(np.array(arr*255.0, dtype=np.byte), "RGB")

# After (fixed):
image = Image.fromarray(np.array(arr*255.0, dtype=np.uint8), "RGB")
```

`PIL.Image.fromarray()` requires unsigned `uint8` for RGB images. The signed type causes a hard crash under NumPy 2.x:
```
TypeError: Cannot handle this data type: (1, 1, 3), |i1
```
This fix was also submitted as a pull request to the original repo: [graphdeco-inria/gaussian-splatting#1330](https://github.com/graphdeco-inria/gaussian-splatting/pull/1330).

---

## Difficulties faced during setup on Windows

The original repo was written for Linux and has no Windows setup guide. Every problem below was hit during actual installation.

| # | Error / Symptom | Root Cause | Fix |
|---|----------------|-----------|-----|
| 1 | `requirements.txt` not found | Repo ships no such file | Install all packages manually (see below) |
| 2 | `vcvars64.bat` had no effect on MSVC | PowerShell does not inherit environment variables set by `.bat` files | Use CMD, not PowerShell, for the extension build step |
| 3 | `No module named 'torch'` during CUDA extension build | pip's build isolation creates a clean env that can't see installed packages | Pass `--no-build-isolation` to pip |
| 4 | `No module named 'pkg_resources'` | setuptools 82.x dropped compatibility with the way PyTorch 2.0.1 calls it | Pin `setuptools==69.5.1` before building extensions |
| 5 | `cudart64_118.dll not found` at runtime | Windows doesn't expose the conda env's CUDA runtime DLL to Python extension modules by default | Add torch's lib directory to the DLL search path via `os.add_dll_directory` (patched in this repo) |
| 6 | NumPy 2.x crash / deprecation errors | PyTorch 2.0.1 was compiled against NumPy 1.x | Downgrade: `pip install "numpy<2"` |
| 7 | `opencv-python` dependency conflict | opencv 4.13+ requires NumPy ≥ 2, which conflicts with #6 | Downgrade: `pip install opencv-python==4.8.1.78` |
| 8 | `Cannot handle this data type: \|i1` | `np.byte` is signed int8; PIL expects unsigned uint8 for RGB | Patched in `scene/dataset_readers.py` (in this repo) |
| 9 | `Could not recognize scene type!` | The repo only supports Blender synthetic and COLMAP formats, not LLFF | Use only `transforms_train.json` (Blender) or `sparse/0/` (COLMAP) datasets |
| 10 | SIBR interactive viewer crashes on launch | The pre-built SIBR binaries were compiled against CUDA 12.1; only CUDA 11.8 was installed | Install CUDA 12.1 alongside 11.8, or use the lightweight [antimatter15/splat](https://github.com/antimatter15/splat) web viewer instead |

---

## Setup (Windows)

### Prerequisites

- NVIDIA driver 527+
- [CUDA Toolkit 11.8](https://developer.nvidia.com/cuda-11-8-0-download-archive)
- [CUDA Toolkit 12.1](https://developer.nvidia.com/cuda-12-1-0-download-archive) *(only needed for the SIBR interactive viewer)*
- Visual Studio 2022 with **Desktop development with C++** workload
- [Miniconda](https://docs.conda.io/en/latest/miniconda.html)

### Install

Open **Anaconda Prompt**:

```cmd
git clone https://github.com/i-am-mushfiq/Fixed-gaussian-splatting --recursive
cd Fixed-gaussian-splatting
```

```cmd
conda create -n gaussian_splatting python=3.10 -y
conda activate gaussian_splatting
```

```cmd
pip install torch==2.0.1 torchvision==0.15.2 torchaudio==2.0.2 --index-url https://download.pytorch.org/whl/cu118
pip install setuptools==69.5.1
pip install plyfile tqdm Pillow scipy lpips tensorboard
pip install opencv-python==4.8.1.78
pip install "numpy<2"
```

Build CUDA extensions — **must use CMD, not PowerShell**:

```cmd
"C:\Program Files\Microsoft Visual Studio\2022\Community\VC\Auxiliary\Build\vcvars64.bat"
set DISTUTILS_USE_SDK=1
conda activate gaussian_splatting
pip install submodules/diff-gaussian-rasterization --no-build-isolation
pip install submodules/simple-knn --no-build-isolation
```

---

## Usage

```cmd
conda activate gaussian_splatting

# Train (Blender synthetic)
python train.py -s <path_to_scene> --eval --white_background

# Train (COLMAP)
python train.py -s <path_to_scene> --eval

# Render
python render.py -m output/<timestamp>

# Metrics
python metrics.py -m output/<timestamp>
```

### Viewing results

**Browser viewer** (no CUDA 12 needed — drag the `.ply` in):
```cmd
cd splat
python -m http.server 8080
```
Open `http://localhost:8080` and drag in `output/<timestamp>/point_cloud/iteration_30000/point_cloud.ply`.

**SIBR interactive viewer** (requires CUDA 12.1):
```cmd
viewers\bin\SIBR_gaussianViewer_app.exe -m output/<timestamp>
```

---

## Original repository

[graphdeco-inria/gaussian-splatting](https://github.com/graphdeco-inria/gaussian-splatting) — licensed under the [LICENSE.md](LICENSE.md) terms from the original authors.
