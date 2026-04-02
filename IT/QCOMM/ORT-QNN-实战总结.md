# ONNX Runtime + QNN (HTP) Android 推理实战总结

> 目标平台：Qualcomm Snapdragon 8 Elite (SM8750)，HTP V79  
> 推理框架：ONNX Runtime + QNN EP（Execution Provider）  
> 应用场景：医学图像语义分割（CTM 模型）

---

## 一、整体架构

```
Android App (Kotlin)
    │
    ├── 预处理（CPU）：YUV→Bitmap→NCHW float32
    │
    ▼
InferSession (C++ AAR SDK)
    │
    ├── OnnxRuntime C++ API
    │   └── QNN Execution Provider
    │       └── HTP V79（NPU）
    │
    ▼
后处理输出：INT32 分割图 [1, H, W]
    │
    ▼
轮廓提取 → Protobuf → Kotlin → UI 渲染
```

---

## 二、模型准备

### 2.1 基本要求

| 项目 | 要求 |
|------|------|
| 格式 | `.onnx`（opset ≥ 11） |
| 输入 | `float32`，NCHW，形状 `[1, C, H, W]` |
| 输出 | `int32` 或 `float32`（见下节） |
| 归一化 | ImageNet（mean=[0.485,0.456,0.406]，std=[0.229,0.224,0.225]） |

### 2.2 支持的输出格式

SDK 支持三种输出格式，**不满足则推理结果为空（0 polygon）**：

| 格式 | 形状 | SDK 处理方式 |
|------|------|-------------|
| `INT32` | `[1, H, W]` | 直接用作类别索引图 |
| `float32` | `[1, C, H, W]` | SDK 内部 argmax → INT32 |
| `float32` | `[1, H, W]` | SDK 内部 cast → INT32 |

---

## 三、常见问题与解决方案

### ❗ 问题 1：推理正常运行但始终 0 polygon

**原因**：ONNX `ArgMax` 算子默认输出 `INT64`，SDK 不支持该类型，结果被丢弃。

**诊断**：
```python
import onnx
m = onnx.load("model.onnx")
for o in m.graph.output:
    print(o.name, o.type.tensor_type.elem_type)
    # 1=FLOAT ✓  6=INT32 ✓  7=INT64 ✗
```

**解决**：在模型末尾追加 Cast 节点，将 INT64 转为 INT32：
```python
import onnx
from onnx import helper, TensorProto

m = onnx.load("model_argmax_fp32.onnx")

# 查找原始输出节点名（通常是 ArgMax 的输出）
original_output = m.graph.output[0].name  # e.g. "out0"

# 追加 Cast 节点
cast_node = helper.make_node(
    "Cast",
    inputs=[original_output],
    outputs=["out0_i32"],
    to=TensorProto.INT32
)
m.graph.node.append(cast_node)

# 更新模型输出
m.graph.output[0].name = "out0_i32"
m.graph.output[0].type.tensor_type.elem_type = TensorProto.INT32

onnx.save(m, "model_argmax_int32.onnx")
```

> ⚠️ 转换后必须清除设备旧缓存（`.qnn_ctx.onnx` 和 `.qnn_ctx_qnn.bin`），否则 SDK 仍加载旧版本。

---

### ❗ 问题 2：首次启动极慢（10~30 秒卡顿）

**原因**：QNN EP 在首次运行时需要将 ONNX 模型编译为 SM8750 原生 HTP 指令集。

**表现**：
- 首次启动编译耗时 **10~30 秒**（取决于模型大小）
- 编译结果缓存在设备本地（`.qnn_ctx_qnn.bin`）
- 后续启动仅约 **200ms**

**建议**：UI 上显示"模型编译中，请稍候"，避免用户误以为崩溃。

---

### ❗ 问题 3：更换模型后速度异常 / 结果不对

**原因**：`.qnn_ctx_qnn.bin` 缓存与新模型不匹配，SDK 仍加载旧编译结果。

**解决**：每次替换模型后，手动删除设备上的缓存文件：
```bash
adb shell "find /sdcard/ -name '*.qnn_ctx*' -delete"
# 或直接清除 App 数据：
adb shell pm clear com.your.package
```

---

### ❗ 问题 4：MediaCodec + ImageReader 导致 JNI 崩溃

**现象**：
```
JNI DETECTED ERROR: non-zero capacity for nullptr pointer: 1
```

**原因**：`ImageReader.acquireLatestImage()` 与 `MediaCodec.releaseOutputBuffer(idx, true)` 存在竞态，Surface 回调时机不稳定。

**解决**：不使用 Surface 输出模式，改用 `getOutputImage()`：
```kotlin
// 配置 MediaCodec（不绑定 Surface）
mediaFormat.setInteger(
    MediaFormat.KEY_COLOR_FORMAT,
    MediaCodecInfo.CodecCapabilities.COLOR_FormatYUV420Flexible
)
codec.configure(mediaFormat, null, null, 0)  // surface = null

// 读取帧
val outputIndex = codec.dequeueOutputBuffer(bufferInfo, 10_000)
if (outputIndex >= 0) {
    val image = codec.getOutputImage(outputIndex)  // ← 正确方式
    // 处理 image...
    codec.releaseOutputBuffer(outputIndex, false)  // false，不渲染到 Surface
}
```

---

## 四、预处理流水线（性能分析）

### 4.1 完整流程与数据类型

```
视频帧 (YUV420)
    │ yuv: 25~34ms（decode 线程，NV21→JPEG→Bitmap，效率较低）
    ▼
Bitmap ARGB_8888 [H×W×4]  uint8
    │ rsz: 3~5ms（createScaledBitmap，双线性插值，输出仍为 uint8）
    ▼
Bitmap ARGB_8888 [512×512×4]  uint8
    │ px: 1~2ms（getPixels → int32[] packed ARGB）
    ▼
int32[] packed  [512×512]
    │ nrm: 5~15ms（逐像素提取 RGB，ImageNet 归一化，排列为 NCHW float32）
    ▼
float32[] NCHW  [1×3×512×512]  → SDK → HTP（内部 fp16）
    │ inf: 43~48ms
    ▼
int32[] [1×512×512]  类别索引图
```

### 4.2 性能关键点

| 步骤 | 线程 | 耗时 | 说明 |
|------|------|------|------|
| yuv（YUV→Bitmap） | Decode | 25~34ms | NV21→JPEG→解码，**最大瓶颈** |
| rsz（resize） | Decode | 3~5ms | createScaledBitmap |
| px（getPixels） | Inference | 1~2ms | |
| nrm（归一化+NCHW） | Inference | 5~15ms | Kotlin 逐像素循环 |
| inf（HTP 推理） | Inference | 43~48ms | |

> Decode 线程和 Inference 线程**并行**执行，实际帧率由 `max(decode, inference)` 决定，约 **56ms/帧（~17fps）**。

### 4.3 优化方向（优先级排序）

1. **INT8 量化** — 最高 ROI，HTP 吞吐量翻倍，推理 43ms → ~20ms，无需改 App 代码
2. **优化 YUV→Bitmap** — 用 `YuvImage` + `libyuv`（Native）替代 JPEG 中转，yuv 可从 30ms 降至 <5ms
3. **Native NEON 归一化** — nrm 从 10ms 降至 <2ms
4. **流水线并行** — 推理第 N 帧时并行预处理第 N+1 帧（当前已有雏形）
5. **模型内置预处理** — 将 Transpose/Resize/Normalize 集成进模型在 NPU 执行，但受限于 SDK 目前需要 float32 输入，实测反而变慢（增加了 CPU 侧 float32 转换开销）

---

## 五、HTP 精度配置

QNN HTP 默认使用 **FP16 精度**（通过 `enable_htp_fp16_precision=1` 开启），即：

- App 层传入 `float32` tensor
- SDK 喂给 HTP 时自动转为 `fp16`
- 输出再转回 `float32` / `int32`

这是性能与精度的折中配置，对分割类任务精度损失可接受。

---

## 六、模型部署清单

```
/sdcard/Android/data/com.your.package/files/
├── model.onnx               ← 推理模型（INT32 输出）
├── model.qnn_ctx.onnx       ← QNN 编译中间产物（自动生成）
└── model.qnn_ctx_qnn.bin    ← HTP 原生指令缓存（自动生成，勿手动编辑）
```

**部署步骤**：
```bash
# 推送模型
adb push model_argmax_int32.onnx /sdcard/Android/data/com.your.package/files/model.onnx

# 清除旧缓存（必须）
adb shell "rm -f /sdcard/Android/data/com.your.package/files/*.qnn_ctx*"

# 重启 App，等待 HTP 重新编译（首次 10~30s）
adb shell am force-stop com.your.package
adb shell monkey -p com.your.package 1
```

---

## 七、参考资料

- [ONNX Runtime QNN EP 官方文档](https://onnxruntime.ai/docs/execution-providers/QNN-ExecutionProvider.html)
- [Qualcomm AI Engine Direct (QNN) SDK](https://developer.qualcomm.com/software/qualcomm-neural-processing-sdk)
- [ONNX ArgMax 算子规范](https://onnx.ai/onnx/operators/onnx__ArgMax.html)（默认输出 INT64）
- 内部 SDK：`onnx-infer-cpp` / `onnx-infer` AAR，InferSession.cpp
