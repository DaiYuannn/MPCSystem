# MPCSystem 加密算法与系统流程总览

> 日期：2025-09-24

本文件汇总本仓库内使用/实现的加密与隐私计算算法、对应源码文件位置、职责说明，并提供端到端的业务流程图（含加密→传输→同态运算→解密标注）。

---

## 一、加密与隐私计算算法清单

- Paillier 同态加密（整数加法同态）
  - 主要文件：
    - enterprise/myapp/paillier.py（算法实现：密钥生成、加解密、工具函数）
    - client/myapp/paillier.py（同上，函数签名一致）
    - government/myapp/paillier.py（同上，函数签名一致）
  - 关键函数：key_generation, paillier_generation, paillier_encryption, paillier_decryption
  - 能力：加法同态；支持线性函数同态计算 a1*x1 + a2*x2 + c。

- 基于格的自研同态方案（HLP / HLP_other，矩阵同态）
  - 主要文件：
    - enterprise/myapp/CC_getongtaiN.py（HLP 实现：矩阵运算、密钥生成、加解密、同态加法）
    - enterprise/myapp/NCC_getongtai2N.py（HLP 变体实现）
    - client/myapp/CC_getongtaiN.py、client/myapp/NCC_getongtai2N.py（对应实现）
    - government/myapp/CC_getongtaiN.py、government/myapp/NCC_getongtai2N.py（对应实现）
  - 关键函数：
    - key_generartion（生成公钥结构 Mhard/Msoft/random_numbers 以及私钥 Deta/A/B 等）
    - lattice_encryption（加密）/ lattice_decryption（解密）
    - MatrixAdd/MatrixDot/matrix_inv 等矩阵/模运算工具
  - 能力：向量/矩阵域的加法同态、支持线性组合；提供 two_to_long/long_to_two 用于整数与比特向量的转换。

- BFV（整数同态，支持加/乘；依赖远端 PySEAL 服务）
  - 主要文件：enterprise/myapp/BFV.py
  - 关键函数：BFV_kengen, BFV_Encrypt, BFV_Decrypt, BFV_add, BFV_sub, BFV_mul, BFV_squr, BFV_pow
  - 说明：所有运算均通过 HTTP 调用 http://39.102.39.63:9000/seal/* 服务完成；密钥管理通过 government/client 侧视图进行下发和更新。

- CKKS（近似同态，浮点/实数向量；依赖远端 PySEAL 服务）
  - 主要文件：enterprise/myapp/CKKS.py
  - 关键函数：CKKS_kengen, CKKS_Encrypt, CKKS_Decrypt, CKKS_add, CKKS_sub, CKKS_mul, CKKS_mul1, CKKS_squr
  - 说明：同样通过 HTTP 接口与远端服务交互。

- 风险模型同态封装（把上述算法统一成多项式/线性形态）
  - 主要文件：enterprise/myapp/RiskModelFunction.py
  - 关键函数：
    - Paillier_HomAdd / Paillier_HomAdd_More：线性同态组合（a1*x1 + a2*x2 + c / 多元一次）
    - HLP_HomAdd / HLP_HomAdd_More：HLP 上的线性同态组合
    - BFV_Hom / BFV_Hom_More：BFV 上的二元二次/多元高次同态（通过远端乘法/加法封装）
    - CKKS_Hom / CKKS_Hom_More：CKKS 上的二元二次/多元高次同态

---

## 二、各目录与核心文件职责说明

- enterprise/
  - myapp/
    - paillier.py：Paillier 核心算法实现。
    - CC_getongtaiN.py / NCC_getongtai2N.py：HLP/HLP_other 核心实现（矩阵/格，含密钥生成与加解密）。
    - BFV.py / CKKS.py：对远端 PySEAL 的 HTTP 封装（密钥生成、加/解密和运算）。
    - RiskModelFunction.py：将各同态算法包装为线性/多项式计算接口；供视图调用。
    - views.py：对外提供接口以执行运算（getQuota_linear / getQuota_multiple 等），路由来自 mysite/urls.py。
    - models.py：业务模型（如工资、用户、加密参数等）。
    - middlewares.py：示例化中间件（当前未启用校验）。
  - mysite/：Django 配置（settings/urls/wsgi）。

- government/
  - myapp/
    - views.py：
      - renew_publicKey：根据当前选定算法生成或从远端获取“公钥（或等效参数）”并写入数据库；并回调 enterprise 的 API 更新“私钥”。
      - getQuota：作为“解密端”，接收 enterprise 返回的密文结果，根据 enalg 类型选择相应私钥和解密流程，输出明文。
      - get_info：对工资数据进行加密后下发给前端/业务方。
    - paillier.py、CC_getongtaiN.py、NCC_getongtai2N.py：算法同 enterprise 一致，便于本地解密。
    - models.py：包含公钥/私钥存储模型、策略配置 En_Algorithm、函数系数等。

- client/
  - myapp/
    - views.py：作为“企业侧/计算侧”对接政府的 API，发起密文计算请求（按策略把输入先加密）。
    - paillier.py、CC_getongtaiN.py、NCC_getongtai2N.py：算法同 enterprise 一致，用于本地加密（如需要）。
    - models.py：与前端/用户交互相关的业务数据模型。

- helianthus/：前端 Vue 工程（不包含加密算法，仅进行页面交互、调用后端 API）。

---

## 三、端到端流程（加/解密标注）

说明：以下以“线性函数 a1*ss + a2*pf + c”的一次同态计算为例（enalg=1~5 分别代表 Paillier / HLP / HLP_other / BFV / CKKS）。

```mermaid
flowchart LR
    subgraph Frontend[前端 helianthus]
        UI[用户请求指标]
    end

    subgraph Enterprise[企业侧 enterprise]
        EV[views.getQuota_linear/multiple]
        RMF[RiskModelFunction 同态封装]
    end

    subgraph Government[政府侧 government]
        GV[views.getQuota]
        Decrypt[按 enalg 选择解密]
        Store[(私钥/策略/函数系数模型)]
    end

    UI -->|发起查询 (GET /opens/getQuota)| GV
    GV -->|读取策略 En_Algorithm/函数系数| Store
    GV -->|生成/查公钥或参数| Store
    GV -->|加密输入数据 pf/ss → 密文| Enterprise

    EV -->|接收加密数据| RMF
    RMF -->|同态运算 (a1*ss + a2*pf + c)| EV
    EV -->|返回密文结果| GV

    GV --> Decrypt
    Decrypt -->|解密结果 → 明文| GV
    GV -->|返回前端（明文结果）| UI

    classDef enc fill:#e3f2fd,stroke:#64b5f6,stroke-width:1px;
    classDef dec fill:#fce4ec,stroke:#f06292,stroke-width:1px;
    classDef comp fill:#e8f5e9,stroke:#66bb6a,stroke-width:1px;

    GV:::enc
    EV:::comp
    RMF:::comp
    Decrypt:::dec
```

方向与文件标注：
- 加密路径（Government → Enterprise）：
  - Paillier：government/myapp/views.py 中 get_info/getQuota 使用 paillier.paillier_encryption
  - HLP/HLP_other：government/myapp/views.py 中 get_info/getQuota 使用 HLP.lattice_encryption / HLP_other.lattice_encryption
  - BFV/CKKS：government/myapp/views.py 调远端 http://39.102.39.63:9000/seal/* 进行 Encrypt
- 同态计算（Enterprise 内部）：
  - enterprise/myapp/RiskModelFunction.py 中 Paillier_HomAdd / HLP_HomAdd / BFV_Hom / CKKS_Hom 等
  - enterprise/myapp/views.py 中 getQuota_linear/getQuota_multiple 路由到 RMF
- 解密路径（Enterprise → Government）：
  - government/myapp/views.py 中 getQuota 根据 enalg 调用对应解密：
    - Paillier：paillier_decryption（government/myapp/paillier.py）
    - HLP/HLP_other：lattice_decryption + two_to_long
    - BFV/CKKS：远端 Decrypt API

---

## 四、同态接口对照（按算法）

- Paillier（整数加法同态）
  - Encrypt：paillier_encryption(m, g, r, n)
  - Decrypt：paillier_decryption(c, g, lamda, n, u)
  - 同态线性：RiskModelFunction.Paillier_HomAdd / Paillier_HomAdd_More

- HLP / HLP_other（矩阵/格，线性同态）
  - KeyGen：key_generartion -> (N, l, Mhard, Msoft, random_numbers, p/mods, Deta, A, B, q, ...)
  - Encrypt：lattice_encryption(plaintext, Mhard, Msoft, random_numbers, N, ns, mods)
  - Decrypt：lattice_decryption(cipher, Deta, A, B, N, mods, q) + two_to_long
  - 同态线性：RiskModelFunction.HLP_HomAdd / HLP_HomAdd_More

- BFV（加/乘同态，经远端服务）
  - Encrypt：BFV_Encrypt(x, public_key)
  - Decrypt：BFV_Decrypt(x_encrypted, secret_key)
  - 运算：BFV_add / BFV_mul 等
  - 封装：RiskModelFunction.BFV_Hom / BFV_Hom_More

- CKKS（近似同态，经远端服务）
  - Encrypt：CKKS_Encrypt(x, public_key)
  - Decrypt：CKKS_Decrypt(x_encrypted, secret_key)
  - 运算：CKKS_add / CKKS_mul 等
  - 封装：RiskModelFunction.CKKS_Hom / CKKS_Hom_More

---

## 五、文件用途逐一说明（加密相关）

- enterprise/myapp/paillier.py：Paillier 核心算法与工具（gcd、lcm、扩展欧几里得、快速幂）。
- enterprise/myapp/CC_getongtaiN.py：HLP 主实现（矩阵/模计算、密钥生成、加解密）。
- enterprise/myapp/NCC_getongtai2N.py：HLP 变体实现（含 Deta 左右合成方式差异）。
- enterprise/myapp/BFV.py：经 HTTP 访问远端 PySEAL 完成 BFV 的 KeyGen/Encrypt/Decrypt/Add/Mul。
- enterprise/myapp/CKKS.py：经 HTTP 访问远端 PySEAL 完成 CKKS 的 KeyGen/Encrypt/Decrypt/Add/Mul。
- enterprise/myapp/RiskModelFunction.py：把 Paillier/HLP/BFV/CKKS 统一封装为线性/多项式型接口以供视图使用。
- enterprise/myapp/views.py：对外的 REST 端点，接收加密输入，调用 RiskModelFunction 完成同态计算并返回密文结果。
- government/myapp/views.py：策略切换、密钥下发、生成公钥参数、接收企业侧密文结果并解密为明文返回前端。
- client/myapp/views.py：企业侧 Web 端，向政府端请求加密数据并转发到 enterprise 做运算（或直接本地加密）。
- client/government/*/myapp/CC_getongtaiN.py、NCC_getongtai2N.py、paillier.py：与 enterprise 同步的算法实现，保证各侧可独立进行加/解密。

---

## 六、注意事项与安全建议

- BFV/CKKS 依赖外部服务：http://39.102.39.63:9000/seal/*。生产环境应配置内网或可信服务，并校验接口返回。
- government 侧保存私钥（或等效解密参数），enterprise 侧仅持有公钥（或等效加密参数），确保密文计算不泄露明文。
- 风险模型计算前有时会将浮点扩大 100 倍再转整数加密（见 get_info/getQuota 中的 *100 处理），解密后需按 LDivisor/MDivisor 缩放回去。
- HLP 的 two_to_long/long_to_two 负责整数与比特向量的互转，需保证 N 足够容纳上界，避免溢出。

---

如需我根据最新策略/接口再输出一张“多元高次（BFV/CKKS）”的运算流程图，请告诉我。