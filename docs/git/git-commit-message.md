
## 前言

Commit message 是开发的日常操作，它可以提供更多的历史信息，方便向团队清晰准确地说明代码变更、进行代码评审，也便于后期快速定位原始需求或缺陷，还可以有效的生成 Change log，对项目的管理实际至关重要，但是实际工作中却常常被大家忽略。

## Commit Message 格式

目前，社区有多种 Commit message 的写法规范，但使用较多的是 [Angular 团队的规范](https://github.com/angular/angular.js/blob/master/DEVELOPERS.md#-git-commit-guidelines)， 继而衍生了 [Conventional Commits specification](https://www.conventionalcommits.org/en/v1.0.0/)。同时，很多配套工具也是基于此规范。

每个 commit message 均由 **header**，**body** 和 **footer** 组成。header 具有一种特殊的格式，其中包括 **type**，**scope** 和 **subject**。

它的 message 格式如下:
```
<type>(<scope>): <subject>
// 空白行
<body>
// 空白行
<footer>
```
Commit message 的任何一行都不能超过 100 个字符，这使得该消息在 GitHub 以及各种 git 工具中更易于阅读。

## Header

header 是必填的，描述主要修改类型和内容，header 的 scope 是可选的。

### Type

必须为以下之一：

- feat：一项新功能  
- fix：一个错误修复  
- docs：仅文档更改  
- style：不会影响代码含义的更改（空格，格式，缺少分号等）  
- refactor：既不修正错误也不增加功能的代码更改  
- perf：代码更改可提高性能  
- test：添加缺失的测试或更正现有的测试  
- build：影响构建系统，CI 配置或外部依赖项的更改（比如：gulp，npm）  
- chore：其他不会修改 src 或测试文件的更改（比如文档修改，构建流程）
- release：发布版本提交 

### Scope

可选的，可以是指定提交更改位置的任何内容，例如菜单，侧边栏等。  

### Subject

subject 包含对变更的简洁描述：

- 使用第一人称现在时的命令性语气开头， 如 change，而不是 changed 或 changes
- 第一个字母不要大写
- 末尾不加句号（。）

## Body

可选的。与 subject 一样，使用命令式现在时态， 如 change，而不是 changed 或 changes。

body 应包括改变的动机，并将其与以前的行为进行对比。也就是说，描述为什么修改，做了什么样的修改，以及开发的思路等，是 commit 的详细描述。

## Footer

可选的。页脚应包含有关 Breaking Changes 的所有信息，也是此次 commit 关闭 GitHub issue 的地方。

**Breaking Changes** 应以单词 **BREAKING CHANGE** 开头：用空格或两个换行符。后面是对变动的描述和变动的理由。

- 关闭 issue

```
Closes #233, #344
```

- Breaking Changes
```
BREAKING CHANGE: isolate scope bindings definition has changed.

    To migrate the code follow the example below:

    Before:

    scope: {
      myAttr: 'attribute',
    }

    After:

    scope: {
      myAttr: '@',
    }

    The removed `inject` wasn't generaly useful for directives so there should be no code using it.
```

### Revert

还有一种特殊情况，如果当前 commit 还原了先前的 commit，则应以 revert：开头，后跟还原的 commit 的 header。在 body 中必须写成：`This reverts commit <hash>`。其中 hash 是要还原的 commit 的 SHA 标识。

```
revert: feat(pencil): add 'delete' option

This reverts commit 667ecc1654a317a13331b17617d973392f415f02.
```

## git commit 模板

如果你想尝试一下这样的规范格式，那么可以为 git 设置 commit 模板， 每次 git commit 的时候在 vim 中带出， 时刻提醒自己:

在 ~/.gitconfig 添加:

```
[commit]
template = ~/.gitmessage
```

新建 ~/.gitmessage 内容可以如下:

```
# head: <type>(<scope>): <subject>
# - type: feat, fix, docs, style, refactor, test, chore
# - scope: can be empty (eg. if the change is a global or difficult to assign to a single component)
# - subject: start with verb (such as 'change'), 50-character line
#
# body: 72-character wrapped. This should answer:
# * Why was this change necessary?
# * How does it address the problem?
# * Are there any side effects?
#
# footer: 
# - Include a link to the ticket, if any.
# - BREAKING CHANGE
#
```


## Commitizen

[commitizen/cz-cli](https://github.com/commitizen/cz-cli) 是一个撰写合格 Commit message 的工具。我们需要借助它提供的 git cz 命令替代我们的 git commit 命令，帮助我们生成符合规范的 commit message。

下面，我们就来安装一下吧。

### 安装

```bash
npm install commitizen -g
```
在项目目录里，运行下面的命令，使其支持 Angular 的 Commit message 规范。

```bash
commitizen init cz-conventional-changelog --save --save-exact
```

那么在对应的项目中执行 `git cz` 替代 `git commit`，之后就会出现选项，用来生成符合规范的 commit message。

效果如下：
![add-commit](https://img-blog.csdnimg.cn/20200721175725796.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L29zY2hpbmFfNDE3OTA5MDU=,size_16,color_FFFFFF,t_70)

更多安装方式可以查看：[commitizen/cz-cli](https://github.com/commitizen/cz-cli) 


## 小结

作为程序猿，我个人认为适当遵守一些开发规范，可以帮助我们减少编程中的一些错误，提高开发效率。所以，一定要养成良好的编程习惯。

## 参考

[Angular Git Commit Guidelines](https://github.com/angular/angular.js/blob/master/DEVELOPERS.md#-git-commit-guidelines)

[优雅的提交你的 Git Commit Message](https://juejin.im/post/5afc5242f265da0b7f44bee4#heading-2)

[Commit message 和 Change log 编写指南](http://www.ruanyifeng.com/blog/2016/01/commit_message_change_log.html)



