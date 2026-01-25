# 实施计划 - 时长格式化修复 (严格审查复查版)

## 功能调用分析与影响评估

经过全局搜索与代码审查，确认了以下函数及其调用者：

### 1. `convertDurationInSec`
- **当前调用**: `src/views/project/components/SAAS/SAASScanDetailHeader.vue`
- **预期变更**: 修复 24 小时回绕显示错误，统一格式。
- **风险**: 低。仅用于显示扫描时长，UI空间允许字符串长度略微增加（如 "25h"）。

### 2. `changeSecondsToHours`
- **当前调用**: `src/views/project/components/SampleProjectForm.vue`
- **预期变更**: 修复回绕问题，修复 `HOURS`/`MINUTES`/`SECONDS` 翻译键缺失导致的显示问题。
- **风险**: 低。当前显示可能为Broken string（如 "3HOURS..."），修复后将正常显示。

### 3. `duration`
- **当前调用**:
  - `src/views/scan/ScanDescription.vue`
  - `src/views/project/SAASProjects.vue`
  - `src/views/project/SAASProjectDetail.vue`
  - `src/views/project/ProjectCard.vue`
  - `src/views/project/components/ScanHistory.vue`
  - `src/views/project/components/SAAS/SAASScanCompareModel.vue`
  - `src/views/project/components/SAAS/SAASScanHistoryModal.vue`
- **预期变更**: 修复 `moment.duration().hours()` 导致的 24 小时回绕问题。
- **风险**: 中。此函数被多个核心页面使用。变更后的逻辑必须确保 `0` 值和空值的处理与原逻辑完全兼容，避免表格或卡片中的显示异常。
- **特别注意**: 原逻辑在时长 > 0 时返回 `Y M D h m s` 组合字符串，无单位部分不显示。新逻辑需严格遵守此格式习惯（即单位间有空格，无值单位隐藏）。

## 核心修复逻辑 (DateHelper.ts)

所有修复均集中在 `src/utils/dateHelper.ts`，不修改调用方代码。

### 通用数学计算函数 (新增内部工具)
为了确保一致性，将时长计算逻辑抽取为通用模式。

```typescript
function formatDurationFromSeconds(totalSeconds: number): string {
  if (totalSeconds < 0) return "-";
  
  const years = Math.floor(totalSeconds / (365.25 * 24 * 3600));
  let remaining = totalSeconds % (365.25 * 24 * 3600);
  const months = Math.floor(remaining / (30.44 * 24 * 3600));
  remaining = remaining % (30.44 * 24 * 3600);
  const days = Math.floor(remaining / (24 * 3600));
  remaining = remaining % (24 * 3600);
  const hours = Math.floor(remaining / 3600);
  remaining = remaining % 3600;
  const minutes = Math.floor(remaining / 60);
  const seconds = Math.floor(remaining % 60);

  const result = [];
  if (years > 0) result.push(`${years} Y`);
  if (months > 0) result.push(`${months} M`);
  if (days > 0) result.push(`${days} D`);
  if (hours > 0) result.push(`${hours} h`);
  if (minutes > 0) result.push(`${minutes} m`);
  if (seconds > 0) result.push(`${seconds} s`);
  
  return result.join(" ");
}
```

### 修正后的导出函数

#### `convertDurationInSec(duration)`
用于 SAAS 扫描详情页。
- 逻辑：直接调用核心计算逻辑，但忽略年月日，只关注时分秒（扫描通常不会持续数天）。
- **修正**: 支持 > 24h。
- **兼容**: `0` 返回 `"-"`。

#### `changeSecondsToHours(value)`
用于 SampleProjectForm。
- 逻辑：同上。
- **修正**: 移除错误的 i18n key，统一使用 `h/m/s`。
- **兼容**: `0` 返回 `""` (原逻辑行为)。

#### `duration(start, end)`
用于项目列表和历史记录。
- 逻辑：计算 `end - start` 得到秒数，调用通用格式化。
- **修正**: 修复 Moment.js 导致的 24h 回绕。
- **兼容**: `0` 或无效时间返回 `"-"`。

## 验证计划 (执行时)

1.  **覆盖率检查**: 确保上述所有受影响组件在修复后被手动或逻辑check。
2.  **边界值测试**:
    - `0` 秒 -> 对应函数应分别返回 `"-"` 或 `""`。
    - `3661` 秒 -> `"1 h 1 m 1 s"` (注意空格)。
    - `90000` 秒 -> `"25 h"`。
