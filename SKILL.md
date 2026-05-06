---
name: wechat-article-publisher
description: 微信公众号文章完整管理工具；支持提取文章并转换为 Markdown、创建草稿、发布文章、生成封面图片等全流程功能
---

# 微信公众号文章管理

## 任务目标
- 本 Skill 用于：微信公众号文章的完整管理流程
- 能力包含：
  - **智能创作**：基于参考文章改写、基于标题创作、直接发布
  - **提取转换**：从 mp.weixin.qq.com 链接提取文章并转换为 Markdown 格式
  - **创建草稿**：将内容创建为公众号草稿
  - **发布文章**：发布草稿到公众号
  - **生成封面**：自动生成文章封面图片（默认千问，支持豆包）
  - **素材管理**：上传图片等素材到微信服务器
- 触发条件：用户需要创作、提取、转换、发布或管理公众号文章
- **默认配置**：
  - 文章字数：1200-1500字（默认要求）
  - 配图数量：3张横屏图片（16:9）
  - 排版主题：根据文章内容自动选择
  - 写作风格：去 AI 化，详见 `HUMANIZE_GUIDE.md`
  - 标题策略：生成标题时**必须**参考 `references/viral_titles_50.md`（50 个爆款标题范例 + 背后逻辑），按"身份/场景锚点 + 量化结果"结构产出 3 个候选标题，并在每个候选后标注命中的爆款逻辑

## 🚨 图片生成核心规范（强制）

**所有图片生成（封面图 + 文中配图）必须按下面的"五要素结构"构造 prompt，禁止使用空泛描述。** 违反此规范会导致图文不贴切、出现"AI 味"或水印感。

### 五要素结构（按顺序拼接，中文逗号分隔）

1. **核心意象**：从文章前 200 字中提取 2-4 个最具画面感的具象名词（人/物/场景/动作），不要抽象概念。
2. **情绪基调**：恰好 1 个词，从下面挑选——`温暖治愈` / `专业冷静` / `紧张焦虑` / `轻松愉悦` / `怀旧沉思` / `励志昂扬` / `神秘高级`。
3. **画面风格**：按文章主题从下表挑选——
   - 鸡汤情感 → 扁平插画，马卡龙色系
   - 科技数码 → 深蓝紫渐变，微缩等距风格
   - 健康养生 → 清新水彩，自然光
   - 财经商业 → 极简商务，金蓝撞色
   - 文艺生活 → 胶片质感，柔焦颗粒
   - 知识科普 → 信息图风格，几何图形
   - 历史人文 → 复古版画，做旧纸张
   - 美食探店 → 食物特写摄影感，浅景深
4. **构图与镜头**：固定为 `16:9 横向构图，主体居中偏左，留白用于标题文字`。
5. **负面提示**：固定追加 `画面中不要出现任何文字、字母、汉字、水印、logo、签名`。

### 文中 3 张配图必须按"故事弧线"分别构造

| 序号 | 画面定位 | 应包含的具象元素 |
| --- | --- | --- |
| 第 1 张 | **痛点画面** | 文章开头描述的困境/焦虑/失败场景的具象化 |
| 第 2 张 | **转折画面** | 解决方案出现的关键时刻/工具/方法的具象化 |
| 第 3 张 | **升华画面** | 文章结尾呈现的理想状态/成果/未来感的具象化 |

**3 张配图 prompt 必须互不相同**，禁止复用同一个 prompt。

### ❌ 禁止使用的空泛 prompt

下面这些 prompt 一律不准出现，发现立即按五要素重写：

- ❌ `科技风格插画`
- ❌ `为文章《XXX》生成封面`
- ❌ `AI 提升工作效率主题插画`
- ❌ `温暖治愈风格的封面`
- ❌ 任何只含"风格 + 主题"两要素的描述

### ✅ 合格 prompt 示例（封面图）

文章主题："上班族用 AI 工具节省下班时间"
```
深夜办公室仅亮着的台式电脑屏幕，桌面散落咖啡杯与便签，窗外城市灯火，
专业冷静，
深蓝紫渐变，微缩等距风格，
16:9 横向构图，主体居中偏左，留白用于标题文字，
画面中不要出现任何文字、字母、汉字、水印、logo、签名
```

## 前置准备
- 凭证准备：
  - 微信公众号凭证：需要 AppID 和 AppSecret
    - 登录微信开发者平台 https://mp.weixin.qq.com
    - 进入"开发"→"基本配置"页面
    - 获取 AppID (开发者ID) 和 AppSecret (开发者密码)
    - 启用/重置AppSecret时需要管理员扫码验证
  - 图像生成API凭证（用于生成封面图片）：
    - **千问（默认，对 watermark=False 更稳定）**：需要 DashScope API Key
      - 访问阿里云控制台 https://dashscope.console.aliyun.com/
      - 获取 API Key
      - 设置环境变量 `DASHSCOPE_API_KEY`
    - **豆包（可选）**：需要 API Key
      - 访问火山引擎控制台 https://console.volc.com/iam/keylist
      - 获取 API Key
      - 确保账号已开通豆包图像生成服务权限
      - 通过 `--provider doubao` 或环境变量 `IMAGE_PROVIDER=doubao` 切换
- 依赖说明：
  - Python 脚本：仅 `generate_cover.py` 依赖 `requests` 和 `pillow`（脚本内置 PEP 723 元数据，可用 `uv run` 自动安装），其它脚本无额外依赖
  - Node.js 转换器：需要安装依赖（首次使用时）
    ```bash
    cd reader && npm install
    ```

## 操作步骤

### 流程 0：智能文章创作工作流（推荐）

当用户需要快速创作并发布文章时，使用智能工作流：

#### 🎯 第一步：识别任务类型（开始前必做）

不同任务类型走不同分支，**不要默认走完整流程**：

| 类型 | 用户输入特征 | 应走的分支 |
| --- | --- | --- |
| A | 给了完整 brief（标题 + 受众 + 关键信息） | 完整流程 0（含搜索 + 三遍审校） |
| B | 只给了一个主题 / 方向，没有标题 | **先做选题讨论 → 再走完整流程**（见下文） |
| C | 给了参考文章 URL，要求改写 | 模式 1（基于参考文章改写） |
| D | 给了完整文章稿，要求审校 / 降 AI 味 | **直接进入三遍审校**（见 [references/三遍审校清单.md](references/三遍审校清单.md)），跳过创作环节 |
| E | 只是问问题、聊功能 | 直接回答，不进入创作流程 |

#### 🧭 选题讨论（任务类型 B 必做，禁止直接动笔）

用户只给了主题方向时，**先输出 3-4 个选题方向给用户挑**，不要自己拍板：

每个选题方向包含：
- 候选标题（参考 [references/viral_titles_50.md](references/viral_titles_50.md) 的爆款逻辑）
- 核心角度（一句话）
- 预估工作量（搜索成本 / 字数）
- 优势 / 劣势
- 大纲（3-5 个小标题）

**等用户选定后才进入下一步。**

#### 🔍 调研规则（强制）

**何时必须调用搜索工具（web_fetch）：**
- 涉及 2024-2025 年的新产品 / 新政策 / 新数据
- 涉及具体的统计数字、市场规模、调研报告
- 涉及不确定的技术细节（API、版本号、配置）
- 涉及具体的人名 / 公司动态 / 时间线

**信息源优先级：**
1. 官方来源（官网、公告、白皮书）
2. 主流科技 / 行业媒体
3. 社区讨论（Reddit / HN / Product Hunt / 知乎高赞）
4. 百度结果（**谨慎使用**，需交叉验证）

**铁律：绝不编造数据和案例。** 搜不到的事实 → 删掉相关表述，或在文中明确说"暂未找到公开数据"。

#### 🎯 工作流程控制

**两阶段发布流程：**
1. **预览阶段**：如果用户没有明确说"发布到草稿箱"，先输出文章内容供预览
2. **发布阶段**：用户确认后再发布到草稿箱

**示例对话：**
```
用户: 写一篇关于"AI提升工作效率"的文章
智能体: 
[输出文章内容]
---
📝 文章已生成！是否需要发布到草稿箱？

用户: 发布
智能体: 
✅ 正在发布到草稿箱...
✅ 完成！草稿已创建，media_id: xxx
```

**直接发布模式：**
```
用户: 写一篇关于"AI提升工作效率"的文章并发布到草稿箱
智能体: 
1. 搜索资料...
2. 创作文章...
3. 生成封面...
4. 上传到草稿箱...
✅ 完成！草稿已创建，media_id: xxx
```

#### 🖼️ 文章内插图片配置

**配图生成规则**：
- **仅生成文章**（预览模式）：不生成文中配图，只输出文字内容
- **发布到公众号**（发布模式）：必须生成文中配图，默认 3 张

**示例对话**：

**场景 1：仅生成文章（不生成配图）**
```
用户: 写一篇关于"AI提升工作效率"的文章
智能体:
1. 搜索资料并创作文章
2. 输出纯文字内容（不含配图）
---
📝 文章已生成！是否需要发布到草稿箱？
```

**场景 2：发布到公众号（必须生成配图）**
```
用户: 写一篇关于"AI提升工作效率"的文章并发布到草稿箱
智能体:
1. 搜索资料并创作文章
2. 生成 3 张文中配图（横屏 16:9）
3. 上传配图到公众号
4. 生成封面图
5. 创建草稿
✅ 完成！草稿已创建，media_id: xxx
```

**配图规则**：
- 配图数量：默认 3 张（可在 `.env` 中配置 `ARTICLE_IMAGE_COUNT`）
- 配图位置：在文章的主要段落结尾插入
- 图片方向：横屏（16:9 比例）
- 图片提示词：**严格按照本文档『🚨 图片生成核心规范』中的五要素结构生成**，3 张图分别对应"痛点画面 / 转折画面 / 升华画面"

#### 模式 1：基于参考文章改写

用户提供参考文章 URL，智能体自动完成：提取 → 改写 → 生成封面 → 创建草稿

**操作步骤：**
1. 提取参考文章内容（使用 `extract_to_markdown.py`）
2. 智能体改写文章内容
3. 生成封面图片（使用 `generate_cover.py`）
4. 上传封面到微信（使用 `upload_material.py`）
5. 创建草稿（使用 `create_draft.py`）

**示例对话：**
```
用户: 帮我改写这篇文章并发布到草稿箱：https://mp.weixin.qq.com/s/xxx
智能体: 
1. 提取原文内容...
2. 改写文章...
3. 生成封面...
4. 上传到草稿箱...
✅ 完成！草稿已创建，media_id: xxx
```

#### 模式 2：基于标题创作

用户提供标题，智能体自动完成：搜索资料 → 创作文章 → 生成封面 → 创建草稿

**操作步骤：**
1. 使用 `web_fetch` 搜索相关资料
2. 智能体基于资料创作文章
3. 生成封面图片
4. 上传封面到微信
5. 创建草稿

**示例对话：**
```
用户: 写一篇关于"AI如何提升工作效率"的文章并发布到草稿箱
智能体:
1. 搜索相关资料...
2. 创作文章...
3. 生成封面...
4. 上传到草稿箱...
✅ 完成！草稿已创建，media_id: xxx
```

#### 模式 3：直接发布

用户提供完整内容，智能体自动完成：生成封面 → 创建草稿

**操作步骤：**
1. 生成封面图片
2. 上传封面到微信
3. 创建草稿

**可选参数：**
- `--provider doubao|qwen` - 选择图像生成引擎
- `--author "作者名"` - 指定作者
- `--digest "摘要"` - 指定摘要
- `--no-cover` - 不生成封面

### 流程 A：提取文章并转换为 Markdown

当用户提供微信公众号文章链接，需要提取内容并转换为 Markdown 格式时：

1. **提取并转换文章**
   - 调用 `scripts/extract_to_markdown.py` 提取并转换文章
   - 参数说明：
     - `url`: 文章链接（必需，mp.weixin.qq.com）
     - `-o, --output`: 保存 Markdown 文件路径（可选）
     - `--json`: 输出 JSON 格式（可选）
   - 示例：
     ```bash
     # 输出 Markdown 到终端
     python3 scripts/extract_to_markdown.py "https://mp.weixin.qq.com/s?__biz=..."
     
     # 保存为文件
     python3 scripts/extract_to_markdown.py "https://mp.weixin.qq.com/s?__biz=..." -o article.md
     
     # 输出 JSON 格式（含元数据）
     python3 scripts/extract_to_markdown.py "https://mp.weixin.qq.com/s?__biz=..." --json
     ```
   - 返回数据包含：
     - 文章标题、作者、发布时间
     - Markdown 格式的内容
     - 封面图 URL、原文链接
     - 完整的 Markdown 文档（含元数据）

2. **后续处理**
   - 可以基于 Markdown 内容进行编辑、分析
   - 或转换回 HTML 后使用流程 B 发布

### 流程 B：发布带封面的文章到公众号

1. **生成文章内容**
   - 智能体根据主题创作文章标题、正文内容
   - **标题创作**：先打开 [references/viral_titles_50.md](references/viral_titles_50.md)，按文章主题命中的类别（痛点 / 热点 / 认知冲突 / 细分人群 / 数据强化）挑 3-5 条范例仿写，输出 3 个候选标题供用户挑选，每个候选标注命中的爆款逻辑
   - **写作风格**：参考 [HUMANIZE_GUIDE.md](HUMANIZE_GUIDE.md) 的去 AI 化规则
   - **三遍审校（强制，初稿写完后必做）**：按 [references/三遍审校清单.md](references/三遍审校清单.md) 顺序过——
     - 第一遍 内容审校：事实 / 逻辑 / 结构（绝不编造数据）
     - 第二遍 风格审校：删 AI 套话 + 拆句式 + 替换书面词 + 加态度（≥ 10 处个人感受）
     - 第三遍 细节打磨：句长 ≤ 30 字 / 段落 3-5 行 / 全角标点 / 引号「」/ 中英文混排空格
     - **未通过审校清单的"终极检查"，不得进入步骤 2 生成封面**
   - 生成摘要和作者信息（可选）
   - 将正文内容转换为 HTML 格式
   - 参考 [references/article-format.md](references/article-format.md) 中的格式规范

2. **生成封面图片**
   - 智能体根据文章内容生成封面描述（prompt）
   - 调用 `scripts/generate_cover.py` 生成封面图片
   - 参数说明：
     - `--prompt`: 图片提示词，描述封面内容（必需）
     - `--provider`: 图像生成服务提供商（可选，可由环境变量 `IMAGE_PROVIDER` 覆盖默认值）
       - `qwen` - 千问（通义万相，**默认**）
       - `doubao` - 豆包
     - `--size`: 图片尺寸（可选，仅千问支持）
       - `1664*928` - 16:9 横屏（默认）
       - `1024*1024` - 正方形
       - `720*1280` - 9:16 竖屏
       - `1280*720` - 16:9 横屏（小尺寸）
     - `--model`: 千问模型（可选，仅千问支持）
       - `qwen-image-2.0` - 标准版（默认）
       - `qwen-image-2.0-pro` - 专业版
   - 示例：
     ```bash
     # 使用千问生成（默认）—— prompt 必须遵循五要素结构
     python3 scripts/generate_cover.py --prompt "深夜办公室仅亮着的台式电脑屏幕，桌面散落咖啡杯与便签，窗外城市灯火，专业冷静，深蓝紫渐变，微缩等距风格，16:9 横向构图，主体居中偏左，留白用于标题文字，画面中不要出现任何文字、字母、汉字、水印、logo、签名"

     # 切换到豆包
     python3 scripts/generate_cover.py --prompt "<五要素 prompt>" --provider doubao

     # 千问生成正方形封面
     python3 scripts/generate_cover.py --prompt "<五要素 prompt>" --provider qwen --size "1024*1024"

     # 千问专业版模型
     python3 scripts/generate_cover.py --prompt "<五要素 prompt>" --provider qwen --qwen-model qwen-image-2.0-pro
     ```
   - 脚本返回图片URL
   - 下载图片到本地临时文件
   - 调用 `scripts/upload_material.py` 上传图片到微信服务器
   - 参数说明：
     - `--app_id`: 微信公众号AppID（必需）
     - `--app_secret`: 微信公众号AppSecret（必需）
     - `--image_path`: 图片文件路径（必需）
   - 脚本返回 thumb_media_id

3. **创建文章草稿**
   - 调用 `scripts/create_draft.py` 创建草稿
   - 参数说明：
     - `--app_id`: 微信公众号AppID（必需）
     - `--app_secret`: 微信公众号AppSecret（必需）
     - `--title`: 文章标题（必需）
     - `--content`: 文章HTML内容（必需）
     - `--digest`: 文章摘要（可选）
     - `--author`: 作者名称（可选）
     - `--thumb_media_id`: 封面图片media_id（可选，从步骤2获取）
     - `--show_cover_pic`: 是否显示封面，默认为1（可选）
   - 脚本返回 media_id

4. **发布草稿到公众号**
   - 调用 `scripts/publish_draft.py` 发布草稿
   - 参数说明：
     - `--app_id`: 微信公众号AppID（必需）
     - `--app_secret`: 微信公众号AppSecret（必需）
     - `--media_id`: 草稿的media_id（必需）
   - 脚本返回 publish_id

5. **验证发布状态**
   - 根据 publish_id 查询发布状态
   - 确认文章是否成功发布

### 流程 C：发布无封面文章

如果不需要封面图片，可以跳过流程 B 的步骤2，直接执行步骤3创建草稿。

### 可选分支
- 当 **需要智能创作**：使用流程 0 的智能工作流（推荐）
- 当 **需要阅读文章**：使用流程 A 提取并转换为 Markdown
- 当 **需要预览文章**：先创建草稿但不发布，使用草稿预览功能
- 当 **需要发布多篇内容**：重复创建草稿和发布步骤，media_id可复用
- 当 **需要转载文章**：使用流程 0 模式 1（基于参考文章改写）

## 资源索引
- 核心脚本：
  - [scripts/create_article.py](scripts/create_article.py) - 智能文章创作工作流（新增，支持多种模式）
  - [scripts/extract_to_markdown.py](scripts/extract_to_markdown.py) - 提取文章并转换为 Markdown（参数：url, -o 输出路径, --json）
  - [scripts/generate_cover.py](scripts/generate_cover.py) - 生成封面图片（参数：prompt, --provider, --size, --model）
  - [scripts/upload_material.py](scripts/upload_material.py) - 上传图片到微信（参数：app_id, app_secret, image_path）
  - [scripts/create_draft.py](scripts/create_draft.py) - 创建文章草稿（参数：app_id, app_secret, title, content, digest, author等）
  - [scripts/publish_draft.py](scripts/publish_draft.py) - 发布草稿到公众号（参数：app_id, app_secret, media_id）
- 转换器模块：
  - [reader/extract_and_convert.js](reader/extract_and_convert.js) - Node.js 提取与转换引擎（由 extract_to_markdown.py 调用）
- 领域参考：
  - [references/wechat-api.md](references/wechat-api.md) - 微信公众号API接口说明（获取access_token、创建草稿、发布接口、上传素材、错误码）
  - [references/article-format.md](references/article-format.md) - 文章HTML格式规范（支持的标签、样式、图片）
  - [references/viral_titles_50.md](references/viral_titles_50.md) - 50 个爆款标题范例 + 背后逻辑
  - [references/三遍审校清单.md](references/三遍审校清单.md) - 三遍审校流程（内容 / 风格 / 细节），初稿写完后强制执行
  - [HUMANIZE_GUIDE.md](HUMANIZE_GUIDE.md) - 去 AI 化写作详细规则与替换词表
  - [CHICKEN_SOUP_STYLE.md](CHICKEN_SOUP_STYLE.md) - 鸡汤情感类文章风格指南

## 注意事项
- 文章内容必须符合微信公众号的HTML格式规范
- 封面图片建议尺寸900*383，大小不超过2MB
- app_id和app_secret必须正确，否则无法获取access_token
- 脚本会自动获取access_token，无需手动管理
- 生成封面图片需要图像 API 凭证（默认千问 `DASHSCOPE_API_KEY`，可选豆包 `DOUBAO_API_KEY`）
- 发布操作不可逆，建议先创建草稿确认内容
- 注意API调用频率限制

## 使用示例

### 示例0：智能工作流 - 基于参考文章改写（推荐）
```
用户对话：
"帮我改写这篇文章并发布到草稿箱：https://mp.weixin.qq.com/s/xxx"

智能体执行流程：
1. 提取原文
   python3 scripts/extract_to_markdown.py "https://mp.weixin.qq.com/s/xxx" --json

2. 改写文章（智能体使用 AI 能力）
   - 分析原文内容
   - 生成新的标题和内容
   - 保持核心观点，改变表达方式

3. 生成封面（按五要素结构构造 prompt，禁用空泛描述）
   python3 scripts/generate_cover.py --prompt "<从原文提取的核心意象>，<情绪基调>，<画面风格>，16:9 横向构图，主体居中偏左，留白用于标题文字，画面中不要出现任何文字、字母、汉字、水印、logo、签名" --provider qwen

4. 下载并上传封面
   wget -O cover.jpg "封面URL"
   python3 scripts/upload_material.py --app_id xxx --app_secret xxx --image_path cover.jpg

5. 创建草稿
   python3 scripts/create_draft.py --app_id xxx --app_secret xxx \
     --title "新标题" --content "改写后的HTML内容" --thumb_media_id xxx

✅ 完成！草稿已创建
```

### 示例0.1：智能工作流 - 基于主题方向创作（任务类型 B：含选题讨论）
```
用户对话第 1 轮：
"写一篇关于 AI 提升工作效率的文章"

智能体（任务类型 B → 先选题讨论，禁止直接动笔）：
我先给你 3 个选题方向，你挑一个：

方向 1：《下班还在加班的人，都没用对这 3 个 AI 工具》
- 角度：痛点切入，落到具体工具实测
- 大纲：现状痛点 / 工具 1 实测 / 工具 2 实测 / 工具 3 实测 / 选型建议
- 优势：标题命中"身份+量化结果"，转发率高 / 劣势：需要真实搜索 3 款工具的最新数据

方向 2：《我用 AI 把每天的 3 小时杂活压到了 20 分钟》
- 角度：第一人称经历叙事
- ...

方向 3：《2025 年还在手动做这 5 件事，就是在浪费命》
- 角度：清单体 + 时效性
- ...

---

用户对话第 2 轮："方向 1"

智能体执行流程：
1. 搜索相关资料（web_fetch 三款工具的官方文档 + 最新评测）
2. 创作文章初稿
3. 三遍审校（按 references/三遍审校清单.md 执行）
   - 第一遍 内容：核对工具官方信息、数据有源
   - 第二遍 风格：删 AI 套话 + 加态度
   - 第三遍 细节：句长 / 标点 / 中英文空格
4. 生成封面（按五要素结构）
   python3 scripts/generate_cover.py --prompt "深夜办公室仅亮着的台式电脑屏幕，桌面散落咖啡杯与便签，窗外城市灯火，专业冷静，深蓝紫渐变，微缩等距风格，16:9 横向构图，主体居中偏左，留白用于标题文字，画面中不要出现任何文字、字母、汉字、水印、logo、签名" --provider qwen
5. 上传封面并创建草稿
   （同上）

✅ 完成！草稿已创建
```

### 示例1：提取文章并转换为 Markdown
```bash
# 提取并输出 Markdown 到终端
python3 scripts/extract_to_markdown.py "https://mp.weixin.qq.com/s?__biz=MzI..."

# 保存为文件
python3 scripts/extract_to_markdown.py "https://mp.weixin.qq.com/s?__biz=..." -o article.md

# 输出 JSON 格式（含完整元数据）
python3 scripts/extract_to_markdown.py "https://mp.weixin.qq.com/s?__biz=..." --json

# 输出示例：
# {
#   "title": "文章标题",
#   "author": "作者",
#   "publishTime": "2026/03/04 10:11:00",
#   "description": "文章摘要",
#   "cover": "https://封面图URL",
#   "link": "https://原文链接",
#   "content": "Markdown 内容...",
#   "fullMarkdown": "# 文章标题\n\n**作者**: ...\n\n..."
# }
```

### 示例2：发布简单的图文文章（无封面）
```bash
# 1. 生成文章内容（智能体完成）
title="如何使用Python处理数据"
content="<section><h2>Python数据处理基础</h2><p>Python提供了强大的数据处理库...</p></section>"

# 2. 创建草稿
python scripts/create_draft.py \
  --app_id "YOUR_APP_ID" \
  --app_secret "YOUR_APP_SECRET" \
  --title "$title" \
  --content "$content" \
  --author "技术团队"
# 输出: media_id="MEDIA_XXX"

# 3. 发布草稿
python scripts/publish_draft.py \
  --app_id "YOUR_APP_ID" \
  --app_secret "YOUR_APP_SECRET" \
  --media_id "MEDIA_XXX"
# 输出: publish_id="PUBLISH_YYY"
```

### 示例3：发布带自动生成封面的文章
```bash
# 1. 生成文章内容和封面描述（智能体完成）
title="产品更新公告"
content="<section><p>本次更新包含以下内容...</p></section>"

# 封面 prompt 必须遵循『🚨 图片生成核心规范』五要素结构（核心意象+情绪基调+画面风格+构图+负面提示）
cover_prompt="发布会舞台中央的新版应用界面投影，前排观众举起手机拍摄，灯光聚焦，专业冷静，极简商务，金蓝撞色，16:9 横向构图，主体居中偏左，留白用于标题文字，画面中不要出现任何文字、字母、汉字、水印、logo、签名"

# 2. 生成封面图片
python scripts/generate_cover.py --prompt "$cover_prompt"
# 输出: {"success": true, "image_url": "https://...", "revised_prompt": "..."}

# 3. 下载图片（智能体执行）
wget -O cover.jpg "https://..."

# 4. 上传图片到微信
python scripts/upload_material.py \
  --app_id "YOUR_APP_ID" \
  --app_secret "YOUR_APP_SECRET" \
  --image_path "cover.jpg"
# 输出: {"thumb_media_id": "THUMB_XXX", ...}

# 5. 创建草稿（使用封面）
python scripts/create_draft.py \
  --app_id "YOUR_APP_ID" \
  --app_secret "YOUR_APP_SECRET" \
  --title "$title" \
  --content "$content" \
  --digest "产品v2.0版本重要更新" \
  --thumb_media_id "THUMB_XXX" \
  --show_cover_pic 1

# 6. 发布草稿
python scripts/publish_draft.py \
  --app_id "YOUR_APP_ID" \
  --app_secret "YOUR_APP_SECRET" \
  --media_id "MEDIA_YYY"
```

### 示例4：提取文章并重新发布（转载流程）
```bash
# 1. 提取文章并保存为 Markdown
python3 scripts/extract_to_markdown.py "https://mp.weixin.qq.com/s?__biz=..." -o article.md

# 2. 编辑 article.md（可选）

# 3. 将 Markdown 转换为 HTML（智能体完成）
# 使用 markdown 库或其他工具转换

# 4. 生成新封面（按五要素结构构造 prompt）
python3 scripts/generate_cover.py --prompt "<核心意象>，<情绪基调>，<画面风格>，16:9 横向构图，主体居中偏左，留白用于标题文字，画面中不要出现任何文字、字母、汉字、水印、logo、签名"

# 5. 上传封面
python3 scripts/upload_material.py \
  --app_id "YOUR_APP_ID" \
  --app_secret "YOUR_APP_SECRET" \
  --image_path "cover.jpg"

# 6. 创建草稿
python3 scripts/create_draft.py \
  --app_id "YOUR_APP_ID" \
  --app_secret "YOUR_APP_SECRET" \
  --title "【转载】原文标题" \
  --content "<section>HTML内容</section>" \
  --thumb_media_id "THUMB_XXX"

# 7. 发布
python3 scripts/publish_draft.py \
  --app_id "YOUR_APP_ID" \
  --app_secret "YOUR_APP_SECRET" \
  --media_id "MEDIA_XXX"
```

### 示例5：批量发布文章
```bash
# 文章1
python scripts/create_draft.py \
  --app_id "YOUR_APP_ID" \
  --app_secret "YOUR_APP_SECRET" \
  --title "第一篇文章" \
  --content "..."
python scripts/publish_draft.py \
  --app_id "YOUR_APP_ID" \
  --app_secret "YOUR_APP_SECRET" \
  --media_id "MEDIA_001"

# 文章2
python scripts/create_draft.py \
  --app_id "YOUR_APP_ID" \
  --app_secret "YOUR_APP_SECRET" \
  --title "第二篇文章" \
  --content "..."
python scripts/publish_draft.py \
  --app_id "YOUR_APP_ID" \
  --app_secret "YOUR_APP_SECRET" \
  --media_id "MEDIA_002"
```
