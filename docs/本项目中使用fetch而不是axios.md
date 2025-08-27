## 当前项目中的 fetch 实现

让我先分析项目当前的实现：

### 核心实现结构
```typescript
// 73:94:src/request.ts
const response: Response = await fetch(
  url,
  stripExtendRequestInitProperties(options)
)

if (!response.ok) {
  throw new HTTPError(response, url, options)
}

return await response.json()
} catch (error) {
if (options.retries > 0 && (error as HTTPError).status === 500) {
  await sleep(500)
  return request<T>(url, { ...options, retries: options.retries - 1 })
}

await (error as HTTPError).buildAdditionalDetails?.()

throw new SDKError(error as HTTPError)
}
```

## Fetch vs Axios 详细对比

### 1. **包体积对比**

| 方案             | 包体积 | 描述                         |
| ---------------- | ------ | ---------------------------- |
| **Fetch (当前)** | 0 KB   | 浏览器原生 API，无需额外依赖 |
| **Axios**        | ~47 KB | 需要额外的第三方库依赖       |

### 2. **浏览器兼容性**

| 方案      | 兼容性                 | 备注                           |
| --------- | ---------------------- | ------------------------------ |
| **Fetch** | 现代浏览器 (IE 不支持) | 需要 polyfill 支持老版本浏览器 |
| **Axios** | 广泛兼容               | 内置兼容性处理，支持 IE        |

### 3. **API 设计对比**

#### Fetch 实现 (当前项目)
```typescript
// 使用方式
const response = await request<LiFiStep>(`${apiUrl}/quote`, {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify(params),
  retries: 1
})
```

#### Axios 实现 (如果使用)
```typescript
// 使用方式
const response = await axios.post<LiFiStep>(`${apiUrl}/quote`, params, {
  headers: { 'Content-Type': 'application/json' },
  retry: 1
})
```

### 4. **功能特性对比**

| 功能               | Fetch (当前实现)               | Axios          |
| ------------------ | ------------------------------ | -------------- |
| **请求拦截器**     | ❌ 需手动实现                   | ✅ 内置支持     |
| **响应拦截器**     | ❌ 需手动实现                   | ✅ 内置支持     |
| **请求取消**       | ✅ AbortController              | ✅ CancelToken  |
| **自动 JSON 解析** | ❌ 需手动调用 `.json()`         | ✅ 自动处理     |
| **重试机制**       | ✅ 自定义实现                   | ❌ 需要插件     |
| **超时处理**       | ✅ AbortController + setTimeout | ✅ 内置 timeout |
| **错误处理**       | ✅ 自定义 HTTPError 类          | ✅ 内置错误处理 |

### 5. **错误处理对比**

#### 当前 Fetch 实现
```typescript
// 自定义错误分类
const statusCodeToErrorClassificationMap = new Map([
  [400, { type: ErrorName.ValidationError, code: LiFiErrorCode.ValidationError }],
  [404, { type: ErrorName.NotFoundError, code: LiFiErrorCode.NotFound }],
  [409, { type: ErrorName.SlippageError, code: LiFiErrorCode.SlippageError }],
  // ...
])

// 详细的错误信息构建
export class HTTPError extends BaseError {
  async buildAdditionalDetails() {
    try {
      this.responseBody = await this.response.json()
      this.message += `. ${this.responseBody?.message.toString()}`
    } catch {}
  }
}
```

#### Axios 错误处理
```typescript
try {
  const response = await axios.get('/api/data')
} catch (error) {
  if (error.response) {
    // 服务器响应错误
    console.log(error.response.status)
    console.log(error.response.data)
  } else if (error.request) {
    // 网络错误
    console.log(error.request)
  }
}
```

### 6. **性能对比**

| 方面         | Fetch           | Axios           |
| ------------ | --------------- | --------------- |
| **启动时间** | 更快 (原生 API) | 较慢 (需加载库) |
| **内存占用** | 更少            | 更多            |
| **网络性能** | 原生优化        | 额外抽象层      |
| **打包大小** | 无影响          | 增加 47KB       |

### 7. **代码维护性对比**

#### 当前 Fetch 的优势
```typescript
// 1. 完全控制请求流程
const response = await fetch(url, options)
if (!response.ok) {
  // 自定义错误处理逻辑
}

// 2. 类型安全的自定义扩展
interface ExtendedRequestInit extends RequestInit {
  retries?: number
}
```

#### Axios 的优势
```typescript
// 1. 简洁的 API
const data = await axios.get(url)

// 2. 丰富的生态系统
axios.interceptors.request.use(config => {
  // 全局请求拦截
  return config
})
```

## 具体使用场景分析

### 适合继续使用 Fetch 的场景
1. **SDK 类库** ✅ - 需要最小依赖
2. **性能敏感应用** ✅ - 减少bundle大小
3. **现代浏览器环境** ✅ - 不需要兼容老浏览器
4. **完全自定义需求** ✅ - 如项目中的重试和错误处理

### 适合使用 Axios 的场景
1. **快速开发** - 减少样板代码
2. **复杂的拦截器需求** - 认证、日志等
3. **团队熟悉度高** - 开发效率
4. **需要兼容老浏览器** - IE 支持

## 建议

基于当前项目的特点，我认为**继续使用 fetch 是正确的选择**，理由如下：

1. **SDK 特性**: 作为 SDK，最小化依赖非常重要
2. **自定义实现完善**: 项目已经实现了完整的错误处理、重试机制
3. **性能优势**: 更小的包体积，更好的性能
4. **现代化**: 使用原生 Web 标准，更符合未来趋势

如果要迁移到 Axios，主要是为了减少自定义代码的维护成本，但会失去当前实现的灵活性和性能优势。
