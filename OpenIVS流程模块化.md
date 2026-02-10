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
| image | RGB 图像（Mat） | Camera -> 推理 -> 存图 |
| result | 结构化结果（JObject） | 推理 -> 匹配 -> 存储 |
| template | 模版对象（JObject） | Load -> Match |
| bool | 布尔值 | ok/ng 判定传递 |
| int | 整数 | sequence 等 |

## 3. 连接规则与行为

### 连接约束
1. 连线只能从输出端口到输入端口，编辑器仅允许相同 data_type 的端口互连
2. 单连接输入端口再次连线时，自动断开旧连线
3. 图内禁止环路（拓扑排序时检测）

### 缺失输入时的运行行为
- 端口未连接时，节点仍可执行，按以下规则处理：
  - `image`：跳过该节点，不产生输出
  - `result`：使用空 JObject
  - `bool`：使用默认值（true/false 由模块定义）
  - `string`：使用空字符串
  - `template`：跳过匹配
- 节点的所有关键输入均缺失时，整个节点跳过，下游按缺失处理

### 编辑器提示
- 推荐连接未接时，编辑器用虚线/灰色标记提示用户，但不阻止保存和运行

 
## 4. 主图模块库

### 4.1 camera（相机采集）
触发拍照或读取测试图片，输出图像与初始结果通道。

| | 端口名 | 类型 |
|---|--------|------|
| 出 | image | image |


属性：
- `device`: 选择物理相机(序列号或IP，用于连接)

### 4.2 plc/read（PLC读取）

从PLC寄存器读取数据。

| | 端口名 | 类型 |
|---|--------|------|
| 出 | value | string |
| 出 | success | bool |

属性：
- `address`：PLC寄存器地址
- `count`：读取的字节/字数量
- `encoding`：字符编码（UTF-8/ASCII/GB2312等）

### 4.3 plc/write（PLC写入）

向PLC寄存器写入检测结果。

| | 端口名 | 类型 |
|---|--------|------|
| 入 | value | bool |
| 出 | success | bool |

属性：
- `address`：起始寄存器地址
- `count`：寄存器数量
- `ok_value`：value=true时写入的值
- `ng_value`：value=false时写入的值
- `timeout_ms`：应答超时时间

### 4.4 inference/subgraph（推理子图）

引用并执行一个推理子图 JSON。这是主图与子图的桥梁——双击此节点可打开推理子图编辑器。

| | 端口名 | 类型 |
|---|--------|------|
| 入 | image | image |
| 出 | image | image |
| 出 | result | result |
| 出 | ok_raw | bool |

属性：
- `subgraph_path`：推理子图 JSON 路径

执行时加载子图 JSON，委托给已有的 `GraphExecutor` 执行，将 image/result 传入子图、收回结果。所有后处理与 OK/NG 判定（缺陷尺寸过滤、缝隙阈值、模版匹配等）均在子图内部完成，主图只接收最终的 ok_raw 和 result。子图模块定义见 `流程图.md`。

### 4.5 template/load（模版加载）

按产品型号和位置加载模版。

| | 端口名 | 类型 |
|---|--------|------|
| 入 | product_type | string |
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
| 入 | result | result |
| 出 | result | result |
| 出 | ok_raw | bool |

属性：
- `position_tolerance_x`、`position_tolerance_y`
- `size_tolerance`
- `min_confidence`


### 4.7 template/create（模版创建）

从当前推理结果和图像创建模版并保存。

| | 端口名 | 类型 |
|---|--------|------|
| 入 | image | image |
| 入 | result | result |
| 入 | product_type | string |
| 入 | should_create | bool（可选，默认 true） |
| 出 | template | template |
| 出 | ok_raw | bool |

属性：
- `save_image`：是否保存参考图

`should_create` 为 false 时跳过创建。典型用法是接 template/load 的 `has_template` 取反，实现"没有模版时自动创建"。

模版创建有两个入口，共享同一个 **TemplateManager 单例**：
- **流程图内**：template/create 模块在 RunOnce 过程中按条件自动执行
- **UI 模版管理窗口**：用户手动从当前检测结果或本地图片创建/编辑/删除

两个入口操作同一份模版数据，流程图创建的模版在 UI 中可见，UI 创建的模版在下次 RunOnce 时 template/load 可加载。

## 5. 执行引擎设计

主图的执行引擎负责按拓扑关系调度所有节点，核心是**数据驱动 + 调度约束**。

### 5.1 执行流程
1. 加载主图 JSON，构建节点和边的拓扑图
2. 校验（环路检测、必填端口、类型匹配）
3. 从 Camera 节点开始执行
4. 节点执行完后，将输出数据分发到所有下游节点的输入缓冲区
5. 当一个节点的所有 required 输入都就绪时（collector 端口需所有入边都到齐），将该节点加入**就绪队列**
6. 从就绪队列取节点执行，重复直到所有节点完成

### 5.2 调度约束
- **Camera 调度**：多个 Camera 节点支持串行或并行两种模式，由引擎全局配置 `camera_schedule` 决定：
  - `serial`（默认）：逐个执行，每个 Camera 取图完成后立即释放下游，再执行下一个。
  - `parallel`：所有 Camera 同时取图。
- **推理并行**：多个 inference/subgraph 节点可并行执行
- **merge 等待**：collector 端口的节点需等待所有入边数据到齐齐

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