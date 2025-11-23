# 功能开发文档：添加 Model 字段

> **Audience**: 开发者 | **Status**: 待实现 | **Created**: 2025-11-22

## 功能概述

### 背景

当前每个监控任务的模型名称嵌入在 `body` 字段的 JSON 请求体中，无法在前端界面直接展示。用户需要查看配置文件才能知道每个监控点测试的是哪个模型。

### 目标

1. 在 `monitors` 配置中新增 `model` 字段，允许用户指定测试的模型名称
2. 在前端监控列表中展示该字段，位置在"通道"和"当前状态"之间
3. 字段为可选，与 `channel` 字段保持一致的设计模式

### 预期效果

配置示例：
```yaml
monitors:
  - provider: "88code"
    service: "cc"
    model: "claude-3-opus"    # 新增字段
    channel: "vip-channel"
    # ... 其他配置
```

前端表格展示：
| 服务商 | 服务 | 通道 | **模型** | 当前状态 | 可用率 |
|--------|------|------|----------|----------|--------|
| 88code | cc | vip-channel | **claude-3-opus** | 🟢 | 99.5% |

---

## 数据流说明

```
config.yaml (model: "claude-3-opus")
    ↓
internal/config/config.go (ServiceConfig.Model)
    ↓
internal/api/handler.go (MonitorResult.Model)
    ↓
/api/status JSON 响应 (model: "claude-3-opus")
    ↓
frontend/src/types/index.ts (MonitorResult.model)
    ↓
frontend/src/hooks/useMonitorData.ts (数据转换)
    ↓
frontend/src/types/index.ts (ProcessedMonitorData.model)
    ↓
frontend/src/components/StatusTable.tsx (UI 展示)
```

---

## 修改文件清单

| 序号 | 文件路径 | 修改类型 | 说明 |
|------|----------|----------|------|
| 1 | `internal/config/config.go` | 修改 | ServiceConfig 添加 Model 字段 |
| 2 | `internal/api/handler.go` | 修改 | MonitorResult 添加 Model 字段 |
| 3 | `frontend/src/types/index.ts` | 修改 | TypeScript 类型添加 model 字段 |
| 4 | `frontend/src/hooks/useMonitorData.ts` | 修改 | 数据转换添加 model 映射 |
| 5 | `frontend/src/components/StatusTable.tsx` | 修改 | 添加"模型"列展示 |
| 6 | `config.yaml.example` | 修改 | 添加 model 字段示例 |

---

## 详细实现步骤

### Step 1: 后端配置层

**文件**: `internal/config/config.go`

在 `ServiceConfig` 结构体中添加 `Model` 字段（约第 21 行，在 `Channel` 字段附近）：

```go
type ServiceConfig struct {
    Provider        string            `yaml:"provider" json:"provider"`
    ProviderURL     string            `yaml:"provider_url" json:"provider_url"`
    Service         string            `yaml:"service" json:"service"`
    Category        string            `yaml:"category" json:"category"`
    Sponsor         string            `yaml:"sponsor" json:"sponsor"`
    SponsorURL      string            `yaml:"sponsor_url" json:"sponsor_url"`
    Channel         string            `yaml:"channel" json:"channel"`
    Model           string            `yaml:"model" json:"model"`  // 新增：测试的模型名称
    URL             string            `yaml:"url" json:"url"`
    Method          string            `yaml:"method" json:"method"`
    APIKey          string            `yaml:"api_key" json:"api_key"`
    Headers         map[string]string `yaml:"headers" json:"headers"`
    Body            string            `yaml:"body" json:"body"`
    SuccessContains string            `yaml:"success_contains" json:"success_contains"`
}
```

**注意**:
- Model 字段设为可选，无需验证逻辑
- 不参与唯一性校验（与 channel 不同）

---

### Step 2: API 响应层

**文件**: `internal/api/handler.go`

#### 2.1 修改 MonitorResult 结构体（约第 37-48 行）

```go
type MonitorResult struct {
    Provider      string              `json:"provider"`
    ProviderURL   string              `json:"provider_url"`
    Service       string              `json:"service"`
    Category      string              `json:"category"`
    Sponsor       string              `json:"sponsor"`
    SponsorURL    string              `json:"sponsor_url"`
    Channel       string              `json:"channel"`
    Model         string              `json:"model"`  // 新增
    CurrentStatus *CurrentStatusInfo  `json:"current_status"`
    Timeline      []TimelineEntry     `json:"timeline"`
    Availability  float64             `json:"availability"`
}
```

#### 2.2 修改 GetStatus 函数中的数据填充（约第 123-133 行）

```go
response = append(response, MonitorResult{
    Provider:      task.Provider,
    ProviderURL:   task.ProviderURL,
    Service:       task.Service,
    Category:      task.Category,
    Sponsor:       task.Sponsor,
    SponsorURL:    task.SponsorURL,
    Channel:       task.Channel,
    Model:         task.Model,  // 新增
    CurrentStatus: currentStatus,
    Timeline:      timeline,
    Availability:  availability,
})
```

---

### Step 3: 前端类型定义

**文件**: `frontend/src/types/index.ts`

#### 3.1 修改 MonitorResult 接口（约第 34-44 行）

```typescript
export interface MonitorResult {
  provider: string;
  provider_url: string;
  service: string;
  category: string;
  sponsor: string;
  sponsor_url: string;
  channel: string;
  model: string;  // 新增
  current_status: {
    status: number;
    latency: number;
    timestamp: number;
  } | null;
  timeline: TimelineEntry[];
  availability: number;
}
```

#### 3.2 修改 ProcessedMonitorData 接口（约第 73-96 行）

在 `channel` 字段后添加：

```typescript
export interface ProcessedMonitorData {
  // ... 其他字段 ...
  channel?: string;
  model?: string;  // 新增
  // ... 其他字段 ...
}
```

---

### Step 4: 数据转换层

**文件**: `frontend/src/hooks/useMonitorData.ts`

在数据转换逻辑中添加 model 字段映射（约第 132-147 行）：

```typescript
return {
  provider: item.provider,
  providerUrl: item.provider_url || undefined,
  service: item.service,
  category: item.category || undefined,
  sponsor: item.sponsor || undefined,
  sponsorUrl: item.sponsor_url || undefined,
  channel: item.channel || undefined,
  model: item.model || undefined,  // 新增
  currentStatus: item.current_status ? {
    status: item.current_status.status,
    latency: item.current_status.latency,
    timestamp: new Date(item.current_status.timestamp * 1000)
  } : null,
  timeline: item.timeline,
  availability: item.availability
};
```

---

### Step 5: UI 展示层

**文件**: `frontend/src/components/StatusTable.tsx`

#### 5.1 添加表头（在"通道"列之后，约第 69-76 行附近）

找到 channel 的表头，在其后添加 model 列：

```tsx
{/* 通道列 */}
<th onClick={() => onSort('channel')}>
  <div className="flex items-center">
    通道 <SortIcon columnKey="channel" />
  </div>
</th>

{/* 新增：模型列 */}
<th onClick={() => onSort('model')}>
  <div className="flex items-center">
    模型 <SortIcon columnKey="model" />
  </div>
</th>
```

#### 5.2 添加表格单元格（在"通道"单元格之后，约第 141-143 行附近）

找到 channel 的单元格展示，在其后添加：

```tsx
{/* 通道单元格 */}
<td className="p-4 text-slate-400 text-xs">
  {item.channel || '-'}
</td>

{/* 新增：模型单元格 */}
<td className="p-4 text-slate-400 text-xs">
  {item.model || '-'}
</td>
```

#### 5.3 更新排序逻辑（如果需要）

确保排序函数支持 `model` 字段。检查 `onSort` 函数的实现，通常基于字符串比较，无需额外修改。

---

### Step 6: 配置示例

**文件**: `config.yaml.example`

为每个监控示例添加 `model` 字段：

```yaml
monitors:
  # --- 88code ---
  - provider: "88code"
    provider_url: "https://88code.com"
    service: "cc"
    category: "commercial"
    sponsor: "团队自有"
    sponsor_url: "https://example.com/sponsor"
    channel: "vip-channel"
    model: "claude-3-opus"  # 新增
    url: "https://api.88code.com/v1/chat/completions"
    method: "POST"
    api_key: "sk-xxxxxxxx"
    headers:
      Authorization: "Bearer {{API_KEY}}"
      Content-Type: "application/json"
    body: |
      {
        "model": "claude-3-opus",
        "messages": [{"role": "user", "content": "hi"}],
        "max_tokens": 1
      }

  - provider: "88code"
    service: "cx"
    category: "public"
    sponsor: "社区赞助"
    channel: "standard-channel"
    model: "gpt-4"  # 新增
    # ... 其他配置
```

---

## 测试验证

### 后端测试

```bash
# 1. 运行配置加载测试
go test -v ./internal/config/

# 2. 启动服务，验证 API 响应
./monitor config.yaml

# 3. 检查 API 响应包含 model 字段
curl http://localhost:8080/api/status | jq '.data[0].model'
```

### 前端测试

```bash
# 1. 启动前端开发服务器
cd frontend && npm run dev

# 2. 访问 http://localhost:5173
# 3. 验证表格中显示"模型"列
# 4. 验证模型名称正确展示
# 5. 测试排序功能
```

### 预期结果

1. API 响应包含 `model` 字段
2. 前端表格显示"模型"列
3. 列位置在"通道"和"当前状态"之间
4. 空值显示为 `-`
5. 支持点击表头排序

---

## 参考资料

### Channel 字段实现（作为参考模板）

Channel 字段的完整实现链路可作为本功能的参考：

| 层级 | 文件 | 关键代码位置 |
|------|------|-------------|
| 配置 | `internal/config/config.go` | 第 21 行 ServiceConfig.Channel |
| API | `internal/api/handler.go` | 第 37-48 行 MonitorResult.Channel |
| 类型 | `frontend/src/types/index.ts` | 第 34-44 行、第 73-96 行 |
| 转换 | `frontend/src/hooks/useMonitorData.ts` | 第 132-147 行 |
| 展示 | `frontend/src/components/StatusTable.tsx` | 第 69-76 行、第 141-143 行 |

### 设计原则

1. **可选字段**: 与 channel 一致，model 为可选字段
2. **空值处理**: 前端显示为 `-`
3. **一致性**: 样式、位置与现有字段保持一致
4. **无数据库影响**: model 字段不持久化，仅从配置传递到前端展示

---

## 风险与注意事项

1. **向后兼容**: 现有配置文件不包含 model 字段，需确保空值处理正确
2. **前端布局**: 新增列可能影响表格在小屏幕上的显示，需测试响应式效果
3. **排序性能**: 如果监控项数量很大，字符串排序可能有性能影响（通常可忽略）

---

## 后续扩展建议

1. **过滤功能**: 参考 channel 的过滤器实现，添加按模型筛选
2. **统计功能**: 按模型维度统计可用率
3. **自动提取**: 从 body JSON 中自动提取 model 字段（作为默认值）
