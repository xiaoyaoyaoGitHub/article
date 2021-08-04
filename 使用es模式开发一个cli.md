# 使用`es`模式开发一个`cli`

###初始化项目

```markdown
$ npm init -y
```

> 因为我们是使用`ES  module`方式，所以需要将`package.json`中的`type`属性值设置为`module`，注意：此处设置完成后，将不能使用`__dirname`获取当前文件路径

我们需要的使用到的__依赖包__

- [`ejs` ](https://ejs.bootcss.com) 模板语言
- [`inquirer`](https://www.npmjs.com/package/inquirer)  命令行交互工具

### 创建`bin`目录

##### 创建模板目录`template`

> 在该目录下创建两个ejs文件`package.ejs`和`koa.ejs`，分别作为`package.json`和`koa.js`的模板文件

`package.ejs`

```ejs
{
    "name": "<%= packageName%>",
    "version": "1.0.0",
    "description": "",
    "main": "index.js",
    "scripts": {
      "test": "echo \"Error: no test specified\" && exit 1"
    },
    "keywords": [],
    "author": "",
    "license": "ISC",
    "dependencies": {
      <% if(middleWear.dom) { %>
        "@vue/compiler-dom": "^3.1.5",
      <% } %>
      <% if(middleWear.sfc){ %>
        "@vue/compiler-sfc": "^3.1.5",
      <% } %>
      "koa": "^2.13.1",
      "vue": "^2.6.14",
      "vue-cli": "^2.9.6"
    }
  }
  
```

> `name`字段取值用户输入的项目名称；
>
> 两个依赖包增加判断，如果`middleware.dom`和`middleware.sfc`为`true`，则展示，否则就不展示

`koa.ejs`

```ejs
const Koa = require('koa')
const path = require('path')
const fs = require('fs')
<% if(middleware.sfc) { %>
    const compilerSfc = require('@vue/compiler-sfc')
<% } %>
<% if(middleware.dom) { %>
    const compilerDom = require('@vue/compiler-dom')
<% } %>
const app = new Koa()

app.use(async (ctx) => {
    const { url, query } = ctx.request;
})

app.listen('<%= port%>', () => {
    console.log('koa server 3001');
})
```

> app监听端口`port`取自用户的输入值
>
> 顶部两个依赖也是根据用户是否选择安装判断是否引入

##### 创建文件读取模板

`createKoa.js`

```js
import fs from "fs";
import { fileURLToPath } from "url";
import ejs from "ejs"
import path from "path"

export default (config) => {
    const __dirname = fileURLToPath(import.meta.url);
  	// 读取模板
    const koaTemp = fs.readFileSync(path.resolve(__dirname, '../template/koa.ejs'), 'utf-8');
  	// 使用ejs.render转化模板，转化模板过程中会读取传入变量并替换
    return ejs.render(koaTemp.toString(), {
        middleware: config.middleware,
        port: config.port
    })
}
```

* 因为我们使用的是`es`方式，所以我们需要通过`import.meta.url`获取到当前文件的完整目录，通过该方法获取到的结果为`file://.../commitCli/bin/createKoa.js`，需要通过`node`内置模块`path`提供的方法`fileURLToPath`方法格式化路径

`createPackage.js` 

```js
import ejs from "ejs"
import fs from "fs"
import { fileURLToPath } from "url"
import path from "path"
export default (config) => {
    const __dirname = fileURLToPath(import.meta.url);
    const packageTemp = fs.readFileSync(path.resolve(__dirname, '../template/package.ejs'), 'utf-8');
    return ejs.render(packageTemp.toString(), {
        middleware: config.middleware,
        packageName: config.packageName
    })
}
```

##### 创建入口文件

`index.js`

```js
import fs from "fs";
import path from "path"
import { fileURLToPath } from "url"
import createPackage from "./createPackage.js";
import createKoa from "./createKoa.js";
const rootPath = path.join('./', 'wangly')
// 模拟用户输入
const config = {
  packageName:'test',
  port:'3000',
  middleware:{
    dom:false,
    sfc:true
  }
}
// 创建文件夹
fs.mkdirSync(rootPath)
// 写入文件
fs.writeFileSync(path.join(rootPath, './packages.json'), createPackage(config))
fs.writeFileSync(path.join(rootPath, './koa.js'), createKoa(config))

```

##### 编写脚本

> 更改根目录下`package.json`

```js
"scripts": {
    "build": "rm -rf ./wangly && node ./bin/index.js"
  }
```

此时，我们还缺少用户的输入，包括项目名称，端口等，此处我们需要通过`inquirer`创建问题

##### 在`bin`目录下创建目录`question`

`index.js`

```js
import inquirer from "inquirer"
import packageName from "./packageName.js"
import middleware from "./middleware.js"
import port from "./port.js"
export default async () => {
    return inquirer.prompt([
        {
            type: 'input',
            message: "请输入项目名称",
            name: 'packageName'
        },
        {
            name: 'port',
            type: 'input',
            default: '3000',
            message: '请输入端口号'
        },
        {
            type: 'checkbox',
            message: '请选择中间件',
            name: "middleware",
            choices: ['@vue/compiler-sfc', '@vue/compiler-dom']
        }
    ])
}
```

> 因为`inquirer.prompt`返回一个`promise`，所以我们可以在入口文件中调用获取，增加代码

```js
+ #!/usr/bin/env node

  import fs from "fs";
  import path from "path"
  import { fileURLToPath } from "url"
  import createPackage from "./createPackage.js";
  import createKoa from "./createKoa.js";
+ import question from "./question/index.js";
+ import handleAnswer from "./config.js"
  const rootPath = path.join('./', 'wangly')

+ const answers = await question()
+ const config = handleAnswer(answers)
  // 创建文件夹
  fs.mkdirSync(rootPath)

  fs.writeFileSync(path.join(rootPath, './packages.json'), createPackage(config))
  fs.writeFileSync(path.join(rootPath, './koa.js'), createKoa(config))
```

其中`./config.js`为我们处理用户输入结果的函数

```js
export default (answer) => {
    return {
        packageName: answer.packageName,
        port:answer.port,
        middleware: {
            sfc: filterMiddleware('@vue/compiler-sfc', answer.middleware),
            dom: filterMiddleware('@vue/compiler-dom', answer.middleware)
        }
    }
}
function filterMiddleware(name, middleware) {
    return middleware.indexOf(name) > -1
}
```

##### 运行脚本

```node
$ npm run build
```

创建完成

##### 关联到本地包管理

```base
$ npm link

/usr/local/bin/commitcli -> /usr/local/lib/node_modules/commitcli/bin/index.js
/usr/local/lib/node_modules/commitcli -> /Users/wangly/Documents/study/CLI/commitCli
```

此处可以看到关联成功

##### 发布

```base
$ npm login  //登录
$ npm publish  // 发布
```

