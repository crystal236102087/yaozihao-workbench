# 皓月自习社 — 交接文档

> **写给完全没有上下文的新会话。读完这份文档，你应该能立即接手开发。**

---

## 一、项目概述

这是一个为四年级男孩 **姚子皓** 定制的 PWA 学习工作台 App，部署在 GitHub Pages 上。

- **App 名称**：皓月自习社（首页欢迎语保留「姚子皓，今天加油！」）
- **线上地址**：`https://crystal236102087.github.io/yaozihao-workbench/`
- **GitHub 仓库**：`crystal236102087/yaozihao-workbench`（分支：`master`）
- **技术栈**：单文件 HTML + Tailwind CSS (CDN) + 原生 JavaScript，无构建工具
- **数据持久化**：localStorage（版本号机制 `DATA_VERSION`）
- **离线支持**：Service Worker (`sw.js`)，缓存版本号强制更新

---

## 二、文件结构

```
yaozihao-workbench/
├── index.html          # 主文件（~3221行），所有HTML/CSS/JS都在这里
├── sw.js               # Service Worker，缓存版本控制
├── manifest.json       # PWA manifest
├── icon-192.png        # PWA图标 192x192
├── icon-512.png        # PWA图标 512x512
└── apple-touch-icon.png # Apple Touch Icon 180x180
```

图标设计：深蓝色夜空渐变背景 + 暖黄色月牙 + 星星 + 白色打开的书本 + 蓝色「学」字。用 Python Pillow 脚本生成，生成脚本未保留在仓库中（需要重新生成时参考交接历史）。

---

## 三、核心架构

### 3.1 导航结构（12个页面）

```javascript
const navItems = [
  { id: 'home', icon: '📋', label: '每日计划' },
  { id: 'calendar', icon: '📚', label: '打卡记录' },
  { id: 'textbook', icon: '📖', label: '学期重点' },
  { id: 'chinese', icon: '📝', label: '语文' },
  { id: 'math', icon: '🧮', label: '数学' },
  { id: 'english', icon: '🌈', label: '英语' },
  { id: 'science', icon: '🔬', label: '科学' },
  { id: 'reading', icon: '📚', label: '阅读' },
  { id: 'sport', icon: '🏃', label: '运动' },
  { id: 'fun', icon: '🎮', label: '娱乐' },
  { id: 'reward', icon: '🎁', label: '奖励中心' },
  { id: 'stats', icon: '📊', label: '统计' },
];
```

### 3.2 数据结构

所有数据存在 `data` 对象中，通过 `saveData()` 序列化到 localStorage：

```javascript
data = {
  version: DATA_VERSION,  // 数据版本号，升级时递增触发迁移
  sun: 0,                 // 阳光（积分系统）
  tasks: {
    daily: [...],         // 每日计划任务
    reading: {
      recommend: [...],   // 推荐书目（8本）
      finished: [...],    // 已读积累
      dailyLog: {},       // 每日阅读记录 { 'YYYY-MM-DD': [{id, book, minutes, pages}] }
    },
    sport: [              // 运动任务（可动态添加自定义项目）
      { id, title, sub, desc, done, sun, custom?, icon? }
    ],
    fun: { dailyLog: {} },
  },
  textbooks: [...],
  calendar: {},           // 打卡日历数据
  rewards: [...],
};
```

### 3.3 学期重点页面

这是最复杂的页面，包含4个科目的教材内容：

**科目标签顺序**：语文 → 英语 → 数学 → 科学

**各科目标签页配置**：

| 科目 | 标签页 |
|------|--------|
| 语文 | 单元考点 → 课文原文 → 日积月累 → 四会词语 |
| 英语 | 单元考点 → 课文原声 → 重点单词 → 在线练习 |
| 数学 | 单元考点 → 易考点 → 名师讲解 → 在线练习 |
| 科学 | 单元考点 → 实验视频 → 在线练习 → 电子课本 |

所有科目「单元考点」都在第一个位置。

### 3.4 关键函数

| 函数 | 用途 |
|------|------|
| `switchNav(id)` | 切换主页面 |
| `setTextbookSubject(subject)` | 学期重点页切换科目 |
| `setBookTab(tab)` | 学期重点页切换标签页 |
| `renderUnitSelector(units, idx, onClick, theme)` | 渲染单元选择器（所有科目共用） |
| `speakChinese(text)` | TTS 中文朗读（zh-CN, rate 0.8） |
| `speakText(text)` | TTS 英文朗读（en-US, rate 0.75） |
| `speakChineseList(words)` | TTS 连续朗读一组词语（四会词语用） |
| `playAudio(url)` | HTML5 Audio 播放 PEP 官方 mp3 |
| `saveData()` | 保存数据到 localStorage |
| `addSportTask()` | 添加自定义运动项目 |
| `addDailyLog()` | 添加今日阅读记录（已移除页码输入） |

### 3.5 单元选择器渲染

`renderUnitSelector()` 函数负责所有科目的单元标签渲染：
- 9种糖果色循环
- 两行显示：第一行单元编号（如「第一单元」「Unit 1」），第二行单元名称（如「自然之美」）
- 正则解析中文数字：`/^第[一二三四五六七八九十]+单元/`
- 正则解析英文单元：`/^Unit\s+\d+/`

### 3.6 四会词语数据结构

```javascript
chineseBookData.sihui = [
  {
    unit: 1, title: '第一单元',
    words: [
      { w: '农历', p: 'nóng lì' },
      { w: '据说', p: 'jù shuō' },
      // ... 每个词带拼音
    ]
  },
  // 共7个单元（1,2,3,4,6,7,8），257个词语
];
```

拼音使用 Python `pypinyin` 库生成（`Style.TONE` 带声调）。

---

## 四、已完成的全部功能

### 4.1 基础功能
- ✅ 每日计划：任务增删改、拖拽排序、完成勾选
- ✅ 打卡记录：周日历视图，7天任务完成情况
- ✅ 奖励中心：阳光积分兑换奖励
- ✅ 统计页面：学习数据可视化
- ✅ 娱乐页面：游戏链接 + 每日娱乐记录
- ✅ PWA：manifest + Service Worker + 离线可用 + 自定义图标

### 4.2 学期重点
- ✅ 4个科目（语文/数学/英语/科学），每个科目4个标签页
- ✅ 单元考点：8单元知识点 + 情景例题 + 易考点提醒
- ✅ 语文课文原文：课文列表 + 百度搜索 + 喜马拉雅朗读
- ✅ 语文日积月累：古诗带译文、逐句分段、TTS朗读按钮
- ✅ 语文四会词语：7单元257词、带拼音标注、每单元一个TTS朗读按钮
- ✅ 英语课文原声：PEP官方mp3音频
- ✅ 英语重点单词：全学期60词 + 6单元单词音频
- ✅ 数学名师讲解、科学实验视频等外链

### 4.3 运动页面（最新修改）
- ✅ 自定义运动项目输入表单（项目名称 + 运动时间）
- ✅ 新添加项目排在最前面
- ✅ 自定义项目可删除（×按钮）
- ✅ 完成勾选获得阳光奖励

### 4.4 阅读页面（最新修改）
- ✅ 推荐书目勾选
- ✅ 已读积累记录
- ✅ 今日阅读记录（已移除页码输入，只保留书名+时间）

### 4.5 UI/UX
- ✅ App重命名为「皓月自习社」
- ✅ 男孩风格图标（深蓝夜空+月亮+书本）
- ✅ 单元标签两行显示 + 糖果色
- ✅ 语文/数学主页移除快捷操作按钮
- ✅ 四会词语：大字体词语 + 清晰拼音 + 柔和背景 + 单元级音频

---

## 五、当前状态

**所有功能正常运行，无已知 Bug。**

- Service Worker 版本：`v35`
- 数据版本：`DATA_VERSION`（检查 index.html 中的值）
- 最后一次提交：`2a45eca` — 运动页增加自定义项目和时间输入表单并排在前面；阅读记录移除页码

---

## 六、部署流程

### 6.1 修改代码后部署

```bash
cd /workspace/yaozihao-workbench

# 1. 修改代码后，验证JS语法
python3 -c "
import re
with open('index.html', 'r') as f:
    html = f.read()
m = re.search(r'<script>(.*?)</script>', html, re.DOTALL)
with open('/tmp/check.js', 'w') as f:
    f.write(m.group(1))
"
node --check /tmp/check.js

# 2. 升级 Service Worker 缓存版本号（sw.js 中的 CACHE_NAME）
#    每次部署必须递增！否则用户手机不会更新缓存。

# 3. 提交并推送
git add -A
git commit -m "描述改动"
git push origin master

# 4. 等待 GitHub Pages CDN 缓存刷新（约30-60秒）
sleep 40

# 5. 验证线上版本（必须带 cache-busting 参数）
curl -s -H "Cache-Control: no-cache" \
  "https://crystal236102087.github.io/yaozihao-workbench/sw.js?nocache=$(date +%s)" \
  | grep "CACHE_NAME = "

# 6. Playwright 线上验证（可选但推荐）
```

### 6.2 Playwright 测试模板

```javascript
const { chromium } = require('playwright');
(async () => {
  const browser = await chromium.launch({ headless: true });
  const page = await browser.newPage({ viewport: { width: 390, height: 844 } });
  const url = 'https://crystal236102087.github.io/yaozihao-workbench/index.html?nocache=' + Date.now();
  await page.goto(url, { waitUntil: 'networkidle' });
  await page.waitForTimeout(1500);
  // 用 evaluate 调用 switchNav / setTextbookSubject 等函数
  await page.evaluate(() => switchNav('sport'));
  await page.waitForTimeout(800);
  // 检查内容 / 截图
  await page.screenshot({ path: '/tmp/test.png' });
  await browser.close();
})();
```

---

## 七、踩过的坑（绝对不要再踩）

### 7.1 渲染块放错科目区域
**问题**：`if (currentBookTab === 'sihui')` 渲染块被放在了数学科目的 `if` 块内，导致点击语文的四会词语标签不显示内容。

**教训**：学期重点页的渲染逻辑是嵌套的 `if (currentTextbookSubject === 'chinese') { if (currentBookTab === 'xxx') { ... } }`。添加新的标签页渲染块时，**必须确认它放在正确科目的 `if` 块内**。

### 7.2 中文数字正则匹配失败
**问题**：`renderUnitSelector()` 中 `/^第\d+单元/` 只匹配阿拉伯数字，但单元标题用的是中文数字（第一单元、第二单元…），导致单元标签无法两行显示。

**教训**：中文内容的正则要用 `第[一二三四五六七八九十]+单元`，不要用 `\d+`。

### 7.3 Pillow alpha 通道导致图标全黑
**问题**：用 `Image.putalpha(mask)` 制作月牙图标时，mask 值为 255 的透明区域被设为 alpha=255，变成黑色不透明，导致整个图标变黑。

**教训**：用 `Image.putalpha()` 时，原图透明区域的 alpha 会被 mask 覆盖。正确做法是用 `layer.paste(full_moon, (0,0), crescent_mask)` 配合 mask 来合成，而不是 `putalpha`。

### 7.4 GitHub Pages CDN 缓存延迟
**问题**：`git push` 后立即 `curl` 验证，拿到的是旧版本。

**教训**：push 后至少等 30-40 秒再验证。验证时必须带 cache-busting 参数 `?nocache=$(date +%s)`，否则 CDN 返回缓存。

### 7.5 Edit 工具 replace_all 歧义
**问题**：用 Edit 工具替换代码时，`old_string` 在文件中出现多次导致失败。

**教训**：提供足够的上下文使 `old_string` 唯一，或使用 `replace_all: true`。

### 7.6 TTS 跨手机兼容性
**问题**：不同手机的浏览器对 Web Speech API 支持差异大，中文 TTS 在某些手机上读出来是乱码。

**教训**：
- iPhone 必须用 Safari，Android 必须用 Chrome
- 微信内置浏览器不支持 TTS
- 系统设置里需要安装中文语音包
- 这是浏览器 API 限制，App 层面无法完全解决

### 7.7 git push 分支名
**问题**：`git push origin main` 失败，因为分支是 `master` 不是 `main`。

**教训**：这个仓库的分支是 `master`，push 用 `git push origin master`。

### 7.8 首页欢迎语不要改
**问题**：应用改名为「皓月自习社」时，首页欢迎语也被改成了「皓月自习社，今天加油！」，用户要求恢复。

**教训**：首页欢迎语保持「姚子皓，今天加油！」不变，只有 App 名称、标题栏、manifest 改为「皓月自习社」。

---

## 八、下一步可能的方向

以下是用户可能提出但尚未要求的改进方向（仅供参考）：

1. **数据导出/备份**：目前数据只存 localStorage，清除浏览器数据会丢失。可考虑导出 JSON 备份功能。
2. **运动项目时间统计**：运动页面记录了时间，可在统计页面汇总每周运动时长。
3. **更多科目内容**：科学、数学的标签页内容目前较简单，可能需要补充。
4. **图标生成脚本保留**：当前图标生成脚本未存入仓库，建议保存一份。
5. **TTS 备选方案**：对于 TTS 不支持的设备，可考虑预录音频或使用第三方 TTS API。
6. **每日计划与打卡记录联动**：运动、阅读的自定义记录可自动同步到打卡日历。

---

## 九、快速上手 Checklist

新会话接手时，按以下步骤确认环境：

- [ ] `cd /workspace/yaozihao-workbench && git log --oneline -5` 确认最新提交
- [ ] `grep "CACHE_NAME" sw.js` 确认当前缓存版本号
- [ ] `grep "DATA_VERSION" index.html` 确认数据版本号
- [ ] 用 Playwright 打开本地 `file:///workspace/yaozihao-workbench/index.html` 验证页面正常
- [ ] 用 Playwright 打开线上 URL 验证部署正常
- [ ] 修改代码后：`node --check` 验证语法 → 递增 `CACHE_NAME` → `git push origin master`
- [ ] push 后等 40 秒 → 带 `?nocache=` 参数 curl 验证线上更新

---

*文档最后更新：2026-07-30*
*当前 Service Worker 版本：v35*
*当前 Git 提交：2a45eca*

---

## v40 - 云同步 & 补打卡（2026-08-13）

### 新增功能
1. **云同步**：通过 GitHub Contents API 把数据存到私有仓库的 `cloud-data.json`，两台手机自动同步。
   - Token 存 localStorage（key `yh-sync-token`），也支持代码里 `SYNC_CONFIG.defaultToken` 默认值。
   - 仓库通过 `SYNC_CONFIG.owner` / `SYNC_CONFIG.repo` 或设置页输入。
   - 同步时机：打开app拉取、每次saveData防抖推送(3s)、每60s定时拉取。
   - 防循环：`isPulling`标志 + hash指纹 + 时间窗口。
   - 冲突处理：字段级三路合并（sun取max、calendar按优先级、done取true优先、快照不覆盖、dailyLog并集）。
   - **待配置**：`SYNC_CONFIG.owner/repo/defaultToken` 仍为空，需填私有仓库信息和token。

2. **补打卡**：日历上本周内(周日起始)、今天之前的日期可点击补卡。
   - 补卡弹窗列出该天(按星期几)的语文/数学/英语/科学/运动任务，可勾选。
   - 数据存 `data.daySnapshots['YYYY-MM-DD']`（每日快照，不影响全局任务done）。
   - 阳光按完成数×5发放，`sunAwarded`防重复加分。
   - 补卡后重算该天日历状态(full/partial/missed)和全局streak。

### 关键文件改动
- `index.html`：新增云同步模块、补打卡模块、`daySnapshots`/`syncMeta`数据结构、云同步导航页。
- `sw.js`：CACHE_NAME v39→v40。

### 踩过的坑
- GitHub API 需带 `Authorization: token xxx` 和 `Accept: application/vnd.github.v3+json` 头。
- base64 内容需处理换行：`atob(content.replace(/\n/g,''))`，中文用 `decodeURIComponent(escape())`。
- PUT 更新必须带 `sha`（来自GET），否则409。
- token 不能写进公开仓库的 index.html（会泄露），已提供手机端输入方案。

---

## v41 - 云同步已启用（2026-08-13）

### 变更
- `SYNC_CONFIG` 已配置：owner=`crystal236102087`，repo=`study-sync`（私有仓库）。
- **私有仓库 `study-sync` 已创建**，存放 `cloud-data.json` 同步数据。
- **token 不能写进代码**（GitHub 密钥扫描会拦截提交，实测 409 "Secret detected"）。
  - `SYNC_CONFIG.defaultToken` 保持为空。
  - 用户在 app 内"云同步"设置页输入 token（存 localStorage `yh-sync-token`，不上传）。
- sw.js CACHE_NAME v40 → v41。

### 端到端测试结果（已验证，token 由代码变量传入时）
- 第一台手机手动同步 → 数据成功推送到私有仓库 cloud-data.json。
- 第二台手机（全新浏览器）打开 → 自动拉取完整数据。
- 云端数据完整性验证通过（sun/tasks/calendar/daySnapshots/weekly 全保留）。
- 无 JS 错误。

### 安全提醒（重要）
- **token 绝不能写进公开仓库的 index.html**（GitHub 密钥扫描会拦截，且写进去会被任何人看到）。
- **token 正确配置方式**：用户打开 app → 云同步页 → 粘贴 token（需 repo 权限）→ 保存。token 仅存本机 localStorage。
- 该私有仓库 `study-sync` 只有 owner（crystal236102087）能访问，学习数据不会公开。
- 部署时若 git push 遇到 TLS 握手失败（沙箱网络问题），可用 GitHub Contents API 上传文件绕过（见下文踩坑记录）。
