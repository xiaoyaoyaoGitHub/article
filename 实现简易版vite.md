# 实现简易版`vite`

### 创建项目

> 目录结构如下

```markdown
|-- vite-project
    |-- App.vue
    |-- index.css
    |-- index.html
    |-- main.js
    |-- package-lock.json
    |-- package.json
    |-- server
        |-- index.js
```

### 引入主文件入口

> 我们按照`main.js`为入口，在`index.html`中已`module`的方式引入

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta http-equiv="X-UA-Compatible" content="IE=edge">
    <meta name="viewport" content="width=device-width, minimum-scale=1.0, initial-scale=1.0, maximum-scale=1.0, user-scalable=no">
    <title>Document</title>
</head>
<body>
    <div id="app"></div>
</body>
<script>
    window.process = {env:{NODE_ENV:"production"}}
</script>
<script type="module" src="./main.js"></script>
</html>
```

> 使用`module`的方式引入，浏览器会按照这个地址去触发请求，目前为止，浏览器获取到的`main。js`的内容如下：

```js
import { createApp, h } from "vue";
import App from "./App.vue"
createApp(App).mount('#app')
```

### 创建服务

>  其实浏览器是无法请求到`vue`的资源，所以我们需要修改下资源地址，指向`node_module`文件夹下的资源，我们首先用`koa`创建个服务，在`server/index.js`

```js
const Koa = require('koa')
const path = require('path')
const fs = require('fs')
const app = new Koa()

app.use(async (ctx) => {
    const { url } = ctx.request;
    if (url === '/') {
        ctx.type = 'text/html';
        const p = path.join(__dirname, '..', '/index.html');
        const htmls = fs.readFileSync(p, 'utf-8');
        ctx.body = htmls
    }
})
// 监听端口
app.listen('3001', () => {
    console.log('koa server 3001');
})
```

### 解析`main.js`

> 当用户请求地址`localhost:3001`时，会向浏览器返回`index.html`文件，因为在改文件中我们又请求到`main.js`，所以我们可以在此处拦截并改造其内容：

```js
// server/index.js中添加分支判断
 if (url === '/') {
       ...
 } else if (url.endsWith('.js')) { //判断js文件请求
   const jsPath = path.join(__dirname, '..', url);
   const jsContents = fs.readFileSync(jsPath, 'utf-8');
   ctx.type = 'text/javascript';
   ctx.body = replaceModelPath(jsContents)
 }

 function replaceModelPath(content) {
    return content.replace(/ from ('|")(.*)(\1)/g, function (s0, s1, s2) {
        if (s2.match(/^(\/|..\/|.\/)/)) {
            return s0
        } else {
            return s0.replace(s2, `/@modules/${s2}`)
        }
    })
}

```

### 解析依赖`Vue`

> 函数`replaceModelPath`的作用就是判断请求`js`的源文件中是否存在`from ""`方式引入的依赖，并替换该路径，即由`from "vue"` 替换为`from "/@modules/vue"`，此时浏览器请求到的`main.js`的文件如下：

```js
import { createApp, h } from "/@modules/vue";
import App from "./App.vue"
createApp(App).mount('#app')
```

> 在浏览器解析到`import { createApp, h } from "/@modules/vue"`，会发起一个路径为`http://localhost:3001/@modules/vue`的请求，我们可以在服务端进行拦截，改造`server/index.js`如下

```js
if (url.endsWith('.js')) {
  ...
} else if (url.startsWith('/@modules/')) {
  const moduleName = url.replace('/@modules/', '');
  const modulePath = path.join(__dirname, '../../node_modules', moduleName);
  // 要加载的文件
  const moduleFile = require(modulePath + '/package.json').module;
  ctx.type = "text/javascript";
  ctx.body = replaceModelPath(fs.readFileSync(modulePath + '/' + moduleFile, 'utf-8'))
} 
```

> 在服务端拦截到浏览器请求`vue`文件时，我们需要向`node_modules`文件夹中去寻找资源，找到`node_modules/vue`下的`package.json`文件，其中的`module`对应的路径就是我们要请求的真是的路径，请求返回如下

```js
import { initCustomFormatter, warn } from '/@modules/@vue/runtime-dom';
export * from '/@modules/@vue/runtime-dom';
function initDev() {
    {
        initCustomFormatter();
    }
}
// This entry exports the runtime only, and is built as
if ((process.env.NODE_ENV !== 'production')) {
    initDev();
}
const compile = () => {
    if ((process.env.NODE_ENV !== 'production')) {
        warn(`Runtime compilation is not supported in this build of Vue.` +
            (` Configure your bundler to alias "vue" to "vue/dist/vue.esm-bundler.js".`
                ) /* should not happen */);
    }
};
export { compile };

```

### `SFC`请求

> 后续浏览器执行到语句`import App from "./App.vue"`，改造`server/index.js`

```js
if (url.startsWith('/@modules/')){
  ...
}else if (url.indexOf('vue')){
  const fileName = path.join(__dirname, '..', url.split('?')[0]);
  const vueContent = fs.readFileSync(fileName, 'utf-8');
  const compileContent = compilerSfc.parse(vueContent);
}
```

> 解析`.vue`文件时，我们需要使用`vue`提供给我们的依赖包`@vue/compiler-sfc`，这个依赖可以帮我们解析`AST`语法树，格式如下：

```json
{
  descriptor: {
    filename: 'anonymous.vue',
    source: '<template>\n' +
      '    <div>{{title}}</div>\n' +
      '</template>\n' +
      '\n' +
      '\n' +
      '<script>\n' +
      "import { ref } from 'vue'\n" +
      '\n' +
      'export default {\n' +
      '    setup() {\n' +
      "        const title = ref('hello');\n" +
      '        \n' +
      '        // setInterval(() => {\n' +
      "        //     title.value = title.value.split('').reverse().join('')\n" +
      '        // }, 1000)\n' +
      '        \n' +
      '        return {\n' +
      '            title\n' +
      '        }\n' +
      '    },\n' +
      '}\n' +
      '</script>\n' +
      '\n' +
      '<style scoped>\n' +
      'div {\n' +
      '    color: red;\n' +
      '}\n' +
      '</style>\n',
    template: {
      type: 'template',
      content: '\n    <div>{{title}}</div>\n',
      loc: [Object],
      attrs: {},
      ast: [Object],
      map: [Object]
    },
    script: {
      type: 'script',
      content: '\n' +
        "import { ref } from 'vue'\n" +
        '\n' +
        'export default {\n' +
        '    setup() {\n' +
        "        const title = ref('hello');\n" +
        '        \n' +
        '        // setInterval(() => {\n' +
        "        //     title.value = title.value.split('').reverse().join('')\n" +
        '        // }, 1000)\n' +
        '        \n' +
        '        return {\n' +
        '            title\n' +
        '        }\n' +
        '    },\n' +
        '}\n',
      loc: [Object],
      attrs: {},
      map: [Object]
    },
    scriptSetup: null,
    styles: [ [Object] ],
    customBlocks: [],
    cssVars: [],
    slotted: false
  },
  errors: []
}
```

> 其中`descriptor.template`对应的是`.vue`中的模板部分，`descriptor.script`对应的是逻辑部分，`descriptor.styles`对应的是样式

我们会将`descriptor.script`中的内容作为用户请求结果返回，在返回之前我们需要对其做些改造：

```js
const fileName = path.join(__dirname, '..', url.split('?')[0]);
const vueContent = fs.readFileSync(fileName, 'utf-8');
const compileContent = compilerSfc.parse(vueContent);
let compilerScript = compileContent.descriptor.script.content;
//将导出内容转化成导出一个变量，后续我们可以在变量上挂载render函数
const scriptContent = compilerScript.replace('export default', 'const __script=')
ctx.type = 'text/javascript';
ctx.body = `
  ${replaceModelPath(scriptContent)}
	// 将css转化为另一个请求，增加后续类型style类型
  import '${url}?type=style'
	// 将template转化为另一个请求，增加template类型
  import { render as __render } from '${url}?type=template'
  __script.render = __render
  export default __script
`
```

> 后续需要将`template`编译为`render`函数挂载到当前执行脚本供`createApp`调用，

### 解析`template`

> 浏览器解析完`.vue`文件后，会发起请求`http://localhost:3001/App.vue?type=template`，继续改造`server/index.js`

```js
const fileName = path.join(__dirname, '..', url.split('?')[0]);
const vueContent = fs.readFileSync(fileName, 'utf-8');
const compileContent = compilerSfc.parse(vueContent);
console.log(compileContent);
if (!query.type) { // 
  let compilerScript = compileContent.descriptor.script.content;
  const scriptContent = compilerScript.replace('export default', 'const __script=')
  ctx.type = 'text/javascript';
  ctx.body = `
${replaceModelPath(scriptContent)}
import '${url}?type=style'
import { render as __render } from '${url}?type=template'
__script.render = __render
export default __script
`
} else { //template
	const compilerTemplate = compileContent.descriptor.template.content;
  // 将模板编译为render函数
  const render = compilerDom.compile(compilerTemplate, { mode: 'module' }).code;
  ctx.type = 'text/javascript';
  ctx.body = replaceModelPath(render)
}
```

> 这里用到依赖`@vue/compiler-dom`进行模板编译

### 解析`css`

> 浏览器请求连接`http://localhost:3001/App.vue?type=style`

```js
if (query.type === 'template') {
  const compilerTemplate = compileContent.descriptor.template.content;
  const render = compilerDom.compile(compilerTemplate, { mode: 'module' }).code;
  console.log(render);
  ctx.type = 'text/javascript';
  ctx.body = replaceModelPath(render)
} else { //style
  const currentStyle = compileContent.descriptor.styles[0].content;
  ctx.type = 'text/javascript';
  // 创建style标签插入
  ctx.body = `
    const css = "${currentStyle.replace(/\n/g, '')}";
    let link = document.createElement('style');
    link.setAttribute('type','text/css');
    document.head.appendChild(link);
    link.innerHTML = css;
    `
}
```

到此我们的请求就全部处理完成

### 启动

> 使用`nodemon`启动`server/index.js`启动服务，完成

