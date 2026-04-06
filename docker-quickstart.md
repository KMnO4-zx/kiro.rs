# Docker Quickstart

本文档面向 Linux VPS 场景，说明如何使用仓库根目录下的 `Dockerfile` 部署 `kiro-rs`。

目标：

- 不使用 `docker-compose`
- 首次部署尽量简单
- 后续更新代码或重建容器时不丢失配置和凭据

## 1. 先安装 Git 和 Docker

以下步骤按最常见的 `Ubuntu` / `Debian` VPS 编写。

先更新包索引并安装基础工具：

```bash
sudo apt update
sudo apt install -y ca-certificates curl git
```

安装完成后可先确认 `git` 已可用：

```bash
git --version
```

如果你的系统是 Ubuntu：

```bash
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "${UBUNTU_CODENAME:-$VERSION_CODENAME}") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```

如果你的系统是 Debian：

```bash
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/debian/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/debian \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```

然后安装 Docker：

```bash
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

验证 Docker：

```bash
sudo docker run hello-world
docker --version
```

如果你希望当前用户以后不用每次都写 `sudo`，再执行：

```bash
sudo usermod -aG docker $USER
newgrp docker
```

然后再次验证：

```bash
docker ps
```

说明：

- 如果 `docker ps` 仍然提示权限不足，退出 SSH 后重新登录一次
- 本文后续默认你已经可以直接执行 `docker ...`
- 如果你的 VPS 不是 Ubuntu/Debian，请按官方文档选择对应发行版安装方式

## 2. 拉取仓库

```bash
git clone https://github.com/hank9999/kiro.rs.git
cd kiro.rs
```

后续所有命令默认都在仓库根目录执行。

## 3. 准备配置目录

在仓库根目录创建一个 `config/` 目录：

```bash
mkdir -p config
```

推荐目录结构：

```text
kiro.rs/
├── Dockerfile
├── config/
│   ├── config.json
│   └── credentials.json
└── ...
```

说明：

- `config.json` 建议手动创建
- `credentials.json` 可以一开始没有，后续通过 Admin UI 导入
- `config/` 会挂载到容器内的 `/app/config`
- 只要这个目录在宿主机上保留，重建容器后配置和凭据就能继承

## 4. 创建 config.json

在 `config/config.json` 中写入最小配置：

```json
{
  "host": "0.0.0.0",
  "port": 8990,
  "apiKey": "sk-kiro-rs-your-key",
  "adminApiKey": "sk-admin-your-key",
  "region": "us-east-1"
}
```

字段说明：

- `host` 必须是 `0.0.0.0`
  - 如果写成 `127.0.0.1`，容器外无法访问
- `port` 建议使用 `8990`
  - 和默认示例、容器暴露端口保持一致
- `apiKey` 必填
  - 客户端调用 `/v1/*` 接口时要带这个 key
- `adminApiKey` 可选但强烈建议配置
  - 配置后可使用 `/admin` 导入和管理凭据
- `region` 通常使用 `us-east-1`

如果你暂时不想开放 Admin UI，也可以先不写 `adminApiKey`。但这样后续不能通过网页导入凭据。

## 5. credentials.json 要不要先建

不需要。

这个项目在启动时如果发现 `credentials.json` 不存在，会按空凭据列表启动。也就是说：

- 服务可以先启动
- 但没有凭据时，实际转发请求不可用
- 如果你配置了 `adminApiKey`，可以启动后访问 `/admin` 再导入凭据

如果你希望文件先存在，也可以手动创建一个空数组：

```bash
printf '[]\n' > config/credentials.json
```

推荐做法：

- 单纯想先跑起来：不创建也可以
- 想让目录结构更明确：创建 `config/credentials.json`，内容填 `[]`

## 6. 构建镜像

在仓库根目录执行：

```bash
docker build -t kiro-rs .
```

这一步会：

- 构建前端 Admin UI
- 编译 Rust 程序
- 生成最终镜像

## 7. 启动容器

```bash
docker run -d \
  --name kiro-rs \
  --restart unless-stopped \
  -p 8990:8990 \
  -v "$(pwd)/config:/app/config" \
  --add-host=host.docker.internal:host-gateway \
  kiro-rs
```

参数说明：

- `-d`
  - 后台运行
- `--restart unless-stopped`
  - VPS 重启后自动拉起
- `-p 8990:8990`
  - 映射宿主机和容器端口
- `-v "$(pwd)/config:/app/config"`
  - 将宿主机配置目录挂载到容器内
- `--add-host=host.docker.internal:host-gateway`
  - 如果容器内需要访问宿主机服务或代理，这个映射通常有用

## 8. 查看日志

```bash
docker logs -f kiro-rs
```

如果启动成功，日志里通常会看到：

- 已加载凭据数量
- 服务监听地址
- 可用 API 路由

## 9. 验证服务

先测试模型列表接口：

```bash
curl http://127.0.0.1:8990/v1/models \
  -H "x-api-key: sk-kiro-rs-your-key"
```

如果你在本地电脑上访问，把 `127.0.0.1` 换成你的 VPS IP 或域名。

如果配置了 `adminApiKey`，也可以访问：

```text
http://你的VPSIP:8990/admin
```

然后在网页里导入凭据。

## 10. 凭据会不会丢

只看操作类型：

### 情况 A：重启容器

```bash
docker restart kiro-rs
```

不会丢。

### 情况 B：删除旧容器，再新建容器

```bash
docker rm -f kiro-rs
docker run ...
```

只要你继续挂载同一个宿主机目录：

```bash
-v "$(pwd)/config:/app/config"
```

就不会丢。

原因是：

- 你的真实配置和凭据保存在 VPS 的 `./config/`
- 容器只是读取和回写这个目录
- 数据不依赖容器本身

### 情况 C：不挂载 volume，直接把数据存在容器里

会丢。

只要你删除容器，里面的数据基本就不该再指望保留。

## 11. 后续更新代码的标准流程

如果你改了代码，或者想拉取上游最新代码，推荐这样更新：

```bash
cd kiro.rs
git pull
docker build -t kiro-rs .
docker rm -f kiro-rs
docker run -d \
  --name kiro-rs \
  --restart unless-stopped \
  -p 8990:8990 \
  -v "$(pwd)/config:/app/config" \
  --add-host=host.docker.internal:host-gateway \
  kiro-rs
```

使用作者提供的远程镜像也可以，例如：

```shell
docker run -d \
  --name kiro-rs \
  --restart unless-stopped \
  -p 8990:8990 \
  -v "$(pwd)/config:/app/config" \
  --add-host=host.docker.internal:host-gateway \
  ghcr.io/hank9999/kiro-rs:v2026.3.1
```

这套流程里：

- 镜像会更新
- 容器会重建
- `config/` 目录不会丢
- 之前通过 Admin 导入的凭据仍会保留

