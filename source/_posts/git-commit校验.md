---
title: git提交的格式校验
tag: git
date: 2021-12-30
---

# git commit 代码提交规范
```
type(scope) : subject
```
* type（必须） : commit 的类别，只允许使用下面几个标识：
* feat : 新功能
* fix : 修复bug
* docs : 文档改变
* style : 代码格式改变
* refactor : 某个已有功能重构
* perf : 性能优化
* test : 增加测试
* build : 改变了build工具 如 grunt换成了 npm
* revert : 撤销上一次的 commit
* chore : 构建过程或辅助工具的变动
* scope（可选） : 用于说明 commit 影响的范围，比如数据层、控制层、视图层等等，视项目不同而不同。
* subject（必须） : commit 的简短描述，不超过50个字符。

# 使用commitizen 规范提交

1. npm install -g commitizen
2. commitizen init cz-conventional-changelog --save --save-exact
3. 用 git cz 命令取代 git commit

# 使用git commit 提交

1. npm i @commitlint/config-conventional @commitlint/cli -D 
2. 
  ```json
  // 配置 package.json
   "commitlint": { 
    "extends": [ 
      "@commitlint/config-conventional" 
    ] 
  }
  // 或者在项目根目录下创建配置文件： .commitlintrc.js
  module.exports = { 
    extends: ['@commitlint/config-conventional'] 
  } 
  ```
3. 安装Husky  npm i husky -D 
4. 配置 package.json
   ```json
  "husky": { 
    "hooks": { 
      "commit-msg": "commitlint -e $HUSKY_GIT_PARAMS" 
    } 
  } 
  // 或者在项目根目录下创建配置文件：.huskyrc
  { 
    "hooks": { 
      "commit-msg": "commitlint -e $HUSKY_GIT_PARAMS" 
    } 
  } 
   ```
6. npx husky install 
7. npx husky add .husky/commit-msg 'npx --no-install commitlint --edit "$1"'

# 提交的时候校验eslint代码规范
