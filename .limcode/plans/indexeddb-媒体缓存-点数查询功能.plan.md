
# IndexedDB 媒体缓存 + 点数查询功能

## 背景

1. **媒体缓存**：当前预览窗口每次打开都要从远程 URL 重新加载图片/视频，且即梦的 CDN URL 有时效性，过期后无法再次预览。需要在任务完成时将媒体文件缓存到浏览器 IndexedDB 中，后续预览优先使用本地缓存。
2. **点数查询**：后端已有 `POST /token/points` 接口可查询积分余额，前端需要新增一个按钮实时展示积分信息。

## 一、IndexedDB 媒体缓存

### 1.1 数据库设计

在 `index.html` 的 `<script>` 部分开头新增一个 IndexedDB 工具模块：

```javascript
// ===== IndexedDB Media Cache =====
const MediaCache = (() => {
  const DB_NAME = 'jm_media_cache';
  const DB_VERSION = 1;
  const STORE_NAME = 'blobs';

  function openDB() {
    return new Promise((resolve, reject) => {
      const req = indexedDB.open(DB_NAME, DB_VERSION);
      req.onupgradeneeded = (e) => {
        const db = e.target.result;
        if (!db.objectStoreNames.contains(STORE_NAME)) {
          db.createObjectStore(STORE_NAME, { keyPath: 'url' });
        }
      };
      req.onsuccess = () => resolve(req.result);
      req.onerror = () => reject(req.error);
    });
  }

  return {
    // 保存 blob：key = 原始 URL
    async put(url, blob, meta = {}) {
      const db = await openDB();
      return new Promise((resolve, reject) => {
        const tx = db.transaction(STORE_NAME, 'readwrite');
        tx.objectStore(STORE_NAME).put({ url, blob, ...meta, cachedAt: Date.now() });
        tx.oncomplete = () => resolve();
        tx.onerror = () => reject(tx.error);
      });
    },

    // 获取 blob
    async get(url) {
      const db = await openDB();
      return new Promise((resolve, reject) => {
        const tx = db.transaction(STORE_NAME, 'readonly');
        const req = tx.objectStore(STORE_NAME).get(url);
        req.onsuccess = () => resolve(req.result || null);
        req.onerror = () => reject(req.error);
      });
    },

    // 创建 Object URL（用于 <img>/<video> src）
    async getObjectURL(url) {
      const entry = await this.get(url);
      if (entry && entry.blob) {
        return URL.createObjectURL(entry.blob);
      }
      return null;
    },

    // 从远程 URL 下载并缓存
    async cacheFromURL(url) {
      try {
        const existing = await this.get(url);
        if (existing) return; // 已缓存，跳过
        const resp = await fetch(url);
        if (!resp.ok) return;
        const blob = await resp.blob();
        await this.put(url, blob, { type: blob.type, size: blob.size });
      } catch (e) {
        console.warn('MediaCache: failed to cache', url, e);
      }
    }
  };
})();
```

### 1.2 自动缓存触发时机

在 `fetchTasks()` 的合并逻辑中，当任务状态变为 `completed` 时，自动触发后台缓存：

```javascript
// 在 serverTasks.forEach 或合并后：
serverTasks.forEach(st => {
  const oldTask = taskMap.get(st.task_id);
  const wasNotComplete = !oldTask || oldTask.status !== 'completed';
  taskMap.set(st.task_id, st);

  // 状态刚变为 completed → 触发缓存
  if (wasNotComplete && st.status === 'completed' && st.result?.data) {
    st.result.data.forEach(item => {
      if (item.url) MediaCache.cacheFromURL(item.url);
    });
  }
});
```

### 1.3 预览时优先使用缓存

修改 `viewResult()` 方法，在打开预览时先查 IndexedDB：

```javascript
const viewResult = async (task) => {
  // ... 原有 URL 提取逻辑 ...
  const urls = task.result.data.map(item => item.url).filter(Boolean);

  // 尝试从 IndexedDB 获取本地 blob URL
  const resolvedUrls = await Promise.all(urls.map(async (url) => {
    const localUrl = await MediaCache.getObjectURL(url);
    return localUrl || url; // 有缓存用缓存，没有就用原始 URL
  }));

  resultModal.urls = resolvedUrls;
  resultModal._originalUrls = urls; // 保留原始 URL 用于 Copy URL / Open Original
  // ... 其余逻辑 ...
};
```

同时修改 `copyUrl` 和 `openOriginal` 使用 `_originalUrls` 而非 `urls`。

### 1.4 翻页时释放 Object URL

在 `watch(resultModal.currentIndex)` 和关闭 modal 时，调用 `URL.revokeObjectURL()` 释放内存。

```javascript
watch(() => resultModal.show, (val) => {
  if (!val) {
    // modal 关闭时释放所有 blob URLs
    resultModal.urls.forEach(u => {
      if (u.startsWith('blob:')) URL.revokeObjectURL(u);
    });
  }
});
```

## 二、点数查询功能

### 2.1 后端接口确认

已有接口：`POST /token/points`，需要 `Authorization` header，返回格式：
```json
[{
  "token": "xxx",
  "points": {
    "giftCredit": 100,
    "purchaseCredit": 0,
    "vipCredit": 0,
    "totalCredit": 100
  }
}]
```

前端直接调用即可，**无需修改后端**。

### 2.2 前端 UI

在 Configuration.exe 窗口的 window-body 底部新增一行：

```html
<div class="field-row" style="justify-content: space-between;">
  <button @click="queryCredits" :disabled="!isConfigValid || creditLoading">
    {{ creditLoading ? 'Querying...' : '💰 Query Credits' }}
  </button>
  <span v-if="creditInfo" style="font-size: 12px;">
    Total: <b>{{ creditInfo.totalCredit }}</b>
    (🎁{{ creditInfo.giftCredit }} 💳{{ creditInfo.purchaseCredit }} 👑{{ creditInfo.vipCredit }})
  </span>
</div>
```

### 2.3 前端逻辑

```javascript
const creditInfo = ref(null);
const creditLoading = ref(false);

const queryCredits = async () => {
  creditLoading.value = true;
  try {
    const api = getApi();
    const res = await api.post('/token/points');
    // 接口返回数组（支持多 token），取汇总
    if (Array.isArray(res.data) && res.data.length > 0) {
      if (res.data.length === 1) {
        creditInfo.value = res.data[0].points;
      } else {
        // 多 token：汇总
        creditInfo.value = res.data.reduce((acc, item) => ({
          giftCredit: acc.giftCredit + item.points.giftCredit,
          purchaseCredit: acc.purchaseCredit + item.points.purchaseCredit,
          vipCredit: acc.vipCredit + item.points.vipCredit,
          totalCredit: acc.totalCredit + item.points.totalCredit,
        }), { giftCredit: 0, purchaseCredit: 0, vipCredit: 0, totalCredit: 0 });
      }
    }
  } catch (err) {
    showError('Query credits failed: ' + (err.response?.data?.message || err.message));
  } finally {
    creditLoading.value = false;
  }
};
```

### 2.4 自动更新积分

在提交任务成功后自动刷新积分：

```javascript
// submitImageTask 和 submitVideoTask 的成功分支末尾加一行
queryCredits();
```

## 三、文件改动清单

| 文件 | 改动 |
|------|------|
| `index.html` | 新增 IndexedDB 工具模块、修改 `viewResult`/`fetchTasks` 逻辑、新增积分查询 UI 和方法 |

**无需改动后端，无需 `npm run build`，只需部署 `index.html` 即可。**

## 四、部署

```bash
scp index.html moeblack@10.0.0.63:~/jimeng-api/index.html
```

刷新浏览器即生效。

## TODO LIST

<!-- LIMCODE_TODO_LIST_START -->
- [ ] fetchTasks 合并逻辑中，任务完成时自动触发 MediaCache.cacheFromURL 缓存媒体文件  `#auto-cache-on-complete`
- [ ] 提交任务成功后自动刷新积分  `#auto-refresh-credit`
- [ ] 实现 queryCredits 方法调用 POST /token/points 接口，支持多 token 汇总  `#credit-query-logic`
- [ ] Configuration.exe 窗口新增 Query Credits 按钮和积分显示区域  `#credit-query-ui`
- [ ] scp index.html 到远程服务器完成部署  `#deploy-to-server`
- [ ] 在 index.html 中实现 IndexedDB MediaCache 工具模块（open/put/get/getObjectURL/cacheFromURL）  `#indexeddb-module`
- [ ] viewResult 改为优先从 IndexedDB 读取 blob URL，fallback 到原始 URL  `#preview-use-cache`
- [ ] modal 关闭和翻页时释放 Object URL 防止内存泄漏  `#revoke-blob-urls`
<!-- LIMCODE_TODO_LIST_END -->
