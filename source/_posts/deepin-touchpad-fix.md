---
title: deepin20使用快捷键启用/禁用触控板
date: 2020-08-20 11:59:47
tags: [工具]
---

## 背景
1. 从windows切换到deepin，并将deepin作为日常主力系统

2. 外出工作，没有外接键盘，需要使用笔记本电脑自带的软键盘

3. deepin20或者说linux的失去焦点策略为单击，比如在你打字输入的时候，只要点击一下输入框外面的部分，光标就会消失，失去焦点；而windows的失焦策略为双击，需要点击两下才会失去焦点。（两者在使用输入法时实测）

4. 由于（3）的存在，只要打字时不小心碰到触摸板一下，就会失去焦点，并且丢失还在输入的内容，非常麻烦

5. deepin v20中只有在设置中关闭触控板的选项，并没有快捷键

## 解决方案

1. 物理解决：随身自带一个外接键盘，这样就不会误触了。

2. 编写shell，映射快捷键

    1. 创建一个内容如下的shell，命名为`touchpad.sh`，内容为
    ```sh
    #!/bin/sh

    enable=$(gsettings get com.deepin.dde.touchpad touchpad-enabled)

    if [ $enable = true ];then 
    gsettings set com.deepin.dde.touchpad touchpad-enabled false
    fi

    if [ $enable = false ];then
    gsettings set com.deepin.dde.touchpad touchpad-enabled true
    ```

    2. 将其移动到 `/usr/bin/` 目录下，运行
    ```
    mv touchpad.sh /user/bin
    ```

    3. 给这个脚本执行权限

    ```
    sudo chmod 744 /usr/bin/touchpad.sh
    ```

    4. 设置 -> 键盘与语言 -> 快捷键 -> 新建快捷键，在命令选项中选择 `/usr/bin/touchpad.sh`，快捷键自己设置。

    5. 这样就可以通过自己设置的快捷键一键切换触控板使用/禁用状态了，当你需要进行文字输入时，按下快捷键，禁用触控板。结束输入的时候，再按一次快捷键，使用触控板。
