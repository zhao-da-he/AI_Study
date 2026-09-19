# GitHub 代码提交文件完整操作流程

> 本文档记录了从进入项目目录到成功推送文件到 GitHub 的完整操作流程，适用于正常无报错情况下的文件提交。

---

## 一、前置准备

在开始之前，请确保你已经完成了以下准备工作：

1. 已在 [GitHub](https://github.com) 上注册账号并创建好仓库
2. 已在本地安装 Git 并配置好用户名和邮箱
3. 本地项目文件已准备好

### 配置 Git 用户信息（首次使用需执行）

```bash
# 设置全局用户名（替换为你的GitHub用户名）
git config --global user.name "你的用户名"

# 设置全局邮箱（替换为你的GitHub绑定邮箱）
git config --global user.email "你的邮箱@example.com"
```

---

## 二、进入项目目录

打开命令行（Windows 下为 CMD 或 PowerShell），使用 `cd` 命令进入你要提交文件的文件夹。

```bash
# 示例：进入 Desktop 下的 docs 文件夹
cd C:\Users\Administrator\Desktop\docs
```

> **提示**：你也可以直接在文件资源管理器中，地址栏输入 `cmd` 回车，即可在当前文件夹打开命令行。

---

## 三、初始化 Git 仓库（仅首次需要）

如果你之前没有在该文件夹中初始化过 Git 仓库，需要先执行初始化命令。

```bash
# 初始化 Git 仓库（会在当前目录创建 .git 隐藏文件夹）
git init
```

初始化成功后，该文件夹就变成了一个 Git 仓库，可以开始版本控制了。

---

## 四、关联远程仓库（仅首次需要）

如果你是在本地新建的仓库，需要将其与 GitHub 上的远程仓库关联起来。

```bash
# 关联远程仓库（替换为你自己的仓库地址）
git remote add origin https://github.com/zhao-da-he/AI_Study.git
```

> **提示**：如果你已经在 GitHub 网页端创建过仓库，可以在仓库页面点击 "Code" 按钮，复制 HTTPS 链接来使用。

---

## 五、添加文件到暂存区

使用 `git add` 命令将文件添加到暂存区（Staging Area）。

### 方式一：添加所有文件

```bash
# 添加当前目录下所有文件和文件夹到暂存区
git add .
```

### 方式二：添加指定文件

```bash
# 添加单个文件（文件名可带路径）
git add 文件名.txt

# 添加指定文件夹下的所有文件
git add 文件夹名/
```

### 方式三：混合添加

```bash
# 可以多次执行 git add，分别添加不同的文件
git add file1.txt
git add file2.md
git add my_folder/
```

---

## 六、提交到本地仓库

使用 `git commit` 命令将暂存区的内容提交到本地仓库。

```bash
# 提交到本地仓库，-m 后跟提交说明（用引号包裹）
git commit -m "提交说明：描述本次提交了什么内容"
```

> **示例**：`git commit -m "上传AI学习笔记和相关项目文件"`

---

## 七、推送到 GitHub 远程仓库

使用 `git push` 命令将本地提交推送到 GitHub 远程仓库。

```bash
# 推送到远程仓库的 main 分支
git push origin main
```

> **注意**：如果你的仓库默认分支名是 `master` 而不是 `main`，请执行：
> ```
> git push origin master
> ```

---

## 八、完整流程速查（复制即用）

以下是从进入目录到成功推送的完整命令序列，可直接复制执行：

```bash
# 1. 进入项目目录
cd C:\Users\Administrator\Desktop\docs

# 2. 初始化Git仓库（仅首次需要）
git init

# 3. 关联远程仓库（仅首次需要）
git remote add origin https://github.com/zhao-da-he/AI_Study.git

# 4. 添加所有文件到暂存区
git add .

# 5. 提交到本地仓库
git commit -m "上传所有文件"

# 6. 推送到GitHub
git push origin main
```

---

## 九、日常提交文件（后续操作）

当你完成第一次提交后，后续每次只需要执行以下三步即可：

```bash
# 第一步：添加变更的文件
git add .

# 第二步：提交到本地
git commit -m "描述本次修改内容"

# 第三步：推送到远程
git push origin main
```

---

## 十、查看提交状态

在操作过程中，可以使用以下命令查看当前仓库状态：

```bash
# 查看当前有哪些文件被修改/新增（未暂存）
git status

# 查看暂存区中有哪些文件（已暂存未提交）
git status

# 查看提交历史
git log

# 查看当前分支
git branch
```

---

## 十一、常见问题速查

| 问题 | 解决方法 |
|------|----------|
| 提示不是 git 仓库 | 执行 `git init` 初始化 |
| 推送时被拒绝 | 先执行 `git pull origin main` 拉取远程更新 |
| 分支名不确定 | 执行 `git branch` 查看当前分支名 |
| 忘记配置用户名邮箱 | 执行 `git config --global user.name "用户名"` 和 `git config --global user.email "邮箱"` |
| 想撤销暂存的文件 | 执行 `git reset HEAD 文件名` |
| 想撤销未提交的修改 | 执行 `git checkout -- 文件名` |

---

## 十二、注意事项

1. **路径问题**：`git add` 后面的路径是相对于当前目录的相对路径
2. **空文件夹**：Git 不会追踪空文件夹，如需占位可在空文件夹中放一个 `.gitkeep` 文件
3. **敏感信息**：上传前确认文件中不包含密码、API 密钥等敏感信息
4. **大文件限制**：GitHub 命令行上传单个文件最大支持 100 MiB，超过需要使用 Git LFS
5. **提交说明**：建议每次提交都写清晰的说明信息，方便后续追溯修改历史
