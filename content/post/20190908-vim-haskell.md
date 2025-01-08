---
title: "配置vim进行Haskell开发"
date: 2019-09-08T12:35:00+08:00
categories:
- 技术
tags:
- vim
- haskell
---

第一步当初是安装vim，推荐使用比较新的8.1+版本。

写haskell需要安装相应的插件，vim没有原生的插件管理系统，所以得先搞定这个。

这里推荐使用vim-plug来安装。按照官方 https://github.com/junegunn/vim-plug 给出的命令即可安装成功。

打开`~/.vimrc`，如果你按照`vim-plug`官方推荐的步骤安装好后，`.vimrc`应该大约是这个样子。

```
call plug#begin('~/.vim/plugged')
call plug#end()
```

在中间添加stylish-haskell https://github.com/jaspervdj/stylish-haskell 。

```
call plug#begin('~/.vim/plugged')
Plug 'jaspervdj/stylish-haskell'
call plug#end()
```

然后在vim中执行`:PlugInstall`进行安装。

当然此时这个插件还不能用，它还依赖了stylish-haskell命令行工具，推荐使用stack或cabel进行安装：https://github.com/haskell/stylish-haskell#installation 。

好了，到这里你应该掌握安装vim插件的基本方法了。仅安装这一个插件可能对很多人来讲还不够，也可以搭配其他插件使用。
