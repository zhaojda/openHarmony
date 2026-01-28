# AutoTest 实时播放自动化测试功能说明

## 概述

本文档说明 `AutoTest.ets` 中实时播放自动化测试功能的实现。该功能支持通过 JSON 配置文件进行自动化音频测试，包括标准渲染和实时播放两种模式。

## 功能特性

### 1. 支持实时播放模式

- 通过 JSON 配置文件中的 `realTime` 或 `real_time` 字段启用实时播放
- 使用 `multiPipelineRealTimeSaveFileBuffer` 进行实时音频渲染
- 实时监控播放进度
- 自动资源管理和清理

### 2. 状态管理

新增的状态变量：

```typescript
@State isRealTimePlaying: boolean = false;        // 标识是否正在实时播放
@State currentPlaybackProgress: number = 0;       // 当前播放进度
@State audioRendererReady: boolean = false;       // 音频渲染器就绪状态
```

### 3. 测试用例执行

#### 单个用例执行
- 点击"执行选中用例"按钮
- 根据 `realTime` 标志选择渲染模式
- 显示执行进度和结果
- 生成单个用例测试报告

#### 批量执行
- 点击"执行全部用例"按钮
- 顺序执行所有测试用例
- 统计成功/失败数量
- 显示总耗时
- 生成综合测试报告

## JSON 测试用例格式

### 基本结构

```json
{
  "summary": "测试用例名称",
  "realTime": true,
  "scenes": [
    {
      "scene_name": "场景1",
      "description": "场景描述",
      "input_files": [
        {
          "path": "input.wav",
          "sample_rate": 48000,
          "bit_depth": 32,
          "channels": 2,
          "format": "wav",
          "effect": [
            {
              "type": "NR",
              "gains": []
            }
          ]
        }
      ],
      "mixer_info": [],
      "output_file": {
        "name_template": "output",
        "format": "wav",
        "sample_rate": 48000,
        "bit_depth": 32,
        "channels": 2
      },
      "expected_output": {
        "md5Value": ["预期的MD5值"]
      }
    }
  ]
}
```

### 字段说明

- `summary`: 测试用例摘要名称
- `realTime`: 是否启用实时播放模式（true/false，或使用 `real_time`）
- `scenes`: 测试场景数组
  - `scene_name`: 场景名称
  - `description`: 场景描述
  - `input_files`: 输入音频文件配置
  - `mixer_info`: 混音器配置
  - `output_file`: 输出文件配置
  - `expected_output`: 期望输出（用于MD5校验）

## 实时播放流程

### 1. 初始化阶段

```
JSON解析 → 检测realTime标志 → 初始化管线(initAllPipeline)
  → 初始化输入节点(initAllInputNode) → 初始化效果节点(initAllEffectNode)
```

### 2. 执行阶段

**实时模式：**
```
设置isRealTimePlaying=true → 调用multiPipelineRealTimeSaveFileBuffer
  → 渲染音频数据 → 保存输出文件
```

**标准模式：**
```
调用multiPipelineSaveFileBuffer → 渲染音频数据 → 保存输出文件
```

### 3. 监控阶段

- 每200ms调用 `audioNapi.getAutoTestProcess()` 获取进度
- 更新 UI 显示当前进度
- 所有管线完成后清理定时器

### 4. 清理阶段

```
保存文件 → MD5校验 → 生成报告 → 销毁管线(destroyPipeline)
  → 重置状态(isRealTimePlaying=false)
```

## 错误处理

### 1. 初始化错误

- 管线初始化失败
- 输入节点初始化失败
- 效果节点初始化失败

所有错误都会记录详细日志并返回 FAILED 状态。

### 2. 渲染错误

使用 try-catch 捕获渲染过程中的异常：

```typescript
try {
  const buffer = await taskpool.execute(multiPipelineRealTimeSaveFileBuffer, ...);
  // 处理结果
} catch (error) {
  Logger.error(TAG, `Real-time playback error: ${JSON.stringify(error)}`);
  return FAILED;
}
```

### 3. 资源清理错误

`destroyPipeline()` 函数包含多层错误处理：
- 管线级别错误捕获
- 输入节点级别错误捕获
- 确保资源列表被清空

## 日志记录

### 关键日志点

1. **初始化：**
   - `executeSingleFileWithSyncInit: realTime mode=true/false`
   - `Real-time playback mode activated`

2. **执行：**
   - `Starting real-time rendering for pipeline ${pipelineId}`
   - `Real-time rendering completed for pipeline ${pipelineId}`

3. **完成：**
   - `Real-time playback: File saved successfully`
   - `Real-time playback mode deactivated, resources cleaned up`

4. **错误：**
   - `Real-time playback error: ${error}`
   - 所有错误都包含详细的上下文信息

## 测试报告

### CSV 报告格式

```csv
执行管线,开始执行时间,结束执行时间,总耗时,输入文件,输入音频时长,输出文件,执行结果
场景名称,2025-01-28 15:00:00,2025-01-28 15:00:30,30s,input.wav,10s,output.wav,执行成功
```

### 报告内容

- 执行管线名称
- 开始和结束时间
- 总耗时（秒）
- 输入文件信息
- 输入音频时长
- 输出文件路径
- 执行结果（成功/失败）

### 批量执行报告

执行全部用例时，生成包含所有用例的综合报告，包括：
- 每个用例的详细执行信息
- 总成功数和失败数
- 总执行时间

## 使用步骤

### 1. 准备测试环境

确保以下目录存在：
- `/storage/Users/currentUser/Download/src.main.audiodemo/case_json` - 测试用例JSON文件
- `/storage/Users/currentUser/Download/src.main.audiodemo/input_files` - 输入音频文件
- `/storage/Users/currentUser/Download/src.main.audiodemo/output_files` - 输出文件目录
- `/storage/Users/currentUser/Download/src.main.audiodemo/expected_output` - 期望输出目录

### 2. 创建测试用例

在 `case_json` 目录下创建 JSON 文件，设置 `realTime: true` 启用实时播放。

### 3. 放置输入文件

将测试所需的音频文件放入 `input_files` 目录。

### 4. 执行测试

- **单个用例：** 选择用例后点击"执行选中用例"
- **全部用例：** 点击"执行全部用例"

### 5. 查看结果

- 实时查看进度对话框
- 执行完成后查看结果消息
- 检查 `output_files` 目录中的输出文件
- 查看生成的 CSV 测试报告

## 技术实现细节

### NAPI 方法调用

通过 `audioNapi from 'libentry.so'` 调用 C++ 层方法：

- `multiPipelineRealTimeSaveFileBuffer` - 实时渲染并保存
- `multiPipelineSaveFileBuffer` - 标准渲染并保存
- `getAutoTestProcess` - 获取执行进度
- `destroyMultiPipeline` - 销毁多管线

### 异步任务处理

使用 `taskpool.execute` 执行耗时的渲染操作：

```typescript
const buffer = await taskpool.execute(
  multiPipelineRealTimeSaveFileBuffer,
  pipelineId,
  outputInfo,
  multiRenderFrameFlag
);
```

### 进度监控

使用 `setInterval` 定期轮询进度：

```typescript
let timer = setInterval(() => {
  let processObj = audioNapi.getAutoTestProcess();
  // 更新进度显示
  // 检查是否完成
}, 200);
```

## 注意事项

1. **文件路径：** 确保所有路径正确且文件存在
2. **JSON格式：** 严格遵守 JSON 格式规范
3. **MD5校验：** 如果提供 expected_output，输出文件会进行 MD5 校验
4. **资源清理：** 系统会自动清理资源，但如果测试中断，可能需要手动重启应用
5. **实时模式：** 实时播放模式适用于需要音频渲染回调的场景

## 调试建议

1. 查看日志输出了解执行流程
2. 检查生成的 CSV 报告文件
3. 验证输出文件的 MD5 值
4. 确认输入文件格式正确
5. 使用单个用例测试验证配置后再批量执行

## 版本信息

- 实现版本：2025-01-28
- 支持的 SDK 版本：6.0.0(20) 及以上
- 运行时：HarmonyOS
