---
title: GitLab持续集成与SonarQube代码质量管理
date: 2019-05-05 08:40:37
tags:
---

# 关于

GitLab内置了持续集成能力，它可以将保存在共享仓库中的代码进行自动化编译、测试，并配合SonarQube进行自动化代码评审和代码质量管理。本文主要重点讲述GitLab与SonarQube的协作部署，并以Angular项目为例演示持续集成实践。

# 部署

为了简化部署流程，这里以docker环境作为基础。

- GitLab
```
//部署
docker run --detach \
  --hostname gitlab.jasontsang.dev \
  --publish 443:443 --publish 80:80 --publish 22:22 \
  --name gitlab \
  --restart always \
  --volume /srv/gitlab/config:/etc/gitlab \
  --volume /srv/gitlab/logs:/var/log/gitlab \
  --volume /srv/gitlab/data:/var/opt/gitlab \
  gitlab/gitlab-ce:latest

//进入容器
docker exec -it gitlab /bin/bash
//查看容器ip，用于GitLab Runner注册
cat /etc/hosts
```

- GitLab Runner

  1. 部署对应的docker容器
  ```
  //部署
  docker run -d --name gitlab-runner --restart always \
    -v /srv/gitlab-runner/config:/etc/gitlab-runner \
    -v /var/run/docker.sock:/var/run/docker.sock \
    gitlab/gitlab-runner:latest

  //进入容器
  docker exec -it gitlab-runner /bin/bash
  //安装node环境和相关工具
  //angular
  npm install -g @angular/cli
  npm install -g protractor
  //sonar-scanner
  npm install -g sonar-scanner
  ```

  2. 进入http://127.0.0.1/username/ng-cari/settings/ci_cd 查看GitLab Runner的注册令牌Registration Token

  3. 注册GitLab Runner
  ```
  docker run --rm -t -i -v /srv/gitlab-runner/config:/etc/gitlab-runner gitlab/gitlab-runner register -n \
    --url `GitLab对应的url` \
    --registration-token `GitLab对应的GitLab Runner Registration Token` \
    --executor shell \
    --description "Docker Runner" \
    --docker-image "docker:stable" \
    --docker-privileged
  ```

- SonarQube

  本文写作时SonarQube的版本已经发布到7.7，但用于实现与GitLab进行交互的SonarQube插件目前仅支持7.5，所以这里以7.5做为演示。

  1. 部署对应的docker容器
  ```
  docker run --detach \
    --publish 9000:9000 \
    --name sonarqube \
    --restart always \
    sonarqube:7.5-community
  ```

  2. 进入http://127.0.0.1:9000/admin/marketplace 并安装GitLab插件

  3. 进入http://127.0.0.1/profile/personal_access_tokens 生成GitLab的Access Token访问令牌

  4. 进入http://127.0.0.1:9000/admin/settings?category=gitlab 配置GitLab插件信息
  ```
      GitLab url: http://127.0.0.1
      GitLab User Token: `步骤2中生成的GitLab Access Token`
  ```

  5. 进入http://127.0.0.1:9000/admin/users 在Administrator用户下建立SonarQube的Token访问令牌。

# 持续集成

1. 编写GitLab持续集成脚本，并上传项目代码。
```
stages:
  - prepare
  - lint
  - build
  - test
  - analyse
  - quality

#缓存node_modules
cache:
  paths:
    - node_modules/

#安装依赖
prepare:ng:
  stage: prepare
  script:
    - npm install
  artifacts:
    paths:
      - node_modules/

#feature分支代码检查
lint:ng:
  stage: lint
  only:
    - /^feature\/*/
  script:
    #lint代码检查如果失败则直接结束，并拒绝merge request
    - ng lint
    #将feature分支代码merge到master分支，并进行SonarQube代码检查
    - git config user.email "jasontsang.dev@gmail.com"
    - git config user.name "Jason Tsang"
    - git checkout origin/master
    - git merge $CI_COMMIT_SHA --no-commit --no-ff
    - >
      sonar-scanner
      -Dsonar.host.url=http://127.0.0.1:9000
      #SonarQube Token
      -Dsonar.login=a0d764c3a0b5856450c5a40d74d4c98e9fe57cd8
      -Dsonar.analysis.mode=preview
      -Dsonar.sources=src,e2e/src,projects/ng-cari/lottie/src
      -Dsonar.gitlab.commit_sha=$CI_COMMIT_SHA
      -Dsonar.gitlab.ref_name=$CI_COMMIT_REF_NAME
      -Dsonar.gitlab.project_id=$CI_PROJECT_ID
      -Dsonar.gitlab.json_mode=CODECLIMATE
      -Dsonar.gitlab.failure_notification_mode=commit-status
  artifacts:
    expire_in: 1 day
    paths:
      - codeclimate.json

#项目构建
build:ng:
  stage: build
  script:
    - ng build @ng-cari/lottie
    - ng build
  artifacts:
    paths:
      - dist/

#测试
test:ng:
  stage: test
  script:
    - ng test --no-watch --no-progress --browsers=ChromeHeadlessCI --code-coverage
  dependencies:
    - build:ng
  artifacts:
    paths:
      - coverage/

#master分支代码检查需要上传测试覆盖，所以在测试之后执行
analyse:master:
  stage: analyse
  only:
    - master
  script:
    #使用lint生成外部报告并导入SonarQube
    - ng lint ng-cari --format json --force > report.json
    - ng lint ng-cari-e2e --format json --force > e2e/report.json
    - ng lint @ng-cari/lottie --format json --force > projects/ng-cari/lottie/report.json
    - >
      sonar-scanner
      -Dsonar.host.url=http://127.0.0.1:9000
      -Dsonar.login=a0d764c3a0b5856450c5a40d74d4c98e9fe57cd8
      -Dsonar.sources=src,e2e/src,projects/ng-cari/lottie/src
      -Dsonar.typescript.tslint.reportPaths=report.json,e2e/report.json,projects/ng-cari/lottie/report.json
      -Dsonar.typescript.lcov.reportPaths=coverage/ng-cari/lcov.info,coverage/ng-cari/lottie/lcov.info
      -Dsonar.gitlab.commit_sha=$CI_COMMIT_SHA
      -Dsonar.gitlab.ref_name=$CI_COMMIT_REF_NAME
      -Dsonar.gitlab.project_id=$CI_PROJECT_ID
      -Dsonar.gitlab.json_mode=CODECLIMATE
      -Dsonar.gitlab.failure_notification_mode=commit-status
  dependencies:
    - test:ng
  artifacts:
    expire_in: 1 day
    paths:
      - codeclimate.json

#上传codeclimate
codequality:
  stage: quality
  variables:
    GIT_STRATEGY: none
  script:
    - echo ok
  artifacts:
    paths:
      - codeclimate.json
```

2. 查看当前commit的global comment，在最底部会有对应生成的代码质量报告。

# 代码评审和代码质量管理

1. 新建feature分支，并添加code smell（部分糟糕的代码）。

2. 提交代码之后，根据脚本设定在持续集成中会首先进行ng lint，如果此时未通过则pipeline失败，开发者应该检查原因并修复代码问题。如果pipeline成功，则可以进行merge request。

3. 项目维护人员在收到merge request之后，可以查看对应commit的pipeline运行情况以及global comment下的SonarQube代码质量报告，并根据情况选择是否同意merge request。

4. merge request被通过之后，会进行新一轮的持续集成，同样可以在对应commit的global comment下看到本次的SonarQube代码检查报告。

5. 项目维护人员可以在SonarQube控制台下查看项目对应的完整质量报告信息，以及定义对应的Quality Gate，来评估项目是否可以发布。
