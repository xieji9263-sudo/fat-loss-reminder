# 减脂提醒 · 云端版（GitHub Actions）

每天定时生成午餐/晚餐点单指令，推送到微信（Server酱）。跑在 GitHub 的机器上，**不需要开电脑**。

仓库：`xieji9263-sudo/fat-loss-reminder`（私有），workflow 文件：`.github/workflows/reminder.yml`

## 文件说明

| 文件 | 作用 |
| --- | --- |
| `remind_lite.py` | **唯一权威源码**。全部逻辑（轮值表 + 周边店铺表 + 推送） |
| `make_workflow.py` | 把 `remind_lite.py` 缩进嵌入 YAML，生成 `single-file-remind.yml` |
| `single-file-remind.yml` | **要放进 GitHub 的那份**。25165 字节 / 368 行 |
| `../tools/gh_update_file.py` | 用 PAT 直接更新仓库文件（连接器只读时的替代路径，见文末） |

改逻辑只改 `remind_lite.py`，然后 `python3 make_workflow.py` 重新生成。

> YAML 的 `run: |` 块要求块内统一缩进，内嵌 Python 从第 1 列开始 —— 手写必踩缩进坑，
> 所以用脚本统一加 10 个空格。注意 heredoc 结束标记 `PYEOF` 也必须带这层缩进。

## 一次性配置

### 1. SendKey（必填）

Server酱：<https://sct.ftqq.com> 登录后复制 SendKey。

仓库 → **Settings → Secrets and variables → Actions → New repository secret**

| Name | Value |
| --- | --- |
| `SENDKEY` | Server酱 SendKey |

不配也能跑，但只打印内容、不推送。

### 2. TENCENT_MAP_KEY（可选）

**不配也能用** —— 脚本内置了一张实测店铺表（2026-09-14 采，16 个品类、90+ 家门店）。
配上之后，每天改用腾讯位置服务**实时**搜周边，店名/距离自动更新，不怕店关了。

`<https://lbs.qq.com/>` → 微信扫码登录 → 实名认证 → 创建应用 → 添加 Key →
**必须勾选 `WebServiceAPI`** → 复制 Key（个人开发者免费，1 万次/日）。

| Name | Value |
| --- | --- |
| `TENCENT_MAP_KEY` | 腾讯位置服务 WebService Key |

申请时若在配额页看到「一键分配」，点一下把额度分给这个 Key。

### 3. 手动跑一次验证

**Actions** → 左侧「减脂提醒」→ **Run workflow** → 选 `dinner` → 绿色按钮。
微信收到、且内容里有「## 附近 1km 内」即成功。

## 什么时候跑

| 餐次 | 北京时间 | cron（UTC） |
| --- | --- | --- |
| 午餐 | 10:37 | `37 2 * * *` |
| 晚餐 | 16:07 | `7 8 * * *` |

- **故意错开整点**：GitHub 定时任务在整点最拥堵，官方承认会延迟。`:07` / `:37` 明显更准。
- **延迟是常态**。手动触发（Run workflow）是立刻跑的，用来验证。
- 仓库 60 天无提交活动，定时任务会被自动暂停；每次跑完会自动提交 `last-run.txt` 保活。

## 周边店铺怎么给的

每个档位带 6–9 家**真实门店**，让书记自己挑，不是只推一家：

```
## 附近 1km 内 · 出餐+配送 25 分钟稳
- [轻食] 食野SAYYEAH创意轻食(拱墅店) · 705m · 七古路85号光明商业2幢111室
- [卤味] 快点来丫衢州鸭头 · 850m · 宸麟路411号
## 1–2.5km 备选 · 得提前点
- [轻食] MARCH&MUNCH 轻食咖啡 · 1017m · 上塘路988号宸融大厦负一楼
```

- **按距离分两段**：1km 内出餐+配送 25 分钟稳；1–2.5km 得提前下单。
- **档位关键词支持多个**（`轻食/卤味`），多品类时每行带 `[品类]` 标签，
  且**每个品类各留配额**（1km 内 4 家、1–2.5km 3 家），避免长名单品类把短名单品类挤没。
- 单品类时两段各列最多 6 家。
- `_distance` 是**直线距离**，不是步行距离。

### 兜底链

配了 key 走实时搜索 → 该关键词搜到的店不足 3 家时用内置 `SHOPS` 表补齐 →
完全没配 key 就直接读内置表 → 内置表也没有则写明「没搜到对口店，外卖 App 直接搜」。
**任一环节失败都不阻断主流程。**

关键词按实测校准过，别想当然：
- 「米粉」「便利店」「潮汕牛肉火锅」在周边搜索里**零结果**，改用「米线」「超市」「火锅」立刻有命中。
- 「清蒸鱼」「炖汤」按菜名搜也是零结果，改用「土菜」「海鲜」。
- 「韩国料理」1km 内确实没有正餐，最近的在 2.2km 外 —— 这是事实，不是 bug。

## 关键实现细节

- **取档公式**：`dayNum = floor(local_midnight / 86400)`，`idx = dayNum % 10` —— 与 `site/index.html`
  的 `nextMealCard` **完全一致**，两边不会算岔。
- **周边搜索 radius 只能是 10–1000 米**（腾讯官方限制，超过会报参数错误，`1500` 会直接被拒）。
- **`SLOT=0..9` 环境变量可手动指定档位**，用于测试和「换一个」（往后顺延一档）。

## 本地测试

```bash
MEAL=dinner python3 remind_lite.py                 # 不推送，只打印
MEAL=lunch  SLOT=4 python3 remind_lite.py          # 强制第 5 档
SENDKEY=xxx MEAL=dinner python3 remind_lite.py     # 真推送
python3 make_workflow.py                           # 重新生成 single-file-remind.yml
```

## 边界

- 只给**店名 + 直线距离 + 地址**。菜单、价格、起送价、能不能配送、优惠 —— 都拿不到，
  美团/饿了么不向个人开放接口。最后下单还是得在外卖 App 里做。
- 内置店铺表会过期（店会关）。想保持新鲜就配 `TENCENT_MAP_KEY`。
- 部分结果是熟食店/农贸市场摊位，不一定上外卖平台，先搜店名确认。

## 怎么直接改仓库里的文件

WorkBuddy 的 GitHub 连接器是**应用级安装令牌，只读**。写操作一律
`403 Resource not accessible by integration`，而且这个权限由应用声明，**用户在网页上提不了权**。

所以要用一个带写权限的 Personal Access Token：

```bash
python3 ../tools/gh_update_file.py xieji9263-sudo/fat-loss-reminder \
    .github/workflows/reminder.yml single-file-remind.yml main
```

Token 放 `~/.workbuddy/.secrets/github_token`（`chmod 600`），或走 `GITHUB_TOKEN` 环境变量。

**这个文件在 `.github/workflows/` 下，所以两条权限都得给**（GitHub 官方限制，缺一不可）：

| 类型 | 权限 |
| --- | --- |
| fine-grained PAT | Repository access 选该仓库；`Contents` **Read and write** + `Workflows` **Read and write** |
| classic PAT | 勾 `repo` + `workflow` |

只给 `Contents` 不够 —— 改 workflow 文件会单独要 `Workflows` / `workflow`。
若还想让脚本顺手触发一次 Actions 试跑，再加 `Actions` Read and write。
