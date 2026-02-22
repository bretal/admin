# 单点登录（SSO）实现流程总结

## 一、前置共识

- **登录态由谁决定**：后端（认证中心 / 业务系统后端）决定 token/ticket 是否有效；前端只根据「本地是否有 token/cookie」和「接口是否返回 401」决定是否跳转登录。
- **两种常见方式**：① URL 带 **ticket**，由后端用 ticket 换 token；② URL 直接带 **token**，前端存 cookie/storage。

下文用到的域名示例（可替换成你自己的）：

- 认证中心：`https://sso.company.com`
- OA 系统：`https://oa.company.com`
- 管理后台（本项目）：`https://admin.company.com`

---

## 二、方式一：Ticket 机制（后端换 token）

### 2.1 流程概览

```
用户直接访问 OA
  → OA 发现未登录，重定向到认证中心（URL 带 redirect_uri）
  → 用户在认证中心登录
  → 认证中心重定向回 OA，URL 带 ticket
  → OA 后端用 ticket 向认证中心换 token/用户信息
  → 后端把 token 下发给前端（Set-Cookie 或响应体）
  → 前端存 token，完成登录
```

### 2.2 具体 URL 与步骤

**步骤 1：用户直接访问 OA（未登录）**

```
用户请求: https://oa.company.com/home
```

OA 发现无 cookie/token → 不展示业务页，而是 302 重定向到认证中心，并告知「登录成功后回哪」：

```
302 Redirect
Location: https://sso.company.com/login?redirect_uri=https://oa.company.com/callback&client_id=oa&response_type=code
```

- `redirect_uri`：登录成功后要跳回 OA 的地址（一般是 OA 的固定回调路径）。
- `client_id`：OA 在认证中心的唯一标识。
- `response_type=code`：表示用 code（一次性凭证）方式，和 ticket 同理，这里用 code 代表「一次性凭证」。

**步骤 2：用户在认证中心登录**

```
用户访问: https://sso.company.com/login?redirect_uri=https://oa.company.com/callback&client_id=oa&response_type=code
```

用户输入账号密码，认证中心校验通过后，生成**一次性 ticket/code**，并 302 重定向回 OA，**URL 里带 ticket**：

```
302 Redirect
Location: https://oa.company.com/callback?ticket=ST-12345-abcdef&state=xxx
```

- `ticket`（或 `code`）：一次性凭证，只能使用一次，有效期短（如 5 分钟）。
- `state`：防 CSRF，可选。

**步骤 3：OA 后端用 ticket 换 token**

浏览器请求 OA 的回调地址（带 ticket）时，**由 OA 后端处理**，而不是前端直接存 ticket：

```
浏览器请求: https://oa.company.com/callback?ticket=ST-12345-abcdef&state=xxx
```

OA 后端收到后：

1. 拿 `ticket` 请求认证中心校验并换用户信息/ token：
   ```
   POST https://sso.company.com/api/validate
   Body: { ticket: "ST-12345-abcdef", service: "https://oa.company.com/callback" }
   ```
2. 认证中心返回用户信息 + token（或仅用户信息，由 OA 自己发 token）：
   ```json
   {
     "userId": "u001",
     "username": "zhangsan",
     "roles": ["staff"],
     "accessToken": "eyJ..."
   }
   ```
3. OA 后端：把 token 写入 Cookie（Set-Cookie）或通过接口响应给前端；前端将 token 存 cookie/storage。

**步骤 4：前端完成登录**

- 若后端 Set-Cookie：浏览器再次访问 `https://oa.company.com/home` 会带上 cookie，OA 认为已登录，直接展示首页。
- 若后端通过 JSON 把 token 给前端：前端把 token 存 cookie 或 storage，后续请求在 Header 里带 token。

**要点小结（Ticket 方式）**

- 认证中心回跳 OA 时，URL 里只带 **ticket**，不带长期 token。
- **换 token 的动作在 OA 后端完成**；前端不拿 ticket 直接请求自己的后端「换 token」也可以（前端把 ticket 交给 OA 后端，后端再去认证中心换）。
- 若 OA 后端或认证中心异常、没返回 token，这次单点登录就失败。

---

## 三、方式二：URL 直接带 Token（前端存 token）

### 3.1 流程概览

```
用户从门户/认证中心点击「进入 OA」
  → 认证中心重定向到 OA，URL 中直接带 accessToken、username、roles 等
  → OA 前端从 URL 解析参数，setToken 存 cookie/storage，再清掉 URL 里的敏感参数
  → 完成登录，后续请求带 cookie/token
```

### 3.2 具体 URL 与步骤

**步骤 1：用户已在认证中心登录，点击「进入 OA」**

认证中心构造跳转 URL，**直接把 token 和用户信息放在 URL 里**（通常用于内网或信任域）：

```
302 Redirect（或前端跳转）
Location: https://oa.company.com/#/home?username=zhangsan&roles=staff&accessToken=eyJhbGciOiJIUzUxMiJ9.xxx
```

或带 path 和 query 的形态：

```
https://oa.company.com/#/permission/page/index?username=sso&roles=admin&accessToken=eyJhbGciOiJIUzUxMiJ9.admin
```

**步骤 2：OA 前端加载，解析 URL 并判断是否为 SSO**

- 从 `location.href` 解析出：`username`、`roles`、`accessToken`。
- 若三者都存在，则判定为「单点登录回跳」。

**步骤 3：前端处理 SSO 逻辑（不经过后端换 token）**

1. `removeToken()`：清空本地旧登录态。
2. `setToken(params)`：把 `username`、`roles`、`accessToken` 等写入 **cookie 或 storage**（如本项目 `auth.ts` 的 Cookie + localStorage）。
3. 删除 URL 中的敏感参数：`delete params.roles; delete params.accessToken;`（避免地址栏长期暴露）。
4. 用 `window.location.replace(newUrl)` 跳转到「干净」的 URL，例如：
   ```
   https://oa.company.com/#/home?username=zhangsan
   ```
   这样刷新时 URL 不再带 token/roles，不会误判为 SSO 造成循环。

**步骤 4：后续请求**

- 请求时从 cookie 或 storage 读 token，放在 Header（如 `Authorization`）或由 cookie 自动携带，后端校验 token 即可。

**要点小结（Token 方式）**

- 认证中心回跳时 URL **直接带 accessToken**（及 username、roles）。
- **不需要后端用 ticket 换 token**；前端自己从 URL 取参数并 `setToken` 存起来。
- 前端必须在存完 token 后**立刻清掉 URL 里的 token/roles**，避免泄露和刷新死循环。

---

## 四、两种方式对比（含 URL）

| 项目                 | Ticket 方式                                     | Token 方式（URL 带 token）                                              |
| -------------------- | ----------------------------------------------- | ----------------------------------------------------------------------- |
| 回跳 OA 时 URL       | `https://oa.company.com/callback?ticket=ST-xxx` | `https://oa.company.com/#/home?username=xx&roles=xx&accessToken=eyJ...` |
| Token 从哪来         | OA **后端**用 ticket 向认证中心换               | 认证中心**直接放在 URL** 里                                             |
| 谁写 cookie/存 token | OA 后端（Set-Cookie）或后端返回后前端存         | **前端**从 URL 取参后 setToken                                          |
| 安全性               | token 不经过地址栏，相对更安全                  | token 会短暂出现在 URL，需立刻 replace 清掉                             |
| 典型场景             | 企业内多系统、标准 CAS/OAuth                    | 内网/信任域、从门户带 token 跳转                                        |

---

## 五、是否登录的判断（两种方式通用）

- **依据**：本地是否有有效 **cookie / token**（以及后端是否认可）。
- **无 token 或后端返回 401**：视为未登录 → 重定向到登录页或认证中心，例如：
  ```
  https://sso.company.com/login?redirect_uri=https://oa.company.com/xxx
  ```
- **有 token 且未 401**：视为已登录 → 直接展示业务页（首页/OA 等），不展示登录页。

---

## 六、强制下线（异地登录 / 管理台踢人）

- **谁控制**：**后端**控制「哪些 token 失效」；前端只根据 401 或推送做「清 token + 跳转登录」。
- **实现要点**：
  - 后端用 **Redis 存 token** 或 **token_version**，在「异地登录」或「管理台强制下线」时作废旧 token（删 Redis 或 version+1）。
  - 接口统一对失效 token 返回 **401**（可带 reason：如 `OTHER_DEVICE_LOGIN`、`ADMIN_FORCE_LOGOUT`）。
  - 前端在全局响应拦截器里对 401 做：`removeToken()` → 跳转登录页，并可提示「您已在其他设备登录」或「管理员已将您下线」。
- 若要**即时**踢下线（不等到下次请求）：后端通过 **WebSocket/SSE** 推送「强制下线」消息，前端收到后同样 `removeToken()` 并跳转登录。

---

## 七、本项目中 sso.ts 对应的方式

- 本项目采用的是 **方式二（URL 直接带 Token）**。
- 测试链接示例（注释中）：
  ```
  http://localhost:8848/#/permission/page/index?username=sso&roles=admin&accessToken=eyJhbGciOiJIUzUxMiJ9.admin
  ```
- 逻辑：解析 URL → 必须同时存在 `username`、`roles`、`accessToken` 才视为 SSO → `removeToken` → `setToken(params)` → 删 URL 中敏感参数 → `location.replace(newUrl)`。

以上即可作为「单点登录实现流程 + 两种方式（ticket / token）+ 具体 URL」的复习总结。
