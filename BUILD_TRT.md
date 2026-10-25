# Building the TensorRT engine (this fork)

How the engine in `output_single_trt/` was produced, and how to rebuild it on
another machine. The generic commands are in the upstream readme
([README_ORIG.md](README_ORIG.md), section "ONNX/TRT"); this file records the settings actually used here and the
things that are easy to get wrong.

## What was built

| | |
|---|---|
| weights | `weights/23-36-37/model_best_bp2_serialize.pth` (vitl) |
| export | single ONNX (`scripts/make_single_onnx.py`), no plugin needed |
| input / output | `left_image`, `right_image`: float32 `1x3x480x640`; `disparity`: float32 `1x1x480x640` (static shapes) |
| `valid_iters` | **8** (baked into the engine) |
| `max_disp` | 192 |
| precision | built with `--fp16` as in the upstream command; not recorded in the engine itself (60 MB engine vs 86 MB fp32 ONNX suggests reduced precision) |
| TensorRT | **10.11.0.33** (checked by deserializing the engine) |
| GPU | NVIDIA RTX 5090 |
| speed | ~13 ms per frame (848x480 IR pair resized to the 640x480 input) |

`output_single_trt/fast_foundationstereo.yaml` is written by the export and
holds these settings; keep it next to the `.onnx` / `.engine`.

## Things that are easy to get wrong

- **The engine is not portable.** A `.engine` is compiled for one GPU
  architecture and one TensorRT version. Copying it to a different GPU model, a
  Jetson, or a machine with another TensorRT version fails at deserialization
  (or, worse, is refused only at runtime). Copy the `.onnx` + `.yaml` (or the
  weights) and build the engine on the target machine.
- **The TensorRT that builds the engine must match the one that runs it.** If
  inference runs from the `tensorrt` Python package, build with `trtexec` from
  the same TensorRT release (here 10.11.0.33). Upgrading either side means
  rebuilding.
- **`valid_iters` and the input size are fixed at export.** Changing them in a
  runtime config has no effect on the engine; re-export the ONNX and rebuild.
  (The PyTorch path reads `valid_iters` at runtime; the engine does not.)
- **Height and width must be divisible by 32.** Inputs of another size are
  resized to 640x480 by the caller before inference.

## Rebuild

With the Pixi environment (`pixi.toml`, feature `trt`: TensorRT 10.11.0.33,
PyTorch 2.9.1, CUDA 12.8), on the machine that will run the engine:

```bash
pixi run -e trt trt-check     # TensorRT / PyTorch / CUDA / GPU versions
pixi run -e trt trt-export    # ONNX -> output_single_trt/ (480x640, valid_iters 8, max_disp 192)
pixi run -e trt trt-build     # engine via scripts/build_single_trt.py --fp16 (no trtexec needed)
```

`trt-export` is only needed if the `.onnx` is missing or the settings change.
Without Pixi, the same with `trtexec` from the matching TensorRT release:

```bash
python scripts/make_single_onnx.py \
    --model_dir weights/23-36-37/model_best_bp2_serialize.pth \
    --save_path output_single_trt/ --height 480 --width 640 \
    --valid_iters 8 --max_disp 192
trtexec --onnx=output_single_trt/fast_foundationstereo.onnx \
        --saveEngine=output_single_trt/fast_foundationstereo.engine --fp16
```

## Verify

```bash
pixi run -e trt trt-demo        # disparity of the demo pair with the engine -> output_trt/
pixi run -e trt trt-benchmark   # 100 timed runs
pixi run run-demo               # the same pair with the PyTorch model, for comparison
```

Compare the engine's disparity with the PyTorch model's on the same pair before
relying on a new engine: an engine built from the wrong ONNX or settings
produces plausible-looking but wrong depth.

## License

NVIDIA's source code license (`LICENSE.txt`): research / non-commercial use
only. Do not redistribute weights or engines outside those terms; point to the
upstream repository instead.
