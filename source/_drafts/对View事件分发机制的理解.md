---
title: 对View事件分发机制的理解
tags:
  - Android
  - 经验
---
View事件分发机制要经历三个主要的函数：

1. `public boolean dispatchTouchEvent(MotionEvent event)`——事件的分发
2. `public boolean onInterceptTouchEvent(MotionEvent ev)`——事件的拦截
3. `public boolean onTouchEvent(MotionEvent event)`——事件的消耗

> 刘老板掌管一家公司，下面有很多员工，有一天他把他的直接下属老李叫到办公室，“老李啊，你们这段时间工作辛苦了，我一直想犒劳一下你们，但是我的钱只够准备这一份礼物，你拿去看着办吧。”说完就让老李出去了。老李是个好人，不会私自收下这份礼物，他想还是问问自己的手下吧。于是他找来他的手下老黄和老何，老黄看了一眼礼物说，“多谢了，我觉得我不需要”

