---
template: main.html
tags:
  - Git
---

# Git 常用命令

## 撤销

|                                    目的                                    |                           备注                           |           解决方案            |
| :------------------------------------------------------------------------: | :------------------------------------------------------: | :---------------------------: |
|                       舍弃工作目录中对一个文件的修改                       |                 修改的文件未被暂存/提交                  |   git checkout --_filename_   |
|                       舍弃工作目录中所有未保存的变更                       |                   文件已暂存，但未提交                   |       git reset --hard        |
|                 合并与某个特定提交（但不含）之间的多个提交                 |                                                          |        reset _commit_         |
|                   移除所有未保存的变更，包含未跟踪的文件                   |                    修改的文件未被提交                    |           reset -fd           |
| 移除所有已暂存的变更和在某个提交之前提交的工作，但不移除工作目录中的新文件 |                                                          |     reset --hard _commit_     |
|            移除之前的工作，但完整保留提交历史记录（前进式回滚）            |             分支已经被发布，工作目录是干净的             |         revert commit         |
|                     从分支历史记录中移除一个单独的提交                     | 修改的文件已经被提交，工作目录是干净的，分支尚未进行发布 | rebase --interactive _commit_ |
|                     保留之前的工作，但以另一个提交合并                     |                 选择 squash（压缩）选项                  | rebase --interactive _commit_ |