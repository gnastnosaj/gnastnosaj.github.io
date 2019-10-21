---
title: RxBus与Angular
date: 2019-10-16 09:44:05
tags:
---

# 引言
笔者在参与Angular项目开发过程中常遇到需要解决组件之间、父子页之间的通信问题，为了降低通信复杂度，以及联想到此前参与过的Android项目开发经验，考虑引进事件总线的解决方案，而由于Angular自带RxJS，故开发一个基于RxJS版本的RxBus成为首选。

# RxBus
RxBus的核心思想是基于Rx实现的事件发布与订阅，网上有许多相关资料，笔者这里就不做详细介绍，下面是笔者根据RxBus核心思想用ts语言编写的RxBus源代码：
```
import { Subject, Observable } from 'rxjs';
import { Injectable } from '@angular/core';

@Injectable({
    providedIn: 'root'
})
export class RxBus {
    bus: Subject<any> = new Subject();
    subjectMapper: {
        [propName: string]: Array<Subject<any>>;
    } = {};

    toObserverable(): Observable<any> {
        return this.bus;
    }

    send(object: any) {
        this.bus.next(object);
    }

    register<T>(tag: any): Observable<T> {
        const subject = new Subject<T>();
        let subjectList = this.subjectMapper[tag];
        if (subjectList == null) {
            subjectList = new Array<Subject<T>>();
            this.subjectMapper[tag] = subjectList;
        }
        subjectList.push(subject);
        return subject;
    }

    unregister(tag: any, observable: Observable<any>) {
        const subjectList = this.subjectMapper[tag];
        if (subjectList != null) {
            this.subjectMapper[tag] = subjectList.filter(subject => subject !== observable);

            if (this.subjectMapper[tag].length === 0) {
                delete this.subjectMapper[tag];
            }
        }
    }

    post<T>(tag: any, object: T) {
        const subjectList = this.subjectMapper[tag];
        if (subjectList != null) {
            subjectList.forEach(subject => subject.next(object));
        }
    }
}
```

# 如何使用
笔者以iframe子页调用父页Angular生命周期中的函数为例加以说明

1. 在Angular应用启动时，将RxBus事件总线加入到window全局变量中，便于iframe子页以top.window.rxbus的形式调用
    ```
    //app.component.ts
    ...
    //通过构造函数注入RxBus事件总线
    constructor(public rxbus: RxBus, private ngZone: NgZone, ...) {
        ...
        (window as any).rxbus = this.rxbus;
        ...
    }
    ...
    ```
2. 在RxBus中注册事件并订阅
    ```
    ...
    this.rxbus.register<any>('graph').subscribe(data => {
        //需要通过NgZone将调用作用域加入到Angular生命周期中
        this.ngZone.run(() => {
            ...
        });
    });
    ...
    ```
3. 在iframe子页中发布事件
    ```
    ...
        top.window.rxbus.post('graph', {
            tag: '',
            payload: {

            }
        });
    ...
    ```
通过事件传递的数据结构开发者可以根据实际情况自行约定
