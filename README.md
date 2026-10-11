# 克图 ketu

手搓的开源美颜修图小 App（醒图平替·能用版）。单文件 Web App，全部本地处理，照片不离开设备。

- **线上版（artifact，私有）**: https://claude.ai/artifact/V8oNe3R5g25yBDCppKHJKh
- **技术栈**: MediaPipe Face Landmarker（WASM，478 关键点，打包在 `vendor/`）+ WebGL 管线
- **功能 v0.1**: 磨皮（导皮肤 mask + 边缘保护）、美白、瘦脸、大眼、瘦鼻、亮眼、6 款滤镜（调性偏清透白皙）、按住对比、保存（claude.ai 里走 downloads 能力，否则长按图片兜底）
- **v0.1.1**: 加法令纹淡化——鼻翼→嘴角专属遮罩带（mask B 通道，双描抗模糊稀释），shader 里 `max(col, blur*1.04)` 只提沟影暗部、保皮肤纹理；实测深纹中年脸有效且无亮带伪影
- **v0.2**（甲方真图调校版）: ①法令纹升级为**侧采样**——参考色从沟两侧干净皮肤取（`uFoldO0/O1` 法向偏移），深沟不再被自己的影子拉暗参考值；②新增**收颊**滑杆（中脸颊 lm 50/205/280/425 向鼻中轴收，和瘦脸分管上下半张脸）；③**画布缩放**——双指捏合/滚轮/双击放大图片（最大 8x），工具条固定不动

## 开发

- 源文件就是 `index.html`（含 DEV-SKELETON 哨兵段，本地直接可开）
- 本地测试：`python3 -m http.server 8788` 然后开 `http://localhost:8788/index.html`
- 发布构建（剥掉哨兵段后发 artifact）：
  ```sh
  mkdir -p dist && sed -e '/DEV-SKELETON-START/,/DEV-SKELETON-END/d' \
    -e '/DEV-SKELETON-TAIL-START/,/DEV-SKELETON-TAIL-END/d' index.html > dist/index.html
  ```
  然后发布 `dist/index.html` 到上面那个 artifact URL（vendor 文件已在线上，没改就不用重传）。

## 调参位置

- 变形力度/半径：`buildOps()`（瘦脸 0.13、大眼 0.22、瘦鼻 0.14）
- 磨皮细节回贴量：FS_MAIN 里 `0.22`；边缘保护阈值 `smoothstep(0.09, 0.28, diff)`
- 滤镜：`FILTERS` 数组，六元组 = exposure(档)/contrast/saturation/temp/tint/fade
- 法令纹：遮罩带宽 `faceW*0.055`、横向偏移 1.5% 脸宽（buildMaskCanvas）；增益 `*1.35`、提亮超调 `blur*1.04`（FS_MAIN）
- 调试钩子（留着给未来调参）：`window.__ketuMask`（遮罩画布）、`__ketuRender`（同步渲染，绕开后台标签页 rAF 停摆）、`__ketuParams`
- 模型文件叫 `face_landmarker.task.wasm`：`.task` 不在 artifact 可服务扩展名里，借 `.wasm` 马甲，内容未动
