
# 音频上传后端支持 + 参考素材 IndexedDB 缓存

## 背景

根据抓包分析，即梦官方 API 在 Seedance 2.0 (Fast) 的 `omni_reference` 模式下已支持音频素材：
- `material_type: "audio"` + `audio_info: { vid, duration, source_from: "upload" }`
- `meta_list` 中用 `meta_type: "audio"` 引用
- `materialTypes` 数组中 `3` 代表音频
- 限制：最多 3 个音频，每个 2~15 秒，≤15MB
- 音频通过 VOD 服务上传（与视频相同的 `vod.bytedanceapi.com`），仅 `FileType` 不同

同时，为保证任务历史的完整性，需要在提交任务时将用户上传的参考素材（图片/视频/音频）缓存到 IndexedDB，在任务详情面板中回显。

---

## 第一部分：后端 - 音频上传支持

### 1.1 创建 `src/lib/audio-uploader.ts`

**核心思路**：音频与视频使用相同的 VOD 上传通道，仅两处差异：
- `ApplyUploadInner` 请求中 `FileType=audio`（而非 `video`）
- Commit 后返回的元信息字段不同（音频没有 Width/Height，有 Duration）

**实现方式**：从 `video-uploader.ts` 复制并修改（不是重构为通用函数，保持改动最小、不影响已有逻辑）。

```typescript
// src/lib/audio-uploader.ts
export interface AudioUploadResult {
  vid: string;
  uri: string;
  audioMeta: {
    duration: number;     // 秒
    durationMs: number;   // 毫秒
    format: string;
    size: number;
    md5: string;
  };
}

export async function uploadAudioBuffer(
  audioBuffer: ArrayBuffer | Buffer,
  refreshToken: string,
  regionInfo: RegionInfo
): Promise<AudioUploadResult>
```

**与 `video-uploader.ts` 的差异点**：
1. **第 67 行**：`FileType=video` → `FileType=audio`
2. **第 267-271 行**（时长校验）：改为 `MAX_AUDIO_DURATION = 15`，`MIN_AUDIO_DURATION = 2`
3. **返回值结构**：不含 `width/height/codec/bitrate`，改为 `audioMeta: { duration, durationMs, format, size, md5 }`
4. **日志标记**：所有 `logger.info` 中的 "视频" → "音频"

同时提供 `uploadAudioFromUrl` 函数（镜像 `uploadVideoFromUrl`）。

### 1.2 修改 `src/api/controllers/videos.ts` - omni_reference 分支

**需要改动的位置**（约 260~500 行的 `isOmniMode` 分支）：

#### 1.2.1 扩展 `MaterialEntry` 类型（~第 268 行）

```typescript
interface MaterialEntry {
  idx: number;
  type: "image" | "video" | "audio";  // 新增 "audio"
  fieldName: string;
  originalFilename: string;
  imageUri?: string;
  videoResult?: VideoUploadResult;
  audioResult?: AudioUploadResult;    // 新增
}
```

#### 1.2.2 扩展 canonical key 集合（~第 278-279 行）

```typescript
for (let i = 1; i <= 3; i++) canonicalKeys.add(`audio_file_${i}`);  // 新增
```

#### 1.2.3 收集音频字段（~第 288-312 行）

新增 `audioFields: string[]`，检测 `audio_file_*` 上传文件和 URL 字段。

#### 1.2.4 新增音频上传循环（在视频上传循环后，~第 438 行之后）

```typescript
// 串行上传音频素材
let totalAudioDuration = 0;
for (const fieldName of audioFields) {
  const audioFile = files?.[fieldName];
  const audioUrlField = httpRequest?.body?.[fieldName];
  // ... 调用 uploadAudioBuffer / uploadAudioFromUrl
  // ... 注册到 materialRegistry
  totalAudioDuration += audioResult.audioMeta.duration;
}

// 验证音频总时长
const MAX_TOTAL_AUDIO_DURATION = 15;
if (totalAudioDuration > MAX_TOTAL_AUDIO_DURATION) {
  throw new APIException(...);
}
```

#### 1.2.5 扩展 `material_list` 构建逻辑（~第 460-499 行）

新增 `audio` 分支：

```typescript
if (entry.type === "audio") {
  const am = entry.audioResult!;
  material_list.push({
    type: "",
    id: util.uuid(),
    material_type: "audio",
    audio_info: {
      type: "audio",
      id: util.uuid(),
      source_from: "upload",
      vid: am.vid,
      duration: am.audioMeta.durationMs, // 毫秒
      name: "",
    },
  });
  materialTypes.push(3); // 3 = audio
}
```

#### 1.2.6 检查素材要求

根据抓包中的 `required_material_types.any_of: [1, 2]`，音频不能单独使用，必须搭配至少一张图片或一段视频。加入验证：

```typescript
const hasImageOrVideo = materialTypes.some(t => t === 1 || t === 2);
if (!hasImageOrVideo) {
  throw new APIException(EX.API_REQUEST_FAILED,
    `omni_reference 模式中使用音频素材时，至少需要同时提供一张图片或一段视频`);
}
```

### 1.3 修改 `src/api/routes/async-tasks.ts`

无需大改。当前异步路由的 `/videos/generations` 端点已经通过 `request.files` 把所有 multipart 字段传给了 `generateVideo`，只要前端在 FormData 中用 `audio_file_1` 等字段名 append 音频文件即可。

唯一需要确认的：formidable（解析器）是否接受音频 MIME 类型。检查 `src/lib/request/Request.ts` 的 formidable 配置，确保不限制文件类型或已包含 `audio/*`。

---

## 第二部分：前端 - 音频上传 UI

### 2.1 扩展 `videoForm` 响应式对象

```javascript
const videoForm = reactive({
  // ... 现有字段
  omniAudios: [],  // 新增：File[]
});
```

### 2.2 新增音频文件处理函数

```javascript
const onOmniAudios = (e) => {
  const files = Array.from(e.target.files);
  videoForm.omniAudios.push(...files.slice(0, Math.max(0, 3 - videoForm.omniAudios.length)));
  e.target.value = '';
};
const removeOmniAudio = (i) => videoForm.omniAudios.splice(i, 1);
```

### 2.3 修改 HTML 模板（~第 459-481 行）

在 `omni_reference` 的 Videos 区块后新增 Audio 区块：

```html
<div class="field-row" style="margin-top: 5px;">
  <label>Audios (≤3):</label>
  <input type="file" accept="audio/*" multiple @change="onOmniAudios($event)" style="flex:1;">
</div>
<div class="file-list" v-if="videoForm.omniAudios.length">
  <div v-for="(f, i) in videoForm.omniAudios" :key="i" class="file-tag">
    🔊 @audio_file_{{ i+1 }}: {{ f.name }} <button @click="removeOmniAudio(i)">&times;</button>
  </div>
</div>
```

更新 Tip 文字：
```
💡 Tip: Use @image_file_1, @video_file_1, @audio_file_1, etc. in your prompt to reference materials.
```

### 2.4 修改 `submitVideoTask` 方法

在 `hasFiles` 判断中加入 `videoForm.omniAudios.length > 0`：

```javascript
const hasFiles = videoForm.standardFiles.length > 0 
  || videoForm.omniImages.length > 0 
  || videoForm.omniVideos.length > 0
  || videoForm.omniAudios.length > 0;  // 新增
```

在 FormData 构建中新增：
```javascript
videoForm.omniAudios.forEach((f, i) => fd.append(`audio_file_${i+1}`, f));
```

---

## 第三部分：前端 - 参考素材 IndexedDB 缓存

### 3.1 扩展 MediaCache 模块

在现有 IndexedDB store 旁新建一个 Object Store `ref_media`（需 DB_VERSION 升级为 2）：

```javascript
const REF_STORE_NAME = 'ref_media';

// onupgradeneeded 中：
if (!db.objectStoreNames.contains(REF_STORE_NAME)) {
  db.createObjectStore(REF_STORE_NAME, { keyPath: 'key' });
}
```

新增方法：
```javascript
// 存储参考素材
async putRef(taskId, files) {
  // files: [{ name, type, fieldName, blob }]
  const db = await this.openDB();
  const tx = db.transaction(REF_STORE_NAME, 'readwrite');
  tx.objectStore(REF_STORE_NAME).put({
    key: taskId,
    files: files,  // Blob 对象数组
    cachedAt: Date.now()
  });
}

// 读取参考素材
async getRef(taskId) { ... }

// 删除参考素材
async removeRef(taskId) { ... }
```

### 3.2 在 `submitVideoTask` 中缓存参考素材

提交成功后，将用户选择的 File 对象序列化存入 IndexedDB：

```javascript
if (res.data?.task_id && hasFiles) {
  const refFiles = [];
  if (videoForm.functionMode === 'first_last_frames') {
    videoForm.standardFiles.forEach((f, i) => {
      refFiles.push({ name: f.name, type: f.type, fieldName: `image_${i+1}`, blob: f });
    });
  } else {
    videoForm.omniImages.forEach((f, i) => {
      refFiles.push({ name: f.name, type: f.type, fieldName: `image_file_${i+1}`, blob: f });
    });
    videoForm.omniVideos.forEach((f, i) => {
      refFiles.push({ name: f.name, type: f.type, fieldName: `video_file_${i+1}`, blob: f });
    });
    videoForm.omniAudios.forEach((f, i) => {
      refFiles.push({ name: f.name, type: f.type, fieldName: `audio_file_${i+1}`, blob: f });
    });
  }
  if (refFiles.length > 0) {
    MediaCache.putRef(res.data.task_id, refFiles);
  }
}
```

### 3.3 在任务详情面板中显示参考素材

在 Task Details `<fieldset>` 中（~第 529-543 行），新增参考素材展示区域：

```html
<div v-if="selectedRefFiles && selectedRefFiles.length" style="margin-top: 6px;">
  <strong>References:</strong>
  <div class="file-list" style="margin-top: 3px;">
    <div v-for="rf in selectedRefFiles" :key="rf.fieldName" class="file-tag">
      {{ rf.type.startsWith('image') ? '🖼️' : rf.type.startsWith('video') ? '🎬' : '🔊' }}
      {{ rf.fieldName }}: {{ rf.name }}
      <button v-if="rf.previewUrl" @click="previewRef(rf)" title="Preview">👁</button>
    </div>
  </div>
</div>
```

新增 computed：
```javascript
const selectedRefFiles = ref([]);
watch(() => selectedTaskId.value, async (taskId) => {
  if (!taskId) { selectedRefFiles.value = []; return; }
  const refs = await MediaCache.getRef(taskId);
  if (refs) {
    selectedRefFiles.value = refs.files.map(f => ({
      ...f,
      previewUrl: f.type.startsWith('image/') ? URL.createObjectURL(f.blob) : null
    }));
  } else {
    selectedRefFiles.value = [];
  }
});
```

### 3.4 删除任务时清理参考素材缓存

在 `deleteTask` 函数中追加：
```javascript
MediaCache.removeRef(taskId);
```

---

## 第四部分：需确认的边界条件

### 4.1 Formidable 文件类型限制

需检查 `src/lib/request/Request.ts` 中的 formidable 配置，确保音频文件（`audio/mpeg`, `audio/wav`, `audio/mp3` 等）不被拒绝。如果有 `filter` 限制，需要放开。

### 4.2 IndexedDB 版本迁移

DB_VERSION 从 1 升级到 2 时，需要在 `onupgradeneeded` 中正确处理旧版本的迁移，确保已有的 `media_cache` store 不被删除。

### 4.3 音频文件大小限制

前端在 `onOmniAudios` 中加入前置校验：
```javascript
if (f.size > 15 * 1024 * 1024) {
  showError(`Audio file "${f.name}" exceeds 15MB limit.`);
  return;
}
```

---

## 文件变更清单

| 文件 | 变更类型 | 说明 |
|---|---|---|
| `src/lib/audio-uploader.ts` | **新建** | 音频上传模块（基于 video-uploader 修改） |
| `src/api/controllers/videos.ts` | 修改 | omni_reference 分支支持 audio 素材 |
| `src/api/routes/async-tasks.ts` | 微调 | 确认 formidable 兼容音频（可能不需要改） |
| `index.html` | 修改 | 新增音频上传 UI + 参考素材缓存 + 详情展示 |

## 预估工作量

- 后端音频上传器：~300 行（大部分复制自 video-uploader）
- 后端控制器修改：~80 行增量
- 前端音频 UI：~30 行 HTML + ~20 行 JS
- 前端参考素材缓存：~60 行 JS + ~20 行 HTML
- 测试验证：需要实际的 session token 和音频文件

## TODO LIST

<!-- LIMCODE_TODO_LIST_START -->
- [ ] 创建 src/lib/audio-uploader.ts 音频上传模块（基于 video-uploader，FileType=audio）  `#audio-uploader`
- [ ] 提交任务成功后将参考素材 File 对象缓存到 IndexedDB  `#cache-on-submit`
- [ ] 检查 formidable 配置确保接受 audio/* MIME 类型  `#check-formidable`
- [ ] deleteTask 时同步清理 ref_media 缓存  `#cleanup-refs-on-delete`
- [ ] 修改 src/api/controllers/videos.ts omni_reference 分支，支持 audio 素材类型  `#controller-audio`
- [ ] 部署到远程服务器并进行端到端测试  `#deploy-and-test`
- [ ] 修改 submitVideoTask，在 FormData 中 append audio_file_* 字段  `#frontend-audio-submit`
- [ ] 前端 index.html 新增音频上传 UI（file input + file tags + handlers）  `#frontend-audio-ui`
- [ ] 扩展 MediaCache 模块，新增 ref_media Object Store（DB_VERSION 升级）  `#indexeddb-ref-store`
- [ ] 任务详情面板中展示缓存的参考素材（文件名 + 类型图标 + 图片预览）  `#show-refs-in-detail`
<!-- LIMCODE_TODO_LIST_END -->
