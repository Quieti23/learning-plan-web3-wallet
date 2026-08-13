

# Day 02：密码学、密钥与 HD Wallet

> 学习目标：理解区块链密码学基础，掌握私钥、公钥、地址和数字签名的关系，能够解释 BIP-32/39/44 派生体系，并了解主流签名算法在各条链中的应用。  
> 建议用时：3～4 小时  
> 完成标准：能够脱离资料画出完整派生链路，并连续回答文末 7 道面试题。

## 一、学习内容

### 1. 密码学基础

#### 1.1 哈希函数

哈希函数将任意长度输入映射为固定长度输出，具备以下性质：

- **确定性**：相同输入始终产生相同输出；
- **单向性**：从输出无法逆推输入；
- **雪崩效应**：输入微小变化导致输出完全不同；
- **抗碰撞性**：难以构造两个不同输入使输出相同。

区块链中常见哈希算法：

| 算法 | 输出长度 | 主要用途 |
|---|---|---|
| SHA-256 | 256 位 | BTC 工作量证明、地址生成（双重 SHA-256） |
| RIPEMD-160 | 160 位 | BTC 地址生成（对 SHA-256 输出再做 RIPEMD-160） |
| Keccak-256 | 256 位 | ETH 地址生成、EVM 事件签名计算 |
| SHA-512 | 512 位 | BIP-32 HMAC-SHA-512 密钥派生 |
| Blake2b/Blake3 | 可变 | Solana、Zcash 等链的哈希场景 |

哈希在钱包系统中的典型用途：

- 公钥 → 地址（减少地址长度，提供额外安全层）；
- 数据完整性校验；
- 幂等键、事务 ID 的确定性生成。

#### 1.2 对称加密

同一密钥用于加密和解密，典型算法为 AES-256-GCM。

在钱包系统中的用途：

- 加密本地存储的私钥文件或密钥库（Keystore）；
- 加密数据库中的敏感字段；
- 服务间传输的对称会话加密。

对称加密不能解决密钥分发问题，但速度快、适合大量数据。

#### 1.3 非对称加密

使用公私钥对：公钥可以公开，私钥必须保密。典型算法包括 RSA 和椭圆曲线（ECC）。

在钱包系统中的核心价值：

- **密钥协商**：TLS 等协议使用非对称加密交换对称密钥；
- **数字签名**：私钥签名、公钥验签，证明操作的合法性；
- 区块链账户和地址直接基于椭圆曲线密钥对构建。

#### 1.4 数字签名

数字签名使用私钥对消息（或消息摘要）产生签名，任何持有公钥的人都可以验证该签名。

签名证明了：

1. **真实性**：签名由私钥持有者发起；
2. **完整性**：消息在签名后未被篡改；
3. **不可否认性**：签名者无法否认曾经发出该消息。

> 注意：数字签名**不保证内容保密**。消息本身通常以明文传输，签名只是附加验证。如需保密，需要额外加密。

在区块链中，签名覆盖交易所有字段（发送方、接收方、金额、费用、nonce 等），保证广播过程中没有人能修改交易内容。

---

### 2. 椭圆曲线密码学（ECC）

#### 2.1 椭圆曲线基础

椭圆曲线是满足 `y² = x³ + ax + b` 的点集（加上无穷远点）。在有限域 𝔽ₚ 上定义的椭圆曲线构成一个有限群，支持点加和标量乘法。

核心操作：

- **点加**：两点相加得到另一个点；
- **标量乘法**：`k × G`（G 为基点，k 为私钥），正向高效，逆向（求 k）计算不可行。

这就是椭圆曲线离散对数问题（ECDLP），构成密码学安全性的基础。

#### 2.2 私钥与公钥的关系

```
私钥 k：随机选取的大整数（256 位），必须保密
公钥 K = k × G：椭圆曲线点，可以公开
```

私钥 → 公钥：单向，正向可计算，逆向不可行。

比特币和以太坊都使用 `secp256k1` 曲线（`a=0, b=7`）。

#### 2.3 公钥格式

- **非压缩格式**：`04 || x || y`（65 字节）；
- **压缩格式**：`02 || x`（y 为偶数）或 `03 || x`（y 为奇数），共 33 字节；
- 生产系统通常使用压缩公钥，以节省链上空间和网络带宽。

---

### 3. 地址生成

不同链采用不同的公钥 → 地址推导方式：

#### 3.1 BTC 地址（P2PKH 示例）

```
私钥
  → secp256k1 点乘 → 公钥（压缩，33 字节）
  → SHA-256 → RIPEMD-160 → PubKeyHash（20 字节）
  → 版本字节 + PubKeyHash → SHA-256 × 2 取前 4 字节作 Checksum
  → Base58Check 编码 → 1 开头的地址
```

不同地址类型（P2SH、P2WPKH、P2WSH、P2TR）的推导稍有不同，但均以公钥或公钥哈希为基础。

#### 3.2 ETH 地址

```
私钥
  → secp256k1 点乘 → 非压缩公钥（去掉 04 前缀，64 字节）
  → Keccak-256 → 取后 20 字节
  → EIP-55 混合大小写 Checksum 编码
```

以太坊地址长度为 20 字节（40 个十六进制字符），通常加 `0x` 前缀。

#### 3.3 Solana 地址

Solana 使用 Ed25519，公钥本身就是地址：

```
私钥（32 字节随机数）
  → Ed25519 标量乘法 → 公钥（32 字节）
  → Base58 编码 → Solana 地址
```

#### 3.4 TON 地址

TON 采用合约账户模型，地址由初始合约代码和数据的哈希确定：

```
StateInit（合约代码 + 初始数据，含公钥）
  → SHA-256 → 32 字节哈希
  → workchain_id + hash → Base64url 或原始格式
```

TON 地址的 `workchain_id` 决定了主链（0）或主分片链（-1）。

---

### 4. 数字签名算法对比

#### 4.1 ECDSA（椭圆曲线数字签名算法）

适用链：BTC（Legacy、SegWit）、ETH（大多数交易）、EVM 兼容链。

签名生成过程（简化）：

1. 生成随机数 k（每次签名必须不同）；
2. 计算 R = k × G，取 r = R.x mod n；
3. 计算 s = k⁻¹ × (hash + r × privateKey) mod n；
4. 签名 = (r, s)。

主要风险：

- **随机数 k 复用**：若两次签名使用相同的 k，攻击者可以从两个签名和消息中还原私钥；
- 使用确定性签名（RFC 6979）可消除随机 k 的风险，生产系统必须使用确定性 ECDSA。

#### 4.2 Schnorr 签名

适用链：BTC Taproot（BIP-340）、Zcash、部分多签方案。

核心优势：

- **线性性**：多个签名可以聚合为一个，节省链上空间，提升隐私；
- **批量验证**：多个签名可以同时验证，效率更高；
- 安全证明比 ECDSA 更简洁，不存在 k 复用漏洞（仍需确定性实现）；
- BIP-340 使用 secp256k1，与 BTC 现有生态兼容。

#### 4.3 Ed25519

适用链：Solana、TON、Cardano、Polkadot 等。

特点：

- 基于 Edwards 曲线 Curve25519，安全性强且实现效率高；
- 签名本身是确定性的（不需要外部随机数），天然避免 k 复用问题；
- 公钥和签名长度均为 32/64 字节，紧凑；
- 不与 secp256k1 兼容，BTC/ETH 不直接使用。

#### 4.4 三种算法对比

| 维度 | ECDSA (secp256k1) | Schnorr (BIP-340) | Ed25519 |
|---|---|---|---|
| 主要用途 | BTC Legacy/SegWit、ETH | BTC Taproot | Solana、TON |
| 签名长度 | ~71 字节（DER 编码） | 64 字节 | 64 字节 |
| 签名聚合 | 不原生支持 | 支持（MuSig2） | 支持（变体） |
| 确定性签名 | 需要 RFC 6979 | BIP-340 规范确定性 | 原生确定性 |
| k 复用风险 | 高（必须规避） | 已由规范解决 | 无此问题 |
| 批量验证 | 不支持 | 支持 | 支持 |
| 安全证明 | 较复杂 | 更简洁 | 严格证明 |

---

### 5. BIP-32、BIP-39、BIP-44：HD Wallet 体系

#### 5.1 BIP-39：助记词

BIP-39 将随机熵编码为易于记忆和备份的单词序列（通常 12 或 24 个单词）。

生成流程：

```
随机熵（128～256 位）
  → SHA-256 → 取前若干位作校验和
  → 熵 + 校验和拼接
  → 按每 11 位分组，映射到 2048 个单词表
  → 助记词（12/15/18/21/24 个单词）
```

从助记词恢复 Seed：

```
助记词 + 可选 Passphrase
  → PBKDF2-HMAC-SHA-512（迭代 2048 次）
  → 512 位 Seed
```

关键点：

- 相同助记词 + Passphrase 始终得到相同 Seed；
- Passphrase 为空也是合法的；
- 助记词只是 Seed 的编码，不是私钥本身；
- 备份助记词就等于备份了所有可派生密钥。

#### 5.2 BIP-32：分层确定性密钥派生

BIP-32 定义了如何从一个 Seed 确定性地派生出密钥树：

```
Seed（512 位）
  → HMAC-SHA-512（以 "Bitcoin seed" 为 key）
  → 左 256 位：主私钥（Master Private Key）
  → 右 256 位：主链码（Master Chain Code）
```

子密钥派生（简化）：

```
父私钥 + 父链码 + 索引
  → HMAC-SHA-512
  → 左 256 位：子私钥调整量（加到父私钥得到子私钥）
  → 右 256 位：子链码
```

强化派生（Hardened Derivation）：

- 普通派生（non-hardened）：索引 0 ～ 2³¹-1，可以从公钥派生子公钥；
- 强化派生（hardened）：索引 2³¹ ～ 2³²-1（记为 0'，1'，…），必须使用私钥，更安全。

派生路径格式：`m / purpose' / coin_type' / account' / change / address_index`

- `m` 代表主密钥；
- `'` 表示强化派生；
- 数字之间用 `/` 分隔。

#### 5.3 BIP-44：多账户分层路径

BIP-44 在 BIP-32 基础上规范了派生路径语义：

```
m / 44' / coin_type' / account' / change / address_index
```

| 层级 | 含义 | 示例 |
|---|---|---|
| `44'` | Purpose，标识遵循 BIP-44 | 固定为 44' |
| `coin_type'` | 链的 SLIP-44 编号 | BTC=0'、ETH=60'、SOL=501'、TON=607' |
| `account'` | 账户隔离，强化派生 | 通常从 0' 开始 |
| `change` | 0 = 外部地址（收款），1 = 内部地址（找零） | BTC 找零用 1，ETH 通常只用 0 |
| `address_index` | 地址序号 | 0, 1, 2, … |

示例路径：

- BTC 第一个收款地址：`m/44'/0'/0'/0/0`
- ETH 第一个地址：`m/44'/60'/0'/0/0`
- Solana 第一个地址：`m/44'/501'/0'/0'`

SegWit（P2WPKH）使用 BIP-84，路径前缀为 `m/84'/0'/…`；Taproot 使用 BIP-86，路径前缀为 `m/86'/0'/…`。

#### 5.4 完整派生链路图

```
随机熵（128/256 位）
        │
        ▼
  [BIP-39] SHA-256 + 单词表
        │
        ▼
  助记词（12/24 个单词）
        │
        ▼
  [BIP-39] PBKDF2-HMAC-SHA-512 + Passphrase
        │
        ▼
  Seed（512 位）
        │
        ▼
  [BIP-32] HMAC-SHA-512("Bitcoin seed", seed)
        │
   ┌────┴────┐
   ▼         ▼
主私钥      主链码
   │
   ▼
  [BIP-32/44] 按路径逐级派生
        │
        ▼
  子私钥（例如 m/44'/60'/0'/0/0）
        │
        ▼
  [ECC] secp256k1 / Ed25519 标量乘法
        │
        ▼
  公钥
        │
        ▼
  [哈希/编码] 链特有算法
        │
        ▼
  地址（BTC / ETH / SOL / TON …）
```

#### 5.5 为什么同一助记词可以派生多条链的地址

Seed 是链无关的原始随机数。通过不同的 `coin_type'` 路径，同一 Seed 可以派生出互相独立的密钥子树：

- BTC：`m/44'/0'/…` 或 `m/84'/0'/…`；
- ETH：`m/44'/60'/…`；
- Solana：`m/44'/501'/…`；
- TON：`m/44'/607'/…`。

各链再使用各自的 ECC 曲线和地址编码算法，生成最终地址。不同 `coin_type'` 的子树之间相互隔离，泄露某链私钥不会直接危及其他链。

---

### 6. 生产钱包中的密码学风险

#### 6.1 确定性签名

生产系统必须使用确定性签名：

- ECDSA 必须遵循 RFC 6979，使用 HMAC-DRBG 生成 k，而不是 OS 随机源；
- 避免 k 复用导致私钥泄露；
- 确定性实现便于测试和审计。

#### 6.2 随机数质量

- 密钥生成和助记词熵必须使用密码学强随机源（CSPRNG）；
- Java 中使用 `SecureRandom`，不要使用 `Random`；
- 在 HSM 或安全飞地中生成密钥时，随机源由硬件提供。

#### 6.3 金额与精度处理

区块链金额处理是常见 Bug 来源：

| 风险 | 说明 | 正确做法 |
|---|---|---|
| 浮点精度 | `double` 无法精确表示十进制小数 | 使用 `BigDecimal` 或以最小单位整数存储 |
| 整数溢出 | 大金额乘以系数可能溢出 `long` | 使用 `BigInteger`；对关键计算做范围校验 |
| 单位混淆 | 把 wei 当作 ETH、sat 当作 BTC | 内部统一使用最小单位（wei、sat），只在展示层转换 |
| 精度截断 | 除法使用整数截断 | 明确舍入策略（一般向下取整）；对手续费计算加断言 |

#### 6.4 字节数组与十六进制

- 私钥、交易数据、哈希值在代码中通常以 `byte[]` 存储；
- 输入/输出和日志中常用十六进制字符串；
- 需注意：不同库可能对十六进制前缀（`0x`）和大小写处理方式不同；
- Base58、Base64、Base64url 之间不可混用，不同链使用不同编码；
- `byte[]` 与 `String` 之间必须显式指定字符集（UTF-8），避免平台差异。

#### 6.5 重放攻击与域隔离

- **重放攻击**：在一个链/网络上有效的签名，被攻击者提交到另一个链/网络上执行；
- 防御方式：在签名内容中包含 `chainId`（EIP-155）、`network`、业务唯一标识；
- TON 使用 `seqno`（与 EVM nonce 类似）防止同一消息重放；
- 测试网和主网必须严格隔离签名域，避免测试网密钥签名的交易被重放到主网。

---

### 7. Java 签名/验签示例

以下示例仅用于本地测试，使用内存中的密钥，不接触任何真实资产或真实私钥：

```java
import org.bouncycastle.crypto.params.ECPrivateKeyParameters;
import org.bouncycastle.crypto.params.ECPublicKeyParameters;
import org.bouncycastle.crypto.signers.ECDSASigner;
import org.bouncycastle.crypto.signers.HMacDSAKCalculator;
import org.bouncycastle.crypto.digests.SHA256Digest;
import org.bouncycastle.asn1.sec.SECNamedCurves;
import org.bouncycastle.asn1.x9.X9ECParameters;
import org.bouncycastle.math.ec.ECPoint;

import java.math.BigInteger;
import java.security.MessageDigest;
import java.security.SecureRandom;

/**
 * 本地 secp256k1 ECDSA 签名/验签示例（仅用于学习，不接触真实资产）
 * 依赖：bouncycastle (bcprov-jdk18on)
 */
public class Secp256k1SignDemo {

    public static void main(String[] args) throws Exception {
        X9ECParameters params = SECNamedCurves.getByName("secp256k1");
        var domainParams = new org.bouncycastle.crypto.params.ECDomainParameters(
                params.getCurve(), params.getG(), params.getN(), params.getH());

        // 1. 生成随机私钥（仅测试用）
        SecureRandom rng = new SecureRandom();
        byte[] privKeyBytes = new byte[32];
        rng.nextBytes(privKeyBytes);
        BigInteger privKey = new BigInteger(1, privKeyBytes);

        // 2. 推导公钥
        ECPoint pubKeyPoint = params.getG().multiply(privKey).normalize();

        // 3. 准备消息摘要
        String message = "transfer:to=0xABC,amount=100";
        byte[] msgHash = MessageDigest.getInstance("SHA-256").digest(message.getBytes());

        // 4. 使用确定性 ECDSA (RFC 6979) 签名
        ECDSASigner signer = new ECDSASigner(new HMacDSAKCalculator(new SHA256Digest()));
        signer.init(true, new ECPrivateKeyParameters(privKey, domainParams));
        BigInteger[] sig = signer.generateSignature(msgHash);
        System.out.printf("签名 r = %s%n", sig[0].toString(16));
        System.out.printf("签名 s = %s%n", sig[1].toString(16));

        // 5. 验签
        ECDSASigner verifier = new ECDSASigner();
        verifier.init(false, new ECPublicKeyParameters(pubKeyPoint, domainParams));
        boolean valid = verifier.verifySignature(msgHash, sig[0], sig[1]);
        System.out.println("验签结果：" + valid); // 应输出 true

        // 安全提示：检查代码和 Git 状态，确认没有真实私钥或助记词进入仓库
    }
}
```

> **安全检查**：所有链上实践仅在测试网进行；私钥、助记词、API Key 不得提交到仓库。

---

## 二、密钥与地址术语表

| 术语 | 含义 |
|---|---|
| 私钥 | 椭圆曲线上的大整数，控制账户并用于生成签名 |
| 公钥 | 私钥乘以曲线基点得到的点，可以公开，用于验签和派生地址 |
| 地址 | 公钥经过哈希或直接编码得到的链上标识符 |
| secp256k1 | BTC 和 ETH 使用的椭圆曲线 |
| Ed25519 | Solana、TON 使用的 Edwards 曲线签名算法 |
| ECDSA | 基于椭圆曲线的数字签名算法，BTC/ETH 主要签名方案 |
| Schnorr 签名 | 可聚合的签名算法，BTC Taproot 使用 |
| 数字签名 | 证明操作由私钥持有者发起且内容未被篡改 |
| 哈希函数 | 单向、确定性、抗碰撞的映射函数 |
| SHA-256 | BTC 主要使用的哈希算法 |
| Keccak-256 | ETH 使用的哈希算法（非标准 SHA3） |
| 确定性签名 | 签名时 k 由消息和私钥确定性生成（RFC 6979），避免随机 k 复用风险 |
| 助记词 | 按 BIP-39 标准编码的单词序列，用于备份和恢复 Seed |
| Seed | 由助记词 + Passphrase 通过 PBKDF2 生成的 512 位随机数 |
| HD Wallet | 分层确定性钱包，从一个 Seed 可派生无限数量密钥 |
| BIP-32 | 定义分层确定性密钥派生算法的规范 |
| BIP-39 | 定义助记词生成与 Seed 推导的规范 |
| BIP-44 | 定义多账户、多链派生路径语义的规范 |
| 主密钥 | 由 Seed 派生的顶级密钥，是整个密钥树的根 |
| 链码 | BIP-32 中与私钥配合用于子密钥派生的 256 位随机数 |
| 强化派生 | 索引 ≥ 2³¹ 的派生，必须使用私钥，子公钥无法从父公钥推导 |
| 普通派生 | 索引 < 2³¹ 的派生，可以只用公钥推导子公钥 |
| coin_type | BIP-44/SLIP-44 中标识不同区块链的数字编号 |
| PBKDF2 | 基于密码的密钥派生函数，BIP-39 使用 HMAC-SHA-512 变体 |
| Passphrase | BIP-39 可选的额外密码，参与 Seed 生成，不同 Passphrase 产生不同 Seed |
| CSPRNG | 密码学强随机数生成器，密钥生成必须使用 |
| RFC 6979 | 规定 ECDSA 确定性 k 生成方式的标准，消除随机 k 复用风险 |
| EIP-155 | ETH 在签名中加入 chainId 以防止跨链重放攻击 |
| Keystore | 加密存储私钥的文件格式，解锁需要密码 |
| Base58Check | BTC 地址使用的带校验和的 Base58 编码 |
| 重放攻击 | 将在一个链/网络有效的签名提交到另一个链/网络执行的攻击 |
| 域隔离 | 在签名内容中包含 chainId 或网络标识，防止跨域重放 |

---

## 三、口头面试题参考答案

> 使用方法：先脱离答案口述，再对照要点补充。不要逐字背诵。

### 1. 私钥、公钥、地址三者如何关联？

**参考回答：**

三者是单向推导关系。私钥是一个随机大整数，通过椭圆曲线点乘（`k × G`）得到公钥，不可逆。公钥再经过链特有的哈希和编码算法得到地址，也不可逆。

因此只有持有私钥才能签名，而验签和收款只需要公钥和地址。私钥的安全性等价于账户的完整控制权，一旦泄露就无法恢复。

不同链的地址推导算法有差异：BTC 用 SHA-256 + RIPEMD-160 + Base58Check；ETH 用 Keccak-256 取后 20 字节；Solana 的公钥本身就是地址（Ed25519，32 字节 Base58）；TON 地址来自合约 StateInit 的哈希。

### 2. 数字签名证明了什么？它能否保证交易内容保密？

**参考回答：**

数字签名证明三件事：真实性（签名由私钥持有者发出）、完整性（签名后内容未被篡改）、不可否认性（签名者无法抵赖）。

但数字签名不保证内容保密。交易通常以明文在 P2P 网络广播，任何节点都可以读取内容，签名只是附加的验证机制。如果需要保密，必须额外对数据加密。

在区块链中，签名覆盖了交易的完整字段（目标地址、金额、nonce、链 ID 等），任何字段被篡改都会导致验签失败，因此保障了交易内容的完整性。

### 3. BIP-39 和 BIP-32 分别解决什么问题？

**参考回答：**

BIP-39 解决密钥备份问题。它把随机熵编码成便于人类抄写和记忆的单词序列（助记词），再通过 PBKDF2 生成 512 位 Seed。助记词可以印在纸上或刻在金属板上离线保存，大幅降低备份门槛。

BIP-32 解决密钥管理和扩展问题。它定义了如何从单个 Seed 确定性地派生出一棵密钥树，支持无限数量的子密钥，且任意子密钥都可以随时从 Seed 重新推导出来。这样，一个 Seed 的备份就覆盖了所有派生密钥。

两者配合：BIP-39 负责产生和备份 Seed，BIP-32 负责从 Seed 按路径派生实际使用的私钥。

### 4. 为什么同一个助记词可以派生多条链的地址？

**参考回答：**

助记词通过 PBKDF2 得到的 Seed 是链无关的原始随机数。BIP-32 将其作为根，按照不同路径分支构建密钥树。BIP-44 规范了路径的 `coin_type'` 层级，不同链使用 SLIP-44 注册的不同编号，例如 BTC 是 0'，ETH 是 60'，Solana 是 501'。

通过不同的 `coin_type'` 路径，同一 Seed 派生出完全独立、互不干扰的子密钥树。每条链的子私钥再按各自的曲线和地址算法生成最终地址。各子树之间隔离：即使泄露了某条链的一个子私钥，也无法推导其他链或其他账户的密钥（强化派生进一步加固了这一点）。

### 5. ECDSA 的随机数被复用会造成什么后果？

**参考回答：**

后果是私钥直接泄露。ECDSA 签名时生成随机数 k，计算 r = (k×G).x，s = k⁻¹(hash + r × privateKey) mod n。如果两次签名使用了相同的 k，则有两组方程：s₁ = k⁻¹(h₁ + r × pk)，s₂ = k⁻¹(h₂ + r × pk)。由于 r 相同，攻击者可以从 s₁ - s₂ 还原 k，再用 k 反推私钥。

历史上 PlayStation 3 的 ECDSA 实现因 k 为固定值，导致私钥被公开还原。Mt. Gox 早期 BTC 库中也有 k 复用相关问题。

解决方案是使用 RFC 6979：根据私钥和消息通过 HMAC-DRBG 确定性生成 k，无需外部随机源，不存在复用风险，且可验证和测试。

### 6. BTC Taproot、ETH、Solana/TON 分别主要使用什么签名算法？

**参考回答：**

- **BTC Legacy/SegWit**：secp256k1 ECDSA；
- **BTC Taproot**：secp256k1 Schnorr（BIP-340），支持签名聚合和 MuSig2 多签，提升隐私和链上效率；
- **ETH**：secp256k1 ECDSA，交易签名中包含恢复参数 v 以方便从签名还原公钥；
- **Solana**：Ed25519（基于 Curve25519），天然确定性签名；
- **TON**：Ed25519，签名覆盖 seqno 和消息内容防止重放。

选择这些算法的原因在于：secp256k1 是 BTC/ETH 历史遗留的成熟选择；Schnorr 在 secp256k1 上提供聚合能力；Ed25519 在现代链中被选用是因为实现更简洁、天然确定性且批量验证高效。

### 7. 为什么生产系统不能自行发明加密算法或签名格式？

**参考回答：**

密码学的安全性依赖于长期、大量、公开的学术分析和攻击检验。自行发明的算法没有经过这种检验，很可能存在设计缺陷或实现漏洞，而开发者通常没有能力在上线前发现这些问题。

历史上因自研或错误实现加密算法导致资产损失的案例非常多：随机数质量不足、签名格式错误、哈希迭代不足、密码学参数选择错误等都造成过严重事故。

正确做法是使用经过广泛审计的标准算法（secp256k1、Ed25519、AES-256-GCM 等）和权威库（如 BouncyCastle、libsodium），并严格遵循相关 BIP/EIP/RFC 规范。如果标准库不满足需求，应先寻求社区或安全专家意见，而不是自行实现。

---

## 四、当天任务

- [ ] 画出"助记词 → Seed → 主密钥 → 子私钥 → 公钥 → 地址"的完整派生图。
- [ ] 制作 ECDSA、Schnorr、Ed25519 对比表（参考本文第 4.4 节）。
- [ ] 运行 Java 签名/验签示例，记录 r、s 输出格式和验签结果。
- [ ] 检查代码和 Git 状态，确认没有私钥、助记词或 API Key 被提交。
- [ ] 不看资料口头回答全部 7 道题，每题控制在 2～5 分钟。
- [ ] 将薄弱点记录到 `progress.md`。
