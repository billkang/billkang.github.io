## VSCode 插件简介

### VSCode 底层实现

VSCode 底层通过 electron 开发实现，electron 的核心构成分别是：chromium、nodejs、native-api。
![vscode-eletron](https://raw.githubusercontent.com/billkang/billkang.github.io/master/images/2022/vscode-eletron.png)

- Chromium（UI 视图）：通过 web 技术栈编写实现 ui 界面，其与 chrome 的区别是开放开源、无需安装可直接使用（可以简单理解 chromium 是 beta 体验版 chrome，新特性会优先在 chromium 中体验并在稳定后更新至 chrome 中）。
- Nodejs（操作桌面文件系统）：通过 node-gyp 编译，主要用来操作文件系统和调用本地网络。
- Native-API（操作系统纬度 API）：使用 Nodejs-C++ Addon 调用操作系统 API（Nodejs-C++ Addon 插件是一种动态链接库，采用 C/C++语言编写，可以通过 require()将插件加载进 NodeJS 中进行使用），可以理解是对 Nodejs 接口的能力拓展。

### VSCode 插件能做什么

vscode插件其实就是类似于一个npm包的vsix文件，其可以实现的功能大概可以分为以下几类：

1. 不受限的本地磁盘访问
2. 自定义命令、快捷键、菜单
3. 自定义跳转、自动补全、悬浮提示
4. 自定义插件设置、自定义插件欢迎页
5. 自定义WebView
6. 自定义左侧功能面板
7. 自定义颜色、图标主题
8. 新增语言支持（代码高亮、语法解析、折叠、跳转、补全等）
9. Markdown增强
10. 其他（比如状态栏修改、通知提示、编辑器控制、git源代码控制、任务定义、Language Server、Debug Adapter等等）


### 开发一个简单的插件

#### 环境准备

- nodejs
- vscode
- 安装 yeoman 和 VSCode Extension Generator

```
npm install -g yo generator-code
```

#### 初始化项目工程

```
yo code
```

创建项目会开启一个交互命令行，会让你选择你想创建的插件类型，目前支持以下几种类型：
1. New Extension (TypeScript) ：ts语法的项目，基础版，内置了hello world的命令
2. New Extension (JavaScript) : js语法的项目，基础版，内置了hello world的命令
3. New Color Theme ：主题项目，内置了主题，可用来自定义主题
4. New Language Support：语言支持项目，内置了语法支持配置，可用来支持特殊语言
5. New Code Snippets：代码片段项目，内置了代码片段配置，可用来配置代码片段，输入触发字符，快速生成代码片段
6. New Keymap：快捷键项目，内置了快捷键配置，可用来自定义快捷键行为
7. New Extension Pack：插件集合项目，内置了插件集合配置，可用于定制插件集，达到快速安装一组插件
8. New Language Pack (Localization)：目前官方文档暂未查到localizations贡献的场景点


![vscode-yo-code](https://raw.githubusercontent.com/billkang/billkang.github.io/master/images/2022/vscode-yo-code.png)

项目目录结构:

```
.
├── .vscode
│   ├── launch.json     // 插件加载和调试的配置
│   └── tasks.json      // 配置TypeScript编译任务
├── .gitignore          // 忽略构建输出和node_modules文件
├── README.md           // 一个友好的插件文档
├── src
│   └── extension.ts    // 插件源代码
├── package.json        // 插件配置清单
├── tsconfig.json       // TypeScript配置

```

#### 插件清单（package.json）

每个 VSCode 插件都必须包含一个 package.json，它就是插件的配置清单。
package.json 混合了 Node.js 字段，如：scripts、dependencies，还加入了一些 VSCode 独有的字段，如：publisher、activationEvents、contributes 等。

- name 和 publisher: VSCode 使用{publisher}.{name}作为一个插件的 ID，用来区分各个不同的插件。
- main: 指定了插件的入口函数。
- activationEvents：指定触发事件，当指定事件发生时才触发插件执行。
- contributes：描述插件的拓展点，用于定义插件要扩展 vscode 哪部分功能，如 commands 命令面板、menus 资源管理面板等。
- engines.vscode: 描述了这个插件依赖的最低 VSCode API 版本。
- postinstall 脚本: 如果你的 engines.vscode 声明的是 1.25 版的 VSCode API，那它就会按照这个声明去安装目标版本。一旦 vscode.d.ts 文件存在于 node_modules/vscode/vscode.d.ts，IntelliSense 就会开始运作，你就可以对所有 VSCode API 进行定义跳转或者语法检查了。

```json
{
  "name": "cabinx-extension",
  "displayName": "cabinx-extension",
  "description": "",
  "version": "0.0.1",
  "engines": {
    "vscode": "^1.69.0"
  },
  "categories": [
    "Other"
  ],
  "activationEvents": [
    "onCommand:cabinx-extension.newCabinxPage"
  ],
  "main": "./out/extension.js",
  "contributes": {
    "commands": [
      {
        "command": "cabinx-extension.newCabinxPage",
        "title": "新建CabinX页面"
      }
    ],
    "menus": {
      "explorer/context": [
        {
          "command": "cabinx-extension.newCabinxPage",
          "group": "z_commands",
          "when": "explorerResourceIsFolder"
        }
      ]
    }
  },
  "scripts": {
    "vscode:prepublish": "npm run compile",
    "build": "tsc -p ./",
    "watch": "tsc -watch -p ./",
    "pretest": "npm run build && npm run lint",
    "lint": "eslint src --ext ts",
    "test": "node ./out/test/runTest.js"
  },
  "devDependencies": {
    ...
  }
}
```

#### 声明指令介绍

初始化插件项目成功后，会看到上图的目录结构。其中 src 目录下的 extension.ts 文件为入口文件，包含 activate 和 deactivate 分别作为插件启动和插件卸载时的生命周期函数，可以将逻辑直接写在两个函数内也可抽象后在其中调用。

我们希望插件在适当的时机启动 activate 或关闭 deactivate，vscode 也给我们提供了多种 onXXX 的事件作为多种执行时机的入口方法。我可以通过配置activationEvents实现插件触发动作。

可配置项包括以下几类：
- onLanguage：打开特定语言文件时激活事件和相关插件
- onCommand：调用命令时激活
- onDebug：调试会话(debug session)启动前激活
  - onDebugInitialConfigurations
  - onDebugResolve
- workspaceContains：文件夹打开后，且文件夹中至少包含一个符合glob模式的文件时触发
- onFileSystem： 以协议（scheme）打开文件或文件夹时触发。通常是file-协议，也可以用自定义的文件供应器函数替换掉，比如ftp、ssh.h
- onView：每当在VS Code侧栏中展开指定ID的视图时
- onUri：插件的系统级URI打开时触发。这个URI协议需要带上vscode或者 vscode-insiders协议。URI主机名必须是插件的唯一标识，剩余的URI是可选的
- onWebviewPanel： VS Code需要恢复匹配到viewType的webview视图时触发
- onCustomEditor：当VS Code需要使用匹配的viewType创建自定义编辑器时触发
- *：当VS Code启动时触发，性能会变差，不建议使用
- onStartupFinished： VS Code启动一定时间后触发，和上述*效果类似，但性能更好一些

那么我们该如何使用这些事件呢？

我们以 onCommand 为例，首先需要在 package.json 文件中注册 activationEvents 和 commands。

```json
{
  "activationEvents": ["onCommand:cabinx-extension.newCabinxPage"],
  "main": "./out/extension.js",
  "contributes": {
    "commands": [
      {
        "command": "cabinx-extension.newCabinxPage",
        "title": "新建CabinX页面"
      }
    ]
  }
}
```

添加目录右键点击事件

![vscode-menus](https://raw.githubusercontent.com/billkang/billkang.github.io/master/images/2022/vscode-menus.png)

```json
{
  "contributes": {
    "menus": {
      "explorer/context": [
        {
          "command": "cabinx-extension.newCabinxPage",
          "group": "z_commands", // 位于命令容器面板
          "when": "explorerResourceIsFolder" // 资源管理器为目录
        }
      ]
    }
  }
}
```

#### 插件入口文件（src/extension.ts）

插件入口文件会导出两个函数，activate 和 deactivate，你注册的激活事件被触发之时执行 activate，deactivate 则提供了插件关闭前执行清理工作的机会。

vscode 模块包含了一个位于 node ./node_modules/vscode/bin/install 的脚本，这个脚本会拉取 package.json 中 engines.vscode 字段定义的 VS Code API。这个脚本执行过后，你就得到了智能代码提示，定义跳转等 TS 特性了。

```ts
// extenson.ts
import * as vscode from "vscode";
import { openInputBox } from "./openInputBox";

// this method is called when your extension is activated
// your extension is activated the very first time the command is executed
export function activate(context: vscode.ExtensionContext) {
  // The command has been defined in the package.json file
  // Now provide the implementation of the command with registerCommand
  // The commandId parameter must match the command field in package.json
  let disposable = vscode.commands.registerCommand(
    "cabinx-extension.newCabinxPage",
    (file: vscode.Uri) => {
      openInputBox(file);
    }
  );

  context.subscriptions.push(disposable);
}

// this method is called when your extension is deactivated
export function deactivate() {}
```

```ts
// openInputBox.ts
import { window, Uri } from "vscode";
import { pathExists } from "fs-extra";
import { join } from "path";
import { createTemplate } from "./createTemlate";

export const openInputBox = (file: Uri): void => {
  const inputBox = window.createInputBox();

  inputBox.placeholder = "请输入页面名称，按Enter确认！";

  let filePath = "";
  const inputVal = inputBox.value;

  inputBox.onDidChangeValue(async (value: string) => {
    if (value.length < 1) {
      return "页面名称不能为空！";
    }

    filePath = join(file.fsPath, value);

    if (await pathExists(filePath)) {
      return `该${filePath}路径已经存在，请换一个名称！`;
    }
  });

  inputBox.onDidHide(() => {
    inputBox.value = "";
    inputBox.enabled = true;
    inputBox.busy = false;
  });

  inputBox.onDidAccept(async () => {
    inputBox.enabled = false;
    inputBox.busy = true;

    const result = createTemplate(filePath, inputVal);

    if (result) {
      inputBox.hide();
      window.showInformationMessage("创建成功！");
    } else {
      window.showInformationMessage("创建失败，请重试！");
    }

    inputBox.enabled = true;
    inputBox.busy = false;
  });

  inputBox.show();
};
```

#### 根据输入面板创建模板文件

```ts
import * as fs from "fs";

export const createTemplate = (path: string, name: string) => {
  const mkdirResult = fs.mkdirSync(path, {
    recursive: true,
  });

  if (!mkdirResult) {
    return false;
  }

  try {
    fs.writeFileSync(`${path}/${name}.dml`, "");
    fs.writeFileSync(`${path}/${name}.dss`, "");

    return true;
  } catch (e) {
    console.error(e);
    return false;
  }
};
```

### 如何本地调试插件

#### 使用 launch 方式

可以在项目根目录创建.vscode 文件夹，下面创建 launch.json 文件，然后开启调试功能。

```json
{
  "version": "0.2.0",
  "configurations": [
    {
      "name": "Run Extension",
      "type": "extensionHost",
      "request": "launch",
      "args": ["--extensionDevelopmentPath=${workspaceFolder}"],
      "outFiles": ["${workspaceFolder}/out/**/*.js"],
      "preLaunchTask": "${defaultBuildTask}"
    }
  ]
}
```

![vscode-extension-launch](https://raw.githubusercontent.com/billkang/billkang.github.io/master/images/2022/vscode-extension-launch.png)


1. 点击运行Run Extension后，会自动打开一个新的vscode界面，此时这个新打开的vscode，会自动安装我们开发的插件，vscode头部会展示[Extension Development Host]
2. 此时我们在这个界面调试插件功能，如果需要测试代码，我们可以在此vscode新建一个本地项目或者打开已有项目测试
3. 我们写的console相关日志，可以在DEBUG CONSOLE面板中查看，也可以在运行过程中增删断点，如下：
![vscode-extension-launch](https://raw.githubusercontent.com/billkang/billkang.github.io/master/images/2022/vscode-extension-debugger.png)
4. 如果修改了插件内容，webpack会自动监听并编译
5. 打开的[Extension Development Host]的vscode调试页面如何更新修改呢？
   - 方法一：断开并重新执行
   - 方法二：在此界面按住command+R，则会自动重启并应用，推荐


#### 安装本地构建包

![vscode-extension-install](https://raw.githubusercontent.com/billkang/billkang.github.io/master/images/2022/vscode-extension-install.png)

### 如何本地打包插件

安装 vsce 进行打包操作。

```nodejs
npm i -g vsce
vsce package
```

### 参考资料

[VSCode 插件文档](https://liiked.github.io/VS-Code-Extension-Doc-ZH/#/get-started/your-first-extension)

[vscode 插件原理浅析与实战](https://mp.weixin.qq.com/s/e0Mmed5aB39EE-RzFUQI5g)

[vscode插件开发指南(一)-理论篇](https://juejin.cn/post/6960626872791072798)

[vscode插件开发指南（二）-实战篇-搭建项目](https://juejin.cn/post/6961370277682872351)

[vscode插件开发指南（三）-实战篇-语法校验实现](https://juejin.cn/post/6961689956347543589)

[vscode插件开发指南（四）-实战篇-自动补全实现](https://juejin.cn/post/6963617930714021924)

[vscode插件开发指南（五）-实战篇-悬停语法提示&webview-完结篇](https://juejin.cn/post/6963954849671020558)