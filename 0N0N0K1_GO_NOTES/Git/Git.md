# 1. 版本控制
### 分布式版本控制系统
客户端并不只提取最新版本的文件快照， 而是把代码仓库完整地镜像下来，包括完整的历史记录。 这么一来，任何一处协同工作用的服务器发生故障，事后都可以用任何一个镜像出来的本地仓库恢复
![[Pasted image 20260914215935.png|443]]

# 2. 文件状态

未跟踪文件一般不会被git操作影响

git status 查看文件详细状态
git status -s  / git status --short 得到格式紧凑的状态

![[Pasted image 20260914223430.png]]

# 3. 本地操作
![[Pasted image 20260915093633.png]]

# 4. 分支操作

```
//初始
main:    A --- B --- C
              \         
feature:       D --- E*

//main 合并完成的 feature 分支
main:    A --- B --- C --- F*
              \         /
feature:       D --- E

//feature 变基为 main 的 c 版本,main可以通过merge到E',与上面合并等效
main:    A --- B --- C  <-main       
                      \        
feature:               D' --- E'*
```
1. 切换分支的时候，Git 会重置你的工作目录，使其看起来像回到了你在那个分支上最后一次提交的样子, 但不会改变暂存区
    - 注意==切换前原分支的内容是否提交，否则再切回去就无了==
    - 注意==切换前暂存区是否有文件，避免切换后载入污染==
  2. 变基可以使分支图更简洁
    - 只对尚未推送或分享给别人的本地修改执行变基操作清理历史， 从不对已推送至别处的提交执行变基操作
3. 合并分支前检查是否要合并的对象是否已提交，在最新提交记录上

|     功能     |                                          命令                                          |
| :--------: | :----------------------------------------------------------------------------------: |
|    创建分支    |                                 git branch < name >                                  |
|   查看本地分支   |                                      git branch                                      |
|   查看远程分支   |                                    git branch -r                                     |
|  重命名当前分支   |                              git branch -m  < newName >                              |
| 删除（已合并）分支  |                                git branch -d < name >                                |
| 删除（未合并）分支  |                                git branch -D < name >                                |
| 当前分支合并其它分支 |                                  git merge < name >                                  |
|     变基     |                                 git rebase < name >                                  |
|  分离HEAD指针  | git checkout HEAD~< n ><br>git checkout < branch_name ><br>git checkout < hash_num > |
|            |                                                                                      |

# 5. 远程操作
![[Pasted image 20260915122921.png]]
```
# 查看远程
git remote -v                         # 查看远程仓库地址
git remote show origin                # 查看 origin 详细信息

# 添加 / 修改 / 删除远程
git remote add origin <url>           # 添加远程仓库
git remote rename origin upstream     # 重命名远程
git remote remove origin              # 删除远程

# 抓取更新，不合并
git fetch origin                      # 抓取 origin 更新
git fetch origin main                 # 抓取指定分支

# 拉取并合并
git pull origin main                  # fetch + merge
git pull --rebase origin main         # fetch + rebase

# 推送
git push origin local:remote          # 本地 local 分支推送到远程 remote
git push --all origin                 # 推送所有本地分支
git push origin --tags                # 推送所有标签
git push origin v1.0                  # 推送指定标签
git push origin --delete tag v1.0     # 删除远程标签

```