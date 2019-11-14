---
layout: post
title:  "配置vim来写Haskell"
date:   2019-09-08 12:35:00 +0800
tags: [Vim, Haskell]
---

第一步当初是安装vim，推荐使用比较新的8.1+版本。

```
brew install vim
```

写haskell需要安装相应的插件，以得先搞定这个。

vim8虽然有插件管理，但我没用过，这里使用vim-plug来安装。按照[vim-plug官方](https://github.com/junegunn/vim-plug)给出的命令即可安装成功。

打开~/.vimrc，如果你按照vim-plug官方推荐的步骤安装好vim-plug后，.vimrc应该大约是这个样子。

```vim
call plug#begin('~/.vim/plugged')
call plug#end()
```

添加[stylish-haskell](https://github.com/jaspervdj/stylish-haskell)插件。

```vim
call plug#begin('~/.vim/plugged')
Plug 'jaspervdj/stylish-haskell'
call plug#end()
```

然后在vim中执行`:PlugInstall`进行安装。

当然此时这个插件还不能用，因为它依赖命令行工具，具体内容可以去看官方文档。

好了，到这里你应该掌握安装vim插件的基本方法了。
