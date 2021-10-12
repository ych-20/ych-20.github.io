# 前端Cli

--- 

### 涉及的库
- commander ：参数解析 --help其实就借助了他~ 解析用户输入的命令
- inquirer ：交互式命令行工具，有他就可以实现命令行的选择功能
- download-git-repo ：拉取GitHub上的文件
- chalk ：帮我们在控制台中画出各种各样的颜色
- ora：小图标 （loading、succeed、warn等）
- metalsmith ：读取所有文件，实现模板渲染
- consolidate ：统一模板引擎

### 创建项目

- **package.json** 中的 bin属性  key为命令名称 value 为命令执行的文件路径,全局安装包后会自动注册到环境变量
- 本地调试 需要 npm link 才能 只能命令
```
    npm init -y  // 初始化项目
    //项目目录
    ├── bin
    │   └── test-cli  // 全局命令执行的根文件
    ├── src
    │   ├── main.js // 入口文件
    │   └── utils   // 存放工具方法
    |       |___constants.js  //  存放用户所需要的常量
    |       |___common.js
    │   ├── create.js // create 命令所有逻辑
    │   ├── config.js // config 命令所有逻辑
    │── .huskyrc    // git hook
    │── .eslintrc.json // 代码规范校
    ├── package.json
    |__ README.md
```
### 入口文件 和  命令行相关参数
```js
// ./bin/test-cli.js
#!/usr/bin/env node
//表示 我要用系统中的这个目录/user/bin/env的node环境来执行此文件，且需要注意必须放在文件开头。
require('../src/main')
```
```js
// ./src/main.js
const {version, name} = require('../src/util/constants')
const path = require('path')
const program = require('commander');

const {mapActions} = require('./util/common')
// 配置command
Reflect.ownKeys(mapActions).forEach((action) =>{
    program.command(action)
        .alias(mapActions[action].alias)
        .description(mapActions[action].description + 'yyyyyyy') // 命令对应的描述
        .action(()=>{
            if(action === '*'){  //访问不到对应的命令 就打印找不到命令
                // console.log(mapActions[action].description);
            }else{
                // 分解命令 到文件里 有多少文件 就有多少配置 create config
                require(path.join(__dirname,action))(...process.argv.slice(3));
            }
        })
})

program.on('--help', () => {
    console.log('\nExamples:');
    Reflect.ownKeys(mapActions).forEach((action) => {
        mapActions[action].examples.forEach((example) => {
            console.log(` ${example}`);
        })
    })})

program.version(version)
    .parse(process.argv); // process.argv就是用户在命令行中传入的参数
```
https://juejin.cn/post/6844904045845577742
