# Fast-FoundationStereo (learnsyslab)

Fork of [NVlabs/Fast-FoundationStereo](https://github.com/NVlabs/Fast-FoundationStereo).
The upstream documentation (method, models, Docker/pip setup, demos) is in
[README_ORIG.md](README_ORIG.md). This file only covers what the fork adds.

## Added here

- **Pixi environments** (`pixi.toml`): `default` (PyTorch, CUDA 12.8) and
  `trt` (+ TensorRT 10.11.0.33).
- **TensorRT build without Docker or `trtexec`**: `scripts/build_single_trt.py`.
- **ROS 2 depth node** for a RealSense IR pair: `scripts/ros2_depth_node.py`.

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

## ROS 2 depth node (Jazzy)

```bash
# terminal 1: RealSense IR pair
ros2 launch realsense2_camera rs_launch.py enable_infra1:=true enable_infra2:=true \
  enable_color:=true depth_module.infra_profile:=640x480x30
# terminal 2
pixi run ros-depth
```

Publishes metric `32FC1` depth on `/fast_foundation_stereo/depth/image_raw`
(+ `camera_info` and a `PointCloud2` on `/fast_foundation_stereo/points`).
It runs the PyTorch model (`valid_iters` 4 by default), not the TensorRT engine;
topics and settings are ROS parameters in the script.

## License

NVIDIA source code license ([LICENSE.txt](LICENSE.txt)): research /
non-commercial use only.
