# Digital Shield 隔空二维码热钱包接入技术手册

> **English** · [English](./README.md) &nbsp;|&nbsp; **简体中文**（当前）

> **文档定位**：面向第三方热钱包 / 观察钱包团队，说明如何对接 Digital Shield 硬件的**账户二维码导入**与 **11 条链转账交易二维码签名**。可直接据此实现扫码交易 Demo，再演进到正式产品入口。  
> **协议字段权威说明**：CBOR Key / Tag / `data_type` 等细节以本文第 5–11 章为准。  
> **语言**：简体中文。

---
## 目录

0. [硬件安全性说明（创建 / 存储助记词）](#10-硬件安全性说明创建--存储助记词)
1. [文档目标与角色边界](#1-文档目标与角色边界)
2. [支持的 11 条链一览](#2-支持的-11-条链一览)
3. [端到端交互模型](#3-端到端交互模型)
4. [设备侧操作入口（联调必读）](#4-设备侧操作入口联调必读)
5. [UR / 动画二维码承载层](#5-ur--动画二维码承载层)
6. [导入账户：`crypto-multi-accounts`](#6-导入账户crypto-multi-accounts)
7. [签名总览与公共约定](#7-签名总览与公共约定)
8. [EVM 六链：`eth-sign-request`](#8-evm-六链eth-sign-request)
9. [Bitcoin / Litecoin / Dogecoin：`crypto-psbt`](#9-bitcoin--litecoin--dogecoincrypto-psbt)
10. [TRON：`tron-sign-request`](#10-trontron-sign-request)
11. [Solana：`sol-sign-request`](#11-solanasol-sign-request)
12. [热钱包模块设计建议](#12-热钱包模块设计建议)
13. [Demo 验证方案（推荐落地顺序）](#13-demo-验证方案推荐落地顺序)
14. [安全校验清单](#14-安全校验清单)
15. [常见问题与排障](#15-常见问题与排障)
16. [附录：Registry / 参考实现](#16-附录registry--参考实现)

---

## 1.0 硬件安全性说明（创建 / 存储助记词）

Digital Shield 硬件在**钱包创建与助记词保管**阶段使用设备侧随机熵，并在安全边界内存储助记词；热端接入时**不接触助记词明文**。下图概括从熵源、助记词派生到账户导出的完整路径，用于理解「硬件侧安全」与本手册隔空签名职责划分的关系：

![Digital Shield 硬件钱包创建与助记词安全流程](./Digital-Shield-Hardware-Wallet-Flow-Integrated.jpeg)

- 大图文件：[Digital-Shield-Hardware-Wallet-Flow-Integrated.jpeg](./Digital-Shield-Hardware-Wallet-Flow-Integrated.jpeg)（本仓库）
- 热端仅应持久化公钥 / xpub / 地址与 `masterFingerprint` 等观察数据
- 隔空签名流程见 [第 3 章](#3-端到端交互模型)；安全校验见 [第 14 章](#14-安全校验清单)

---

## 1. 文档目标与角色边界

### 1.1 你要做成什么

第三方钱包在 App 中增加：

1. **扫描 Digital Shield 账户二维码** → 导入账户（含至多11条链）。
2. **发起各链转账** → 生成待签二维码 → 用户用硬件扫描并确认 → App 扫描回签二维码 → 组装交易并广播。

全程**不经过 USB / 蓝牙**，仅二维码交换 [BC-UR](https://github.com/BlockchainCommons/Research/blob/master/papers/bcr-2020-005-ur.md) 载荷。

### 1.2 职责划分

| 角色 | 持有私钥 | 职责 | 不做 |
| --- | --- | --- | --- |
| **热端 App**（本手册读者） | 否 | 扫账户码、构建未签交易、展示待签码、扫回签、验签、组装 raw、用自有 RPC 广播 | 不存助记词、不在热端本地签硬件账户 |
| **Digital Shield 硬件** | 是 | 导出公钥账户、扫待签码、屏上确认、签名、展示回签码 | 不联网、不广播 |

### 1.3 实现路径（二选一或组合）

| 方式 | 说明 |
| --- | --- |
| **按本文自研** | 自行接入 BC-UR + CBOR Registry；字段以本文为准 |
| **使用官方热端 SDK** | 专用于热端钱包接入的sdk（见仓库 `digitalshield-airgap-sdk`），可缩短联调时间，接入后兼容支持冷钱包app以及此硬件钱包的账户导入和隔空签名；仍建议对照本文核对 UR type / 路径 |

无论哪种方式，**广播、余额、Gas、节点**均由接入方钱包自有链路负责。

---

## 2. 支持的 11 条链一览

隔空转账签名覆盖 **6 条 EVM + 5 条非 EVM**。

**导入二维码整体结构（与 EVM / 非 EVM 无关）：**

- 设备只导出 **一种** 账户码：`UR:crypto-multi-accounts`（tag `1103`）。
- 这是**整包容器**：一次动画二维码里带上多条链的扩展公钥；不是「EVM 一种码、非 EVM 另一种码」。
- 包内 `keys[]` 每一项才是具体的 `crypto-hdkey`（tag `303`）；热端按 `coin_type` + 路径 + `note` 挑出下表所需账户。

| # | 网络 | Chain ID / Coin Type | 导入账户 UR | 从 multi-accounts 取用的 HDKey | 签名请求 UR | 回签 UR |
| --- | --- | --- | --- | --- | --- | --- |
| 1 | Ethereum | `chain_id = 1` | `crypto-multi-accounts`（11 链共用） | ETH `crypto-hdkey`（coin `60`） | `eth-sign-request` | `eth-signature` |
| 2 | BNB Chain | `56` | 同上 | 同上（与 ETH 共用一套） | 同上 | 同上 |
| 3 | Polygon | `137` | 同上 | 同上 | 同上 | 同上 |
| 4 | X Layer | `196` | 同上 | 同上 | 同上 | 同上 |
| 5 | Arbitrum | `42161` | 同上 | 同上 | 同上 | 同上 |
| 6 | Base | `8453` | 同上 | 同上 | 同上 | 同上 |
| 7 | Bitcoin | SLIP-44 `0` | 同上 | 独立 BTC HDKey（多脚本） | `crypto-psbt` | `crypto-psbt` |
| 8 | Litecoin | SLIP-44 `2` | 同上 | 独立 LTC HDKey | `crypto-psbt`（或 `crypto-psbt-extend`） | 同请求类型 |
| 9 | Dogecoin | SLIP-44 `3` | 同上 | 独立 DOGE HDKey | 同上 | 同上 |
| 10 | TRON | SLIP-44 `195` | 同上 | 独立 TRX HDKey | `tron-sign-request` | `tron-signature` |
| 11 | Solana | SLIP-44 `501` | 同上 | 独立 SOL HDKey（ed25519） | `sol-sign-request` | `sol-signature` |

要点：

- **导入 UR**：始终是 `UR:CRYPTO-MULTI-ACCOUNTS/…` 动画码（设备「连接 App → 二维码 → Digital Shield」导出）。表中「取用的 HDKey」只说明从该 UR 的 `keys[]` 里认哪几条，不表示另有导入 UR type。
- **EVM 六链**：导入只拿一套 `m/44'/60'/…` 扩展公钥；六链地址相同，签名靠 `chain_id` 区分。多地址用**末位**地址索引：`m/44'/60'/0'/0/i`。
- **BTC / LTC / DOGE**：导入按 coin type + 脚本类型分 HDKey；**多账户用 BIP44 第三段账户号**（与 Digital Shield 热钱包「增加账户」一致）：`m/{purpose}'/{coin}'/{i}'/0/0`。签名多为 BIP-174 PSBT。LTC/DOGE 亦可使用 Keystone 风格的 `crypto-psbt-extend`（见第 9 章）。
- 设备导出的二维码里**可能还含** Polkadot、Aptos、Ledger 兼容路径等；**本阶段第三方只需解析上表 11 条链所需 HDKey，其余可忽略**。固件当前**不对 DOT/Aptos 提供隔空签名 UR**。

---

## 3. 端到端交互模型

### 3.1 导入账户（单向：设备 → App）

```
┌──────────────┐   动画 UR:crypto-multi-accounts    ┌──────────────┐
│ Digital Shield│ ─────────────────────────────────▶ │  热钱包 App  │
│ 展示账户二维码 │                                    │ 扫描并持久化 │
└──────────────┘                                    └──────────────┘
```

App **不需要**向设备回传任何内容。

### 3.2 隔空签名（双向）

```
热钱包: 构造未签交易 → 编码 Sign Request → 动画二维码展示
    │
    ▼  用户用硬件摄像头扫描
硬件: 解析 → 屏上展示收款/金额/网络 → 用户确认 → 签名
    │
    ▼  硬件展示 Signature 动画二维码
热钱包: 扫描回签 → 校验 request_id → 组装已签交易 → 广播
```

`request_id`（16 字节 UUID）必须在请求与回签中一致，用于绑定会话、防止扫到其它交易的码。

### 3.3 会话状态（热端必须维护）

建议在内存中保存 `pendingSign`（签名完成或超时后清除）：

```text
pendingSign = {
  requestId,          // UUID bytes / hex
  chain,              // eth | bsc | … | btc | ltc | doge | trx | sol
  urTypeRequest,      // eth-sign-request | crypto-psbt | …
  unsignedPayload,    // RLP / PSBT / raw_data / sol message
  derivationPath,
  masterFingerprint,  // 导入时的 xfp
  fromAddress,
  meta                // 金额、to、symbol 等 UI 信息
}
```

---

## 4. 设备侧操作入口（联调必读）

### 4.1 导出账户二维码（给热钱包扫描）

典型路径（固件 UI）：

1. 解锁设备进入主页。
2. 进入 **连接 App / Connect**。
3. 选择 **二维码** 连接方式。
4. 选择 **Digital Shield**。
5. 设备展示 **动画** `UR:CRYPTO-MULTI-ACCOUNTS/…` 二维码。

热钱包持续扫帧直到 UR 解码完成。

### 4.2 扫描待签二维码（硬件签交易）

1. 热钱包展示待签动画二维码。
2. 设备进入扫码（主页扫码 / 连接相关扫码入口，视固件版本而定）。
3. 扫满所有分片后，设备解析 UR type → 进入对应链确认页。
4. 用户确认后，设备展示回签动画二维码。
5. 热钱包扫描回签。

### 4.3 设备账户二维码编码参数（与热端展示建议对齐）

| 参数 | 设备侧常见值 | 热端建议 |
| --- | --- | --- |
| UR 分片长度 `max_fragment_len` | 账户导出约 **100** 字节 | 待签码建议 **100–200**；过大易导致设备识读失败 |
| 动画刷新 | 约 **250 ms** 量级 | **200–500 ms**；过快相机跟不上，过慢体验差 |
| 字符串大小写 | 大写 `UR:…` | **统一大写** |
| 二维码纠错 | 设备展示侧自定 | 展示侧建议 ECC **M 或 Q**；过密时优先减小分片而非无限提高 ECC |

---

## 5. UR / 动画二维码承载层

### 5.1 格式

| 形态 | 字符串形态 | 说明 |
| --- | --- | --- |
| 单帧 | `UR:<TYPE>/<bytewords>` | 载荷较短 |
| 多帧（Fountain） | `UR:<TYPE>/<SEQ>-<TOTAL>/<bytewords>` | 循环播放；解码端收集至 `isComplete()` |

`<TYPE>` 为 registry 类型名（如 `CRYPTO-MULTI-ACCOUNTS`、`ETH-SIGN-REQUEST`）。Bytewords 为 BC-UR minimal 编码。

### 5.2 热端必备能力

1. **Encoder（展示侧）**：将业务对象编码为 CBOR，再封装为 BC-UR；按约定的 `max_fragment_len` 切分为有序（或 Fountain）分片字符串列表，由 UI 按固定间隔轮播展示。
2. **Decoder（扫描侧）**：相机逐帧识别二维码文本，持续调用 `receivePart(frame)` 喂入解码器，直至 `isComplete()`；完成后取出 UR `type` 与完整 `cborBytes`，再交由 Registry 反序列化。
3. **Registry**：按本文 Map Key 编解码各业务对象。

推荐开源能力栈（语言不限，概念等价即可）：

- UR：Blockchain Commons `bc-ur` / 生态中的 JS、Swift、Kotlin 移植
- CBOR：标准 CBOR 库
- 业务对象：可参考 Keystone `ur-registry` 生态中同名类型（字段以**本文**为准）

### 5.3 进度 UI 建议

解码端宜展示已收分片数、预估总量或完成百分比，便于用户对准镜头。分片数量由载荷体积与 `max_fragment_len` 共同决定：`crypto-multi-accounts`、大型 PSBT 等多帧属常态；即便是体积较小的 `eth-sign-request`，在分片长度偏保守时也可能拆成多帧。热端与硬件均应按**多帧可完成**实现，不得假定任一业务类型恒为单帧。

---

## 6. 导入账户：`crypto-multi-accounts`

### 6.1 类型

| 项目 | 值 |
| --- | --- |
| UR type | `crypto-multi-accounts` |
| CBOR tag（外层对象） | `1103` |

### 6.2 外层 CBOR Map

| Key | 字段 | 类型 | 必填 | 说明 |
| --- | --- | --- | --- | --- |
| `1` | `masterFingerprint` | uint32 | 是 | 钱包主指纹（xfp），**后续签名必须回传** |
| `2` | `keys` | array of tagged `crypto-hdkey` (303) | 是 | 多链 / 多脚本扩展公钥 |
| `3` | `device` | text | 否 | 设备名称 |
| `4` | `deviceId` | text | 否 | 设备 ID |
| `5` | `deviceVersion` | text | 否 | 固件版本 |

热端应持久化：`masterFingerprint`、`deviceId`（若有）、以及各链选出的账户元数据。

### 6.3 `crypto-hdkey`（tag `303`）

| Key | 字段 | 类型 | 说明 |
| --- | --- | --- | --- |
| `2` | `is_private` | bool | 始终为 `false` |
| `3` | `key_data` | bytes | secp256k1 压缩公钥 33B；Solana 为 ed25519 公钥 32B |
| `4` | `chain_code` | bytes | 32B；Solana 通常无 |
| `5` | `use_info` | tagged `crypto-coin-info` (305) | coin type + network |
| `6` | `origin` | tagged `crypto-keypath` (304) | 派生路径 + source fingerprint |
| `7` | `children` | tagged `crypto-keypath` (304) | 相对可派生路径（可能缺省） |
| `8` | `parent_fingerprint` | uint32 | 父指纹 |
| `9` | `name` | text | 可选 |
| `10` | `note` | text | 账户类型标记，见下表 |

`crypto-coin-info`：

| Key | 字段 | 值 |
| --- | --- | --- |
| `1` | `coin_type` | BTC `0`、LTC `2`、DOGE `3`、ETH `60`、TRX `195`、SOL `501` |
| `2` | `network` | `0` = MainNet |

`crypto-keypath`：

| Key | 字段 | 说明 |
| --- | --- | --- |
| `1` | `components` | `[index, hardened, …]`；通配符为 empty + hardened |
| `2` | `source_fingerprint` | 与 masterFingerprint 相同的 xfp |
| `3` | `depth` | 路径深度 |

### 6.4 如何挑选 11 条链账户（不要按下标）

按 **`use_info.coin_type` + `origin` 路径 + `note`** 匹配：

| 链 | note（优先） | origin path（账户 #0） | 曲线 | 地址 / 多账户如何得到 |
| --- | --- | --- | --- | --- |
| EVM 六链共用 | `account.standard` | `m/44'/60'/0'` | secp256k1 | 再派生 `0/i` → 第 i 个地址（末位地址索引） |
| Bitcoin Legacy | `account.btc_legacy` | `m/44'/0'/0'` | secp256k1 | 账户 i：`m/44'/0'/i'/0/0`（第三段账户号） |
| Bitcoin Nested SegWit | `account.btc_segwit` | `m/49'/0'/0'` | secp256k1 | 账户 i：`m/49'/0'/i'/0/0` |
| Bitcoin Native SegWit | `account.btc_native_segwit` | `m/84'/0'/0'` | secp256k1 | 账户 i：`m/84'/0'/i'/0/0`（推荐默认脚本） |
| Bitcoin Taproot | `account.btc_taproot` | `m/86'/0'/0'` | secp256k1 | 账户 i：`m/86'/0'/i'/0/0`（BIP86，x-only + tweak） |
| Litecoin Legacy | `account.ltc_legacy` | `m/44'/2'/0'` | secp256k1 | 账户 i：`m/44'/2'/i'/0/0` |
| Litecoin Nested SegWit | `account.ltc_segwit` | `m/49'/2'/0'` | secp256k1 | 账户 i：`m/49'/2'/i'/0/0` |
| Litecoin Native SegWit | `account.ltc_native_segwit` | `m/84'/2'/0'` | secp256k1 | 账户 i：`m/84'/2'/i'/0/0`（推荐默认脚本） |
| Dogecoin | `account.standard` | `m/44'/3'/0'` | secp256k1 | 账户 i：`m/44'/3'/i'/0/0` |
| TRON | `account.standard` | `m/44'/195'/0'` | secp256k1 | 再派生 `0/i` → 第 i 个地址（Base58Check） |
| Solana | `account.standard` | `m/44'/501'/0'/0'` | ed25519 | 路径已到地址级，直接用 `key_data`；多账户常见 `m/44'/501'/i'/0'` |
| Solana Ledger Live | `account.ledger_live` | `m/44'/501'/0'` | ed25519 | 兼容路径，按需 |

可能出现但本阶段可忽略：`account.ledger_legacy` / `account.ledger_live`（ETH）、Polkadot（`354`）、Aptos（`637`）等。

#### 6.4.1 UTXO 多账户索引（与 Digital Shield 热钱包对齐）

BTC / LTC / DOGE 不要用「同一账户下末位 `0/i` 递增」来表示热钱包里的「账户 #0 / #1 / #2」：

```text
m/{purpose}'/{coinType}'/$$INDEX$$'/0/0
```

| 热钱包 / 逻辑序号 | 完整收款路径（Legacy BTC 例） | 硬件 multi-accounts 的 account 级 path |
| --- | --- | --- |
| `#0` | `m/44'/0'/0'/0/0` | `m/44'/0'/0'` |
| `#1` | `m/44'/0'/1'/0/0` | `m/44'/0'/1'` |
| `#2` | `m/44'/0'/2'/0/0` | `m/44'/0'/2'` |

说明：

- Digital Shield 热钱包 UI 常显示「账户 #1」对应逻辑序号 `#0`（界面从 1 起算）。
- 设备导出的 `crypto-multi-accounts` 通常至少含账户 `#0` 的各脚本 HDKey；若要展示 `#1`、`#2`，助记词热端自行按上表派生。
- 从账户级 HDKey（`…/i'`）得到收款地址：再派生相对路径 `0/0`（change=0，address_index=0）。
- **对比**：EVM / TRON 仍为 `m/44'/{coin}'/0'/0/i`（第三段固定 `0'`，末位为地址索引）。

### 6.5 导入后热端应存储的最小集

```text
ImportedDevice = {
  masterFingerprintHex,   // 8 位小写 hex，如 "caeff70f"
  deviceId?, deviceVersion?, deviceName?,
  accounts: [
    {
      chainFamily,        // evm | btc | ltc | doge | tron | sol
      note, coinType, path,   // UTXO: 账户级如 m/84'/0'/0'；完整收款路径另存或拼 /0/0
      publicKey, chainCode?,
      accountIndex,       // UTXO: BIP44 第三段；EVM: 末位地址索引
      receiveAddress,     // 默认展示地址（UTXO 为 …/i'/0/0）
      xpub?               // UTXO 可选
    }
  ]
}
```

---

## 7. 签名总览与公共约定

### 7.1 请求 / 回签对照

| 链族 | 扫描热端 → 设备 | 扫描设备 → 热端 |
| --- | --- | --- |
| EVM × 6 | `eth-sign-request` (401) | `eth-signature` (402) |
| BTC | `crypto-psbt` (310) | `crypto-psbt` (310) |
| LTC / DOGE | `crypto-psbt` (310) **或** `crypto-psbt-extend` (312) | 与请求同类型 |
| TRON | `tron-sign-request` (5201) | `tron-signature` (5202) |
| Solana | `sol-sign-request` (1101) | `sol-signature` (1102) |

> **兼容说明（非本手册主路径）**：部分第三方 App 对 TRON 使用 `keystone-sign-request` / `keystone-sign-result`（gzip + protobuf）。Digital Shield 固件可识别该路径，但**第三方自研热钱包应优先使用原生 `tron-sign-request`**，实现更简单、字段更清晰。

### 7.2 UUID（tag `37`）

`request_id`：16 字节 UUID，回签原样返回。热端用它匹配 `pendingSign`。

### 7.3 Derivation Path（tag `304`）

必须是**完整、无通配符**路径，且 `source_fingerprint` = 导入时的 xfp。设备用其校验当前钱包；不匹配则拒绝签名。

示例：

- EVM 地址 #0：`m/44'/60'/0'/0/0`；地址 #1：`m/44'/60'/0'/0/1`
- BTC Native SegWit 账户 #0：`m/84'/0'/0'/0/0`；账户 #1：`m/84'/0'/1'/0/0`
- BTC Legacy 账户 #0：`m/44'/0'/0'/0/0`；账户 #1：`m/44'/0'/1'/0/0`
- TRON：`m/44'/195'/0'/0/0`
- Solana：`m/44'/501'/0'/0'`

### 7.4 `origin` 文本（可选，仅展示）

设备确认页可展示主机名；若需显示代币信息，可用 query-string 风格附加（**不影响签名字节本身**）：

```text
MyWallet&symbol=USDT&decimals=6&amount=1000000
```

Solana 还可附加：

```text
MyWallet&symbol=SOL&decimals=9&amount=1000000&toAddress=<base58>
```

---

## 8. EVM 六链：`eth-sign-request`

六链共用协议，用 `chain_id` 区分网络。

### 8.1 请求字段

| Key | 字段 | 类型 | 必填 | 说明 |
| --- | --- | --- | --- | --- |
| `1` | `request_id` | tagged UUID (37) | 强烈建议 | 16 字节 |
| `2` | `sign_data` | bytes | 是 | 见 `data_type` |
| `3` | `data_type` | uint | 是 | 见下表 |
| `4` | `chain_id` | uint | 是 | `1/56/137/196/42161/8453` |
| `5` | `derivation_path` | tagged crypto-keypath | 是 | 完整路径 + xfp |
| `6` | `address` | bytes | 建议 | 20 字节，供设备展示 |
| `7` | `origin` | text | 否 | 见 7.4 |

| `data_type` | 含义 | `sign_data` |
| --- | --- | --- |
| `1` | Legacy Transaction | RLP(`[nonce, gasPrice, gasLimit, to, value, data, v, r, s]`)，v/r/s 置空占位 |
| `2` | EIP-712 Typed Data | UTF-8 JSON |
| `3` | personal_sign | 原始消息（不含 `\x19Ethereum Signed Message:\n` 前缀） |
| `4` | EIP-1559 | `0x02 \|\| rlp([chainId, nonce, maxPriorityFeePerGas, maxFeePerGas, gasLimit, to, value, data, accessList])` |

**转账验证 Demo 优先实现 `data_type = 4`（EIP-1559）**；部分链仍可用 Legacy（`1`）。

### 8.2 回签 `eth-signature`

| Key | 字段 | 说明 |
| --- | --- | --- |
| `1` | `request_id` | 与请求一致 |
| `2` | `signature` | 见下 |
| `3` | `origin` | 可忽略 |

| 交易类型 | 签名编码 |
| --- | --- |
| EIP-1559 | `r(32) \|\| s(32) \|\| v(1)`，`v` 为 y-parity（0/1） |
| Legacy | `r(32) \|\| s(32) \|\| v(4, big-endian)`，已含 EIP-155 |
| 消息类 | 65 字节 secp256k1 |

热端：填回 `r/s/v` → 序列化 raw → 本地 recover 地址校验 → 广播。

### 8.3 EVM 转账端到端步骤（提纲）

> 覆盖构造待签码 → 硬件签名 → 扫回签 → 广播的完整联调顺序；非正式协议约束，字段与编码以 8.1 / 8.2 为准。

```text
1. 选网络 chainId，选 from = 导入地址 index 0
2. 拉 nonce / 建议 gas（自有 RPC）
3. 构造未签 EIP-1559 字节 → eth-sign-request
4. 展示 UR 动画
5. 扫 eth-signature → 组装 → eth_sendRawTransaction
```

---

## 9. Bitcoin / Litecoin / Dogecoin：`crypto-psbt`

### 9.1 标准路径：`crypto-psbt`（推荐自研热钱包主路径）

| 项 | 说明 |
| --- | --- |
| UR type | `crypto-psbt`（tag `310`） |
| CBOR | **单段 bytes** = BIP-174 PSBT 二进制 |
| 请求 | 热端构造的**未签名** PSBT（须含 UTXO / 派生等字段，见 9.1.1） |
| 回签 | 同为 `crypto-psbt`；CBOR 仍为单段 PSBT bytes，但对应输入已写入签名字段（见下） |

**设备回签写入的签名字段（与固件行为一致）：**

| 脚本 | 回签 PSBT 输入字段 | 说明 |
| --- | --- | --- |
| Legacy / Nested SegWit / Native SegWit / DOGE | BIP-174 `partial_sigs` | map：**key** = 压缩公钥；**value** = DER 签名 ‖ sighash 类型字节。单签转账常见 `SIGHASH_ALL = 0x01` |
| Taproot (P2TR) key-path | BIP-371 `tap_key_sig` | Schnorr 签名；勿期待再走普通 `partial_sigs` |

全局 / 输入 / 输出上的 UTXO、脚本、`bip32_derivation` 等元数据一般原样保留，供热端 `finalize`。

**热端在收到回签之后（摘要；逐步说明见 9.1.2）：** 解析已签 PSBT → 核对与当前待签会话一致且签名字段齐全 → `finalize`（生成 `final_scriptsig` / `final_scriptwitness`）→ `extractTransaction` 得到网络层 raw → 自有节点 / 浏览器 API 广播。**编解码与地址所用网络参数必须对应币种**（不可用 Bitcoin mainnet 参数处理 LTC / DOGE）。

#### 9.1.1 待签 PSBT 每个输入应具备的字段（BIP-174 / BIP-371）

设备侧靠这些字段识别「花谁的钱、用哪条路径签」。缺字段或路径/xfp 不一致时，可能无法展示或拒绝签名。

| 地址 / 脚本类型 | 必填 / 强烈建议 | 说明 |
| --- | --- | --- |
| Legacy / Nested SegWit / Native SegWit | `non_witness_utxo` **或** `witness_utxo` | Legacy / DOGE 常用完整前序交易（`non_witness_utxo`）；SegWit 可用 `witness_utxo`（script + amount） |
| Nested SegWit (P2SH-P2WPKH) | 另加 `redeem_script` | 与地址脚本一致 |
| 上述非 Taproot 脚本 | `bip32_derivation` | BIP-174：**以压缩公钥为 key** 的映射；value = `master_fingerprint`（4 字节）+ 完整派生路径索引序列。逻辑上即 `{ pubkey → { fingerprint, path } }`，xfp 必须与导入一致，`path` 为完整路径（见 9.2） |
| Taproot (P2TR) | `witness_utxo` + `tap_internal_key` + `tap_bip32_derivation` | 遵循 **BIP-371**：内部密钥为 x-only（32 字节）；`tap_bip32_derivation` 同样以公钥为 key；key-path 花费时 leaf hashes 通常为空。不要把 Taproot 只填成普通 `bip32_derivation` 了事 |
| **找零输出**（输出角色，非脚本名） | 输出侧同样写 `bip32_derivation`（Taproot 用 `tap_bip32_derivation` + `tap_internal_key`） | 路径/xfp/公钥与发送地址一致。设备据此把该输出识别为找零，**只确认对外转账金额**；若省略，固件会把找零当成第二笔收款地址再弹一次签名确认 |

字段语义注意：

- `bip32_derivation` / `tap_bip32_derivation` 在 PSBT 里是 **key-value map（key = 公钥）**，不是「只有 path 的普通列表」。各语言库的 API 形状可能不同，但编码进 PSBT 后必须符合 BIP-174 / BIP-371。
- 费率估算、选 UTXO、找零等属于热端本地构造逻辑，**不是**本 UR 协议字段；实现时注意金额与费率使用整数聪（sat），避免浮点进入序列化层。

#### 9.1.2 回签后热端步骤

1. 解码回签 UR，得到已签 PSBT 字节；确认 UR type 仍为 `crypto-psbt`（若走 9.3 的 extend，则类型与 `coinId` 须与请求一致）。
2. 检查各输入是否已具备所需签名字段：非 Taproot 为 `partial_sigs`；Taproot key-path 为 `tap_key_sig`。缺签或签名无法对应 `bip32_derivation` / `tap_bip32_derivation` 中的公钥时，应中止，勿广播。
3. 调用库的 PSBT `finalize`（或等价步骤），生成各输入的 `final_scriptsig` / `final_scriptwitness`。
4. `extractTransaction`（或等价）抽出可上链的 raw 交易十六进制 / 字节。
5. 用自有全节点、RPC 或可信浏览器 API 广播；广播链路**不属于**空气隙 UR 协议本身。

#### 9.1.3 UTXO 转账端到端步骤（提纲）

> 覆盖选 UTXO → 构造待签 PSBT → 硬件签名 → 扫回签 → 广播；非正式协议约束，字段以 9.1 / 9.1.1 / 9.1.2 为准。

```text
1. 选网络（btc / ltc / doge）与脚本类型，选账户 i（路径见 9.2）
2. 拉 UTXO、估算费率，构造未签 PSBT（含找零输出派生信息）
3. 编码为 crypto-psbt → 展示 UR 动画
4. 扫回签 crypto-psbt → finalize → extractTransaction
5. 广播 raw；用浏览器或节点确认 txid
```

与 Digital Shield 热钱包一致，UTXO **账户序号**落在 BIP44 **第三段**（hardened account），每个账户默认收款地址为相对路径 `0/0`：

| 网络 | 脚本 | purpose | 账户 i 的完整路径 |
| --- | --- | --- | --- |
| Bitcoin | Legacy (P2PKH) | `44` | `m/44'/0'/i'/0/0` |
| Bitcoin | Nested SegWit (P2SH-P2WPKH) | `49` | `m/49'/0'/i'/0/0` |
| Bitcoin | Native SegWit (P2WPKH) | `84` | `m/84'/0'/i'/0/0` |
| Bitcoin | Taproot (P2TR) | `86` | `m/86'/0'/i'/0/0` |
| Litecoin | Legacy / Nested / Native | `44` / `49` / `84` | `m/{purpose}'/2'/i'/0/0` |
| Dogecoin | Legacy (P2PKH) | `44` | `m/44'/3'/i'/0/0` |

签名时 `bip32_derivation.path` 必须是上表**完整路径**（含末尾 `/0/0`），且 fingerprint 与导入 xfp 一致。

### 9.3 可选：`crypto-psbt-extend`（LTC / DOGE，Keystone 风格）

部分生态用扩展类型显式带 coinId：

| 项 | 值 |
| --- | --- |
| UR type | `crypto-psbt-extend` |
| Tag | `312` |
| CBOR Map | `1` = PSBT bytes；`2` = `coinId`（Litecoin=`2`，Dogecoin=`3`） |
| 回签 | 同类型，带回已签 PSBT + 相同 `coinId` |

自研验证 Demo：**优先 `crypto-psbt`**；若需兼容已有 Keystone LTC/DOGE 码流，再实现 extend。

---

## 10. TRON：`tron-sign-request`

### 10.1 请求

| Key | 字段 | 说明 |
| --- | --- | --- |
| `1` | `request_id` | UUID |
| `2` | `sign_data` | 交易：`raw_data` 字节（或约定的 `raw_data_hex` 解码结果）；消息：原始消息 |
| `3` | `derivation_path` | 如 `m/44'/195'/0'/0/0` + xfp |
| `4` | `address` | 可选 |
| `5` | `origin` | 可选 |
| `6` | `request_type` | `1` 交易 · `2` Msg V1 · `3` Msg V2 |

转账验证 Demo 使用 **`request_type = 1`**。原生 TRX 用 `wallet/createtransaction`；**USDT(TRC20)** 用 `wallet/triggersmartcontract` 调用主网合约 `TR7NHqjeKQxGTCi8q8ZY4pL8otSzgjLj6t` 的 `transfer(address,uint256)`（6 位小数），`sign_data` 仍为返回的 `raw_data` 字节。`origin` 建议带 `symbol=USDT&decimals=6&amount=<最小单位>`，供设备展示。TRC20 仍需少量 TRX 支付能量，不会扣 USDT 以外的 TRX 本金。

### 10.2 回签 `tron-signature`

| Key | 字段 |
| --- | --- |
| `1` | `request_id` |
| `2` | `signature`（65 字节 secp256k1） |

将签名放入交易 JSON 的 `signature` 数组后广播。`txid` 应对 `raw_data` 做 SHA-256，并与构造时一致。

---

## 11. Solana：`sol-sign-request`

### 11.1 请求

字段布局与 TRON 类似（Key `1–6`）。`request_type`（Key `6`）取值如下：

| 值 | 名称 | 含义 | `sign_data` |
| --- | --- | --- | --- |
| `1` | Transaction | 未签名交易 | 未签名 message（legacy `serializeMessage()` 或 Versioned `message.serialize()`） |
| `2` | Unsafe Message | 原始消息签名 | 原始消息字节 |
| `3` | Off-chain Message (Legacy) | Solana off-chain 消息（旧格式） | 按 Solana off-chain Legacy 规范编码 |
| `4` | Off-chain Message (Standard) | Solana off-chain 消息（标准格式） | 按 Solana off-chain Standard 规范编码 |

转账验证 Demo：`request_type = 1`（Transaction），路径 `m/44'/501'/0'/0'`。

### 11.2 回签 `sol-signature`

| Key | 字段 |
| --- | --- |
| `1` | `request_id` |
| `2` | `signature`（64 字节 ed25519） |

附加到交易后广播，并用导入公钥验签。

---

## 12. 热钱包模块设计建议

建议按模块拆分，便于 Demo → 正式产品演进：

```text
┌─────────────────────────────────────────────────────────┐
│                     Hot Wallet App                        │
├──────────────┬──────────────────┬─────────────────────────┤
│ AccountImport│  TxBuilder       │  AirGapSession          │
│ 扫 multi-acc │  各链未签交易     │  encode request QR      │
│ 解析 HDKey   │  (RPC/UTXO/…)    │  decode signature QR    │
│ 派生地址     │                  │  pendingSign 状态机     │
├──────────────┴──────────────────┴─────────────────────────┤
│ UR Codec（bc-ur） + Registry（本文 CBOR Map）              │
├───────────────────────────────────────────────────────────┤
│ Broadcast / Explorer（自有节点，非空气隙协议一部分）        │
└───────────────────────────────────────────────────────────┘
```

**UI 最小页面：**

1.「添加硬件钱包」→ 扫码导入  
2. 资产列表（观察地址）  
3. 转账表单 →「硬件签名」→ 展示待签码  
4.「扫描签名结果」→ 成功页 / txid  

**相机注意：**

- 导入与回签：热端相机扫设备屏。
- 待签：热端**亮屏展示**二维码，设备相机扫热端；注意亮度、勿加过大中心 Logo 遮挡模块。

---

## 13. Demo 验证方案（推荐落地顺序）

目标：用最小可行热钱包证明与真机联通，再铺开 11 链。

### 阶段 A — 账户导入（1–2 天）

| 步骤 | 验收标准 |
| --- | --- |
| A1 扫 `crypto-multi-accounts` | 动画扫满，解析出 xfp |
| A2 解析 ETH + BTC Native + SOL + TRX HDKey | 地址与设备/官方 App 一致 |
| A3 持久化 | 杀进程后仍能显示观察地址 |

### 阶段 B — 单链转账打通（优先 ETH）

| 步骤 | 验收标准 |
| --- | --- |
| B1 构造测试网或小额主网 EIP-1559 转账 | `eth-sign-request` 可被设备识别 |
| B2 设备确认页金额/地址正确 | 用户可确认或取消 |
| B3 扫 `eth-signature` | `request_id` 匹配，本地验签通过 |
| B4 广播 | 浏览器可查到 tx |

### 阶段 C — 铺开其余链

建议顺序：

1. **EVM 其余五链**（只改 `chain_id` + RPC）  
2. **Bitcoin** Native SegWit PSBT 小额  
3. **TRON** 原生 `tron-sign-request`  
4. **Solana**  
5. **Litecoin / Dogecoin** PSBT  

每条链至少跑通：导入地址正确 → 待签可扫 → 回签可扫 → 验签 → 广播（或测试网等价）。

### 阶段 D — 产品化

- 错误码与多语言提示（扫码超时、xfp 不匹配、用户取消）
- 待签码亮度 / 分片长度可配置
- 与现有钱包账户体系合并（标签「Digital Shield 空气隙」）

### 验证Demo 技术选型参考（非强制）

| 层 | 可选 |
| --- | --- |
| Web Demo | 浏览器 + `jsqr` / `html5-qrcode` + `@ngraveio/bc-ur` 等 |
| RN / 移动 | `digitalshield-airgap-sdk` 的 `@digitalshield/airgap-hot` + 相机组件 |
| 链库 | ethers / viem、bitcoinjs-lib、tronweb、@solana/web3.js |

---

## 14. 安全校验清单

接入方至少实现：

1. **xfp 一致**：请求路径中的 fingerprint = 导入 `masterFingerprint`。  
2. **request_id 一致**：回签 UUID 匹配当前 `pendingSign`。  
3. **本地验签**：用导入公钥验证签名；EVM recover 地址须等于展示地址。  
4. **链参数正确**：EVM `chain_id`；UTXO 网络魔法数 / 地址前缀；勿跨链复用 PSBT。  
5. **用户所见即所签**：热端展示的 to / amount 与送入 UR 的未签交易一致；硬件屏二次确认。  
6. **超时与取消**：用户取消或超时清除 `pendingSign`，禁止误绑下一笔回签。  
7. **无私钥**：热端存储仅为公钥 / xpub / 地址元数据。

---

## 15. 常见问题与排障

| 现象 | 可能原因 | 处理 |
| --- | --- | --- |
| 导入进度一直 0% | 分片过大导致二维码过密；刷新过快；环境光差 | 确认设备固件已用较小 `max_fragment_len`；热端侧提高曝光、稳定持机 |
| 设备报无法识别 / ECC 错误 | 热端待签码过密或反光 | 摄像头有效分辨率有限 + 单帧 QR 过密 **详见 [15.1](#151-硬件扫描热端交易码失败摄像头分辨率--二维码过密)** |
| 设备拒绝签名 | xfp 或路径与当前钱包不符；Passphrase 钱包未导出对应账户 | 重新导入当前钱包的 multi-accounts |
| 回签扫到了但对不上交易 | `request_id` 未校验或会话被覆盖 | 强制校验 UUID；同时只允许一笔 pending |
| LTC/DOGE 签名异常 | 误用 Bitcoin 网络参数 | 检查 address version / PSBT 网络 |
| EVM 广播失败 | `chain_id` 与节点不符；v 编码弄错（EIP-1559 y-parity vs Legacy） | 按第 8.2 节区分 data_type |
| TRON 与某交易所 App 码不兼容 | 对方可能走 Keystone gzip 路径 | 自研用 `tron-sign-request`；兼容层另议 |

### 15.1 硬件扫描热端交易码失败（摄像头分辨率 / 二维码过密）

联调中较常见的一类问题是：**热钱包已正确生成 UR 待签动画码，协议字段也无误，但 Digital Shield 硬件摄像头「扫不上」或扫很久仍无法进入确认页**。这往往不是链上交易构造错误，而是**光学识读能力与单帧二维码密度不匹配**。


#### 典型现象（屏幕侧可观察）

| 阶段 | 热钱包开发者在设备上通常看到 |
| --- | --- |
| 一直找不到码 / 码过密 | 停留在扫码页，进度长期为 **0%** 或不前进；预览里似乎有码，但始终进不了交易确认页 |
| 偶发识读不稳 | 进度偶有跳动又回落，长时间无法到 100%；缩小分片或放大码面后明显好转 |
| 与载荷相关 | **BTC / LTC / DOGE 的 `crypto-psbt`（尤其多输入、较大 PSBT）** 最容易复现；单帧偏「挤」的 ETH / TRON / SOL 动画码在手机高分屏上也偶发 |
| 对照 | 同一台设备扫**稀疏**短码往往正常；换更大展示区域 / 更小 `max_fragment_len` 后成功率明显上升 |

对接 OKX 等手机端高密度动画 UR（部分 BTC 待签单帧可到约 **QR Version 12–17** 量级）时，历史上也出现过上述「扫很久进不了确认页」的情况。

#### 原因说明（结合硬件能力）

1. **硬件摄像头有效解码分辨率有限**  
   传感器采集为 VGA（约 640×480），扫码时再**中心裁剪**送入解码器（固件常见裁剪边长约 **360×360**）。手机全屏高分待签码若单帧模块过密，落在裁剪区域内的「每模块像素数」不足，定位或采样就会失败。

2. **单帧信息量过大 → QR 版本升高 → 模块变密**  
   UR 分片的 `max_fragment_len` 过大时，每一帧 Bytewords 变长，生成的静态 QR 版本升高、黑白格更碎。手机 OLED 子像素排列、反光、自动亮度还会进一步恶化边缘对比度。

3. **动画刷新与对焦窗口**  
   分片刷新过快时，硬件来不及稳定取景；过慢则体验差，但不解决「单帧过密」本身。

4. **与「协议 / 编码错误」的区分（看屏幕弹窗提示）**  

| 类型 | 屏幕表现 | 含义 |
| --- | --- | --- |
| 光学问题（本节） | 进度卡住、**无弹窗**或始终无法扫满 | 单帧过密 / 光照 / 刷新；先减小分片 |
| 钱包不匹配 | 弹窗类似 **Wallet mismatch / 钱包不匹配**（与 `Invalid signer` 同类：当前设备钱包与请求中的 xfp/signer 不一致） | 重新导入本机 `crypto-multi-accounts`，或核对路径与指纹 |
| 编码字段错误 | 进度能到 **100%**（UR 已收齐），随即弹窗 **Invalid QR Code / 无效的数据格式**（或「请检查输入数据」类提示） | CBOR 字段类型不对等（例如二进制被编成 `{type:"Buffer", data:[…]}`），属热端编码问题，不是摄像头密度问题 |

#### 热端（第三方钱包）建议做法

| 项 | 建议 |
| --- | --- |
| 分片长度 | 待签码优先 **`max_fragment_len` ≈ 100–150**（宁可多几帧，也不要单帧过密）；大 PSBT 尤其不要用过大的 200+ |
| 纠错等级 | 优先 **M / Q**；过密时**先减小分片**，不要只靠提高到 H |
| 画面布局 | 二维码尽量大、居中；**避免大面积中心 Logo / 贴纸**遮挡模块 |
| 屏幕 | 提高亮度、关闭深色半透明遮罩；减少强烈反光 |
| 刷新 | 约 **200–500 ms** 一帧；与设备侧账户导出常见约 250 ms 同量级 |
| 自测（无调试线） | 真机扫：短 ETH → 中等 PSBT → 大 PSBT；若进度长期 0% / 进不了确认页，即回调小分片；若扫满后弹「无效数据」，查 CBOR bytes 编码 |

#### 设备侧（对照）

- 账户导出动画已常用较小分片（约 **100** 字节），便于**手机扫硬件**；热端展示给硬件扫时，应对称采用相近策略。  
- 固件裁剪分辨率属设备能力边界；第三方**不能假设**硬件能稳定识别「尽量少帧、尽量密」的交易所风格大码。  
- 协议解析 / 钱包不匹配类错误会在**屏幕弹窗**提示；光学失败阶段不会每帧弹窗（避免刷屏），以扫码进度与是否进入确认页为准。

#### 小结

> **硬件扫热端交易码失败时：进度卡住 → 先怀疑单帧过密；扫满后弹「无效数据」→ 查热端 CBOR 编码；弹「钱包不匹配」→ 查 xfp/账户。**  
> 光学问题处理原则：**减小每帧载荷 → 增加帧数 → 保证码面清晰**。

---

## 16. 附录：Registry / 参考实现

### 16.1 Registry 一览

| UR type | Tag | 方向 |
| --- | --- | --- |
| `uuid` | 37 | 内嵌 |
| `crypto-hdkey` | 303 | 导入 |
| `crypto-keypath` | 304 | 导入 / 签名 |
| `crypto-coin-info` | 305 | 导入 |
| `crypto-psbt` | 310 | BTC/LTC/DOGE 请求与回签 |
| `crypto-psbt-extend` | 312 | LTC/DOGE 可选 |
| `eth-sign-request` | 401 | EVM 请求 |
| `eth-signature` | 402 | EVM 回签 |
| `sol-sign-request` | 1101 | Solana 请求 |
| `sol-signature` | 1102 | Solana 回签 |
| `crypto-multi-accounts` | 1103 | 导入 |
| `tron-sign-request` | 5201 | TRON 请求 |
| `tron-signature` | 5202 | TRON 回签 |

### 16.2 外部规范

- [BCR-2020-005 UR](https://github.com/BlockchainCommons/Research/blob/master/papers/bcr-2020-005-ur.md)
- [BCR-2020-006 UR Types](https://github.com/BlockchainCommons/Research/blob/master/papers/bcr-2020-006-urtypes.md)
- [BCR-2020-007 HDKey](https://github.com/BlockchainCommons/Research/blob/master/papers/bcr-2020-007-hdkey.md)
- [BIP-174 PSBT](https://github.com/bitcoin/bips/blob/master/bip-0174.mediawiki)

---

## 接入清单（可复制到项目 Issue）

**导入**

- [ ] 扫描 `UR:CRYPTO-MULTI-ACCOUNTS/…`
- [ ] 解析并持久化 xfp / device 元数据
- [ ] 按 coin type + path + note 选取 11 链 HDKey
- [ ] 派生默认地址并与设备侧核对

**签名（每条链）**

- [ ] 生成 `request_id`
- [ ] 编码对应 Sign Request（路径带 xfp）
- [ ] 动画展示，设备可完整扫描
- [ ] 扫描回签，校验 type + `request_id`
- [ ] 本地验签并组装
- [ ] 自有 RPC 广播成功

**产品**

- [ ] 入口文案：「扫描 Digital Shield 硬件」
- [ ] 待签 / 回签双相机流程引导（见 **3.2**、**4.2**、**12**的「相机注意」：待签为热端亮码、硬件扫；回签为硬件亮码、热端扫）
- [ ] 错误与取消路径（见 **3.3**、**14** 的第 6 条：取消 / 超时须清除 `pendingSign`；验签失败、广播失败等须有提示且不可误绑下一笔回签）

---

*文档版本：与 Digital Shield 固件隔空 UR 实现对齐；若固件增加新 UR 类型，会同步修订本文。*
