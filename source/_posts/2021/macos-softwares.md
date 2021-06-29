---
title: 我的macOS软件清单
date: 2021-06-29 20:31:06
tags:
- macos
categories: 
- macos
---

工作后一直用macbook-pro(15款和17款)，已经完全习惯了在macOS下开发。除了公司发的，自己也有一台mbp，所以一些常用的软件两台电脑都要安装一遍。有时忘了装哪个，需要用时就很麻烦。所以将我常用的macOS软件记录在此。

<!-- more -->

# 操作系统

已经升级到了Big Sur，但发现很多软件有兼容性问题，所以还是推荐大家用Catalina。

- Catalina

# 浏览器

虽然我很喜欢Firefox，但这些年的发展实在不怎么亮眼。而且Firefox还搞了中国特供版，账号系统和国际版不能兼容，登录时经常不知道到底使用了哪个版本，实在太窝囊了。还是老老实实用Chrome。

- Chrome

# 编辑器

已经2021年了，编辑器首选vscode。虽然Sublime Text出了3.x，但用着不如vscode顺手。

- Visual studio code

# IDE

不用想，写Java/Scala永远离不开Jetbrains家的Intellij IDEA。

- Intellij IDEA

# 容器

经常需要启动一些后端中间件，Docker必须有。

- docker

# 终端

只有它了。

- iterm2

# 命令行

我这几年积累了很多命令行工具，每一个都无可替代。

- ohmyzsh
- ohmyzsh theme: powerlevel10k
- ohmyzsh plugins: zsh-autosuggestions, zsh-syntax-highlighting, fzf, git-open, docker
- fzf - 快速查找文件、进程
- nvm - 管理nodejs版本
- jenv - 管理jdk版本
- z - 智能目录跳转
- tig - git客户端
- tree - 目录树
- ack - grep替代品
- mosh - mosh客户端
- ncdu - 查看磁盘使用情况
- htop - 查看进程
- diff-so-fancy - 很好的文本比对工具

# 字体

更纱黑体作为等宽字体，不仅适配了多国语言，也同时拥有命令行(Term)和等宽(Mono)这两种不同场景的等宽字体。

- [更纱黑体 Sarasa Gothic](https://github.com/be5invis/Sarasa-Gothic/)

# 窗口管理

macOS的原生窗口管理功能很少，必须用其他软件加强。rectangle是一个，hammerspoon自己写配置也是一个，其他的都要付费，不是很感冒。

- Rectangle

# 输入法

我喜欢五笔，所以并没有太多选择。

- 清歌输入法

# 绘画

我画的是板绘，可惜windows下流行的板绘软件很多并不支持macOS，最终选择了KDE开源的Krita。

- Krita

# 键盘化

键盘的效率远高于鼠标，所以能用键盘完成的事情要尽可能用键盘。

- alfred4
- hammerspoon（可以参考我的上一篇博文）

# 总结

这一套配置更侧重于命令行和键盘操作，需要配置的东西也有一些（比如hammerspoon)，但调教好了之后用起来还是很爽的。可惜macOS下好像没什么工具可以很好的操作鼠标，有用过vimac，但发现会提高输入延迟。
