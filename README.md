# AudioFlow 音频播放器

基于 Vue 3 构建的现代化在线音频播放器。

## How to Run

```bash
# 使用 Docker Compose 启动
docker-compose up --build -d

# 停止服务
docker-compose down
```

## Services

| 服务 | 端口 | 说明 |
|------|------|------|
| frontend-user | 8081 | 用户端音频播放器 |

访问地址：http://localhost:8081

## 测试账号

本项目为纯前端应用，无需登录账号。

## 题目内容

制作一个 HTML 界面，可以选择多个音频导入，随后进行随机排列，按照排列顺序播放，屏幕上显示名称（文件名，不带扩展名），有进度条、倍数、单曲循环等，可选择每个音频播放完下一曲或固定多少时长下一曲。

---

## 项目介绍

AudioFlow 是一个功能完善的在线音频播放器，支持以下特性：

### 核心功能
- 多音频文件导入（点击选择或拖拽上传）
- 随机排列播放列表
- 按顺序播放
- 显示文件名（自动去除扩展名）
- 可拖动进度条
- 播放倍速调节（0.5x - 2x）
- 单曲循环
- 两种切换模式：播放完毕切换 / 固定时长切换

### 技术栈
- Vue 3 (Composition API)
- Vite
- Docker + Nginx

## 项目结构

```
├── frontend-user/          # 用户端前端项目
│   ├── src/
│   │   ├── App.vue
│   │   ├── main.js
│   │   └── style.css
│   ├── Dockerfile
│   ├── nginx.conf
│   ├── package.json
│   └── README.md
├── docker-compose.yml
├── .gitignore
└── README.md
```
