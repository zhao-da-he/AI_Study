# Git Clone 完整操作指南

## 一、核心概念

`git clone` 用于将远程 GitHub 仓库完整复制到本地，自动创建同名文件夹、同步分支名（如 `main`）、绑定远程追踪关系，避免手动初始化带来的分支冲突、历史不相关等问题。

**优势**：
- 自动创建同名文件夹
- 自动同步分支名
- 自动绑定远程追踪关系
- 避免 `git init` 方式常见的分支名不匹配错误

---

## 二、前置准备：GitHub 上新建空仓库

1. 登录 GitHub，点击右上角 `+` → `New repository`
2. 填写仓库名，例如：`AI_Study`
3. **Description** 可选填
4. 选择 `Public`（公开）或 `Private`（私有）
5. **关键**：下方三个选项**全部不要勾选**
   - ❌ Add a README file
   - ❌ Add .gitignore
   - ❌ Choose a license
6. 点击 `Create repository`

创建完成后会得到仓库地址，例如：

```
https://github.com/zhao-da-he/AI_Study.git
```

---

## 三、基础克隆流程

### 1. 切换到目标盘符/目录

```bash
# 切换到 D 盘根目录（直接输入盘符+英文冒号即可）
D:

# 如果想放到 D 盘指定文件夹，可先创建并进入目标目录
mkdir D:\MyProjects
cd D:\MyProjects
```

### 2. 执行克隆命令

```bash
# 替换为你的 GitHub 仓库地址，执行后会自动创建同名文件夹
git clone https://github.com/zhao-da-he/AI_Study.git
```

克隆完成后，目标目录下会出现 `AI_Study` 文件夹，包含仓库所有文件、完整提交历史和本地 `main` 分支。

### 3. 进入克隆的仓库目录

```bash
cd AI_Study
```

---

## 四、文件上传标准流程

### 1. 把文件复制到仓库文件夹

把需要上传的文件/文件夹直接复制/拖入克隆下来的 `AI_Study` 文件夹中。

### 2. 执行 Git 提交三步曲

```bash
# 1. 添加所有变更到暂存区（. 代表当前目录所有内容）
git add .

# 2. 提交到本地仓库，备注本次修改内容
git commit -m "上传测试文件"

# 3. 推送到 GitHub（克隆已自动绑定追踪关系，可直接简写）
git push
```

**说明**：
- 第一次推送如果提示 `set upstream`，可以执行 `git push -u origin main`
- 后续推送直接 `git push` 即可

---

## 五、推送时的登录问题

GitHub 已经不支持密码登录，推送时会弹窗要求输入凭据：

- **Username**：填你的 GitHub 用户名
- **Password**：填 **Personal Access Token (PAT)**，不是 GitHub 密码

### 如何生成 PAT：

1. GitHub 右上角头像 → `Settings`
2. 左侧最下方 `Developer settings`
3. `Personal access tokens` → `Tokens (classic)`
4. `Generate new token` → `Generate new token (classic)`
5. Note 随便填，比如 `my-pc`
6. Expiration 选有效期（建议 90 days 或 No expiration）
7. **勾选 `repo` 这个权限**（必须）
8. 拉到最下面点 `Generate token`
9. **复制生成的 token**（只显示一次，关掉页面就再也看不到了）

把这个 token 当密码粘贴进弹窗即可。配置凭据记忆避免每次输入：

```bash
git config --global credential.helper manager
```

---

## 六、日常使用注意事项

1. 每次上传前建议先执行 `git pull` 拉取远程最新内容，避免推送冲突；
2. 克隆的仓库无需再执行 `git init`、`git remote add` 等初始化操作；
3. 本地分支名默认和远程一致（如 `main`），无需额外重命名；
4. **大文件不要直接传**：单个文件超过 50MB 会报错，超过 100MB GitHub 直接拒绝。需要用 Git LFS。
5. **不要传敏感信息**：`.env`、`密码.txt`、`*.key` 等文件上传前务必删除或加进 `.gitignore`

---

## 七、常见问题

### 问题 1：克隆时报错 `folder already exists`

说明目标文件夹已存在，可删除空文件夹后重新克隆，或更换克隆路径：

```bash
# 方案 A：删除已存在的空文件夹后重新克隆
rd /s /q AI_Study
git clone https://github.com/zhao-da-he/AI_Study.git

# 方案 B：克隆到指定目录
git clone https://github.com/zhao-da-he/AI_Study.git D:\OtherFolder\AI_Study
```

### 问题 2：克隆/推送速度慢

国内访问 GitHub 经常慢，可配置 Git 代理（以 Clash 为例，端口 7890）：

```bash
git config --global http.proxy http://127.0.0.1:7890
git config --global https.proxy http://127.0.0.1:7890
```

推送完想取消代理：

```bash
git config --global --unset http.proxy
git config --global --unset https.proxy
```

或者使用国内镜像源加速：

```bash
# 使用 gitee 镜像（需要先在 gitee 导入 GitHub 仓库）
git clone https://gitee.com/你的用户名/AI_Study.git
```

### 问题 3：`error: src refspec main does not match any`

原因：本地没有 `main` 分支，也没提交。

```bash
git branch                # 看本地有什么分支
git status                # 看有没有提交

git add .
git commit -m "first commit"
git branch -M main
git push -u origin main
```

### 问题 4：`failed to push some refs to ...`

原因：远程仓库有本地没有的提交。

```bash
git pull origin main --rebase
git push origin main
```

---

## 八、完整命令速查

```bash
# ========== 1. 克隆空仓库 ==========
D:
mkdir D:\MyProjects
cd D:\MyProjects
git clone https://github.com/zhao-da-he/AI_Study.git
cd AI_Study

# ========== 2. 复制文件 ==========
# （手动把要上传的文件复制到这个目录里）

# ========== 3. 提交推送 ==========
git add .
git commit -m "上传文件"
git push

# ========== 4. 以后再修改文件 ==========
git pull                    # 先拉取最新内容
git add .
git commit -m "描述你改了什么"
git push
```

---

## 九、对比：clone 方式 vs init 方式

| 方式 | 优点 | 缺点 |
|------|------|------|
| `git clone` | 本地远程自动关联，分支不会出错 | 多一步克隆操作 |
| `git init` | 直接在现有文件夹上操作 | 需要手动配 remote，容易分支名不匹配 |

**推荐新手用 `git clone` 方式**，更不容易出错。
