# GitHub Profile 自动更新主页 — 部署说明

> 从 [wdfk-prog/wdfk-prog](https://github.com/wdfk-prog/wdfk-prog) 改造而来。
> 全部运行在 GitHub 免费额度内，**不需要租任何服务器**。

---

## 0. 先看结论

| 项目 | 花费 |
|---|---|
| GitHub Actions（public 仓库） | 免费，无限分钟 |
| Profile 仓库本体 | 必须 public，所以免费 |
| GitHub Pages / 徽章渲染 | 免费 |

**唯一的成本是你的时间。**

---

## 1. 目录结构

```
github-profile/
├── README.md                          # 主页内容
├── SETUP.md                           # 本文件
└── .github/
    └── workflows/
        ├── metrics.yml                # 生成 github-metrics/*.svg（徽章、图表）
        ├── snake.yml                  # 贪吃蛇贡献动画
        ├── contrib.yml                # 3D 贡献图
        └── fork_star.yml              # 把 star/fork 数写进仓库简介
```

**所有用户名都已改成 `${{ github.repository_owner }}` 自动获取，不用手填。**
（前提是你按第 2 步建了同名仓库 —— 这也是 Profile README 的硬性要求。）

---

## 2. 建仓库

1. GitHub 右上角 **+ → New repository**
2. **Repository name 必须和你的用户名一模一样**，例如用户名是 `shbao`，仓库名就填 `shbao`
3. 可见性选 **Public**（私有仓库的 README 不会渲染到主页）
4. 勾选 **Add a README file**
5. Create

这一步做完，你的个人主页 `https://github.com/<用户名>` 顶部就会显示这个仓库的 README。

> 想省事：GitHub 会提示你 "You found a secret! `<用户名>/<用户名>` is a special repository." —— 看到这句话就说明建对了。

---

## 3. 上传这套文件

把本目录下的内容推上去。用命令行：

```bash
cd github-profile
git init
git branch -M main
git remote add origin https://github.com/<你的用户名>/<你的用户名>.git
git add .
git commit -m "init profile"
git push -u origin main
```

> 如果远端已有 README（第 2 步勾选生成的），先 `git pull --rebase origin main` 再 push，
> 或者干脆 `git push -f`（仓库刚建，没有历史包袱）。

---

## 4. 配置 Secrets

路径：仓库 → **Settings → Secrets and variables → Actions → New repository secret**

需要 2 个（名字必须完全一致，大小写敏感）：

| Secret 名 | 用途 | 怎么拿 |
|---|---|---|
| `METRICS_TOKEN` | metrics.yml 拉取数据 | 见下方「PAT 怎么建」 |
| `ACCESS_TOKEN` | fork_star.yml 改仓库简介 | 同上，可复用同一个 PAT |

### PAT 怎么建

1. GitHub → 右上角头像 → **Settings**
2. 左侧拉到最底 → **Developer settings**
3. **Personal access tokens → Tokens (classic)**
4. **Generate new token (classic)**
5. Note 随便填，Expiration 建议选 **No expiration**（否则过期后 workflow 会静默失败）
6. **勾选 `repo`**（整个大项全勾）
7. 再勾 **`read:user`**（metrics 需要读用户资料）
8. Generate → **立刻复制**（只显示一次）

> ⚠️ **必须用 classic token，fine-grained token 不行** —— metrics 对细粒度 token 支持有 bug，
> 这是社区里反复踩的坑。

> 💡 两个 secret 填同一个 PAT 完全没问题，省事。

---

## 5. 开启 Actions 写权限（容易漏）

路径：仓库 → **Settings → Actions → General → Workflow permissions**

选 **Read and write permissions** → Save。

> 2023 年 2 月之后新建的仓库，`GITHUB_TOKEN` 默认是**只读**的。
> 不设这一步，`snake.yml` / `contrib.yml` 会在 push 时报 403。这是最常见的失败原因。

（本套配置里已经在每个需要的 workflow 中写了 `permissions: contents: write`，但仓库级的这个开关仍建议打开。）

---

## 6. 首次验证

进 **Actions** 标签页：

1. 左侧选一个 workflow，比如 **Snake Contrib**
2. 右边 **Run workflow** → 绿色按钮
3. 等一两分钟，看它是否变绿
4. 检查仓库里有没有多出 `profile-snake-contrib/` 目录

逐个手动跑一遍，全部变绿就说明配好了。

之后每天自动执行，四条**刻意错开**（`metrics` 08:13 / `snake` 08:43 / `3D Contrib` 09:13，`fork_star` 08:13 只改描述不推代码）：

| workflow | 北京时间 | 干什么 |
|---|---|---|
| GitHub Metrics | 08:13 | 更新 `github-metrics/*.svg` |
| Update Fork Star | 08:13 | 更新仓库简介里的 star / fork 数（走 API，不推 git） |
| Snake Contrib | 08:43 | 更新 `profile-snake-contrib/*.svg` |
| 3D Contrib | 09:13 | 更新 `profile-3d-contrib/*.svg` |

> **为什么不用整点、为什么间隔 30 分钟**：GitHub 的 schedule 在整点（尤其 UTC 00:00）是全网最拥堵的时刻，官方文档明确说会被延迟，实测延迟十几分钟很常见。
> `metrics` / `snake` / `3D Contrib` 三条都要写 `main` 分支，一旦被延迟到同一时刻就会互相把 push 顶掉（`! [rejected] ... (fetch first)`）。
> 所以这里一方面避开整点、拉开 30 分钟间隔，另一方面 `snake` 和 `3D Contrib` 的 push 都带 `git pull --rebase` 重试兜底，即使偶发撞车也能自愈。

---

## 7. 需要你自己改的地方

| 文件 | 改什么 |
|---|---|
| `README.md` | 不需要的图片区块，整段删掉即可 |
| `metrics.yml` | 末尾那段被注释的区块，说明见第 8 节 |

> `README.md` 顶部的自我介绍（姓名 / 行业 / 技术与平台 / 正在啃）和「💼 工作内容」清单已按 **shawn**（光通信行业）填好，想改直接动那几行。
> 联系方式（邮箱）写在自我介绍里，若担心被爬虫采集，可以改成 `shbao [at] outlook.com` 这类写法。
>
> `blog.yml`（博客 RSS 抓取）已移除。以后想加回来，把博客地址给我就行。
>
> `waka.yml`（WakaTime 统计）及其在 `metrics.yml` 中的 WakaTime 步骤已一并移除 ——
> 这也是 Secret 从原本的 4 个降到 **2 个**的原因。以后想加回来再说。

---

## 8. 常见坑速查

| 现象 | 原因 |
|---|---|
| `403 Permission denied` | 没开 Read and write permissions，或 PAT 没勾 `repo` |
| workflows 列表里看不到文件 | 文件不在默认分支，或 YAML 缩进错了 |
| 定时任务不触发 | 仓库 60 天无活动时 GitHub 会自动停用 scheduled workflow（本套每天都 commit，不会触发这条） |
| 图片显示 404 | 对应的 workflow 还没跑过，SVG 还没生成 |
| 定时不准时 | GitHub 的 cron 在高峰期会排队延迟，甚至跳过，属于正常现象 |
| `! [rejected] main -> main (fetch first)` | 多个 workflow 同时在推 `main`。本套已错开定时并加了重试，偶发一次重跑即可 |
| 图上显示 **Unexpected error** | 见下方专门说明 |
| 语言统计显示 **No push activity found** | 没抓到可分析的仓库。本套已改用标准模式，会读 GitHub 的语言元数据；若你只往私有仓库提交，也可能仍为空 |

### 关于图上的「Unexpected error」

2026-10-10 排查确认：**这是上游 `lowlighter/metrics` 与 GitHub API 变更不兼容，不是本仓库配置错**
（token 权限 `read:user, repo`、仓库设置都正常，同在跑的其他 8 张图全部正常）。
问题出在 3 个插件上，`metrics.yml` 里这三段已注释停用：

| 图 | 插件 | 根因 | 证据 |
|---|---|---|---|
| 成就 | `achievements` | 查询里带 `projects`（Projects classic）字段，GitHub 已废弃该字段并直接返回错误，整个 GraphQL 请求被判失败 | `achievements/queries/achievements.graphql` 第 50 行 |
| 最近动态 | `activity` | 插件读 `payload.commits`，而 **GitHub 事件 API 现在返回的 PushEvent 里已经没有 `commits` 字段** | 实测 `torvalds` 84 条、`lowlighter` 34 条 PushEvent，`commits` 全部缺失 |
| 编码习惯 | `habits` | 同上，`payload.commits` 缺失 → 插件内部 `TypeError: Cannot destructure property 'author'` | 运行日志 |

因为 `lowlighter/metrics` 已处于维护停滞状态（作者 burnout，主仓库最后更新 2026-05），
短期内不会修。上游修复后，把 `metrics.yml` 末尾对应段落的注释去掉、并在 `README.md` 加回图片引用即可。
另外 `repositories.pinned`（需要先在主页置顶公开仓库）和 `discussions`（仓库没开 Discussions）也一并注释了
—— 它们不报错，只是显示空图。

---

## 9. 想关掉某个功能

直接删掉 `.github/workflows/` 下对应文件，再从 `README.md` 里移除图片引用即可，互不影响。

| 不要这个 | 删哪个文件 |
|---|---|
| 贪吃蛇 | `snake.yml` |
| 3D 贡献图 | `contrib.yml` |
| 全部 metrics 徽章 | `metrics.yml` |
| 仓库简介 star 计数 | `fork_star.yml` |
