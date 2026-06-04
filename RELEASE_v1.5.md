# Emby删除兜底清理 v1.5 发布说明

## 🎉 新功能：智能等待清理插件完成

### 问题背景
在 v1.4 中，"双重保险名称搜索"是和原路径复查**并行执行**的。这导致了以下问题：
- 清理插件正在删除文件
- 分身插件正在删除硬链接
- 名称搜索同时在扫描硬盘 📁

这会造成：
1. **重复扫描硬盘**，浪费 IO
2. **可能遗漏**（清理插件删到一半就开始扫描）
3. **硬盘压力大**，影响寿命

---

### ✨ v1.5 解决方案

**方式1（推荐）：文件系统事件监控**

使用 `watchdog` 库实时监控清理插件的监控路径：
- 监听 DELETE 文件删除事件
- 如果在 N 秒内没有新的删除事件 = 清理完成
- 开始名称搜索兜底检查

**方式2（回退）：固定延迟**

如果没有安装 `watchdog`，使用固定延迟模式：
- 默认等待 30 秒（可配置）
- 仍然比同时扫描要好

---

### 📋 新增配置项

| 配置项 | 默认值 | 说明 |
|--------|--------|------|
| `name_search_wait_for_cleaner` | True | 是否等待清理插件完成后再搜索 |
| `name_search_watch_timeout` | 600 | 监控超时秒数（最长等待时间） |
| `name_search_cleaner_idle` | 30 | 清理插件空闲判定秒数（无删除操作多久后认为完成） |

---

### 🛠️ 安装方法

#### 方法1：使用安装包（推荐）
```bash
# 下载 v1.5 安装包
wget https://github.com/Xc518600/MoviePilot-Plugin-EmbyDeleteGuard/releases/download/EmbyDeleteGuard_v1.5/embydeleteguard_v1.5.zip

# 在 MoviePilot 插件管理中上传安装
```

#### 方法2：手动安装
```bash
# 复制到 MoviePilot 插件目录
cp -r plugins.v2 /你的MoviePilot路径/plugins/EmbyDeleteGuard

# 重启 MoviePilot
```

---

### 🐳 Docker 环境安装 watchdog

如果使用 MoviePilot Docker 容器，需要先安装 `watchdog` 库：

```bash
# 进入 MoviePilot 容器
docker exec -it moviepilot bash

# 安装 watchdog
pip install watchdog

# 重启容器
exit
docker restart moviepilot
```

或者修改 Dockerfile / docker-compose.yml：
```yaml
services:
  moviepilot:
    build:
      context: .
    # 或者使用环境变量安装
    command: >
      sh -c "pip install watchdog && python3 main.py"
```

---

### 📊 执行流程对比

#### v1.4（并行执行）
```
webhook收到删除事件
    ↓
等待 60 秒
    ↓
同时执行：
    ├── 1. 原路径复查
    ├── 2. 名称搜索  ⚠️ 可能扫到正在删除的文件
    └── 3. 查找种子
```

#### v1.5（串行执行）
```
webhook收到删除事件
    ↓
等待 60 秒
    ↓
第1步：原路径复查
    ↓
第2步：等待清理插件完成 ⏱️
    ├── 监控文件系统删除事件
    ├── 如果 30 秒无删除 = 清理完成 ✅
    └── 开始名称搜索
    ↓
第3步：名称搜索兜底检查  🛡️
    ↓
第4步：查找种子
```

---

### 💡 使用建议

1. **强烈建议安装 `watchdog`**：获得精确的文件系统监控，硬盘保护效果最好
2. **合理设置超时时间**：默认 10 分钟（600秒），如果清理任务很长可以增加
3. **调整空闲判定秒数**：默认 30 秒，如果清理插件删除很慢可以增加到 60 秒

---

### 🔍 兼容性说明

- ✅ **向后兼容**：v1.4 的配置完全兼容 v1.5
- ✅ **回退机制**：如果没有安装 `watchdog`，自动使用固定延迟模式
- ✅ **可选开启**：可以通过 `name_search_wait_for_cleaner` 关闭此功能

---

### 📝 更新历史

- **v1.5** (2026-06-05): 优化：名称搜索等待清理插件完成。新增文件系统事件监控，智能等待清理插件（RemoveLink及其分身）删除操作完成后，再执行名称搜索，避免重复扫描硬盘，保护硬盘寿命。
- **v1.4**: 新增：双重保险名称搜索。媒体服务器删除事件触发后，除按原路径复查外，还会在对应清理插件/分身插件路径内按剧集名/电影名搜索残留。
- **v1.3**: 新增：自动参考 清理媒体文件 及其分身插件的 monitor_dirs/STRM 映射目录。
- **v1.2**: 新增：插件页面历史记录。
- **v1.1**: 新增：检测到漏删时发送明确通知。
- **v1.0**: 新增：监听 Emby/Jellyfin/Plex 删除类 Webhook，延迟复查清理媒体库残留文件。

---

## 🙏 反馈与支持

- **GitHub**: https://github.com/Xc518600/MoviePilot-Plugin-EmbyDeleteGuard
- **问题反馈**: 提交 Issue

---

**Happy Cleaning! 🧹**