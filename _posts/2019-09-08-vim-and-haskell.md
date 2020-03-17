---
layout: post
title:  "配置vim来写Haskell"
date:   2019-09-08 12:35:00 +0800
tags: [vim, haskell, vim-plug, vim-plugin]
category: vim
---

第一步当然是安装vim，用所在系统上的包管理器安装即可。

第二步是安装vim haskell插件，但在安装前需要先安装插件管理器。

这里使用[vim-plug](https://github.com/junegunn/vim-plug)来安装，按照主页给出的命令安装即可。

安装成功后，`.vimrc`应该大约是这个样子。

```vim
call plug#begin('~/.vim/plugged')
call plug#end()
```

此时就可以添加[stylish-haskell](https://github.com/jaspervdj/stylish-haskell)插件了。

```vim
call plug#begin('~/.vim/plugged')
Plug 'jaspervdj/stylish-haskell'
call plug#end()
```

然后在vim中执行`:PlugInstall`进行安装。

当然此时这个插件还不能用，因为它依赖命令行工具`stylish-haskell`。

使用[stack](https://docs.haskellstack.org/en/stable/README/)安装。

```
stack install stylish-haskell
```

好了，到这里你应该可以愉快地作用vim写Haskell了。
