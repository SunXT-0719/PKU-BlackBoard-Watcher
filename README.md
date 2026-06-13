# PKU-BlackBoard-Watcher

某些课莫名其妙会蹦出作业挺让人心烦的，所以我做了本项目。

在本地/云端（WSL/Linux）运行的教学网（Blackboard Learn）更新监控脚本：抓取课程内容 → SQLite 去重 → 多渠道推送（Bark / Server酱微信通知）。

> 📝 更详细的计划、里程碑与设计说明见 [PLAN.md](PLAN.md)

远期规划：进一步优化通知体系，加上客户端，可视化 ddl 日历等

---

## 📋 目录

- [功能特性](#功能特性)
- [快速开始](#快速开始)
- [配置说明](#配置说明)
- [使用指南](#使用指南)
- [定时运行](#定时运行)
- [贡献与反馈](#贡献与反馈)

---

## 功能特性

### 监控范围

当前覆盖 4 个板块：

- **课程通知**（Announcements）
- **教学内容**（Course Content）
- **课程作业**（Assignments）- 含在线提交作业的到期/满分/提交状态
- **个人成绩**（Grades）- 含评分项类别/评分时间/成绩变化

### 核心功能

- ✅ 自动抓取课程更新
- ✅ SQLite 本地去重
- ✅ 多渠道推送通知（Bark iOS / Server酱 微信）
- ✅ 云端无人值守运行（自动刷新登录态）
- ✅ 首次运行防刷屏机制

---

## 快速开始

### 1. 环境准备

**系统要求：**
- Python 3.10+
- Linux / WSL / macOS。推荐部署在 CLAB 上。

**推荐使用 Conda 环境：**

```bash
# 创建并激活虚拟环境（可选）
conda create -n bbwatcher python=3.10
conda activate bbwatcher
```

### 2. 安装依赖

```bash
# 安装 Python 包
python -m pip install -r requirements.txt

# 安装 Playwright 浏览器
python -m playwright install chromium

# 如果提示缺少系统依赖（Ubuntu/Debian）
python -m playwright install-deps chromium  # 可能需要 sudo
```

### 3. 配置文件

1. 复制配置模板：
   ```bash
   cp .env.example .env
   ```

2. 编辑 `.env`，选择推送方式：
   ```env
   # 推送后端（二选一）
   PUSH_BACKEND=serverchan          # 或 bark

   # Bark（iOS 用户）
   BARK_ENDPOINT=<你的Bark Token>

   # Server酱（Android 用户，通过微信接收）
   SERVERCHAN_SENDKEY=<你的SendKey>

   BB_USERNAME=<学号>
   BB_PASSWORD=<密码>
   ```

### 4. 生成登录态

如果未配置 `BB_USERNAME/BB_PASSWORD`，需要手动导出登录态：

```bash
python scripts/export_state.py
```

在弹出的浏览器窗口中完成登录，然后回到终端按 Enter 保存。

### 5. 测试运行

**预览模式（不推送）：**

```bash
python -m app.main --run --dry-run --dry-run-out data/push_preview.json --course-limit 1 --limit 10
```

**正式运行（会推送）：**

```bash
python -m app.main --run --limit 100
```

> ⚠️ **首次运行说明**：如果数据库中没有已通知记录，程序会发送 1 条"初始化完成"消息，并将当前所有条目标记为已通知，避免刷屏。后续运行只推送新增/变更内容。

---

## 配置说明

### 默认配置

一般情况下无需修改。

| 配置项 | 说明 | 示例 |
|--------|------|------|
| `BB_BASE_URL` | 教学网基础 URL | `https://course.pku.edu.cn` |
| `BB_LOGIN_URL` | 登录入口 URL | 未登录时跳转的统一认证页 |
| `BB_COURSES_URL` | 课程列表页 URL | 登录后可访问的 portal 页 |
| `COURSE_TERM_FILTER` | 课程学期过滤模式 | `current`（默认）或 `off` |

### 课程范围过滤

- `COURSE_TERM_FILTER=current`（默认）：仅抓取课程名里“`xx-xx学年第N学期`”与当前时间匹配的课程
- `COURSE_TERM_FILTER=off`：关闭学期过滤（会抓到历史课程）

### 推送配置

| 配置项 | 说明 | 可选值 |
|--------|------|--------|
| `PUSH_BACKEND` | 推送后端选择 | `bark`（iOS）或 `serverchan`（Android 微信） |

**Bark 方式（iOS/iPadOS）：**

| 配置项 | 说明 | 格式 |
|--------|------|------|
| `BARK_ENDPOINT` | Bark 推送地址 | 完整 URL：`https://api.day.app/<token>`<br>或仅 Token：`<token>` |

**Server酱 方式（Android，通过微信接收）：**

| 配置项 | 说明 | 格式 |
|--------|------|------|
| `SERVERCHAN_SENDKEY` | Server酱 SendKey | 从 https://sct.ftqq.com/ 获取 |

> 📱 **注意**：
> - Bark 仅支持 iOS/iPadOS；Server酱 通过微信推送，支持 Android 和 iOS
> - ⚠️ **Server酱 免费版每日最多推送 5 条**。正常使用（非首次初始化）每天教学网更新通常不超过 5 条，一般够用。如果不够可升级付费版或提 PR 增加其他渠道

### 自动登录配置

如果不填写的话可能需要时不时手动更新登录态了。

| 配置项 | 说明 |
|--------|------|
| `BB_USERNAME` | 学号/用户名 |
| `BB_PASSWORD` | 密码 |

- 配置后，程序会在登录态失效时自动刷新
- 当前仅支持 IAAA 登录（非 IAAA 登录需求欢迎提 PR）

---

## 使用指南

### 常用命令

**预览（不推送）：**
```bash
python -m app.main --run --dry-run --dry-run-out data/push_preview.json
```

**正式运行（推送）：**
```bash
python -m app.main --run --limit 100
```

### 调试命令

**导出本轮抓取数据：**
```bash
python -m app.main --fetch-all --items-json data/items.json
```

**校验登录态：**
```bash
python -m app.main --check-login
```

### 产物文件

| 文件路径 | 说明 |
|----------|------|
| `logs/run.log` | 运行日志 |
| `data/storage_state.json` | 登录态文件 |
| `data/state.db` | SQLite 数据库（去重/通知状态） |
| `data/push_preview.json` | Dry-run 预览输出 |

---

## 定时运行

### 使用 check.sh 脚本

```bash
bash check.sh
```

**特性：**
- 完整闭环运行：`--run --limit 100`
- 默认使用 `conda run -n bbwatcher`（可用 `CONDA_ENV_NAME` 环境变量覆盖）
- 使用文件锁 `data/cron.lock` 避免并发

### Crontab 配置

每 20 分钟运行一次：

```cron
*/20 * * * * /bin/bash /绝对路径/PKU-BlackBoard-Watcher/check.sh
```

---

## 贡献与反馈

欢迎提交 Issue 和 Pull Request！

**特别欢迎的贡献方向：**
- 📧 邮件通知支持
- 🔐 非 IAAA 登录支持
- 🎨 更多推送渠道（PushDeer、Telegram 等）
- 🐛 Bug 修复和功能改进
