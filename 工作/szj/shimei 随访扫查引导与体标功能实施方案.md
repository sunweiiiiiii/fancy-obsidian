# ShiMei「随访扫查引导」与「体标」功能实施方案

> 状态：Plan，待确认后实施  
> 编写日期：2026-07-16  
> 代码基线：`C:\code\shimei-main\shimei-main\native-app`  
> 本阶段不包含智能配准、图像相似度判断或自动探头方向提示。

## 1. 目标

在现有【面诊扫查】和【医美注射】共用扫查界面中增加两个顶部工具：

1. **体标**：在面部简图上记录当前超声切面的大致位置和方向，随当前证据一起保存。
2. **随访扫查引导**：选择同一顾客带体标的历史证据，左侧查看历史原始超声和体标，右侧继续实时扫查，由操作者人工寻找接近切面。

这两个功能是扫查过程中的辅助工具，不改变面诊、注射原有的采集、暂停、保存和结束流程。

## 2. 现有 UI 与数据链路结论

### 2.1 当前 UI 基线

- 当前实际交付主线是原生 Android `native-app`，使用 Kotlin + Jetpack Compose。
- 首页已经包含【面诊扫查】和【医美注射】入口，二者最终共用 `feature:scan` 中的 `ScanScreen`。
- 普通真实/虚拟探头模式下：
  - 顶部是 `ScanInfoBar`，包含返回、当前业务标题，面诊真实探头还可以显示增益入口；
  - 中部默认是原始超声和 AI 解释图双屏；
  - 底部面诊使用 `ScanDashboard`，注射使用 `InjectionControlPanel`。
- `SCREEN_CAPTURE` 模式为保证外部应用和 ShiMei 窗格对齐，当前会隐藏顶部栏和底部面板，改用画面内悬浮控件。
- AI 标注目前在界面上提供三种可见样式：渐变、线框、实心。代码中已经有 `ORIGINAL` 状态，只是当前被隐藏。
- 当前已有正面、双侧脸导航素材和区域点击逻辑，但它们用于“部位选择”，不是可编辑、可持久化的体标，不能直接等同于本次功能。

主要参考位置：

- `native-app/app/src/main/java/ai/shimei/app/HomeLayout.kt`
- `native-app/feature/scan/src/main/java/ai/shimei/feature/scan/ScanScreen.kt`
- `native-app/feature/scan/src/main/java/ai/shimei/feature/scan/ScanViewModel.kt`
- `native-app/feature/scan/src/main/assets/face-navigator-portrait.png`
- `native-app/feature/scan/src/main/assets/face-navigator-lateralized.png`

### 2.2 当前数据基线

- 面诊证据：扫查帧先进入 `BufferedScanImage`，随后落入 `AestheticReport / LocalReport / ReportDetailUltrasoundImage`，原始 JPEG 通过 `ScanImageStore` 独立落盘。
- 医美注射证据：主要通过 `PausedFeedbackSnapshot` 保存为 MCAP，manifest 中已有顾客、会话、部位、注射参数等信息。
- 两类历史数据目前不是同一种存储结构，因此随访列表不应在点击时临时遍历报告 JSON 和所有 MCAP 文件，应增加统一的“随访参考索引”。
- 已有 `EncounterType.FOLLOW_UP_COMPARE` 表示独立复查业务，但本需求是面诊/注射流程内临时开启和关闭的辅助状态，不建议把当前会话切换成该类型。

## 3. 核心产品决策

### 3.1 体标的绑定粒度

体标应绑定到**具体保存的超声证据帧/注射证据**，不只绑定整次会话。

原因是一次面诊可能包含多个部位，一次注射也可能包含多个注射点。如果整次会话只有一个体标，历史图像与体标很容易错配。

界面中可以保留一个“当前体标草稿”，但每次保存证据时都要复制一份不可变快照到该证据中。后续编辑当前体标不能反向修改已经保存的历史证据。

### 3.2 随访扫查引导的业务性质

新增扫查子状态，而不是新增页面或改变 `EncounterType`：

```text
NormalScan
   ├─ 打开体标编辑器 → BodyMarkerEditing → 确定/取消 → NormalScan
   └─ 选择历史记录 → FollowUpGuiding → 再次点击关闭 → NormalScan
```

随访引导关闭后：

- 恢复进入前的面诊/注射界面布局和控制项；
- 探头视频流、AI 推理和原有采集状态继续存在；
- 当前体标默认沿用所选历史体标的副本；
- 只有用户之后执行原有“保存证据/保存注射数据”动作，体标才随本次数据真正落盘。

### 3.3 “用户”的定义

本方案暂按“用户 = 当前业务顾客 `customerId`”处理，不按设备操作员或登录账号处理。

- 有明确 `customerId`：可以使用随访扫查引导。
- 匿名面诊：允许创建体标并随数据保存，但在关联顾客前不进入其他顾客的历史列表。
- 医美注射当前本就要求关联顾客，直接按该顾客查询。

## 4. 顶部工具设计

### 4.1 工具位置

在【面诊扫查】和【医美注射】中统一增加：

- `体标`
- `随访扫查引导`

普通真实/虚拟探头：放在现有 `ScanInfoBar` 右侧，位于增益入口之前。推荐使用“图标 + 短文字”的紧凑按钮，触摸区域不小于 48 dp。

`SCREEN_CAPTURE`：当前顶部栏被刻意隐藏，不能直接恢复完整顶部栏，否则会破坏现有对齐。两个工具改为画面顶部居中的半透明悬浮工具条，避开左上角返回键和右上角现有显示模式按钮。

### 4.2 按钮状态

**体标按钮**：

- 无体标：默认状态；
- 已有草稿：高亮并显示小圆点；
- 继承历史体标：显示“已沿用”状态；
- 当前草稿有未确认编辑：离开时提示保存或放弃。

**随访扫查引导按钮**：

- 未关联顾客：禁用，提示“请先关联顾客”；
- 无可用历史体标：禁用或点击后展示空状态“暂无带体标的历史数据”；
- 选择中：打开历史选择面板；
- 引导中：按钮高亮并改为“退出引导”，再次点击直接恢复正常扫查。

## 5. 【体标】功能

### 5.1 编辑器入口和容器

点击【体标】后打开大尺寸模态编辑器。平板横屏建议宽度占屏幕 70%～80%，手机使用全屏 Dialog/Sheet。

不建议“点击画面外自动保存”。体标属于需要明确确认的证据数据，外部点击容易造成误保存。建议规则：

- `确定`：保存到当前体标草稿并关闭；
- `取消`：恢复打开编辑器前的草稿；
- 点击外部或系统返回：无修改时直接取消，有修改时询问“放弃修改？”；
- 提供 `清除体标` 和 `撤销`。

### 5.2 人脸简图

第一版提供三个明确视图：

- 正面；
- 左侧；
- 右侧。

现有正面和双侧素材可作为交互原型参考，但正式体标建议使用中性、轮廓清晰、无具体人物身份的简图，避免照片纹理干扰标记。

### 5.3 标记内容

第一版不要只保存一个点，建议使用“探头/切面指示器”：

- 中心位置；
- 切面线段长度；
- 旋转角度；
- 方向箭头；
- 可选备注，如“轻微向上倾斜”。

编辑方式：

1. 选择正面/左侧/右侧；
2. 打开“编辑”Toggle；
3. 点击简图放置指示器；
4. 拖动整体改变位置；
5. 拖动两端改变长度和方向，或使用旋转手柄；
6. 关闭“编辑”Toggle可以预览，避免误触；
7. 点击确定。

第一版每个证据只要求一个主指示器。数据模型可以预留多个指示器，但 UI 暂不开放多标记，避免操作复杂化。

### 5.4 扫查页缩略图

当前存在有效体标时，在**原始图像窗格右下角**显示体标缩略图：

- 推荐占窗格宽度约 20%，最小 96 dp，最大 160 dp；
- 半透明深色底、1 dp 边框，不能遮挡主要超声结构；
- 显示视图和指示器，隐藏编辑控件；
- 点击缩略图重新打开体标编辑器；
- 随访引导模式中，左侧历史窗格显示历史体标，右侧实时窗格显示当前/沿用体标，二者用“历史”“本次”角标区分。

### 5.5 保存语义

- 面诊：点击现有“采集证据/保存当前部位”时，把当前体标快照附加到该批证据帧。若一批保存多帧，同一批帧复制同一体标快照。
- 注射：点击现有“保存注射数据”时，把当前体标快照写入 MCAP manifest/业务元数据，并为该次保存的参考帧建立索引。
- 体标不是必填项，不阻塞现有保存流程；没有体标的数据只是不出现在随访扫查引导的可选列表中。
- 从历史沿用的体标必须生成新的快照，并记录来源 ID，不能与历史对象共用可变引用。

## 6. 【随访扫查引导】功能

### 6.1 历史选择面板

点击工具后，从统一索引查询：

```text
customerId = 当前顾客
bodyMarker != null
referenceFrame 可读取
按 createdAt 倒序
```

列表需要显示：

- 日期和时间；
- 来源标签：面诊 / 医美注射；
- 部位或注射点；
- 历史原图缩略图；
- 体标视图缩略图；
- 可选的质量等级。

需求要求显示所有合格历史数据，因此列表不做强制过滤；为降低选择成本，建议优先排序“同模块 + 同部位/同注射点”，其余数据继续显示在“其他历史记录”分组中。

一次保存可能包含多张原图。随访列表的最小选择单元应是**一张带体标的历史证据帧**，而不是只选择整份报告。界面可先按日期/会话分组，组内再选择具体缩略图。

空状态：

- 无顾客：请先关联顾客；
- 有顾客但无体标历史：暂无可用于引导的历史记录；
- 文件缺失：该历史原图不可用，不允许进入引导并提示数据已损坏/未同步完成。

### 6.2 引导中的画面布局

进入引导后，中部影像区切换为：

```text
┌──────────────────────────┬──────────────────────────┐
│ 历史原始超声             │ 本次实时超声             │
│                          │                          │
│                 历史体标 │                 本次体标 │
└──────────────────────────┴──────────────────────────┘
```

- 左侧固定显示所选历史原始帧，不播放、不做 AI 重算；
- 左下或顶部显示历史日期、来源和部位；
- 左侧右下角显示该历史证据保存时的体标；
- 右侧继续显示当前实时帧；
- 右侧体标默认复制历史体标，用户之后可以通过【体标】重新编辑；
- 底部面诊 `ScanDashboard` 或注射 `InjectionControlPanel` 保持原有功能和状态，不重建会话；
- 平板横屏保持左/右；宽度不足时沿用现有响应式策略改为上/下，避免每个窗格过窄。

### 6.3 AI 叠加显示

引导模式右侧默认显示当前原图，避免 AI 覆盖影响人工比对。

当前 UI 已经有三种可见 AI 标注样式：渐变、线框、实心；同时代码已有但隐藏了 `ORIGINAL`。本功能中将 `ORIGINAL` 显示出来，形成四种选择：

1. 原图 / 关闭 AI 叠加；
2. 渐变；
3. 线框；
4. 实心。

该选择只影响右侧实时图，不能改变左侧历史原图。建议在引导模式中使用会话级临时状态，退出引导后恢复用户进入前的 AI 标注样式，不意外覆盖全局设置。

### 6.4 退出引导

用户再次点击高亮的【随访扫查引导/退出引导】按钮：

- 立即恢复原始的“当前原图 + 当前 AI 解释图”双屏，或对应 `SCREEN_CAPTURE` 布局；
- 不自动保存当前帧；
- 保留当前体标草稿；
- 保留原有部位、注射参数、暂停/运行状态和已缓存帧；
- 之后执行原流程保存时，体标随本次证据保存。

若用户重新选择另一条历史记录，默认用新记录替换左侧历史参考；如果本次体标已被手工修改，弹出“是否用新历史体标覆盖本次体标”的确认，避免丢失编辑。

## 7. 建议数据模型

体标需要同时保存结构化数据和可显示预览：

```kotlin
@Serializable
enum class BodyMarkerView { FRONT, LEFT, RIGHT }

@Serializable
data class NormalizedPoint(
    val x: Float, // 0..1
    val y: Float, // 0..1
)

@Serializable
data class ProbePlaneMarker(
    val center: NormalizedPoint,
    val lengthRatio: Float,
    val rotationDegrees: Float,
    val arrowForward: Boolean = true,
    val note: String = "",
)

@Serializable
data class BodyMarkerSnapshot(
    val schemaVersion: Int = 1,
    val view: BodyMarkerView,
    val marker: ProbePlaneMarker,
    val previewStorageId: String = "",
    val inheritedFromReferenceId: String? = null,
    val updatedAtMillis: Long,
)
```

坐标必须归一化为 0～1，不能保存屏幕像素，否则不同屏幕和横竖屏无法正确回放。

建议同时生成一张小尺寸 PNG/WebP 预览，满足列表和右下角快速显示；结构化数据用于后续编辑、缩放和未来扩展。只保存合成位图会导致历史体标无法继续编辑。

统一随访索引建议为：

```kotlin
@Serializable
data class FollowUpReference(
    val id: String,
    val customerId: String,
    val encounterType: EncounterType,
    val sourceRecordId: String,        // reportId 或 injectionSession/clipId
    val sourceImageStorageId: String,  // 可直接读取的历史原始帧
    val createdAtMillis: Long,
    val regionKey: String? = null,
    val regionLabel: String,
    val bodyMarker: BodyMarkerSnapshot,
    val qualityGrade: String = "",
)
```

索引应在证据保存成功后写入，并在原始记录删除时同步删除。不要只存绝对文件路径，避免应用目录变化后失效。

## 8. 后续实现落点

### 8.1 Core/Data

- 在 `core:model` 增加体标和统一随访参考模型。
- 新增 `BodyMarkerAssetStore` 保存预览图；不要把它混进超声 JPEG 文件命名空间。
- 新增 `FollowUpReferenceRepository`，提供：
  - `referencesForCustomer(customerId)`；
  - `save/reference/delete`；
  - 原图和体标资源存在性校验。
- 数据迁移：旧报告和旧 MCAP 没有体标，按 `null` 兼容，不进入随访列表。

### 8.2 面诊保存链路

- `BufferedScanImage` 增加保存瞬间的 `BodyMarkerSnapshot?`。
- `ReportUltrasoundImage / ReportDetailUltrasoundImage` 增加体标字段或对应引用。
- `AestheticReport.toLocalReport()` 持久化体标。
- 报告成功保存后，为每张带体标且原图可读取的证据写入 `FollowUpReference`。

### 8.3 注射保存链路

- `PausedFeedbackSnapshot` 增加 `BodyMarkerSnapshot?`。
- MCAP manifest 增加体标、参考帧 ID 和预览资源 ID。
- 保存 MCAP 成功后把参考帧独立落到可直接读取的图像存储，并写入统一索引；随访列表不应为了展示一个缩略图读取和解压完整 MCAP。
- `McapIndex.kt` 扩展体标和参考帧元数据，兼容旧 manifest。

### 8.4 Scan UI/ViewModel

`ScanUiState` 建议新增：

```kotlin
val bodyMarkerDraft: BodyMarkerSnapshot? = null
val bodyMarkerDirty: Boolean = false
val scanAssistMode: ScanAssistMode = ScanAssistMode.Normal
val followUpReferences: List<FollowUpReference> = emptyList()
val selectedFollowUpReference: FollowUpReference? = null
val followUpAiMode: AnnotationDisplayMode = ORIGINAL
```

事件至少包括：

- `openBodyMarkerEditor / confirmBodyMarker / cancelBodyMarker / clearBodyMarker`
- `loadFollowUpReferences(customerId)`
- `selectFollowUpReference(reference)`
- `enterFollowUpGuide / exitFollowUpGuide`
- `setFollowUpAiMode(mode)`

`ScanScreen` 增加顶部工具条、体标缩略图、历史选择面板和随访双屏布局。两种业务继续使用同一组组件，只根据 `EncounterType` 调整历史来源标签和底部控制区。

## 9. 实施顺序

### 阶段 A：模型、存储与索引

1. 定义体标结构化模型和版本号。
2. 实现预览资源存储。
3. 实现统一随访参考仓库。
4. 增加旧数据兼容和资源删除测试。

### 阶段 B：体标编辑与保存

1. 增加顶部【体标】按钮。
2. 实现正面/左侧/右侧编辑器。
3. 实现原始图右下角缩略图。
4. 接入面诊保存链路。
5. 接入注射 MCAP 保存链路。

### 阶段 C：随访扫查引导

1. 增加顶部【随访扫查引导】按钮和可用性状态。
2. 实现历史日期/证据选择面板。
3. 实现左历史、右实时布局。
4. 增加“原图、渐变、线框、实心”四种右侧显示状态。
5. 实现退出后恢复原流程和沿用体标。

### 阶段 D：同步与完善

1. 把体标和随访索引纳入现有同步协议；需要跨设备随访时，补充 backend/proto 字段。
2. 增加报告详情中的体标查看入口。
3. 验证文件清理、报告删除、顾客合并/认领后的索引一致性。
4. 真机验证横屏、竖屏、真实探头、虚拟探头和 `SCREEN_CAPTURE`。

## 10. 验收标准

### 10.1 体标

- 面诊扫查和医美注射均能看到【体标】工具。
- 可选择正面、左侧、右侧，放置并编辑切面指示器。
- 确定后原始图右下角出现缩略图；取消不改变原草稿。
- 保存面诊证据或注射数据后，重新打开应用仍可恢复对应体标。
- 同一会话不同部位/注射点可以保存不同体标，互不覆盖。
- 旧数据无体标时可以正常读取和展示。

### 10.2 随访扫查引导

- 只显示当前顾客且带有效体标和可读取原图的历史证据。
- 历史列表按时间倒序，并区分面诊/注射和部位/注射点。
- 选择后左侧为历史原图和历史体标，右侧为实时图和本次体标。
- 右侧可在原图、渐变、线框、实心之间切换；左侧始终保持历史原图。
- 退出引导后恢复进入前的扫查布局、部位、注射参数和运行状态。
- 退出引导不会自动保存；之后按原流程保存时沿用的体标进入本次证据。
- 匿名状态不能读取其他顾客历史。

### 10.3 工程质量

- 所有新增交互控件有 `data-testid/testTag` 和可访问名称，触摸目标不小于 48 dp。
- ViewModel 状态转移、仓库查询、旧数据兼容和体标复制语义有单元测试。
- Compose 测试覆盖两个业务入口、历史空状态、进入/退出引导和四种显示状态。
- 删除报告/MCAP 后不留下可点击但无法读取的随访记录。

## 11. 风险与建议

1. **现有正面/侧面素材是照片式人脸**：可快速原型，但正式体标建议换成中性简图。
2. **不要只保存合成后的体标图片**：必须同时保存结构化坐标，否则无法再次编辑，也不利于后续智能化。
3. **不要把体标只挂在整份报告上**：多部位数据会错配，应挂在具体证据帧。
4. **不要把随访引导实现为 `FOLLOW_UP_COMPARE` 新会话**：本需求需要随时退出并继续原面诊/注射流程。
5. **`SCREEN_CAPTURE` 可用宽度可能很窄**：在 Android 分屏下强制左/右双栏会影响可读性，必须保留窄屏上/下布局或全屏历史参考切换方案。
6. **跨设备历史依赖同步**：第一版若只做本机索引，要在产品文案中明确；若要求任意设备查看同一顾客历史，必须同步体标结构、预览和原始参考帧。
7. **功能命名保持“引导”**：不使用“精准对齐”“同一切面”等承诺性表达。

## 12. 实施前需要确认的问题

1. “隶属于该用户”是否确定指顾客 `customerId`，而不是操作医生/登录账号？本方案按顾客处理。
2. 随访历史是否允许跨模块选择，例如在医美注射中引用以前的面诊体标？本方案允许，并优先展示同模块、同部位记录。
3. 体标第一版是否只需要一个探头/切面指示器？本方案建议一个；若要同时画多个注射点，需要提前调整编辑器交互。
4. 历史面诊一次可能保存多张原图，是否接受“按日期分组后选择具体历史帧”？本方案认为必须选择到具体帧，才能保证体标与原图对应。
5. 是否要求体标历史跨设备可用？如果要求，实施范围必须包含 backend/proto 和资源同步，不能只修改 native UI。

