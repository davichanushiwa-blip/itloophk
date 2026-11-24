# IT Loop CMS — IT Loop Hong Kong 内容管理平台

SpringBoot 3 + Vue 3 自建 CMS。**v1.5 起内容生产流程调整为：选题雷达（只抓标题/摘要/链接）→ AI 生成初稿 → 人工审校 → 发布**，不再转载全文。

## 为什么不再转载全文

| 维度 | 全文转载（旧方案） | 选题雷达 + AI 初稿（现方案） |
|---|---|---|
| 版权 | 香港《版权条例》下需授权，风险高 | 标题 + 短摘要 + 链接属合理引用，风险低 |
| SEO | 原文权重高于新站，被判重复内容，几乎无排名 | 内容有本地视角，可正常参与排名 |
| 内容差异 | 全网同款新闻，无竞争力 | 香港 IT 实务角度，形成差异 |
| 维护成本 | RSS 改版、反爬、失效源排查（2–5 h/月） | 只抓列表页，不用进详情页，维护极低 |
| 费用 | — | LLM API 约 HK$30–60 / 100 篇 |

## 项目结构

```text
itloop-cms/
├── backend/            # SpringBoot 3.2 后端（单体，模块化分包）
│   └── src/main/java/com/itloop/cms/
│       ├── common/         # 统一返回 / JWT / HTML 清洗（防 XSS）
│       ├── config/         # Security、MyBatis-Plus、AI 配置、启动初始化
│       ├── entity/         # article / topic / spider_source 等实体
│       ├── service/spider/ # 选题雷达（RSS + HTML，只抓标题摘要）
│       ├── service/ai/     # AI 写作（OpenAI 兼容，DeepSeek/OpenAI/通义皆可）
│       ├── scheduler/      # 02:30 抓选题 → 03:00 生成初稿 → 每分钟发布扫描
│       └── controller/     # /api/admin/* 管理端 + /api/portal/* 门户端
├── admin-frontend/     # Vue3 + Element Plus + wangEditor 管理后台
└── docker/             # 部署编排 + 建表/迁移 SQL
```

## 内容生产流程

```text
每日 02:30  选题雷达抓取 RSS/列表页  ──→  topic 表（标题 + 摘要 + 原文链接）
每日 03:00  AI 为待处理选题生成初稿  ──→  article 表（status=1 待审核）
人工        「选题中心」可单篇/批量生成、忽略、恢复
人工        在编辑器内核对事实、补本地观点  ──→  审核通过发布
```

关键约束（代码层面强制，不依赖操作纪律）：

- 爬虫**不抓详情页正文**，`excerpt` 上限 300 字；
- AI 稿一律 `status=1 待审核`，不会自动发布；
- 每篇 AI 稿由程序追加「消息来源」段落（`rel="nofollow noopener"`），模型无法省略；
- 单次批量 5 篇、每日上限 20 篇双重限流，防止额度被刷爆。

## 本地开发

```bash
# 1. 启动本地开发版 MySQL（必须用 dev 文件！它把 3306 映射到宿主机）
docker compose -f docker/docker-compose.dev.yml up -d
docker compose -f docker/docker-compose.dev.yml ps      # 等 STATUS 变 healthy

# 2. 后端（端口 8080）
cd backend
mvn spring-boot:run 2>&1 | tee /tmp/itloop-run.log
# 启动失败时：grep -A 12 "Caused by" /tmp/itloop-run.log

# 3. 管理后台（端口 5173，/api 代理到 8080）
cd admin-frontend && pnpm install && pnpm dev
# 打开 http://localhost:5173，默认 admin / admin123
```

> ⚠ `docker-compose.yml` 是**生产部署**用（MySQL 不映射宿主机端口）。
> 本机 `mvn spring-boot:run` 连 `localhost:3306`，必须用 `docker-compose.dev.yml`，否则报 `Connection refused`。

### 从旧版本升级（v1.0–v1.4 → v1.5）

```bash
# 已有数据卷的库，需执行一次迁移（脚本可重复执行，已存在的列会自动跳过）
docker exec -i itloop-mysql-dev mysql -uroot -proot123 itloop_cms < docker/sql/migrate_v1.5.sql
```

旧库里已转载的全文文章不会被删除，可自行在「文章管理」筛选清理。

## 配置 AI 写作

任选一家 OpenAI 兼容服务，改三个环境变量即可，无需改代码：

| 服务 | `AI_BASE_URL` | `AI_MODEL` | 说明 |
|---|---|---|---|
| DeepSeek（推荐，最便宜） | `https://api.deepseek.com` | `deepseek-chat` | 中文佳，注册送额度 |
| OpenAI | `https://api.openai.com/v1` | `gpt-4o-mini` | 稳定，价格中等 |
| 通义千问 | `https://dashscope.aliyuncs.com/compatible-mode/v1` | `qwen-plus` | 国内直连快 |
| Moonshot | `https://api.moonshot.cn/v1` | `moonshot-v1-8k` | 上下文长 |

```bash
# 本地启动时直接带 key
cd backend
AI_API_KEY=sk-xxxxxxxx mvn spring-boot:run
```

未配置 key 时系统仍可运行：生成的是「占位大纲」（明确标注、不含编造内容），用于走通流程，**不产生任何 API 费用**。

其他可调项（`application.yml` → `cms.ai`）：`temperature`、`max-tokens`、`daily-limit`（每日篇数上限）、`auto-generate`（是否 03:00 自动生成）、`min-words` / `max-words`（目标字数）、`audience`（读者画像，直接影响文风）。

## 选题来源说明

预置 4 个源（默认停用，到「選題雷達 → 來源配置」启用）：

| 源 | 类型 | 说明 |
|---|---|---|
| IT PRO Magazine（香港） | RSS | ✅ 已实测抓取成功，作为主力选题源 |
| iThome（台灣） | RSS | `/rss` 摘要足够支撑选题 |
| IT之家（中國大陸） | RSS | description 含 HTML，只取纯文字摘要 |
| 51CTO | HTML | ⚠ 有 WAF 人机验证，暂不启用，需 Playwright 兜底 |

每日 02:30（香港时间）自动抓取所有启用源。单源单次条数由 `max_items_per_run` 控制，改抓取时间请改 `CmsScheduler.dailySpider` 注解。

HTML 源只配列表页规则即可（不再需要详情页规则）：

```json
{"items": "a.post-link", "linkAttr": "href"}
```

## DigitalOcean 生产部署（新加坡 $48/月）

```bash
# 服务器上（Ubuntu 22.04 + Docker）
git clone <你的仓库> && cd itloop-cms/docker

cat > .env <<EOF
MYSQL_ROOT_PASSWORD=改成强密码
DB_PASSWORD=改成强密码
JWT_SECRET=随机32位以上字符串
AI_API_KEY=sk-你的key
AI_BASE_URL=https://api.deepseek.com
AI_MODEL=deepseek-chat
EOF

docker compose up -d --build
# 管理后台 http://<服务器IP>/admin
```

**上线检查清单**

1. 登录后立即改 admin 密码
2. 域名解析到服务器 IP，`SitemapController` 的 `SITE` 常量改为正式域名
3. 加 HTTPS：certbot 申请证书，`nginx.conf` 开放 443
4. MySQL 备份：`docker exec itloop-mysql mysqldump -uitloop -p... itloop_cms | gzip > backup.sql.gz`（建议 crontab 每日一次）
5. `.env` 不要提交到 git；定期在服务商后台轮换 `AI_API_KEY`

## API 速览

| 接口 | 说明 |
|---|---|
| `POST /api/auth/login` | 登录 |
| `GET /api/admin/topics` | 选题分页（`status`: 0 待处理 / 1 已生成 / 2 已忽略） |
| `POST /api/admin/topics/{id}/generate` | 单篇生成 AI 初稿 → 待审核文章 |
| `POST /api/admin/topics/generate-batch?limit=5` | 批量生成（受每日上限约束） |
| `GET /api/admin/topics/status` | 待处理数量 + AI 配置状态 |
| `GET/POST /api/admin/articles` | 文章分页 / `save` 保存（planTime 定时发布） |
| `POST /api/admin/articles/{id}/approve` | 审核通过并发布 |
| `GET/POST /api/admin/spider/sources` | 选题来源 CRUD，`/{id}/run` 手动抓取 |
| `GET /api/admin/spider/logs` | 抓取日志 |
| `GET /api/portal/articles` | 门户列表（公开，仅已发布） |
| `GET /api/portal/articles/{slug}` | 门户详情（按 slug） |
| `GET /sitemap.xml` | 自动 sitemap |

## 后续路线（v2）

1. 门户前台 Nuxt3 SSR（SEO 关键）+ 51CTO Playwright 兜底
2. 封面图自动配图（AI 生图或图库 API）、图片转存对象存储
3. 选题评分（按关键词热度/历史阅读量排序，优先送审高价值选题）
4. Redis 缓存与多实例部署、邮件/TG 失败告警
# itloophk
