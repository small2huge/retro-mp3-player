# 工作流：从音频到双语字幕播放器

如果你有 MP3 音频和中英双语文稿，想做成像这个播放器一样带逐句高亮同步的 Web 页面，按下面步骤操作。

---

## 准备工作

| 需要 | 说明 |
|------|------|
| MP3 文件 | 每首音频一个文件 |
| 英文原文文稿 | 按段落整理好的纯文本 |
| 中文翻译文稿 | 与英文段落一一对应的中文 |
| ElevenLabs API Key | 可选（用于精确时间戳），Starter $6/月 |

## 第 1 步：获取播放器框架

```bash
git clone https://github.com/small2huge/retro-mp3-player.git
cd retro-mp3-player
```

## 第 2 步：放音频文件

把 MP3 复制到 `audio/` 目录，文件名简洁（不要中文和特殊字符）：

```
audio/
├── 01-intro.mp3
├── 02-tim.mp3
├── 03-diane.mp3
└── ...
```

## 第 3 步：获取逐词时间戳

播放器的句子高亮需要知道每个单词在音频中的起止时间（秒）。

### 方案 A：ElevenLabs Scribe v2（推荐）

```bash
curl -X POST "https://api.elevenlabs.io/v1/speech-to-text" \
  -H "xi-api-key: $YOUR_API_KEY" \
  -F "audio=@audio/01-intro.mp3" \
  -F "model_id=scribe_v2" \
  -F "language_code=eng" \
  -F "timestamps_granularity=word" \
  -F "tag_audio_events=true" > 01-timestamps.json
```

返回的 JSON 里找 `words` 数组，每个元素：
```json
{"text": "Hello", "start": 0.5, "end": 0.8}
```

### 方案 B：本地 Whisper（免费）

```bash
pip install openai-whisper
whisper audio/01-intro.mp3 --model large-v3 --language en \
  --word_timestamps True --output_format json
```

## 第 4 步：整理文稿和数据

编辑 `app/data.js`，替换示例数据。格式如下：

```javascript
var M = [{
  "m": "audio/01-intro.mp3",           // ← 音频文件路径
  "l": "Introduction",                 // ← 显示标题
  "cp": ["这是第一段中文",              // ← 中文段落数组
         "这是第二段中文"],
  "ep": ["This is paragraph one.",     // ← 英文段落数组（与 cp 一一对应）
         "This is paragraph two."],
  "wt": [[0.0, 0.3], [0.4, 0.7],      // ← 逐词时间戳，顺序对应英文单词
         [0.8, 1.2], [1.3, 1.6],       //     每个 [start, end] 对应一个单词
         [1.7, 2.0]]                   //     数量 = 英文总单词数
}];
```

> **⚠️ 关键规则**：`wt` 的条目数必须等于 `ep` 中所有英文单词的总数（按空格拆分）。少一个或少多个，句子切割就会错位。

### 如何生成 wt

写一个简单脚本把 ElevenLabs 输出转成 `wt` 数组：

```javascript
// ElevenLabs JSON → wt 数组
const data = require('./01-timestamps.json');
const wt = data.words.map(w => [w.start, w.end]);
console.log(JSON.stringify(wt));
```

## 第 5 步：本地预览

```bash
python3 -m http.server 8765
```

浏览器打开 **http://localhost:8765/app/index.html**

检查：
- [ ] 所有轨道可选
- [ ] 播放时句子高亮跟随
- [ ] 中/英文切换正常
- [ ] 速度调节（0.5x ~ 2x）有效
- [ ] 音量滑块可弹出
- [ ] 皮肤切换（暗色/亮色）
- [ ] 手机浏览器有声纹

## 第 6 步：部署上线

### 简单方式（python 服务器）

```bash
# 在服务器上
nohup python3 -m http.server 8765 &
# 访问 http://你的IP:8765/app/index.html
```

### 正式方式（nginx + HTTPS）

```bash
# 安装 nginx
apt install nginx certbot python3-certbot-nginx

# 创建站点配置
cat > /etc/nginx/conf.d/player.conf << 'EOF'
server {
    listen 80;
    server_name yourdomain.com;
    root /var/www/retro-mp3-player;
    index index.html;
    location /audio/ {
        add_header Cache-Control "public, max-age=86400";
    }
    location ~ \.(html|js|json|css)$ {
        add_header Cache-Control "public, max-age=3600";
    }
}
EOF

# 申请 SSL 证书
certbot --nginx -d yourdomain.com
```

## 自定义技巧

| 想改什么 | 在哪里改 |
|----------|---------|
| 颜色主题 | `index.html` 中 `.light` 和默认 CSS 的颜色值 |
| 播放器宽度 | `.player` 的 `max-width` |
| 默认显示语言 | `render()` 里的 `lang` 变量（'cn' 或 'en'） |
| 支持其他语言对 | 替换 `cp`/`ep` 标签文本和数据结构 |
| 界面文字 | 搜索替换 `中文`、`English`、`音量` 等 UI 文本 |

## 技术要点

### 手机声纹兼容

手机上 AudioContext 默认是 `suspended` 状态，需要用异步方式激活：

```javascript
if (actx.state === 'suspended') {
  actx.resume().then(setupAnalyser);
  return;
}
```

并且 `createMediaElementSource` 对一个 `<audio>` 元素只能调用一次，需要 `vizConnected` 标志位保护。

### 大数据处理

如果 data.js 超过 50KB，保持独立文件加载（`<script src="data.js">`），不要内联到 HTML 中。

### 避免"按钮跑远"

绝对定位的元素（如皮肤切换按钮），其父容器必须有 `position: relative`，否则按钮会相对视口定位。
