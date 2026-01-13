# WebSocket实时通讯指南

> **基于代码版本**: v4.10.0
> **最后验证时间**: 2025-12-15
> **代码文件来源**: `src/utils/websocket.js` (113行)
> **WebSocket实现**: 原生WebSocket API
>
> ⚠️ **当前状态**: 项目中的WebSocket实现极为简单，主要用于团队消息通讯

⚠️ **重要提示**: 本文档基于实际项目代码，反映当前实现的WebSocket功能。

## 目录

- [概述](#概述)
- [实际实现分析](#实际实现分析)
- [核心函数详解](#核心函数详解)
- [使用示例](#使用示例)
- [注意事项](#注意事项)
- [未来扩展建议](#未来扩展建议)

---

## 概述

项目使用 **原生WebSocket API** 实现了一个非常简单的实时通讯功能，主要用于团队消息通讯。

**核心设计**:
- 基于浏览器原生WebSocket API实现
- 单个函数封装，功能极简
- 主要用于团队频道的消息接收
- 包含简单的连接和消息处理

**当前实现特点**:
- **轻量级**: 仅113行代码，单一函数
- **简单**: 没有复杂的连接管理和重连机制
- **特定用途**: 主要用于团队消息通讯
- **回调机制**: 通过回调函数处理消息

---

## 实际实现分析

### 代码结构

实际代码位于 `src/utils/websocket.js`，仅包含一个导出函数：

```javascript
// src/utils/websocket.js (实际实现)
import { config } from "@/config";

const connectWebSocket = (team_id, callback) => {
  console.log("[team]", team_id);
  // Create WebSocket connection.
  const access_token = localStorage.getItem("access_token");
  if (!access_token) {
    return;
  }
  const socket = new WebSocket(
    `wss://api-v4dev.scantist.io/ws/msg/private-${team_id}/?access_token=${access_token}`,
  );
  socket.addEventListener("message", (event) => {
    console.log("[wss]Message from server ", event.data);
    // update project list table
  });

  // Connection opened
  socket.addEventListener("open", (event) => {
    socket.send("[wss]Hello Server!");
  });
  setTimeout(() => {
    console.log("[wss], callback");
    callback("[wss], callback message");
  }, 3000);
};

export default connectWebSocket;
```

### 关键特性

1. **URL格式**: `wss://api-v4dev.scantist.io/ws/msg/private-${team_id}/?access_token=${access_token}`
2. **认证方式**: 使用localStorage中的access_token进行认证
3. **频道模式**: 私有频道模式 (`private-${team_id}`)
4. **回调机制**: 3秒后自动执行回调函数

---

## 核心函数详解

### `connectWebSocket(team_id, callback)`

**位置**: `src/utils/websocket.js:3`

**参数说明**:
- `team_id` (string/number): 团队ID，用于构建WebSocket频道
- `callback` (function): 连接成功后的回调函数

**函数流程**:

```javascript
// 1. 检查access_token
const access_token = localStorage.getItem("access_token");
if (!access_token) {
  return; // 无token，不建立连接
}

// 2. 创建WebSocket连接
const socket = new WebSocket(
  `wss://api-v4dev.scantist.io/ws/msg/private-${team_id}/?access_token=${access_token}`
);

// 3. 监听消息事件
socket.addEventListener("message", (event) => {
  console.log("[wss]Message from server ", event.data);
  // 注释说明：这里应该更新项目列表表格
});

// 4. 连接成功时发送初始消息
socket.addEventListener("open", (event) => {
  socket.send("[wss]Hello Server!");
});

// 5. 3秒后执行回调
setTimeout(() => {
  console.log("[wss], callback");
  callback("[wss], callback message");
}, 3000);
```

**实际消息处理**:
- 当前实现只是简单打印消息到控制台
- 注释显示意图是"更新项目列表表格"
- 没有复杂的消息解析和处理逻辑

---

## 使用示例

### 基本使用

```javascript
import connectWebSocket from '@/utils/websocket';

// 连接到团队123的WebSocket
connectWebSocket(123, (message) => {
  console.log('收到回调消息:', message);
  // 这里可以处理连接成功后的逻辑
});
```

### 在Vue组件中使用

```vue
<template>
  <div>
    <button @click="connectToTeam">连接团队WebSocket</button>
  </div>
</template>

<script setup>
import { ref } from 'vue'
import connectWebSocket from '@/utils/websocket'

const teamId = ref(123) // 从store或props获取团队ID

function connectToTeam() {
  connectWebSocket(teamId.value, (message) => {
    console.log('WebSocket连接成功:', message)
    // 这里可以更新UI状态或触发其他操作
  })
}
</script>
```

### 在Store中使用

```javascript
// stores/team/actions.js
import connectWebSocket from '@/utils/websocket'

export const connectTeamWebSocket = ({ commit }, teamId) => {
  connectWebSocket(teamId, (message) => {
    console.log('团队WebSocket连接成功')
    commit('SET_WEBSOCKET_CONNECTED', true)
  })
}
```

---

## 注意事项

### 当前实现限制

1. **功能简单**: 只有连接功能，没有重连机制
2. **消息处理简单**: 仅打印消息到控制台
3. **无错误处理**: 连接失败时没有错误处理
4. **无断开机制**: 没有提供断开连接的方法

### 认证要求

- 必须存在有效的`access_token`在localStorage中
- 服务器会验证access_token的有效性

### 使用场景

当前实现适合：
- 简单的团队消息通知
- 不需要复杂重连逻辑的场景
- 临时性的实时通讯需求

### 性能考虑

- 每个连接都会消耗服务器资源
- 建议按需连接，避免不必要的连接
- 页面切换时可能需要手动处理连接

---

## 未来扩展建议

### 功能增强

如果需要更强大的WebSocket功能，建议：

1. **添加重连机制**
```javascript
function reconnect(attempts = 0, maxAttempts = 5) {
  if (attempts < maxAttempts) {
    setTimeout(() => {
      connectWebSocket(teamId, callback);
    }, 3000 * Math.pow(2, attempts));
  }
}
```

2. **完善消息处理**
```javascript
socket.addEventListener('message', (event) => {
  try {
    const data = JSON.parse(event.data);
    // 根据消息类型处理
    if (data.type === 'scan_status_update') {
      handleScanStatusUpdate(data);
    }
  } catch (error) {
    console.error('消息解析失败:', error);
  }
});
```

3. **添加连接状态管理**
```javascript
const connectionState = {
  isConnected: false,
  lastMessage: null,
  reconnectAttempts: 0
};
```

---

## 总结

当前项目的WebSocket实现非常简洁，主要特点：

- **实现方式**: 原生WebSocket API，单一函数
- **核心功能**: 团队频道的简单消息通讯
- **认证方式**: 基于access_token的私有频道
- **使用场景**: 适合简单的实时通知需求

该实现适合项目当前的简单需求，如果未来需要更复杂的实时通讯功能，建议在此基础上进行扩展。