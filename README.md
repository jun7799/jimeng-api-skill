# 即梦 AI API Skill

> Claude Code / OpenClaw 可用的即梦 AI 图像/视频生成技能

## 功能特性

- **图片生成**：5 种模型，支持纯文本和参考图生成
- **视频生成**：5 种模型，支持纯文本、首帧、首尾帧生成
- **OpenAI 兼容**：完全兼容 OpenAI API 格式
- **完整封装**：提供 Python 工具类，开箱即用

## 快速开始

### 1. 部署服务

```bash
docker run -it -d --init --name jimeng-free-api \
  -p 8001:8000 \
  -v jimeng-data:/app/data \
  -e TZ=Asia/Shanghai \
  ghcr.io/zhizinan1997/jimeng-free-api-all:latest
```

### 2. 获取 Session ID

1. 登录 [即梦官网](https://jimeng.jianying.com/)
2. 按 F12 → Application → Cookies
3. 复制 `sessionid` 值

### 3. 安装 Skill

将 `SKILL.md` 复制到你的 skills 目录：

```bash
# Claude Code
cp SKILL.md ~/.claude/skills/jimeng-api/

# OpenClaw
cp SKILL.md ~/.openclaw/workspace/skills/jimeng-api/
```

### 4. 使用

```python
from jimeng_api import JimengAPI

api = JimengAPI(api_key="你的sessionid")

# 生成图片
urls = api.generate_image("一只可爱的橘猫")

# 首尾帧视频
url = api.generate_video(
    "从第一张图过渡到第二张图",
    model="jimeng-video-3.0",
    first_frame="start.jpg",
    last_frame="end.jpg"
)
```

## 模型说明

| 类型 | 推荐模型 | 说明 |
|------|---------|------|
| 图片测试 | `jimeng-image-3.0` | 最省积分 |
| 图片高质量 | `jimeng-image-4.5` | 最新旗舰 |
| 视频测试 | `jimeng-video-s2.0` | 最省积分 |
| 首尾帧 | `jimeng-video-3.0` | **必须用 3.0 系列** |

## 相关链接

- [即梦官网](https://jimeng.jianying.com/)
- [jimeng-free-api-all](https://github.com/zhizinan1997/jimeng-free-api-all)

## License

MIT
