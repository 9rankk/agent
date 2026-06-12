# 鲁迅故里文旅知识库构建项目

## 项目概述

基于B站视频爬虫收集鲁迅故里相关视频数据，构建智慧文旅导览系统的知识库。

**目标：** 为智慧文旅导览智能体提供多模态数据支持

**核心功能：**
- 景点智能导览
- 个性化路线规划
- 文旅知识问答

## 技术栈

- **爬虫：** DrissionPage + yt-dlp
- **数据处理：** Python + ffmpeg
- **语音识别：** Whisper (OpenAI)
- **知识抽取：** Claude/GPT-4
- **知识库：** 向量数据库 (待定)

## 数据处理流程

### 1. 爬虫阶段 (bilibili.py)

```
搜索关键词 → 获取视频列表 → 下载视频+元数据+评论
```

**处理逻辑：**
- 检查视频时长
- 时长 > 20分钟：标记为`pending`（待拆分）
- 时长 ≤ 20分钟：标记为`no_need`（不需拆分）

**输出文件：**
- `bilibili_videos/` - 视频文件
- `raw_data.json` - 视频元数据
- `comment_data.json` - 评论数据
- `bilibili_processed_bvids.txt` - 已处理记录

### 2. 预处理阶段 (视频拆分)

**仅处理长视频 (`拆分状态 == "pending"`)**

```
视频 → Whisper转文本 → AI识别景点 → ffmpeg切分 → 更新JSON
```

**拆分策略：**
- 用Whisper提取带时间戳的文本
- 用大模型分析文本，识别景点和话题边界
- 用ffmpeg按时间戳切分视频
- 每个片段独立存储，带景点标签

**文件组织：**
```
bilibili_videos/
  ├── originals/        # 原视频（可选保留）
  │   └── BV1xx.mp4
  └── segments/         # 切分后的片段
      ├── BV1xx_三味书屋.mp4
      ├── BV1xx_鲁迅故居.mp4
      └── BV1xx_百草园.mp4
```

### 3. 清洗阶段

- 清洗转录文本（去噪、分句）
- 用大模型生成内容标签
- 提取关键信息和摘要

### 4. 知识库构建

- 文本向量化
- 建立索引
- 关联元数据和视频片段

## 数据结构设计

### raw_data.json 结构

**核心原则：单一维度切分**
- 每个视频选择一个主要切分维度（`split_by`）
- 每个片段对应这个维度的一个值（`value`）
- 切分维度可选：`location`（地点）、`theme`（主题）、`content_type`（内容类型）、`time`（时间段）

---

**短视频（不需拆分）：**
```json
{
  "id": "bili_BV1xx",
  "type": "video",
  "title": "鲁迅故里快速游览",
  "source": "B站",
  "file_path": "bilibili_videos/鲁迅故里快速游览_BV1xx.mp4",
  "url": "https://www.bilibili.com/video/BV1xx",
  "pubdate": "2026-05-15 10:30:00",
  "duration": 600,
  "split_status": "no_need",
  "video_stats": {
    "view": 10000,
    "like": 500,
    "coin": 100,
    "favorite": 200,
    "share": 50
  },
  "comment_stats": {
    "count": 10,
    "total_likes": 100,
    "avg_likes": 10
  },
  "tags": {}
}
```

---

**长视频（已拆分）：**
```json
{
  "id": "bili_BV1upWvzZEYU",
  "type": "video",
  "title": "绍兴鲁迅故里游玩全攻略",
  "source": "B站",
  "url": "https://www.bilibili.com/video/BV1upWvzZEYU",
  "pubdate": "2025-10-16 16:00:22",
  "duration": 1800,
  "split_status": "completed",
  "split_by": "location",
  
  "original_video": {
    "file_path": "bilibili_videos/originals/BV1upWvzZEYU.mp4",
    "kept": false
  },
  
  "video_stats": {
    "view": 831,
    "like": 8,
    "coin": 3,
    "favorite": 8,
    "share": 1
  },
  
  "comment_stats": {
    "count": 0,
    "total_likes": 0,
    "avg_likes": 0
  },
  
  "tags": {},
  
  "segments": [
    {
      "segment_id": "BV1upWvzZEYU_seg1",
      "value": "三味书屋",
      "file_path": "bilibili_videos/segments/BV1upWvzZEYU_三味书屋.mp4",
      "time_range": [0, 320],
      "duration": 320,
      "summary": "介绍三味书屋的历史和参观要点"
    },
    {
      "segment_id": "BV1upWvzZEYU_seg2",
      "value": "鲁迅故居",
      "file_path": "bilibili_videos/segments/BV1upWvzZEYU_鲁迅故居.mp4",
      "time_range": [320, 680],
      "duration": 360,
      "summary": "讲解鲁迅故居的布局和生活场景"
    }
  ]
}
```

---

### 关键字段说明

| 字段 | 说明 | 示例 |
|------|------|------|
| `id` | 视频唯一标识 | "bili_BV1upWvzZEYU" |
| `duration` | 视频时长（秒） | 1800 |
| `split_status` | 拆分状态 | "completed" |
| `split_by` | 切分维度 | "location" |
| `original_video.kept` | 是否保留原视频 | false |
| `segments[].value` | 片段主题（对应split_by的值） | "三味书屋" |
| `segments[].time_range` | 片段时间范围（秒） | [0, 320] |

---

### 切分维度示例

**按地点切分（split_by="location"）：**
```json
"segments": [
  {"value": "三味书屋", "time_range": [0, 320]},
  {"value": "鲁迅故居", "time_range": [320, 680]}
]
```

**按主题切分（split_by="theme"）：**
```json
"segments": [
  {"value": "建筑介绍", "time_range": [0, 400]},
  {"value": "历史故事", "time_range": [400, 800]},
  {"value": "游览攻略", "time_range": [800, 1200]}
]
```

**按内容类型切分（split_by="content_type"）：**
```json
"segments": [
  {"value": "讲解", "time_range": [0, 600]},
  {"value": "实拍", "time_range": [600, 1200]}
]
```

### 拆分状态说明

| 状态 | 说明 | 使用场景 |
|------|------|----------|
| `no_need` | 不需拆分 | 短视频（≤20分钟） |
| `pending` | 等待拆分 | 长视频，未处理 |
| `processing` | 拆分中 | 正在处理 |
| `completed` | 已完成 | 拆分成功 |
| `failed` | 拆分失败 | 处理出错，需人工介入 |

## 配置说明

### key.env

```env
cookie=你的B站cookie
```

**获取方式：**
1. 登录B站
2. F12打开开发者工具
3. Network → 找到任意请求 → 复制Cookie

### bilibili.py 配置项

```python
SEARCH_KEYWORDS = ["鲁迅故里"]  # 搜索关键词
MAX_DOWNLOAD = 5                 # 最多下载数量
MAX_PAGES = 3                    # 每个关键词最多翻页数
SCROLL_TIMES = 2                 # 页面滚动次数
```

## 开发规范

### 代码风格

- 函数命名：小写+下划线
- 关键操作添加注释
- 错误处理：静默失败，记录到日志

### 数据处理原则

1. **增量处理**：已处理的不重复处理
2. **状态追踪**：用状态字段标识处理进度
3. **数据完整性**：原视频和片段保持关联
4. **可恢复性**：支持断点续传

### 长视频处理策略

**判断标准：**
- 时长 > 20分钟 → 需要拆分
- 时长 ≤ 20分钟 → 直接使用

**拆分原则：**
- 按景点/主题切分，不是简单的时间切分
- 每个片段5-15分钟为宜
- 保留片段与原视频的关联

**AI识别流程：**
1. Whisper转录 → 带时间戳的文本
2. 大模型分析 → 识别景点和边界
3. 输出切分方案 → JSON格式
4. ffmpeg执行切分 → 生成片段文件
5. 更新元数据 → 记录片段信息

## 景点列表

鲁迅故里核心景点：
- 三味书屋
- 鲁迅故居
- 百草园
- 周家老宅
- 寿家台门
- 土谷祠
- 长庆寺
- 咸亨酒店

## 待开发功能

- [ ] 视频智能拆分脚本
- [ ] Whisper语音识别集成
- [ ] 大模型景点识别
- [ ] 标签自动生成
- [ ] 知识库向量化
- [ ] 导览智能体开发

## 注意事项

1. **Cookie过期**：定期更新key.env中的cookie
2. **ffmpeg路径**：确保ffmpeg已安装并配置正确路径
3. **存储空间**：长视频拆分会占用较多空间
4. **处理时间**：Whisper转录需要时间，建议分批处理
5. **API限制**：大模型API有调用频率限制，注意控制速度
