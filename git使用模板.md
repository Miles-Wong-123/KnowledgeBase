# git使用模板

# **进入项目目录**

```
cd /你的/项目路径
```

说明：cd 就是“进入文件夹”。你后面的 Git 命令都要在项目根目录执行，不然 Git 不知道你要操作哪个项目。

# **初始化 Git 仓库**

```
git init
```

说明：这一步是告诉电脑：“从现在开始，这个文件夹要交给 Git 管理。”

执行后，项目里会多出一个隐藏的 .git 目录，它就是 Git 的数据库和记录本。

# **绑定远程** **GitHub** **仓库**

```
git remote add origin 你的GitHub仓库地址
```

例子：

```
git remote add origin ``https://github.com/``你的用户名/你的仓库名.git
```

说明：origin 可以理解成“远程仓库的昵称”。

这一步是把你本地项目和 GitHub 上的仓库连起来，后面 push 才知道往哪里传。

# **查看当前 Git 状态**

```
git status
```

说明：这是最常用的检查命令。

它会告诉你：

1. 哪些文件是新文件
2. 哪些文件被修改了
3. 哪些文件已经准备提交
4. 当前在哪个分支

你可以把它理解成“看当前进度表”。

# **新建 .gitignore**

```
touch .gitignore
```

说明：.gitignore 是“忽略名单”。

你不想上传到 GitHub 的东西，比如：

1. 编译产物
2. 本地配置
3. 密钥
4. IDE 配置 都应该写在这里。

比如 Java 项目常见内容可以写成：

```
target/ ``.idea/ ``*.iml ``.env ``application-local.yml
```

说明：这样 Git 就会自动忽略这些文件，不会轻易把它们提交上去。

查看.gitignore文件的方式：

```
cat .gitignore
```

# **把文件加入暂存区**

```
git add .
```

或者更稳一点：

```
git add pom.xml ``git add src ``git add README.md
```

说明：git add 的意思不是“已经提交了”，而是“把这些文件放进待提交清单”。

你可以把它理解成：先把要寄出的东西装进包裹箱。

git add . 是“把当前目录下所有改动都放进去”。

但更推荐按文件/目录加，这样不容易把不该传的东西一起传上去。

# **再检查一遍**

```
git status
```

说明：这一步非常重要。

它是确认你刚才 git add 进去的内容是不是你真正想提交的内容。

# **查看即将提交的改动**

```
git diff --cached
```

说明：这会显示“已经暂存、准备提交”的具体内容。

你可以理解成“寄出去之前最后看一眼包裹里装了什么”。

如果只想看概要，可以用：

```
git diff --cached --stat
```

# **提交到本地仓库**

```
git commit -m "写一句清楚的提交说明"
```

例子：

```
git commit -m "Add chat page and update project artifact name"
```

说明：commit 就是“生成一次正式存档”。

它还没有上传到 GitHub，只是先记到你本地的 Git 历史里。

-m 后面的文字是“这次改了什么”的一句话说明。

尽量写清楚，不要总是写 update、fix 这种太模糊的话。

# **看提交历史**

```
git log --oneline
```

说明：这是看历史记录。

你可以把它理解成“看这个项目之前都提交过什么版本”。

# **第一次推送到** **GitHub**

```
git branch -M main ``git push -u origin main
```

说明：

1. git branch -M main 是把当前主分支名字统一成 main
2. git push -u origin main 是把本地 main 分支传到 GitHub 上

这里的 -u 意思是“建立跟踪关系”。

做完以后，你下次再推送就可以直接用：

```
git push
```

# 以后更新推送到github

**看当前状态**

```Plain
git status
```

说明：先看一眼你改了什么。

它会告诉你：

1. 哪些文件改了
2. 哪些是新文件
3. 哪些已经准备提交

你可以把它理解成：

“先检查一下这次准备寄出去的东西有哪些。”

**把改动加入暂存区**

```Plain
git add .
```

说明：把这次改动放进“待提交清单”。

这里的 . 意思是“当前目录下所有改动”。

如果你不想一次加全部，也可以只加某些文件：

```Plain
git add src
git add pom.xml
```

通俗理解：

这一步像是“把这次要寄的文件装进箱子里”。

**提交到本地 Git 记录**

```Plain
git commit -m "写这次更新的说明"
```

例子：

```Plain
git commit -m "Add streaming chat page"
```

说明：这一步是把你的改动正式记到本地 Git 历史里。 注意：这时候还**没有上传到** **GitHub**，只是先存在你自己电脑里。

通俗理解：

这一步像是“给这次修改拍一张带标题的快照”。、

**先把** **GitHub** **上最新内容拉下来**

```Plain
git pull --rebase origin main
```

说明：这一步很重要。

因为你本地改了代码，GitHub 上也可能已经有更新。

所以推送前，先把远端最新内容拿下来，再把你的提交接到后面。

为什么推荐 --rebase？

因为这样提交历史会更整齐，不容易多出乱七八糟的 merge 记录。

通俗理解：

这一步像是“先确认远端有没有别人已经放进去的新内容，再把你的内容接上”。

**上传到 GitHub**

```Plain
git push origin main
```

说明：这一步才是真正把本地提交传到 GitHub。

通俗理解：

前面只是“整理、打包、确认”，这一步才是“真正寄出去”。