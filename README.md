## 什么是 strm-bridge？

strm-bridge 是一个 **WebDAV 转流媒体桥接工具**。它能自动扫描 WebDAV 云盘中的媒体文件（电影、电视剧等），生成对应的 `.strm` 格式指针文件，并通过内置的 HTTP 代理提供流媒体播放服务。

简单来说：**把你网盘里的视频文件，变成 Emby、Infuse 等播放器可以直接在线播放的媒体库。**

### 工作流程

```
你的 WebDAV 云盘（123云盘、天翼云等）
        │
        ▼
strm-bridge 扫描并生成 .strm 文件
        │
        ▼
播放器（Emby / Infuse / Jellyfin）读取 .strm 文件
        │
        ▼
通过 strm-bridge 代理直接从云盘播放视频
```

---

## 环境要求

- 已安装 [Docker](https://docs.docker.com/get-docker/)
- 一个支持 WebDAV 的云盘账号（如 123云盘、天翼云等）

---

## 部署方式（二选一）

### 方式一：使用 Docker Compose（推荐）

#### 1. 创建项目目录

```bash
mkdir -p strm-bridge-data/media strm-bridge-data/config
cd strm-bridge-data
```

> `media` 目录存放生成的 `.strm` 文件，`config` 目录保存配置信息。

#### 2. 创建 docker-compose.yml

在 `strm-bridge-data` 目录下创建 `docker-compose.yml`：

```yaml
services:
  strm-bridge:
    image: xiaoniezi/strm-bridge:latest
    container_name: strm-bridge
    hostname: strm-bridge
    restart: unless-stopped
    network_mode: bridge
    ports:
      - "8099:8099"
    volumes:
      - ./media:/app/media
      - ./config:/app/config
```

#### 3. 启动服务

```bash
docker compose up -d
```

Docker 会自动拉取镜像并启动容器。

#### 4. 打开管理页面

在浏览器中访问：**http://127.0.0.1:8099**

> 如果你是在 NAS 或服务器上运行，将 `127.0.0.1` 替换为你的设备 IP 地址。

---

### 方式二：使用 docker run 命令行

#### 1. 创建项目目录

```bash
mkdir -p media config
```

#### 2. 拉取镜像

```bash
docker pull xiaoniezi/strm-bridge:latest
```

#### 3. 启动容器

```bash
docker run -d \
  --name strm-bridge \
  --hostname strm-bridge \
  --restart unless-stopped \
  -p 8099:8099 \
  -v "./media:/app/media" \
  -v "./config:/app/config" \
  xiaoniezi/strm-bridge:latest
```

参数说明：

| 参数 | 说明 |
|------|------|
| `-d` | 后台运行 |
| `--name strm-bridge` | 容器名称 |
| `--hostname strm-bridge` | 固定容器主机名，避免更新重建后设备 ID 变化 |
| `--restart unless-stopped` | 容器停止后自动重启 |
| `-p 8099:8099` | 端口映射（宿主机:容器） |
| `-v "./media:/app/media"` | 挂载 media 目录，存放 .strm 文件 |
| `-v "./config:/app/config"` | 挂载 config 目录，持久化配置 |

> 如果在 Linux 或 NAS 上遇到 `permission denied`，请确认宿主机的 `media` 和 `config` 目录允许容器内的非 root 用户写入。默认镜像用户 UID 是 `65532`，可以执行：`sudo chown -R 65532:65532 media config`。

#### 4. 打开管理页面

在浏览器中访问：**http://127.0.0.1:8099**

---

## 首次登录

打开管理页面后，使用默认账号登录：

| 字段 | 默认值 |
|------|--------|
| 用户名 | `admin` |
| 密码 | `123456` |

首次登录后系统会**强制要求修改密码**，请设置一个安全的密码。

---

## 配置 WebDAV 服务器

登录后，点击左侧菜单的 **设置（Settings）** 页面。

### 添加服务器

在"WebDAV 服务器"区域，填写以下信息：

| 设置项 | 说明 |
|--------|------|
| **名称** | 给你的云盘起个名字（如"我的123云盘"） |
| **地址** | WebDAV 服务器地址（如 `https://webdav.123pan.com`） |
| **用户名** | WebDAV 账号用户名 |
| **密码** | WebDAV 账号密码 |
| **超时** | 连接超时（秒），一般填 `30` 或 `60` |
| **扫描目录** | 要扫描的文件夹路径（如 `/电影`、`/电视剧`），可填多个 |

配置完成后点击页面底部的 **保存** 按钮。

> **注意**：免费版只能配置 **1 个** WebDAV 服务器，多个服务器需要 Pro 授权。

### 测试连接

保存后系统会自动检测 WebDAV 连接状态。如果显示"连接正常"则配置成功。

---

## 扫描媒体文件

配置好 WebDAV 服务器后，点击左侧菜单的 **扫描（Scan）** 页面。

### 手动扫描

点击 **开始扫描** 按钮，系统会：

1. 连接你配置的 WebDAV 云盘
2. 遍历设置的扫描目录
3. 找出所有媒体文件（如 `.mp4`、`.mkv`、`.avi` 等）
4. 在 `media` 目录中生成对应的 `.strm` 文件
5. 自动清理已经不存在的文件的旧 `.strm` 文件
6. 如果配置了 Emby，自动触发 Emby 媒体库刷新

扫描过程中可以实时查看进度。

### 自动扫描（Pro 功能）

在设置页面中配置 `扫描间隔`（单位：分钟），系统会按照设定的时间自动运行扫描。

- 设置为 `240` 表示每 4 小时自动扫描一次
- 设置为 `0` 表示关闭自动扫描
- 自动扫描需要 **Pro 授权**

---

## 在播放器中使用

直接将 `media` 目录挂载到 Emby、Jellyfin 等媒体服务器中，它们会自动识别 `.strm` 文件并建立媒体库。

---

## 授权购买流程

1. 登录管理页面，在 **设置 → 授权** 中复制 **机器码（Machine ID）**
2. 通过闲鱼联系卖家购买授权
3. 将机器码发送给卖家
4. 将卖家返回的授权码粘贴到 **授权（License）** 输入框中
5. 点击保存即可激活

> ✅ 授权信息保存在 `config` 目录中，重装容器后配置和授权不会丢失。

### 功能对比

| 功能 | 免费版 | Pro 版 |
|------|--------|--------|
| 单个 WebDAV 服务器 | ✅ | ✅ |
| 手动扫描 | ✅ | ✅ |
| 多个 WebDAV 服务器 | ❌ | ✅ |
| 自动定时扫描 | ❌ | ✅ |
| Emby 库自动刷新 | ❌ | ✅ |
| 多线程高速扫描 | 1 线程 | 8 线程 |
