# 5. 前端架构 — 框架层

> 本层只回答一件事：**代码怎么组织**。不涉及任何视觉样式与具体交互细节——那分别属于第 6、7 节。
> 技术栈：React 18 + TypeScript 5 + Vite 5 + TanStack Query v5 + Zustand + React Router 6

---

## 5.1 分层模型

前端严格分为四层，依赖方向单向向下，**禁止反向引用与跨层跳跃**：

```
┌─────────────────────────────────────────────────────────┐
│  app/        应用装配层                                   │
│   · 路由表  · Provider 组合  · 错误边界  · 布局骨架        │
│   只做"接线"，不含业务逻辑                                 │
└────────────────────────┬────────────────────────────────┘
                         │ 只能向下依赖
┌────────────────────────▼────────────────────────────────┐
│  features/   业务功能层（按领域切分，垂直切片）             │
│   · auth/  upload/  audit/  playback/  reader/  download/ │
│   每个 feature 内部自含 components / hooks / api / model  │
│   feature 之间禁止直接 import，跨域协作走 shared/          │
└────────────────────────┬────────────────────────────────┘
                         │
┌────────────────────────▼────────────────────────────────┐
│  shared/     共享能力层                                   │
│   · api/     HTTP 客户端、错误映射、拦截器                 │
│   · hooks/   通用 hook（防抖、媒体查询、键盘导航…）         │
│   · lib/     纯函数工具（时间格式化、签名校验、脱敏）        │
│   · types/   全局类型定义                                  │
│   禁止 import features/ 或 app/                            │
└────────────────────────┬────────────────────────────────┘
                         │
┌────────────────────────▼────────────────────────────────┐
│  ui/         设计系统层（纯展示，零业务耦合）               │
│   · 原子组件：Button / Input / Modal / Table / Tag …      │
│   只接受 props，不发起请求、不读全局状态                   │
└─────────────────────────────────────────────────────────┘
```

### 5.1.1 为什么这样切

| 层 | 变更频率 | 变更原因 | 隔离收益 |
|---|---|---|---|
| `ui/` | 低 | 设计规范调整 | 改按钮样式不会波及业务逻辑 |
| `shared/` | 低 | 通用能力增强 | 一处修 HTTP 错误处理，全站生效 |
| `features/` | 高 | 业务规则演进 | 审核流程改动只影响 `features/audit/` |
| `app/` | 低 | 新增页面/路由 | 装配集中，路由权限一目了然 |

**关键约束的可执行落地**（用 ESLint 强制，而非靠人自觉）：

```javascript
// .eslintrc.cjs 片段 —— 用 no-restricted-imports 把架构约束变成编译期错误
module.exports = {
  rules: {
    'no-restricted-imports': ['error', {
      patterns: [
        // ui 层不得依赖任何上层
        { group: ['**/features/**', '**/shared/**', '**/app/**'],
          message: 'ui/ 必须是纯展示层，不得依赖业务代码。' },
        // shared 层不得反向依赖
        { group: ['**/features/**', '**/app/**'],
          message: 'shared/ 不得依赖 features/ 或 app/。' },
        // feature 之间禁止直接互相引用（走 shared 或 app 层编排）
        { group: ['@/features/*/!(index)', '@/features/*/*/**'],
          message: 'feature 之间禁止深层直接引用，请通过 index.ts 公开接口。' },
      ],
    }],
  },
};
```

> **架构约束与认证状态的例外处理**：`shared/api/httpClient.ts` 需要读取当前会话状态以注入令牌、并在 401 时清理会话。若严格禁止 `shared/` 引用 `features/`，此处会违反规则。
>
> 正确的解法不是放宽规则，而是**把认证状态的所有权下移到 `shared/`**：
>
> ```
> shared/auth/sessionStore.ts     ← 令牌与用户身份的持有者（归属 shared）
> features/auth/                  ← 登录表单、MFA 交互、注册流程（归属 features）
> ```
>
> `httpClient` 依赖 `shared/auth/sessionStore`，`features/auth` 的登录逻辑在成功后调用 `sessionStore.setSession(...)`。依赖方向变为 `features → shared`，单向且合法。会话状态本就是横切关注点（路由守卫、HTTP 客户端、WebSocket 都要用），放在 `shared/` 在架构上更正确。

---

## 5.2 目录结构

```
src/
├── app/
│   ├── main.tsx                     # 入口：挂载 Provider 树
│   ├── App.tsx                      # 根组件
│   ├── providers/
│   │   ├── QueryProvider.tsx        # TanStack Query 配置
│   │   ├── AuthProvider.tsx         # 认证上下文（含静默刷新）
│   │   ├── ThemeProvider.tsx        # 主题切换（见第 7 节）
│   │   └── I18nProvider.tsx
│   ├── router/
│   │   ├── index.tsx                # 路由表定义
│   │   ├── guards.tsx               # RequireAuth / RequirePermission
│   │   └── lazyRoutes.ts            # 路由级代码分割
│   └── layouts/
│       ├── RootLayout.tsx
│       ├── AdminLayout.tsx
│       └── ImmersiveLayout.tsx      # 沉浸式阅读/播放（无导航栏）
│
├── features/
│   ├── auth/
│   │   ├── api/authApi.ts
│   │   ├── hooks/useLogin.ts
│   │   ├── hooks/useMfaVerify.ts
│   │   ├── model/authStore.ts       # 该 feature 私有的**表单/流程**状态（不持有会话身份）
│   │   ├── components/LoginForm.tsx
│   │   ├── components/MfaChallenge.tsx
│   │   ├── types.ts
│   │   └── index.ts                 # 唯一对外出口
│   ├── upload/
│   │   ├── api/uploadApi.ts
│   │   ├── hooks/useUploadIntent.ts
│   │   ├── hooks/useDirectUpload.ts # 直传对象存储（XHR 进度）
│   │   ├── hooks/useUploadQueue.ts  # 批量上传队列（特权用户）
│   │   ├── model/uploadStore.ts
│   │   ├── components/UploadDropzone.tsx
│   │   ├── components/UploadQuotaBar.tsx
│   │   └── index.ts
│   ├── playback/
│   │   ├── api/playbackApi.ts
│   │   ├── hooks/usePlaybackSession.ts
│   │   ├── hooks/useQualitySelector.ts
│   │   ├── hooks/useProgressReporter.ts
│   │   ├── model/playerStore.ts
│   │   ├── components/VideoPlayer.tsx      # 仅装配，样式在 player.css
│   │   ├── components/AudioPlayer.tsx
│   │   ├── components/QualityMenu.tsx
│   │   └── index.ts
│   ├── danmaku/
│   │   ├── api/danmakuApi.ts
│   │   ├── hooks/useDanmakuStream.ts
│   │   ├── model/danmakuStore.ts
│   │   ├── components/DanmakuLayer.tsx
│   │   └── index.ts
│   ├── reader/
│   │   ├── api/readerApi.ts
│   │   ├── hooks/useReadingProgress.ts
│   │   ├── hooks/useReaderPreferences.ts
│   │   ├── model/readerStore.ts
│   │   ├── components/ReaderView.tsx
│   │   ├── components/TableOfContents.tsx
│   │   └── index.ts
│   ├── audit/
│   │   ├── api/auditApi.ts
│   │   ├── hooks/useAuditQueue.ts
│   │   ├── hooks/useClaimContent.ts
│   │   ├── hooks/useAuditDecision.ts
│   │   ├── model/auditStore.ts
│   │   ├── components/AuditQueue.tsx
│   │   ├── components/AuditDetail.tsx
│   │   ├── components/RiskBadge.tsx
│   │   ├── components/DecisionPanel.tsx
│   │   └── index.ts
│   ├── appeal/
│   ├── category/
│   ├── download/
│   └── profile/
│
├── shared/
│   ├── api/
│   │   ├── httpClient.ts            # fetch 封装 + 拦截器（依赖 shared/auth）
│   │   ├── errorMap.ts              # 错误码 → 前端语义
│   │   └── types.ts
│   ├── auth/
│   │   └── sessionStore.ts          # 会话身份与 Access Token 的持有者
│   │                                # 横切关注点，故归属 shared 而非 features
│   ├── hooks/
│   │   ├── useDebounce.ts
│   │   ├── useMediaQuery.ts
│   │   ├── useKeyboardShortcut.ts
│   │   ├── useIntersection.ts
│   │   └── useCursorPagination.ts
│   ├── lib/
│   │   ├── format.ts                # 时长、字节、相对时间
│   │   ├── sanitize.ts              # 富文本净化（DOMPurify 包装）
│   │   ├── hash.ts                  # 客户端 SHA-256（Web Crypto）
│   │   └── url.ts                   # 签名 URL 过期检测
│   └── types/
│       ├── api.d.ts                 # 由 OpenAPI 生成
│       └── domain.ts
│
├── ui/
│   ├── Button/  Input/  Select/  Modal/  Drawer/
│   ├── Table/   Tag/    Badge/   Tooltip/ Toast/
│   ├── Skeleton/ Spinner/ EmptyState/ ErrorState/
│   ├── primitives/                  # 无样式基元（供 ui 组合）
│   └── index.ts
│
└── styles/                          # 见第 7 节，此处仅列出存在性
    ├── tokens/  themes/  base/  utilities/
```

### 5.2.1 `index.ts` 作为 feature 边界

每个 feature 只通过 `index.ts` 暴露**最小公共接口**：

```typescript
// features/audit/index.ts
export { AuditQueue } from './components/AuditQueue';
export { useAuditQueue } from './hooks/useAuditQueue';
export type { AuditQueueItem } from './types';
// 内部实现（store、api、子组件）不导出 → 外部无法依赖，可自由重构
```

**收益**：`features/audit/` 内部重命名文件或重构 store，只要 `index.ts` 的签名不变，其他代码零改动。

---

## 5.3 路由与权限装配

### 5.3.1 路由表

```typescript
// app/router/index.tsx
import { lazy } from 'react';
import { createBrowserRouter } from 'react-router-dom';

// 路由级懒加载：审核后台不会进入普通用户的初始包
const HomePage        = lazy(() => import('@/features/category/pages/HomePage'));
const ContentPage     = lazy(() => import('@/features/playback/pages/ContentPage'));
const ReaderPage      = lazy(() => import('@/features/reader/pages/ReaderPage'));
const AuditPage       = lazy(() => import('@/features/audit/pages/AuditPage'));
const AppealPage      = lazy(() => import('@/features/appeal/pages/AppealPage'));
const AdminUsersPage  = lazy(() => import('@/features/profile/pages/AdminUsersPage'));

export const router = createBrowserRouter([
  {
    path: '/',
    element: <RootLayout />,
    errorElement: <RouteErrorBoundary />,
    children: [
      { index: true, element: <HomePage /> },
      { path: 'c/:categorySlug', element: <CategoryPage /> },
      { path: 'v/:contentId', element: <ContentPage /> },
      { path: 'a/:contentId', element: <ContentPage /> },   // 音频，复用页面按类型渲染
      { path: 'read/:contentId', element: <ReaderPage /> },

      // 需登录
      {
        element: <RequireAuth />,
        children: [
          { path: 'upload', element: <UploadPage /> },
          { path: 'me/uploads', element: <MyUploadsPage /> },
          { path: 'appeals/new', element: <AppealPage /> },
        ],
      },

      // 需权限 + 强制 MFA
      {
        element: <RequirePermission permission="content:audit" requireMfa />,
        children: [
          { path: 'admin/audit', element: <AuditPage /> },
          { path: 'admin/audit/:contentId', element: <AuditDetailPage /> },
          { path: 'admin/appeals', element: <AppealsQueuePage /> },
        ],
      },
      {
        element: <RequirePermission permission="user:read_pii" requireMfa />,
        children: [{ path: 'admin/users', element: <AdminUsersPage /> }],
      },
    ],
  },

  // 沉浸式布局：独立于 RootLayout（无导航栏、无侧边栏）
  {
    path: '/immersive',
    element: <ImmersiveLayout />,
    children: [{ path: 'read/:contentId', element: <ImmersiveReaderPage /> }],
  },
]);
```

### 5.3.2 路由守卫

**核心原则**：前端守卫只做**体验优化**（避免用户看到无权页面后闪跳），**真正的权限判定在服务端**。前端守卫可被绕过，因此不承担安全职责。

```typescript
// app/router/guards.tsx
export function RequireAuth() {
  const { isAuthenticated, isInitializing } = useAuth();
  const location = useLocation();

  // 初始化期间不做判定，避免刷新页面时误跳登录页
  if (isInitializing) return <FullPageSpinner />;

  if (!isAuthenticated) {
    // 记录来源，登录后跳回原页面
    return <Navigate to="/login" state={{ from: location }} replace />;
  }
  return <Outlet />;
}

export function RequirePermission({
  permission,
  requireMfa = false,
}: { permission: string; requireMfa?: boolean }) {
  const { user, isInitializing } = useAuth();

  if (isInitializing) return <FullPageSpinner />;
  if (!user) return <Navigate to="/login" replace />;

  // 权限快照来自 Access Token 的 perms 声明
  if (!user.permissions.includes(permission)) {
    return <ForbiddenState />;   // 403 页面，不跳转（避免用户困惑）
  }

  // 特权页面强制要求本会话已完成 MFA
  if (requireMfa && !user.sessionMfaVerified) {
    return <MfaGate />;          // 引导完成第二因子
  }

  return <Outlet />;
}
```

### 5.3.3 代码分割策略

| 分包 | 内容 | 触发时机 |
|---|---|---|
| `vendor-react` | react, react-dom, react-router | 首屏必需 |
| `vendor-query` | TanStack Query | 首屏必需 |
| `main` | app + ui + 首页 feature | 首屏 |
| `player` | video.js / hls.js / dash.js | 首次进入播放页 |
| `reader` | 富文本渲染器 | 首次进入阅读页 |
| `admin` | 全部 `features/audit` + `features/appeal` | 管理员首次进入后台 |
| `editor` | 上传/编辑相关 | 首次上传 |

**约束**：普通用户的初始包**不得包含** `admin` chunk。审核工作台体积不小（表格、播放器预览、批量操作），不应拖累全站首屏。

---

## 5.4 状态管理：三分类原则

**这是本项目最容易失控的地方**。按数据性质分三类，各用各的工具，**不允许混用**：

| 类型 | 例子 | 工具 | 理由 |
|---|---|---|---|
| **服务端状态** | 内容列表、审核队列、用户资料、分类树 | **TanStack Query** | 有缓存、失效、重试、竞态、后台刷新等需求，手写必然出错 |
| **全局客户端状态** | 主题、播放器全局设置、上传队列、阅读偏好 | **Zustand**（切片化） | 无服务端真相源，需跨组件共享 |
| **局部 UI 状态** | 弹窗开关、下拉展开、表单输入 | **useState / useReducer** | 只影响一个组件，上升为全局是过度设计 |

### 5.4.1 服务端状态：TanStack Query

```typescript
// features/audit/hooks/useAuditQueue.ts
export function useAuditQueue(filters: AuditQueueFilters) {
  return useInfiniteQuery({
    queryKey: ['audit', 'queue', filters],
    queryFn: ({ pageParam }) => auditApi.getQueue({ ...filters, cursor: pageParam }),
    initialPageParam: undefined as string | undefined,
    getNextPageParam: (last) =>
      last.meta.pagination.hasMore ? last.meta.pagination.nextCursor : undefined,

    // 审核队列需要较高新鲜度，但不必每次挂载都请求
    staleTime: 30_000,

    // 工作台页面：切回标签页时刷新，让审核员看到最新队列
    refetchOnWindowFocus: true,

    // 队列轮询：每 60s 拉取新任务
    refetchInterval: 60_000,
  });
}
```

**查询键的规范化**是缓存正确性的基础。定义统一的 key factory：

```typescript
// features/audit/api/queryKeys.ts
export const auditKeys = {
  all: ['audit'] as const,
  queues: () => [...auditKeys.all, 'queue'] as const,
  queue: (filters: AuditQueueFilters) => [...auditKeys.queues(), filters] as const,
  detail: (contentId: string) => [...auditKeys.all, 'detail', contentId] as const,
  metrics: (period: Period) => [...auditKeys.all, 'metrics', period] as const,
};

// 失效时用前缀，一次失效整个域
queryClient.invalidateQueries({ queryKey: auditKeys.queues() });
```

**乐观更新用于高频、低风险的交互**（如点赞、弹幕发送）：

```typescript
// features/danmaku/hooks/useSendDanmaku.ts
export function useSendDanmaku(contentId: string) {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: (input: DanmakuInput) => danmakuApi.send(contentId, input),

    // 乐观插入：弹幕必须"立刻"出现在屏幕上，等网络往返就晚了
    onMutate: async (input) => {
      await queryClient.cancelQueries({ queryKey: danmakuKeys.range(contentId, input.videoTimeMs) });
      const previous = queryClient.getQueryData(danmakuKeys.range(contentId, input.videoTimeMs));

      queryClient.setQueryData(danmakuKeys.range(contentId, input.videoTimeMs), (old) =>
        old ? [...old, { ...input, id: `optimistic-${crypto.randomUUID()}`, pending: true }] : old
      );

      return { previous };   // 回滚快照
    },

    onError: (_err, input, context) => {
      // 失败（如命中敏感词被拒）→ 回滚并提示
      queryClient.setQueryData(danmakuKeys.range(contentId, input.videoTimeMs), context?.previous);
      toast.error('弹幕发送失败，可能包含违规内容');
    },

    onSettled: (_data, _err, input) => {
      queryClient.invalidateQueries({
        queryKey: danmakuKeys.range(contentId, input.videoTimeMs),
      });
    },
  });
}
```

**审核决策不用乐观更新**——这是高风险操作，必须等服务端确认状态流转成功后再更新 UI，避免界面上显示"已通过"但服务端实际失败造成误判。

### 5.4.2 全局客户端状态：Zustand 切片

```typescript
// features/playback/model/playerStore.ts
interface PlayerState {
  volume: number;
  muted: boolean;
  playbackRate: number;
  preferredQuality: 'auto' | '1080p' | '720p' | '480p';
  subtitlesEnabled: boolean;
  danmakuEnabled: boolean;
  danmakuOpacity: number;

  setVolume: (v: number) => void;
  setPlaybackRate: (r: number) => void;
  setQuality: (q: PlayerState['preferredQuality']) => void;
  toggleDanmaku: () => void;
}

export const usePlayerStore = create<PlayerState>()(
  persist(
    (set) => ({
      volume: 0.8,
      muted: false,
      playbackRate: 1,
      preferredQuality: 'auto',
      subtitlesEnabled: false,
      danmakuEnabled: true,
      danmakuOpacity: 0.9,

      setVolume: (volume) => set({ volume, muted: volume === 0 }),
      setPlaybackRate: (playbackRate) => set({ playbackRate }),
      setQuality: (preferredQuality) => set({ preferredQuality }),
      toggleDanmaku: () => set((s) => ({ danmakuEnabled: !s.danmakuEnabled })),
    }),
    {
      name: 'player-preferences',
      // 只持久化偏好，不持久化瞬时状态
      partialize: (s) => ({
        volume: s.volume, muted: s.muted, playbackRate: s.playbackRate,
        preferredQuality: s.preferredQuality, subtitlesEnabled: s.subtitlesEnabled,
        danmakuEnabled: s.danmakuEnabled, danmakuOpacity: s.danmakuOpacity,
      }),
    }
  )
);
```

**为什么播放器偏好用 Zustand 而非 Context**：Context 值变化会导致所有消费者重渲染。播放进度每 250ms 更新一次，若放 Context，整个页面树会高频重渲染。Zustand 的**选择器订阅**只重渲染真正用到该字段的组件：

```typescript
// 只订阅 volume，进度更新不会触发此组件重渲染
const volume = usePlayerStore((s) => s.volume);
```

### 5.4.3 状态归属决策表

| 问题 | 归属 |
|---|---|
| 数据来自服务端？ | TanStack Query |
| 刷新页面后需要保留？ | Zustand + persist |
| 多个不相邻组件需要读？ | Zustand |
| 只有一个组件或父子链使用？ | useState / useReducer |
| 用于表单？ | React Hook Form 内部状态（不进全局） |
| 是派生值？ | 计算得出，**不存** |

> **反模式警告**：不要因为"方便"就把服务端数据从 Query 复制到 Zustand。两份真相源必然不一致，且需要手工同步失效——这是前端状态 bug 的头号来源。

---

## 5.5 数据层：HTTP 客户端

### 5.5.1 客户端封装

```typescript
// shared/api/httpClient.ts
// ✅ 会话状态归属 shared/auth/，因此 shared → shared 的依赖是合法的。
//    绝不能改为 import from '@/features/auth/...'——那会违反 5.1.1 的
//    分层约束（shared 不得依赖 features），ESLint 会直接报错。
import { sessionStore } from '@/shared/auth/sessionStore';

const BASE_URL = import.meta.env.VITE_API_BASE_URL;   // 构建时注入，非硬编码

class ApiError extends Error {
  constructor(
    public status: number,
    public code: string,
    public detail?: unknown,
    public requestId?: string
  ) {
    super(code);
  }
}

// Access Token 只存在内存，不放 localStorage（XSS 无法窃取）
// 实际实现位于 shared/auth/sessionStore.ts，此处仅示意其接口。
// 该 store 同时持有用户身份快照，供路由守卫与 httpClient 共用。
// 定义（shared/auth/sessionStore.ts）：
//   let accessToken: string | null = null;
//   let currentUser: SessionUser | null = null;
//   export const sessionStore = {
//     getToken:  () => accessToken,
//     setSession: (t: string, u: SessionUser) => { accessToken = t; currentUser = u; },
//     getUser:   () => currentUser,
//     clearSession: () => { accessToken = null; currentUser = null; },
//     subscribe: (fn) => { /* 供 React 组件用 useSyncExternalStore 订阅 */ },
//   };

// 静默刷新：并发请求共享同一个刷新 Promise，避免刷新风暴
let refreshPromise: Promise<string | null> | null = null;

async function refreshAccessToken(): Promise<string | null> {
  if (refreshPromise) return refreshPromise;   // 复用进行中的刷新

  refreshPromise = (async () => {
    try {
      const res = await fetch(`${BASE_URL}/auth/refresh`, {
        method: 'POST',
        credentials: 'include',   // 携带 HttpOnly Cookie
      });
      if (!res.ok) return null;
      const body = await res.json();
      sessionStore.setSession(body.data.accessToken, body.data.user);
      return body.data.accessToken;
    } catch {
      return null;
    } finally {
      refreshPromise = null;
    }
  })();

  return refreshPromise;
}

export async function request<T>(
  path: string,
  options: RequestInit & { skipAuth?: boolean; _retried?: boolean } = {}
): Promise<T> {
  const { skipAuth, _retried, ...init } = options;

  const headers = new Headers(init.headers);
  headers.set('Content-Type', 'application/json');

  const token = sessionStore.getToken();
  if (token && !skipAuth) headers.set('Authorization', `Bearer ${token}`);

  const res = await fetch(`${BASE_URL}${path}`, { ...init, headers });

  // 401 → 尝试静默刷新一次，成功则重放原请求
  if (res.status === 401 && !skipAuth && !_retried) {
    const newToken = await refreshAccessToken();
    if (newToken) {
      return request<T>(path, { ...options, _retried: true });
    }
    // 刷新失败 → 清理会话；订阅了 sessionStore 的路由守卫会自动跳登录
    sessionStore.clearSession();
    throw new ApiError(401, 'SESSION_EXPIRED');
  }

  if (res.status === 204) return undefined as T;

  const body = await res.json().catch(() => null);

  if (!res.ok) {
    const err = body?.error ?? {};
    throw new ApiError(res.status, err.code ?? 'UNKNOWN_ERROR', err.detail, err.requestId);
  }

  return body as T;
}

export const api = {
  get:  <T>(p: string, o?: RequestInit) => request<T>(p, { ...o, method: 'GET' }),
  post: <T>(p: string, data?: unknown, o?: RequestInit) =>
          request<T>(p, { ...o, method: 'POST', body: data ? JSON.stringify(data) : undefined }),
  patch:<T>(p: string, data?: unknown) =>
          request<T>(p, { method: 'PATCH', body: data ? JSON.stringify(data) : undefined }),
  del:  <T>(p: string) => request<T>(p, { method: 'DELETE' }),
};
```

### 5.5.2 错误码映射

服务端返回英文 `code`，前端集中映射为本地化文案。**不在业务组件里散落 if-else 判断错误码**：

```typescript
// shared/api/errorMap.ts
type ErrorPresentation = {
  messageKey: string;                    // i18n key
  action?: 'RETRY' | 'LOGIN' | 'UPGRADE' | 'WAIT' | 'APPEAL' | 'NONE';
  severity: 'error' | 'warning' | 'info';
};

export const ERROR_MAP: Record<string, ErrorPresentation> = {
  INVALID_CREDENTIALS:          { messageKey: 'error.invalidCredentials', severity: 'error' },
  TOO_MANY_REQUESTS:            { messageKey: 'error.rateLimited', action: 'WAIT', severity: 'warning' },
  DAILY_COUNT_QUOTA_EXCEEDED:   { messageKey: 'error.quotaCount', action: 'WAIT', severity: 'warning' },
  DAILY_BYTES_QUOTA_EXCEEDED:   { messageKey: 'error.quotaBytes', action: 'UPGRADE', severity: 'warning' },
  FILE_TOO_LARGE:               { messageKey: 'error.fileTooLarge', severity: 'error' },
  FORMAT_MISMATCH:              { messageKey: 'error.formatMismatch', severity: 'error' },
  CONTENT_BLOCKED:              { messageKey: 'error.contentBlocked', action: 'APPEAL', severity: 'warning' },
  ALREADY_CLAIMED_OR_RESOLVED:  { messageKey: 'error.alreadyClaimed', action: 'RETRY', severity: 'warning' },
  CONFLICT_OF_INTEREST:         { messageKey: 'error.conflictOfInterest', severity: 'error' },
  SESSION_EXPIRED:              { messageKey: 'error.sessionExpired', action: 'LOGIN', severity: 'info' },
  TOO_MANY_CONCURRENT_STREAMS:  { messageKey: 'error.tooManyStreams', severity: 'warning' },
};

export function presentError(err: unknown): ErrorPresentation {
  if (err instanceof ApiError && ERROR_MAP[err.code]) return ERROR_MAP[err.code];
  return { messageKey: 'error.unexpected', action: 'RETRY', severity: 'error' };
}
```

---

## 5.6 类型安全：从后端的单一真相源

### 5.6.1 OpenAPI 生成类型

后端 NestJS 用 `@nestjs/swagger` 装饰器自动生成 OpenAPI Schema，前端 CI 拉取并生成 TS 类型：

```bash
# package.json scripts
"api:sync":   "openapi-typescript https://api.example.com/openapi.json -o src/shared/types/api.d.ts",
"api:check":  "openapi-typescript ./openapi.snapshot.json -o /tmp/api.d.ts && diff /tmp/api.d.ts src/shared/types/api.d.ts"
```

CI 中运行 `api:check`，若后端接口变更而前端类型未同步，**构建失败**——把接口不兼容扼杀在合并前。

```typescript
// 使用生成的类型
import type { components } from '@/shared/types/api';

type ContentDetail = components['schemas']['ContentDetailResponse'];
type AuditQueueItem = components['schemas']['AuditQueueItem'];
```

### 5.6.2 运行时校验（边界处）

类型在编译期消失，服务端返回的 JSON 仍需运行时校验。**只在数据进入应用时校验一次**（API 层），之后全链路信任类型：

```typescript
// shared/api/validate.ts
import { z } from 'zod';

const PaginationSchema = z.object({
  nextCursor: z.string().nullable(),
  hasMore: z.boolean(),
  limit: z.number().int().positive(),
});

const EnvelopeSchema = <T extends z.ZodTypeAny>(dataSchema: T) =>
  z.object({
    success: z.literal(true),
    data: dataSchema,
    meta: z.object({
      requestId: z.string(),
      pagination: PaginationSchema.optional(),
    }).optional(),
  });

// 用法：解析失败抛出明确错误，而不是让 undefined 在 UI 深处爆炸
const parsed = EnvelopeSchema(z.array(AuditQueueItemSchema)).parse(rawResponse);
```

> **为什么不在每个组件里校验**：重复开销且代码噪音大。API 层是唯一的数据入口，在此把守即可。

---

## 5.7 构建与工程配置

### 5.7.1 Vite 配置要点

```typescript
// vite.config.ts
export default defineConfig({
  plugins: [react(), tsconfigPaths()],

  build: {
    target: 'es2022',
    sourcemap: false,          // 生产不发布 sourcemap（防止源码泄露）
    rollupOptions: {
      output: {
        manualChunks: {
          'vendor-react': ['react', 'react-dom', 'react-router-dom'],
          'vendor-query': ['@tanstack/react-query'],
          // 播放器与阅读器独立分包，按需加载
          'player':       ['video.js', 'hls.js'],
          'reader':       ['@tiptap/react', 'dompurify'],
        },
      },
    },
    chunkSizeWarningLimit: 300,  // 超过 300KB 警告，及时拆分
  },

  server: {
    proxy: {
      '/api': {
        target: process.env.VITE_DEV_API_TARGET,   // 环境变量，非硬编码
        changeOrigin: true,
        secure: true,
      },
    },
  },
});
```

### 5.7.2 环境变量约定

| 变量 | 用途 | 是否敏感 |
|---|---|---|
| `VITE_API_BASE_URL` | API 基址 | 否 |
| `VITE_CDN_BASE_URL` | 静态资源 CDN | 否 |
| `VITE_WS_URL` | WebSocket 地址（审核队列推送、弹幕） | 否 |
| `VITE_SENTRY_DSN` | 错误上报 | 公开 DSN，非密钥 |
| `VITE_CAPTCHA_SITE_KEY` | 验证码站点公钥 | 公开（配对的 secret 只在后端） |

> **安全红线**：`VITE_` 前缀的变量会被打包进客户端产物，**任何密钥都不得放入**。后端 API 密钥、KMS 凭据、验证码 secret 一律留在服务端。

### 5.7.3 质量门禁

```json
{
  "scripts": {
    "typecheck":  "tsc --noEmit",
    "lint":       "eslint . --max-warnings 0",
    "test":       "vitest run --coverage",
    "test:e2e":   "playwright test",
    "api:check":  "openapi-typescript ./openapi.snapshot.json -o /tmp/api.d.ts && diff /tmp/api.d.ts src/shared/types/api.d.ts",
    "verify":     "npm run typecheck && npm run lint && npm run api:check && npm run test"
  }
}
```

**CI 必须阻塞的条件**：
1. TypeScript 编译错误
2. ESLint 错误或警告（架构约束在此生效）
3. `api:check` 检测到接口类型不同步
4. 单测覆盖率低于 70%（`features/audit` 与 `features/auth` 要求 85%——这两个模块的正确性直接影响内容安全与账户安全）
5. 构建产物中 `admin` chunk 被 `main` 引用（说明分包失效，普通用户会加载后台代码）
6. `npm audit` 发现 high/critical 漏洞

---

## 5.8 可观测性接入

```typescript
// app/providers/QueryProvider.tsx
const queryClient = new QueryClient({
  defaultOptions: {
    queries: {
      staleTime: 60_000,
      retry: (failureCount, error) => {
        // 4xx 不重试（参数/权限错误重试无意义，只会加剧负担）
        if (error instanceof ApiError && error.status < 500) return false;
        return failureCount < 2;
      },
      retryDelay: (attempt) => Math.min(1000 * 2 ** attempt, 10_000),
    },
    mutations: { retry: false },   // 变更操作绝不自动重试（防重复提交）
  },
});

// 全局查询错误上报，附带 requestId 便于与后端日志关联
queryClient.getQueryCache().subscribe((event) => {
  if (event.type === 'updated' && event.query.state.status === 'error') {
    const err = event.query.state.error;
    if (err instanceof ApiError) {
      telemetry.captureError(err, {
        requestId: err.requestId,          // 后端日志的关联键
        queryKey: JSON.stringify(event.query.queryKey),
      });
    }
  }
});
```

**性能埋点**：关键用户旅程（登录 → 找到内容 → 播放成功、上传 → 审核通过）打点，测端到端时延。播放场景额外采集首帧时间、卡顿次数、码率切换次数。

---

## 5.9 层间契约速查

| 从 | 到 | 允许 | 方式 |
|---|---|---|---|
| app | features | ✅ | import feature 的 `index.ts` |
| app | ui, shared | ✅ | 直接引用 |
| features | shared | ✅ | 直接引用 |
| features | ui | ✅ | 直接引用 |
| features | features | ❌ | 禁止；跨域逻辑上移到 app 层编排 |
| shared | features, app | ❌ | ESLint 拦截 |
| ui | 任何业务层 | ❌ | ESLint 拦截；ui 必须可独立发布 |
| 任意层 | 样式细节 | — | 见第 7 节，组件不写内联魔法值 |
| 任意层 | 交互行为 | — | 见第 6 节，交互逻辑集中在 hooks |
