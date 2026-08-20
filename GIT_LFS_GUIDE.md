# Git LFS 团队使用指南

## 1. 什么是 Git LFS

Git LFS（Large File Storage）是 Git 的扩展工具，用于管理大文件。

核心机制：
- Git 仓库中只保存一个**指针文件**（几行文本），记录文件 hash 和大小
- 真实文件内容存储在 **LFS 服务器**（GitHub/GitLab 等提供的独立存储）
- 本地 `.git/lfs/objects/` 作为缓存，避免重复下载

指针文件示例：
```text
version https://git-lfs.github.com/spec/v1
oid sha256:70292b0c0e0f8e8f8e8f8e8f8e8f8e8f8e8f8e8f8e8f8e8f8e8f8e8f8e8f8e
size 1843200
```

## 2. 为什么需要 LFS

Git 本身不适合直接管理二进制大文件：

| 问题 | 说明 |
|------|------|
| 仓库膨胀 | 每次修改二进制文件都会生成完整副本，历史记录急剧增大 |
| 操作变慢 | `git clone`、`git log`、`git diff` 都会受影响 |
| 冲突难处理 | 二进制文件无法像文本一样 diff/merge |

建议用 LFS 管理的文件类型：
- 图片：`*.png`、`*.jpg`、`*.svg`、`*.gif`
- 设计稿：`*.psd`、`*.ai`、`*.sketch`
- 音视频：`*.mp4`、`*.mp3`、`*.wav`
- 文档：`*.pdf`、`*.docx`、`*.pptx`
- 模型/数据：`*.bin`、`*.model`、`*.vsg`

## 3. 安装与初始化

### 3.1 安装 Git LFS

```bash
# macOS
brew install git-lfs

# Ubuntu/Debian
sudo apt-get install git-lfs

# Windows
# 下载安装包：https://git-lfs.com/
```

验证安装：
```bash
git lfs version
```

### 3.2 在仓库中启用 LFS

```bash
git lfs install
```

每个本地仓库只需执行一次。

## 4. 追踪文件

### 4.1 添加追踪规则

```bash
# 追踪所有 png 文件
git lfs track "*.png"

# 追踪某个目录下的所有 psd
git lfs track "designs/*.psd"

# 追踪单个文件
git lfs track "assets/logo.svg"
```

执行后会生成/修改 `.gitattributes` 文件。

### 4.2 重要：规则语法

`.gitattributes` 中**每行一个规则**，不支持逗号分隔：

```text
# ✅ 正确
*.png filter=lfs diff=lfs merge=lfs -text
*.svg filter=lfs diff=lfs merge=lfs -text

# ❌ 错误
*.png,*.svg filter=lfs diff=lfs merge=lfs -text
```

### 4.3 查看已追踪的文件

```bash
git lfs track
```

## 5. 日常使用

### 5.1 提交大文件

```bash
git add assets/image.png
git commit -m "add image via lfs"
git push
```

提交时 LFS 会自动把真实文件上传到 LFS 服务器，Git 仓库中只保留指针文件。

### 5.2 拉取代码

```bash
git pull
```

默认会自动下载 LFS 文件并替换指针。

### 5.3 跳过自动下载（只拉指针）

```bash
GIT_LFS_SKIP_SMUDGE=1 git pull
```

适合只需要代码、不需要大文件的场景。

### 5.4 手动拉 LFS 文件

```bash
# 拉当前分支所有 LFS 文件
git lfs pull

# 只拉特定文件
git lfs pull --include="*.png"

# 先下载，再手动 checkout
git lfs fetch
git lfs checkout
```

## 6. 验证 LFS 是否生效

```bash
# 查看哪些文件被 LFS 追踪
git lfs ls-files

# 查看 LFS 状态
git lfs status

# 查看某个文件是否走 LFS filter
git check-attr filter -- assets/image.png

# 查看文件真实类型
file assets/image.png
```

如果 LFS 生效，工作区中的文件是真实二进制；如果失效，Git 仓库中存的是完整二进制，仓库会膨胀。

## 7. 团队推广建议

### 7.1 仓库初始化时统一配置

项目创建后立即添加 `.gitattributes`：

```text
*.png filter=lfs diff=lfs merge=lfs -text
*.jpg filter=lfs diff=lfs merge=lfs -text
*.jpeg filter=lfs diff=lfs merge=lfs -text
*.gif filter=lfs diff=lfs merge=lfs -text
*.svg filter=lfs diff=lfs merge=lfs -text
*.psd filter=lfs diff=lfs merge=lfs -text
*.ai filter=lfs diff=lfs merge=lfs -text
*.pdf filter=lfs diff=lfs merge=lfs -text
*.mp4 filter=lfs diff=lfs merge=lfs -text
*.mp3 filter=lfs diff=lfs merge=lfs -text
*.bin filter=lfs diff=lfs merge=lfs -text
*.vsg filter=lfs diff=lfs merge=lfs -text
```

### 7.2 已仓库中已有大文件怎么办

如果仓库已经存了很多二进制文件，需要迁移到 LFS：

```bash
git lfs migrate import --include="*.png,*.psd,*.mp4"
```

注意：这会**重写 Git 历史**，需要团队所有人重新 clone。

### 7.3 CI/CD 配置

在 CI 中需要安装并启用 LFS：

```bash
git lfs install
git lfs pull
```

或者使用支持 LFS 的 checkout action（如 GitHub Actions 的 `lfs: true`）。

## 8. 常见问题

### Q1: 为什么 push 成功但同事 clone 下来没有图片？

检查同事是否安装了 `git-lfs` 并执行了 `git lfs install`。

### Q2: 文件已经 add 了但没进 LFS？

`.gitattributes` 必须在 `git add` 之前存在。如果顺序错了，需要重新 add：

```bash
git add --renormalize .
```

### Q3: LFS 文件能回退版本吗？

可以，Git 会按 commit 中的指针找到对应版本的 LFS 对象。

### Q4: 删除 LFS 文件能减小仓库体积吗？

不能立即减小，因为历史记录中仍然保留指针。需要 `git lfs migrate` 或清理历史。

## 9. 注意事项

- 不要把 `.gitattributes` 和 `.git/lfs` 搞混，后者是本地缓存
- 不要手动编辑 `.git/lfs`
- 提交前养成习惯检查 `git lfs status`
- 迁移历史前务必备份并通知团队
