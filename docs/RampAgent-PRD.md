# RampAgent · Phase 1 PRD（MVP — 只做“Agent 支付闭环”）

> **目标**：跑通「（一次性支付）→ 可选配置 Agent →（复购时）Agent 代付 → 商户收款（链上分润 1%）」的最小闭环。  
> **登录**：阶段一仅支持 **Google 登录**。  
> **链上授权范围**：链上仅部署 **MandateRegistry**（保存 `paramsHash + active`）与 **PaymentManager**（校验授权 + 广播事件）；授权参数生成、风险控制、扣款编排仍由链下承担。  
> **平台分润**：商户 99% / 平台 **1%**。  
> **资金前提**：**一次性支付可不充值**；**Agent 自动扣款必须先充值（USDC）**。

---

## 0. 用户与价值（简）

- **C 端用户**：首单跳转支付；可选开启“Agent 自动支付”（额度/有效期/商户白名单），复购免跳转。  
- **商户**：以 USDC 收款；分润链上可见；更高转化（推荐渠道）。  
- **Agent（执行体）**：由用户授权的后台进程/脚本；通过 Session Key 调用 RampAgent API 自动支付。  
- **差异点**：**推荐支付渠道（Route Advisor）** —— 可用 **Agent（如 Coze）mock** 推荐结果与理由（费率/成功率/延迟）。

---

## 1. 用户故事 & 端到端流程

### US-1 首次支付（一次性支付，可不充值）

- **我作为用户**，在商户 A 下单跳转到 RampAgent，看到**推荐支付渠道**并完成付款后回到商户。  
- **验收**：Checkout 展示「**推荐 + 0–2 备选**」及理由（可 mock）；完成支付 → 回跳商户，获得 `payment_id, status=success`。

### US-2 配置 Agent（授权）

- **我作为用户**，开启自动支付，设置**额度/有效期/商户白名单**；此时可先不充值，但**启用自动扣款前必须完成充值**。  
- **验收**：生成 **Mandate（链下 EIP-712 签名 + `paramsHash`，存库，简称 `mandateHash`）** → 调用链上 **MandateRegistry.register(paramsHash)**（默认 `active=true`） → 颁发 **Session Key**（平台保存）；授权列表可见；支持**暂停/撤销**（触发 `MandateRegistry.setActive(paramsHash, false)`）；撤销后 `/api/agent/pay` 返回 403。

### US-3 充值（Agent 自动扣款的必选前提）

- **我作为用户**，为后续 Agent 支付**预存 USDC**。  
- **验收**：输入金额 → 选择 Provider（可 mock）→ 成功后余额增加，余额页显示可用 USDC；若未充值则自动扣款不可用（前端与 API 提示）。

### US-4 Agent 代付（复购免跳转）

- **我作为平台（代 Agent）**，商户请求扣款时在授权范围内自动完成支付，商户到账。  
- **验收**：`/api/agent/pay` 在校验 **Mandate/SessionKey/额度/白名单/余额** 后执行；合约发 `PaymentExecuted` 事件；`FeeDistributor` 按 **99:1** 分润；商户端可见入账与明细。

#### 流程图（概览）

```
商户A下单 → 跳转 RampAgent Checkout
→ 推荐支付渠道（Agent/Coze mock）
→ Google 登录（创建账户 + 生成托管地址）
→ 选择渠道并完成“一次性支付”（可不充值）
→ （可选）进入“配置 Agent”：额度/有效期/白名单 → 生成 Mandate + paramsHash（链下） → MandateRegistry.register(paramsHash, active=true)
→ （需要自动扣款时）充值 USDC
→ 回跳商户A（payment_id, success）

（复购自动扣款）
商户A后端 → /api/agent/pay
→ 校验：Mandate有效 + SessionKey有效 + 额度未超 + 商户在白名单 + 余额充足
→ 记账/扣余额 → 合约：PaymentManager.validateAndRecord(mandateHash, merchant, amount, order_ref)（require MandateRegistry.active(paramsHash)）→ distribute(99:1)
→ 返回 payment_id → 商户端入账展示
```

---

## 2. 前端（Frontend）

### 2.1 Checkout（支付页）

- 信息：商户名、订单金额。  
- **Route Advisor（Agent/Coze mock）**：  
  - 顶部**推荐渠道卡片**（理由=费率最低/成功率高/预计 N 秒）。  
  - 下方 0–2 个备选（列出差异）。  
- 登录：Google OAuth。  
- 完成：第三方（或 mock）→ 回跳成功页 → 回跳商户。  
- 引导：显著入口「配置 Agent（下次免跳转）」。

### 2.2 配置 Agent

- 表单：**额度（USDC）/ 有效期（天）/ 商户白名单（默认当前商户）**。  
- 操作：确认授权（生成 Mandate + `paramsHash`、写入 MandateRegistry、颁发 Session Key）、暂停/撤销（更新 MandateRegistry `active`）。  
- 列表：授权记录（最近 5 条）。  
- 提示：**未充值则自动扣款不可用**（提供“去充值”按钮）。

### 2.3 用户中心 / 余额

- USDC 余额、**充值入口**（mock）、最近 10 笔交易。  
- 状态提示：自动扣款 **可用/不可用**（依据是否已充值与授权状态）。

### 2.4 商户后台（非样板，最小可用）

- 今日/累计收款、最近 10 笔、分润统计、Webhook 配置。  
- 支持按 `payment_id`、时间范围查询。

### 2.5 商户站点样板（Template，前端提供）

- 目的：5 分钟本地跑通“下单 → 跳转 RampAgent → 回调入账”。  
- 页面：`/` 商品列表、`/checkout` 下单（集成 Checkout 按钮/iframe）、`/success` 成功页。  
- 集成：  
  - 下单：`POST /merchant/create-order` → 返回 `checkout_url`  
  - 回调：`POST /merchant/webhook`（`status, payment_id, amount`）  
  - 查询：`GET /merchant/payment/:id`

---

## 3. 后端（Backend）

### 3.1 业务服务

- **Auth**：`/api/auth/google`（创建账户 + 生成托管地址）。  
- **Mandate**：  
  - `POST /api/mandate`（`amount_limit, valid_until, merchant_allowlist[]`）→ 链下 EIP-712 签名、生成 `paramsHash`、持久化 → 调用链上 `MandateRegistry.register(paramsHash)`（默认 `active=true`）→ 颁发 Session Key。  
  - `POST /api/mandate/revoke` / `POST /api/mandate/pause`（如有）→ 调用 `MandateRegistry.setActive(paramsHash, false)` 并更新链下状态；`POST /api/mandate/resume`（如有）→ 调用 `setActive(..., true)`。  
  - `GET /api/mandate/list` 聚合链下字段与链上 `active` 状态。  
- **Balance**：`POST /api/topup/mock`、`GET /api/balance`。  
- **Agent Pay**：  
  - `POST /api/agent/pay`（`mandate_id, merchant_id, amount, order_ref`）  
  - 校验 Mandate/SessionKey/额度/白名单/余额 + `MandateRegistry.active(paramsHash)` → 记账/扣余额 → 合约：`PaymentManager.validateAndRecord(mandateHash, merchant, amount, order_ref)`（内部 require MandateRegistry 仍为 `active`，并发事件）→ 合约：`distribute` → 返回 `payment_id`。  
- **Merchant**：  
  - `POST /merchant/webhook`（RampAgent → 商户）  
  - `GET /merchant/payment/:payment_id`（商户轮询/WS）

### 3.2 数据模型（简）

- `users(user_id, google_id, email, wallet_address, created_at)`  
- `balances(user_id, asset=USDC, available, updated_at)`  
- `mandates(mandate_id, user_id, agent_id, params_hash, merchant_allowlist(json), amount_limit, used_amount, valid_until, signature, status)`  
- `session_keys(id, mandate_id, agent_id, valid_until, scope, status)`  
- `payments(payment_id, user_id, merchant_id, amount, status, tx_hash?, created_at)`

### 3.3 安全与风控

- `/api/agent/pay` 必检：  
  - `mandate.status == active`  
  - `now < valid_until`  
  - `used_amount + amount <= amount_limit`  
  - `merchant ∈ merchant_allowlist`  
  - `MandateRegistry.active(params_hash) == true`（链上读）  
  - `balance >= amount`  
- 支持**立即撤销/暂停**（Session Key 失效）。  
- 审计日志：IP/UA/签名指纹。  
- 托管地址动账仅通过受限方法；默认**小额限额**（如 50 USDC/30 天）。

---

## 4. 合约组件（Contracts）

> 注：文中 `paramsHash` 与 `mandateHash` 等价，均为链下授权参数的哈希摘要。

### 4.1 MandateRegistry（授权登记）

- 状态：`mapping(bytes32 paramsHash => bool active)`，仅存储授权哈希与开关；原始参数全部留在链下。  
- 函数：  
  - `register(paramsHash)`：仅限平台运营地址调用；`require` 未存在；写入 `active=true`；发 `MandateRegistered(paramsHash, msg.sender)`。  
  - `setActive(paramsHash, bool active)`：仅限平台/合规角色；更新状态；按布尔值发 `MandateActivated` / `MandateDeactivated`。  
  - `isActive(paramsHash)`：view 方法，供 PaymentManager 与链下调用方读取。  
- 事件：`MandateRegistered`、`MandateActivated`、`MandateDeactivated`（携带 `block.timestamp`、触发地址）。

### 4.2 PaymentManager（校验 + 事件）

- `validateAndRecord(paramsHash, merchant, amount, orderRef)`：仅平台服务调用；`require MandateRegistry.isActive(paramsHash)`；可选地对 `orderRef` 做防重复；发 `PaymentValidated(paramsHash, merchant, amount, orderRef)`。  
- `emitPaymentExecuted(paramsHash, merchant, amount, txRef)`（可与上函数合并）：在完成链下扣款与分润记账后调用，发 `PaymentExecuted(paramsHash, merchant, amount, txRef, block.timestamp)`。  
- 所有写操作受 `onlyOwner/roles` 控制；PaymentManager 自身不记录余额，只负责校验与广播事件。

### 4.3 FeeDistributor（分润）

- `distribute(merchant, platform, amount)`  
- **比例**：商户 99% / 平台 **1%**（配置项）

> 部署：测试网（如 Base Sepolia）或本地链；链上事件用于 Demo 可视化与对账。

---

## 5. Demo 验收（功能性）

- 能完成**一次性支付**（无需充值），并回跳商户取得支付结果。  
- 能完成**配置 Agent**（生成/展示/暂停/撤销授权），并能在链上 `MandateRegistry` 看到注册/状态事件。  
- **充值后**，Agent 可在授权与余额约束下**自动扣款**。  
- `PaymentManager` 能校验 `MandateRegistry` 状态并发出 **PaymentValidated/PaymentExecuted** 事件，随后 `FeeDistributor` 完成**99:1 分润**；商户端能看到入账明细。  
- 在撤销或余额不足等异常条件下，**自动扣款被拒**并有清晰错误提示。
