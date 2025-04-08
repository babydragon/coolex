+++
title = "在intel gpu上尝试Rust ORT"
date = "2025-04-08"
[taxonomies]
tags = ["rust", "onnxruntime", "openvino"]
categories = ["Rust"]
+++

自动更新了新的电脑之后，一直想尝试下如何在intel core ultra9的内置GPU上运行神经网络模型。
intel的GPU支持openvino，onnxruntime也支持openvino作为后端，因此可以尝试在rust中使用onnxruntime的openvino后端。
<!-- more -->

## 准备工作
### 安装openvino
首先需要安装openvino，具体安装步骤可以参考[openvino的官方文档](https://www.intel.com/content/www/us/en/developer/tools/openvino-toolkit/download.html?PACKAGE=OPENVINO_BASE&VERSION=v_2025_0_0&OP_SYSTEM=LINUX&DISTRIBUTION=GITHUB)。
由于Gentoo系统没有官方支持，只能选择从源码进行安装。

首先准备源码：
```bash
git clone -b 2025.0.0 https://github.com/openvinotoolkit/openvino.git
cd openvino
git submodule update --init --recursive
```

开始构建openvino：
```bash
mkdir build
cd build
cmake -DCMAKE_BUILD_TYPE=Release -DENABLE_CPPLINT=OFF ..
cmake --build . --parallel 8
```
注意这里需要使用`--parallel`参数来指定并行编译的线程数，按照电脑CPU核心数量来进行设置。因为有很多第三方库会导致cpplint失败，这里关闭cpplint检查。

构建完成后，安装到指定目录：
```bash
cmake --install   ~/data/ai/intel/openvino
```

### 安装onnxruntime
由于要使用openvino EP，直接使用了intel fork的onnxruntime版本，参照官方[文档](https://onnxruntime.ai/docs/build/eps.html#openvino)，首先还是拉取代码：
```bash
git clone https://github.com/intel/onnxruntime.git
```

在构建之前，需要先加载openvino的环境变量：
```bash
source ~/data/ai/intel/openvino/setupvars.sh
```

然后在进入onnxruntime的目录，准备开始构建：
```bash
cd onnxruntime
./build.sh --update --config Release --use_openvino GPU --build_shared_lib
```
注意这里的`--use_openvino`参数，指定GPU作为后端，不知道什么原因，如果使用文档中描述的`HETERO:`模式，构建的时候会出错。
构建完成后，可以在`onnxruntime/build/Linux/Release`目录下找到构建好的动态链接库，后续rust的ort库需要使用。

## 使用ort
这里使用ort提供的yolo实例来进行测试，代码基本上是从[example](https://github.com/pykeio/ort/tree/main/examples/yolov8)中copy过来的，去除了图片展示部分，直接将标记的图片生成到了文件中。

首先添加依赖：
```toml
[dependencies]
ort = { version = "2.0.0-rc.9", features = ["openvino"] }
tracing = "0.1"
tracing-subscriber = { version = "0.3", default-features = false, features = [ "env-filter", "fmt" ] }
anyhow = "1.0"
raqote = { version = "0.8", features = ["png"]}
image = "0.25"
ndarray = "0.16"

[patch.crates-io]
ort = {path = "../ort"}
```

注意，这里添加了ort依赖，需要使用rc版本，老版本无法支持新的onnxruntime。另外，ort需要通过features来指定使用openvino作为后端。
这里fork了ort，为了解决ort针对openvino EP使用老版本API导致的初始化问题，具体下文会提到。

代码中需要先初始化tracing和ort：
```rust
tracing_subscriber::fmt::init();
ort::init().commit()?;

let mut builder = Session::builder()?;
let provider
    = OpenVINOExecutionProvider::default().with_device_type("GPU").with_opencl_throttling(true);

if provider.is_available()? {
    println!("OpenVINO GPU execution provider is available");
} else {
    println!("OpenVINO GPU execution provider is not available");
}
provider.register(&mut builder)?;

let mut model = builder
    .commit_from_file(Path::new(env!("CARGO_MANIFEST_DIR")).join("yolov8m.onnx"))?;
```
注意在代码最开始初始化`tracing_subscriber`和`ort`，否则会导致初始化出错。
注册openvino的EP需要使用`OpenVINOExecutionProvider::default()`来进行初始化，
注意这里的`with_device_type`参数指定GPU作为后端。

后面调用模型和创建图片，和原来的代码基本一致，只是去掉了图片展示的部分，直接将标记的图片保存到文件中。

## 运行
运行之前，需要通过环境变量来指定onnxruntime的动态链接库路径：
```bash
ORT_LIB_LOCATION=~/data/ai/onnxruntime/build/Linux/Release cargo build
ORT_LIB_LOCATION=~/data/ai/onnxruntime/build/Linux/Release LD_LIBRARY_PATH=~/data/ai/onnxruntime/build/Linux/Release cargo run
```

参照ort文档，需要动态链接onnxruntime的时候，可以通过`ORT_LIB_LOCATION`来指定onnxruntime的动态链接库路径。
执行的时候，需要指定`LD_LIBRARY_PATH`来指定动态链接库的查找路径，否则执行的时候会报找不到动态链接库。

执行的时候，可以通过`intel_gpu_top`命令来查看GPU的使用情况，确认是否使用了openvino的EP是否使用了GPU。

如果需要显示详细的日志，可以通过设置`RUST_LOG`环境变量来指定ort的日志级别：
```bash
RUST_LOG="ort=debug" ORT_LIB_LOCATION=~/data/ai/onnxruntime/build/Linux/Release LD_LIBRARY_PATH=~/data/ai/onnxruntime/build/Linux/Release cargo run
```

## 问题
rc版本的ort在使用openvino EP的时候，可能会出现以下错误：
```
[ERROR] [OpenVINO-EP] enable_opencl_throttling  should be a boolean.
```
实际在ort代码中传入的的确是bool类型的参数，而且确认了下，ort使用的是legacy的openvino EP API，按照[官方API文档](https://onnxruntime.ai/docs/api/c/struct_ort_open_v_i_n_o_provider_options.html)的描述，
实际对应的类型也是`unsigned char`。

找到onnxruntime报错的地方(`onnxruntime/core/providers/openvino/openvino_provider_factory.cc`)：
```c++
bool ParseBooleanOption(const ProviderOptions& provider_options, std::string option_name) {
  if (provider_options.contains(option_name)) {
    const auto& value = provider_options.at(option_name);
    if (value == "true" || value == "True") {
      return true;
    } else if (value == "false" || value == "False") {
      return false;
    } else {
      ORT_THROW("[ERROR] [OpenVINO-EP] ", option_name, " should be a boolean.\n");
    }
  }
  return false;
}
```
从代码逻辑可以看出，onnxruntime实际是从字符串来解析bool类型的参数的。而factory代码中，已经使用新的参数了，
legacy的参数在`onnxruntime/core/session/provider_bridge_ort.cc`中进行了适配：
```c++
// Adapter to convert the legacy OrtOpenVINOProviderOptions to ProviderOptions
ProviderOptions OrtOpenVINOProviderOptionsToOrtOpenVINOProviderOptionsV2(const OrtOpenVINOProviderOptions* legacy_ov_options) {
  ProviderOptions ov_options_converted_map;
  if (legacy_ov_options->device_type != nullptr)
    ov_options_converted_map["device_type"] = legacy_ov_options->device_type;

  if (legacy_ov_options->num_of_threads != '\0')
    ov_options_converted_map["num_of_threads"] = std::to_string(legacy_ov_options->num_of_threads);

  if (legacy_ov_options->cache_dir != nullptr)
    ov_options_converted_map["cache_dir"] = legacy_ov_options->cache_dir;

  if (legacy_ov_options->context != nullptr) {
    std::stringstream context_string;
    context_string << legacy_ov_options->context;
    ov_options_converted_map["context"] = context_string.str();
  }

  ov_options_converted_map["enable_opencl_throttling"] = legacy_ov_options->enable_opencl_throttling;

  if (legacy_ov_options->enable_dynamic_shapes) {
    ov_options_converted_map["disable_dynamic_shapes"] = "false";
  } else {
    ov_options_converted_map["disable_dynamic_shapes"] = "true";
  }

  if (legacy_ov_options->enable_npu_fast_compile) {
    LOGS_DEFAULT(WARNING) << "enable_npu_fast_compile option is deprecated. Skipping this option";
  }
  // Add new provider option below
  ov_options_converted_map["num_streams"] = "1";
  ov_options_converted_map["load_config"] = "";
  ov_options_converted_map["model_priority"] = "DEFAULT";
  ov_options_converted_map["enable_qdq_optimizer"] = "false";
  return ov_options_converted_map;
}
```
这里的代码很迷惑，其他bool类型的参数，都使用了对应的字符串进行替换，唯独`enable_opencl_throttling`，直接复制了laogacy的参数（也就是unsigned char类型）。
而按照参数的定义（`0 = disabled, nonzero = enabled`参见前面链接的API文档），这里传的值，在后面`ParseBooleanOption`函数中，一定会报错的。

由于ort的代码中，针对onnxruntime的c语言绑定（ort-sys）中，已经定义了V2的函数，因此决定先改成V2的函数，绕开这个神奇的判断。
具体的改动在[这个提交](https://github.com/babydragon/ort/commit/c1519fd309737cc46f02798ddebc6c3f280d4556)中。

ps：写博客的时候，ort已经修复了这个[问题](https://github.com/pykeio/ort/issues/380)，因此不在需要使用fork的版本了。

运行效果：
输入图片：
![input](/ort_yolo/input.jpg)

输出图片：
![output](/ort_yolo/result.png)
