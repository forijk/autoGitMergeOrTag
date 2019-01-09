# Jenkins 配合 GitLab 实现分支的自动合并、自动创建 Tag

## 背景

### GitFlow工作流简介

**Gitflow**工作流定义了一个围绕项目发布的严格分支模型，它会相对复杂一点，但提供了用于一个健壮的用于管理大型项目的框架，非常适合用来管理大型项目的发布和维护。 贯穿整个开发周期，**master**和**develop**分支是一直存在的，master分支可以被视为稳定的分支， 而develop分支是相对稳定的分支，特性开发会在**feature**分支上进行，发布会在**release**分支上进行，而bug修复则会在**hotfix**分支上进行，这样也有效避免了不同类型的开发工作在代码层级的耦合和干扰。

### GitFlow工作流优化

- hotfix和release的结果都要合并到master和develop中，为什么？因为它们的修改结果将持续影响这后续的开发和维护，必须合并以保证代码的一致性。

- 当线上项目需要版本回退，或者需要简单记录迭代版本时，我们常在master分支上打上 Tag 标签，例如：

	- 功能发布：release\_20190101\_当月版本数
	- BUG修复：hotfix\_20190101\_修复次数

本文基于GitFlow工作流，将利用Jenkins配合GitLab实现以下**自动化任务**：

- master分支代码更新后，自动将代码合并到develop分支
- master分支代码更新后，自动在master分支提交中打 tag

## Jenkins自动任务Job的创建
>Jenkins是一个用Java编写的开源的持续集成工具，可以与Git打通，监听Git的merge, push事件，触发执行Jenkins的指定任务(job)，例如执行单元测试。更多的是：当代码变更时可以触发打包部署、性能测试、接口测试、监控、日志分析等。项目发布的任何一个环节都可自动完成，无需太多的人工干预，有利于减少重复过程以节省时间和工作量等。

### 如何创建Jenkins Job
下面列出自动任务Jenkins Job的创建过程，供参考。创建过程如下：

- **首先**，创建一个任务，输入名称，选择“构建一个自由风格的软件项目” 确定即可。

![](https://img2018.cnblogs.com/blog/1414425/201901/1414425-20190110001016094-2027726449.png)

- **General：**填写项目名称及描述；Label Expression 是在Jenkins admin 中配置的节点（container, node），一个Label Expression 是一组docker，当多个不同的Job同时执行的时候，可以实现并行。

![](https://img2018.cnblogs.com/blog/1414425/201901/1414425-20190110001035403-1848522405.png)

- **源码管理：**在Repository URL中填写GitLab中的项目地址；Branches to build此处的Branch Specifier是触发Jenkins Job时由Jenkins自动拉取的代码的分支，可以填写一个固定的指定分支，如master，也可以写正则表达式。另外，也可以填写${gitlabSourceBranch}。如果填写${gitlabSourceBranch}，表示从git读取Merge Request的源分支，使用该变量，则不能手工点击Job的“立即构建”执行job了，因为读取不到这个变量只能通过git push事件触发了。具体可以根据应用场景选择。

![](https://img2018.cnblogs.com/blog/1414425/201901/1414425-20190110001055782-140519907.png)

- **构建触发器：**下面截图中指定的是：仅当产生 push 事件时且目标分支为master时触发此job。这里可以根据需求来具体配置。Filter branches by name：仅当目标分支为 master 时触发Job。

![](https://img2018.cnblogs.com/blog/1414425/201901/1414425-20190110001108191-1852265305.png)

- **构建环境 & 构建：**构建环境勾选第一项指在每次构建开始之前先清空工作空间；构建中的Execute shell Command 中就可以配置我们自动化脚本，我们可以把这些脚本放到项目根目录当中，也可以放到一个GitLab仓库当中，来统一管理脚本（文末附脚本示例）。

![](https://img2018.cnblogs.com/blog/1414425/201901/1414425-20190110001117877-612823650.png)
![](https://img2018.cnblogs.com/blog/1414425/201901/1414425-20190110001124787-637956332.png)

- **构建后操作（Editable Email Notification）**：用于配置邮件提醒。

	- Disable Extended Email Publisher：勾选后，邮件就不发送；
	- Project Recipient List：收件人地址；多个收件人邮件地址用逗号进行分割；
	- Project Reply-To List：允许回复人的地址；默认为$DEFAULT_REPLYTO；
	- Content Type：邮件文档的类型，可以设置HTML等格式；
	- Default Subject：默认邮件标题；也可以使用$DEFAULT_SUBJECT；
	- Default Content：默认邮件内容，这里可以写HTML文件，引用Jenkins内部的一些变量；也可以使用默认内容：$DEFAULT_CONTENT；
	- Attach Build Log：发送的邮件是否包含日志。

	**Triggers 中的配置需要注意下，一般配置为Job执行失败的时候发送邮件**

![](https://img2018.cnblogs.com/blog/1414425/201901/1414425-20190110001140935-1294496622.png)
![](https://img2018.cnblogs.com/blog/1414425/201901/1414425-20190110001148540-770025116.png)

## Jenkins Job 如何与 Git 关联

在GitLab项目的Settings中找到如下图的配置：勾选“Active”，指定在Git Push 或 mr  创建/更新/合并时触发指定的 Jenkins url，Project name 为Jenkins 中配置的Job名称，用户名、密码是jenkins的账号和密码。

![](https://img2018.cnblogs.com/blog/1414425/201901/1414425-20190110001203478-49653835.png)

## 整体步骤梳理

1. GitLab上准备一个项目工程； 
2. 安装Jenkins以及相关的GitLab插件；
3. Jenkins配置GitLab访问权限；
4. Jenkins上创建一个Job，对应步骤1中的项目工程；
5. GitLab项目中配置Jenkins；
6. 修改项目工程的源码，并提交到GitLab上； 
7. 检查Jenkins的构建任务是否会触发配置的脚本。

## 脚本示例
### 以下为master分支代码更新后，自动在master分支提交中打 tag的脚本示例，仅供参考：
（ Tag 标签命名规则: release\_当前日期\_当月版本\_当季度版本\_当年版本 ）

```
#!/bin/sh
echo **********************************Start********************************
date
# 获取最近一次远程 master 提交的 commit id
sha1=`git rev-parse remotes/origin/master^{commit}`
# 获取姓名及邮箱，来配置git提交者信息
name=`git show --pretty=%an $sha1 | awk 'NR==1{print}'`
email=`git show --pretty=%ce $sha1 | awk 'NR==1{print}'`
echo '################# 当前提交人信息:'
echo $name 
echo $email 
git config --global user.name $name
git config --global user.email $email

# 获取 merge 的源分支前缀
function getOriginPrefix(){
  # 获取分支所属
  info_sha1=`git show $sha1 | grep 'Merge:' | cut -d' ' -f3`
  info_branch=`git branch -r --contains $info_sha1`
  # 判断是否 hotfix 分支
  isHotfix=`echo "${info_branch}" | grep 'origin/hotfix'`
  if [ -n "$isHotfix" ]; then 
    echo 'hotfix'
  else
    echo 'release'
  fi
}
originBra=$(getOriginPrefix)
echo '################# 获取的源分支前缀为:' $originBra

# 获取最近一次创建的标签
latestTag=`git for-each-ref --sort=-taggerdate --format "%(tag)" refs/tags | grep $originBra | head -n 1`
# 获取最近标签的年
latestYear=`echo "${latestTag}" | awk -F_ '{print substr($2,1,4)}'`
# 获取最近标签的月
latestMonth=`echo "${latestTag}" | awk -F_ '{print substr($2,5,2)}'`
# 获取最近标签的季度
latestQuarter=`echo "${latestMonth}" | awk '{print int(($0-1)/3)+1}'`

# 获取当年
currentYear=`date +%Y`
# 获取当月
currentMonth=`date +%m`
# 获取当日
currentDay=`date +%Y%m%d`
# 获取当前季度
currentQuarter=`echo $currentMonth | awk '{print int(($0-1)/3)+1}'`

# 计算当月版本号
if [ $latestMonth -eq $currentMonth ]; then 
  currentMonthVersion=`echo "${latestTag}" | awk -F_ '{print $3+1}'`
else
  currentMonthVersion='1'
fi

# 计算当季度版本号
if [ $latestQuarter -eq $currentQuarter ]; then 
  currentQuarterVersion=`echo "${latestTag}" | awk -F_ '{print $4+1}'`
else
  currentQuarterVersion='1'
fi

# 计算当年版本号
if [ $latestYear -eq $currentYear ]; then 
  currentVersion=`echo "${latestTag}" | awk -F_ '{print $5+1}'`
else
  currentVersion='1'
fi

# 获取最终标签名 
newVersion=$originBra'_'$currentDay'_'$currentMonthVersion'_'$currentQuarterVersion'_'$currentVersion

# 创建标签
git tag -a $newVersion -m '提交人: '$name
git push origin --tags
newTag=`git tag -l | grep $newVersion`
echo '################# 最近创建的标签为:' $latestTag
echo '################# 自动计算的标签为:' $newVersion
echo '################# 自动创建的标签为:' $newTag
echo **********************************End**********************************
```

## 参考资料

[Jenkins 官方文档](https://jenkins.io/zh/doc/)

[GitLab 官方文档](https://docs.gitlab.com/ce/ci/yaml/README.html)

[GitFlow 工作流](https://www.jianshu.com/p/bdb05232dbc1)

#### 转载请标注出处：https://github.com/forijk/autoGitMergeOrTag
