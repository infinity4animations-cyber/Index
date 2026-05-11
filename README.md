# Build your ChatGPT UI

This repository now contains a practical guide for building a ChatGPT-compatible UI component using the MCP Apps bridge.

## Overview

UI components render structured MCP tool results in a human-friendly interface. In ChatGPT:

- Components run inside an iframe.
- The host bridge uses JSON-RPC 2.0 over `postMessage`.
- Your UI can receive tool input/output, call tools, and update model-visible context.
- `window.openai` remains available for Apps SDK compatibility plus optional ChatGPT-only extensions.

## Recommended architecture

Use the **MCP Apps bridge** as your default integration path:

- Notifications:
  - `ui/notifications/tool-input`
  - `ui/notifications/tool-result`
- Requests:
  - `tools/call`
  - `ui/message`
  - `ui/update-model-context`

### Listen for tool results

```ts
window.addEventListener(
  "message",
  (event) => {
    if (event.source !== window.parent) return;
    const message = event.data;
    if (!message || message.jsonrpc !== "2.0") return;
    if (message.method !== "ui/notifications/tool-result") return;

    const data = message.params?.structuredContent;
    // Re-render your UI from `data`.
  },
  { passive: true }
);
```

### Call a tool from the iframe

Send a JSON-RPC request to `tools/call` via `postMessage`, and ensure the tool is visible to the UI in its descriptor.

### Ask ChatGPT to post a follow-up message

```ts
window.parent.postMessage(
  {
    jsonrpc: "2.0",
    method: "ui/message",
    params: {
      role: "user",
      content: [
        { type: "text", text: "Draft a tasting itinerary for my picks." },
      ],
    },
  },
  "*"
);
```

### Update model-visible context

```ts
await rpcRequest("ui/update-model-context", {
  content: [{ type: "text", text: "User selected 3 items." }],
});
```

## Best practice: decouple data tools from render tools

To avoid frequent remount/re-render noise:

1. **Data tool** fetches/computes data and returns `structuredContent`.
2. The model inspects and refines that data.
3. **Render tool** receives final prepared input and returns widget output template metadata.

### Why this helps

- More deliberate model reasoning before render.
- Better reuse of data tools.
- Less unnecessary widget remounting.
- Cleaner separation of business logic vs presentation.

## `window.openai` usage

Use `window.openai` as a compatibility layer and for ChatGPT extensions when needed:

- `callTool(...)`
- `uploadFile(file, { library?: boolean })`
- `selectFiles()`
- `getFileDownloadUrl({ fileId })`
- `requestClose()`
- `requestDisplayMode({ mode })`
- `requestModal({ template? })`

Design baseline behavior around MCP Apps standard APIs first; feature-detect ChatGPT-specific helpers.

## Suggested project layout

```text
app/
  server/            # MCP server (Node or Python)
  web/               # UI source
    package.json
    tsconfig.json
    src/component.tsx
    dist/component.js
```

## Scaffold the web component

```bash
cd app/web
npm init -y
npm install react@^18 react-dom@^18
npm install -D typescript esbuild
```

## Bundle step

`package.json` build script:

```json
{
  "scripts": {
    "build": "esbuild src/component.tsx --bundle --format=esm --outfile=dist/component.js"
  }
}
```

Then run:

```bash
npm run build
```

## React helper hook for tool results

```tsx
type ToolResult = { structuredContent?: unknown } | null;

export function useToolResult() {
  const [toolResult, setToolResult] = useState<ToolResult>(null);

  useEffect(() => {
    const onMessage = (event: MessageEvent) => {
      if (event.source !== window.parent) return;
      const message = event.data;
      if (!message || message.jsonrpc !== "2.0") return;
      if (message.method !== "ui/notifications/tool-result") return;
      setToolResult(message.params ?? null);
    };

    window.addEventListener("message", onMessage, { passive: true });
    return () => window.removeEventListener("message", onMessage);
  }, []);

  return toolResult;
}
```

## Localization

Read locale from `document.documentElement.lang` and load message catalogs accordingly.

## References

- MCP Apps compatibility in ChatGPT:
  https://developers.openai.com/apps-sdk/mcp-apps-in-chatgpt
- Apps SDK quickstart:
  https://developers.openai.com/apps-sdk/quickstart#build-a-web-component
- Apps SDK examples:
  https://developers.openai.com/apps-sdk/build/examples
- Apps SDK `window.openai` reference:
  https://developers.openai.com/apps-sdk/reference#windowopenai-component-bridge
- Examples repository:
  https://github.com/openai/openai-apps-sdk-examples
