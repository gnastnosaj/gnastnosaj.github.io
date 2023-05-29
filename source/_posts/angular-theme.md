---
title: Angular主题方案
date: 2019-05-28 09:44:05
tags:
---

# 基于NgClass的主题切换

Angular常见的主题方案是将组件设置NgClass属性并动态绑定主题class，通过编写对应class的样式以及切换class来达到切换主题的目的。为了简化实践，我们可以将此过程封装成一个Directive。

参考代码如下
```
import { BehaviorSubject } from 'rxjs';

//主题Subject
const theme: BehaviorSubject<string> = new BehaviorSubject(null);

//所有主题数组
const themes: Array<string> = [];

//订阅主题Subject，将主题class应用到body
theme.subscribe(c => {
    themes.forEach(t => {
        if (t === c) {
            document.body.classList.add(t);
        } else {
            document.body.classList.remove(t);
        }
    });
});

//注册主题
function registerThemes(...themesToRegister: Array<string>) {
    themes.push(...themesToRegister);
    if (theme.value == null) {
        theme.next(themesToRegister[0]);
    }
}

import { Directive, ElementRef } from '@angular/core';

//主题Directive，将主题class应用到对应元素
@Directive({
    selector: '[ngCariTheme]'
})
export class ThemeDirective {
    constructor(el: ElementRef) {
        theme.subscribe(c => {
            themes.forEach(t => {
                if (t === c) {
                    el.nativeElement.classList.add(t);
                } else {
                    el.nativeElement.classList.remove(t);
                }
            });
        });
    }
}
```
使用方式
1. 应用初始化时调用registerThemes注册所有主题
2. 在对应的组件上添加ngCariTheme属性来绑定主题class
3. 编写对应class样式
4. 最后通过theme.next('theme')切换主题。

# NG-ZORRO主题切换

基于NgClass的主题切换虽然能实现我们想要的效果，但是想要修改整体效果却需要耗费很大的工作量，考虑到目前项目中使用了NG-ZORRO组件与样式库，可以采用先切换NG-ZORRO主题再调整部分主题细节的方案。

实现整体切换NG-ZORRO主题有两种方案，第一种是预先将多种主题less编译成css并以静态资源的方式来进行打包处理，在应用加载时动态加载对应主题的css，第二种方案是将多种主题的less文件以静态资源的方式来进行打包处理，在应用加载时动态编译less并应用到全局样式。考虑到我们目前的项目最终以C的形态呈现，不存在网络延迟导致静态资源加载速度过慢的问题，所以采用第二种方案，以达到简便灵活的目的。

参考代码如下:
```
    //主题节点元素
    style = null;
    //缓存主题css样式
    styles = {};
    ...

    //订阅主题切换
    theme.subscribe(t => {
      if (t != null) {
        //创建主题节点元素
        if (this.style == null) {
          this.style = document.createElement('style');
          this.style.setAttribute('type', 'text/css');
          document.head.appendChild(this.style);
        }
        if (this.styles[t]) {
          //应用缓存主题css样式
          this.style.innerHTML = this.styles[t];
        } else {
          //动态编译主题less并缓存
          less.render(`@import "/theme/${t}.less";`, {
            javascriptEnabled: true
          }).then(output => {
            this.style.innerHTML = output.css;
            this.styles[t] = output.css;
          });
        }
      }
    });

    //注册主题
    registerThemes('light', 'dark');

```