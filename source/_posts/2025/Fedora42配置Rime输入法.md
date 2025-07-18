---
title: Fedora42配置Rime输入法
date: 2025-7-18
tags:
  - Linux
  - 2025年
  - Fedora
  - 配置
  - 教程
  - 输入法
---

## 前言

我用Linux系统已有五年之久。从进入Linux世界以来，用的输入法一直是系统自带的`ibus-libpinyin`。这个输入法对于刚踏上Linux学习之旅的新手来说相当的方便：无须配置、系统自带、小白友好。可是随着使用Linux桌面发行版的频率增加，甚至直接取代Windows的地位，成为我日用的发行版后，使用自带输入法就显得蹩脚：无法跟上打字和思考速度，常常因拼写成语和短语时牛头不对马嘴的候选词而花大量时间在挑选一个个正确的字上。

如今我已经受够了系统自带输入法愚蠢的候选词，下定决心今天必须把这个输入法换掉，不再为Linux上敲打长文而痛苦。

我第一个想到的就是 [Rime输入法](https://rime.im/)。曾经我有过换输入法的打算，考虑的也是这个输入法。可是当看到 [官方用户指南](https://github.com/rime/home/wiki/UserGuide)这繁杂又不够入门的文档时，直接把我给劝退了：繁体字内容、章节排不合理、无新手引导、术语遍地。今天又硬着头皮翻了一遍，再次体会到官方繁体文档是多么杂乱。最后靠着其他网友博客的配置经验，尝试自己配置。现在算是配置完用得比较顺手。虽然输入法的设置方式对新手不友好，按照其他用户的经验文章和官方文档，就不需要太多头脑负担了。

Rime是一个输入法引擎。在 Windows 平台上叫**小狼毫 Weasel**，在 Mac 平台上叫**鼠须管 Squirrel**，在 Linux 平台叫**中州韵 Rime**。通常都指同一个 **Rime**。

除了安装 Rime，还需安装【方案】才能打字。Rime 默认带了几种简单的方案可以使用，但是默认方案不够强大，大多用户使用第三方方案。这里我就以 [雾凇拼音](https://github.com/iDvel/rime-ice) 来说明，它支持全拼和双拼。配置也简单，有简体中文说明。

## 安装

1. Fedora GNOME桌面自带`ibus`框架，安装以下两个包：

```bash
sudo dnf install ibus-rime librime-lua

# 如果是 Ostree 系统
# sudo rpm-ostree install -A ibus-rime librime-lua

```

2. 打开 Gnome Settings -> Keyboard -> Add Input Source -> Chinese(China) 选择 【Chinese (Rime)】添加

3. 点击桌面右上角输入法图标，切换到 Chinese(Rime)，也可以按组合键【Win+Space】切换。

![Fedora42配置Rime输入法.webp](images/2025/Fedora42配置Rime输入法.webp)

图上显示【自然码双拼】是因为我已经配置好了才写博客，刚安装启用显示的是繁体【维护】。

4. 点击【部署】，桌面会弹窗提示【Rime is under maintenance...】。此时输入法会生成`~/.config/ibus/rime`并构建默认词库。等待一两分钟，弹窗提示【Rime is ready】。按【Shift】键可以切换繁体中文与英文。

## 配置【方案】

Rime没有GUI配置界面，只能通过组合键启动候选词形式的菜单或者修改`~/.config/ibus/rime`中的配置文件。在输入状态下按【Ctrl+｀】或者【F4】可以启动Rime候选词样式的输入法菜单：点击候选词【朙月拼音】，箭头左边是当前状态，箭头右边是点击候选词之后的状态，简繁转换点一下即可；如果打算换自带的其他方案，点击方案对应的候选词即可。

忘记说了，我是**自然码双拼**用户，所以下面的配置过程以第三方方案【雾凇拼音】的双拼来说明，全拼也差不多。

1. 执行以下命令，安装方案【雾凇拼音】

```bash
git clone git@github.com:iDvel/rime-ice.git ~/.config/ibus/rime --depth 1

```

2. 点击桌面右上角输入法图标，点击【部署】，桌面会弹窗提示【Rime is under maintenance...】。此时输入法会构建默认词库。等待一两分钟，弹窗提示【Rime is ready】，此时点击桌面右上角输入法图标可以看到当前是【雾凇拼音】模式。

3. 编辑文件`~/.config/ibus/rime/default.yaml`，配置其他内容。文件默认自带中文说明，下面的内容是我自己的，主要是按键和候选词方面的配置：

```txt
# Rime default settings
# encoding: utf-8


# 要比共享目录的同名文件的 config_version 大才可以生效
config_version: '2025-07-17'


# 方案列表
schema_list:
  # 可以直接删除或注释不需要的方案，对应的 *.schema.yaml 方案文件也可以直接删除
  # 除了 t9，它依赖于 rime_ice，用九宫格别删 rime_ice.schema.yaml
  # - schema: rime_ice               # 雾凇拼音（全拼）
  # - schema: t9                     # 九宫格（仓输入法）
  - schema: double_pinyin          # 自然码双拼
  # - schema: double_pinyin_abc      # 智能 ABC 双拼
  # - schema: double_pinyin_mspy     # 微软双拼
  # - schema: double_pinyin_sogou    # 搜狗双拼
  # - schema: double_pinyin_flypy    # 小鹤双拼
  # - schema: double_pinyin_ziguang  # 紫光双拼
  # - schema: double_pinyin_jiajia   # 拼音加加双拼


# 菜单
menu:
  page_size: 5  # 候选词个数
  # alternative_select_labels: [ ①, ②, ③, ④, ⑤, ⑥, ⑦, ⑧, ⑨, ⑩ ]  # 修改候选项标签
  # alternative_select_keys: ASDFGHJKL  # 如编码字符占用数字键，则需另设选字键（注意在雾凇中，大写字母也作为编码了）


# 方案选单相关
switcher:
  caption: 「方案选单」
  hotkeys:
    - F4
    # - Control+grave
    # - Control+Shift+grave
    # - Alt+grave
  save_options:  # 开关记忆（方案中的 switches），从方案选单（而非快捷键）切换时会记住的选项，需要记忆的开关不能设定 reset
    - ascii_punct
    - traditionalization
    - emoji
    - full_shape
    - search_single_char
  fold_options: false            # 呼出时是否折叠，多方案时建议折叠 true ，一个方案建议展开 false
  abbreviate_options: true      # 折叠时是否缩写选项
  option_list_separator: ' / '  # 折叠时的选项分隔符


# 中西文切换
#
# good_old_caps_lock:
# true   切换大写
# false  切换中英
# macOS 偏好设置的优先级更高，如果勾选【使用大写锁定键切换“ABC”输入法】则始终会切换输入法。
#
# 切换中英：
# 不同的选项表示：打字打到一半时按下了 CapsLock、Shift、Control 后：
# commit_code  上屏原始的编码，然后切换到英文
# commit_text  上屏拼出的词句，然后切换到英文
# clear        清除未上屏内容，然后切换到英文
# inline_ascii 切换到临时英文模式，按回车上屏后回到中文状态
# noop         屏蔽快捷键，不切换中英，但不要屏蔽 CapsLock
ascii_composer:
  good_old_caps_lock: true  # true | false
  switch_key:
    Caps_Lock: clear      # commit_code | commit_text | clear
    Shift_L: commit_code  # commit_code | commit_text | inline_ascii | clear | noop
    Shift_R: noop         # commit_code | commit_text | inline_ascii | clear | noop
    Control_L: noop       # commit_code | commit_text | inline_ascii | clear | noop
    Control_R: noop       # commit_code | commit_text | inline_ascii | clear | noop


###################################################################################


# 下面的 punctuator recognizer key_binder 写了一些所有方案通用的配置项。
# 写在 default.yaml 里，方便多个方案引用，就是不用每个方案都写一遍了。


# 标点符号
# 设置为一个映射，就自动上屏；设置为多个映射，如 '/' : [ '/', ÷ ] 则进行复选。
#   full_shape: 全角没改，使用预设值
#   half_shape: 标点符号全部直接上屏，和 macOS 自带输入法的区别是
#              '|' 是半角的，
#              '~' 是半角的，
#              '`'（反引号）没有改成 '·'（间隔号）。
punctuator:
  digit_separators: ",.:"  # 在此处指定的字符，在数字后被输入，若再次输入数字，则连同数字直接上屏；若双击，则恢复映射。 # librime >= 28a234f
  # digit_separator_action: commit  # 关闭上述行为 librime >= 1.13.1
  full_shape:
    ' ' : { commit: '　' }
    ',' : { commit: ， }
    '.' : { commit: 。 }
    '<' : [ 《, 〈, «, ‹ ]
    '>' : [ 》, 〉, », › ]
    '/' : [ ／, ÷ ]
    '?' : { commit: ？ }
    ';' : { commit: ； }
    ':' : { commit: ： }
    '''' : { pair: [ '‘', '’' ] }
    '"' : { pair: [ '“', '”' ] }
    '\' : [ 、, ＼ ]
    '|' : [ ·, ｜, '§', '¦' ]
    '`' : ｀
    '~' : ～
    '!' : { commit: ！ }
    '@' : [ ＠, ☯ ]
    '#' : [ ＃, ⌘ ]
    '%' : [ ％, '°', '℃' ]
    '$' : [ ￥, '$', '€', '£', '¥', '¢', '¤' ]
    '^' : { commit: …… }
    '&' : ＆
    '*' : [ ＊, ·, ・, ×, ※, ❂ ]
    '(' : （
    ')' : ）
    '-' : －
    '_' : ——
    '+' : ＋
    '=' : ＝
    '[' : [ 「, 【, 〔, ［ ]
    ']' : [ 」, 】, 〕, ］ ]
    '{' : [ 『, 〖, ｛ ]
    '}' : [ 』, 〗, ｝ ]
  half_shape:
    ',' : '，'
    '.' : '。'
    '<' : '《'
    '>' : '》'
    '/' : '/'
    '?' : '？'
    ';' : '；'
    ':' : '：'
    '''' : { pair: [ '‘', '’' ] }
    '"' : { pair: [ '“', '”' ] }
    '\' : '、'
    '|' : '|'
    '`' : '`'
    '~' : '~'
    '!' : '！'
    '@' : '@'
    '#' : '#'
    '%' : '%'
    '$' : '¥'
    '^' : '……'
    '&' : '&'
    '*' : '*'
    '(' : '（'
    ')' : '）'
    '-' : '-'
    '_' : ——
    '+' : '+'
    '=' : '='
    '[' : '【'
    ']' : '】'
    '{' : '「'
    '}' : '」'


# 处理符合特定规则的输入码，如网址、反查
# 此处配置较为通用的选项，各方案中另增加了和方案功能绑定的 patterns。
recognizer:
  patterns:
    email: "^[A-Za-z][-_.0-9A-Za-z]*@.*$"                            # email @ 之后不上屏
    url: "^(www[.]|https?:|ftp[.:]|mailto:|file:).*$|^[a-z]+[.].+$"  # URL
    underscore: "^[A-Za-z]+_.*"  # 下划线不上屏
    # url_2: "^[A-Za-z]+[.].*"   # 句号不上屏，支持 google.com abc.txt 等网址或文件名，使用句号翻页时需要注释掉
    # colon: "^[A-Za-z]+:.*"     # 冒号不上屏


# 快捷键
key_binder:
  # Lua 配置: 以词定字（上屏当前词句的第一个或最后一个字），和中括号翻页有冲突
  # select_first_character: "bracketleft"  # 左中括号 [
  # select_last_character: "bracketright"  # 右中括号 ]

  bindings:
    # Tab / Shift+Tab 切换光标至下/上一个拼音
    - { when: composing, accept: Shift+Tab, send: Shift+Left }
    - { when: composing, accept: Tab, send: Shift+Right }
    # Tab / Shift+Tab 翻页
    # - { when: has_menu, accept: Shift+Tab, send: Page_Up }
    # - { when: has_menu, accept: Tab, send: Page_Down }

    # Option/Alt + ←/→ 切换光标至下/上一个拼音
    - { when: composing, accept: Alt+Left, send: Shift+Left }
    - { when: composing, accept: Alt+Right, send: Shift+Right }

    # 翻页 - =
    # - { when: has_menu, accept: minus, send: Page_Up }  # 上一页设置为 paging 时会导致直接上屏并输出减号
    # - { when: has_menu, accept: equal, send: Page_Down }

    # 翻页 , .
    - { when: paging, accept: comma, send: Page_Up }
    - { when: has_menu, accept: period, send: Page_Down }

    # 翻页 [ ]  ⚠️ 开启时请修改上面以词定字的快捷键
    # - { when: paging, accept: bracketleft, send: Page_Up }
    # - { when: has_menu, accept: bracketright, send: Page_Down }

    # 两种按键配置，鼠须管 Control+Shift+4 生效，小狼毫 Control+Shift+dollar 生效，都写上了。
    # numbered_mode_switch:
    # - { when: always, select: .next, accept: Control+Shift+1 }                  # 在最近的两个方案之间切换
    # - { when: always, select: .next, accept: Control+Shift+exclam }             # 在最近的两个方案之间切换
    # - { when: always, toggle: ascii_mode, accept: Control+Shift+2 }             # 切换中英
    # - { when: always, toggle: ascii_mode, accept: Control+Shift+at }            # 切换中英
    # - { when: always, toggle: ascii_punct, accept: Control+Shift+3 }              # 切换中英标点
    # - { when: always, toggle: ascii_punct, accept: Control+Shift+numbersign }     # 切换中英标点
    # - { when: always, toggle: traditionalization, accept: Control+Shift+4 }       # 切换简繁
    # - { when: always, toggle: traditionalization, accept: Control+Shift+dollar }  # 切换简繁
    # - { when: always, toggle: full_shape, accept: Control+Shift+5 }             # 切换全半角
    # - { when: always, toggle: full_shape, accept: Control+Shift+percent }       # 切换全半角

    # emacs_editing:
    # - { when: composing, accept: Control+p, send: Up }
    # - { when: composing, accept: Control+n, send: Down }
    # - { when: composing, accept: Control+b, send: Left }
    # - { when: composing, accept: Control+f, send: Right }
    # - { when: composing, accept: Control+a, send: Home }
    # - { when: composing, accept: Control+e, send: End }
    # - { when: composing, accept: Control+d, send: Delete }
    # - { when: composing, accept: Control+k, send: Shift+Delete }
    # - { when: composing, accept: Control+h, send: BackSpace }
    # - { when: composing, accept: Control+g, send: Escape }
    # - { when: composing, accept: Control+bracketleft, send: Escape }
    # - { when: composing, accept: Control+y, send: Page_Up }
    # - { when: composing, accept: Alt+v, send: Page_Up }
    # - { when: composing, accept: Control+v, send: Page_Down }

    # optimized_mode_switch:
    # - { when: always, accept: Control+Shift+space, select: .next }
    # - { when: always, accept: Shift+space, toggle: ascii_mode }
    # - { when: always, accept: Control+comma, toggle: full_shape }
    # - { when: always, accept: Control+period, toggle: ascii_punct }
    # - { when: always, accept: Control+slash, toggle: traditionalization }

    # 将小键盘 0~9 . 映射到主键盘，数字金额大写的 Lua 如 R1234.5678 可使用小键盘输入
    - {accept: KP_0, send: 0, when: composing}
    - {accept: KP_1, send: 1, when: composing}
    - {accept: KP_2, send: 2, when: composing}
    - {accept: KP_3, send: 3, when: composing}
    - {accept: KP_4, send: 4, when: composing}
    - {accept: KP_5, send: 5, when: composing}
    - {accept: KP_6, send: 6, when: composing}
    - {accept: KP_7, send: 7, when: composing}
    - {accept: KP_8, send: 8, when: composing}
    - {accept: KP_9, send: 9, when: composing}
    - {accept: KP_Decimal, send: period, when: composing}
    # 将小键盘 + - * / 映射到主键盘，使计算器 如 1+2-3*4 可使用小键盘输入
    - {accept: KP_Multiply, send: asterisk, when: composing}
    - {accept: KP_Add,      send: plus,     when: composing}
    - {accept: KP_Subtract, send: minus,    when: composing}
    - {accept: KP_Divide,   send: slash,    when: composing}

```

4. 打开 `~/.config/ibus/rime/double_pinyin.schema.yaml`，以下是编辑部分的配置：

```txt
# 开关
# reset: 默认状态。注释掉后，切换窗口时不会重置到默认状态。
# states: 方案选单显示的名称。可以注释掉，仍可以通过快捷键切换。
# abbrev: 默认的缩写取 states 的第一个字符，abbrev 可自定义一个字符
switches:
  - name: ascii_mode
    states: [ 中, Ａ ]
  - name: ascii_punct  # 中英标点
    states: [ ¥, $ ]
  - name: traditionalization
    states: [ 简, 繁 ]
  - name: emoji
    states: [ 💀, 😄 ]
    reset: 0
  - name: full_shape
    states: [ 半角, 全角 ]
  - name: search_single_char  # search.lua 的功能开关，辅码查词时是否单字优先
    states: [正常, 单字]
    abbrev: [词, 单]


# 拼写设定
speller:
  # 如果不想让什么标点直接上屏，可以加在 alphabet，或者编辑标点符号为两个及以上的映射
  alphabet: zyxwvutsrqponmlkjihgfedcbaZYXWVUTSRQPONMLKJIHGFEDCBA`
  # initials 定义仅作为始码的按键，排除 ` 让单个的 ` 可以直接上屏
  initials: zyxwvutsrqponmlkjihgfedcbaZYXWVUTSRQPONMLKJIHGFEDCBA
  delimiter: " '"  # 第一位<空格>是拼音之间的分隔符；第二位<'>表示可以手动输入单引号来分割拼音。
  algebra:
    - erase/^xx$/
    - derive/^([jqxy])u$/$1v/
    - derive/^([aoe])([ioun])$/$1$1$2/
    - derive/([ei])n$/$1ng/            # en => eng, in => ing
    - derive/([ei])ng$/$1n/            # eng => en, ing => in
    - xform/^([aoe])(ng)?$/$1$1$2/

```

## 相关参考链接

- [Linux 下 rime 输入法小鹤双拼配置](https://blog.moe233.net/posts/3c46778c/#%E9%85%8D%E7%BD%AE%E6%96%87%E4%BB%B6%E5%90%8C%E6%AD%A5)
- [说明书](https://rimeinn.github.io/rime/)
- [Rime 配置：雾凇拼音 ](https://dvel.me/posts/rime-ice/)
