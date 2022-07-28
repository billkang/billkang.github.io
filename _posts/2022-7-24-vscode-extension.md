## VSCode 插件简介

### VSCode底层实现

VSCode 底层通过 electron 开发实现，electron 的核心构成分别是：chromium、nodejs、native-api。
![vscode-eletron](https://raw.githubusercontent.com/billkang/billkang.github.io/master/images/2022/vscode-eletron.png)

- Chromium（UI 视图）：通过 web 技术栈编写实现 ui 界面，其与 chrome 的区别是开放开源、无需安装可直接使用（可以简单理解 chromium 是 beta 体验版 chrome，新特性会优先在 chromium 中体验并在稳定后更新至 chrome 中）。
- Nodejs（操作桌面文件系统）：通过 node-gyp 编译，主要用来操作文件系统和调用本地网络。
- Native-API（操作系统纬度 API）：使用 Nodejs-C++ Addon 调用操作系统 API（Nodejs-C++ Addon 插件是一种动态链接库，采用 C/C++语言编写，可以通过 require()将插件加载进 NodeJS 中进行使用），可以理解是对 Nodejs 接口的能力拓展。


### VSCode插件能做什么

- 改变 VSCode 的颜色和图标主题
- 在 UI 中添加自定义组件和视图
- 创建 Webview，使用 HTML/CSS/JS 显示自定义网页
- 支持新的编程语言
- 支持特定运行时的调试


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

每个 VSCode插件都必须包含一个package.json，它就是插件的配置清单。
package.json混合了Node.js字段，如：scripts、dependencies，还加入了一些VSCode独有的字段，如：publisher、activationEvents、contributes等。

- name 和 publisher: VSCode 使用{publisher}.{name}作为一个插件的ID，用来区分各个不同的插件。
- main: 指定了插件的入口函数。
- activationEvents：指定触发事件，当指定事件发生时才触发插件执行。
- contributes：描述插件的拓展点，用于定义插件要扩展 vscode 哪部分功能，如 commands 命令面板、menus 资源管理面板等。
- engines.vscode: 描述了这个插件依赖的最低VSCode API版本。
- postinstall 脚本: 如果你的engines.vscode声明的是1.25版的VSCode API，那它就会按照这个声明去安装目标版本。一旦vscode.d.ts文件存在于node_modules/vscode/vscode.d.ts，IntelliSense就会开始运作，你就可以对所有VSCode API进行定义跳转或者语法检查了。


``` json
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

我们希望插件在适当的时机启动 activate 或关闭 deactivate，vscode 也给我们提供了多种 onXXX 的事件作为多种执行时机的入口方法。

事件列表：

``` js
// 当打开特定语言时，插件被激活
onLanguage
// 当调用命令时，插件被激活
onCommand
// 当调试时，插件被激活
onDebug
// 打开文件夹且该文件夹包含设置的文件名模式时，插件被激活
workspaceContains
// 每当读取特定文件夹 or 文件时，插件被激活
onFileSystem
// 在侧边栏展开指定id的视图时，插件被激活
onView
// 在基于vscode或 vscode-insiders协议的url打开时（类似schema），插件被激活
onUri
// 在打开自定义设置viewType的 webview 时，插件被激活
onWebviewPanel
// 在打开自定义设置viewType的自定义编辑器，插件被激活
onCustomEditor
// 每当扩展请求具有authentication.getSession()匹配的providerId时，插件被激活
onAuthenticationRequest
// 在vscode启动一段时间后，插件被激活，类似 * 但不会影响vscode启动速度
onStartupFinished
// 在所有插件都被激活后，插件被激活，会影响vscode启动速度，不推荐使用
```

那么我们该如何使用这些事件呢？

我们以 onCommand 为例，首先需要在 package.json 文件中注册 activationEvents 和 commands。

``` json
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

``` json
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
  },
}
```

#### 插件入口文件（src/extension.ts）

插件入口文件会导出两个函数，activate 和 deactivate，你注册的激活事件被触发之时执行activate，deactivate则提供了插件关闭前执行清理工作的机会。

vscode模块包含了一个位于node ./node_modules/vscode/bin/install的脚本，这个脚本会拉取package.json中engines.vscode字段定义的VS Code API。这个脚本执行过后，你就得到了智能代码提示，定义跳转等TS特性了。

``` ts
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

``` ts
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

``` ts
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

#### 使用launch方式
可以在项目根目录创建.vscode文件夹，下面创建launch.json文件，然后开启调试功能。
``` json
{
	"version": "0.2.0",
	"configurations": [
		{
			"name": "Run Extension",
			"type": "extensionHost",
			"request": "launch",
			"args": [
				"--extensionDevelopmentPath=${workspaceFolder}"
			],
			"outFiles": [
				"${workspaceFolder}/out/**/*.js"
			],
			"preLaunchTask": "${defaultBuildTask}"
		}
	]
}
```

![vscode-extension-launch](https://raw.githubusercontent.com/billkang/billkang.github.io/master/images/2022/vscode-extension-launch.png)

#### 安装本地构建包

![vscode-extension-install](https://raw.githubusercontent.com/billkang/billkang.github.io/master/images/2022/vscode-extension-install.png)


### 如何本地打包插件

安装vsce进行打包操作。

``` nodejs
npm i -g vsce
vsce package
```

### 参考资料

[VSCode 插件文档](https://liiked.github.io/VS-Code-Extension-Doc-ZH/#/get-started/your-first-extension)

[vscode 插件原理浅析与实战](https://mp.weixin.qq.com/s/e0Mmed5aB39EE-RzFUQI5g)
