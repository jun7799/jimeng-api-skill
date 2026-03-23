---
name: jimeng-api
description: 即梦 AI 图像/视频生成 API。当用户请求生成图片、生成视频、AI绘图、AI视频、首尾帧视频、参考图生成，或代码中涉及 jimeng API 调用时使用此技能。
---

# 即梦 AI 图像/视频生成 API

> OpenAI 兼容的 AI 图像和视频生成服务

## 服务信息

| 项目 | 值 |
|------|------|
| Base URL | `http://localhost:8001/v1` |
| API Key | `<你的sessionid>` |
| 管理控制台 | `http://localhost:8001` |

> **获取 sessionid**：登录 [即梦官网](https://jimeng.jianying.com/) → F12 → Application → Cookies → 复制 `sessionid` 值

## 认证方式

```
Authorization: Bearer <你的sessionid>
```

## 支持的模型

### 图片模型

| 模型 | 分辨率 | 积分消耗 | 说明 |
|------|--------|---------|------|
| `jimeng-image-3.0` | 1K | ~2 | **推荐测试用**，最省积分 |
| `jimeng-image-3.1` | 1K | ~3 | 3.1 版本 |
| `jimeng-image-4.0` | 2K | ~5 | 4.0 版本 |
| `jimeng-image-4.1` | 2K | ~6 | 4.1 版本 |
| `jimeng-image-4.5` | 2K | ~8 | 最新旗舰版 |

### 视频模型

| 模型 | 时长 | 首尾帧 | 积分消耗 | 说明 |
|------|------|--------|---------|------|
| `jimeng-video-s2.0` | 5s | ❌ 不支持 | ~5 | 轻量版，仅首帧 |
| `jimeng-video-2.0-pro` | 5s | ❌ 不支持 | ~10 | 2.0 专业版 |
| `jimeng-video-3.0-fast` | 5s/10s | ✅ 支持 | ~15 | 快速版 |
| `jimeng-video-3.0` | 5s/10s | ✅ **支持** | ~20 | **首尾帧推荐** |
| `jimeng-video-3.0-pro` | 5s/10s | ✅ **支持** | ~30 | 专业版，质量最高 |

> **重要**：首尾帧功能必须使用 **视频 3.0** 系列模型（3.0/3.0-fast/3.0-pro），S2.0 和 2.0-pro 不支持尾帧！

---

## Python 工具类 (可直接复用)

### 完整封装版 (推荐)

```python
import json
import base64
import urllib.request
import re

class JimengAPI:
    """即梦 AI API 客户端"""

    BASE_URL = "http://localhost:8001/v1"
    API_KEY = "<你的sessionid>"

    def __init__(self, api_key=None):
        self.api_key = api_key or self.API_KEY

    def _request(self, endpoint, payload):
        """发送请求"""
        req = urllib.request.Request(
            f"{self.BASE_URL}{endpoint}",
            data=json.dumps(payload).encode('utf-8'),
            headers={
                'Content-Type': 'application/json',
                'Authorization': f'Bearer {self.api_key}'
            }
        )
        with urllib.request.urlopen(req, timeout=300) as resp:
            return json.loads(resp.read().decode('utf-8'))

    @staticmethod
    def encode_image(path):
        """图片转 Base64"""
        with open(path, 'rb') as f:
            return f'data:image/jpeg;base64,{base64.b64encode(f.read()).decode()}'

    @staticmethod
    def extract_url(content):
        """从响应中提取媒体 URL"""
        match = re.search(r'!\[.*?\]\((https://[^)]+)\)', content)
        return match.group(1) if match else None

    def generate_image(self, prompt, model="jimeng-image-3.0", ref_image=None):
        """生成图片

        Args:
            prompt: 提示词
            model: 模型名称
            ref_image: 参考图片路径 (可选)

        Returns:
            图片 URL 列表
        """
        content = [{"type": "text", "text": prompt}]
        if ref_image:
            content.append({
                "type": "image_url",
                "image_url": {"url": self.encode_image(ref_image)}
            })

        result = self._request('/chat/completions', {
            'model': model,
            'messages': [{'role': 'user', 'content': content}]
        })

        text = result['choices'][0]['message']['content']
        urls = re.findall(r'!\[.*?\]\((https://[^)]+)\)', text)
        return urls

    def generate_video(self, prompt, model="jimeng-video-3.0",
                       first_frame=None, last_frame=None):
        """生成视频

        Args:
            prompt: 提示词
            model: 模型名称 (首尾帧必须用 video-3.0 系列!)
            first_frame: 首帧图片路径
            last_frame: 尾帧图片路径 (可选)

        Returns:
            视频 URL
        """
        content = [{"type": "text", "text": prompt}]

        if first_frame:
            content.append({
                "type": "image_url",
                "image_url": {"url": self.encode_image(first_frame)}
            })
        if last_frame:
            content.append({
                "type": "image_url",
                "image_url": {"url": self.encode_image(last_frame)}
            })

        result = self._request('/chat/completions', {
            'model': model,
            'messages': [{'role': 'user', 'content': content}]
        })

        text = result['choices'][0]['message']['content']
        return self.extract_url(text)

    def list_models(self):
        """获取模型列表"""
        req = urllib.request.Request(
            f"{self.BASE_URL}/models",
            headers={'Authorization': f'Bearer {self.api_key}'}
        )
        with urllib.request.urlopen(req, timeout=30) as resp:
            return json.loads(resp.read().decode('utf-8'))


def download_media(url, save_path):
    """下载媒体文件"""
    urllib.request.urlretrieve(url, save_path)
    print(f"已保存到: {save_path}")
    return save_path


# ============ 使用示例 ============

if __name__ == "__main__":
    api = JimengAPI()

    # 1. 生成图片
    urls = api.generate_image("一只可爱的橘猫在阳光下打盹")
    print("图片:", urls)

    # 2. 参考图生成
    urls = api.generate_image("变成梵高风格", ref_image="input.jpg")
    print("生成图片:", urls)

    # 3. 纯文本生成视频
    video_url = api.generate_video("一只猫咪在草地上奔跑，5秒视频")
    print("视频:", video_url)

    # 4. 首帧视频 (单张图片)
    video_url = api.generate_video(
        "让这张图片动起来",
        first_frame="image.jpg"
    )

    # 5. 首尾帧视频 (两张图片)
    video_url = api.generate_video(
        "从第一张图过渡到第二张图，5秒视频",
        model="jimeng-video-3.0",  # 必须用 3.0!
        first_frame="start.jpg",
        last_frame="end.jpg"
    )
    print("视频:", video_url)

    # 6. 下载视频
    download_media(video_url, "output_video.mp4")
```

---

## 快速调用代码片段

### 生成图片

```python
import urllib.request, json, base64, re

def generate_image(prompt, model="jimeng-image-3.0"):
    payload = {
        'model': model,
        'messages': [{'role': 'user', 'content': prompt}]
    }
    req = urllib.request.Request(
        'http://localhost:8001/v1/chat/completions',
        data=json.dumps(payload).encode('utf-8'),
        headers={
            'Content-Type': 'application/json',
            'Authorization': 'Bearer <你的sessionid>'
        }
    )
    with urllib.request.urlopen(req, timeout=120) as resp:
        result = json.loads(resp.read().decode('utf-8'))
        return re.findall(r'!\[.*?\]\((https://[^)]+)\)',
                         result['choices'][0]['message']['content'])

# 使用
urls = generate_image("一只可爱的橘猫")
print(urls)
```

### 首尾帧生成视频

```python
import urllib.request, json, base64, re

def generate_video_frames(prompt, start_img, end_img, model="jimeng-video-3.0"):
    # 读取图片
    with open(start_img, 'rb') as f:
        img1 = 'data:image/jpeg;base64,' + base64.b64encode(f.read()).decode()
    with open(end_img, 'rb') as f:
        img2 = 'data:image/jpeg;base64,' + base64.b64encode(f.read()).decode()

    payload = {
        'model': model,
        'messages': [{
            'role': 'user',
            'content': [
                {'type': 'text', 'text': prompt},
                {'type': 'image_url', 'image_url': {'url': img1}},
                {'type': 'image_url', 'image_url': {'url': img2}}
            ]
        }]
    }

    req = urllib.request.Request(
        'http://localhost:8001/v1/chat/completions',
        data=json.dumps(payload).encode('utf-8'),
        headers={
            'Content-Type': 'application/json',
            'Authorization': 'Bearer <你的sessionid>'
        }
    )

    with urllib.request.urlopen(req, timeout=180) as resp:
        result = json.loads(resp.read().decode('utf-8'))
        content = result['choices'][0]['message']['content']
        match = re.search(r'!\[video\]\((https://[^)]+)\)', content)
        return match.group(1) if match else content

# 使用
url = generate_video_frames("从地板飞向天空，5秒视频", "1.jpg", "2.jpg")
print(url)
```

### 下载媒体文件

```python
import urllib.request

def download_media(url, save_path):
    urllib.request.urlretrieve(url, save_path)
    print(f"已保存: {save_path}")
    return save_path

# 使用
download_media(video_url, "output_video.mp4")
```

### 查看容器日志

```python
import subprocess

def view_docker_logs(lines=50):
    result = subprocess.run(
        ["docker", "logs", "jimeng-free-api", "--tail", str(lines)],
        capture_output=True, text=True
    )
    print(result.stdout)
    return result.stdout

# 使用
view_docker_logs(20)
```

---

## 比例控制

在提示词中添加比例关键词：

**图片**：`16:9`、`9:16`、`1:1`、`4:3`、`3:4`、`21:9`、`横屏`、`竖屏`、`方形`

**视频**：`16:9`、`9:16`、`1:1`、`4:3`、`3:4`

## 视频时长

提示词中添加 `5秒` 或 `10秒` 控制时长（部分模型支持）

---

## 常见问题排查

| 问题 | 原因 | 解决方案 |
|------|------|---------|
| 首尾帧不生效 | 模型不支持 | 必须用 `jimeng-video-3.0` 系列 |
| 图片上传失败 | 外部 URL 防盗链 | 使用 Base64 编码图片 |
| 积分不足 | 账号积分用完 | 换账号/隔天重试（每天送积分） |
| 视频生成失败 | 图片格式/大小问题 | 确保图片 < 10MB |
| 视频链接打不开 | 链接有时效 | 及时下载保存 |
| 服务无响应 | Docker 容器问题 | `docker restart jimeng-free-api` |

---

## Docker 管理命令

```bash
# 查看状态
docker ps | grep jimeng

# 查看日志
docker logs jimeng-free-api --tail 50

# 重启服务
docker restart jimeng-free-api

# 停止服务
docker stop jimeng-free-api

# 启动服务
docker start jimeng-free-api
```

---

## 注意事项

1. **图片输入推荐使用 Base64 编码**，外部 URL 可能有防盗链问题
2. **测试时用 `jimeng-image-3.0` 和 `jimeng-video-s2.0`** 节省积分
3. **首尾帧必须用 `jimeng-video-3.0` 系列**
4. 图片/视频链接有时效，建议及时保存
5. 视频生成较慢，通常需要 1-3 分钟
