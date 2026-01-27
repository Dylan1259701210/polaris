# VS Code 扩展开发指南（中文）

本文档说明：**① 用 VS Code API 读写本地文件**；**② 如何添加自定义 @ 参与者（如 @HSBC）并配置角色/设置**；**③ 实现 @HSBC 的完整步骤**。面向做「GitHub Agent」类扩展、或扩展 Copilot Chat 的场景。

---

## 一、用 VS Code API 读写本地文件

### 1.1 推荐：`workspace.fs`

优先使用 `vscode.workspace.fs`，兼容本地、虚拟工作区、远程（SSH 等）。

| 方法 | 说明 |
|------|------|
| `readFile(uri)` | 读整个文件，返回 `Uint8Array` |
| `writeFile(uri, data)` | 写入文件，覆盖全部内容 |
| `readDirectory(uri)` | 列出目录项 `[name, FileType][]` |
| `createDirectory(uri)` | 创建目录（mkdirp 语义） |
| `stat(uri)` | 文件/目录元信息 |

**读文本示例：**

```ts
import * as vscode from 'vscode';

async function readTextFile(uri: vscode.Uri): Promise<string> {
  const bytes = await vscode.workspace.fs.readFile(uri);
  return new TextDecoder().decode(bytes);
  // 若用 workspace.fs，需自行 TextDecoder；部分环境提供 workspace 的 decode，见 API 文档。
}
```

**写文本示例：**

```ts
async function writeTextFile(uri: vscode.Uri, content: string): Promise<void> {
  const bytes = new TextEncoder().encode(content);
  await vscode.workspace.fs.writeFile(uri, bytes);
}
```

**编码/解码：** 若用 `workspace.fs` 读二进制，需自己 `TextEncoder`/`TextDecoder`。部分环境提供 `workspace.fs` 的 `decode`，可查当前 VS Code API 文档。

**构建 `file` URI：**

```ts
const uri = vscode.Uri.file("D:\\project\\README.md");
// 或从 workspaceFolder：
const folder = vscode.workspace.workspaceFolders?.[0];
const uri = folder ? vscode.Uri.joinPath(folder.uri, "README.md") : null;
```

### 1.2 使用 `TextDocument`（反映未保存修改）

需要**当前编辑器内容**（含未保存修改）时，用 `openTextDocument`：

```ts
const doc = await vscode.workspace.openTextDocument(uri);
const text = doc.getText();
const isDirty = doc.isDirty;
```

- `workspace.fs`：针对磁盘上的文件。
- `openTextDocument`：针对已打开的文档，包含未保存的修改。

### 1.3 注意

- 尽量不用 `URI.fsPath` + Node `fs`，在虚拟/远程工作区可能不可用。
- 用 `workspace.fs` 可适配多环境。

---

## 二、自定义 @ 参与者（如 @HSBC）与「角色 / 设置」

### 2.1 什么是 Chat 参与者？

- 在 Copilot Chat 里输入 **`@名称`** 可呼出自定义参与者。
- 内置的有 `@github`、`@terminal`、`@vscode`、`@workspace`。
- 扩展可以注册自己的参与者，例如 **@HSBC**，实现专属逻辑（领域知识、内部规范等）。

### 2.2 「角色」「设置」指什么？

在你说的「定义为 Agent、角色、设置」的语境下，通常包括：

| 概念 | 实现方式 |
|------|----------|
| **角色** | 在请求里加系统提示（system prompt），例如「你是一个 HSBC 内部的合规助手……」 |
| **设置** | 在扩展里读 `workspace.getConfiguration('yourExt')`，或 `package.json` 的 `contributes.configuration`，按配置改变参与者行为 |
| **Agent 感** | 参与者内部可以调用 `lm.invokeTool`、跑任务、读 `#` 上下文，实现多步推理与执行，类似 Agent |

所以：**@HSBC = 一个自定义 Chat 参与者 + 你配置的「角色」提示词 + 可选配置项**。

### 2.3 和 GitHub Coding Agent 的区别

- **VS Code Chat 参与者（如 @HSBC）**：跑在你本机/当前工作区，你与 Chat 交互，立即改本地文件。
- **GitHub Coding Agent**：任务在 GitHub 云端跑，改远程仓库并开 PR。  
你要的「@HSBC 这种命令」是前者（Chat 参与者），不是后者。

---

## 三、实现 @HSBC 的完整步骤

### 3.1 初始化扩展

```bash
npm install -g yo generator-code
yo code
# 选 New Extension (TypeScript)，填名称如 github-agent
```

### 3.2 在 `package.json` 里注册参与者

在 `package.json` 的 `contributes` 中增加 `chatParticipants`：

```json
{
  "contributes": {
    "chatParticipants": [
      {
        "id": "github-agent.hsbc",
        "name": "hsbc",
        "fullName": "HSBC Agent",
        "description": "HSBC 内部规范与合规助手",
        "isSticky": true,
        "commands": [
          {
            "name": "review",
            "description": "按 HSBC 规范审查选中代码"
          }
        ]
      }
    ]
  }
}
```

- **`name`**：用户输入 **@hsbc** 时用到。
- **`fullName`**：在 UI 里展示的完整名称。
- **`commands`**：该参与者支持的 **斜杠命令**，如 `/review`。

### 3.3 实现 Request 逻辑（含「角色」）

在 `src/extension.ts`（或你放激活逻辑的文件）里：

```ts
import * as vscode from 'vscode';

const HSBC_SYSTEM_PROMPT = `你是一个 HSBC 内部助手，熟悉 HSBC 的合规与开发规范。
回答时请引用相关规范，并给出具体、可操作的建议。`;

export function activate(context: vscode.ExtensionContext) {
  const handler: vscode.ChatRequestHandler = async (request, context, stream, token) => {
    // 1. 角色 / 系统提示
    const basePrompt = request.command === 'review'
      ? `${HSBC_SYSTEM_PROMPT}\n\n请对用户选中的代码做合规与规范审查。`
      : HSBC_SYSTEM_PROMPT;

    const messages = [
      vscode.LanguageModelChatMessage.User(basePrompt),
      vscode.LanguageModelChatMessage.User(request.prompt)
    ];

    // 2. 发请求并流式输出
    const response = await request.model.sendRequest(messages, {}, token);
    for await (const chunk of response.text) {
      stream.markdown(chunk);
    }
  };

  const participant = vscode.chat.createChatParticipant('github-agent.hsbc', handler);
  context.subscriptions.push(participant);
}
```

- **角色**：通过 `HSBC_SYSTEM_PROMPT` 注入；可再根据 `request.command` 区分 `/review` 等不同「模式」。
- **配置**：可从 `vscode.workspace.getConfiguration('githubAgent')` 读配置，动态改 `basePrompt`，实现你要的「设置」。

### 3.4 可选：读本地文件当上下文

在 `handler` 里用 `workspace.fs` / `openTextDocument` 读本地的规范文件，拼进 `basePrompt`，就能做到「根据本地配置 / 规范文件定制 @HSBC 行为」：

```ts
async function loadLocalRules(workspaceRoot: vscode.Uri): Promise<string> {
  const uri = vscode.Uri.joinPath(workspaceRoot, '.hsbc-rules.md');
  try {
    const bytes = await vscode.workspace.fs.readFile(uri);
    return new TextDecoder().decode(bytes);
  } catch {
    return '';
  }
}

// 在 handler 里：
const folder = vscode.workspace.workspaceFolders?.[0];
const extra = folder ? await loadLocalRules(folder.uri) : '';
const basePrompt = `${HSBC_SYSTEM_PROMPT}\n\n${extra}`;
```

### 3.5 可选：配置项驱动「设置」

在 `package.json` 里增加 `configuration`：

```json
{
  "contributes": {
    "configuration": {
      "title": "GitHub Agent",
      "properties": {
        "githubAgent.hsbc.strictMode": {
          "type": "boolean",
          "default": false,
          "description": "HSBC 严格合规模式"
        }
      }
    }
  }
}
```

在 handler 里读取：

```ts
const config = vscode.workspace.getConfiguration('githubAgent');
const strict = config.get<boolean>('hsbc.strictMode') ?? false;
const mode = strict ? '严格合规模式' : '常规模式';
// 把 mode 写进 basePrompt
```

---

## 四、小结

| 目标 | 做法 |
|------|------|
| **读写本地文件** | 用 `vscode.workspace.fs`（`readFile` / `writeFile`）或 `workspace.openTextDocument` |
| **加 @HSBC 这种命令** | 在 `package.json` 里 `contributes.chatParticipants`，再 `createChatParticipant` + `ChatRequestHandler` |
| **角色 / 设置** | 在 handler 里写 system prompt（角色），用 `workspace.getConfiguration` 或本地文件（设置） |
| **像 Agent** | 在参与者里调用 `lm.invokeTool`、`#` 上下文、多轮对话等 |

按上述步骤即可在 VS Code 里实现 **@HSBC**，并扩展成你想要的「Agent + 角色 + 设置」。

---

## 五、参考链接

- [VS Code 扩展 API：workspace.fs](https://code.visualstudio.com/api/references/vscode-api#FileSystem)
- [Chat Participant API](https://code.visualstudio.com/api/extension-guides/ai/chat)
- [Chat 教程](https://code.visualstudio.com/api/extension-guides/ai/chat-tutorial)
