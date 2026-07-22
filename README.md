# OutlookGetToken

给**已注册**但还没有 Graph `refresh_token` 的 Outlook / Hotmail 账号，批量走一遍 OAuth2 授权，把 token 写入结果文件。

核心脚本：`get_refresh_token.py`（patchright 浏览器自动化 + 授权码换 token）。

---

## 1. 如何使用

### 1.1 环境

```bash
git clone <本仓库地址>
cd OutlookGetToken   # 若仓库根目录即本工具，可省略
pip install patchright requests
patchright install chromium
```

代理软件需已启动（默认示例为本机 `127.0.0.1:7890`）。

### 1.2 准备配置与账号（本地文件，不会进 Git）

仓库**不提交**你的账号与本地配置。首次使用：

```bash
# Windows
copy config.example.json config.json
copy not_oauth2.example.txt not_oauth2.txt

# Linux / macOS
cp config.example.json config.json
cp not_oauth2.example.txt not_oauth2.txt
```

编辑 `not_oauth2.txt`，**每行一个账号**：

```text
邮箱----密码
```

示例：

```text
user1@outlook.com----YourPass1
user2@hotmail.com----YourPass2
```

- 以 `#` 开头的行视为注释，忽略  
- 也兼容更长行（如已有 `邮箱----密码----client_id----token`）：**只取前两段** 邮箱与密码  

按需编辑 `config.json`（代理、并发等，见下一节）。

### 1.3 运行

```bash
python get_refresh_token.py
```

### 1.4 输出与日志

| 项 | 说明 |
|----|------|
| 成功 | **追加**写入 `oauth2.txt`（可用 `output_file` 改）：`邮箱----密码----client_id----refresh_token` |
| 进度 | `[进度] 完成数/总数 成功N 失败M` |
| 单号结果 | `[结果] OK 邮箱` 或 `[结果] FAIL 邮箱 \| 原因=...` |
| 失败 | 过程中有 `[FAIL] 原因=...`；结束时打印失败明细与原因统计 |

`client_id` 固定为：`9e5f94bc-e8a4-4e73-b8be-63364c29d753`。

### 1.5 使用注意

- 工具会**弹真实浏览器窗口**（`headless=False`），便于观察卡在哪一页  
- **不会**自动绑定/接码辅助邮箱；密码后的「保护帐户」页会点 **暂时跳过**  
- 若已绑过辅助邮箱且弹出「验证你的电子邮件」，会点 **使用密码** 继续  
- 出现 **帐户已锁定** → 该号直接失败，并在日志里写明原因  
- 同一账号多次成功会**重复追加**到输出文件，请自行去重  

---

## 2. `config.json` 配置

当前仓库示例：

```json
{
    "input_file": "not_oauth2.txt",
    "output_file": "oauth2.txt",
    "concurrent": 3,
    "proxy": {
        "mode": "single",
        "type": "http",
        "host": "127.0.0.1",
        "single_port": 7890,
        "port_start": 24000,
        "port_end": 24005,
        "max_per_proxy": 20
    }
}
```

### 2.1 顶层字段

| 字段 | 类型 | 说明 |
|------|------|------|
| `input_file` | string | 相对本目录的输入文件名，默认 `not_oauth2.txt` |
| `output_file` | string | 相对本目录的输出文件名，默认 `oauth2.txt`，**追加**写入 |
| `concurrent` | number | 并发浏览器数；实际为 `min(concurrent, 账号数)` |
| `proxy` | object | 代理配置，见下表 |

### 2.2 `proxy`

| 字段 | 说明 |
|------|------|
| `mode` | `single`：只用 `single_port`；`multiple`：使用 `port_start`～`port_end` 端口池 |
| `type` | 代理协议，如 `http`、`socks5` |
| `host` | 代理主机，常见 `127.0.0.1` |
| `single_port` | `mode=single` 时的端口 |
| `port_start` / `port_end` | `mode=multiple` 时的端口闭区间 |
| `max_per_proxy` | 单个端口在进程内最多被选中次数；用满后该端口暂不选，全满则重置计数再选 |

浏览器与换 token 的 `requests` **尽量使用同一代理 URL**，减少出口不一致。

---

## 3. 代码流程详解

入口：`python get_refresh_token.py` → `main()`。

### 3.1 总览

```text
load_config()
  → load_accounts(input_file)
  → ThreadPoolExecutor(concurrent)
       每个账号: process_single_account(email, password, proxy, idx, total)
  → 成功: 追加 oauth2 行
  → 失败: 记录原因；结束时打印失败明细 + 原因统计
```

### 3.2 单账号：`process_single_account`

```text
启动 Chromium（带代理）
  → 打开 Graph OAuth 授权 URL（sso_reload=true，冷会话）
  → 状态循环（总窗口约 120s，200ms 轮询）
       ├─ 一旦 URL / 请求 / 导航里出现 localhost...?code=  → 立刻结束登录阶段
       ├─ 「帐户已锁定」                                 → FAIL 原因=帐户已锁定
       ├─ 「验证你的电子邮件」                            → 点「使用密码」
       ├─ 密码框 #passwordEntry / 可见 passwd             → 填密码 + primaryButton「下一步」
       ├─ 邮箱框 #i0116 等                                → 填登录邮箱
       ├─ 「让我们来保护你的帐户」                         → #iShowSkip 暂时跳过
       ├─ 「保持登录状态？」                               → secondaryButton「否」
       └─ 授权同意页                                      → 接受
  → POST /token 用 code 换 refresh_token（可走同一代理）
  → 关浏览器，返回 (email, ok, token|原因)
```

监听 `request` + `framenavigated` 抓 code；循环内也会读 `page.url` / `location.href`。  
**拿到 code 后立即换 token 并退出**，不再干等同意按钮超时或长时间 sleep。

### 3.3 密码之后的分支（重要）

顺序与线上一致：**保护帐户 / 锁定页都在输入密码之后**。

| 页面 | 识别要点 | 行为 |
|------|----------|------|
| 验证你的电子邮件 | 文案 /「使用密码」`span[role=button]` | 点「使用密码」，不接辅助邮箱验证码 |
| 输入密码 | `#passwordEntry`；**跳过**隐藏预填桩 `#i0118.moveOffScreen` | fill + `data-testid=primaryButton` 下一步 |
| 让我们来保护你的帐户 | `#iShowSkip`、`#EmailAddress`、相关文案 | **暂时跳过**（6/7 天后必须输入） |
| 帐户已锁定 | `h1[data-testid="title"]` 等 | **直接失败**，不点下一步做人机 |
| 保持登录 | `secondaryButton` + 文案 | 点「否」 |
| 同意 / 已授权 | 同意按钮或已跳转 | 点接受，或直接已有 code |

### 3.4 失败原因日志

所有失败走统一格式：

```text
[1/3] 12:00:01 | +15s [FAIL] 原因=帐户已锁定 | 页面标题=帐户已锁定
[结果] FAIL user@outlook.com | 原因=帐户已锁定
```

| 原因示例 | 含义 |
|----------|------|
| `帐户已锁定` | 密码后出现锁定页 |
| `密码错误` | 页面提示密码不正确 |
| `帐户不存在` | 找不到该用户 |
| `卡在保护帐户页(未能跳过)` | 有保护页但未点到跳过 |
| `卡在验证电子邮件页` | 未成功改走密码 |
| `登录超时未拿到code` | 约 120s 内无回调 code（附标题/URL） |
| `token交换失败` / `token网络请求失败` | code 有了但换 token 失败 |
| `浏览器启动失败` | Chromium / 代理启动问题 |
| `运行异常` | 其它未归类异常（已压缩 Call log） |

结束时若有失败：

```text
--- 失败明细 ---
  a@outlook.com | 原因=帐户已锁定
--- 失败原因统计 ---
  1x 帐户已锁定
```

### 3.5 主要函数一览

| 函数 | 作用 |
|------|------|
| `load_config` / `load_accounts` | 读配置与账号列表 |
| `ProxyPicker` | 单端口 / 端口池 + `max_per_proxy` |
| `_fill_email` | 登录邮箱（`#i0116` 或 Fluent） |
| `_click_use_password_if_present` | 验证邮箱页改走密码 |
| `_find_password_input` / `_fill_password_and_submit` | 可见密码框 + 下一步 |
| `_click_protect_skip_if_present` | `#iShowSkip` 暂时跳过 |
| `_page_is_account_locked` | 帐户已锁定 → 失败 |
| `_click_kmsi_no_if_present` | 保持登录「否」 |
| `_click_consent_if_present` | 授权同意 |
| `_try_capture_code` / `_extract_code_from_url` | 抓 OAuth code |
| `_fail` / `_diagnose_fail_reason` | 失败原因日志与推断 |
| `_compact_exc` | 去掉 Playwright 多行 Call log |

### 3.6 与 OutlookRegister 的差异（简要）

| | OutlookRegister | OutlookGetToken |
|--|-----------------|-----------------|
| 目标 | 注册 + 尽量拿 token | **只**给已有账号补 token |
| 输入 | 配置任务数 / 注册流程 | `邮箱----密码` 列表 |
| 辅助邮箱 | 可绑定 + jwt 接码验证 | **不接码**；保护页跳过；验证页用密码 |
| Cookie | 同浏览器优先 / NEW 注入 | 纯冷开浏览器 |
| 验证码按压 | 有 | 无（不注册） |

---

## 4. 文件结构

```text
OutlookGetToken/
  get_refresh_token.py      # 主程序（可提交）
  config.example.json       # 配置模板（可提交）
  not_oauth2.example.txt    # 账号格式示例（可提交）
  .gitignore                # 忽略账号/token/本地 config
  README.md                 # 本文档
  config.json               # 本地配置（gitignore，自行从 example 复制）
  not_oauth2.txt            # 本地输入（gitignore）
  oauth2.txt                # 本地输出（gitignore）
```

## 5. 隐私与开源说明

公开仓库时请只推送**代码与示例**，不要推送：

| 文件 | 原因 |
|------|------|
| `not_oauth2.txt` | 含真实邮箱与密码 |
| `oauth2.txt` | 含 refresh_token |
| `config.json` | 可能含本机代理/路径习惯（可用 example 代替） |
| `__pycache__/`、`*.log` | 运行时垃圾 |

以上已由 `.gitignore` 默认忽略。若某文件**曾经**被 `git add` 过，忽略规则不会自动取消跟踪，需在独立仓库初始化时**不要** `git add` 这些文件，或执行：

```bash
git rm --cached not_oauth2.txt oauth2.txt config.json
```
