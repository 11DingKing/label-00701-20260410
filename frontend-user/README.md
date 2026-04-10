# 音频播放器 - 用户端

基于 Vue 3 + Vite 构建的现代化音频播放器。

## 功能特性

- 支持多音频文件导入（点击或拖拽）
- 随机排列播放列表
- 播放进度条（支持拖动）
- 播放倍速调节（0.5x - 2x）
- 单曲循环
- 两种切换模式：播放完毕切换 / 固定时长切换
- 响应式设计，支持移动端

## 本地开发

```bash
npm install
npm run dev
```

## 构建

```bash
npm run build
```

## Docker 运行

```bash
docker build -t audio-player .
docker run -p 8081:80 audio-player
```
