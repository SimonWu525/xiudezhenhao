# 老照片修复与上色工具 技术方案设计（V1.0）

- 关联 PRD：`docs/old-photo-restore-prd-v1.0.md`
- 文档目标：将 PRD 转换为可开发、可联调、可测试的技术实现蓝图
- 适用范围：MVP（单图上传、两类任务、单页流程）
- 更新时间：2026-03-11

---

## 1. 设计目标与边界

### 1.1 设计目标

- 在不偏离 PRD 的前提下，明确前后端接口和职责边界
- 定义端到端处理链路，保证“上传 → 处理 → 预览 → 下载”主链稳定
- 对失败路径做统一错误码和重试策略，避免前后端语义不一致
- 通过日志与指标设计保障可观测性，为上线后优化提供依据

### 1.2 本期边界（MVP）

- 单页单图单任务
- 任务类型：`restore` / `colorize`
- 同步请求（前端等待结果返回）
- 结果临时存储（24 小时）
- 不做用户系统、历史记录、批量任务、异步队列

---

## 2. 总体架构设计

## 2.1 逻辑架构

```text
[Browser SPA]
    |
    | multipart/form-data
    v
[API Gateway / BFF]
    |
    +--> [Input Validator]
    |
    +--> [Image Preprocessor]
    |
    +--> [Model Service Adapter]
    |
    +--> [Result Storage]
    |
    +--> [Response Composer]
    |
    +--> [Logging & Metrics]
```

### 2.2 模块职责

1. **Input Validator**
   - 校验 MIME、大小、分辨率、任务类型
   - 生成/透传 `requestId`
2. **Image Preprocessor**
   - 图片解码、基础标准化（方向修正、颜色空间统一）
   - 转模型可接受的输入格式
3. **Model Service Adapter**
   - 根据 `taskType` 选择 Prompt 模板
   - 调用图像模型接口
   - 校验模型返回是否为可解码图像
4. **Result Storage**
   - 保存结果图到对象存储（TTL=24h）
   - 产出 `resultImageUrl`
5. **Response Composer**
   - 输出统一成功/失败响应
6. **Logging & Metrics**
   - 记录结构化日志、埋点指标、错误码分布

---

## 3. 前端设计

## 3.1 页面结构

- `PageHeader`
- `UploadPanel`
- `ImagePreview`
- `ActionButtons`
- `ProcessingStatus`
- `ResultViewer`
- `ErrorBanner`

## 3.2 状态机（核心）

```text
IDLE
  -> (upload_success) UPLOADED
UPLOADED
  -> (click_restore/click_colorize) PROCESSING
PROCESSING
  -> (request_success) SUCCESS
  -> (request_failed) FAILED
FAILED
  -> (retry) PROCESSING
SUCCESS
  -> (reupload) UPLOADED
ANY
  -> (reset) IDLE
```

### 3.3 前端数据模型（建议）

```ts
type TaskType = 'restore' | 'colorize'
type PageState = 'IDLE' | 'UPLOADED' | 'PROCESSING' | 'SUCCESS' | 'FAILED'

interface PhotoTaskState {
  pageState: PageState
  requestId?: string
  taskType?: TaskType
  originalFile?: File
  originalPreviewUrl?: string
  resultImageUrl?: string
  errorCode?: string
  errorMessage?: string
  startedAt?: number
  endedAt?: number
}
```

### 3.4 交互细节

- 上传成功后滚动到预览区（`scrollIntoView`）
- `PROCESSING` 时禁用上传与任务按钮
- 失败态保留原图，显示“再试一次”
- 成功态支持下载和重新上传

---

## 4. 后端设计

## 4.1 API 契约

### Endpoint

- `POST /api/photo/restore`
- `Content-Type: multipart/form-data`

### Request Params

- `file`（必填）
- `taskType`（必填，`restore | colorize`）
- `requestId`（可选，若无则后端生成）

### Success Response

```json
{
  "success": true,
  "requestId": "req_20260311_001",
  "taskType": "restore",
  "resultImageUrl": "https://cdn.xxx.com/result/restore_20260311_001.png",
  "width": 2048,
  "height": 2048
}
```

### Error Response

```json
{
  "success": false,
  "requestId": "req_20260311_001",
  "errorCode": "MODEL_PROCESS_FAILED",
  "message": "当前处理失败，请稍后重试"
}
```

## 4.2 后端流程伪代码

```text
receive multipart request
  -> resolve requestId
  -> validate file type/size/taskType
  -> decode image and validate dimensions
  -> preprocess image
  -> select prompt by taskType
  -> call model service with image + prompt
  -> verify model output image bytes
  -> upload result image to object storage (ttl=24h)
  -> return success payload
on any known error
  -> map to standard errorCode + user message
  -> return failure payload
always
  -> write structured log + latency metrics
```

## 4.3 错误码映射

| 场景 | errorCode | 前端提示 |
| --- | --- | --- |
| MIME 非法 | INVALID_FILE_TYPE | 文件格式不支持，请重新上传 |
| 文件超限 | FILE_TOO_LARGE | 图片过大，请上传 15MB 以内图片 |
| 解码失败 | IMAGE_DECODE_FAILED | 图片解析失败，请更换图片 |
| 模型失败 | MODEL_PROCESS_FAILED | 当前处理失败，请稍后重试 |
| 返回空图 | RESULT_EMPTY | 当前处理失败，请稍后重试 |
| 超时 | REQUEST_TIMEOUT | 服务繁忙，请稍后再试 |

---

## 5. 模型接入设计

## 5.1 Prompt 策略

- `restore`：使用“修复”模板
- `colorize`：使用“修复并上色”模板
- 两者均需强调“身份、服饰、姿态、构图不变”

## 5.2 调用约束

- 入参必须含用户原图字节 + MIME + taskType
- 出参必须是单张可解码图片
- 禁止文本替代图像返回

## 5.3 结果质量守卫（服务端）

- 返回体非空
- 图像可解码
- 分辨率大于最小阈值
- 不通过则转 `RESULT_EMPTY` 或 `MODEL_PROCESS_FAILED`

---

## 6. 存储与安全

## 6.1 存储策略

- 原图：仅内存态处理，不落持久化
- 结果图：对象存储，TTL 24h
- 日志：服务端持久化，保留排障字段

## 6.2 安全约束

- 服务端再次校验 MIME，不信任前端
- 对上传文件做解码级校验，防止伪造扩展名
- 下载链接采用随机对象路径，不暴露内部存储结构

---

## 7. 可观测性设计

## 7.1 结构化日志字段

- `requestId`
- `taskType`
- `fileName`
- `fileSize`
- `mimeType`
- `width`, `height`
- `modelName`
- `latencyMs`
- `status`
- `errorCode`

## 7.2 指标（Metrics）

- `upload_success_rate`
- `task_submit_rate`
- `task_success_rate`
- `p50/p90/p95_latency_ms`
- `download_conversion_rate`
- `error_rate`
- `error_code_distribution`

## 7.3 告警建议

- `task_success_rate` 低于阈值告警
- `REQUEST_TIMEOUT` 占比异常升高告警
- `p95_latency_ms` 超过 8s 告警

---

## 8. 测试与验收设计

## 8.1 接口测试（后端）

- 正常：`restore` / `colorize` 各 1 例
- 异常：非法类型、超大文件、损坏图片、模型空返回、超时

## 8.2 前端交互测试

- 按钮禁用时机正确
- 失败后可重试且保留原图
- 成功后下载命名符合规则

## 8.3 E2E 验收清单

- 上传 → 处理 → 结果展示 → 下载完整可达
- 两种任务均可成功返回结果图
- 错误场景文案准确，不暴露技术栈细节

---

## 9. 研发落地计划（工程视角）

1. 第 1 周：前端单页状态机 + 上传校验 + API 桩
2. 第 2 周：后端校验/预处理/统一响应 + 模型适配器
3. 第 3 周：联调 + 错误码对齐 + 监控指标打通
4. 第 4 周：质量测试 + 性能压测 + 上线检查

---

## 10. 风险与缓解

- **风险 1：模型耗时波动大**
  - 缓解：加超时控制、优化输入尺寸策略、做 p95 监控
- **风险 2：历史照片质量跨度大导致失败率上升**
  - 缓解：增加预检与错误提示分流，引导用户更换样本
- **风险 3：上色结果主观评价不一致**
  - 缓解：明确“自然还原”标准，先稳定默认策略，不做多风格

---

## 11. 版本演进建议（V1.1+）

- 异步任务 + 轮询/回调
- 历史记录与结果管理
- 局部区域精修
- 批量处理
- 多风格上色可选项

