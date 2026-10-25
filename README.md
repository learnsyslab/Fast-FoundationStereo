# Fast-FoundationStereo (learnsyslab)

Fork of [NVlabs/Fast-FoundationStereo](https://github.com/NVlabs/Fast-FoundationStereo).
The upstream documentation (method, models, Docker/pip setup, demos) is in
[README_ORIG.md](README_ORIG.md). This file only covers what the fork adds.

## Added here

- **Pixi environments** (`pixi.toml`): `default` (PyTorch, CUDA 12.8) and
  `trt` (+ TensorRT 10.11.0.33).
- **TensorRT build without Docker or `trtexec`**: `scripts/build_single_trt.py`.

## Setup

```bash
pixi install            # PyTorch environment
pixi install -e trt     # + TensorRT
```

Weights go in `weights/23-36-37/` (see README_ORIG.md for the download).

## TensorRT engine

```bash
pixi run -e trt trt-export      # ONNX -> output_single_trt/
pixi run -e trt trt-build       # FP16 engine -> output_single_trt/fast_foundationstereo.engine
pixi run -e trt trt-demo        # run it on demo_data/
pixi run -e trt trt-benchmark   # timing
```

The engine is fixed at **640x480, 8 iterations, max_disp 192** (set at export).
Engines only run on the GPU and TensorRT version they were built with: rebuild
on every new machine. Details and verification: [BUILD_TRT.md](BUILD_TRT.md).

## License

NVIDIA source code license ([LICENSE.txt](LICENSE.txt)): research /
non-commercial use only.
