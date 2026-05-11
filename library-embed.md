# Excalidraw 素材库与嵌入资源加载限流机制

## 1. 素材库文件解析 (Library)

### 1.1 核心模块位置
- 主文件: `packages/excalidraw/data/library.ts`
- 类型定义: `packages/excalidraw/types.ts`
- 解析器: `packages/excalidraw/data/restore.ts`

### 1.2 素材库 URL 白名单验证

```typescript
// 允许的素材库 URL 域名列表 (library.ts:54-58)
const ALLOWED_LIBRARY_URLS = [
  "excalidraw.com",
  // 当从 GitHub PR 安装时允许
  "raw.githubusercontent.com/excalidraw/excalidraw-libraries",
];
```

### 1.3 URL 验证机制

```typescript
// library.ts:497-528
export const validateLibraryUrl = (
  libraryUrl: string,
  validator: ((libraryUrl: string) => boolean) | string[] = ALLOWED_LIBRARY_URLS,
): true => {
  if (
    typeof validator === "function"
      ? validator(libraryUrl)
      : validator.some((allowedUrlDef) => {
          const allowedUrl = new URL(
            `https://${allowedUrlDef.replace(/^https?:\/\//, "")}`,
          );

          const { hostname, pathname } = new URL(libraryUrl);

          return (
            // 主机名部分匹配（支持子域名）
            new RegExp(`(^|\\.)${allowedUrl.hostname}$`).test(hostname) &&
            // 路径前缀匹配
            new RegExp(
              `^${allowedUrl.pathname.replace(/\/+$/, "")}(/+|$)`,
            ).test(pathname)
          );
        })
  ) {
    return true;
  }

  throw new Error(`Invalid or disallowed library URL: "${libraryUrl}"`);
};
```

### 1.4 从 URL 导入素材库流程

```typescript
// library.ts:718-779
const importLibraryFromURL = async ({
  libraryUrl,
  idToken,
}: {
  libraryUrl: string;
  idToken: string | null;
}) => {
  // 1. 解码并验证 URL
  libraryUrl = decodeURIComponent(libraryUrl);
  libraryUrl = toValidURL(libraryUrl);
  validateLibraryUrl(libraryUrl, optsRef.current.validateLibraryUrl);

  // 2. 异步获取素材库 Blob
  const request = await fetch(libraryUrl);
  const blob = await request.blob();

  // 3. 基于 Token 判断是否需要用户确认
  const shouldPrompt = idToken !== excalidrawAPI.id;

  // 4. 更新素材库（支持合并模式）
  await excalidrawAPI.updateLibrary({
    libraryItems: libraryPromise,
    prompt: shouldPrompt,
    merge: true,
    defaultStatus: "published",
    openLibraryMenu: true,
  });
};
```

### 1.5 素材库更新队列机制

```typescript
// library.ts:351-400
setLibrary = (
  libraryItems: LibraryItems | Promise<LibraryItems> | ((latestLibraryItems) => LibraryItems | Promise<LibraryItems>),
): Promise<LibraryItems> => {
  const task = new Promise<LibraryItems>(async (resolve, reject) => {
    // 等待前一个任务完成，避免竞态条件
    await this.getLastUpdateTask();

    if (typeof libraryItems === "function") {
      libraryItems = libraryItems(this.currLibraryItems);
    }

    this.currLibraryItems = cloneLibraryItems(await libraryItems);
    resolve(this.currLibraryItems);
  })
    .catch((error) => {
      if (error.name === "AbortError") {
        console.warn("Library update aborted by user");
        return this.currLibraryItems;
      }
      throw error;
    })
    .finally(() => {
      this.updateQueue = this.updateQueue.filter((_task) => _task !== task);
      this.notifyListeners();
    });

  this.updateQueue.push(task);
  this.notifyListeners();

  return task;
};
```

---

## 2. URL 拖拽白名单 (Embeddable)

### 2.1 核心模块位置
- 主文件: `packages/element/src/embeddable.ts`
- 支持的平台: YouTube, Vimeo, Google Drive, Figma, Twitter, Reddit, GitHub Gist, 等

### 2.2 允许嵌入的域名白名单

```typescript
// embeddable.ts:133-150
const ALLOWED_DOMAINS = new Set([
  "youtube.com",
  "youtu.be",
  "vimeo.com",
  "player.vimeo.com",
  "drive.google.com",
  "figma.com",
  "link.excalidraw.com",
  "gist.github.com",
  "twitter.com",
  "x.com",
  "*.simplepdf.eu",      // 通配符支持
  "stackblitz.com",
  "val.town",
  "giphy.com",
  "reddit.com",
  "forms.microsoft.com",
]);
```

### 2.3 域名匹配算法

```typescript
// embeddable.ts:439-472
const matchHostname = (
  url: string,
  allowedHostnames: Set<string> | string,
): string | null => {
  try {
    const { hostname } = new URL(url);
    const bareDomain = hostname.replace(/^www\./, "");

    if (allowedHostnames instanceof Set) {
      // 精确匹配
      if (ALLOWED_DOMAINS.has(bareDomain)) {
        return bareDomain;
      }

      // 通配符匹配（如 *.simplepdf.eu）
      const bareDomainWithFirstSubdomainWildcarded = bareDomain.replace(
        /^([^.]+)/,
        "*",
      );
      if (ALLOWED_DOMAINS.has(bareDomainWithFirstSubdomainWildcarded)) {
        return bareDomainWithFirstSubdomainWildcarded;
      }
      return null;
    }

    const bareAllowedHostname = allowedHostnames.replace(/^www\./, "");
    if (bareDomain === bareAllowedHostname) {
      return bareAllowedHostname;
    }
  } catch (error) {
    // ignore
  }
  return null;
};
```

### 2.4 自定义验证器支持

```typescript
// embeddable.ts:502-535
export const embeddableURLValidator = (
  url: string | null | undefined,
  validateEmbeddable: ExcalidrawProps["validateEmbeddable"],
): boolean => {
  if (!url) {
    return false;
  }
  
  // 支持多种验证方式
  if (validateEmbeddable != null) {
    // 1. 自定义函数
    if (typeof validateEmbeddable === "function") {
      const ret = validateEmbeddable(url);
      if (typeof ret === "boolean") {
        return ret;
      }
    } 
    // 2. 布尔值开关
    else if (typeof validateEmbeddable === "boolean") {
      return validateEmbeddable;
    } 
    // 3. 正则表达式
    else if (validateEmbeddable instanceof RegExp) {
      return validateEmbeddable.test(url);
    } 
    // 4. 域名数组（支持混合正则和字符串）
    else if (Array.isArray(validateEmbeddable)) {
      for (const domain of validateEmbeddable) {
        if (domain instanceof RegExp) {
          if (url.match(domain)) {
            return true;
          }
        } else if (matchHostname(url, domain)) {
          return true;
        }
      }
      return false;
    }
  }

  // 默认使用白名单
  return !!matchHostname(url, ALLOWED_DOMAINS);
};
```

---

## 3. iframe 沙箱与跨域消息

### 3.1 iframe 渲染配置

```typescript
// App.tsx:1832-1854
<iframe
  ref={(ref) => this.cacheEmbeddableRef(el, ref)}
  className="excalidraw__embeddable"
  // 文档类型嵌入使用 srcDoc
  srcDoc={
    src?.type === "document"
      ? src.srcdoc(this.state.theme)
      : undefined
  }
  // 普通嵌入使用 src
  src={
    src?.type !== "document" ? src?.link ?? "" : undefined
  }
  scrolling="no"
  referrerPolicy="no-referrer-when-downgrade"
  title="Excalidraw Embedded Content"
  // 权限策略
  allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture"
  allowFullScreen={true}
  // 沙箱配置
  sandbox={`${
    src?.sandbox?.allowSameOrigin
      ? "allow-same-origin"
      : ""
  } allow-scripts allow-forms allow-popups allow-popups-to-escape-sandbox allow-presentation allow-downloads`}
/>
```

### 3.2 沙箱权限详解

| 权限 | 说明 | 配置条件 |
|------|------|----------|
| `allow-same-origin` | 允许 iframe 内容视为同源 | 仅白名单内特定域名（YouTube, Vimeo, Google Drive, Figma 等） |
| `allow-scripts` | 允许执行 JavaScript | 默认启用 |
| `allow-forms` | 允许提交表单 | 默认启用 |
| `allow-popups` | 允许弹出窗口 | 默认启用 |
| `allow-popups-to-escape-sandbox` | 允许弹出窗口脱离沙箱 | 默认启用 |
| `allow-presentation` | 允许演示模式（如全屏） | 默认启用 |
| `allow-downloads` | 允许下载 | 默认启用 |

### 3.3 同源权限控制

```typescript
// embeddable.ts:152-165
const ALLOW_SAME_ORIGIN = new Set([
  "youtube.com",
  "youtu.be",
  "vimeo.com",
  "player.vimeo.com",
  "drive.google.com",
  "figma.com",
  "twitter.com",
  "x.com",
  "*.simplepdf.eu",
  "stackblitz.com",
  "reddit.com",
  "forms.microsoft.com",
]);
```

### 3.4 跨域消息通信 (ExcalidrawPlus)

```typescript
// ExcalidrawPlusIframeExport.tsx:157-217
const handleMessage = async (event: MessageEvent<MESSAGE_FROM_PLUS>) => {
  // 1. 严格验证来源
  if (event.origin !== EXCALIDRAW_PLUS_ORIGIN) {
    throw new ExcalidrawError("Invalid origin");
  }

  if (event.data.type === EVENT_REQUEST_SCENE) {
    // 2. JWT 令牌验证
    try {
      await verifyJWT({
        token: event.data.jwt,
        publicKey: import.meta.env.VITE_APP_PLUS_EXPORT_PUBLIC_KEY,
      });
    } catch (error: any) {
      throw new ExcalidrawError("Failed to verify JWT");
    }

    // 3. 解析场景数据
    const parsedSceneData = await parseSceneData({...});

    // 4. 发送响应（仅限指定来源）
    event.source!.postMessage(parsedSceneData, {
      targetOrigin: EXCALIDRAW_PLUS_ORIGIN,
    });
  }
};

window.addEventListener("message", handleMessage);
```

### 3.5 JWT 验证流程

```typescript
// ExcalidrawPlusIframeExport.tsx:89-151
const verifyJWT = async ({ token, publicKey }: { token: string; publicKey: string }) => {
  // 1. 解析 JWT 结构
  const [header, payload, signature] = token.split(".");

  // 2. Base64URL 解码
  const decodedPayload = base64urlToString(payload);
  const decodedSignature = base64urlToString(signature);

  // 3. 导入公钥
  const key = await crypto.subtle.importKey(
    "spki",
    keyArrayBuffer,
    { name: "RSASSA-PKCS1-v1_5", hash: "SHA-256" },
    true,
    ["verify"],
  );

  // 4. 验证签名
  const isValid = await crypto.subtle.verify(
    "RSASSA-PKCS1-v1_5",
    key,
    signatureArrayBuffer,
    new TextEncoder().encode(data),
  );

  // 5. 检查过期时间
  const parsedPayload = JSON.parse(decodedPayload);
  const currentTime = Math.floor(Date.now() / 1000);
  if (parsedPayload.exp && parsedPayload.exp < currentTime) {
    throw new Error("JWT has expired");
  }
};
```

### 3.6 YouTube/Vimeo 播放器控制

```typescript
// App.tsx:1414-1470
if (iframe.src.includes("youtube")) {
  const state = YOUTUBE_VIDEO_STATES.get(iframeLikeElement.id);
  // 播放/暂停控制
  iframe.contentWindow.postMessage(
    JSON.stringify({
      event: "command",
      func: state === 1 ? "pauseVideo" : "playVideo",
      id: iframeLikeElement.id,
    }),
    "*",
  );
}

if (iframe.src.includes("player.vimeo.com")) {
  iframe.contentWindow.postMessage(
    JSON.stringify({ method: "play" }),
    "*",
  );
}
```

---

## 4. 安全机制总结

### 4.1 素材库安全
- **URL 白名单**: 仅允许从 excalidraw.com 和 GitHub 官方仓库导入
- **域名部分匹配**: 支持子域名但限制路径前缀
- **用户确认**: 跨实例导入时需要用户确认（通过 idToken 验证）
- **更新队列**: 异步更新队列防止竞态条件

### 4.2 嵌入资源安全
- **域名白名单**: 仅允许可信平台嵌入
- **通配符支持**: 灵活的子域名匹配
- **自定义验证**: 支持宿主应用扩展验证逻辑
- **沙箱隔离**: 默认严格的 iframe 沙箱策略
- **同源控制**: 仅特定白名单域名允许 `allow-same-origin`

### 4.3 跨域通信安全
- **来源验证**: 严格检查 `event.origin`
- **JWT 认证**: 基于 RSA-SHA256 的签名验证
- **过期检查**: 令牌过期时间验证
- **目标限制**: `postMessage` 指定 `targetOrigin`
