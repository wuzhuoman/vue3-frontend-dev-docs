# WebSocket 连接配置

## 📝 问题描述

在排查深度许可证扫描结果UI不自动更新的问题时，需要理解WebSocket连接如何确定主机和端口，特别是当URL中包含端口号时的处理逻辑。

## 🔍 根本原因 / 核心需求

WebSocket连接需要准确获取服务器的主机地址和端口号，当用户访问的URL包含非标准端口时，必须正确提取和使用该端口，否则会导致连接失败。

## 💡 解决方案

### 方案概述

通过正则表达式提取URL中的端口号，并结合当前主机信息构建WebSocket连接地址。

### 技术实现

```javascript
// 在 src/views/project/DashboardLayout.vue 中的实现
const setupWebSocket = () => {
  // 从当前URL提取主机和端口
  const url = window.location.href
  const match = url.match(/^(https?:\/\/[^:\/]+)(?::(\d+))?/)
  
  let wsHost = 'localhost' // 默认值
  let wsPort = null
  
  if (match) {
    wsHost = match[1].replace('https://', '').replace('http://', '')
    // 如果URL中指定了端口，则使用该端口
    if (match[2]) {
      wsPort = match[2]
    }
  }
  
  // 根据环境获取WebSocket端口号
  const wsUrl = getWebSocketUrl(wsHost, wsPort)
  // ... 连接WebSocket的后续代码
}
```

### 关键要点

- 要点1: 使用正则表达式 `/^(https?:\/\/[^:\/]+)(?::(\d+))?/` 可以完整提取URL中的主机和端口部分
- 要点2: 第二个捕获组 `(match[2])` 专门用于获取端口号，如果存在则使用该端口
- 要点3: 当URL不包含端口时，回退到默认端口配置

## ⚠️ 注意事项

- **适用场景**: 需要精确控制WebSocket连接地址的前端应用
- **不适用场景**: 使用标准WebSocket连接且不关心端口的应用
- **潜在风险**: 在某些代理环境下，前端的端口可能与实际WebSocket服务端口不匹配

## 📚 相关资源

- WebSocket MDN文档: https://developer.mozilla.org/zh-CN/docs/Web/API/WebSocket
- 项目中的WebSocket实现: src/views/project/DashboardLayout.vue
- Pusher.js 文档: https://pusher.com/docs/channels/library_auth_reference/javascript

## 📅 文档元信息

- 创建日期: 2025-11-28
- 最后更新: 2025-11-28
- 关联项目版本: v4.6.0