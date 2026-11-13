# GUI 与渲染问题定位

症状往往具有欺骗性：**「一片黑 / 一个字都不显示」在多数情况下不是字体或渲染阶段坏了，而是矩阵或缓冲状态错了**。先按本文件的顺序排除，再动渲染路径。

## 坐标系与矩阵（最高频根因）

- JOML 的 `new Matrix4f(...)` 构造参数是**列优先**（mCR：先第 0 列的四个分量）。按行优先写基向量，会把整块画布推到几十格外，表现就是全黑、无文字。正确顺序是：
  ```
  第 0 列 = 画布 X 轴 (right.x, right.y, right.z, 0)
  第 1 列 = 画布 Y 轴 (-up.x, -up.y, -up.z, 0)
  第 2 列 = 朝向   (-n.x, -n.y, -n.z, 0)
  第 3 列 = 原点   (ox, oy, oz, 1)
  ```
- 站在屏幕外侧看时，画布右手边是 `up × n`；写成 `n × up` 文字会镜像。第三列取 `-n` 可把行列式掰正，避免四边形被背面剔除。
- **方块实体渲染器（BER）的 PoseStack 已经是「相机相对 + 方块原点」**，直接用本地坐标乘画布矩阵即可，不要再手动减相机位置——减了就成了双倍相机偏移。
- **`RenderLevelStageEvent` 的 pose 是单位矩阵**（实测 m30/m31/m32 全 0），那条路要自己减相机。两条路不要混用。
- 画布平面与方块表面要留偏移（约 0.01～0.05 格）避免 z-fighting；太小会出现网格闪烁。

## 缓冲与 RenderType

- `RenderType.debugQuads()` 的 BufferBuilder 可能被别处回收，取到时抛 `IllegalStateException: Not building!`。
- 同一个 `MultiBufferSource` 里**交替画色块和文字**时，每次切换 RenderType 都会 `endBatch`，再取回的 builder 已失效，同样崩。
  - 稳定写法：分两趟。先把所有色块用一个 `ByteBufferBuilder` 画完并 flush，再逐条画文字（每条独立 flush）。用完 `close()`。
- 用 `RenderType.entityTranslucent(自定义贴图)` 画纯色块可能取到整张图集而非目标贴图，颜色会变成诡异的暗红。纯色块改用 `RenderType.lightning()`（position_color，顶点色即最终颜色）。
- 顶点元素要按 RenderType 的 vertex format 给全：带贴图的类型需要 uv / overlay / light / normal，缺一个就报 `Missing elements in vertex`；纯色类型只给 color。
- `Font.drawInBatch` 本身通常是好的。「文字不出来」优先怀疑矩阵、绕序、深度，而不是换 SEE_THROUGH、换 immediate BufferSource 或重置 shaderColor。

## GUI（Screen）

- `Screen.render()` 内部会**再次调用** `renderBackground()`。若在 `render()` 里又画一遍背景，就会出现面板模糊 + 压暗、按钮文字看不清。把所有自绘内容收敛到 `renderBackground` 中，`render()` 只留控件。
- 输入框黑块遮挡：`EditBox` 设 `setBordered(false)` 并显式设文字颜色。
- 按钮与面板同色会看不出边界，按原生按钮贴图的调色板画亮边/暗边做 3D 感。
- 顶层模态框：先给整块面板压一层半透明暗色遮罩，再用**不透明**底色画模态，并且最后绘制，避免下层槽位透出来。
- 页签切换时按页严格分支创建/销毁控件，否则会出现「输入框常驻」「切页画面不变」。

## 事件总线

- `EntityRenderersEvent.RegisterRenderers` 必须挂在 **MOD 总线**（`@EventBusSubscriber(bus = Bus.MOD)`）。该参数在新版本虽标记废弃，默认值仍可能是 GAME 总线；不写就静默不注册，渲染器看起来「完全没生效」。

## 逐面状态指示

- 多方块/逐面配置的状态点宜小（约面宽的 10%），大色块会糊住整个表面、看不出方块本体材质。
- 正面若是固定装饰面（舱门/呼吸灯），应固定不参与配置，也不在该面暴露连接能力。
