## VSCode 插件实践

### 是什么实现了 vscode

### 什么是 VSCode 插件

##### Electron

vscode 底层通过 electron 开发实现，electron 的核心构成分别是：chromium、nodejs、native-api。
![vscode-eletron](https://raw.githubusercontent.com/billkang/billkang.github.io/master/images/2022/vscode-eletron.png)

- Chromium（UI 视图）：通过 web 技术栈编写实现 ui 界面，其与 chrome 的区别是开放开源、无需安装可直接使用（可以简单理解 chromium 是 beta 体验版 chrome，新特性会优先在 chromium 中体验并在稳定后更新至 chrome 中）。

- Nodejs（操作桌面文件系统）：通过 node-gyp 编译，主要用来操作文件系统和调用本地网络。

- Native-API（操作系统纬度 API）：使用 Nodejs-C++ Addon 调用操作系统 API（Nodejs-C++ Addon 插件是一种动态链接库，采用 C/C++语言编写，可以通过 require()将插件加载进 NodeJS 中进行使用），可以理解是对 Nodejs 接口的能力拓展。

### 插件能做什么

我们看看使用插件 API 能做些什么：

- 改变 VS Code 的颜色和图标主题
- 在 UI 中添加自定义组件和视图
- 创建 Webview，使用 HTML/CSS/JS 显示自定义网页
- 支持新的编程语言
- 支持特定运行时的调试

### 环境准备

- nodejs
- vscode
- 安装 yeoman 和 VS Code Extension Generator

```
npm install -g yo generator-code
```

- 初始化项目工程

```
yo code
```

### 具体实现

1. package.json 介绍

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

- main: 指定了插件的入口函数。

- activationEvents：指定触发事件，当指定事件发生时才触发插件执行。需额外关注\*这个特殊的插件类型，因为他在初始化完成后就会触发插件执行，并不需要任何自定义触发事件。

- contributes：描述插件的拓展点，用于定义插件要扩展 vscode 哪部分功能，如 commands 命令面板、menus 资源管理面板等。

1. 声明指令

初始化插件项目成功后会看到上图的目录结构，其中我们需要重点关注 src 目录和 package.json 文件，其中 src 目录下的 extension.ts 文件为入口文件，包含 activate 和 deactivate 分别作为插件启动和插件卸载时的生命周期函数，可以将逻辑直接写在两个函数内也可抽象后在其中调用。

同时我们希望插件在适当的时机启动 activate 或关闭 deactivate，vscode 也给我们提供了多种 onXXX 的事件作为多种执行时机的入口方法。那么我们该如何使用这些事件呢？

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

如何使用这些事件呢？我们以 onCommand 为例。首先需要在 package.json 文件中注册 activationEvents 和 commands。

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

然后在 extension.ts 文件的 activate 方法中编写自定义逻辑。

``` ts
// The module 'vscode' contains the VS Code extensibility API
// Import the module and reference it with the alias vscode in your code below
import * as vscode from "vscode";

// this method is called when your extension is activated
// your extension is activated the very first time the command is executed
export function activate(context: vscode.ExtensionContext) {}

// this method is called when your extension is deactivated
export function deactivate() {}
```

2. 添加目录右键点击事件
   ![menus](https://raw.githubusercontent.com/billkang/billkang.github.io/master/images/2022/vsode-menus.png)
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
3. 唤起组件名称输入面板
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

4. 根据输入面板创建模板文件

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

### 参考资料

[VSCode 插件文档](https://liiked.github.io/VS-Code-Extension-Doc-ZH/#/get-started/your-first-extension)

[vscode 插件原理浅析与实战](https://mp.weixin.qq.com/s/e0Mmed5aB39EE-RzFUQI5g)
