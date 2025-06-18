### 为什么需要 Git Stash？

设想一下这个场景：你正在一个分支上开发一个新功能，写了一半，突然接到一个紧急任务，需要你立刻切换到另一个分支去修复一个 Bug。但你现在手上的代码还没写完，也不想提交一个半成品。

这时，`git stash` 就派上用场了。它能帮你：

* **快速切换分支：** 让你在未完成当前工作的情况下，安全地切换到其他分支处理紧急事务。
* **保持工作区干净：** 在执行一些需要干净工作区的操作（如 `git pull`、`git rebase`、`git reset` 等）时，提前保存你的修改。
* **保存临时修改：** 当你有一些不想立即提交但又不想丢失的临时修改时。

---

### Git Stash 的常用命令

下面是一些最常用的 `git stash` 命令：

#### 1. `git stash` 或 `git stash save <message>`

* **作用：** 将你当前工作区和暂存区的所有未提交修改保存起来。
* **细节：**
  * **工作区修改：** 默认会保存所有修改过的文件。
  * **暂存区修改：** 默认也会保存你已 `git add` 但未提交的文件。
  * **未跟踪文件：** 不会保存新增的、未被 Git 跟踪的文件（即 `git status` 中显示为 Untracked files 的文件）。
  * 你可以选择添加一条描述信息，方便以后区分不同的 stash。例如：`git stash save "正在开发新功能 A 的一半"`。

#### 2. `git stash list`

* **作用：** 显示所有已保存的 stash 列表。
* **细节：** 每个 stash 都会有一个索引，例如 `stash@{0}` 是最新的，`stash@{1}` 是次新的，以此类推。

#### 3. `git stash apply [stash@{n}]`

* **作用：** 将指定 stash 的修改应用到当前分支的工作区，但**不删除** stash 列表中的该条记录。
* **细节：**
  * 如果你不指定 `[stash@{n}]`，默认会应用最新的那个 stash（即 `stash@{0}`）。
  * 应用后，你的工作区会恢复到 stash 时的状态，文件会被修改，但它们仍然是未暂存的状态。
  * 你可以将同一个 stash 应用到不同的分支上。

#### 4. `git stash pop [stash@{n}]`

* **作用：** 将指定 stash 的修改应用到当前分支的工作区，并且**从 stash 列表中删除**该条记录。
* **细节：**
  * 这是 `apply` 和 `drop` 的组合操作。
  * 如果你不指定 `[stash@{n}]`，默认会应用并删除最新的那个 stash。
  * 通常，当你确定不再需要某个 stash 时，使用 `pop` 更方便。

#### 5. `git stash drop [stash@{n}]`

* **作用：** 从 stash 列表中删除一个指定的 stash。
* **细节：**
  * 如果你不指定 `[stash@{n}]`，默认会删除最新的那个 stash。
  * 这个操作是不可逆的，一旦删除，stash 中的修改就找不回来了，所以请谨慎使用。

#### 6. `git stash clear`

* **作用：** 删除所有已保存的 stash。

#### 7. `git stash show [stash@{n}]`

* **作用：** 显示指定 stash 的修改概览（默认显示最新的 stash）。
* **细节：** 它会显示哪些文件被修改了。如果你想看具体的修改内容（diff），可以使用 `git stash show -p [stash@{n}]`。

---

### 示例用法

1. **保存修改：**
   **Bash**

   ```
   # 你正在 dev 分支上写代码，但有紧急 bug 要修复
   # 先把当前修改保存起来
   git stash save "开发功能A到一半，紧急修复bug"
   ```
2. **切换并修复 Bug：**
   **Bash**

   ```
   # 切换到 main 或 hotfix 分支
   git checkout main
   # 修复 bug，提交代码，推送到远程仓库
   # ...
   ```
3. **回到原分支并恢复修改：**
   **Bash**

   ```
   # 切换回 dev 分支
   git checkout dev
   # 查看有哪些 stash
   git stash list
   # 假设最新的 stash 就是你想要的
   git stash pop
   # 现在你就可以继续开发功能A了
   ```

---

### 注意事项

* **未跟踪文件：**`git stash` 默认不会保存未跟踪文件。如果你想保存它们，可以使用 `git stash -u` 或 `git stash --include-untracked`。
* **忽略文件：** 已经添加到 `.gitignore` 的文件不会被 stash。
* **冲突：** 如果你在应用 stash 时，当前工作区有和 stash 内容冲突的修改，Git 会提示冲突，你需要手动解决。
* **误操作恢复：** 如果你不小心 `pop` 或 `drop` 了某个 stash，理论上可以通过 `git reflog` 找到对应的 commit 哈希值来恢复，但操作相对复杂，建议谨慎使用 `pop` 和 `drop`。

Git Stash 是一个非常实用的工具，掌握它能让你的 Git 工作流更加灵活和高效。
