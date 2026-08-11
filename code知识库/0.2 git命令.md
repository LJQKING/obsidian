# 🧭 一、基础配置（第一次用 Git 必备）

```bash
git config --global user.name "你的名字"
git config --global user.email "你的邮箱"
```

查看配置：

```bash
git config --list
```

---

# 📁 二、仓库初始化 / 克隆

## 初始化本地仓库

```bash
git init
```

## 克隆远程仓库

```bash
git clone https://github.com/user/repo.git
```

---

# 📌 三、基本提交流程（最常用）

## 查看状态

```bash
git status
```

## 添加文件到暂存区

```bash
git add 文件名
git add .   # 添加全部
```

## 提交代码

```bash
git commit -m "提交说明"
```

## 推送到远程

```bash
git push origin main
```

---

# 🔄 四、分支操作（非常重要）

## 查看分支

```bash
git branch
```

## 创建分支

```bash
git branch dev
```

## 切换分支

```bash
git checkout dev
```

或（推荐）

```bash
git switch dev
```

## 创建并切换

```bash
git checkout -b dev
```

## 删除分支

```bash
git branch -d dev
```

---

# 🔀 五、合并分支

## 合并到当前分支

```bash
git merge dev
```

---

# 🌍 六、远程仓库操作

## 查看远程

```bash
git remote -v
```

## 添加远程仓库

```bash
git remote add origin https://github.com/user/repo.git
```

## 拉取代码

```bash
git pull origin main
```

## 推送代码

```bash
git push origin main
```

---

# 🧠 七、撤销 / 回退（面试高频）

## 撤销工作区修改

```bash
git checkout -- 文件名
```

## 撤销暂存区

```bash
git reset HEAD 文件名
```

## 回退 commit

```bash
git reset --soft HEAD~1   # 保留代码
git reset --hard HEAD~1   # 删除代码（危险）
```

---

# 📜 八、日志查看

```bash
git log
```

简洁版：

```bash
git log --oneline
```

图形分支：

```bash
git log --oneline --graph --all
```

---

# 🧩 九、stash（临时保存代码）

```bash
git stash        # 保存
git stash list   # 查看
git stash pop    # 恢复
```

---

# 🚀 十、实战最常用一套流程

```bash
git pull

git add .

git commit -m "fix: update"

git push
```

---

# 💡 如果你是 Android 开发者建议重点掌握

- git clone / pull / push
    
- branch / merge
    
- reset / revert
    
- stash
    
- conflict 解决
    

---

如果你想，我可以帮你整理一份：

👉「Git 面试高频题 + 答案（Android / Java 面试版）」  
👉「Git 冲突解决实战教程」  
👉「GitFlow 工作流详解」

直接说就行 👍