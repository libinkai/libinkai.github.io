# 模块化思想

## 作用域

- 全局作用域
- 局部作用域（立即执行函数）
- 块级作用域（ES6）

## 模块化的优点

- 作用域封装
- 复用性
- 解耦合

## 模块化演进

- 命名空间，使用对象封装
- 局部作用域，使用立即执行函数
- AMD异步模块定义，使用define函数定义模块
- COMMONJS，使用exports导出，require引入
- ES6 MODULE，export导出，import引入（原生语法支持）

# webpack的打包机制

1. 从入口文件开始，分析整个应用的依赖树
2. 将每个依赖模块包装起来，放到一个数组中等待调用
3. 实现模块加载的方法，并把它放到模块执行的环境中，确保模块间可以互相调用
4. 把执行入口文件的逻辑放在一个函数表达式中，并立即执行这个函数

# 包管理器

- 初始化：`npm init -y`

## package.json



## npm install

- `npm install -s`，安装并在packjson.js文件`dependencies`中声明依赖
- `npm install --save-dev`，安装为开发环境依赖`devDependencies`
- 重新安装部署时可以指定环境
- 过程
  - 寻找报版本信息文件(package.json)，依照它来进行安装
  - 查找package.json中的依赖，并检查项目中其他的版本信息文件
  - 如果发现了新包，就更新版本信息文件