
# 修复 Omni 文件列表重复渲染与结构断裂

## 问题分析

`index.html` 第 595-638 行，omni_reference 文件列表区域存在三个问题：

### 1. Videos 区域（616-621 行）：旧标签残留 + 孤儿按钮
```html
<!-- 旧的 file-tag（没有 insertable、没有预览）-->
<div v-for="..." class="file-tag">                    ← 应删除
  🎬 @video_file_{{ i+1 }}: {{ f.name }} <button>×</button>
</div>
  <button @click.stop="removeOmniVideo(i)">×</button> ← 孤儿按钮，脱离了 div
</div>                                                  ← 多余闭合
```
新的 insertable 标签**完全缺失**。

### 2. Audios 区域（629-637 行）：旧标签残留 + 新标签未闭合
```html
<!-- 旧的 file-tag -->
<div v-for="..." class="file-tag">                    ← 应删除
  🔊 @audio_file_{{ i+1 }}: ... <button>×</button>
</div>
<!-- 新的 insertable 标签 -->
<div v-for="..." class="file-tag insertable">
  🔊 @audio_file_{{ i+1 }}: ...
  <button>👁</button>
  <!-- 缺少 × 按钮，缺少 </div> 闭合 -->
<p>💡 Tip: ...</p>                                     ← Tip 嵌在 file-list 内部
```

### 3. Tip 文本重复出现在每个标签旁边
因为 `<p>` 在 `file-list` div 内且在 `v-for` 循环附近，视觉上看起来每个文件都带一条 Tip。

## 修复方案

**将第 595-638 行整体替换为以下正确结构**：

三种素材（Images/Videos/Audios）统一使用相同的 `file-tag insertable` 模板，Tip 独立放在最后。

```html
            <div v-if="videoForm.functionMode === 'omni_reference'">
              <div class="field-row">
                <label>Images (&le;9):</label>
                <span style="flex:1;"></span>
                <button @click="$refs.omniImgInput.click()">Browse...</button>
                <input type="file" ref="omniImgInput" accept="image/*" multiple @change="onOmniImages($event)" style="display:none;">
              </div>
              <div class="file-list" v-if="videoForm.omniImages.length">
                <div v-for="(f, i) in videoForm.omniImages" :key="i" class="file-tag insertable" @click="insertRefToEditor('image_file_' + (i+1))" title="Click to insert @reference">
                  &#128444;&#65039; @image_file_{{ i+1 }}: {{ f.name }}
                  <button @click.stop="previewLocalFile(f)" title="Preview">👁</button>
                  <button @click.stop="removeOmniImage(i)">×</button>
                </div>
              </div>
              <div class="field-row" style="margin-top: 5px;">
                <label>Videos (&le;3):</label>
                <span style="flex:1;"></span>
                <button @click="$refs.omniVidInput.click()">Browse...</button>
                <input type="file" ref="omniVidInput" accept="video/*" multiple @change="onOmniVideos($event)" style="display:none;">
              </div>
              <div class="file-list" v-if="videoForm.omniVideos.length">
                <div v-for="(f, i) in videoForm.omniVideos" :key="i" class="file-tag insertable" @click="insertRefToEditor('video_file_' + (i+1))" title="Click to insert @reference">
                  &#127916; @video_file_{{ i+1 }}: {{ f.name }}
                  <button @click.stop="previewLocalFile(f)" title="Preview">👁</button>
                  <button @click.stop="removeOmniVideo(i)">×</button>
                </div>
              </div>
              <div class="field-row" style="margin-top: 5px;">
                <label>Audios (&le;3):</label>
                <span style="flex:1;"></span>
                <button @click="$refs.omniAudInput.click()">Browse...</button>
                <input type="file" ref="omniAudInput" accept="audio/*" multiple @change="onOmniAudios($event)" style="display:none;">
              </div>
              <div class="file-list" v-if="videoForm.omniAudios.length">
                <div v-for="(f, i) in videoForm.omniAudios" :key="i" class="file-tag insertable" @click="insertRefToEditor('audio_file_' + (i+1))" title="Click to insert @reference">
                  &#128266; @audio_file_{{ i+1 }}: {{ f.name }}
                  <button @click.stop="previewLocalFile(f)" title="Preview">👁</button>
                  <button @click.stop="removeOmniAudios(i)">×</button>
                </div>
              </div>
              <p style="font-size:10px; color:#808080; margin-top:5px; margin-bottom: 2px;">
                💡 Click a file tag or type <b>@</b> in prompt to insert reference.
              </p>
            </div>
```

### 关键变化
| 项目 | 修复前 | 修复后 |
|------|--------|--------|
| Videos | 旧 file-tag + 孤儿按钮 | 统一 file-tag insertable（含 👁 + ×） |
| Audios | 旧+新标签重复 + 未闭合 | 统一 file-tag insertable（含 👁 + ×） |
| Tip 文本 | 嵌在 file-list 内，每文件旁出现 | 独立 `<p>` 在所有列表之后，仅出现一次 |
| Tip 内容 | 冗长重复的字段名列举 | 简洁："Click a tag or type @ to insert" |

## 操作
用 `apply_diff` 将第 595-638 行替换为上述正确 HTML。

## TODO LIST

<!-- LIMCODE_TODO_LIST_START -->
- [ ] 替换 595-638 行：统一三类素材的 file-tag insertable 结构，修复 Videos/Audios 的旧代码残留和 HTML 断裂  `#fix-omni-html`
- [ ] Tip 文本移至文件列表末尾独立显示，简化文案  `#fix-tip-text`
<!-- LIMCODE_TODO_LIST_END -->
