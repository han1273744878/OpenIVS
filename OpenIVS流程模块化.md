# 主流程图模块化设计

本文档定义主流程图的模块系统设计，使所有主流程步骤（触发、采集、模版、判定、输出等）都能在编辑器中自由拖拽和连接。推理子图的模块定义见 `流程图.md`。

## 1. 模块统一结构

每个主图模块是一个节点，由以下部分组成：
- **type**：模块类型标识，如 `trigger/plc`、`camera`
- **inputs[]**：输入端口列表，每个端口定义 `name`、`data_type`、`required`
- **outputs[]**：输出端口列表，每个端口定义 `name`、`data_type`
- **properties{}**：模块配置参数，由属性面板编辑

所有模块实现统一运行时接口，接收上游数据、执行逻辑、将结果写入输出端口。

## 2. 端口与数据类型

| 类型 | 含义 | 典型用途 |
|------|------|----------|
| signal | 触发信号 | Trigger -> Camera |
| image_chan | BGR 图像（Mat） | Camera -> 推理 -> 存图 |
| result_chan | 结构化结果（JObject） | 推理 -> 匹配 -> 存储 |
| template | 模版对象 | Load -> Match |
| bool | 布尔值 | ok/ng 判定传递 |
| string | 字符串 | product_type、position 等 |
| int | 整数 | sequence 等 |

端口分为两种模式：
- **单连接端口**（默认）：一个输入端口只能接收一条连线
- **收集端口（collector）**：可接收多条同类型连线，运行时收齐所有输入后才执行（用于 merge 等聚合节点）

## 3. 连接规则

1. 只能从输出端口连到输入端口，且两端 data_type 必须相同
2. 单连接输入端口只允许一条入边；收集端口允许多条
3. 一个输出端口可连接多个下游输入端口（扇出）
4. `required = true` 的输入端口必须有连接，否则校验失败
5. 图内禁止环路
6. Trigger 节点必须唯一，且作为图的入口
7. 至少存在一个 output/* 节点

## 4. 主图模块库

### 4.1 trigger（触发）

图的唯一入口，产生启动信号。

| | 端口名 | 类型 |
|---|--------|------|
| 出 | signal | signal |

属性：
- `proto`：plc / software
- PLC 模式：`signal_address`、`poll_ms`、`edge`（rising/falling）
- 软件模式：`interval_ms`

约束：全图只能有一个 trigger 节点。

### 4.2 camera（相机采集）

触发拍照或读取测试图片，输出图像与初始结果通道。

| | 端口名 | 类型 |
|---|--------|------|
| 入 | signal | signal |
| 出 | image_chan | image_chan |
| 出 | result_chan | result_chan |
| 出 | position | string |

属性：
- `mode`：Real / Test
- `device_config`：设备配置（Real 模式）
- `image`：测试图片路径或目录（Test 模式）
- `loop`：是否循环（Test 模式）
- `interval_ms`：采集间隔（Test 模式）
- `position`：摄像头位置标识
- `rotation`：旋转角度（0/90/180/270）
- `trigger_mode`：SoftTrigger / HardTrigger

约束：
- 多个 Camera 节点由执行引擎强制按 position 配置顺序**串行执行**，避免网络丢包
- 每个 Camera 的 position 不能重复

### 4.3 input/data（数据输入）

从外部读取一个字符串值，可用于获取产品型号等信息。

| | 端口名 | 类型 |
|---|--------|------|
| 入 | signal | signal |
| 出 | value | string |
| 出 | success | bool |

属性：
- `source`：plc_register / manual / constant
- `address`、`count`、`encoding`（PLC 模式）
- `default_value`（手动/常量模式）

### 4.4 inference/subgraph（推理子图）

引用并执行一个推理子图 JSON。这是主图与子图的桥梁——双击此节点可打开推理子图编辑器。

| | 端口名 | 类型 |
|---|--------|------|
| 入 | image_chan | image_chan |
| 入 | result_chan | result_chan（可选） |
| 入 | position | string（可选） |
| 出 | image_chan | image_chan |
| 出 | result_chan | result_chan |
| 出 | ok_raw | bool |

属性：
- `subgraph_path`：推理子图 JSON 路径

执行时加载子图 JSON，委托给已有的 `GraphExecutor` 执行，将 image_chan/result_chan 传入子图、收回结果。所有后处理与 OK/NG 判定（缺陷尺寸过滤、缝隙阈值、模版匹配等）均在子图内部完成，主图只接收最终的 ok_raw 和 result_chan。子图模块定义见 `流程图.md`。

### 4.5 template/load（模版加载）

按产品型号和位置加载模版。

| | 端口名 | 类型 |
|---|--------|------|
| 入 | product_type | string |
| 入 | position | string |
| 出 | template | template |
| 出 | has_template | bool |

属性：
- `templates_path`：模版根目录
- `allow_historical_unknown`：是否加载历史 Unknown 模版

### 4.6 template/match（模版匹配）

将推理结果与模版对比，输出匹配判定。

| | 端口名 | 类型 |
|---|--------|------|
| 入 | template | template |
| 入 | result_chan | result_chan |
| 出 | result_chan | result_chan |
| 出 | ok_raw | bool |

属性：
- `position_tolerance_x`、`position_tolerance_y`
- `size_tolerance`
- `min_confidence_threshold`
- `text_similarity_threshold`
- `min_match_score`

匹配规则详见 `功能.md` 5.7 节。

### 4.7 template/create（模版创建）

从当前推理结果和图像创建模版并保存。

| | 端口名 | 类型 |
|---|--------|------|
| 入 | image_chan | image_chan |
| 入 | result_chan | result_chan |
| 入 | product_type | string |
| 入 | position | string |
| 入 | should_create | bool（可选，默认 true） |
| 出 | template_id | string |
| 出 | success | bool |

属性：
- `save_image`：是否保存参考图

`should_create` 为 false 时跳过创建。典型用法是接 template/load 的 `has_template` 取反，实现"没有模版时自动创建"。

模版创建有两个入口，共享同一个 **TemplateManager 单例**：
- **流程图内**：template/create 模块在 RunOnce 过程中按条件自动执行
- **UI 模版管理窗口**：用户手动从当前检测结果或本地图片创建/编辑/删除

两个入口操作同一份模版数据，流程图创建的模版在 UI 中可见，UI 创建的模版在下次 RunOnce 时 template/load 可加载。

### 4.8 result/merge_positions（多位置结果汇总）

收集所有相机位置的判定结果，汇总为最终 OK/NG。

| | 端口名 | 类型 | 模式 |
|---|--------|------|------|
| 入 | ok_raw | bool | collector |
| 入 | result_chan | result_chan | collector |
| 入 | position | string | collector |
| 出 | overall_ok | bool | - |
| 出 | position_statuses | result_chan | - |
| 出 | first_ng_reason | string | - |

属性：`any_ng_is_ng`（默认 true）

collector 端口：接收多条连线，执行引擎根据入边数量判断"收齐"条件，全部到齐后才执行此节点。用户每连接一路结果，merge 就自动多等一路。

### 4.9 decision/force_ok（空跑判定）

空跑模式下将写回结果强制为 OK。

| | 端口名 | 类型 |
|---|--------|------|
| 入 | ok_in | bool |
| 出 | ok_send | bool |

属性：`dry_run`（是否空跑，引用全局设置）

规则：dry_run = true -> ok_send = true；否则 ok_send = ok_in。

### 4.10 output/*（输出模块）

**output/modbus_write_block**

| | 端口名 | 类型 |
|---|--------|------|
| 入 | ok_send | bool |
| 出 | success | bool |

属性：`start_address`、`count`、`ok_value`、`ng_value`、`ack_value`、`timeout_ms`

**output/save_image**

| | 端口名 | 类型 |
|---|--------|------|
| 入 | image_chan | image_chan |
| 入 | overall_ok | bool |
| 入 | product_type | string |
| 入 | position | string |
| 出 | image_path | string |

属性：`save_root`、`save_ok_images`、`save_ng_images`、`ok_format`、`ng_format`、`jpg_quality`

**output/save_json**

| | 端口名 | 类型 |
|---|--------|------|
| 入 | result_chan | result_chan |
| 入 | overall_ok | bool |
| 入 | product_type | string |
| 入 | position | string |
| 出 | json_path | string |

属性：`save_root`、`save_ok_results`、`save_ng_results`

**output/save_csv**

| | 端口名 | 类型 |
|---|--------|------|
| 入 | overall_ok | bool |
| 入 | product_type | string |
| 入 | position_statuses | result_chan |
| 入 | first_ng_reason | string |
| 出 | csv_path | string |

属性：`save_root`

**output/save_visualization**

| | 端口名 | 类型 |
|---|--------|------|
| 入 | image_chan | image_chan |
| 入 | result_chan | result_chan |
| 出 | vis_path | string |

属性：`suffix`、`jpg_quality`

## 5. 执行引擎设计

主图的执行引擎负责按拓扑关系调度所有节点，核心是**数据驱动 + 调度约束**。

### 5.1 执行流程
1. 加载主图 JSON，构建节点和边的拓扑图
2. 校验（环路检测、必填端口、类型匹配）
3. 从 Trigger 节点开始执行
4. 节点执行完后，将输出数据分发到所有下游节点的输入缓冲区
5. 当一个节点的所有 required 输入都就绪时（collector 端口需所有入边都到齐），将该节点加入**就绪队列**
6. 从就绪队列取节点执行，重复直到所有节点完成

### 5.2 调度约束
- **Camera 串行**：多个 Camera 节点不并发执行，按 position 配置顺序逐个执行。每个 Camera 取图完成后立即释放下游（inference/subgraph 可开始），再执行下一个 Camera
- **推理并行**：多个 inference/subgraph 节点可并行执行
- **merge 等待**：collector 端口的节点需等待所有入边数据到齐
- **output 并行**：多个 output 节点可并行执行

### 5.3 流水线效果
Camera 串行 + 推理并行 + 数据驱动，自然形成流水线：
```
Trigger:    [触发]
Camera_A:          [==取图==]
Camera_B:                     [==取图==]
Camera_C:                                [==取图==]
Infer_A:                      [==========推理==========]
Infer_B:                                  [==========推理==========]
Infer_C:                                             [==========推理==========]
Merge:                                                                         [汇总]
Output:                                                                                [存图/写PLC/...]
```
不需要硬编码流水线逻辑，由执行引擎的调度约束自然实现。

### 5.4 错误处理
- 节点执行失败 -> 标记 success=false，输出携带错误信息
- 下游节点仍可收到数据（含错误标记），自行决定是否继续
- 最终 output 节点输出可追溯的完整结果

## 6. 典型连接示例

### 6.1 基础多相机检测

以 3 路相机检测为例，用户在编辑器中拖拽模块并连线：

```
                                                 +-> template/load --template-> template/match --ok_raw->+
trigger(PLC) --signal-> camera(A) --image/result-> inference/subgraph --result-+                          |
             --signal-> camera(B) --image/result-> inference/subgraph --ok_raw-------------------------->+-> merge --overall_ok-> decision --ok_send-> output/modbus
             --signal-> camera(C) --image/result-> inference/subgraph --ok_raw-------------------------->+         |
input/data --product_type-> template/load                                                                          +-> output/save_image
                                                                                                                   +-> output/save_json
                                                                                                                   +-> output/save_csv
```

- Trigger 的 signal 扇出到 3 个 Camera
- 每路 Camera -> inference/subgraph，子图内部完成推理、后处理和 OK/NG 判定
- A 路需要模版匹配：inference/subgraph 输出 result_chan -> template/match -> ok_raw
- B/C 路不需要模版匹配：直接使用 inference/subgraph 输出的 ok_raw
- 所有 ok_raw 汇入 merge_positions 的 collector 端口
- merge 后接 decision 和多个 output

用户可以自由调整：增减相机数量、决定哪些位置需要模版匹配、增减 output 节点。所有后处理在推理子图内部完成，主图只负责串联采集、推理、汇总、输出。

### 6.2 带模版自动创建的连接

在示例 6.1 的基础上，A 路增加"没有模版时自动创建"的逻辑：

```
input/data --product_type--+-> template/load --has_template-> NOT --should_create->+
                           |   |                                                    |
                           |   +--template-> template/match --ok_raw-> merge        |
                           |                 ^ result_chan                           v
                           +-> template/create <-- image_chan <-- inference/subgraph <-- camera(A)
                                ^ position        ^ result_chan
                                +-- camera(A)     +-- inference/subgraph
```

执行流程：
1. input/data 获取 product_type，camera(A) 采集图像
2. inference/subgraph 完成推理，输出 result_chan 和 image_chan
3. template/load 按 product_type + position 查找模版
4. 若 has_template=true：template/match 执行匹配，ok_raw 传入 merge
5. 若 has_template=false：NOT 输出 true，template/create 从推理结果创建模版
6. 后续 RunOnce 中 template/load 能加载到刚创建的模版，自动进入匹配流程

用户也可以不在流程图中放 template/create，改为通过 UI 模版管理窗口手动创建，两者共享同一个 TemplateManager，效果相同。
