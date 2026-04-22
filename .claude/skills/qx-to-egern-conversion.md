# Quantumult X 到 Egern 模块转换指南

## 概述

将 Quantumult X (QX) 的重写规则脚本转换为 Egern 模块格式，包括 YAML 模块定义和 JavaScript 脚本适配。

## 转换步骤

### 1. 创建 Egern 模块 YAML 文件

YAML 文件包含模块元数据和配置：

```yaml
name: "模块名称"
description: "模块描述"
author: "作者"
icon: "SF Symbols 名称或图标URL"

scriptings:
  - http_request:
      name: "请求脚本名称"
      match: "^https://api\\.example\\.com/.*"  # URL 匹配正则
      script_url: "script.js"                    # 脚本文件路径或URL
      timeout: 30
      body_required: false                       # 是否需要请求体
  - schedule:
      name: "定时任务名称"
      cron: "30 8,20 * * *"                      # cron 表达式
      script_url: "script.js"
      timeout: 120

mitm:
  hostnames:
    includes:
      - "api.example.com"
      - "*.example.com"
```

### 2. JavaScript 脚本 API 转换

#### 主要 API 映射表

| QX API | Egern API | 说明 |
|---------|-----------|------|
| `[rewrite_local]` + `script-request-header` | `scriptings: - http_request:` | HTTP 请求拦截 |
| `[rewrite_local]` + `script-response-body` | `scriptings: - http_response:` | HTTP 响应拦截 |
| `[task_local]` + cron | `scriptings: - schedule:` + `cron:` | 定时任务 |
| `[MITM] hostname =` | `mitm: hostnames: includes:` | MITM 主机名配置 |
| `$request` | `ctx.request` | 请求对象 |
| `$done({})` | `return {}` | 完成回调 |
| `$done()` | `return` | 完成回调（无返回值） |
| `$notify(title, subtitle, body)` | `ctx.notify({title, subtitle, body})` | 通知 |
| `$prefs.valueForKey(key)` | `localStorage.getItem(key)` | 读取持久化数据 |
| `$prefs.setValueForKey(value, key)` | `localStorage.setItem(key, value)` | 写入持久化数据 |
| `$task.fetch({url, method, headers})` | `await ctx.http.get(url, {headers})` | HTTP 请求 |

#### 脚本结构转换

**QX 脚本结构：**
```javascript
// QX 顶层执行
if (typeof $request !== 'undefined' && $request) {
  // 抓包模式
  $done({});
} else {
  // 定时任务模式
  $done();
}
```

**Egern 脚本结构：**
```javascript
// Egern 导出异步函数
export default async function(ctx) {
  if (ctx.request) {
    // http_request 模式（抓包）
    // 使用 localStorage 代替 $prefs
    const data = localStorage.getItem('key');
    localStorage.setItem('key', value);
    return {};
  } else {
    // schedule 模式（定时任务）
    const data = localStorage.getItem('key');
    return;
  }
}
```

### 3. 持久化存储转换

**QX 方式：**
```javascript
const raw = $prefs.valueForKey('key');
const data = raw ? JSON.parse(raw) : default;
$prefs.setValueForKey(JSON.stringify(data), 'key');
```

**Egern 方式（使用 Web Storage API）：**
```javascript
const raw = localStorage.getItem('key');
const data = raw ? JSON.parse(raw) : default;
localStorage.setItem('key', JSON.stringify(data));
```

**注意：** `localStorage` 是同步 API，无需 `await`。

### 4. HTTP 请求转换

**QX 方式：**
```javascript
$task.fetch({
  url: 'https://api.example.com/data',
  method: 'GET',
  headers: { 'Authorization': 'Bearer token' }
}).then(res => {
  const data = JSON.parse(res.body);
  console.log(data);
});
```

**Egern 方式：**
```javascript
const resp = await ctx.http.get('https://api.example.com/data', {
  headers: { 'Authorization': 'Bearer token' }
});
const data = await resp.json();
console.log(data);
```

### 5. Cron 表达式说明

Egern 支持标准 5 字段和 6 字段 cron 格式：

```
# 5 字段（分 时 日 月 周）
30 8,20 * * *     # 每天 8:30 和 20:30
0 8 * * *         # 每天 8:00

# 6 字段（秒 分 时 日 月 周）
0 0 8 * * *       # 每天 8:00:00
```

### 6. 模块文件组织

推荐的项目结构：
```
Egern/
├── PingMe.yaml          # 模块定义
├── PingMe.js            # 脚本文件
├── WeTalk.yaml
└── WeTalk.js
```

YAML 中 `script_url` 可以是：
- 本地文件名：`"PingMe.js"`
- 远程 URL：`"https://raw.githubusercontent.com/user/repo/main/Egern/PingMe.js"`
- 相对路径：`"./scripts/PingMe.js"`

## 注意事项

1. **API 差异**：Egern 不支持 QX 的 `$task`、`$prefs` 等全局对象，需使用 `ctx.http`、`localStorage` 替代

2. **异步处理**：Egern 脚本必须使用 `async/await`，并导出为 `export default async function(ctx)`

3. **返回值**：
   - http_request 脚本返回空对象 `{}` 表示不修改请求
   - schedule 脚本无需返回值

4. **持久化存储**：使用标准 `localStorage` API，数据以字符串形式存储

5. **URL 匹配**：YAML 中的 `match` 字段使用正则表达式，注意转义特殊字符（如 `.` → `\\.`）

6. **MITM 配置**：确保 YAML 中的 `mitm.hostnames.includes` 包含所有需要解密的主机名

## 示例

完整示例见项目中的 [PingMe.yaml](Egern/PingMe.yaml) 和 [PingMe.js](Egern/PingMe.js)。

## 参考资源

- Egern 官方文档：https://egernapp.com/docs/intro
- Egern 模块文档：https://egernapp.com/docs/configuration/modules/
- Egern 脚本文档：https://egernapp.com/docs/configuration/scriptings/
