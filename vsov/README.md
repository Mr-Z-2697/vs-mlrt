# VapourSynth OpenVINO

The vs-openvino plugin provides optimized *pure* CPU runtime for some popular AI filters.

## Building and Installation

To build, you will need [OpenVINO](https://docs.openvino.ai/latest/get_started.html) and its dependencies.
Only `Model Optimizer` and `Inference Engine` are required.

You can download official Intel releases:
- [Linux](https://docs.openvino.ai/latest/openvino_docs_install_guides_installing_openvino_linux_header.html)
- [Windows](https://docs.openvino.ai/latest/openvino_docs_install_guides_installing_openvino_windows_header.html)
- [macOS](https://docs.openvino.ai/latest/openvino_docs_install_guides_installing_openvino_macos_header.html)

Or, you can use our prebuilt Windows binary releases from [AmusementClub](https://github.com/AmusementClub/openvino/releases/latest/), our release has the benefit of static linking support.

Sample cmake commands to build:
```bash
cmake -S . -B build -G Ninja -D CMAKE_BUILD_TYPE=Release
	-D CMAKE_MSVC_RUNTIME_LIBRARY=MultiThreaded
	-D InferenceEngine_DIR=openvino/runtime/cmake
	-D VAPOURSYNTH_INCLUDE_DIRECTORY="path/to/vapoursynth/include"
cmake --build build
cmake --install build --prefix install
```
You should find `vsov.dll` (or libvsov.so) under `install/bin`. You will also need Intel TBB (you can get
`tbb.dll` from OpenVINO release). On windows, `tbb.dll` must be placed under `vapoursynth/plugins/vsov/`
directory for `vsov.dll` to find.

## Usage

Prototype: `core.ov.Model(clip[] clips, string network_path[, int[] overlap = None, int[] tilesize = None, string device = "CPU", bint builtin = 0, string builtindir="models", bint fp16 = False, function config = None, bint path_is_serialization = False])`

Arguments:
 - `clip[] clips`: the input clips, only 32-bit floating point RGB or GRAY clips are supported. For model specific input requirements, please consult our [wiki](https://github.com/AmusementClub/vs-mlrt/wiki).
 - `string network_path`: the path to the network in ONNX format.
 - `int[] overlap`: some networks (e.g. [CNN](https://en.wikipedia.org/wiki/Convolutional_neural_network)) support arbitrary input shape where other networks might only support fixed input shape and the input clip must be processed in tiles. The `overlap` argument specifies the overlapping (horizontal and vertical, or both, in pixels) between adjacent tiles to minimize boundary issues. Please refer to network specific docs on the recommended overlapping size.
 - `int[] tilesize`: Even for CNN where arbitrary input sizes could be supported, sometimes the network does not work well for the entire range of input dimensions, and you have to limit the size of each tile. This parameter specify the tile size (horizontal and vertical, or both, including the overlapping). Please refer to network specific docs on the recommended tile size.
 - `string device`: Specifies the device to run the inference on. Currently `"CPU"` and `"GPU"` are supported. `"GPU"` requires Intel graphics (Broadwell+ processors with Gen8+ integrated GPUs or Xe discrete GPUs) with compatible graphics driver and compute runtime.
 - `bint builtin`: whether to load the model from the VS plugins directory, see also `builtindir`.
 - `string builtindir`: the model directory under VS plugins directory for builtin models, default "models".
 - `bint fp16`: whether to quantize model to fp16 for faster and memory efficient computation.
 - `function config`: plugin configuration parameters. It must be a callable object (e.g. a function) with no positional arguments, and returns the configuration parameter in a dictionary `dict`. The dictionary must use string `str` for its key and `int`, `float` or `str` for its values. Supported parameters: [CPU](https://docs.openvino.ai/2021.4/openvino_docs_IE_DG_supported_plugins_CPU.html#supported-configuration-parameters), [GPU](https://docs.openvino.ai/2021.4/openvino_docs_IE_DG_supported_plugins_GPU.html#supported-configuration-parameters) (the prefix `KEY_` has to be removed). Example: `config = lambda: dict(CPU_THROUGHPUT_STREAMS=2)`
 - `bint path_is_serialization`: whether the `network_path` argument specifies an onnx serialization of type `bytes`.

When `overlap` and `tilesize` are not specified, the filter will internally try to resize the network to fit the input clips. This might not always work (for example, the network might require the width to be divisible by 8), and the filter will error out in this case.

The general rule is to either:
1. left out `overlap`, `tilesize` at all and just process the input frame in one tile, or
2. set all three so that the frame is processed in `tilesize[0]` x `tilesize[1]` tiles, and adjacent tiles will have an overlap of `overlap[0]` x `overlap[1]` pixels on each direction. The overlapped region will be throw out so that only internal output pixels are used.
