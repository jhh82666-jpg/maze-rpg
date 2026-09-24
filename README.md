# 迷宫 RPG · 发布版（可直接运行的静态站点）

**体积 1.77 MB / 124 个文件** —— 只含游戏运行所必需的东西，不含源码与开发产物。

- 构建来源：`maze-rpg` 主仓库，`vite build`（Vite 7.3.6 + Phaser 3.90.0）
- 构建日期：2026-09-24
- 逻辑分辨率基准：1920×1080（`自适应窗口` 会按窗口向下取档：1280/1600/1920/2560/3840）
- 纯前端单机、无后端、无网络请求
- **独立验收结论：PASS** —— 见第六节（真实浏览器实测，0 报错）

---

## 一、怎么运行

这是 **HTTP 静态站点**，不是双击即开的单文件。**必须经一个静态服务器打开**，
原因：产物用 `<script type="module">`，从 `file://` 打开会被浏览器的 CORS 策略拦掉，
页面会停在「正在初始化…」（`index.html` 里已内置这段兜底提示）。

任选一种（都无需 Node 工具链）：

```bash
# 方式 1：Python 自带（最省事）
python -m http.server 8080
#   然后浏览器打开 http://127.0.0.1:8080

# 方式 2：Node 生态
npx serve .            # 或 npx http-server -p 8080

# 方式 3：直接托管到 GitHub Pages / Netlify / Vercel / 任意静态空间
#   把本目录整体上传即可（base 已是 './'，放在子路径下也能跑）
```

> 想要**双击即玩**（无需服务器）？那需要回主仓库跑一次
> `npm run build:offline`，它会产出 `dist/index.offline.html`（脚本内联的离线单文件版）。
> 本目录是标准静态站点形态，不包含该单文件。

---

## 二、目录结构

```
maze-rpg-release/
├── index.html                     4.0 KB   页面外壳（画布容器 + 首帧加载占位 + file:// 兜底提示）
├── assets/
│   ├── phaser-DXXWZowi.js     1 179.7 KB   Phaser 3.90.0 引擎（独立 vendor chunk）
│   ├── index-Bqb3Txiu.js        400.6 KB   游戏逻辑：场景 / 系统 / UI + **全部配表 JSON**
│   └── portraits/               222 KB     角色立绘占位图 120 张（见第三节）
│       ├── portrait_01..40.png            256×256   头像（角色卡 / 图鉴 / 状态栏）
│       ├── bust_01..40.png                384×512   半身像（战斗角色卡）
│       └── full_01..40.png                512×1024  全身像
└── README.md                               本文件
```

**文件数说明**：124 = 1（index.html）+ 2（打包 JS）+ 120（立绘占位）+ 1（本 README）。
其中只有 **123 个是运行必需**，`README.md` 是给人看的说明。

### 为什么没有独立的数据文件？

全部配表（角色 / 技能 / 敌人 / 道具 / 科技 / 地牢 / 大地图 / 事件 / 任务 / 战术 / 饰品）
都在 `src/data/tables/*.json`，构建时经 `src/data/tables/*.gen.ts` **打进 `index-*.js`**，
运行期由 `DataLoader` 同步读取。所以运行时不依赖任何 `.json` 文件，也**不需要** `fetch()` —— 这也是它能纯静态托管的原因。

### 为什么没有 `.webp`

源码里 `src/config/constants.ts` 的 `PortraitAssets` 三个取值函数**一律返回 `.png`**，
全仓 `src/` 对 `webp` 零引用 ⇒ 占位版把 120 个 WebP 全部剔除（原为 24 MB 死重）。
⚠️ 主仓库为了断言完整性仍然保留 WebP；**只有本发布目录如此精简**。

---

## 三、立绘是白图占位（重要）

`assets/portraits/` 下的 120 张 PNG 是**纯白占位图**，不是真实美术：

- 尺寸**严格按三档规格**生成（256×256 / 384×512 / 512×1024），色彩类型 **RGBA** ——
  与真实立绘一致，所以**布局不会因为换占位图而位移**；
- 这样做的目的是把发布体积从 **47 MB 压到 222 KB**，同时让游戏能正常加载、无 404、无报错。

**换回真实立绘**：把主仓库 `public/assets/portraits/` 下的同名文件覆盖过来即可（120 张 PNG），
或直接重新构建（见第四节）。

> 立绘加载失败**不会阻塞游戏**：`PreloadScene` 监听 `FILE_LOAD_ERROR` 只打一条 warning，
> `PortraitCard` 会自行降级占位。所以即使一张图都不放，游戏也能跑起来。

---

## 四、怎么重新生成这个目录

在主仓库 `maze-rpg/` 下：

```bash
# 1) 类型检查 + 构建，直接输出到发布目录
npm run typecheck
node node_modules/vite/bin/vite.js build --outDir ../maze-rpg-release --emptyOutDir

# 2) 把立绘换成白图占位，并剔除零引用的 WebP
python scripts/make-release-placeholders.py ../maze-rpg-release
```

`--emptyOutDir` 是必需的：`outDir` 在项目根目录之外时，Vite 默认**不会**清空它。

---

## 五、附：源码仓库（GitHub）上传范围

本发布目录是**构建产物**。若你要上传的是**游戏本体源码**，上传范围如下：

**应上传（8 项，合计约 1.5 MB）**

| 路径 | 说明 |
|---|---|
| `src/` | 全部游戏源码（104 文件，1.3 MB） |
| `public/assets/portraits/` | 立绘（**建议同样用白图占位**；只放 120 个 PNG 即可，WebP 可省） |
| `index.html` | 页面外壳 |
| `package.json` + `package-lock.json` | 依赖声明（运行时依赖只有 `phaser`） |
| `tsconfig.json` | TS 配置（`@/*` 路径别名在 `vite.config.ts` 与这里各有一份，缺一不可） |
| `vite.config.ts` | 构建配置 |
| `.gitignore` | 已在仓库内 |
| `README.md` | 仓库门面 |

**必须排除（开发过程产物 / 素材仓库，占 2 GB 以上）**

| 路径 | 体积 | 为什么排除 |
|---|---|---|
| `docs/evidence/` | ~1.5 GB | 截图取证的浏览器 profile 与产物 |
| `image/`、`image2/`、`assets-src/` | ~338 MB | 立绘源图（构建产物已在 `public/`） |
| `.backup/` | ~118 MB | 大改前的全量备份 |
| `_portrait-swap/`、`zz-tactic-out/`、`_ui-plan/` | ~165 MB | 过程报告、跑批输出、方案原图 |
| `dist/`、`node_modules/` | — | 构建产物与依赖（`.gitignore` 已覆盖） |

`design/`（GDD）、`docs/handbook.md`（技术总文档）、`scripts/`（验证链与生成器）
属**可选**——它们不影响游戏运行，但会公开内部设计与工具链。

---

## 六、独立验收（实测结论，2026-09-24）

用真实浏览器（headless Edge）把本目录跑起来实测，不是"看起来应该能跑"：

| 检查项 | 结果 |
|---|---|
| 游戏就绪（canvas 出现 **且** `#boot-splash` 被隐藏 —— 后者只在 `PreloadScene` 走完全部加载步骤后才发生） | ✅ 是（canvas 1600×900） |
| 主菜单渲染 | ✅ 标题 + 主 CTA「开始新游戏」+ 配表统计「角色 40 / 技能 101 / 敌人 44 / 地牢 4」全部正常 |
| 控制台 error / exception | ✅ **0** |
| 控制台 warning | ✅ **0** |
| 网络加载失败 | ✅ **0** |
| HTTP ≥400 | ✅ **0**（游戏资产 120 张立绘全部 200） |
| 与「干净重建」逐字节比对 | ✅ `index.html` / `index-*.js` / `phaser-*.js` 三个产物 **哈希完全一致**（构建确定性） |

**唯一一条无伤大雅的 404**：浏览器会**自动**请求 `/favicon.ico`（`index.html` 并未声明图标），
服务器返回 404。它不影响任何功能，也不是游戏资产。若想让控制台彻底干净，二选一：

- 在 `index.html` 的 `<head>` 里加一行 `<link rel="icon" href="data:,">`（零字节，直接抑制该请求）；
- 或放一个真实的 `favicon.ico` / `favicon.png`。

