# 实现一个简易的monorepo 

#### 介绍

> 是管理项目的一种方式，通常我们是每个模块创建一个`repo`(即项目仓库)。而`monorepo`则是将这些模块放在同一个`repo`里面进行管理「此处我们使用`yarn`进行管理」

#### 创建项目

> 首先我们创建一个项目结构，格式如下

```markdown
|-- monorepo
    |-- README.md
    |-- package.json
    |-- rollup.config.js
    |-- tsconfig.json
    |-- yarn-error.log
    |-- yarn.lock
    |-- packages
    |   |-- readme.md
    |   |-- reactivity
    |   |   |-- package.json
    |   |   |-- src
    |   |       |-- index.ts
    |   |-- share
    |       |-- package.json
    |       |-- src
    |           |-- index.ts
    |-- scripts
        |-- build.js
```

从项目的整体目录可以看到，我们在`packages`的目录下包含了两个模块`reactivity`和`share`，这两个模块各自对应有自己的`package.json`，因为我们后续要根据不同的需求打包输出不同的结果，所以在各自的`package.json`中我们增加自定义的字段`buildOption`标记当前模块需要打包的方式，展示如下：

`reactivity/package.json`

```json
{
  "name": "reactivity",
  "version": "1.0.0",
  "main": "index.js",
  "license": "MIT",
  "buildOption":{
    "name":"VueReactivity",
    "formats":[
      "esm-bundler",
      "cjs",
      "global"
    ]
  }
}
```

`share/package.json`

```json
{
  "name": "share",
  "version": "1.0.0",
  "main": "index.js",
  "license": "MIT",
  "buildOption":{
    "name":"Share",
    "formats":[
      "esm-bundler"
    ]
  }
}
```

其中，`formats`中的值：

* `esm-bundler`表示以`ES Modules`类型打包
* `cjs`表示以`commonJS`类型打包
* `global`表示以`iife`类型打包

#### 编译项目 

> scripts/build.js 的运行流程

- 首先，我们需要分析`packages`目录，查看模块目录

  ```js
  const fs = require('fs');
  const path = require('path');
  // 过滤下非文件夹 文件
  const dirs = fs.readdirSync(path.resolve(__dirname, '../', 'packages')).filter(dir => fs.statSync(`packages/${dir}`).isDirectory())
  ```

* 渠道目录后，我们就循环目录，通过`rollup`打包

  ```js
  const execa = require('execa')
  // 循环打包
  async function runParallel(dirs, iterFn) {
      let result = [];
      for (let dir of dirs) {
          result.push(iterFn(dir))
      }
      return Promise.all(result)
  }
  // 创建线程
  async function build(target) {
    	// sudio：映射到父进程中
      await execa('rollup', ['-c', '--environment', `TARGET:${target}`], { stdio: 'inherit' })
  }
  runParallel(dirs, build).then(() => {
      console.log('success');
  })
  ```

#### 创建`rollup.config.js`

> `rollup`是`ES`模块打包工具，所以本身配置文件也是`ES`模块，使用`export default`导出配置，所以我们导出的文件配置格式如下

```js
[
  {
    input: '',
    output: {
      file: '',
      format: 'es'
    },
    plugins: []
  },
  {
    input: '',
    output: {
      file: '',
      format: 'cjs'
    },
    plugins: []
  },
  {
    input: '',
    output: {
      file: '',
      format: 'iife'
    },
    plugins: []
  }
]
```

这样，我们可以把一个项目打包，同时输出不同文件，当然，输出哪些文件我们还是需要依靠模块里面的配置来决定，所以就需要去读取各个模块下的`package.json`中的`buildOption`配置：

```js
import path from "path";
// 获取package 目录
const packages = path.resolve(__dirname, 'packages');
const packagDir = path.resolve(packages, process.env.TARGET); //获取打包目录
// 获取package.json文件
const packageJson = require(path.resolve(packagDir, 'package.json')) //获取package.json内容
// 获取当前packagejson中的buildOption
const packageOptions = packageJson.buildOption;
console.log(`packageOptions`,packageOptions);
```

输出的结果为:

```js
packageOptions { name: 'Share', formats: [ 'esm-bundler' ] }
packageOptions { name: 'VueReactivity', formats: [ 'esm-bundler', 'cjs', 'global' ] }
```

然后，我们根据我们提前预设好的配置对`packageOptions.formsts`做格式化，输出`rollup`需要的格式：

```js
import ts from "rollup-plugin-typescript2"
import resolvePlugin from "@rollup/plugin-node-resolve"

// 基本配置
const optionsConfig = {
    "esm-bundler": {
        file: path.resolve(packagDir, `dist/${name}.esc-bundler.js`),
        format: "es"
    },
    "cjs": {
        file: path.resolve(packagDir, `dist/${name}.cjs.js`),
        format: 'cjs'   
    },
    'global': {
        file: path.resolve(packagDir, `dist/${name}.global.js`),
        format: 'iife'
    }
}

function createConfig( output) {
    output.name = packageOptions.name; // iife需要名称
    output.sourcemap = true; // 开启sourcemap
    output.exports = 'default' // commonJS需要
    return {
        input: path.resolve(packagDir, 'src/index.ts'),
        output,
        plugins:[
            ts({
                tsconfig:path.resolve(__dirname, 'tsconfig.json')
            }),
            resolvePlugin()
        ]
    }
}

export default packageOptions.formats.map(format => createConfig(optionsConfig[format]))

```

> 转换`commonJS`获取文件方式，需要依赖`@rollup/plugin-node-resolve`

> 因为我们使用的是ts，所以需要使用依赖`rollup-plugin-typescript2`，并初始化`tsconfig.json`文件

```js
$ tsc --init // 创建tsconfig.json

//修改文件配置, 默认为commonjs
"module": "ESNext", /* Specify module code generation: 'none', 'commonjs', 'amd', 'system', 'umd', 'es2015', 'es2020', or 'ESNext'. */
```

到此，我们的配置就完成了，可以根目录配置`package.json`

```json
  "scripts": {
    "build": "node ./scripts/build.js",
    "dev": "node ./scripts/dev.js"
  },
```

#### 运行命令

```bash
npm run build
```

在各个模块下面会生成dist目录

```markdown
  |-- packages
    |   |-- readme.md
    |   |-- reactivity
    |   |   |-- package.json
    |   |   |-- dist
    |   |   |   |-- reactivity.cjs.js
    |   |   |   |-- reactivity.cjs.js.map
    |   |   |   |-- reactivity.esc-bundler.js
    |   |   |   |-- reactivity.esc-bundler.js.map
    |   |   |   |-- reactivity.global.js
    |   |   |   |-- reactivity.global.js.map
    |   |   |-- src
    |   |       |-- index.ts
    |   |-- share
    |       |-- package.json
    |       |-- dist
    |       |   |-- share.esc-bundler.js
    |       |   |-- share.esc-bundler.js.map
    |       |-- src
    |           |-- index.ts
    |-- scripts
        |-- build.js
        |-- dev.js
```

完成
