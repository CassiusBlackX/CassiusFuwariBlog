---
title: autohotkey-configure
published: 2025-08-08
description: '配置autohotkey v2'
image: ''
tags: ["windows", "autohotkey"]
category: 'deployment'
draft: false 
lang: ''
---
# 需求
因为我日常使用的键盘是vgn v98 pro，没有`home`和`end`键，非常不方便，`ins`键又几乎没有什么用，所以想以`ins`键为核心，实现`home`,`end`键的功能

# 脚本
```plaintext
#Requires AutoHotkey v2.0

; === 禁用默认行为 ===
Ins::return                      ; 禁用单独 Ins 键
+Ins::return                     ; 禁用 Shift+Ins（阻止粘贴）

; === 定义组合键规则 ===
#HotIf GetKeyState("Ins", "P") && !GetKeyState("Ctrl", "P")  ; 仅当 Ins 按下且 Ctrl 未按下时触发

PgUp::Home                       ; Ins + PgUp → Home
PgDn::End                        ; Ins + PgDn → End

+PGUP::Send("{Shift Down}{Home}{Shift Up}")  ; Shift+Ins+PgUp → Shift+Home（直接发送完整按键）
+PGDN::Send("{Shift Down}{End}{Shift Up}")   ; Shift+Ins+PgDn → Shift+End

#HotIf                           ; 结束条件块

^Ins::return                     ; Ctrl + Ins → 保持原功能
^Del::+Ins                       ; Ctrl + Del → Shift+Ins（原功能）
```