---
name: deploy-memory-site
description: 把一段微信聊天回忆做成可部署网页的完整流程：数据挖掘、视觉识别、语音克隆、地图/相册/聊天人设模块，一键复刻"前任纪念站"。
user-invocable: true
---

# 前任纪念站 · 一键部署手册

目标：拿到一份 WeFlow 微信导出（jsonl + media），产出可部署的回忆网站（点阵动画入口 + 卫星地图 + 时间线 + 相册 + 语音 + 聊天人设模块），并训练对方的语音模型。

## 0. 产物结构（memory-map 文件夹）
- index.html —— 点阵浮动动画入口（可自定义名字，点击进入主站）
- main.html —— 主站（地图/时间线/对话框/第一次/梗与昵称/相册/语音/聊天模块）
- vision_all / vision_her / vision_check / vision_approve / vision_emoji.html —— 识别档案与审批页
- photos/ —— 精选照片；leaflet.js/css —— 地图库；*.wav —— 语音

## 1. 数据挖掘（聊天记录）
1. jsonl 每行一个 JSON（_type: message / header / member）。字段：timestamp、accountName（听说=自己、眠=对方）、content。
2. 时间换算：timestamp 是 UTC，北京时间 = 加 8 小时。
3. 挖内容：逐行 json.loads，正则 [位置] 名称 (lat,lng) 提取微信定位（权威坐标）；media/(images|emojis|videos)/xxx 提取媒体索引（文件→时间→发送者）。
4. 大语料精读：按月切片（每片不超过 115KB），交给子代理逐行通读，输出结构化报告（places/events/loveQuotes/conflicts/梗/个人信息/plans）。禁止子代理用 grep 代替通读。

## 2. 视觉识别（DeepSeek API）
- 模型：deepseek-v4-flash-vision-exp；图片最多 384 token；开思考更细、关思考（thinking disabled）省一半。
- 单张成本约 0.2-0.4 分（高峰价）；全量几千张也就几块钱。
- 批量脚本见 _build/vision_worker.py（断点续传、429/402 重试）；可分片交给多个子代理并行。
- 结果入库 vision_results.json：{文件名:{dt,sender,rel,content,reasoning}}。
- 上下文推断：为每张照片取发送前后约 7 条消息，连同识别描述给子代理写"当时在干什么"，铁律：无依据写"无法从上下文推断"，绝不编造。
## 3. 语音
1. 转写：GPT-SoVITS 自带 FunASR（tools/asr/models 本地模型，device=cuda）。
2. 男女声粗筛：pyworld dio 加 stonemask 取中位 F0，175Hz 以上判女声（唱歌和音域边缘不可靠，需人工听）。
3. 克隆训练（GPT-SoVITS v2Pro，参考 logs/LERYI 全流程）：
   - 素材：对方语音 2-4 分钟起步，越多越好；FunASR 生成 .list（wav路径|name|zh|文本）。
   - 预处理（cwd=GPT-SoVITS 根目录，设环境变量）：inp_text、inp_wav_dir、exp_name、opt_dir=logs/名、i_part=0、all_parts=1、CUDA_VISIBLE_DEVICES=0、is_half=True（必须大写 True，小写 true 会 NameError）、version=v2Pro、bert/cnhubert/sv 路径；依次跑 prepare_datasets 里的 1-get-text.py、2-get-hubert-wav32k.py、2-get-sv.py、3-get-semantic.py。
   - 坑一：3-get-semantic 的 s2config 里 model 不能带 version 字段，否则 SynthesizerTrn 双重传参报错。
   - 坑二：单卡产物是 2-name2text-0.txt 和 6-name2semantic-0.tsv，要改名去掉 -0。
   - SoVITS 训练：python GPT_SoVITS/s2_train.py -c 配置json（克隆 LERYI config.json，改 exp_dir/name/save_weight_dir/epochs）。
   - GPT 训练：python GPT_SoVITS/s1_train.py -c 平铺 yaml（train_semantic_path、train_phoneme_path、output_dir、pretrained_s1=s1v3.ckpt）。
   - 推理：api_v2.py 起服务，/set_gpt_weights 和 /set_sovits_weights 换权重，再 /tts（GET 只返回第一段！要用 streaming_mode=2 全量流，并重建 WAV 头）。
   - 零训练克隆：s1v3.ckpt 加 v2Pro/s2Gv2Pro.pth 加对方 3 段参考音即可。
4. 合成后验证：FunASR 回听确认内容，pyworld 音高确认女声。

## 4. 网页
- 地图：Leaflet 加高德瓦片（style=6 卫星、style=8 路网标注，路网可开关）；坐标优先用聊天 [位置] 消息，自标点注明"坐标约"；点与点之间细线连接；默认视野框住核心区域。
- 相册：瀑布流（columns 多列，图片 width:auto 加 max-height 300，禁止 object-fit:cover 裁切）；点图放大（滚轮每格 1.1 倍、上限 6 倍、不做拖拽）；点文字弹详情（完整描述、时间、聊天上下文、推断，推断不可信标红"此来源不可信"）。
- 聊天模块：人设 prompt 直接内嵌 SKILL.md 全文；API Key 只存 localStorage，绝不写进源码（防 GitHub 泄露）；模型用 deepseek-v4-flash。
- 入口页：index.html 放用户自带的点阵动画（只改名字，其余分毫不动），点击跳转 main.html。

## 5. 部署检查清单
1. 删除或替换所有真实 API Key（聊天模块从 localStorage 读）。
2. 隐私：私聊原数据（media 和 jsonl）不要上传公开仓库，照片只用精选集。
3. GitHub Pages：文件夹结构就是站点根，index.html 为入口。
4. 音频与图片用相对路径，file:// 与 http(s) 都能打开。
5. 常见坑：层层嵌入字符串会吃掉反斜杠（用 chr(92) 构造）；PHOTOS 数组漏逗号会让 forEach 里 undefined 崩掉整段脚本；数据块必须包在 script 标签里。

## 6. 参考产物（本次项目）
- 子代理报告：_build/reports/slice_*.json
- 识别结果：vision_results.json、emoji_results.json
- 照片详情库：her_details.json
- 训练产物：logs/TINGTING（已归档到 A 盘 LTT）

