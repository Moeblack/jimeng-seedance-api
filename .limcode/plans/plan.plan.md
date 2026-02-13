### 1. 软重置分支历史 (Soft Reset)
* 使用 PowerShell 运行 `git reset HEAD~2`，保留所有代码改动到工作区，但撤销包含脏文件的两次提交。

### 2. 还原或清理不应该提交的文件
* `git checkout -- .gitignore` 还原忽略文件到主分支的干净状态。
* `Remove-Item .gitattributes` 彻底删除强制换行符规则文件，避免产生全盘 CRLF 错乱警告。
* 删除之前可能残留的本地个人文件，如 `index.html`。

### 3. 修正 README 文档中的不实描述
* 使用 `apply_diff` 从 `README.md` 和 `README.CN.md` 中删除关于内置 `index.html` 异步界面的引导话术，以防误导新用户。

### 4. 重新提交整洁的代码
* 将真正有用的特性代码添加到暂存区：`git add README.CN.md README.md src/`。
* 重新执行第一次提交 (Async Task 核心接口)。
* 重新执行第二次提交 (Audio 上传与全能引用支持)。
* 这样生成的分支就是最标准、无毒副作用、可以直接提 PR 的版本。

## TODO LIST

<!-- LIMCODE_TODO_LIST_START -->
- [ ] Revert .gitignore and delete .gitattributes to solve CRLF issues.  `#clean_local_configs`
- [ ] Remove 'index.html' references from README.md and README.CN.md.  `#fix_readmes`
- [ ] Stage changes and recreate clean commits using powershell.  `#git_recommit`
- [ ] Soft reset recent 2 commits to working directory.  `#git_soft_reset`
<!-- LIMCODE_TODO_LIST_END -->
