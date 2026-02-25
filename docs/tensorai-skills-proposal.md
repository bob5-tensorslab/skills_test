# TensorAI Claude Code Skills 设计方案

## 1. 目标与概述
为了吸引更多用户注册 TensorAI 平台，我们将设计两个标准化的 Claude Code Skills（图片生成和视频生成）。这两个 Skills 会基于 [HuggingFace Skills](https://github.com/huggingface/skills.git) 的标准进行构建。设计的核心理念是：**优先解决用户最高频、最痛点的 8 大图像和 4 大视频视听需求，并在终端里通过自然语言提供“傻瓜式”的极佳体验。Agent 的核心作用主要体现在：1. 针对用户的简短意图进行丰富和改写 prompt； 2. 调用正确的工具流并执行。**

## 2. API Key 获取与环境配置体验
**用户痛点**：用户并不知道如何配置认证信息。
**Skill 设计**：
1. **自动拦截与提示**：当用户首次呼叫该 Skill 时，脚本会先检查系统环境变量 `TENSORAI_API_KEY`。
2. **友好的引导文案**：如果未设置，温和地输出：
   ```text
   您好！要生成高质量的内容，您需要先进行简单的配置：
   1. 访问 https://tensorai.tensorslab.com/ 登录并订阅。
   2. 在控制台中获取您的专属 API Key。
   3. 将其保存为环境变量：
      - Windows (PowerShell): $env:TENSORAI_API_KEY="您的Key"
      - Mac/Linux: export TENSORAI_API_KEY="您的Key"
   ```

## 3. 图片生成 Skill (tensorai-image) 的 8 大高频核心场景与通用场景

依据大语言模型在生图交互中的强项，Agent 的核心处理流程统一化为：**提取实体 -> 丰富 Prompt -> 分配参数 -> 执行 API 生成并本地输出**。以下是我们重点优化的 8 个高频场景，以及通用的兜底场景：

### 【高频场景 1】：智能变宽高比 (Aspect Ratio Changing / Outpainting 模拟)
- **用户输入**：“把这张竖屏的壁纸变成 16:9 的横屏”
- **Agent 处理**：提取本地图片绝对路径，指定 `resolution="16:9"`，并**利用扩写 Prompt 补全边缘场景描述**。

### 【高频场景 2】：高清无损放大至 4K (Upscaling)
- **用户输入**：“帮我把这张模糊的草图放大到 4K 级别的清晰度”
- **Agent 处理**：针对原图选择最高质量模型（`seedreamv4.5`），设置分辨率 `resolution="4K"`，**并丰富原图的细节刻画 prompt**。

### 【高频场景 3】：智能局部重绘与替换 (Inpainting/Replacement)
- **用户输入**：“保留这张图的背景，把图里的苹果换成一个赛博朋克风格的西瓜”
- **Agent 处理**：读取本地图片，将用户的简单替换需求**转化为丰富的、包含材质和光影的赛博朋克风格 Prompt**。

### 【高频场景 4】：智能背景消除与一键替换 (Background Removal/Replacement)
- **用户输入**：“把这人的背景去掉，换成一个热带海滩的风景”
- **Agent 处理**：读取图片，**重新将主体特征与新的热带海滩进行高质量 Prompt 融合创作**，执行改图流程。

### 【高频场景 5】：元素精准擦除 (Object Removal)
- **用户输入**：“帮我擦掉这张风景图右上角的那根电线杆”
- **Agent 处理**：利用模型强大的理解能力与原图参考，辅以精准的负面提示词提取，完成自然擦除的生成流程。

### 【高频场景 6】：快速线稿/草图上色渲染 (Sketch to Real/Render)
- **用户输入**：“参考我画的线稿 `./sketch.png`，帮我渲染成 3D 游戏高模材质”
- **Agent 处理**：线稿作为 `sourceImage` 上传，**Agent 负责将“3D材质”扩写转化为大量极具质感的提示词（如 UE5, octane render, 8k）**。

### 【高频场景 7】：风格极速转换 (Style Transfer)
- **用户输入**：“把这张真实的宠物照片转换成皮克斯 3D 动画风格”
- **Agent 处理**：原图传入，**Agent 负责精准扩写目标风格（如 Pixar 3D animation, soft lighting, expressive eyes）**。

### 【高频场景 8】：多版本批量推演 (Batch Idea Generation)
- **用户输入**：“给我设计 3 个不同配色的科技公司 Logo”
- **Agent 处理**：**丰富对于“科技感 Logo”的多样性描述**，自动配置 `batchsize=3` 并批处理下载。

### 【通用场景 1】：通用文生图 (Generic Text-to-Image)
- **用户输入**：“画一个在月球上吃热狗的宇航员”
- **Agent 处理**：**接管 Prompt 扩写任务，丰富环境光、材质、构图等细节描述**，随后调用最佳模型生成。

### 【通用场景 2】：通用图片编辑与合成 (Generic Image Editing & Synthesis)
- **用户输入**：“把 `./cat.png` 和 `./space.png` 结合一下，做成一张海报”
- **Agent 处理**：处理多张本地图片路径数组，将其作为 `sourceImage` 传入。同时，**极大程度地丰富提示词以指导模型如何将这两张图的内容和谐共融在同一个画面内**。

### 3.3 交付体验
- **本地直接保存并展示**：生成完成后，所有的图都会安静地躺在 `./tensorai_output/` 下供用户即拿即用。

## 4. 视频生成 Skill (tensorai-video) 的四大高频痛点场景设计

视频生成的核心门槛在于“不知道如何描述运镜”、“不会控制动态幅度”、“难以忍受长久的黑盒等待”。因此针对视频，Agent 将承担“剧组导演”和“制片助理”的双重身份。

### 【高频场景 1】：老照片/静态图唤醒与微动 (Subtle Image-to-Video Animation)
- **用户痛点**：希望让某张人物照片或风景图有微微的呼吸感或动态，但不希望图像崩坏。
- **用户输入**：“让这张人物合影 `./family.jpg` 动起来，只需轻微点头微笑”
- **Agent 处理**：提取本地图片绝对路径，将简单的输入转化为极精细的 Prompt（例如 `"cinematic breathing, subtle gentle smile on faces, gentle nodding, hyper-realistic, keeping original features intact"`）。

### 【高频场景 2】：大范围运镜与时空穿梭 (Dynamic Cinematic Text-to-Video)
- **用户痛点**：文字描述干瘪，难以营造电影大片的镜头移动感。
- **用户输入**：“做一段 10 秒钟横屏的宇宙飞船穿梭星际的视频”
- **Agent 处理**：敏锐地提取 `duration=10`, `ratio="16:9"`。同时接管运镜，**大幅扩写**：“*Breathtaking cinematic wide shot, FPV drone flying rapidly past glowing neon nebulas, a sleek advanced spacecraft soaring into the cosmic depth, hyper-speed trails, dramatic volumetric lighting, 8k resolution, Unreal Engine 5 render...*”。

### 【高频场景 3】：短视频口播/数字人场景 (Talking Avatar / Portrait Video)
- **用户痛点**：需要大量竖屏的数字人口播或角色讲解短视频素材。
- **用户输入**：“用这张竖版头像 `./avatar.png` 和演讲稿 `./script.txt`，做一个女主持人在演播室讲解的视频”
- **Agent 处理**：识别竖屏调用 `ratio="9:16"`。**优先提取用户提供的文案内容（或本地文案文件内容）作为口播输入**；如果用户未提供明确讲述方案，Agent 将自动围绕图文主题生成口播台词。传入原图与台词，激活相关数字人功能（若所选模型支持，如 `generate_audio=1`），并为其丰富场景 Prompt（如：“*professional news anchor broadcasting in a modern studio, steady camera, realistic facial expressions, crisp speech*”）。

### 【高频场景 4】：商业产品动效展陈 (Product Highlight Animation)
- **用户痛点**：电商和新媒体需要大量商品的高级转场与光影动画，而非干瘪的平摇。
- **用户输入**：“给这个香水瓶 `./perfume.png` 做个产品宣传广告，高逼格一点”
- **Agent 处理**：调用最高画质的 `seedancev15pro`。极大幅度地重写光影与动态 Prompt，如：“*macro slow-motion shot, professional studio lighting, caustic light reflections passing through a glass perfume bottle, slow spin, elegant luxurious vibe...*”。

### 【通用兜底场景】：万能生成与时长控制
- 无明确特殊要求的输入下，Agent 默认为 5s 短视频以节约积分并加快展示，自动选择最佳模型，并且自动根据“宽窄概念”适配 ratio (`16:9` 或 `9:16`)。

### 4.3 彻底消除等待焦虑与“傻瓜式”极简交互
- **终端心跳防假死**：视频渲染长达数分钟，底层脚本必须内建 **终端进度条/心跳日志**（如 "🚀 正在渲染电影级大片，已耗时 120 秒，请稍安勿躁..."），让用户实时感知程序存活。
- **直接的本地交付**：直到最后无缝自动下载至本地 `./tensorai_output/` 目录，输出：“🎉 您的视频处理完毕！已存放于 `./tensorai_output/cosmic_ship.mp4`”。
- **同理心失败降级**：余额不足、尺寸不符等冷冰冰的 HTTP 报错，需要在脚本内通过 JSON 逆解析，转换为友好的中文提示（“亲，积分用完啦，请前往...充值”）。

## 5. 接入 Marketplace 与分发机制
为了允许用户如同 HuggingFace 规范中那样轻松添加到 Marketplace：
- 在根目录下配置 `.claude-plugin/marketplace.json`。
- 将重点支持的功能（8大图像魔法，4大视频视效引擎）提炼为极具科技感与解决痛点吸引眼球的宣传语，提升工具的自发裂变下载。
