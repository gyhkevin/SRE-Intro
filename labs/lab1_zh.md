# 实验室 1 — SRE 哲学：部署、破坏、理解

![难度](https://img.shields.io/badge/难度-初学者-success)
![主题](https://img.shields.io/badge/主题-SRE基础-blue)
![积分](https://img.shields.io/badge/积分-10%2B2-orange)
![技术](https://img.shields.io/badge/技术-Docker%20Compose-informational)

> **目标:** 部署 QuickTicket 系统，有系统地破坏它，并映射故障如何在服务间传播。
> **交付成果:** 从 `feature/lab1` 发出 PR 到课程仓库，其中包含 `submissions/lab1.md`。通过 Moodle 提交 PR 链接。

---

## 概述

在本实验中，您将练习：
- 使用 Docker Compose 部署多服务系统
- 阅读和理解应用架构
- **有系统地破坏事物** 以发现故障模式
- 记录依赖关系和 blast radius（影响范围）

> **您不需要编写应用程序。** QuickTicket 已在 `app/` 目录中提供。您需要部署它、使用它，并对其进行破坏。这是 SRE（站点可靠性工程）第一天的思维方式——在尝试让系统可靠之前，先理解系统如何失败。

---

## 任务 1 — 部署并破坏 QuickTicket (6 积分)

**目标:** 部署 3 服务系统，验证其正常工作，然后系统地杀死每个组件并记录会发生什么。

### 1.1: 部署 QuickTicket

```bash
cd app/
docker compose up --build -d
```

等待所有服务变为健康状态：

```bash
docker compose ps
```

应应看到 5 个容器正在运行：`gateway`、`events`、`payments`、`postgres`、`redis`。

### 1.2: 验证系统正常工作

测试关键路径——列出事件、预订票务、支付：

```bash
# 列出事件
curl -s http://localhost:3080/events | python3 -m json.tool

# 预订 1 张票给事件 1
curl -s -X POST http://localhost:3080/events/1/reserve \
  -H "Content-Type: application/json" \
  -d '{"quantity": 1}' | python3 -m json.tool

# 为预订付费（使用之前响应中的 reservation_id）
curl -s -X POST http://localhost:3080/reserve/RESERVATION_ID_HERE/pay | python3 -m json.tool

# 检查健康状态
curl -s http://localhost:3080/health | python3 -m json.tool
```

### 1.3: 阅读架构

打开并阅读这三个文件（它们都很短——约 300 行总计）：
- `app/gateway/main.py` — API 路由器
- `app/events/main.py` — 票务管理
- `app/payments/main.py` — 支付处理

绘制依赖图：哪个服务调用哪个？如果一个依赖项不可用会发生什么？

### 1.4: 系统化故障探索

一个接一个地杀死每个组件。对于每个组件，请记录会发生什么：

```bash
# 停止 payments 服务
docker compose stop payments
# 测试：我是否仍然可以列出事件？我是否可以预订？我是否可以支付？
curl -s http://localhost:3080/events | python3 -m json.tool
curl -s -X POST http://localhost:3080/events/1/reserve \
  -H "Content-Type: application/json" -d '{"quantity": 1}'
curl -s http://localhost:3080/health | python3 -m json.tool

# 将其恢复
docker compose start payments
```

对以下每个组件重复此操作：`events`、`redis`、`postgres`。对于每个组件：
1. 哪些端点仍然正常工作？
2. 哪些端点会失败？
3. 用户看到什么错误？
4. 健康端点是否反映了问题？

### 1.5: 运行负载生成器

启动负载生成器并观察行为：

```bash
chmod +x app/loadgen/run.sh
./app/loadgen/run.sh 5 30
```

每秒发送 5 个请求，持续 30 秒。记录成功/失败计数。

现在，在另一个终端中，在负载运行期间停止 `payments`：

```bash
docker compose stop payments
```

观察负载生成器输出中的错误率如何变化。

### 1.6: 工作证据

**粘贴到 `submissions/lab1.md`:**

1. `docker compose ps` 的输出，显示所有 5 个服务都在运行
2. 完整的关键路径输出（列出 → 预订 → 支付），带有真实数据
3. `curl -s http://localhost:3080/health` 的输出，当一切正常时
4. 依赖映射（Mermaid 图或简单文本）：
   ```
   gateway → events → postgres
   gateway → events → redis
   gateway → payments
   ```
5. 故障表：

   ```markdown
   | 组件 | 列出事件 | 预订 | 支付 | 健康检查 | 用户影响 |
   |--------|----------|------|------|----------|----------|
   | payments |          |      |      |          |          |
   | events   |          |      |      |          |          |
   | redis    |          |      |      |          |          |
   | postgres |          |      |      |          |          |
   ```

6. 展示 payments 故障时错误率飙升的负载生成器输出

<details>
<summary>💡 提示</summary>

- `docker compose ps` 显示容器状态
- `docker compose logs payments --tail=20` 显示服务的最近日志
- `docker compose stop <service>` 在不删除的情况下停止一个服务
- `docker compose start <service>` 重新启动它
- 健康端点显示依赖状态——检查每个杀死之后的它
- 如果 `reserve` 在杀掉某些东西后仍然可行，这很有意思——请记录原因

</details>

---

## 任务 2 — 优雅降级 (3 积分)

> ⏭️ 此任务是可选的。跳过它不会影响后续实验。

**目标:** 当 `payments` 宕机时，网关应优雅地处理（而不是返回 502）。

### 1.7: 实现优雅降级

当 `payments` -down 时，预订仍应可用——用户只是暂时无法支付。修改 `app/gateway/main.py`：

- `/events` 和 `/events/{id}` 端点始终应工作（它们不需要 payments）
- `/events/{id}/reserve` 端点始终应工作（它只与 events 通信）
- `/reserve/{id}/pay` 端点应返回一个清晰的消息，如 `{"error": "payments_unavailable", "message": "Payment service is temporarily down. Your reservation is held — try again in a few minutes.", "reservation_id": "..."}` 配合 503 状态码，而不是通用的 502

### 1.8: 验证

```bash
docker compose stop payments
# 预订仍应可用：
curl -s -X POST http://localhost:3080/events/1/reserve \
  -H "Content-Type: application/json" -d '{"quantity": 1}'
# 支付应返回清晰的 503 和可操作的消息：
curl -s -X POST http://localhost:3080/reserve/RESERVATION_ID/pay
docker compose start payments
```

**粘贴到 `submissions/lab1.md`:**
- 您的 gateway 更改差异 (`git diff app/gateway/main.py`)
- 当 payments -down 时，预订（可用）和 pay（清晰的 503）的输出

<details>
<summary>💡 提示</summary>

- 专门捕获 `httpx.ConnectError` 用于 payments 调用
- 返回 `JSONResponse(status_code=503, content={...})` 而不是抛出 `HTTPException(502)`
- 预订仍然驻留在 Redis 中——用户可以在服务恢复时重试支付

</details>

---

## 任务 3 — GitHub 社区互动 (1 积分)

**目标:** 探索支持协作和发现的 GitHub 社交功能。

**必行动:**
1. **星标** 课程仓库
2. **星标** [simple-container-com/api](https://github.com/simple-container-com/api) 项目——一个用于容器管理的有前途的开源工具
3. **关注** 您的教授和教学助理 on GitHub：
   - 教授: [@Cre-eD](https://github.com/Cre-eD)
   - TA: [@Naghme98](https://github.com/Naghme98)
   - TA: [@pierrepicaud](https://github.com/pierrepicaud)
4. **关注** 至少 3 位来自课程的同学

**添加到 `submissions/lab1.md`:**

一个 "GitHub 社区" 章节，包含 1-2 句解释：
- 为什么在开源中星标仓库很重要
- 如何通过关注开发人员帮助团队项目和职业发展

<details>
<summary>💡 GitHub 社交功能</summary>

**为什么星标很重要:**
- 星标帮助您以后参考感兴趣的项目
- 星标计数表明项目的受欢迎程度和社区信任度
- 星标的仓库出现在您的 GitHub 配置文件中，展示您的兴趣
- 星标鼓励维护者，并有助于项目获得关注

**为什么关注很重要:**
- 查看其他开发人员正在从事的工作
- 通过他们的活动发现新项目
- 建立超出课堂的专业联系
- 关注同学的工作以便未来合作

</details>

---

##  bonus 任务 — 在负载下的资源使用 (2 积分)

> 🌟 想要额外挑战并获得体验的同学。

**目标:** 测量 QuickTicket 在休眠 vs 在负载下时的资源消耗，并确定哪个服务最昂贵。

### B.1: 基线 (空闲)

所有服务运行但无流量时：

```bash
docker stats --no-stream --format "table {{.Name}}\t{{.CPUPerc}}\t{{.MemUsage}}\t{{.NetIO}}\t{{.PIDs}}"
```

### B.2: 在负载下

在一个终端中启动负载生成器：

```bash
./app/loadgen/run.sh 10 30
```

在另一个终端中， while 它正在运行时捕获统计信息：

```bash
docker stats --no-stream --format "table {{.Name}}\t{{.CPUPerc}}\t{{.MemUsage}}\t{{.NetIO}}\t{{.PIDs}}"
```

### B.3: 带故障注入的压力

带故障注入重启 payments：

```bash
docker compose stop payments
PAYMENT_FAILURE_RATE=0.3 PAYMENT_LATENCY_MS=500 docker compose up -d payments
```

再次运行负载生成器并捕获统计信息。然后恢复正常 payments：

```bash
docker compose stop payments
PAYMENT_FAILURE_RATE=0.0 PAYMENT_LATENCY_MS=0 docker compose up -d payments
```

**向您的报告添加:**
- 所有三种情况（空闲、负载、混沌）的统计表
- 哪个服务使用最多内存？在负载下会变化吗？
- 哪个服务在负载下使用最多 CPU？为什么？
- 故障注入如何影响 payments 中 gateway 的资源使用？(提示：慢 payments → gateway 更长时间地保持连接)

---

## 接受标准

### 任务 1 (6 积分)
- ✅ `docker compose ps` 输出，显示所有 5 个服务正在运行
- ✅ 完整的关键路径输出（列出 → 预订 → 支持），带有真实数据
- ✅ 显示所有服务关系的依赖映射
- ✅ 包含所有 4 个组件（payments、events、redis、postgres）的完整故障表
- ✅ 展示故障期间错误率飙升的负载生成器输出

### 任务 2 (3 积分)
- ✅ 当 payments -down 时，网关返回清晰的 503 和可操作的消息
- ✅ 当 payments -down 时，预订仍然可用
- ✅ 展示代码更改的干净 diff

### 任务 3 (1 积分)
- ✅ 星标课程仓库和 simple-container-com/api
- ✅ 关注教授、TAs 和 3+ 位同学
- ✅ 提交包含 GitHub 社区章节的 submission

### Bonus 任务 (2 积分)
- ✅ 3 种情况（空闲、负载、混沌）的统计表
- ✅ 内存/CPU 使用模式分析，说明哪个服务最多以及为什么
- ✅ 故障注入对 gateway 资源使用的影响观察

---

## 如何提交

1. 创建分支并推送：

   ```bash
   git switch -c feature/lab1
   git add <工作证明 + lab1.md>
   git commit -m "docs(lab1): add submission1 — deploy and failure exploration"
   git push -u origin feature/lab1
   ```

2. 从您的分支 `feature/lab1` → **课程仓库 main 分支**打开 PR。

3. 在 PR 描述中包含：

   ```text
   - [x] Task 1 done — deployed QuickTicket, failure exploration complete
   - [ ] Task 2 done — graceful degradation in gateway
   - [x] Task 3 done — GitHub community engagement
   - [ ] Bonus Task done — resource usage under load
   ```

4. **在 Moodle 前提交 PR URL**。

---

## 评分标准

| 任务 | 积分 | 标准 |
|------|------:|------|
| **任务 1** — 部署 & 故障探索 | **6** | 所有 5 个服务运行，完整的故障表（测试所有 4 个组件），负载生成器输出，依赖映射 |
| **任务 2** — 优雅降级 | **3** | gateway 当 payments -down 时返回清晰的 503，预订仍可用，干净的代码 diff |
| **任务 3** — GitHub 社区互动 | **1** | 星标、关注，以及书面解释 |
| **Bonus 任务** — 在负载下的资源使用 | **2** | 3 种情况（空闲、负载、混沌）的统计表，内存/CPU 模式分析，故障注入影响 |
| **总计** | **12** | 10 项主任务 + 2 项 bonus |

---

## 资源

<details>
<summary>📚 文档</summary>

- [Docker Compose 文档](https://docs.docker.com/compose/) — compose 命令的参考
- [Google SRE 书，第一章](https://sre.google/sre-book/introduction/) — 什么是 SRE
- [Google SRE 书，第三章](https://sre.google/sre-book/embracing-risk/) — 拥抱风险

</details>

<details>
<summary>🛠️ 工具</summary>

- [curl 手册](https://curl.se/docs/manual.html) — 从终端发出 HTTP 请求
- [jq](https://jqlang.github.io/jq/) — JSON 处理器（`python3 -m json.tool` 的替代方案）
- [Docker Compose CLI 参考](https://docs.docker.com/compose/reference/) — stop、start、logs、ps

</details>

<details>
<summary>⚠️ 常见陷阱</summary>

- **端口冲突:** 如果端口 3080 已繁忙，请在 `docker-compose.yaml` 中更改端口（例如，`4080:8080`）
- **Postgres 未就准备:** `events` 服务等待 postgres 健康检查——如果它失败，请检查 `docker compose logs postgres`
- **Redis 连接错误:** events 服务即使 Redis-down 也会启动并记录警告——请检查 `docker compose logs events`
- **负载生成器需要 `bc`:** 通过 `apt install bc` 或 `brew install bc` 安装（如果缺失）

</details>