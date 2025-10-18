# RampAgent · Phase 1 PRD

> **目标**：跑通「（一次性支付）→ 可选配置 Agent →（复购时）Agent 代付 → 商户收款（链上分润 1%）」的最小闭环。  
> **登录**：阶段一仅支持 **Google 登录**（可选，用于查看余额/充值/配置 Agent）。  
> **KYC**：阶段一不做 KYC，引导用户在所选 Provider 处完成。  
> **链上授权范围**：链上部署 **MandateRegistry**（保存 `paramsHash + active`）、**PaymentManager**（校验授权 + 广播事件）与 **AgentRegistry**（登记受控 Agent + 记录声誉）；授权参数生成、风险控制、扣款编排仍由链下承担。  
> **平台分润**：商户 99% / 平台 **1%**。  
> **资金前提**：**一次性支付可不充值**；**Agent 自动扣款必须先充值（USDC）**。

---

## 0. 用户与价值（简）

- **C 端用户**：首单跳转支付；可选开启“Agent 自动支付”（额度/有效期/商户白名单），复购免跳转。  
- **商户**：以 USDC 收款；分润链上可见；更高转化（推荐渠道）。  
- **Agent（执行体）**：由用户授权的后台进程/脚本；通过 Session Key 调用 RampAgent API 自动支付；推荐 Agent 由平台托管（Coze 等），按链上声誉排序后返回。  
- **差异点**：**推荐支付渠道（Route Advisor）** —— 可用 **Agent（如 Coze）mock** 推荐结果与理由（费率/成功率/延迟），阶段一不涉及 KYC；若 Provider 要求 KYC，则用户跳转后在对方页面完成。

---

## 1. 用户故事 & 端到端流程

### US-1 首次支付（一次性支付，可不充值）

- **我作为用户**，在商户 A 下单跳转到 RampAgent，看到多个 route agent 的推荐结果（含费率、预计时间等）并完成付款后回到商户。  
- **验收**：Checkout 展示 route agent 推荐列表；异步加载结果时有骨架/提示；用户无需登录即可选择任意推荐 → 若选择 Pay Agent 且余额充足则直接扣款，否则跳转 Provider 完成支付 → 回跳商户，获得 `payment_id, status=success`。  

### US-2 查看余额与充值（可选）

- **我作为用户**，可在支付前登录 RampAgent，查看 custodial 钱包余额并发起充值。  
- **验收**：登录成功后可在余额页查看资产与最近交易；充值入口跳转至独立流程（mock），充值完成后余额更新；如需 KYC，由 Provider 在其页面处理。

### US-3 配置 Pay Agent（授权）

- **我作为用户**，登录后可开启自动支付，设置**额度/有效期/商户白名单**。  
- **验收**：生成 **Mandate（链下 EIP-712 签名 + `paramsHash`，存库，简称 `mandateHash`）** → 调用链上 **MandateRegistry.register(paramsHash)**（默认 `active=true`） → 颁发 **Session Key**（平台保存）；授权列表可见；支持**撤销**（触发 `MandateRegistry.setActive(paramsHash, false)`）；撤销后平台的 Agent 扣款接口返回 403。

### US-4 Agent 代付（复购免跳转）

- **我作为平台（代 Agent）**，商户请求扣款时在授权范围内自动完成支付，商户到账。  
- **验收**：平台的 Agent 扣款接口在校验 **Mandate/SessionKey/额度/白名单/余额** 后执行；合约发 `PaymentExecuted` 事件；`FeeDistributor` 按 **99:1** 分润；商户端可见入账与明细。

#### 流程图（概览）

- 商户下单，跳转到 RampAgent 界面  
- 异步查询 route agent 计算结果  
- 显示多个 route agent 推荐结果及相关信息（包括费率、预估时间等）  
- 用户选择其中一个推荐结果  
- 可选 Google 登录，登录后可查看钱包余额、充值、配置自动支付（Pay Agent）  
- 选择支付方式  
  1. Pay Agent 支付（若已登录且余额足够）  
  2. 跳转 Provider 页面完成支付  
- 完成支付，回跳商户  

---

## 2. 前端（Frontend）

### 2.1 Checkout（支付页）

- 信息：商户名、订单金额。  
- **Route Advisor（Agent/Coze mock）**：  
  - 异步加载多个 Route Agent 推荐，展示费率、预估时间、成功率等指标。  
  - 允许用户在 Pay Agent（余额扣款）与跳转 Provider 支付之间做出选择。  
  - 后端整合所有托管 Agent（Coze）输出的方案，按链上声誉排序后返回。  
- 登录（可选）：Google OAuth（查看余额/充值/配置 Agent 时使用），Checkout 保持无登录支付体验。  
- 完成：第三方（或 mock）→ 回跳成功页 → 回跳商户。  
- 引导：显著入口「配置 Agent（下次免跳转）」。

### 2.2 配置 Agent

- 表单：**额度（USDC）/ 有效期（天）/ 商户白名单（默认当前商户）**。  
- 操作：确认授权（生成 Mandate + `paramsHash`、写入 MandateRegistry、颁发 Session Key）、撤销（更新 MandateRegistry `active=false`）。  
- 列表：授权记录（最近 5 条）。  
- 提示：**未充值则自动扣款不可用**（提供“去充值”按钮）。

### 2.3 用户中心 / 余额

- USDC 余额、**充值入口**（mock）、最近 10 笔交易（需登录访问，KYC 由 Provider 页面处理）。  
- 状态提示：自动扣款 **可用/不可用**（依据是否已充值与授权状态）。

### 2.4 商户后台（非样板，最小可用）

- 今日/累计收款、最近 10 笔、分润统计、Webhook 配置。  
- 支持按 `payment_id`、时间范围查询。

### 2.5 商户站点样板（Template，前端提供）

- 目的：5 分钟本地跑通“下单 → 跳转 RampAgent → 回调入账”。  
- 页面：`/` 商品列表、`/checkout` 下单（集成 Checkout 按钮/iframe）、`/success` 成功页。  
- 集成：样板后端需提供下单接口返回 `checkout_url`，能够接收支付结果回调，并支持按 `payment_id` 查询交易状态。

---

## 3. 后端（Backend）

### 3.1 业务服务

- **Auth**：支持 Google 登录完成账户创建与托管地址生成，提供前端可调用的 OAuth 交换流程。  
- **Mandate 管理**：负责链下参数签名、`paramsHash` 生成与持久化；可创建、撤销授权，保持与链上 `MandateRegistry` 状态同步，并颁发 Session Key。  
- **Agent Recommendation**：汇聚订单上下文、provider quote、用户偏好，调用托管的 Coze Agent；整合结果并依据 `AgentRegistry` 声誉排序，返回推荐及备选方案。  
- **Balance 服务**：提供充值（mock）与余额查询能力，维持用户 USDC 余额账本，KYC 流程由各 Provider 自己处理。  
- **Agent Pay**：在商户请求扣款时校验授权、额度、白名单与余额，完成记账与链上调用（`PaymentManager.validateAndRecord` → `FeeDistributor.distribute` → `PaymentManager.confirmAndRecord`）。  
- **Merchant 通知**：向商户侧推送支付结果 Webhook，并提供查询明细的接口或订阅机制。

> 具体接口契约参见 `docs/openapi-frontend-backend.yaml` 与 `docs/openapi-backend-agent.yaml`。

### 3.2 数据模型（简）

- `users(user_id, google_id, email, wallet_address, created_at)`  
- `balances(user_id, asset=USDC, available, updated_at)`  
- `mandates(mandate_id, user_id, agent_id, params_hash, merchant_allowlist(json), amount_limit, used_amount, valid_until, signature, status)`  
- `session_keys(id, mandate_id, agent_id, valid_until, scope, status)`  
- `agents(agent_address, name, metadata_uri, status, created_at)`  
- `agent_reputation(agent_address, adoption_count, total_amount, last_payment_id, updated_at)`  
- `payments(payment_id, user_id, merchant_id, amount, status, tx_hash?, created_at)`

### 3.3 安全与风控

- Agent 扣款接口必检：  
  - `mandate.status == active`  
  - `now < valid_until`  
  - `used_amount + amount <= amount_limit`  
  - `merchant ∈ merchant_allowlist`  
  - `MandateRegistry.active(params_hash) == true`（链上读）  
  - `balance >= amount`  
- 支持**立即撤销授权**（Session Key 失效）。  
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

### 4.2 AgentRegistry（托管 Agent + 声誉）

- 状态：`mapping(address agent => AgentProfile)`（含 `metadataURI`、`status`）与 `mapping(address agent => Reputation)`（`adoptionCount`、`totalAmount`、`lastPaymentId`、`lastUpdated`）。  
- 函数：  
  - `registerAgent(address agent, bytes calldata metadataURI)`：仅限平台脚本/运营调用；登记或更新受控 Agent。  
  - `setStatus(address agent, AgentStatus status)`：开启/停用 Agent。  
  - `getAgent(address agent)` / `getReputation(address agent)`：公开查询接口，供前端排序/展示。  
  - `recordAdoption(address agent, uint256 paymentId, uint256 amount)`：仅限授权服务调用，支付成功后累计声誉与金额。  
- 事件：`AgentRegistered`、`AgentStatusChanged`、`AgentAdoptionRecorded`。

### 4.3 PaymentManager（校验 + 声誉回写）

- `validateAndRecord(paramsHash, merchant, amount, orderRef)`：仅平台服务调用；`require MandateRegistry.isActive(paramsHash)`；可选地对 `orderRef` 做防重复；发 `PaymentValidated(paramsHash, merchant, amount, orderRef)`。  
- `emitPaymentExecuted(paramsHash, merchant, amount, txRef)`（可与上函数合并）：在完成链下扣款与分润记账后调用，发 `PaymentExecuted(paramsHash, merchant, amount, txRef, block.timestamp)`。  
- `confirmAndRecord(paymentId, address agent, uint256 amount)`：支付成功后由后端调用，内部确认订单闭环并调用 `AgentRegistry.recordAdoption`。  
- 所有写操作受 `onlyOwner/roles` 控制；PaymentManager 自身不记录余额，只负责校验、事件和声誉写入。

### 4.4 FeeDistributor（分润）

- `distribute(merchant, platform, amount)`  
- **比例**：商户 99% / 平台 **1%**（配置项）

> 部署：测试网（如 Base Sepolia）或本地链；链上事件用于 Demo 可视化与对账。

---

## 5. Demo 验收（功能性）

- 能完成**一次性支付**（无需充值），并回跳商户取得支付结果。  
- 能完成**配置 Agent**（生成/展示/撤销授权），并能在链上 `MandateRegistry` 看到注册/状态事件。  
- **充值后**，Agent 可在授权与余额约束下**自动扣款**。  
- `PaymentManager` 能校验 `MandateRegistry` 状态并发出 **PaymentValidated/PaymentExecuted** 事件，随后 `FeeDistributor` 完成**99:1 分润**；商户端能看到入账明细。  
- `AgentRegistry` 可通过脚本注册受控 Agent，并在 `PaymentManager.confirmAndRecord` 后发出 `AgentAdoptionRecorded` 事件，声誉累计正确。  
- 在撤销或余额不足等异常条件下，**自动扣款被拒**并有清晰错误提示。
