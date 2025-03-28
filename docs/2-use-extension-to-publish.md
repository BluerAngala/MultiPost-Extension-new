# 使用扩展程序发布内容
> 为了让我们的扩展更加强大，我们对外提供了相关接口，允许网页调用我们的扩展程序。
你开一再自己的网页中，集成 MultiPost 扩展，允许用户一键发布内容到多个平台上。


## 实现方式

我们通过 `window.postMessage` 来实现网页和扩展程序之间的通信。扩展首先会在每个网页中都注入 `src/contents/extension.ts` 中的脚本，然后通过 `window.postMessage` 来发送消息和接收消息并将消息发送给 `src/background/index.ts` 中的脚本进行处理。

## 接口

### 通用接口请求、响应

具体内容参考 `src/types/external.ts` 中的 `ExternalRequest` 和 `ExternalResponse` 类型。

```typescript
export type ExtensionExternalRequest<T> = {
  type: 'request';
  traceId: string;
  action: string;
  data: T;
};
```

一个基本的请求示例：

```typescript
window.postMessage(
  {
    type: 'request',
    traceId: '123', // traceId 用于标识一次请求，由调用方生成
    action: 'getToken',
    data: { name: 'test' },
  },
  '*',
);
```

### 未授权响应

如果网页调用扩展程序的接口时，扩展程序用户未授权过该网页的域名，则会返回未授权的响应。

```typescript
{
  type: 'response',
  traceId: request.traceId,
  action: request.action,
  code: 403,
  message: 'Untrusted origin',
  data: null,
}
```

### 确定扩展当前状态

```js
window.postMessage({
  type: 'request',
  traceId: '',
  action: 'MUTLIPOST_EXTENSION_CHECK_SERVICE_STATUS',
  data: {},
});
```

响应体：

```typescript
interface CheckServiceStatusResponse {
  extensionId: string; // 扩展程序的 ID
}
```

### 打开扩展设置页

```js
window.postMessage({
  type: 'request',
  traceId: '',
  action: 'MUTLIPOST_EXTENSION_OPEN_OPTIONS',
  data: {},
});
```

### 获取扩展访问权限

由于我们扩展的特殊性，我们允许网页调用我们的扩展程序，但是需要网页先获取扩展的访问权限。

```js
window.postMessage({
  type: 'request',
  traceId: '',
  action: 'MUTLIPOST_EXTENSION_REQUEST_TRUST_DOMAIN',
  data: {},
});
```

响应体：

```typescript
interface TrustDomainResponse {
  trusted: boolean; // 是否信任
  status: 'confirm' | 'cancel'; // 确认或取消
}
```

### 获取当前可用于发布的平台

```js
window.postMessage({
  type: 'request',
  traceId: '',
  action: 'MUTLIPOST_EXTENSION_PLATFORMS',
  data: {},
});
```

响应体：

```typescript
interface PlatformResponse {
  platforms: PlatformInfo[]; // 当前可用于发布的平台
}

interface PlatformInfo {
  type: 'DYNAMIC' | 'VIDEO';
  name: string;
  homeUrl: string;
  faviconUrl?: string;
  platformName: string;
  username?: string;
  userAvatarUrl?: string;
  injectUrl: string;
  injectFunction: (data: SyncData) => Promise<void>;
}
```

### 发布内容

有关 `SyncData` 的和其他的类型定义，请参考 `src/sync/common.ts` 中的 `SyncData` 类型。

```js
interface SyncData {
  platforms: string[];
  auto_publish: boolean;
  data: DynamicData | PostData | VideoData;
}

window.postMessage({
  type: 'request',
  traceId: '',
  action: 'MUTLIPOST_EXTENSION_PUBLISH',
  data: {
    platforms: ['DYNAMIC', 'VIDEO'],
    auto_publish: true,
    data: {
      title: 'test',
    },
  },
});
```

## 參考

```typescript
import { v4 as uuidv4 } from 'uuid';

export type ExtensionExternalRequest<T> = {
  type: 'request';
  traceId: string;
  action: string;
  data: T;
};

export interface ExtensionExternalResponse<T> {
  type: 'response';
  traceId: string;
  action: string;
  code: number;
  message: string;
  data: T;
}

export async function sendRequest<T>(action: string, data?: T, timeout: number = 5000): Promise<T> {
  const traceId = uuidv4();

  return new Promise<T>((resolve, reject) => {
    // Create message handler
    const messageHandler = (event: MessageEvent) => {
      if (event.data.type === 'response' && event.data.action === action && event.data.traceId === traceId) {
        cleanup();
        resolve(event.data.data);
      }
    };

    // Create timeout handler
    let timeoutId: NodeJS.Timeout | undefined;
    if (timeout > 0) {
      timeoutId = setTimeout(() => {
        cleanup();
        reject(new Error(`Request timeout after ${timeout}ms`));
      }, timeout);
    }

    // Cleanup function
    const cleanup = () => {
      window.removeEventListener('message', messageHandler);
      if (timeoutId) {
        clearTimeout(timeoutId);
      }
    };

    // Add event listener
    window.addEventListener('message', messageHandler);

    // Send the message
    window.postMessage(
      {
        type: 'request',
        traceId,
        action,
        data,
      },
      '*',
    );
  });
}

export async function checkServiceStatus(timeout: number = 5000): Promise<boolean> {
  try {
    // Send request and wait for actual response
    await sendRequest<void>('MUTLIPOST_EXTENSION_CHECK_SERVICE_STATUS', undefined, timeout);
    return true;
  } catch (error) {
    console.error('Service check failed:', error);
    return false;
  }
}
```