
# Omni Prompt 富文本编辑器 - @ 引用选择与预览

## 背景
当前 omni_reference 模式的 prompt 是一个普通 `<textarea>`，用户需手动输入 `@image_file_1` 这样的引用键名。体验差且易出错。

## 目标
1. **@ 自动补全**：输入 `@` 后弹出 Win2000 风格的下拉菜单，列出所有已上传的素材（图片/视频/音频），支持按键过滤
2. **原子标签**：选中素材后插入一个不可编辑的内联标签（chip），不是纯文本。删除时整体删除，不可部分编辑
3. **点击预览**：点击编辑器中的标签或文件列表中的标签，弹出 Win2000 风格的预览窗口（图片/视频/音频均支持）
4. **数据同步**：编辑器内容实时同步到 `videoForm.prompt`（带 `@field_name` 的纯文本），提交时后端 `parseOmniPrompt` 直接可用
5. **Win2000 风格**：所有新增 UI 元素保持 98.css / Win2000 复古风格

## 技术方案

### 核心：contenteditable + contenteditable="false" 子元素
- 用 `<div contenteditable="true">` 替代 omni 模式下的 `<textarea>`
- 引用标签为 `<span contenteditable="false" class="ref-tag" data-field="image_file_1" data-type="image">🖼️ @image_file_1</span>`
- 浏览器原生行为：`contenteditable="false"` 的子元素在 backspace/delete 时会被整体删除
- 标签之间插入零宽空格 `\u200B` 保证光标可定位

### 组件结构

```
┌─ Prompt Editor Wrapper (.prompt-editor-wrapper) ─────────┐
│  ┌─ contenteditable div (.prompt-editor) ───────────────┐ │
│  │  "使用"                                               │ │
│  │  [🖼️ @image_file_1]  ← ref-tag (atomic)             │ │
│  │  "的角色，配上"                                        │ │
│  │  [🔊 @audio_file_1]  ← ref-tag (atomic)             │ │
│  │  "声线，生成视频"                                      │ │
│  └──────────────────────────────────────────────────────┘ │
│  ┌─ Dropdown (.ref-dropdown) ───────────────────────────┐ │
│  │  🖼️ image_file_1: 野比大雄.jpg                       │ │
│  │  🎬 video_file_1: dance.mp4                          │ │
│  │  🔊 audio_file_1: voice.mp3                          │ │
│  └──────────────────────────────────────────────────────┘ │
└──────────────────────────────────────────────────────────┘
```

## 详细改动清单

### 1. CSS 新增样式 (~50 行)

在 `<style>` 末尾（第 303 行 `}` 之后）追加：

```css
/* === Prompt Rich Editor === */
.prompt-editor-wrapper {
  flex: 1;
  min-width: 200px;
  position: relative;
}
.prompt-editor {
  border: 2px inset;
  background: white;
  min-height: 60px;
  max-height: 120px;
  padding: 4px;
  font-family: 'Pixelated MS Sans Serif', Arial;
  font-size: 11px;
  overflow-y: auto;
  white-space: pre-wrap;
  word-break: break-word;
  cursor: text;
  outline: none;
}
.prompt-editor:empty::before {
  content: attr(data-placeholder);
  color: #808080;
  pointer-events: none;
}
.ref-tag {
  display: inline-block;
  background: #d4d0c8;
  border: 1px outset #fff;
  padding: 1px 5px;
  margin: 1px 2px;
  font-size: 10px;
  font-weight: bold;
  cursor: pointer;
  user-select: all;
  vertical-align: baseline;
  white-space: nowrap;
  border-radius: 0;
}
.ref-tag:hover { background: #e0ddd5; }
.ref-tag[data-type="image"] { border-left: 3px solid #008000; }
.ref-tag[data-type="video"] { border-left: 3px solid #000080; }
.ref-tag[data-type="audio"] { border-left: 3px solid #800000; }

/* @ Dropdown */
.ref-dropdown {
  position: absolute;
  left: 0; right: 0;
  bottom: 100%;
  max-height: 160px;
  overflow-y: auto;
  background: white;
  border: 2px outset #dfdfdf;
  box-shadow: 2px 2px 0 rgba(0,0,0,0.3);
  z-index: 50;
}
.ref-dropdown-item {
  padding: 3px 8px;
  font-size: 11px;
  cursor: pointer;
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
}
.ref-dropdown-item:hover,
.ref-dropdown-item.active {
  background: #000080;
  color: white;
}
.ref-dropdown-empty {
  padding: 6px 8px;
  font-size: 11px;
  color: #808080;
  font-style: italic;
}

/* Reference Preview Modal */
.ref-preview-overlay {
  position: fixed;
  top: 0; left: 0; right: 0; bottom: 0;
  background: rgba(0,0,0,0.4);
  display: flex;
  justify-content: center;
  align-items: center;
  z-index: 2000;
  padding: 20px;
}
.ref-preview-window {
  max-width: 500px;
  width: 100%;
  max-height: 80vh;
}
.ref-preview-body {
  padding: 8px;
  text-align: center;
}
.ref-preview-body img,
.ref-preview-body video {
  max-width: 100%;
  max-height: 50vh;
  border: 2px inset;
  background: #000;
  object-fit: contain;
}
.ref-preview-body audio {
  width: 100%;
  margin: 10px 0;
}

/* 文件列表标签可点击插入 */
.file-tag.insertable {
  cursor: pointer;
}
.file-tag.insertable:hover {
  background: #e8e8ff;
}
```

### 2. HTML 模板改动

#### 2a. Video tab prompt 区域（第 518-521 行）
替换为条件渲染：omni 模式用富文本编辑器，其他模式保留 textarea。

```html
<!-- Omni mode: Rich prompt editor -->
<div v-if="videoForm.functionMode === 'omni_reference'" class="field-row" style="align-items: flex-start;">
  <label>Prompt:</label>
  <div class="prompt-editor-wrapper">
    <div ref="promptEditor" 
         class="prompt-editor" 
         contenteditable="true"
         data-placeholder="Type text, use @ to insert references..."
         @input="onEditorInput"
         @keydown="onEditorKeydown"
         @click="onEditorClick"
         @paste="onEditorPaste"></div>
    <!-- @ Autocomplete dropdown -->
    <div class="ref-dropdown" v-if="showRefDropdown">
      <div v-if="filteredMaterials.length === 0" class="ref-dropdown-empty">
        No materials uploaded yet.
      </div>
      <div v-for="(item, idx) in filteredMaterials" 
           :key="item.field"
           class="ref-dropdown-item"
           :class="{ active: refDropdownIndex === idx }"
           @mousedown.prevent="insertRefFromDropdown(item)">
        {{ item.icon }} {{ item.field }}: {{ item.displayName }}
      </div>
    </div>
  </div>
</div>
<!-- Other modes: plain textarea -->
<div v-else class="field-row" style="align-items: flex-start;">
  <label>Prompt:</label>
  <textarea v-model="videoForm.prompt" placeholder="Enter video description..."></textarea>
</div>
```

#### 2b. Submit 按钮禁用条件（第 522 行）
修改为：omni 模式下检查 `editorHasContent`（因为 contenteditable 不绑定 v-model）：

```html
<button class="btn-block" @click="submitVideoTask" 
  :disabled="!isConfigValid || (videoForm.functionMode === 'omni_reference' ? !editorHasContent : !videoForm.prompt) || submitting">
  {{ submitting ? 'Submitting...' : 'Generate Video' }}
</button>
```

#### 2c. 文件列表标签改为可点击插入（第 486-489 行 omniImages 区域）
给 file-tag 增加 `insertable` class 和点击事件，以及预览按钮：

```html
<div v-for="(f, i) in videoForm.omniImages" :key="i" 
     class="file-tag insertable" 
     @click="insertRefToEditor('image_file_' + (i+1))" 
     title="Click to insert @reference">
  🖼️ @image_file_{{ i+1 }}: {{ f.name }}
  <button @click.stop="previewLocalFile(f)" title="Preview">👁</button>
  <button @click.stop="removeOmniImage(i)">×</button>
</div>
```

同理 omniVideos（497-499）、omniAudios（508-510）也做同样修改。

#### 2d. 新增 Reference Preview Modal
在 Error Modal 之前（第 601 行前）插入：

```html
<!-- Reference Preview Modal -->
<div class="ref-preview-overlay" v-if="refPreview.show" @click.self="closeRefPreview">
  <div class="window ref-preview-window">
    <div class="title-bar">
      <div class="title-bar-text">📎 Preview - {{ refPreview.name }}</div>
      <div class="title-bar-controls">
        <button aria-label="Close" @click="closeRefPreview"></button>
      </div>
    </div>
    <div class="window-body ref-preview-body">
      <img v-if="refPreview.mediaType === 'image'" :src="refPreview.url">
      <video v-else-if="refPreview.mediaType === 'video'" :src="refPreview.url" controls autoplay></video>
      <audio v-else-if="refPreview.mediaType === 'audio'" :src="refPreview.url" controls autoplay></audio>
      <p v-else style="color: #808080;">Preview not available</p>
    </div>
  </div>
</div>
```

### 3. JavaScript 新增逻辑

#### 3a. 新增响应式状态

```javascript
// Rich editor state
const showRefDropdown = ref(false);
const refDropdownIndex = ref(0);
const refFilterText = ref('');
const editorHasContent = ref(false);
const promptEditor = ref(null); // template ref

// Preview state
const refPreview = reactive({
  show: false,
  url: '',
  name: '',
  mediaType: '' // 'image' | 'video' | 'audio'
});
```

#### 3b. computed: availableMaterials

```javascript
const availableMaterials = computed(() => {
  const list = [];
  videoForm.omniImages.forEach((f, i) => {
    list.push({ field: `image_file_${i+1}`, type: 'image', icon: '🖼️', displayName: f.name, file: f });
  });
  videoForm.omniVideos.forEach((f, i) => {
    list.push({ field: `video_file_${i+1}`, type: 'video', icon: '🎬', displayName: f.name, file: f });
  });
  videoForm.omniAudios.forEach((f, i) => {
    list.push({ field: `audio_file_${i+1}`, type: 'audio', icon: '🔊', displayName: f.name, file: f });
  });
  return list;
});

const filteredMaterials = computed(() => {
  const filter = refFilterText.value.toLowerCase();
  if (!filter) return availableMaterials.value;
  return availableMaterials.value.filter(m => 
    m.field.toLowerCase().includes(filter) || 
    m.displayName.toLowerCase().includes(filter)
  );
});
```

#### 3c. 编辑器事件处理函数

```javascript
// 创建 ref-tag HTML
function createRefTagHTML(field, type, icon) {
  return `<span contenteditable="false" class="ref-tag" data-field="${field}" data-type="${type}">${icon} @${field}</span>\u200B`;
}

// 从编辑器内容提取 prompt 字符串
function extractPromptFromEditor() {
  const editor = promptEditor.value;
  if (!editor) return '';
  let result = '';
  function walk(node) {
    if (node.nodeType === Node.TEXT_NODE) {
      // 移除零宽空格
      result += node.textContent.replace(/\u200B/g, '');
    } else if (node.nodeType === Node.ELEMENT_NODE) {
      if (node.classList?.contains('ref-tag')) {
        result += '@' + node.getAttribute('data-field');
      } else if (node.tagName === 'BR') {
        result += '\n';
      } else {
        node.childNodes.forEach(walk);
      }
    }
  }
  editor.childNodes.forEach(walk);
  return result.trim();
}

// 同步编辑器 → videoForm.prompt
function syncEditorToPrompt() {
  videoForm.prompt = extractPromptFromEditor();
  editorHasContent.value = videoForm.prompt.length > 0;
}

// Input 事件：检测 @ 并显示下拉
function onEditorInput(e) {
  syncEditorToPrompt();
  // 检测是否正在输入 @xxx
  detectAtSymbol();
}

function detectAtSymbol() {
  const sel = window.getSelection();
  if (!sel.rangeCount) { showRefDropdown.value = false; return; }
  const range = sel.getRangeAt(0);
  const node = range.startContainer;
  if (node.nodeType !== Node.TEXT_NODE) { showRefDropdown.value = false; return; }
  
  const text = node.textContent.substring(0, range.startOffset);
  const atIdx = text.lastIndexOf('@');
  if (atIdx === -1) { showRefDropdown.value = false; return; }
  
  // 确保 @ 后面没有空格
  const filterStr = text.substring(atIdx + 1);
  if (/\s/.test(filterStr)) { showRefDropdown.value = false; return; }
  
  refFilterText.value = filterStr;
  refDropdownIndex.value = 0;
  showRefDropdown.value = true;
}

// 键盘事件：在下拉菜单打开时处理方向键/回车/ESC
function onEditorKeydown(e) {
  if (showRefDropdown.value) {
    const items = filteredMaterials.value;
    if (e.key === 'ArrowDown') {
      e.preventDefault();
      refDropdownIndex.value = Math.min(refDropdownIndex.value + 1, items.length - 1);
    } else if (e.key === 'ArrowUp') {
      e.preventDefault();
      refDropdownIndex.value = Math.max(refDropdownIndex.value - 1, 0);
    } else if (e.key === 'Enter' || e.key === 'Tab') {
      e.preventDefault();
      if (items.length > 0) {
        insertRefFromDropdown(items[refDropdownIndex.value]);
      }
    } else if (e.key === 'Escape') {
      e.preventDefault();
      showRefDropdown.value = false;
    }
  }
}

// 选择下拉项 → 删除 @filter 文本 → 插入标签
function insertRefFromDropdown(item) {
  showRefDropdown.value = false;
  const sel = window.getSelection();
  if (!sel.rangeCount) return;
  
  const range = sel.getRangeAt(0);
  const node = range.startContainer;
  if (node.nodeType !== Node.TEXT_NODE) return;
  
  const text = node.textContent;
  const cursorPos = range.startOffset;
  const atIdx = text.lastIndexOf('@', cursorPos - 1);
  if (atIdx === -1) return;
  
  // 删除 @filter 文本
  const before = text.substring(0, atIdx);
  const after = text.substring(cursorPos);
  node.textContent = before;
  
  // 插入 tag span
  const tagSpan = document.createElement('span');
  tagSpan.contentEditable = 'false';
  tagSpan.className = 'ref-tag';
  tagSpan.dataset.field = item.field;
  tagSpan.dataset.type = item.type;
  tagSpan.textContent = `${item.icon} @${item.field}`;
  
  // 插入到 before 之后
  const afterNode = document.createTextNode('\u200B' + after);
  const parent = node.parentNode;
  parent.insertBefore(tagSpan, node.nextSibling);
  parent.insertBefore(afterNode, tagSpan.nextSibling);
  
  // 把光标放到 afterNode 开头（跳过零宽空格）
  const newRange = document.createRange();
  newRange.setStart(afterNode, 1);
  newRange.collapse(true);
  sel.removeAllRanges();
  sel.addRange(newRange);
  
  syncEditorToPrompt();
}

// 从文件列表点击插入引用到编辑器
function insertRefToEditor(fieldName) {
  const item = availableMaterials.value.find(m => m.field === fieldName);
  if (!item) return;
  
  const editor = promptEditor.value;
  if (!editor) return;
  
  // 在末尾追加
  const tagHTML = createRefTagHTML(item.field, item.type, item.icon);
  editor.innerHTML += tagHTML;
  
  // 光标移到末尾
  const sel = window.getSelection();
  const range = document.createRange();
  range.selectNodeContents(editor);
  range.collapse(false);
  sel.removeAllRanges();
  sel.addRange(range);
  
  editor.focus();
  syncEditorToPrompt();
}

// 点击编辑器：检查是否点击了 ref-tag → 显示预览
function onEditorClick(e) {
  const tag = e.target.closest?.('.ref-tag');
  if (tag) {
    const fieldName = tag.dataset.field;
    const item = availableMaterials.value.find(m => m.field === fieldName);
    if (item?.file) {
      showFilePreview(item.file, item.type, item.displayName);
    }
  }
}

// 粘贴事件：只粘贴纯文本
function onEditorPaste(e) {
  e.preventDefault();
  const text = e.clipboardData.getData('text/plain');
  document.execCommand('insertText', false, text);
}
```

#### 3d. 预览功能

```javascript
function showFilePreview(file, type, name) {
  // 释放旧 URL
  if (refPreview.url && refPreview.url.startsWith('blob:')) {
    URL.revokeObjectURL(refPreview.url);
  }
  refPreview.url = URL.createObjectURL(file);
  refPreview.name = name || file.name;
  refPreview.mediaType = type; // 'image' | 'video' | 'audio'
  refPreview.show = true;
}

function closeRefPreview() {
  if (refPreview.url && refPreview.url.startsWith('blob:')) {
    URL.revokeObjectURL(refPreview.url);
  }
  refPreview.show = false;
  refPreview.url = '';
}

// 从文件列表预览按钮调用
function previewLocalFile(file) {
  let type = 'image';
  if (file.type.startsWith('video/')) type = 'video';
  else if (file.type.startsWith('audio/')) type = 'audio';
  showFilePreview(file, type, file.name);
}
```

#### 3e. 修改 openRefPreview（第 1154 行）
原有的 `openRefPreview` 改为使用新的预览弹窗替代 `window.open`：

```javascript
const openRefPreview = (url, name, type) => {
  if (refPreview.url && refPreview.url.startsWith('blob:')) {
    URL.revokeObjectURL(refPreview.url);
  }
  refPreview.url = url;
  refPreview.name = name || 'Reference';
  refPreview.mediaType = type || 'image';
  refPreview.show = true;
};
```

#### 3f. Task Details 中的参考文件预览按钮改进
修改第 579 行，传递更多信息给 openRefPreview：

```html
<button v-if="rf.previewUrl" 
  @click="openRefPreview(rf.previewUrl, rf.name, rf.type.startsWith('image') ? 'image' : rf.type.startsWith('video') ? 'video' : 'audio')" 
  title="Preview">👁</button>
```

#### 3g. watcher: 素材列表变化时清理失效标签
当用户删除已上传文件后，编辑器里对应的标签应标记为失效或移除：

```javascript
watch([() => videoForm.omniImages.length, () => videoForm.omniVideos.length, () => videoForm.omniAudios.length], () => {
  // 清理编辑器中引用了不存在素材的标签
  if (!promptEditor.value) return;
  const validFields = new Set(availableMaterials.value.map(m => m.field));
  const tags = promptEditor.value.querySelectorAll('.ref-tag');
  tags.forEach(tag => {
    if (!validFields.has(tag.dataset.field)) {
      // 移除失效标签
      tag.parentNode?.removeChild(tag);
    }
  });
  syncEditorToPrompt();
});
```

#### 3h. watcher: 切换 functionMode 时清空编辑器
```javascript
watch(() => videoForm.functionMode, (newMode, oldMode) => {
  // 已有: 清空文件列表
  // 新增: 当切回 omni 模式时，同步 prompt 到编辑器
  if (newMode === 'omni_reference') {
    Vue.nextTick(() => {
      if (promptEditor.value && videoForm.prompt) {
        promptEditor.value.textContent = videoForm.prompt;
        syncEditorToPrompt();
      }
    });
  }
});
```

#### 3i. submitVideoTask 修改
在提交前确保从编辑器同步 prompt：

```javascript
// 在 submitVideoTask 开头添加：
if (videoForm.functionMode === 'omni_reference') {
  syncEditorToPrompt(); // 确保最新
}
```

#### 3j. return 对象更新
新增暴露：
```javascript
return {
  // ...existing...
  promptEditor, showRefDropdown, refDropdownIndex, filteredMaterials, editorHasContent,
  refPreview,
  onEditorInput, onEditorKeydown, onEditorClick, onEditorPaste,
  insertRefFromDropdown, insertRefToEditor, previewLocalFile,
  closeRefPreview, showFilePreview,
};
```

## 交互流程

### @ 自动补全流程
```
用户输入 @ → detectAtSymbol() → showRefDropdown = true
  ↓ 继续输入 "im" → refFilterText = "im" → filteredMaterials 过滤
  ↓ 按 ↓↑ → refDropdownIndex 移动
  ↓ 按 Enter/Tab 或鼠标点击 → insertRefFromDropdown(item)
  ↓ 删除 "@im" 文本 → 插入 <span class="ref-tag">🖼️ @image_file_1</span>
  ↓ syncEditorToPrompt() → videoForm.prompt 更新
```

### 预览流程
```
点击编辑器中的 ref-tag → onEditorClick → showFilePreview(file)
点击文件列表的 👁 按钮 → previewLocalFile(file)
点击任务详情的 👁 按钮 → openRefPreview(url, name, type)
  ↓ refPreview.show = true → 显示 Win2000 弹窗
  ↓ 根据 mediaType 渲染 <img> / <video> / <audio>
```

### 文件列表点击插入流程
```
点击 omniImages 的文件标签 → insertRefToEditor('image_file_1')
  ↓ 在编辑器末尾追加 ref-tag
  ↓ syncEditorToPrompt()
```

## 注意事项
1. **contenteditable 粘贴安全**：`onEditorPaste` 拦截粘贴事件，只保留纯文本，防止粘入 HTML 格式
2. **零宽空格**：标签前后的 `\u200B` 保证光标可以在标签之间定位
3. **文件索引重编**：删除中间文件后，剩余文件的 field 名会自动重编号（因为 computed 根据数组 index 计算），编辑器中的旧标签会被 watcher 清理
4. **后端兼容**：不需要修改后端，`videoForm.prompt` 仍然是 `"@image_file_1使用@audio_file_1声线..."` 格式

## TODO LIST

<!-- LIMCODE_TODO_LIST_START -->
- [ ] 新增 CSS 样式：prompt-editor、ref-tag、ref-dropdown、ref-preview 等  `#css-styles`
- [ ] HTML：omni 模式条件渲染富文本编辑器 + 下拉菜单，其他模式保留 textarea  `#html-editor`
- [ ] HTML：文件列表标签增加点击插入引用 + 预览按钮  `#html-file-tags`
- [ ] HTML：新增 Reference Preview Modal（Win2000 风格弹窗）  `#html-preview`
- [ ] JS：computed availableMaterials / filteredMaterials  `#js-computed`
- [ ] JS：编辑器核心逻辑（@检测、标签插入、prompt提取、同步）  `#js-editor-core`
- [ ] JS：键盘交互（方向键选择、Enter/Tab 确认、Escape 关闭）  `#js-keyboard`
- [ ] JS：预览功能（showFilePreview, closeRefPreview, previewLocalFile）  `#js-preview`
- [ ] JS：return 对象新增暴露所有新增方法/状态  `#js-return`
- [ ] JS：新增响应式状态（showRefDropdown, refPreview, editorHasContent 等）  `#js-state`
- [ ] JS：submitVideoTask 提交前同步编辑器内容  `#js-submit-sync`
- [ ] JS：watcher - 素材删除时清理失效标签 + 模式切换同步  `#js-watchers`
- [ ] 测试验证：@ 补全、原子删除、预览弹窗、数据同步、提交  `#test-verify`
<!-- LIMCODE_TODO_LIST_END -->
