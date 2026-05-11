# Excalidraw 外部资源加载限流分析报告

## 概述

本文档分析 Excalidraw 中两类外部资源加载链路的完整流程与安全机制：

1. **Library 素材库加载** - 从 URL 或本地存储导入素材
2. **iframe 嵌入资源** - 嵌入第三方平台的可嵌入内容

两条链路都遵循：**入口 → 校验 → 拉取 → 排队/缓存 → 失败回退** 的完整流程。

---

## 第一部分：Library 素材库加载链路

### 1. 入口分析

| 入口类型 | 触发方式 | 代码位置 |
|---------|---------|---------|
| URL Hash 导入 | `#addLibrary=<url>&token=<id>` | `library.ts:718-779` |
| URL Query 导入 | `?addLibrary=<url>` | `library.ts:530-543` |
| 本地存储初始化 | `getInitialLibraryItems()` / Adapter | `library.ts:806-918` |
| API 编程式导入 | `excalidrawAPI.updateLibrary()` | `library.ts:287-349` |

**入口代码示例**：
```typescript
// library.ts:530-543
export const parseLibraryTokensFromUrl = () => {
  const libraryUrl =
    // 优先从 hash 读取
    new URLSearchParams(window.location.hash.slice(1)).get(
      URL_HASH_KEYS.addLibrary,
    ) ||
    // 降级从 query 读取
    new URLSearchParams(window.location.search).get(URL_QUERY_KEYS.addLibrary);
  const idToken = libraryUrl
    ? new URLSearchParams(window.location.hash.slice(1)).get("token")
    : null;
  return libraryUrl ? { libraryUrl, idToken } : null;
};
```

---

### 2. 校验机制

#### 2.1 URL 白名单校验

```typescript
// library.ts:54-58
const ALLOWED_LIBRARY_URLS = [
  "excalidraw.com",                          // 官方域名
  "raw.githubusercontent.com/excalidraw/excalidraw-libraries",  // GitHub 官方仓库
];

// library.ts:497-528
export const validateLibraryUrl = (
  libraryUrl: string,
  validator: ((libraryUrl: string) => boolean) | string[] = ALLOWED_LIBRARY_URLS,
): true => {
  if (typeof validator === "function") {
    return validator(libraryUrl);  // 自定义验证器
  }
  
  return validator.some((allowedUrlDef) => {
    const allowedUrl = new URL(`https://${allowedUrlDef.replace(/^https?:\/\//, "")}`);
    const { hostname, pathname } = new URL(libraryUrl);

    return (
      // 主机名部分匹配：(^|\\.)domain.com$
      new RegExp(`(^|\\.)${allowedUrl.hostname}$`).test(hostname) &&
      // 路径前缀匹配：^path(/+|$)
      new RegExp(`^${allowedUrl.pathname.replace(/\/+$/, "")}(/+|$)`).test(pathname)
    );
  });
};
```

#### 2.2 跨实例安全校验

```typescript
// library.ts:725-779
const importLibraryFromURL = async ({ libraryUrl, idToken }) => {
  // 跨实例时需要用户确认
  const shouldPrompt = idToken !== excalidrawAPI.id;
  
  // 等待窗口聚焦后再提示用户
  await (shouldPrompt && document.hidden
    ? new Promise<void>((resolve) => {
        window.addEventListener("focus", () => resolve(), { once: true });
      })
    : null);

  await excalidrawAPI.updateLibrary({
    libraryItems: libraryPromise,
    prompt: shouldPrompt,  // 用户确认开关
    merge: true,
    defaultStatus: "published",
    openLibraryMenu: true,
  });
};
```

---

### 3. 拉取与解析

```typescript
// library.ts:725-739
const libraryPromise = new Promise<Blob>(async (resolve, reject) => {
  try {
    libraryUrl = decodeURIComponent(libraryUrl);  // 1. URL 解码
    libraryUrl = toValidURL(libraryUrl);           // 2. URL 规范化
    validateLibraryUrl(libraryUrl, optsRef.current.validateLibraryUrl);  // 3. 安全校验
    
    const request = await fetch(libraryUrl);       // 4. 网络请求
    const blob = await request.blob();             // 5. Blob 转换
    resolve(blob);
  } catch (error: any) {
    reject(error);
  }
});
```

**解析流程**：
1. `loadLibraryFromBlob(blob)` - 从 Blob 反序列化素材库
2. `restoreLibraryItems()` - 版本兼容修复与数据规范化
3. `mergeLibraryItems()` - 合并策略：新项前置，去重

---

### 4. 排队机制（防止竞态条件）

```typescript
// library.ts:210, 351-400
private updateQueue: Promise<LibraryItems>[] = [];

setLibrary = (libraryItems) => {
  const task = new Promise<LibraryItems>(async (resolve, reject) => {
    // 关键：等待前一个任务完成，串行化更新
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
        return this.currLibraryItems;  // 用户取消：保持原状态
      }
      throw error;
    })
    .finally(() => {
      // 任务完成后从队列移除
      this.updateQueue = this.updateQueue.filter((_task) => _task !== task);
      this.notifyListeners();
    });

  this.updateQueue.push(task);  // 入队
  this.notifyListeners();       // 通知状态更新（loading）

  return task;
};
```

**持久化队列**：
```typescript
// library.ts:546-568
class AdapterTransaction {
  static queue = new Queue();  // 单例队列，串行化所有持久化操作

  static async getLibraryItems(adapter, source, _queue = true) {
    const task = () => new Promise(...);
    if (_queue) {
      return AdapterTransaction.queue.push(task);  // 排队执行
    }
    return task();
  }
}
```

---

### 5. 缓存机制

```typescript
// useLibraryItemSvg.ts:10-12
export type SvgCache = Map<LibraryItem["id"], SVGSVGElement>;
export const libraryItemSvgsCache = atom<SvgCache>(new Map());

// useLibraryItemSvg.ts:29-65
export const useLibraryItemSvg = (id, elements, svgCache, ref) => {
  useEffect(() => {
    if (elements && id) {
      const cachedSvg = svgCache.get(id);
      if (cachedSvg) {
        setSvg(cachedSvg);  // 缓存命中
      } else {
        // 缓存未命中：异步导出 SVG 并缓存
        (async () => {
          const exportedSvg = await exportLibraryItemToSvg(elements);
          exportedSvg.querySelector(".style-fonts")?.remove();
          if (exportedSvg) {
            svgCache.set(id, exportedSvg);  // 写入缓存
            setSvg(exportedSvg);
          }
        })();
      }
    }
  }, [id, elements, svgCache]);
};
```

**缓存策略**：
- **Key**: LibraryItem ID
- **Value**: 预处理后的 SVG 元素（已移除字体样式）
- **生命周期**: 随编辑器实例销毁
- **清除时机**: `library.destroy()` 调用时

---

### 6. 失败回退

| 失败场景 | 回退策略 | 代码位置 |
|---------|---------|---------|
| URL 校验失败 | 抛出错误，显示错误消息 | `library.ts:761-767` |
| 网络请求失败 | 中止导入，保持原状态 | `library.ts:736-737` |
| 用户取消确认 | `AbortError` 静默失败 | `library.ts:384-388` |
| 持久化失败 | 显示保存错误，内存数据保留 | `library.ts:964-977` |
| 迁移失败 | 回退到旧存储数据加载 | `library.ts:882-894` |

**失败回退代码**：
```typescript
// library.ts:384-388
.catch((error) => {
  if (error.name === "AbortError") {
    console.warn("Library update aborted by user");
    return this.currLibraryItems;  // 静默回退：不改变当前状态
  }
  throw error;
})

// library.ts:964-977
.catch((error: any) => {
  console.error(`couldn't persist library update: ${error.message}`);
  if (isLoaded && optsRef.current.excalidrawAPI) {
    optsRef.current.excalidrawAPI.updateScene({
      appState: { errorMessage: t("errors.saveLibraryError") },
    });
  }
})
```

---

## 第二部分：iframe 嵌入资源加载链路

### 1. 入口分析

| 入口类型 | 触发方式 | 代码位置 |
|---------|---------|---------|
| 拖拽/粘贴 URL | 拖入或粘贴支持的平台链接 | `App.tsx: 9139` |
| 工具栏嵌入工具 | 选择 embeddable 工具后输入 URL | `actions/actionEmbeddable.ts` |
| 场景恢复加载 | 从保存的场景恢复 iframe 元素 | `data/restore.ts` |

---

### 2. 校验机制

#### 2.1 域名白名单

```typescript
// embeddable.ts:133-150
const ALLOWED_DOMAINS = new Set([
  "youtube.com", "youtu.be",                    // YouTube
  "vimeo.com", "player.vimeo.com",              // Vimeo
  "drive.google.com",                           // Google Drive
  "figma.com", "link.excalidraw.com",           // Figma / Excalidraw
  "gist.github.com",                            // GitHub Gist
  "twitter.com", "x.com",                       // Twitter
  "*.simplepdf.eu", "stackblitz.com",           // 开发工具
  "val.town", "giphy.com",                      // 其他平台
  "reddit.com", "forms.microsoft.com",          // 社区 / 表单
]);
```

#### 2.2 域名匹配算法

```typescript
// embeddable.ts:439-472
const matchHostname = (url: string, allowedHostnames: Set<string> | string) => {
  try {
    const { hostname } = new URL(url);
    const bareDomain = hostname.replace(/^www\./, "");

    if (allowedHostnames instanceof Set) {
      // 1. 精确匹配
      if (ALLOWED_DOMAINS.has(bareDomain)) {
        return bareDomain;
      }
      // 2. 通配符匹配：如 *.simplepdf.eu
      const bareDomainWithFirstSubdomainWildcarded = bareDomain.replace(/^([^.]+)/, "*");
      if (ALLOWED_DOMAINS.has(bareDomainWithFirstSubdomainWildcarded)) {
        return bareDomainWithFirstSubdomainWildcarded;
      }
      return null;
    }

    const bareAllowedHostname = allowedHostnames.replace(/^www\./, "");
    return bareDomain === bareAllowedHostname ? bareAllowedHostname : null;
  } catch (error) {
    return null;
  }
};
```

#### 2.3 自定义验证器支持

```typescript
// embeddable.ts:502-535
export const embeddableURLValidator = (url, validateEmbeddable) => {
  if (!url) return false;

  if (validateEmbeddable != null) {
    if (typeof validateEmbeddable === "function") {
      const ret = validateEmbeddable(url);
      if (typeof ret === "boolean") return ret;
    } else if (typeof validateEmbeddable === "boolean") {
      return validateEmbeddable;  // 全局开关：允许全部 / 禁止全部
    } else if (validateEmbeddable instanceof RegExp) {
      return validateEmbeddable.test(url);  // 正则匹配
    } else if (Array.isArray(validateEmbeddable)) {
      // 混合数组：支持正则 + 字符串域名
      for (const domain of validateEmbeddable) {
        if (domain instanceof RegExp && url.match(domain)) return true;
        if (matchHostname(url, domain)) return true;
      }
      return false;
    }
  }

  // 默认白名单校验
  return !!matchHostname(url, ALLOWED_DOMAINS);
};
```

---

### 3. 拉取与 URL 规范化

```typescript
// embeddable.ts:171-400
export const getEmbedLink = (link: string): IframeDataWithSandbox | null => {
  if (!link) return null;

  // 1. 缓存检查
  if (embeddedLinkCache.has(link)) {
    return embeddedLinkCache.get(link)!;
  }

  // 2. 同源权限检查
  const allowSameOrigin = ALLOW_SAME_ORIGIN.has(
    matchHostname(link, ALLOW_SAME_ORIGIN) || "",
  );

  // 3. 平台特定 URL 规范化（示例：YouTube）
  const ytLink = link.match(RE_YOUTUBE);
  if (ytLink?.[2]) {
    const startTime = parseYouTubeLikeTimestamp(originalLink);
    const time = startTime > 0 ? `&start=${startTime}` : ``;
    link = `https://www.youtube.com/embed/${ytLink[2]}?enablejsapi=1${time}`;
    aspectRatio = isPortrait ? { w: 315, h: 560 } : { w: 560, h: 315 };
    
    const result = { link, intrinsicSize: aspectRatio, type: "video", sandbox: { allowSameOrigin } };
    embeddedLinkCache.set(originalLink, result);
    return result;
  }

  // ... 其他平台解析逻辑 (Vimeo, Google Drive, Figma, Twitter, Reddit, Gist 等)
};
```

---

### 4. 缓存机制

```typescript
// embeddable.ts:23
const embeddedLinkCache = new Map<string, IframeDataWithSandbox>();
```

**缓存策略**：
- **Key**: 原始 URL 字符串
- **Value**: `{ link, intrinsicSize, type, sandbox, error? }`
- **特性**: 内存缓存，页面刷新后失效
- **命中时机**: 同一场景内重复嵌入相同 URL 时

---

### 5. iframe 渲染与沙箱隔离

```typescript
// App.tsx:1832-1854
<iframe
  ref={(ref) => this.cacheEmbeddableRef(el, ref)}
  className="excalidraw__embeddable"
  // 文档型嵌入使用内联 srcDoc（Twitter, Reddit, Gist）
  srcDoc={src?.type === "document" ? src.srcdoc(this.state.theme) : undefined}
  // 普通嵌入使用 src 属性
  src={src?.type !== "document" ? src?.link ?? "" : undefined}
  scrolling="no"
  referrerPolicy="no-referrer-when-downgrade"
  title="Excalidraw Embedded Content"
  // Feature Policy 权限
  allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture"
  allowFullScreen={true}
  // 沙箱权限配置
  sandbox={`${
    src?.sandbox?.allowSameOrigin ? "allow-same-origin" : ""
  } allow-scripts allow-forms allow-popups allow-popups-to-escape-sandbox allow-presentation allow-downloads`}
/>
```

#### 沙箱权限详解

| 权限 | 说明 | 配置条件 |
|------|------|----------|
| `allow-same-origin` | 允许 iframe 内容视为同源 | **仅白名单内特定域名**：YouTube, Vimeo, Google Drive, Figma, Twitter, Reddit, Stackblitz, Microsoft Forms |
| `allow-scripts` | 允许执行 JavaScript | **默认启用**（几乎所有嵌入内容都需要） |
| `allow-forms` | 允许提交表单 | 默认启用 |
| `allow-popups` | 允许弹出窗口 | 默认启用 |
| `allow-popups-to-escape-sandbox` | 允许弹出窗口脱离沙箱 | 默认启用 |
| `allow-presentation` | 允许演示模式（全屏） | 默认启用 |
| `allow-downloads` | 允许下载 | 默认启用 |

**同源权限白名单**：
```typescript
// embeddable.ts:152-165
const ALLOW_SAME_ORIGIN = new Set([
  "youtube.com", "youtu.be",
  "vimeo.com", "player.vimeo.com",
  "drive.google.com", "figma.com",
  "twitter.com", "x.com",
  "*.simplepdf.eu", "stackblitz.com",
  "reddit.com", "forms.microsoft.com",
]);
```

---

### 6. 失败回退

| 失败场景 | 回退策略 | 代码位置 |
|---------|---------|---------|
| URL 不在白名单 | 显示错误提示"无法识别的链接格式" | `App.tsx:9139` |
| iframe 加载失败 | 显示占位标签 "Empty Web-Embed" | `App.tsx:1409-411` |
| 平台解析失败 | 返回原始 URL，尝试直接嵌入 | `embeddable.ts:388-399` |
| Vimeo URL 无效 | 设置 error 字段，渲染错误状态 | `embeddable.ts:228-230` |

---

## 第三部分：两条跨域消息链路分析

### 跨域通信总览

| 链路 | 通信方向 | 用途 | 安全级别 |
|------|---------|------|---------|
| **视频播放器控制** | 父页面 ↔ iframe (YouTube/Vimeo) | 播放/暂停同步 | ⭐⭐ 中等 |
| **Excalidraw Plus 导出** | 父窗口 ↔ iframe | 场景数据导出 | ⭐⭐⭐⭐⭐ 最高 |

---

### 链路 1：视频播放器控制消息

#### 来源校验

```typescript
// App.tsx:869-875
private onWindowMessage(event: MessageEvent) {
  // 严格限制消息来源
  if (
    event.origin !== "https://player.vimeo.com" &&
    event.origin !== "https://www.youtube.com"
  ) {
    return;  // 非白名单来源直接丢弃
  }

  // 验证消息格式
  let data = null;
  try {
    data = JSON.parse(event.data);
  } catch (e) {}
  if (!data) {
    return;
  }
}
```

#### 消息处理边界

| 平台 | 允许的消息类型 | 处理逻辑 | 响应边界 |
|------|-------------|---------|---------|
| **YouTube** | `infoDelivery` | 仅更新播放状态到本地 Map | **无响应**，仅更新内部状态 |
| **Vimeo** | `paused` | 查找对应 iframe 后回发控制命令 | 仅回发给匹配的 iframe |

```typescript
// App.tsx:885-929
switch (event.origin) {
  case "https://player.vimeo.com":
    if (data.method === "paused") {
      // 查找匹配的 iframe
      let source: Window | null = null;
      const iframes = document.body.querySelectorAll("iframe.excalidraw__embeddable");
      for (const iframe of iframes) {
        if (iframe.contentWindow === event.source) {
          source = iframe.contentWindow;
        }
      }
      // 仅回发给来源 iframe
      source?.postMessage(
        JSON.stringify({ method: data.value ? "play" : "pause", value: true }),
        "*",  // targetOrigin 宽松：来源已在校验阶段过滤
      );
    }
    break;
    
  case "https://www.youtube.com":
    if (data.event === "infoDelivery" && data.info?.playerState !== undefined) {
      // 仅更新本地状态，**不向外发送任何消息**
      YOUTUBE_VIDEO_STATES.set(data.id, data.info.playerState);
    }
    break;
}
```

---

### 链路 2：Excalidraw Plus 导出通信

#### 来源校验（四层安全防护）

```typescript
// ExcalidrawPlusIframeExport.tsx:157-217
const handleMessage = async (event: MessageEvent<MESSAGE_FROM_PLUS>) => {
  // 🔒 第一层：严格的 Origin 校验
  if (event.origin !== EXCALIDRAW_PLUS_ORIGIN) {
    throw new ExcalidrawError("Invalid origin");
  }

  if (event.data.type === EVENT_REQUEST_SCENE) {
    // 🔒 第二层：JWT 令牌存在性校验
    if (!event.data.jwt) {
      throw new ExcalidrawError("JWT is missing");
    }

    try {
      // 🔒 第三层：RSA-SHA256 签名验证
      await verifyJWT({
        token: event.data.jwt,
        publicKey: import.meta.env.VITE_APP_PLUS_EXPORT_PUBLIC_KEY,
      });
    } catch (error: any) {
      throw new ExcalidrawError("Failed to verify JWT");
    }

    // 🔒 第四层：数据解析（隐式校验数据结构）
    const parsedSceneData = await parseSceneData({
      rawAppStateString: localStorage.getItem(STORAGE_KEYS.LOCAL_STORAGE_APP_STATE),
      rawElementsString: localStorage.getItem(STORAGE_KEYS.LOCAL_STORAGE_ELEMENTS),
    });

    // 严格指定 targetOrigin 响应
    event.source!.postMessage(parsedSceneData, {
      targetOrigin: EXCALIDRAW_PLUS_ORIGIN,
    });
  }
};
```

#### JWT 验证流程

```typescript
// ExcalidrawPlusIframeExport.tsx:89-151
const verifyJWT = async ({ token, publicKey }) => {
  // 1. 解析 JWT 三段结构
  const [header, payload, signature] = token.split(".");
  if (!header || !payload || !signature) {
    throw new ExcalidrawError("Invalid JWT format");
  }

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

  // 4. 密码学签名验证
  const isValid = await crypto.subtle.verify(
    "RSASSA-PKCS1-v1_5",
    key,
    signatureArrayBuffer,
    new TextEncoder().encode(data),
  );

  // 5. 过期时间校验
  const parsedPayload = JSON.parse(decodedPayload);
  const currentTime = Math.floor(Date.now() / 1000);
  if (parsedPayload.exp && parsedPayload.exp < currentTime) {
    throw new Error("JWT has expired");
  }
};
```

#### 响应边界控制

| 消息类型 | 响应数据 | 目标限制 | 失败响应 |
|---------|---------|---------|---------|
| `REQUEST_SCENE` | `{ elements, appState, files }` | **仅** `EXCALIDRAW_PLUS_ORIGIN` | `{ type: "ERROR", message: "..." }` |
| `READY` | 无 - 仅通知就绪状态 | **仅** `EXCALIDRAW_PLUS_ORIGIN` | - |

---

## 第四部分：机制对比与总结

### 1. 白名单 vs 安全校验 区别

| 维度 | URL 白名单 | 安全校验 |
|------|-----------|---------|
| **作用阶段** | 资源入口阶段，拒绝非法请求 | 通信/操作阶段，验证请求合法性 |
| **匹配粒度** | 主机名 + 路径前缀 | Origin 精确匹配、密码学签名、过期时间 |
| **容错性** | 宽松匹配（支持子域名、通配符） | 严格精确匹配，一点不符即拒绝 |
| **扩展性** | 支持自定义验证函数/正则 | 固定算法，不可扩展 |
| **性能开销** | 低（字符串匹配/正则） | 高（加密运算、网络 IO） |
| **典型应用** | Library URL、iframe 嵌入源 | 跨域消息通信、API 授权 |

### 2. 限流机制总结

| 链路 | 限流方式 | 实现原理 |
|------|---------|---------|
| **Library 导入** | 任务队列串行化 | Promise 队列，等待前一个任务完成 |
| **Library 持久化** | 事务队列 | `Queue` 类，确保数据库操作原子性 |
| **iframe 嵌入** | 内存缓存 | URL Map 缓存，避免重复解析 |
| **Library 渲染** | SVG 缓存 | 按 Item ID 缓存导出结果 |

### 3. 失败策略总结

| 策略 | Library 链路 | iframe 链路 |
|------|-------------|------------|
| **静默失败** | 用户取消导入时保持原状态 | - |
| **降级渲染** | - | URL 解析失败时使用原始链接尝试嵌入 |
| **错误提示** | 校验/持久化失败显示 Toast | 不支持的 URL 显示错误消息 |
| **优雅回退** | 迁移失败回退到旧存储 | 加载失败显示占位标签 |

---

## 附录：核心文件索引

| 功能模块 | 文件路径 |
|---------|---------|
| Library 核心逻辑 | `packages/excalidraw/data/library.ts` |
| Embeddable 解析 | `packages/element/src/embeddable.ts` |
| iframe 渲染 | `packages/excalidraw/components/App.tsx:1832` |
| 跨域消息处理器 | `packages/excalidraw/components/App.tsx:869` |
| Plus 导出通信 | `excalidraw-app/ExcalidrawPlusIframeExport.tsx` |
| Library SVG 缓存 | `packages/excalidraw/hooks/useLibraryItemSvg.ts` |
