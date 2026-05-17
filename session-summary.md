# Ayu Tab v5 开发会话总结

日期：2026-04-17 至 2026-04-18
主项目：`/Users/wayne/Documents/Ayu Desktop/index_v5.html`
关联仓库：`https://github.com/AyuStudio11/index`

---

## 1. 会话概览

本次会话对单文件 HTML 仪表盘（番茄钟 / 笔记 / 待办 / 快捷方式 / 宠物伴侣）进行了深度迭代，核心围绕**宠物系统重构**和**UI/UX 打磨**。最终产物推送到远端 `main` 的 `index_beta.html` 作为下一版公开 beta。

---

## 2. 环境 / 工具配置

| 项 | 操作 |
|---|---|
| Chrome DevTools MCP | `claude mcp add chrome-devtools -- npx chrome-devtools-mcp@latest` |
| Preview hook | `/Users/wayne/.claude/hooks/preview-html.sh` —— 编辑 `index_v*.html` 后自动 reload Chrome 里的对应 tab（按 basename 匹配避免 URL 空格编码问题） |
| AppleScript JS 权限 | 用户在 Chrome 菜单启用 "查看 → 开发者 → 允许 Apple 事件中的 JavaScript"，之后可通过 osascript 直接操作 Chrome tab |
| Claude Code 权限模式 | 项目 `settings.local.json` 加 `"defaultMode": "bypassPermissions"`，自动放行所有工具调用 |
| Git 远端迁移 | `origin` 从 `WayneLeo935/index.git` 更新到 `AyuStudio11/index.git` |

---

## 3. 宠物系统主要改动（index_v5.html）

### 3.1 架构重构：删除 pet-home 卡片
- 移除 `#pet-home`（右下角 280×180 卡片）和内层 `#pet-main`
- 宠物直接挂在 `#main-container`，在全屏范围内活动
- 活跃 / 安静模式都基于 main-container 坐标系，只是步长和选点逻辑不同
- 删除"重置宠物"按钮和 modal，相关函数 `openPetResetConfirm` / `executePetReset` 一并清除
- 坐标系迁移：`pet.posVersion=2` 标记清除旧 `pet.pos`（旧版是相对 pet-main 坐标）

### 3.2 FSM 模式切换编排

**Quiet → Active**（happy 2s → walking 一步 → 活跃 FSM）
```
triggerOneShotState('happy', 2000)
  → (2s 后) Phase 2：pickQuietTarget 小走一步 + startWalkingAnimation
  → (walk 完成) Phase 3：startPetRoam 进入活跃 FSM
```
定时器句柄 `petTimers.modeA/modeB`，快速切换时被 stopPetRoam 清理。

**Active → Quiet**（sitting 5s → 安静 FSM）
```
setPetState('sitting')
  → (5s) startPetRoam 进入 quiet FSM
```

### 3.3 走路帧动画
- 素材：每个品种新增 `walking 1.png` / `walking 2.png`（文件名含空格）
- `startWalkingAnimation()` / `stopWalkingAnimation()` 以 setInterval 交替 src
- 步频按品种：`PET_WALK_FRAME_MS = { cat: 350, pig: 350, dog: 200, monkey: 200 }`
- 直接换 src（不动 opacity），预加载保证缓存命中 → 零闪烁
- FSM walking 和 enterActiveMode Phase 2 都复用这对函数

### 3.4 Sitting 状态
- 素材：每个品种新增 `sitting.png`
- `PET_STATE_IMAGE.sitting = 'sitting'`
- `runFsmSitting` 直接 `setPetState('sitting')`，不再用 sleeping 素材占位

### 3.5 目标选点避障
- `pickActiveTarget`：围绕当前位置在 `MAX_STEP=320px` 内随机选点
- `pickQuietTarget`：`step=80px`，围绕当前位置
- 两者都用 `getPetObstacleRectsLocal` 避开 `.search-container` / `#panel-shortcuts` / `#panel-note-col` / `#panel-todo-col`
- 拖拽**不**做避障（用户可自由拖进卡片），只 clamp 到 main-container 边界

### 3.6 状态与事件
- 一次性动画（happy / celebrate / eating / sad）→ `triggerOneShotState`，用 `petTimers.oneShot[state]` 管理
- `recomputePetState` 末尾回落：若 `petFsm.state === 'walking'` 或 `'sitting'`，一次性事件结束后自动恢复走路 / 坐姿视觉（不会卡在 idle）
- `stopPetRoam` 清理所有瞬时状态 + 冻结当前视觉位置（`freezePetPosition`），防止旧 transition 继续滑向旧目标
- `petFaceMouse` 守卫：sleeping / yawning / walking 状态不朝鼠标翻面；也覆盖 `dataset.state` 和 `petActiveStates` 判断，兼顾 sitting-as-sleeping 素材的老行为已被删除

### 3.7 Sleep 态净化
- FSM sleeping 自睡醒 20-40s
- 仅点击 / 自醒两种方式结束睡眠
- Hover 不唤醒、拖拽不唤醒也不翻面

### 3.8 位置持久化
- `window.addEventListener('pagehide', ...)` 保存宠物当前位置到 `pet.pos`
- 加载时 transition 临时关闭，瞬间落位，不从 (0,0) 滑过来
- 未保存位置时兜底落到右上角默认点（`getPetInitialCorner()`）

---

## 4. UI 组件

### 4.1 首次引导气泡
- SVG 对话气泡 + 一体化三角尾巴，尾巴尖精确对准右上角宠物按钮中心
- 宽 170px，高 64px，内含文字"开启宠物功能"（非点击）+ × 关闭按钮
- 显示条件：`!isPetInitialized() && !isPetHintDismissed()`
- 悬停右上角宠物按钮时（`body:has(#pet-nav-wrap:hover) #pet-hint`）气泡淡出避让下拉菜单

### 4.2 呼吸灯引导
- `#pet-nav-btn` 和下拉菜单"初始化"按钮同步呼吸（`.breath` class）
- 首次初始化完成或关闭提示后，两处同时熄灭

### 4.3 名字 + 心情悬停显示
- `#pet-name` 位于宠物下方 `top:104px`，字号 17px
- hover 宠物触发，和心情爱心一起淡入淡出
- 心情在宠物头顶 `top: 4px`

### 4.4 下拉菜单"初始化"置顶
- 下拉项顺序：初始化 → 活跃 → 显示
- 未初始化时"初始化"项呼吸（`petNavInitBreath` 背景脉动）

### 4.5 z-index 布局
- 顶栏 wrapper 固定 `z-30`（不再随 hover 跳到 z-50）
- 宠物 `z-40`（默认在图标上方）
- pet-hint `z-20`（低于顶栏，避免挡其他下拉）
- 悬停顶栏按钮时，`body:has(#top-nav-buttons:hover) #pet` → 宠物整体淡出 0.25s，下拉菜单完整可见，松开恢复

### 4.6 布局数值
- 搜索栏 `max-w-[800px]`
- 下方分栏 `px-[72px]`（加上 main-container 的 p-12 = 距两侧 120px）
- 时间块 `-mt-[50px]`
- 格言 `-mt-[20px]`、`mb-16`
- 搜索栏和下方分栏间距 `mt-16`

---

## 5. 系统修复

### 5.1 定位权限不记录
file:// 下 Chrome 不持久化 geolocation。实现：坐标首次获取后存 `localStorage.weather.coords`，7 天 TTL，缓存命中直接复用，不再触发权限弹窗。

### 5.2 Lucide 图标版本
`lucide@latest` 移除了 `github` 等品牌图标 → 固定到 `lucide@0.294.0`，保留默认书签的 GitHub logo。

### 5.3 Yawning 素材缺失
无 `yawning.png`，`PET_STATE_IMAGE.yawning = 'sleeping'` 复用睡姿视觉（yawning 是 1.5s 入睡过渡，衔接自然）。

### 5.4 图片切换闪烁
所有素材通过 `preloadPetImages` 预加载。`setPetState` 删除 80ms opacity 0 → 1 假过渡，直接换 src，缓存命中下零闪烁。

### 5.5 默认活跃模式
迁移：首次访问 `pet.active === null` 时，默认设为 `'1'`（活跃）。下拉菜单标签显示"活跃"，FSM 进入活跃模式。

### 5.6 初始化流程
- 首次访问：`pet` 元素 `display: none`，引导气泡 + 呼吸灯双重提示
- 完成引导（点击"初始化"→ 填名字 → 选动物）→ `confirmPetOnboard` 设 active flag + 落位右上角 + happy 2s + 启动 FSM
- 初始化后气泡永久隐藏，呼吸灯熄灭

---

## 6. 代码质量改进

| 改进 | 实现 |
|---|---|
| 全局污染 | `window._pet*` → `petTimers = { modeA, modeB, celebrate, oneShot:{} }` 命名空间 |
| 代码重复 | 抽 `getPetInitialCorner()` / `freezePetPosition()` / `clearPetModeTimers()` |
| Race condition | Phase 2 回调显式清 `happy` oneShot，避免与 Phase 2 setPetState 顺序依赖 |
| bfcache | 去掉 `beforeunload`，只保留 `pagehide`（现代浏览器推荐） |

---

## 7. Git 操作

### 7.1 Committer 重写
用 `git filter-branch --env-filter` 重写全部 34 个本地 main 提交的 committer / author 为 `Ayu Studio <AyuStudio11@users.noreply.github.com>`。

### 7.2 推送分支策略
- 本地 main 和 origin/main 历史分叉（committer 重写后）
- 新功能推送通过 worktree + 临时分支 fast-forward 到 origin/main，避免 force push
- 最终 commit `24b4827` 同步 v5 最新功能到 `index_beta.html`（32 个文件变更）

### 7.3 远端 URL 更新
从 `WayneLeo935/index.git` 迁移到 `AyuStudio11/index.git`。

---

## 8. 分享链接

**GitHub Pages（推荐，几分钟更新）**
```
https://ayustudio11.github.io/index/index_beta.html
```

**GitHub 源码视图**
```
https://github.com/AyuStudio11/index/blob/main/index_beta.html
```

---

## 9. 已知限制 / 后续工作

- Tailwind CDN 生产警告：单文件无构建流程，暂忽略
- SVG 气泡的 viewBox / 尺寸 / right 偏移三处耦合，调整需同步三个值
- 移动端未专门适配
- 宠物名字字号 / 位置、气泡尺寸等仍是硬编码，如需主题化可抽 CSS 变量

---

## 10. 文件清单

**修改的核心文件**
- `index_v5.html`（工作文件，本地一直在迭代）
- `index_beta.html`（远端 main，已同步 v5 最新）

**新增素材**（每个品种）
- `assets/pets/<breed>/sitting.png`
- `assets/pets/<breed>/walking 1.png`
- `assets/pets/<breed>/walking 2.png`

**修改素材**（19 个现有 PNG 被用户更新）
- 各品种的 cheering / dragged / focused / idle / sad / sleeping / eating 等

**工具脚本**
- `/Users/wayne/.claude/hooks/preview-html.sh`（PostToolUse 预览钩子）
- `/Users/wayne/Documents/Ayu Desktop/.claude/settings.local.json`（包含 hooks、permissions 配置）
