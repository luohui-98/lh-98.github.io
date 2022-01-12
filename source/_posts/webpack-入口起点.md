---
title: webpack入口起点
tag: webpack
date: 2021-9-16
---

# 单个入口（简写）语法
用法：entry: string | [string]
webpack.config.js
```js
module.exports = {
  entry: './path/to/my/entry/file.js',
};

module.exports = {
  entry: {
    main: './path/to/my/entry/file.js',
  },
};
```

我们也可以将一个文件路径数组传递给 entry 属性，这将创建一个所谓的 "multi-main entry"。在你想要一次注入多个依赖文件，并且将它们的依赖关系绘制在一个 "chunk" 中时，这种方式就很有用。
```js
module.exports = {
  entry: ['./src/file_1.js', './src/file_2.js'],
  output: {
    filename: 'bundle.js',
  },
};
```

# 对象语法
用法：entry: { <entryChunkName> string | [string] } | {}
```js
module.exports = {
  entry: {
    app: './src/app.js',
    adminApp: './src/adminApp.js',
  },
}
```
对象语法会比较繁琐。然而，这是应用程序中定义入口的最可扩展的方式。
