# nuScenes JSON 关系入门教程（给新手）

> 目标：用一篇文档，快速理解 nuScenes 里常见 JSON 表是什么、怎么互相关联、以及如何从一个 `sample` 一路找到传感器数据和标注。

---

## 1. 先建立一个整体观

nuScenes 的标注数据不是一个大 JSON，而是很多“表”（table），每个表都是一个 JSON 数组。
每一行记录都有一个全局唯一主键：`token`。

你可以把它理解成一个轻量的关系型数据库：

- `sample`：关键帧（2Hz）
- `sample_data`：某个传感器在某个时刻的具体数据（图像/点云/雷达）
- `sample_annotation`：3D 框标注
- `instance`：同一目标在一个 scene 内的轨迹身份
- `scene`：约 20 秒片段
- `log`：原始采集日志
- `sensor` / `calibrated_sensor` / `ego_pose`：传感器与位姿
- `category` / `attribute` / `visibility`：语义字典表
- `map`：地图资源入口

官方 schema 文档也采用这个表结构描述。建议把“`token` 像主键，`xxx_token` 像外键”作为第一原则。  

---

## 2. 最重要的 4 张表（先学这四张）

### 2.1 `scene`：一个驾驶片段

`scene` 表描述一个时序片段，关键字段：

- `first_sample_token`
- `last_sample_token`
- `nbr_samples`

也就是说：scene 本身不存每一帧内容，而是告诉你“从哪一帧开始，到哪一帧结束”。

### 2.2 `sample`：关键帧骨架

`sample` 记录每个关键帧，并且有：

- `prev` / `next`：串成链表
- `scene_token`：属于哪个 scene

你可以把 `sample` 想成“索引帧”，它不直接放图像点云，而是把数据分发给 `sample_data`。

### 2.3 `sample_data`：真正的数据文件入口

`sample_data` 是最常用的一张表，因为里面有：

- `filename`：文件相对路径
- `calibrated_sensor_token`：该文件来自哪个传感器标定
- `ego_pose_token`：车体在该时刻的位置与朝向
- `is_key_frame`：是不是关键帧
- `prev` / `next`：同一传感器时间链

一句话：**想拿到图像/点云文件路径，最终都要到 `sample_data.filename`。**

### 2.4 `sample_annotation`：3D 框标注

`sample_annotation` 是检测任务最核心的 GT 表，包含：

- `sample_token`：属于哪一帧
- `instance_token`：属于哪个目标轨迹
- `translation / size / rotation`：框参数
- `num_lidar_pts / num_radar_pts`
- `prev` / `next`：同一目标跨时间链

一句话：**按帧看标注，用 `sample_token`；按目标轨迹看标注，用 `instance_token`。**

---

## 3. 三条最常见“查询链路”

下面这三条链路，覆盖 80% 新手需求。

### 链路 A：从 `scene` 遍历所有关键帧

1. 从 `scene.first_sample_token` 开始。
2. 不断沿着 `sample.next` 走到结尾。

适合做：可视化播放、按场景导出。

### 链路 B：从一个 `sample` 找到多传感器数据

1. 找到当前 `sample`。
2. 用 `sample_data` 中 `sample_token == 当前sample.token` 且 `is_key_frame=True` 的记录。
3. 每条 `sample_data` 再通过 `calibrated_sensor_token -> sensor_token` 找到传感器通道（如 `CAM_FRONT` / `LIDAR_TOP`）。

适合做：融合任务的数据读取。

### 链路 C：从一个 `sample` 找到该帧所有 3D 框

1. 取 `sample_annotation` 中 `sample_token == 当前sample.token` 的所有记录。
2. 每个 annotation 再连到 `instance` 看轨迹身份。
3. `instance.category_token -> category` 得到类别名。

适合做：检测/跟踪 GT 构建。

---

## 4. 关系图（文字版）

```text
log ──1:N── scene ──1:N── sample ──1:N── sample_data
                          └──1:N── sample_annotation ──N:1── instance ──N:1── category

sample_data ──N:1── ego_pose
sample_data ──N:1── calibrated_sensor ──N:1── sensor
sample_annotation ──N:M── attribute
sample_annotation ──N:1── visibility
log ──N:1── map (通过 log_tokens 反查)
```

> 注意：`sample_annotation` 与 `attribute` 在逻辑上是多对多（一个框可有多个属性 token）。

---

## 5. 一个“最小可运行”的理解顺序（推荐）

如果你是第一次接触，建议按这个顺序理解：

1. 只看 `scene` + `sample`，先跑通时序。
2. 再加 `sample_data`，能拿到每个通道文件路径。
3. 再加 `sample_annotation`，把框画到可视化里。
4. 最后补 `ego_pose`、`calibrated_sensor` 做坐标变换。

这样学习曲线最平滑，不容易被坐标系细节劝退。

---

## 6. 与 nuScenes-devkit 代码如何对应

在 devkit 中，`NuScenes` 类会把这些 JSON 表全部读入，并建立 `token -> index` 的反向索引。
因此你在代码里常见的是：

- `nusc.get(table_name, token)`：按 token O(1) 查记录
- `nusc.field2token(table, field, value)`：按字段线性查 token 列表

理解这一层后，你就能很自然地把“表关系”映射到“API 调用链”。

---

## 7. 新手最容易踩的坑

1. **把 `sample` 和 `sample_data` 混淆**：前者是关键帧索引，后者才是具体传感器文件。
2. **忽略 `is_key_frame`**：很多任务默认只用关键帧。
3. **把 `instance` 当全局 ID**：instance 不跨 scene。
4. **路径拼接错误**：`sample_data.filename` 是相对 `dataroot` 的相对路径。
5. **先做坐标变换再确认链路**：建议先把 token 关联跑通，再处理坐标系。

---

## 8. 一句话总结

把 nuScenes JSON 想成“多张表 + token 外键”的小数据库：
**先抓住 `scene -> sample -> sample_data / sample_annotation` 这条主干，再补传感器和位姿表**，你就能快速上手几乎所有任务。
