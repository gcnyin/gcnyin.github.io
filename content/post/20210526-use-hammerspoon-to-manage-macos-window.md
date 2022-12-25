---
title: 使用hammerspoon管理macOS窗口
date: 2021-05-26T19:49:27+08:00
tags:
- macos
---

macOS没有内置窗口管理功能，需要安装第三方软件来实现。常用的免费软件有Spectacle和ShiftIt，这已不再维护。今天将macOS升级到11.3后，发现ShiftIt彻底不能使用了。搜寻后在[ShiftIt wiki](https://github.com/fikovnik/ShiftIt/wiki/The-Hammerspoon-Alternative)里找到了替代品[hammerspoon](http://www.hammerspoon.org/)。

# 介绍

> Hammerspoon 是一款macOS平台的免费开源软件，通过桥接操作系统与 Lua 脚本引擎的方式，让我们可以通过编写 Lua 代码来实现操作应用程序、窗口、鼠标、文本、音频设备、电池、屏幕、剪切板、定位、wifi等。基本囊括了系统的各方面。

# 安装

```
brew cask install hammerspoon
```

# 编写脚本

创建`.hammerspoon`目录和`init.lua`文件。

```
mkdir ~/.hammerspoon
cd ~/.hammerspoon
touch init.lua
```

编辑`init.lua`，填写以下内容。

```lua
hs.window.animationDuration = 0
units = {
  right50       = { x = 0.50, y = 0.00, w = 0.50, h = 1.00 },
  left50        = { x = 0.00, y = 0.00, w = 0.50, h = 1.00 },
  top50         = { x = 0.00, y = 0.00, w = 1.00, h = 0.50 },
  bot50         = { x = 0.00, y = 0.50, w = 1.00, h = 0.50 },
  maximum       = { x = 0.00, y = 0.00, w = 1.00, h = 1.00 }
}

hs.hotkey.bind({ 'ctrl', 'alt', 'cmd'}, 'n', function()
  local win = hs.window.focusedWindow()
  -- get the screen where the focused window is displayed, a.k.a. current screen
  local screen = win:screen()
  -- compute the unitRect of the focused window relative to the current screen
  -- and move the window to the next screen setting the same unitRect
  win:move(win:frame():toUnitRect(screen:frame()), screen:next(), true, 0)
end)

mash = { 'ctrl', 'alt', 'cmd' }
hs.hotkey.bind(mash, 'right', function() hs.window.focusedWindow():move(units.right50, nil, true) end)
hs.hotkey.bind(mash, 'left', function() hs.window.focusedWindow():move(units.left50, nil, true) end)
hs.hotkey.bind(mash, 'up', function() hs.window.focusedWindow():move(units.top50, nil, true) end)
hs.hotkey.bind(mash, 'down', function() hs.window.focusedWindow():move(units.bot50, nil, true) end)
hs.hotkey.bind(mash, 'm', function() hs.window.focusedWindow():move(units.maximum, nil, true) end)
```

这个脚本将若干功能绑定到快捷键上，包括

- `ctrl`+`alt`+`cmd`+`n` - 将当前窗口移动到下一个显示器
- `ctrl`+`alt`+`cmd`+`↑` - 将当前窗口移动到屏幕上半边
- `ctrl`+`alt`+`cmd`+`↓` - 将当前窗口移动到屏幕下半边
- `ctrl`+`alt`+`cmd`+`←` - 将当前窗口移动到屏幕右半边
- `ctrl`+`alt`+`cmd`+`→` - 将当前窗口移动到屏幕左半边
- `ctrl`+`alt`+`cmd`+`m` - 将当前窗口最大化

配置好后并不能立即使用，还需要启动Hammerspoon。

# 启动

打开Hammerspoon软件，点击`Reload config`，这样就可以愉快地管理窗口了。

# 结论

至此，我们已经实现了窗口管理的基本功能。当然，Hammerspoon的功能远不止此，感兴趣的读者可以去[Hammerspoon官网](http://www.hammerspoon.org/)了解。
