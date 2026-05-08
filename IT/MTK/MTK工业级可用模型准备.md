# YOLO26 模型：MTK APU 工业级 INT8 量化与部署方案

## 0. 核心目标
* **模型底座**：YOLO26
* **硬件目标**：MTK APU (NeuroPilot SDK) 原生 `.dla` 文件
* **部署指标**：INT8 硬件级全加速，确保精度损失 mIoU ≤ 2%
* **业务场景**：SZJ 面部医疗超声影像实时分割

---

## 阶段一：量化感知训练 (QAT) - 精度无损转换
在 PyTorch 环境下执行，通过模拟量化噪声使模型在训练中适应 INT8 的精度损失。

* **算子替换**：利用 `torch.ao.quantization` 将模型中的卷积层和全连接层包装为支持伪量化的版本。
* **训练微调**：加载 FP32 预训练权重，开启 QAT 模式，使用较低学习率（通常为原先的 10%）进行 10-20 个 Epoch 的微调。
* **伪量化核心**：在微调过程中人为模拟截断效应，使模型权重自动规避量化溢出。
* **产出**：带有量化参数（Scale & Zero Point）的 FP32 ONNX 模型。

---

## 阶段二：算子预检与架构优化
在转换前，通过静态分析确保模型与 MTK 硬件指令集 100% 对齐。

* **兼容性扫描**：运行 `neuro_pilot_inspector` 工具扫描 ONNX 模型，确保所有算子均为 Green（APU 加速）。
* **算子平替**：若发现不支持的算子（如某些硬件版本的 SiLU），需在代码中平替为 ReLU 或 HardSwish。
* **静态化约束**：锁定模型输入尺寸（例如 1x3x640x640），禁用动态张量以优化寄存器调度。
* **逻辑剥离**：确保 NMS (非极大值抑制) 不在模型内部，准备在 C++ 层实现。

---

## 阶段三：校准数据准备工具 (Standalone Data Tool)
**目的**：从原始采集序列中提取样本，生成与 C++ 推理代码“像素级同步”的校准集。

### 3.1 工具功能定义
* **输入源**：指向用户原始扫描数据的存储路径 `SOURCE_DIR`。
* **抽样算法**：从每个用户的序列中按固定步长抽取代表性帧（建议总量 200-500 张）。
* **处理流水线 (Preprocessing)**：
    1. **Resize**：缩放至模型输入尺寸，必须指定固定的插值算法（建议 `cv::INTER_LINEAR`）。
    2. **Normalization**：执行标准化（参数必须与训练阶段完全一致）。
    3. **格式转换**：将像素数据转换为 `float32` 并输出为二进制文件 `.bin`。

### 3.2 预处理逻辑参考
```python
# 此逻辑必须与 C++ 推理代码中的预处理逻辑完全一致
def preprocess_for_calibration(image, target_size, mean, std):
    # 1. Resize
    img = cv2.resize(image, target_size, interpolation=cv2.INTER_LINEAR)
    # 2. BGR to RGB (若训练时使用 RGB)
    img = cv2.cvtColor(img, cv2.COLOR_BGR2RGB)
    # 3. Normalize
    img = (img / 255.0 - mean) / std
    return img.astype(np.float32)
```

---

## 阶段四：NeuroPilot 编译与 DLA 转换
利用阶段三生成的二进制校准集，驱动 MTK 编译器进行指令集映射。

* **校准执行**：`ncc` 编译器读取预处理后的 `.bin` 文件，确定 INT8 映射的动态范围。
* **DLA 编译**：
    * 开启 `Layout Optimization`（优先 NHWC 格式）。
    * 开启 `Operator Fusion`（卷积、偏置与激活函数的硬件级合并）。
* **产出**：`.dla` 原生二进制包。

---

## 阶段五：C++ 推理引擎开发 (Runtime)
基于 Tauri (Rust) + C++ 实现高性能推理环。

* **预处理同步**：C++ 中的 `PreProcess()` 函数必须直接复用阶段三定义的算法逻辑。
* **零拷贝内存 (Zero-copy)**：利用 Ion Memory 实现采集数据到 APU 的直接映射，消除 `memcpy` 开销。
* **异步持久化**：推理结果通过异步线程写入 SQLite 数据库。

---

## 验收清单 (Checklist)
- [ ] **精度验证**：INT8 与 FP32 的 mIoU 差距 ≤ 2%。
- [ ] **算子验证**：100% 运行在 APU，无 CPU Fallback。
- [ ] **实时性**：单帧推理延迟 ≤ 30ms。
- [ ] **一致性**：校准工具输出数据与 C++ 预处理输出数据的 MD5 值匹配。
