# AI 工程师的 Linux 学习指南

> 本文档面向**新手小白**，系统讲解学习 AI / 大模型 / RAG 过程中**必须掌握**的 Linux 知识。
> 所有内容结合本项目实战（Milvus 部署、Python 多进程、Docker、远程服务器）。
> 跳转链接使用 `file:///` 协议，可直接定位到项目源码。

---

## 📚 目录

- [一、为什么 AI 工程师要学 Linux](#一为什么-ai-工程师要学-linux)
- [二、Linux 入门：它到底是什么](#二linux-入门它到底是什么)
- [三、选哪个 Linux 发行版](#三选哪个-linux-发行版)
- [四、虚拟机 / WSL / 云服务器 三种使用方式](#四虚拟机--wsl--云服务器-三种使用方式)
- [五、终端与最常用的 30 个命令](#五终端与最常用的-30-个命令)
- [六、文件与目录：一切的根基](#六文件与目录一切的根基)
- [七、文本处理三剑客](#七文本处理三剑客)
- [八、用户、权限与 sudo](#八用户权限与-sudo)
- [九、进程与服务管理](#九进程与服务管理)
- [十、SSH 远程登录与文件传输](#十ssh-远程登录与文件传输)
- [十一、网络与端口](#十一网络与端口)
- [十二、Shell 脚本入门](#十二shell-脚本入门)
- [十三、Python 环境管理（conda / venv / pip）](#十三python-环境管理conda--venv--pip)
- [十四、Docker 入门（Milvus 部署必备）](#十四docker-入门milvus-部署必备)
- [十五、与本项目结合的实战场景](#十五与本项目结合的实战场景)
- [十六、学习路线图](#十六学习路线图)
- [十七、常见问题 FAQ](#十七常见问题-faq)

---

## 一、为什么 AI 工程师要学 Linux

| 场景 | Linux 必要性 |
| --- | --- |
| 部署 Milvus 向量数据库 | ⭐⭐⭐⭐⭐ Docker 几乎只在 Linux 上最稳定 |
| 跑大模型训练 / 推理 | ⭐⭐⭐⭐⭐ GPU 驱动、CUDA、PyTorch 在 Linux 上最完善 |
| 云服务器（阿里云/腾讯云/AWS） | ⭐⭐⭐⭐⭐ 默认就是 Linux |
| 跑 LangChain / LangGraph 服务 | ⭐⭐⭐⭐ FastAPI/Docker 部署通常都在 Linux |
| 公司内部 AI 平台 | ⭐⭐⭐⭐⭐ 几乎都是 Linux 内核 |

> 一句话：**Linux 是 AI 工程化的"操作系统母语"**。

---

## 二、Linux 入门：它到底是什么

### 2.1 一句话定义

**Linux** 是一个**开源、免费**的操作系统内核，由 Linus Torvalds 在 1991 年发布。围绕这个内核，各公司/社区打包出各种"发行版"（Distro）。

### 2.2 Linux vs Windows 核心差异

| 维度 | Windows | Linux |
| --- | --- | --- |
| 出身 | 闭源、微软 | 开源、社区 |
| 文件系统 | NTFS、C:\ | ext4、/ |
| 大小写 | 不敏感 | **敏感**（`File.txt` ≠ `file.txt`） |
| 路径分隔符 | `\` | `/` |
| 终端 | PowerShell / CMD | Bash / Zsh |
| 软件安装 | 双击 .exe | `apt install` / `yum install` / `pip install` |
| 权限模型 | 用户 + UAC | 用户 + 文件 rwx + sudo |

### 2.3 Linux 的"长相"

不像 Windows 有桌面图标，Linux 服务器通常长这样：

```
root@server:~# _
```

一个**黑底白字的终端**就是你的全部界面。新手不要怕，所有操作就是"敲命令 + 回车"。

---

## 三、选哪个 Linux 发行版

| 发行版 | 包管理器 | 适合 | 推荐度 |
| --- | --- | --- | --- |
| **Ubuntu 22.04 / 24.04** | `apt` | 新手入门、云服务器、文档最多 | ⭐⭐⭐⭐⭐ |
| **Debian 12** | `apt` | 服务器、稳定性优先 | ⭐⭐⭐⭐ |
| **CentOS Stream / Rocky Linux** | `yum` / `dnf` | 国内企业服务器常见 | ⭐⭐⭐⭐ |
| **macOS**（类 Unix） | `brew` | 苹果电脑开发者 | ⭐⭐⭐⭐⭐ |
| **Arch Linux** | `pacman` | 折腾爱好者 | ⭐⭐ |

> 🎯 **推荐**：新手直接用 **Ubuntu 22.04 LTS**，本教程所有命令都基于 Ubuntu。

---

## 四、虚拟机 / WSL / 云服务器 三种使用方式

| 方式 | 适合 | 优点 | 缺点 |
| --- | --- | --- | --- |
| **虚拟机**（VMware / VirtualBox） | 想完全模拟 | 隔离好 | 占资源 |
| **WSL2**（Windows 内嵌 Linux） | Windows 用户首选 | 无缝切换 | 性能略低于原生 |
| **云服务器**（阿里云/腾讯云） | 真实生产环境 | 公网 IP | 要花钱 |

### 4.1 WSL2 安装（Windows 用户推荐）

```powershell
# PowerShell（管理员）
wsl --install                 # 默认安装 Ubuntu
wsl --set-default-version 2
wsl -l -v                     # 查看已安装的发行版
```

> 安装后在开始菜单搜索 "Ubuntu" 即可打开终端。

### 4.2 虚拟机安装 Ubuntu

1. 下载 [VirtualBox](https://www.virtualbox.org/) 或 VMware；
2. 下载 [Ubuntu 22.04 ISO](https://releases.ubuntu.com/22.04/)；
3. 新建虚拟机 → 选择 ISO → 启动 → 按提示安装。

---

## 五、终端与最常用的 30 个命令

> 建议每天敲 10 遍，先形成肌肉记忆。

### 5.1 文件与目录（10 个）

| 命令 | 作用 | 示例 |
| --- | --- | --- |
| `pwd` | 显示当前目录 | `pwd` → `/home/zs` |
| `ls` | 列出文件 | `ls -lh`（详细+人类可读大小） |
| `cd` | 切换目录 | `cd /usr/local` |
| `mkdir` | 创建目录 | `mkdir -p a/b/c`（递归创建） |
| `rm` | 删除 | `rm -rf dir/`（慎用！） |
| `cp` | 复制 | `cp a.txt b.txt` |
| `mv` | 移动/重命名 | `mv a.txt b.txt` |
| `touch` | 创建空文件 | `touch test.py` |
| `cat` | 查看全部内容 | `cat file.txt` |
| `less` / `more` | 分页查看 | `less huge.log` |

### 5.2 文本查看与编辑（5 个）

| 命令 | 作用 |
| --- | --- |
| `head -n 10 file` | 看前 10 行 |
| `tail -n 10 file` | 看后 10 行 |
| `tail -f file` | **实时追踪日志**（AI 训练必备） |
| `nano file` | 简单编辑器（新手友好） |
| `vim file` | 强大编辑器（先学 `i` 输入、`Esc`、`:wq` 保存退出） |

### 5.3 系统与进程（5 个）

| 命令 | 作用 | 示例 |
| --- | --- | --- |
| `ps aux` | 查看所有进程 | `ps aux \| grep python` |
| `top` / `htop` | 实时资源监控 | 看 CPU/内存占用 |
| `kill PID` | 杀进程 | `kill -9 12345`（强杀） |
| `free -h` | 看内存 | |
| `df -h` | 看磁盘 | |
| `nvidia-smi` | 看 GPU（深度学习必备） | |

### 5.4 搜索与权限（5 个）

| 命令 | 作用 | 示例 |
| --- | --- | --- |
| `find` | 找文件 | `find / -name "*.py"` |
| `grep` | 找文本 | `grep "error" log.txt` |
| `chmod` | 改权限 | `chmod +x run.sh` |
| `chown` | 改所有者 | `chown zs:zs file.txt` |
| `which` / `whereis` | 找命令位置 | `which python` |

### 5.5 帮助（5 个）

| 命令 | 作用 |
| --- | --- |
| `man cmd` | 查看命令手册（**最权威**） |
| `cmd --help` | 简要帮助 |
| `history` | 历史命令 |
| `tab` 键 | **自动补全**（必用！） |
| `Ctrl + C` | 终止当前命令 |
| `Ctrl + L` | 清屏 |
| `↑ / ↓` | 翻历史命令 |

### 5.6 速记口诀

```
ls 看文件，cd 换目录
cat 看内容，grep 找关键字
ps 看进程，kill 杀进程
chmod 改权限，sudo 提权
man 是老师，tab 是神器
```

---

## 六、文件与目录：一切的根基

### 6.1 目录结构

Linux 文件系统是一棵**倒过来的树**：

```
/                    ← 根目录（一切从这里开始）
├── home/            ← 用户家目录
│   └── zs/          ← 用户 zs 的家目录（~ 表示）
├── root/            ← 超级用户家目录
├── etc/             ← 配置文件
│   └── apt/         ← apt 源配置
├── var/             ← 变化的数据（日志、缓存）
│   └── log/         ← 系统日志
├── usr/             ← 用户程序
│   └── local/       ← 手动安装的软件
├── opt/             ← 可选软件（如 Milvus）
├── tmp/             ← 临时文件
└── proc/            ← 进程信息（虚拟文件系统）
```

### 6.2 重要目录速记

| 路径 | 用途 |
| --- | --- |
| `/home/用户名` | 你的工作区 |
| `/etc` | 配置文件（如 `/etc/nginx/nginx.conf`） |
| `/var/log` | 日志（如 `/var/log/syslog`） |
| `/tmp` | 临时文件（重启清空） |
| `/usr/local/bin` | 自编译软件 |
| `/opt` | 第三方大型软件 |

### 6.3 绝对路径 vs 相对路径

```bash
cd /home/zs/project      # 绝对路径（从 / 开始）
cd project               # 相对路径（从当前目录开始）
cd ..                    # 上级目录
cd ~                     # 家目录
cd -                     # 上次所在目录
```

---

## 七、文本处理三剑客

### 7.1 grep —— 找内容

```bash
grep "error" log.txt             # 找含 error 的行
grep -n "error" log.txt          # 显示行号
grep -i "ERROR" log.txt          # 忽略大小写
grep -r "TODO" src/              # 递归目录
grep -v "debug" log.txt          # 反选（不含 debug）
ps aux | grep python             # 管道组合
```

### 7.2 sed —— 改内容

```bash
sed -i 's/old/new/g' file.txt          # 全局替换
sed -i 's/old/new/' file.txt           # 只换每行第一个
sed -n '10,20p' file.txt               # 打印 10~20 行
sed -i '/^$/d' file.txt                # 删除空行
```

### 7.3 awk —— 切列统计

```bash
awk '{print $1}' file.txt              # 打印第 1 列
awk -F',' '{print $2}' data.csv        # 按逗号切分，打印第 2 列
awk '{sum+=$1} END {print sum}' f.txt  # 求和
ps aux | awk '{print $2, $11}'         # 看进程 PID 和命令
```

> 🎯 **实战**：本项目跑训练时，`tail -f train.log | grep loss` 实时盯 loss 下降。

---

## 八、用户、权限与 sudo

### 8.1 文件权限

```bash
ls -l file.txt
# -rw-r--r-- 1 zs zs 1024 Sep 16 10:00 file.txt
#  ↑  ↑↑↑  ↑↑↑
#  |  |   └── 其他用户（others）
#  |  └────── 组（group）
#  └───────── 所有者（user）
```

权限位含义：

```
r (read)    = 4   读
w (write)   = 2   写
x (execute) = 1   执行
-           = 0   无

rwx = 4+2+1 = 7
r-x = 4+0+1 = 5
rw- = 4+2+0 = 6
```

### 8.2 修改权限

```bash
chmod 755 script.sh     # rwxr-xr-x
chmod +x script.sh      # 给所有人加执行权限
chmod -R 644 dir/       # 递归改目录
chown zs:zs file.txt    # 改所有者和组
```

### 8.3 sudo —— 临时提权

```bash
sudo apt update         # 以 root 权限执行
sudo -i                 # 切换到 root 终端（危险！）
```

---

## 九、进程与服务管理

### 9.1 查看进程

```bash
ps aux                  # 所有进程
ps aux | grep python    # 找 python 进程
top                     # 实时监控（按 q 退出）
htop                    # 更友好的 top（需安装）
```

### 9.2 杀进程

```bash
kill PID                # 优雅退出（发送 SIGTERM）
kill -9 PID             # 强杀（SIGKILL）
pkill python            # 按名字杀
killall python          # 同 pkill
```

### 9.3 前后台运行

```bash
python train.py              # 前台跑（占终端）
python train.py &            # 后台跑（释放终端）
nohup python train.py &      # 后台跑，关闭终端也不停
nohup python train.py > log.txt 2>&1 &   # 输出到日志
```

> 🎯 **本项目实战**：[documents/write_milvus.py](../documents/write_milvus.py) 的多进程写入就需要在 Linux 后台跑。

### 9.4 systemd 服务（生产部署）

```ini
# /etc/systemd/system/milvus.service
[Unit]
Description=Milvus Server
After=network.target

[Service]
ExecStart=/usr/local/bin/milvus run standalone
Restart=always
User=milvus

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl daemon-reload
sudo systemctl start milvus
sudo systemctl enable milvus     # 开机自启
sudo systemctl status milvus
```

---

## 十、SSH 远程登录与文件传输

### 10.1 SSH 登录

```bash
ssh user@server_ip                  # 密码登录
ssh -p 2222 user@server_ip          # 指定端口
ssh -i ~/.ssh/key.pem user@ip       # 密钥登录（云服务器）
```

### 10.2 文件传输

```bash
# scp（基于 SSH）
scp local.txt user@server:/home/user/         # 本地 → 远程
scp user@server:/home/user/remote.txt ./      # 远程 → 本地
scp -r dir/ user@server:/home/user/           # 传目录

# rsync（增量同步，更快）
rsync -avz --progress ./project user@server:/home/user/

# 推荐工具
# Windows: WinSCP、MobaXterm（带图形界面）
# macOS: Termius、Transmit
```

### 10.3 配置免密登录

```bash
# 本地执行
ssh-keygen -t rsa                  # 生成密钥（一路回车）
ssh-copy-id user@server_ip         # 把公钥推送到服务器
# 之后 ssh user@server_ip 就不需要密码了
```

---

## 十一、网络与端口

### 11.1 常用命令

```bash
ip addr                    # 看网卡 IP（旧命令：ifconfig）
ping baidu.com             # 测试网络
curl https://api.openai.com # 测试 HTTP 接口
wget https://example.com/file.zip  # 下载文件

# 端口与连接
netstat -tulnp             # 查看所有监听端口（旧）
ss -tulnp                  # 新版
lsof -i :19530             # 谁在用 19530 端口（Milvus）
```

### 11.2 防火墙

```bash
# Ubuntu（ufw）
sudo ufw allow 22          # 开放 SSH
sudo ufw allow 19530       # 开放 Milvus 端口
sudo ufw enable            # 启用防火墙
sudo ufw status            # 查看状态

# CentOS（firewalld）
sudo firewall-cmd --permanent --add-port=19530/tcp
sudo firewall-cmd --reload
```

### 11.3 本项目相关端口

| 端口 | 用途 |
| --- | --- |
| 22 | SSH |
| 80 / 443 | HTTP / HTTPS |
| 19530 | Milvus |
| 9091 | Milvus 健康检查 |
| 8000 | MCP 服务端（[test_mcp/mcp_server.py](../test_mcp/mcp_server.py)） |
| 7860 | Gradio UI（[graph2/graph_gradio.py](../graph2/graph_gradio.py)） |

---

## 十二、Shell 脚本入门

### 12.1 第一个脚本

新建 `hello.sh`：

```bash
#!/bin/bash
# 这是注释
echo "Hello, Linux!"
echo "今天是 $(date)"
echo "当前用户: $USER"
```

```bash
chmod +x hello.sh
./hello.sh
```

### 12.2 变量与参数

```bash
name="张三"
echo "你好, $name"
echo "第 1 个参数: $1"
echo "所有参数: $@"
echo "参数个数: $#"
```

### 12.3 条件与循环

```bash
# if
if [ -f "file.txt" ]; then
    echo "文件存在"
else
    echo "文件不存在"
fi

# for 循环
for i in 1 2 3 4 5; do
    echo "第 $i 次"
done

# while
count=0
while [ $count -lt 5 ]; do
    echo $count
    count=$((count + 1))
done
```

### 12.4 实战脚本：自动启动 Milvus

```bash
#!/bin/bash
# start_milvus.sh
set -e  # 出错立即退出

echo "[1/3] 启动 Milvus..."
cd /opt/milvus
sudo docker-compose up -d

echo "[2/3] 等待服务就绪..."
sleep 10
until curl -sf http://localhost:9091/health > /dev/null; do
    echo "等待 Milvus..."
    sleep 5
done

echo "[3/3] Milvus 已就绪 ✅"
```

### 12.5 实战脚本：批量运行 Python 任务

```bash
#!/bin/bash
# run_pipeline.sh
set -e

source .venv/bin/activate           # 激活虚拟环境

echo "[1/3] 解析文档..."
python documents/markdown_parser.py

echo "[2/3] 写入向量库..."
python documents/write_milvus.py

echo "[3/3] 启动服务..."
python graph2/graph_gradio.py
```

---

## 十三、Python 环境管理（conda / venv / pip）

### 13.1 venv（轻量、推荐）

```bash
python3 -m venv .venv              # 创建
source .venv/bin/activate          # 激活（Linux/macOS）
# .venv\Scripts\activate           # Windows
deactivate                         # 退出
```

### 13.2 conda（数据科学常用）

```bash
wget https://repo.anaconda.com/miniconda/Miniconda3-latest-Linux-x86_64.sh
bash Miniconda3-latest-Linux-x86_64.sh
source ~/.bashrc

conda create -n rag python=3.11    # 创建环境
conda activate rag                  # 激活
conda install pytorch -c pytorch    # 装 PyTorch（含 CUDA）
pip install -r requirements.txt     # 装项目依赖
```

### 13.3 pip 常用命令

```bash
pip install package
pip install package==1.2.3
pip install -r requirements.txt
pip list
pip freeze > requirements.txt
pip install --upgrade package
```

### 13.4 换源（国内加速）

```bash
pip install -i https://pypi.tuna.tsinghua.edu.cn/simple package
# 阿里源
pip install -i https://mirrors.aliyun.com/pypi/simple/ package
```

永久换源：

```bash
mkdir -p ~/.pip
cat > ~/.pip/pip.conf <<EOF
[global]
index-url = https://pypi.tuna.tsinghua.edu.cn/simple
trusted-host = pypi.tuna.tsinghua.edu.cn
EOF
```

---

## 十四、Docker 入门（Milvus 部署必备）

### 14.1 为什么需要 Docker

**通俗解释**：Docker 就像一个**标准化的集装箱**，把程序和它依赖的所有环境（Python 版本、CUDA、系统库）打包在一起，**在哪儿跑都一样**。

### 14.2 安装 Docker（Ubuntu）

```bash
# 卸载旧版
sudo apt remove docker docker-engine docker.io containerd runc

# 安装依赖
sudo apt update
sudo apt install -y ca-certificates curl gnupg

# 添加 Docker 官方 GPG key
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg

# 设置仓库
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# 安装
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

# 测试
sudo docker run hello-world
```

### 14.3 Docker 核心命令

```bash
docker pull milvusdb/milvus:v2.5.6     # 拉镜像
docker images                          # 看本地镜像
docker ps                              # 看运行中的容器
docker ps -a                           # 看所有容器（含停止的）
docker logs -f milvus                  # 看日志
docker exec -it milvus bash            # 进容器
docker stop milvus                     # 停容器
docker rm milvus                       # 删容器
docker rmi image_id                    # 删镜像

# docker compose
docker compose up -d                   # 后台启动
docker compose down                    # 停止并删除
docker compose ps                      # 看状态
docker compose logs -f milvus          # 看日志
```

### 14.4 部署 Milvus（Docker Compose）

参见 [数据库.md](./数据库.md)：

```bash
wget https://github.com/milvus-io/milvus/releases/download/v2.5.6/milvus-standalone-docker-compose.yml -O docker-compose.yml
sudo docker compose up -d
curl http://localhost:9091/health       # 验证
```

---

## 十五、与本项目结合的实战场景

### 15.1 场景 1：Linux 服务器上跑 Milvus + 整个 RAG

```bash
# 1. SSH 登录服务器
ssh user@server_ip

# 2. 拉取项目
git clone <your_repo>.git
cd RAG_PROJECT

# 3. 创建虚拟环境
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt

# 4. 启动 Milvus（Docker 方式）
sudo docker compose up -d

# 5. 配置 .env
cat > .env <<EOF
OPENAI_API_KEY=sk-xxxx
ZHIPU_API_KEY=xxxx
EOF

# 6. 构建向量库
python documents/write_milvus.py

# 7. 后台启动 Gradio 服务
nohup python graph2/graph_gradio.py > app.log 2>&1 &

# 8. 查看日志
tail -f app.log
```

### 15.2 场景 2：远程调试 MCP 服务

[test_mcp/mcp_server.py](../test_mcp/mcp_server.py) 启动在 8000 端口：

```bash
# 服务器端
nohup python test_mcp/mcp_server.py > mcp.log 2>&1 &

# 检查端口
ss -tulnp | grep 8000

# 客户端测试
curl http://server_ip:8000/sse
```

### 15.3 场景 3：多进程批量入库

[documents/write_milvus.py](../documents/write_milvus.py) 使用 `multiprocessing`：

```bash
# 放到后台跑，日志输出
nohup python documents/write_milvus.py > milvus_write.log 2>&1 &

# 实时追踪
tail -f milvus_write.log

# 查看进程
ps aux | grep write_milvus
```

### 15.4 场景 4：磁盘空间不足

```bash
# 看磁盘
df -h

# 看大文件
du -sh /* 2>/dev/null | sort -h

# 清理 Docker
sudo docker system prune -a

# 清理日志
sudo journalctl --vacuum-time=7d

# 清理 pip 缓存
pip cache purge
```

### 15.5 场景 5：监控 GPU（深度学习）

```bash
# 安装（一次性）
pip install nvidia-ml-py3

# 实时监控
watch -n 1 nvidia-smi           # 每秒刷新

# Python 中调用
python -c "import torch; print(torch.cuda.is_available())"
```

---

## 十六、学习路线图

### 阶段 1：入门（1~2 周）

- [ ] 安装 Ubuntu（虚拟机或 WSL2）
- [ ] 熟悉终端 + 30 个常用命令
- [ ] 文件权限（chmod / chown）
- [ ] 文本三剑客（grep / sed / awk）
- [ ] SSH 远程登录

### 阶段 2：进阶（2~3 周）

- [ ] Shell 脚本编写
- [ ] systemd 服务管理
- [ ] Python 环境管理（venv / conda）
- [ ] 进程管理（ps / kill / nohup）
- [ ] 网络与防火墙

### 阶段 3：实战（3~4 周）

- [ ] Docker 入门
- [ ] docker compose 多容器编排
- [ ] 部署 Milvus / Ollama / vLLM
- [ ] GPU 驱动 + CUDA + cuDNN
- [ ] 监控（nvidia-smi / htop / prometheus）

### 阶段 4：高级（持续）

- [ ] K8s（Kubernetes）
- [ ] CI/CD（GitHub Actions / GitLab CI）
- [ ] Nginx 反向代理 + HTTPS
- [ ] 日志系统（ELK / Loki）
- [ ] 性能调优（perf / strace）

---

## 十七、常见问题 FAQ

### Q1：忘记 sudo 密码？

```bash
# 重启进入 recovery 模式，passwd root
```

### Q2：终端乱码？

```bash
export LANG=en_US.UTF-8
export LC_ALL=en_US.UTF-8
```

永久生效：写入 `~/.bashrc` 的末尾，然后 `source ~/.bashrc`。

### Q3：磁盘写满了怎么办？

```bash
df -h                    # 看哪个盘满了
du -sh /* 2>/dev/null | sort -h | tail -20    # 找大文件
# 常见大文件：日志、Docker 镜像、conda/pip 缓存
```

### Q4：Docker 拉镜像超时？

```bash
sudo mkdir -p /etc/docker
sudo tee /etc/docker/daemon.json <<-'EOF'
{
  "registry-mirrors": ["https://docker.mirrors.ustc.edu.cn"]
}
EOF
sudo systemctl restart docker
```

### Q5：忘记进程 PID？

```bash
ps aux | grep python
pgrep python
pgrep -f "graph_gradio"
```

### Q6：让脚本开机自启？

```bash
sudo crontab -e
# 添加
@reboot /home/user/start.sh
```

### Q7：中文文件名乱码？

```bash
# 安装中文 locale
sudo apt install -y language-pack-zh-hans
sudo locale-gen zh_CN.UTF-8
```

### Q8：`Permission denied` 错误？

```bash
chmod +x script.sh          # 给执行权限
sudo chown -R $USER:$USER dir/   # 拿回所有权
```

### Q9：远程连接 Linux 复制粘贴不方便？

- Windows：装 **MobaXterm**（自带 SSH + SFTP 图形界面）
- macOS：装 **Termius**
- 终端内：用 `Ctrl+Shift+C` / `Ctrl+Shift+V`

### Q10：磁盘 IO 高、CPU 占用 100%？

```bash
top           # 看哪个进程吃 CPU
iostat -x 1   # 看 IO（需安装 sysstat）
```

---

## 🎯 速查表（建议截图保存）

```
┌──────────────────────────────────────────┐
│  Linux 新手必背命令（每天敲一遍）          │
├──────────────────────────────────────────┤
│  ls -lh      看文件大小                   │
│  cd ~        回家目录                     │
│  pwd         我在哪                       │
│  cat file    看文件                       │
│  grep "x" f  找关键字                     │
│  ps aux|grep py  找进程                  │
│  kill -9 PID  杀进程                      │
│  chmod +x f  给执行权限                   │
│  sudo cmd    提权                         │
│  man cmd     看帮助                       │
│  history     历史命令                     │
│  tab 键      自动补全                     │
│  Ctrl+C      中断                         │
│  Ctrl+L      清屏                         │
│  ↑/↓         翻历史                       │
└──────────────────────────────────────────┘
```

---

> 📖 配套阅读：[数据库.md](./数据库.md)（Milvus 在 Linux 上的部署详解）、[各概念名词用法说明.md](./各概念名词用法说明.md)（向量数据库术语）。
