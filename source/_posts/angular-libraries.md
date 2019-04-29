---
title: Angular Libraries构建与npm packages发布
date: 2019-04-28 13:25:43
tags:
---

# 关于

按照官方的说法，Angular Library是一个不同于app的Angular工程，它需要在一个app中被引入使用。开发者可以以Angular Libraries的形式来为特定的场景提供通用的解决方案，从而可以在不同的app中得到重用。

# Angular Libraries构建

1. Angular Libraries不能单独存在，首先需要创建一个app工程
```
    ng new ng-cari -p ng-cari
```
这里的"-p"是指组件的前缀

2. 进入新建的app工程根目录，并创建一个Angular Library工程
```
    ng g library @ng-cari/lottie -p ng-cari
```
这里的"@ng-cari/"是指npm包作用域，Angular CLI会自动修改angular.json，新增名为"@ng-cari/lottie"的工程，并生成对应的Angular Library工程目录，工程目录位于projects/ng-cari/lottie

3. 修改Angular Library代码，并添加依赖项以及其他对应信息
```
    {
        "name": "@ng-cari/lottie",
        "version": "1.0.0",
        "author": "Jason Tsang",
        "peerDependencies": {
            "@angular/common": "^7.2.0",
            "@angular/core": "^7.2.0"
        },
        "dependencies": {
            "lottie-web": "^5.5.2"
        }
    }
```

4. 在app工程根目录下添加Angular Library所需的依赖项
```
    npm install -S lottie-web
```

5. 在app工程根目录下构建Angular Library工程
```
    ng build @ng-cari/lottie
```

6. 修改app工程代码，引入Angular Library工程，并使用对应的组件
```
    import { LottieModule } from '@ng-cari/lottie';

    <ng-cari-lottie [ngStyle]="{'width.px': '256', 'height.px': '256', 'background': 'black'}"
    path="assets/lottie/progress.json" progress="20"></ng-cari-lottie>
```

7. 运行app工程查看实际效果
```
    ng serve
```

# npm packages发布
 
```
    //进入构建后的Angular Library工程目录，示例位于dist/ng-cari/lottie
    cd dist/ng-cari/lottie

    //如果使用其他源推荐使用nrm源管理工具
    npm install -g nrm

    //添加nexus源
    //组
    nrm add <group> http://127.0.0.1:8081/repository/npm-group/
    //私有
    nrm add <hosted> http://127.0.0.1:8081/repository/npm-hosted/
    //查看源
    nrm ls
    //使用私有
    nrm use <hosted>

    //登录npm账号
    npm login

    //发布，带包作用域的需要有创建私有包的权限或添加公开权限参数
    npm publish
    //由于示例带@ng-cari包作用域，所以这里使用公开权限参数
    npm publish --access public

    //使用组
    nrm use <group>

    //在其他工程中使用发布后的包
    npm install -S @ng-cari/lottie
```