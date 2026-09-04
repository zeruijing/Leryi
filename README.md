# 前任纪念站 · deploy-memory-site

> 把一段微信聊天回忆，做成可以部署的「前任纪念站」——卫星地图、时间线、相册、语音克隆、聊天人设，一键复刻 ta 的样子。
>
> 灵感与致敬：[therealXiaomanChu/ex-skill](https://github.com/therealXiaomanChu/ex-skill) —— 把前任蒸馏成 AI Skill，用 ta 的方式跟你说话。

## 这是什么

拿到一份 WeFlow 微信导出（jsonl + media），经过 **数据挖掘 → 视觉识别 → 语音克隆 → 网页生成**，产出可部署的回忆网站，并训练出 ta 的声音模型。把整个站点丢到 GitHub Pages 就能访问。

## 功能特性

- **主站**（main.html）—— 卫星地图 + 时间线 + 对话框 + 「第一次」+ 梗与昵称 + 相册 + 语音 + 聊天模块。
- **聊天人设**（chat.html）—— 把 ta 的人设 prompt 内嵌成 AI 对话，API Key 只存浏览器 localStorage，绝不写进源码。
- **视觉识别档案**（vision_all / vision_her / vision_check / vision_approve / vision_emoji.html）—— 照片识别、审批与 emoji 档案页。
- **语音克隆** —— GPT-SoVITS v2Pro 训练 ta 的声音，支持零训练克隆 + 完整训练两套路线。

## 怎么用

### 前提

- WeFlow 微信导出：`jsonl`（聊天记录）+ `media`（图片/表情/视频/语音）。
- 可选：DeepSeek API Key（视觉识别 + 聊天模块）。

### 流程

1. **数据挖掘**：逐行解析 jsonl，正则提取 `[位置]` 定位与 media 索引，按月切片交给子代理逐行精读，输出结构化报告。
2. **视觉识别**：调 DeepSeek 批量识别照片，结合前后约 7 条聊天消息推断「当时在干什么」（无依据就写「无法确定」，绝不编造）。
3. **语音**：FunASR 转写 → pyworld 男女声粗筛 → GPT-SoVITS 克隆训练 → 合成后回听验证。
4. **网页**：生成 `main.html`（主站）、相册、语音、聊天模块；站点入口由使用者自行指定。
5. **部署**：整站丢到 GitHub Pages，主站 `main.html` 为入口（若平台要求默认页为 `index.html`，则将其重命名）。

## 项目结构

```
memory-map/
  main.html               # 主站（地图/时间线/对话框/第一次/梗与昵称/相册/语音/聊天）
  chat.html               # 聊天人设模块
  vision_all.html         # 识别总览
  vision_her.html         # 识别档案
  vision_check.html       # 待核验
  vision_approve.html     # 已通过
  vision_emoji.html       # emoji 档案
  photos/                 # 精选照片
  leaflet.js / leaflet.css # 地图库（底图瓦片：自行解决）
  *.wav                   # 语音
```

## 部署检查清单

1. 删除或替换所有真实 API Key（聊天模块从 localStorage 读）。
2. 隐私：私聊原数据（media 与 jsonl）**不要上传公开仓库**，照片只用精选集。
3. GitHub Pages：文件夹结构就是站点根，入口以 `main.html` 为准（如需默认访问，可将其改名为 `index.html`）。
4. 音频与图片用相对路径，`file://` 与 `http(s)` 都能打开。

## 开源协议与合规声明

### 许可证
- 本仓库代码以 [MIT License](LICENSE) 发布（Copyright (c) 2026 zerujing）：可自由使用、修改、商用，需保留版权声明。

### 引用与衍生
- 本项目为独立开发，仅**灵感**参考 [therealXiaomanChu/ex-skill](https://github.com/therealXiaomanChu/ex-skill)（对方仓库带 `LICENSE` 文件，**引用其任何内容前请先核实并遵守其协议**；不确定时保持「借鉴思路、独立编写」，不要整段复制）。

### 生成站点内第三方组件许可

| 组件 | 许可证 | 要求 |
|---|---|---|
| Leaflet 1.9.4 | BSD-2-Clause | 保留版权声明 |
| ECharts | Apache-2.0 | 保留 LICENSE / NOTICE |
| GPT-SoVITS | MIT | 保留版权声明 |
| FunASR | MIT | 保留版权声明 |
| 底图瓦片（卫星/路网） | 自行解决 | 本技能不指定任何瓦片源；请使用合规来源并按源署名 |

### 隐私与法律风险（重要）
- 聊天记录、照片、语音合成均涉及当事人**个人信息、肖像与声音权益**（《民法典》第 1019 条、第 1023 条，《个人信息保护法》）。**禁止将原始 jsonl / media / 语音克隆上传公开仓库或公网展示**，仅供个人纪念使用。
- 若确需公开，先取得当事人书面同意，并对照片、语音做脱敏处理。
- 第三方服务（DeepSeek API 等）的调用请遵守各自平台条款。

### 免责声明
- 本 skill 仅供个人学习与技术交流，请勿用于骚扰、伪造身份、恶意传播等场景；因不当使用产生的一切后果由使用者自行承担。
- **素材权利**：聊天记录、照片、语音等数据均由使用者自行提供，本项目不拥有任何授权；使用者须自行确认处理行为合法（含已取得当事人同意）。
- **生成式 AI 内容**：语音克隆、AI 人设对话等涉及生成式 AI 内容，使用者应自行按相关规定进行标识，作者不对生成内容的真实性负责。
- **平台合规**：部署到 GitHub Pages 等平台前，请自行确认不违反平台服务条款与当地法律法规；因部署产生的投诉、下架或损失由使用者承担。
- **不承担保证**：本 skill 按「现状」（as-is）提供（见 LICENSE），作者不对其可用性与准确性负责。

## 后续计划（待补充）

以下功能暂未实现，先占位，后续再写：

- [ ] 底图瓦片源：方案自行解决（待实现）
- [ ] 入口页：去掉点阵动画入口后，站点入口方案待设计（待实现）
- [ ] 移动端适配与整体打磨（待实现）

## 结束语

- 仅此纪念Shenzhen high school的一位女孩
<!-- 结束语由作者亲自写 —— 此处留白 -->
