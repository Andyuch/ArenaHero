# 摩尔勇士复兴版 · Flash Game 自动化（AI Agent 接手文档）

> 当前前沿（g48，2026-09-27）：已稳定进游戏、走完开场女神对话，
> 卡在**取名页**——"请叫我"输入框没把名字敲进去（`g48_z1.png` 为空）。
> 照着这篇文档可直接续跑。分支：`arena/01a0bfc0-arenahero`。

## 0. TL;DR

- 游戏：摩尔勇士复兴版（Mole Hero，Flash AS3）。
  客户端：`https://github.com/RecTaoMee/RecTaoMee_Hero.git`（跑 `Client/Client.swf`）。
- 服务器：socket `202.189.23.31:1800`，配置 `http://202.189.23.31:93`。
- 跑法：GitHub Actions（ubuntu-latest）+ xvfb(1280x900x24) +
  **Ruffle 桌面版** + xdotool + scrot；结果截图/日志经 git 推回 `gh-out/`。
- Workflow：`.github/workflows/game.yml`，**只有 push
  `.github/workflows/.trigger` 才会触发一次 run**（约 6–7 分钟）。
- 输入三要素：`windowfocus --sync` → 点中输入框 → 普通 XTEST `type`，
  每次输入后截图验字节/RMSE，不变就重试。

## 1. 账号（已注册；邮箱验证链接没点）

| 项 | 值 |
|---|---|
| 鼹兴号 | `116031` |
| 密码 | `Tmp#eeYhnBdlB`（b64：`VG1wI2VlWWhuQmRsQg==`；workflow 里只存 b64，绝不 echo 明文） |
| 绑定邮箱 | `mmee3377@protonmail.com` |
| 验证政策 | 用户决定**不做邮箱验证**（`no_verify`）。若服务器硬性要求验证而被挡，接受被挡并停止。 |

用户目标：① 进游戏（选**魔法师**）；② 尽力满级/全装备/宠物/属性/每种10级宝石100颗。
只允许正当机制 + 自动化 + 协议 bot PoC；**不做**刷物品/复制/伪造封包/SQLi/
绕验证/洪水。对服务器温柔：短 burst、不扫描不 fuzz、两次 run 之间留间隔。

## 2. 跑一次的标准流程

```bash
git checkout arena/01a0bfc0-arenahero
# 1. 改 .github/workflows/game.yml，提交并推
git add .github/workflows/game.yml && git commit -m "game gN: ..." && git push origin arena/01a0bfc0-arenahero
# 2. 触发（只有 .trigger 变化才会跑；想停就别碰它）
date -u > .github/workflows/.trigger && git add .github/workflows/.trigger \
  && git commit -m "trigger gN" && git push origin arena/01a0bfc0-arenahero
# 3. 轮询（gh 认证已配好）
gh run list --limit 5 | grep 'game gN'
# 4. CI 自动把 gh-out/ 推回分支，拉下来看图说话
git pull --rebase origin arena/01a0bfc0-arenahero
```

注意：本仓库可能是单分支克隆，先执行一次
`git config --add remote.origin.fetch "+refs/heads/arena/01a0bfc0-arenahero:refs/remotes/origin/arena/01a0bfc0-arenahero"`
再 `git fetch origin`，否则看不到远端分支。

## 3. 当前配方（g48，经 48 轮验证，改前先读 `game.yml` 全文）

启动环境（run.sh 内）：

```bash
export LIBGL_ALWAYS_SOFTWARE=1 XDG_RUNTIME_DIR=/tmp/xdg; mkdir -p $XDG_RUNTIME_DIR
unset VK_ICD_FILENAMES
ruffle -p low --width 1012 --height 670 \
  --tcp-connections allow --socket-allow 202.189.23.31:1800 \
  --open-url-mode allow --filesystem-access-mode allow \
  Client.swf > /tmp/ruffle.log 2>&1 &
```

- `--socket-allow` 等 flag（g18）免掉了 Ruffle 联网授权弹窗；仍保留一次
  `(700,460)` 的 early_allow 兜底点击（裸屏上该点是空白面板，无害）。
- 片头动画：用 24 字节 stub SWF 覆盖 `Client/res/movie/movie_1..4.swf`（g28/g29），
  否则 Ruffle 播片极慢；另装 `openh264` + ALSA null 音频（g19–g21）。
- Screenshot 即 ground truth：`scrot` 全屏；字节数只做启发式（g13 出现过
  同字节不同内容的碰撞）；跨场景判断用
  `compare -metric RMSE a.png b.png null:`，阈值约 3900（见 `gate()`）。

全流程坐标（1280×900，Ruffle 窗口在 (0,0)，截图坐标 = xdotool 坐标）：

| 步骤 | 动作 | 备注 |
|---|---|---|
| splash 封面(~930KB) | 点开始 (570,573) | 按 t0 字节>500000 判断是否还在封面 |
| 登录面板(空=382111B，逐字节稳定) | 账号框 (560,196)，密码框 (560,238)，开始 (582,385) | 20×BackSpace 清空→`type --delay 120`→睡 8s→截图验字节→重试×3 |
| 选服/进场 | 提交后 w0/w1/w2 连拍 | 约 30s 后进洞穴场景（`g48_w2.png`） |
| 点小精灵开对话 | (206/218/230, 278) 连点 | RMSE 门限确认场景变化 |
| 女神对话翻页 | 点继续箭头 (717,350)，7s 一张 p0–p3 | 时之女神·贝尔丹蒂剧情 |
| **取名页（当前卡点）** | 点输入框 (577,470)→清空→type 名字→Return | g48 用名 `Mage116031`，z1 显示**空**，没进去 |

ruffle.log 关键 trace：`加载登录面板成功`、`>>> [CONNECT] 202.189.23.31:1800`、
`你使用的是MIMI号`；legacy 域名 401/404（account-httpd.61.com、ip.txt）非致命。

## 4. 证据目录约定（`gh-out/`）

- `gN_t0..t6.png`：splash→登录→账号→密码→提交后；`w*` 进场连拍；
  `f/r/p/m/c/x/z` 各代自定义（p=对话翻页，m=取名，z=终态）。
- `prog.txt`： checkpoints + 字节数 + RMSE 值；`gNdiag.txt`：环境/依赖记录；
  `ruffle.log`：播放器 + AS3 trace（最新一次 run 的）。
- **每次 run 必须留截图证据**，结论要落实到某张图，不许凭字节数脑补。

## 5. 台账 g1–g48（凭 commit message + 截图整理）

- g1–g7：渲染/克隆/字体/Vulkan 修到 Ruffle 能跑（`-p low` + 全量 clone）。
- g8–g10：证明"无 WM 下键盘必须 windowfocus；`type --window` 被 winit 忽略"。
- g11：XTEST `type` 有效；Tab 切框不可靠。
- g12：坐标教训——(588,221) 落在两框缝隙，输入全丢。
- g13：修正坐标，账号密码一次填对；发现 Ruffle 联网弹窗挡 socket。
- g14–g18：Allow 点击 → `--tcp-connections allow --socket-allow` 一劳永逸。
- g19–g22：openh264 + null 音频 + hover 扫片头。
- g23–g26：进服 10 + 跳过 frenzy（Esc/space/测得的 skip 坐标）。
- g27–g29：删片头实验 → stub movie SWF 定稿。
- g30–g36：tutorial 交互、对话爆破、真箭头循环 + NPC 重试点。
- g37–g40：skip 猎杀/探索、仙子对话 battery、翻页 battery、talk 复刻重试。
- g41–g44：取名+推进、真坐标+干净对话协议、木桩点击隔离。
- g45–g48：女神推进 battery、RMSE 门限、取名页 + 确认（名字未落定）。

## 6. 坟场（别再踩）

- openbox + 裸 `wait`：openbox 永不退出，`timeout` 打死整个 step（g9）。
- `xdotool type --window`：XSendEvent 合成键，winit 直接无视（g10）。
- 盲 Tab 切输入框：Flash 里行为不定（g11）。
- 死坐标不验就往下走：必须"输入→截图→比对→重试"闭环（g12）。
- 同字节=同画面：PNG 字节碰撞真实存在，看图为准（g13）。
- Ruffle 里播真片头：极慢，用 stub SWF 跳过（g27/g28）。

## 7. 下一步（按顺序，欢迎认领）

1. **取名页**：`g48_z1.png` 输入框为空。先加登录同款 verify-retry（重 focus→
   重点 (577,470)→重输→RMSE 验），再找确认按钮（可能 Return 不是确认键，
   要点某个确认坐标）。
2. 取名成功后继续 tutorial，职业选择出现时选**魔法师**。
3. 之后：主线/挂机/刷级脚本、装备宠物宝石（用户目标 §1），或协议层 bot PoC。
4. 基础设施：模板匹配代替死坐标、RMSE 自动门限、run 时间压缩到 5 分钟内。

贡献规矩：沿用 `game gN` 命名（workflow 改动 / trigger / results 三 commit），
`gh-out/` 留截图，`prog.txt` 留数字，密码只许 b64，短 burst 爱护服务器。
