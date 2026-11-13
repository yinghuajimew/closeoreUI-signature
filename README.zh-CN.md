<div align="center">

<h1 align="center">closeoreUI-signature</h1>

<p>从 <code>libminecraftpe.so</code> 推导并校验 Minecraft 基岩版 <b>OreUI 配置函数特征码</b><br>
纯字节静态分析，不需要 IDA、不需要反编译、不需要 root，单个 <code>.so</code> 几十秒出结果。</p>

<p>
<a href="./README.md">English</a> · <b>中文</b>
</p>

</div>

<br/>

> [!NOTE]
> 本仓库**只包含工具与工作流**，不包含任何 Minecraft 二进制文件或其衍生物。
> `.so` 需要你自己准备。

---

## 它解决什么问题

`ForceCloseOreUI` 这类模块需要一条「特征码」来定位游戏里的 OreUI 配置函数：

```
? ? ? D1 ? ? ? A9 ? ? ? A9 ? ? ? A9 ? ? ? A9 ? ? ? A9 ? ? ? A9 ? ? ? 91 ? ? ? B9 ? ? ? D5 F9 03 03 2A
```

游戏每更新一个版本，函数序言里的寄存器分配就会变，旧特征码失效。
本工具从新版 `libminecraftpe.so` **自动推导**出新的那一条，不必等别人试出来。

---

## 快速开始

### 1. 在 GitHub Actions 上跑（推荐，无需本地环境）

1. Fork 本仓库
2. 把你的 `libminecraftpe.so` 上传到**自己 fork 的 Releases**（新建一个 tag，比如 `1.26.62.1`）
   - 注意：`.so` 有 240~340 MB，超过 git 单文件 100 MB 限制，**只能走 Release 资产**
3. 进入 Actions → `derive-signature` → Run workflow，填：
   - `target`: `libminecraftpe.so`（资产名）
   - `from_release`: `1.26.62.1`（你的 tag）
   - `derive`: ✅
4. 几十秒后，日志末尾直接打印新特征码，同时可下载 artifact

### 2. 本地跑

需要 **Python 3.10+**（用到 `X | None` 注解）。**没有任何第三方依赖**，只用标准库。

```bash
python3 tools/derive_sig.py libminecraftpe.so
python3 tools/derive_sig.py libminecraftpe.so --json out.json
```

### 3. 校验现有特征码能命中哪些版本

```bash
python3 tools/scan.py libminecraftpe.so          # 直接传 .so
python3 tools/scan.py "Minecraft PE x.xx.apk"    # 也可以直接传 APK
python3 tools/scan.py --json report.json *.so
```

---

## 算法

### 第 1 步：定位函数 —— 靠字符串，不靠特征码

在 `.rodata` 里找 OreUI 路由串：

```
/oreui-marketplace/search   /oreui-marketplace/wishlist
/start-from-template        /create-new-world
/player-permissions         /send-invites
/gameplay/bedtime           /pre-start
```

再在 `.text` 里找引用它们的 `ADRP+ADD` 指令对，解出计算地址：

```
ADRP Xn, page        ; 4KB 对齐的页地址
ADD  Xn, Xn, #off    ; 补齐低 12 位 → (PC & ~0xFFF) + page + off
```

### 第 2 步：回溯函数头

从引用处向前扫，找最近的、真正像函数起点的位置。判据：

- 指令是 `sub sp,sp` 或 `stp`
- **前一条指令是 `ret` / `br` / 无条件 `b`**
- 第一条指令不能是 `ADRP`

> 第三条很重要：`ADRP` 与 `sub sp,sp` 在掩码下会混淆，早期版本漏了这个判据导致选错函数。

### 第 3 步：取序言并按类别通配

AArch64 是小端，**opcode 落在最高字节**，所以约定「低 3 字节抹成 `?`，最高字节保留」：

| 类别 | 保留 | 说明 |
|---|---|---|
| `sub sp,sp,#imm` | `D1` | 分配栈帧 |
| `stp` / `ldp` | `A9` | 保存被调用者保存寄存器 |
| `add xN,xM,#imm` | `91` | 建立帧指针 |
| `str` / `ldr` | `B9` / `F9` | 存参数/加载 |
| `mrs xN,<sysreg>` | `D5` | 栈保护金丝雀 |
| `mov xN,xM` | `2A` / `AA` | 参数搬运 |

**版本落款**：`mov` 且源寄存器是 `x0`~`x7`（参数）时**整条保留固定**。
不同版本把哪个参数搬进哪个 callee-saved 寄存器是不同的，这就是区分版本的依据。

### 第 4 步：候选函数评分

引用同一路由串的函数可能有几十个（每个界面一个注册函数）。按序言结构打分：

```
+10  第一条是 sub sp,sp
+ 4  ×  stp 的条数         （保存的寄存器越多 = 函数越大）
+ 3  含 add_imm
+ 3  含 mrs                （栈保护金丝雀）
+ 6  含 mov_reg            （参数搬运 = 落款所在）
```

**不要按引用密度排序** —— 实测某些版本上引用最多的反而是相邻的辅助小函数。

---

## 验证

对 1.26.62.1 推导，得到：

```
? ? ? D1 ? ? ? A9 ? ? ? A9 ? ? ? A9 ? ? ? A9 ? ? ? A9 ? ? ? A9 ? ? ? 91 ? ? ? B9 ? ? ? D5 F9 03 03 2A
```

- 与手工维护的 `1.26.60` 那条**逐字节一致**
- 命中 `0x9AF937C`，命中处正好是函数入口（前一条为 `ret`）
- 该函数体引用 `/oreui-marketplace/search`、`/send-invites`、`/main-menu` 等路由
- 在 GitHub Actions 上跑出的结果与本地**逐字节一致**

---

## 重要限制

### 1. OreUI 是逐步扩张的，不是开关

| 版本 | 状态 |
|---|---|
| 1.20.10.01 | HBUI/OreUI **运行时核心**已存在，界面仍以 JSON-UI 为主 |
| 1.21.x | OreUI 界面逐个出现 |
| 1.26.x | 几乎所有界面都是 OreUI |

1.20.10.01 上特征码命中 `0x56DBB0C`（是函数入口），但该函数引用的是
`/hbui/gameplay.html`、`core:routing:not-found`、`core:gui:resize-hack`、
`core:telemetry:firstContentfulPaint`、`JsException` 这类运行时核心，
**不是具体界面** —— 与 1.26 的钩子目标不是同一个函数，不能直接复用。

### 2. 路由串每个版本都在增减、反复

有些界面在某个版本消失、下个版本又回来。所以固定特征码表必然随版本腐化，
**必须靠本工具对新版本重新推导**。

### 3. 命中不等于可用

| 版本 | 命中地址 | 前一条指令 | 是否函数入口 |
|---|---|---|---|
| 1.20.10.01 | 0x56DBB0C | `ret` | ✅ |
| 1.21.0.03 | 0x5F85100 | `ret` | ✅ |
| 1.21.90.3 | 0x5EC9928 | `ret` | ✅ |
| 1.21.132.1 | 0x907E4BC | `bl` | ❌ 真实函数头在 0x907E3D4 |
| 1.26.0.2 | 0x913DF54 | `bl` | ❌ |
| 1.26.62.1 | 0x9AF937C | `ret` | ✅ |

在函数中间下钩子会覆盖被调用指令或破坏栈帧。**推导出的特征码务必用
`scan.py` 回验一遍「命中处是否落在函数入口」**。

---

## 目录

```
tools/derive_sig.py    推导新特征码
tools/scan.py          校验特征码命中情况（支持直接传 APK）
tools/README.md        算法细节
.github/workflows/derive-signature.yml   手动触发的轻量工作流
```

---

## 许可证

**GNU Affero General Public License v3.0**（AGPL-3.0）

选它的关键原因：AGPL 第 13 条要求**网络服务的提供者也要开源其修改**。
如果有人把这套推导逻辑接进在线服务（比如一个上传 `.so` 就返回特征码的网站）
而不公开改动，AGPL 要求他把源码一并提供。

完整条款见 [LICENSE](./LICENSE)。

## 免责声明

本工具仅做 ELF 静态分析，不修改、不重新分发任何游戏文件。
使用者需自行准备 `.so`，并自行承担相应法律责任。
