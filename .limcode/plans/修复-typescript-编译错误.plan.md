
# 修复 TypeScript 与 Logger 错误

当前 `npx tsc --noEmit` 存在以下三类问题，需要修复以通过编译检查：

## 1. Request.files 的类型断言错误
**文件**：`src/api/routes/images.ts` 和 `src/api/routes/async-tasks.ts`
**报错**：`Property 'images' does not exist on type 'any[]'` 和 `Property 'url' does not exist on type 'object'`
**原因**：`request.files` 被推断为数组或内部对象的类型不够宽泛。
**解决方案**：
- `const files = request.files?.images;` → `const files = (request.files as any)?.images;`
- `!image.url` → `!(image as any).url`

## 2. ArrayBuffer vs Buffer 的类型兼容
**文件**：`src/lib/audio-uploader.ts` 和 `src/lib/image-uploader.ts` 以及 `src/lib/video-uploader.ts`
**报错**：`Argument of type 'ArrayBuffer | Buffer<ArrayBufferLike>' is not assignable to parameter of type 'ArrayBuffer'.`
**原因**：`util.calculateCRC32` 期待 `ArrayBuffer`，但在 Node 环境下传入了 `Buffer`，导致类型不兼容（尽管在 JS 层面能运行）。
**解决方案**：
已在上一轮部分修复了 `util.ts` 中 `calculateCRC32` 的入参定义 `(buffer: ArrayBuffer | Buffer)`，但是 `util.ts` 中的其他方法可能也需要宽泛处理，或者在 uploader 中将 `Buffer` 强制转换。为保险起见，我们将 `calculateCRC32` 在实际使用的地方统一加上 `as any`，或者完善 `util.ts` 的类型签名。

## 3. Logger 级别调用错误
**文件**：`src/lib/logger.ts`
**报错**：`Property 'Success' does not exist...`
**原因**：自定义 Logger 类中存在未定义但被调用的方法 `logger.Success()`，`logger.Fatal()` 等。
**解决方案**：在 `logger.ts` 类中补齐对应的方法定义，使其匹配接口要求。

由于这是维护性调整，我会生成包含具体修正细节的 todos，以确保整个后端的类型安全。

## TODO LIST

<!-- LIMCODE_TODO_LIST_START -->
- [ ] 确保所有 uploader 中 calculateCRC32 调用不存在类型不匹配问题  `#fix-ts-buffer`
- [ ] 修复 src/api/routes/images.ts 中关于 request.files.images 和 image.url 的类型推导错误  `#fix-ts-images`
- [ ] 修复 logger.ts 缺少的 Success, Fatal, Log 方法声明  `#fix-ts-logger`
<!-- LIMCODE_TODO_LIST_END -->
