# paperroute 画风渲染 · 实现原理

> 这份文档讲的是 `house-lab.html`（画风试验台）里**每一层效果是怎么算出来的**，以及为什么必须这么算。
> 玩法侧的设计蓝图在 `LEARNING.md`；所有数值都能在页面上用滑杆实时验证，看数字用 `window.__lab.*`（见 §6）。

---

## 0. 怎么跑

单文件、零构建。`<head>` 里一个 importmap 把 `three` 指到 CDN（0.160），`<script type="module">` 里就是全部逻辑。

打开方式：本机 HTTP 服务，或宿主提供的 `local://workspace/paperroute/house-lab.html`。
**注意**：别双击（`file://`）打开。模块本身是从 CDN 拉的、没有相对模块导入，所以页面能起来；卡住的是 `gouache-atlas.png`——three 的 Loader 默认带 `crossOrigin = 'anonymous'`，`file://` 下的跨源图片请求会被拒，肌理加载不出来，画面退化成纯平色。

画面由**两个叠起来的 canvas** 组成：

```
┌─ #gl    (WebGL canvas)      ← three 渲染 3D 场景
├─ #paper (2D canvas, 叠在上面) ← 把 gl 画进来 + 叠加纸纹，再交给 DOM 显示
└─ #hud   (DOM)               ← 滑杆/下拉，绝对定位在最上层
```

为什么中间要夹一层 2D canvas：纸纹是**屏幕空间**的整幅后处理，用 CSS `mix-blend-mode` 做不出来（会和 HUD 一起被混），用 WebGL 后处理又得再写一个 pass；2D canvas 的 `globalCompositeOperation = 'screen' / 'multiply'` 一行就够了。代价是每帧一次 `drawImage`（2560×1720 的中转），以及**必须开 `preserveDrawingBuffer: true`**，否则 `drawImage` 抓到的是空帧。

主循环只做一件事：

```
needsDraw → renderer.render() → composite()   // 中间 canvas 画 gl，再叠纸纹
```

---

## 1. 总览：一层卡通底子 + 五层"手绘感"

```
                ┌──────────────── 3D 场景 ────────────────┐
几何 ──► MeshToonMaterial（补丁三处注入）
                │
                ├─(A) 卡通量化        4 档 gradientMap
                ├─(B) 色块层          抖动 / 交界起毛 / 交界印痕  ← C 键
                │      └─ 颜料厚薄 gPrThick ──► (C) 同色相浓淡    ← HSL 派生
                ├─(D) 水粉肌理图集     盒式投影采样             ← A 键
                └─(E) 描边壳           BackSide + 沿法线外推    ← O/W/线宽/分层
                ┌──────────────── 屏幕空间 ────────────────┐
                └─(F) 纸纹            两层 tile + 屏幕混合       ← P 键
配色（全局）：色相环旋转 + 统一饱和度/明度，连天空/雾/灯光一起走
```

关键取向：**(B)(C)(D)(E) 全部锚在世界坐标上**，只有 (F) 故意贴屏幕。这是整份文档里最重要的一条设计约束，下面会反复回到它。

---

## 2. 卡通底子：量化渐变

```js
const gradData = new Uint8Array([96, 158, 212, 255]);   // 4 档
gradientMap = DataTexture(..., NearestFilter, 无 mipmap)
```

`MeshToonMaterial` 的光照本来就查 `gradientMap`：把 `N·L` 映射到 0~1 再取样，于是明暗变成**四块平色**（这正是"漫画感"的来源，也是后面所有"色块"操作的舞台）。

采样只发生在一个函数里——`getGradientIrradiance(normal, lightDirection)`。补丁的全部工作就是在它前后动手脚：

```glsl
float dotNL = dot(normal, lightDirection);
vec2  coord = vec2(dotNL * 0.5 + 0.5, 0.0);   // ← ① 抖动这里
float g = texture2D(gradientMap, coord).r;     // ← ② 查表，g 是"这一档多亮"
return vec3(max(0.0, g * (1.0 - ink)));        // ← ③ 交界压深
```

---

## 3. 补丁挂在哪四个点

（前三个是画风补丁共用的注入点，第 4 个是水粉肌理图集单独加的。）

用 `material.onBeforeCompile` 往 three 的 shader 里塞代码，而不是自己写 `ShaderMaterial`——这样阴影、雾、色调映射、贴图编码这些"免费功能"全都还在。

| # | 注入点（three 的 chunk） | 塞什么 |
|---|---|---|
| 1 | vertex `#include <common>` 之后 | `varying vec3 vPrPos/vPrNormal` + `varying float vPrDist`，并在 `begin_vertex` 后写入 |
| 2 | fragment `#include <gradientmap_pars_fragment>` | 替换成 `GRADIENT_PARS`：噪声、量化改写、`gPrThick`、HSL 工具 |
| 3 | fragment `#include <opaque_fragment>` | 追加 `PIGMENT_MIX`：按 `gPrThick` 把当前像素色派生浓淡 |
| 4 | fragment `#include <map_fragment>` | 追加 `ATLAS_MIX`：盒式投影采样水粉图集并乘进 `diffuseColor` |

顶点阶段塞进去的三样东西：

```glsl
vec4 prWorld = modelMatrix * vec4(transformed, 1.0);
vPrPos    = prWorld.xyz;
vPrNormal = normalize(mat3(modelMatrix) * normal);   // ← 世界法线，不是视图法线
vPrDist   = -(viewMatrix * prWorld).z;
```

描边材质也在 `begin_vertex` 之后改 `transformed`（沿法线外推，见 §4.6），但它**只注入这一处**，不注入上面那三个 varying——它是 `MeshBasicMaterial`，本来也不需要它们。

---

## 4. 逐层原理

### 4.1 世界坐标基准：为什么不能贴屏幕

第一版把噪声（`gl_FragCoord`）和法线（`vNormal`）都用屏幕/视图空间，结果转视角时**纹理不动**——像隔着一层有纹理的玻璃看游戏。判定标准很简单：

> 相机绕着物体转一圈，物体表面的斑点必须跟着物体转。做不到就说明锚错了空间。

所以：噪声输入 `vPrPos`（世界坐标）、投影面选轴 `vPrNormal`（世界法线）。附带好处是纹理在物体上"长着"，缩放镜头也不会变形。

`vPrDist` 用来做**远处高频淡出**：

```glsl
float atten = 1.0 / (1.0 + vPrDist * vPrDist * 0.0022);
```

细笔触（`PR_BODY_C`）和交界毛边（`PR_HIGH_SCALE`）乘 `atten`，近处清楚、远处自动收干净——不然远处亚像素噪声会闪成沙。

### 4.2 色块抖动 & 交界起毛

量化是"硬"的，所以四档色块的边界是几何上的等值线，看起来死板。办法是在**查表之前**把 `coord.x` 推一下：

```glsl
coord.x += (nLow  - 0.5) * uLowAmp  * 2.0                       // 整片边界一起来回抖
         + (nHigh - 0.5) * uHighAmp * 2.0 * (0.45 + 0.55 * atten); // 交界处的细碎毛边
```

五个噪声（`prNoise3`，值噪声 + 平滑插值）分工明确，频率单位是 1/世界米：

| 频率常量 | 值 | 干什么 |
|---|---|---|
| `PR_BODY_A` | 1.05 | 面级厚薄（大块颜料） |
| `PR_BODY_B` | 3.30 | 局部块状起伏 |
| `PR_BODY_C` | `vec3(8.3, 23.0, 8.3)` | 细笔触，y 方向频率高 → 横向短笔 |
| `PR_LOW_SCALE` | 1.54 | 色块边界抖动 |
| `PR_HIGH_SCALE` | 10.60 | 交界毛边 |

### 4.3 交界印痕：必须"按比例"压

想要"颜料在交界线上堆了一道"的效果，直觉是 `g -= thickness`。**这会毁掉暗面**：暗面本来 `g` 就小（比如 0.15），减 0.1 等于砍掉三分之二 → 一条黑线。

正确做法是按比例乘：

```glsl
float tq = coord.x * 4.0;                              // 4 档 → 每档宽度 1
float dEdge = abs(tq - floor(tq + 0.5));               // 到"档边界"的距离，0 就在交界上
float ink = uInk * pow(max(0.0, 1.0 - dEdge * 3.2), 2.0);
return vec3(max(0.0, g * (1.0 - ink)));                // ← 乘，不是减
```

暗面被压 10%、亮面也被压 10%，交界线才是"一道印痕"而不是"一条黑缝"。

### 4.4 颜料厚薄 → 同色相浓淡

`getGradientIrradiance` 只输出**一个标量**（浓淡权重），颜色不在这里决定：

```glsl
float body = (nBodyA - 0.5) * uBodyAmp * 0.75
           + (nBodyB - 0.5) * uBodyAmp
           + (nBodyC - 0.5) * uBodyAmp * 0.45 * atten;
gPrThick = clamp(body * 0.5 + 0.5, 0.0, 1.0);          // 0=薄透, 0.5=本色, 1=浓重
```

真正上色在 `PIGMENT_MIX`（替换 `opaque_fragment`）里：**取这一面当前的颜色**，转 HSL，只动 S/L、**色相严格不动**，再转回来与本色按权重混合。

```glsl
vec3 hsl   = prRgb2Hsl( prLin2Srgb( gl_FragColor.rgb ) );   // 此刻还在线性空间，先转 sRGB 再进 HSL
vec3 heavy = vec3(hsl.x, hsl.y * 1.14, hsl.z * 0.92);       // 浓：饱和 +14%、明度 −8%
vec3 thin  = vec3(hsl.x, hsl.y * 0.86, hsl.z * 1.08);       // 薄：饱和 −14%、明度 +8%
gl_FragColor.rgb = prSrgb2Lin( mix(hsl2rgb(thin), hsl2rgb(heavy), gPrThick) );
```

三个不能动的点：

1. **必须走 HSL。** 直接 `color.rgb *= vec3(0.9, 1.1, 0.9)` 或缩放饱和度，色相会跟着漂：绿色偏黄偏褐，看着就是"跑色"。
2. **"浓"靠加饱和度、不是靠变暗。** 只压亮度的话，浓的地方就是暗斑，暗面上直接糊成黑。
3. **两色对称**（围绕本色各偏 ±14%），所以权重 0.5 = 本色；滑杆归零画面完全回到原样，不留痕迹。

### 4.5 水粉肌理图集

素材是 `gouache-atlas.png`：一张 3×3 的手绘水粉。加载时**切成 9 张独立 512×512 纹理**（各自 mipmap，远处不会串格到隔壁）。

三件关键处理：

**① 对比拉伸（围绕每格自己的均值）**
手绘水粉本身对比度极低（每格 sd 只有 3.6~13），直接乘上去肉眼看不见。围绕均值拉 1.8 倍：

```js
d[i] = (d[i] - meanR) * 1.8 + meanR;
```

**均值不变**，所以第 ② 步的归一化照样成立。

**② 均值必须在 linear 空间求，而且要在 shader 里除掉**
纹理会做 sRGB→linear 解码（`CanvasTexture` + `colorSpace = SRGBColorSpace`），所以 shader 里：

```glsl
vec3 gouache = texture2D(uPrAtlas, auv).rgb / max(vec3(0.02), uPrAtlasMean);
diffuseColor.rgb *= mix(vec3(1.0), gouache, uPrAtlasK);
```

`texture/mean` 的均值是 1 → 肌理只贡献**起伏**，不改整体明暗。如果跳过这一步，等于给物体乘上一个 0.6 左右的灰度，整片被压暗（而且在 sRGB 上求的均值对不上 linear 解码的结果，误差还会跑偏）。

**③ 盒式投影，选轴用世界法线**

```glsl
vec3 an = abs(normalize(vPrNormal));
if (an.x >= an.y && an.x >= an.z) auv = vec2(vPrPos.z, vPrPos.y);
else if (an.y >= an.z)            auv = vec2(vPrPos.x, vPrPos.z);
else                              auv = vec2(vPrPos.x, vPrPos.y);
auv *= uPrAtlasScale;
```

三平面投影，按世界法线的主轴选面。**用 `vNormal`（视图空间）会怎样**：相机一转，同一个面的"主轴"就换一个，投影 UV 直接跳变 → 又变回"贴在玻璃上"。这是一个只改了一半的 bug，症状隐蔽（静止看没错，一转动就发现纹理在滑）。

**九格身份**（看图确认，别靠猜）：

| 索引 | 内容 | 索引 | 内容 |
|---|---|---|---|
| 0 | 竖纹木板 | 5 | 帆布/麻布 |
| 1 | 横向壁板（板条线） | 6 | 奶油色水粉 |
| 2 | 灰泥/水渍墙 | 7 | 平滑粉刷墙 |
| 3 | 深灰抹灰 | 8 | 绿色笔触（草/叶） |
| 4 | 屋瓦 shingle | | |

**自动分配**：`atlasCellFor(mat)` 按材质本色的 HSL 判断——绿→8、红→4、深棕→0、蓝灰→7、浅色→1、其余→2。HUD 也能全场统一指定某一格，还能点九宫格缩略图直接选。

**密度乘数**：草 3.2×（要密得像一簇簇草叶）、木 1.5×、瓦 1.1×，基准 `atlasScale = 0.55` 格/米。

### 4.6 描边（含"屏幕等宽"与分层）

**几何**：同一份几何体再来一个 `MeshBasicMaterial({ side: BackSide })`，颜色不用纯黑，而是深褐 `0x40302c`。

**宽度怎么做**——两代方案，第一代是错的，值得记：

| | 旧：整体放大一份 | 现：沿法线外推 |
|---|---|---|
| 顶点位移 | `scale = 1.045`（绕原点缩放） | `transformed += normal * uWidth` |
| 长条物体（栅栏横杆） | **两头伸出去一大坨、侧面没线**（伸出的量 ∝ 到中心的距离） | 处处等宽 ✓ |
| 几何体要求 | 必须居中 | 无要求 ✓ |
| 屏幕等宽 | 难算 | 直接可算 ✓ |

```glsl
// 描边材质的 vertex
transformed += normal * ( uWidth + (prNoise3(position * uWobFreq + uWobSeed) - 0.5) * uWobble * uWobScale );
//                         ↑ 固定宽度                    ↑ 勾线起毛（W 键），幅度 ≈ 一个线宽
```

**屏幕等宽（距离补偿）**——每帧在 `updateOutlines()` 里换算：

```
屏幕像素宽 = 世界宽 / 距离 × 焦距        （焦距 = (H_dev/2) / tan(fov/2)，fov 38° 时 1720 高的窗口 ≈ 2498 设备像素）
⇒ 世界宽 = 目标像素宽 × 距离 / 焦距
⇒ 局部宽 = 世界宽 / 物体的累积缩放
```

于是目标线宽用 **CSS 像素**表达（滑杆 0~4，默认 1.4），近处远处**屏幕上线一样粗**。验证：整帧"墨线像素占比"随相机距离只按轮廓周长（∝1/距离）缩小；旧方案是世界等宽，占比按 ∝1/距离² 缩，远处线会细到消失。

**分层**：建模时每段给一个类别标签（`CAT`），描边宽度再乘该类别的倍率：

| 类别 | 归属 |
|---|---|
| `house` | 墙、屋顶、烟囱 |
| `trim` | 门、把手、窗、台阶 |
| `prop` | 栅栏、邮筒 |
| `plant` | 树干、树冠、灌木 |
| `cloud` | 云 |

手绘里常见"小物重勾线、大面轻勾线"，这个层次现在可以分别调。

**边界条件**：线宽调到 0 时**直接隐藏描边**，不是把宽度设成 0——宽度 0 时两层壳完全重合，会闪 z-fighting。同理 `O` 键（描边总开关）关掉时，`W`（勾线起毛）看不出任何变化：起毛作用在描边壳上，**描边是它的前提**，但两者是独立开关。

### 4.7 纸纹（屏幕空间，故意贴屏）

生成两张 256×256 的 tile：

```js
const n = (rnd() - 0.5) * 34 + (rnd() - 0.5) * 20 + fibre;   // 居中在 0：细粒 + 粗粒 + 纸纤维
v_screen   = 34  + n;    // 中值 34（接近黑）→ 只提亮
v_multiply = 221 - n;    // 中值 221（接近白）→ 只压暗
```

合成时用同一个 `n` 一正一反：

```js
pctx.globalCompositeOperation = 'screen';   fillRect(整幅);   // 提亮
pctx.globalCompositeOperation = 'multiply'; fillRect(整幅);   // 压暗
```

**为什么成对出现**：单向的 soft-light/overlay 在暗部会结成一坨黑斑（暗部本来就接近 0，再压就是死黑），一正一反两条曲线在暗部互相抵消，只在高频上留下颗粒。

**为什么贴屏幕不贴世界**：这是唯一"故意"的一层——画纸是不动的，颗粒应该像在屏幕上，而不是随镜头漂。代价是它不随物体缩放（可以接受，因为它是"纸张"而不是"材质"）。

**性能**：两张预生成 pattern + 两次 `fillRect`。千万别每帧做全屏逐像素软光，会吃满主线程直接卡死。

### 4.8 自动配色

思路：**不动各个物体的"角色"**（草是绿的、屋顶是红棕的），只把整套颜色沿色相环整体旋转 + 统一调饱和度/明度。相对色相不变 → 转到哪儿都协调。

```js
registerTint(mat)  // 创建时记下原始 HSL（当前 114 个材质）
applyTint(hs, sm, lm) {
  tintList.forEach(e => e.mat.color.setHSL((e.h + hs + 1) % 1, clamp(e.s * sm), clamp(e.l * lm)));
  paintSky(hs, sm, lm);                       // 天空渐变重画
  scene.fog.color = tintOf(FOG_BASE, ...);    // 雾
  sun.color = ...; amb.color = ...; fill.color = ...;   // 三盏灯
}
```

两个细节：**从原始值重算**（不是在上一次结果上再乘），所以反复切换不会漂移累积；**连天空/雾/灯光一起走**，否则物体会偏色而环境不变，看起来像蒙了一层滤色片。

HUD 里 6 套方案 + 一个"随机和声"（随便转色相、配比不动，所以怎么转都不脏）。

---

## 5. 参数与开关速查

| 键 | 作用 | 默认 |
|---|---|---|
| `R` | 复位视角 | θ=0.72, φ=1.38, r=9.2 |
| `C` | 光照色块层（厚薄/抖动/交界起毛/交界印痕） | 开 |
| `A` | 水粉肌理图集 | 开 |
| `W` | 勾线起毛 | 开 |
| `O` | 描边总开关 | 开 |
| `P` | 纸纹 | 开 |

| 滑杆 | 参数 | 范围 / 默认 |
|---|---|---|
| 颜料厚薄 / 色块抖动 / 交界起毛 / 交界印痕 | `bodyAmp` `lowAmp` `highAmp` `ink` | 0~1.2 / 0~0.4 / 0~0.3 / 0~0.5，默认 0.35 / 0.2 / 0.1 / 0.18 |
| 勾线起毛 | `wobble` | 0~3，默认 1（幅度 ≈ 一个线宽） |
| 描边线宽 | `outlinePx` | 0~4 **CSS 像素**，默认 1.4 |
| 描边分层 | `catMul[house/trim/prop/plant/cloud]` | 各 0~3，默认 1 |
| 纸纹 | `grain` | 0~0.7，默认 0.28 |
| 水粉肌理 | `atlas` / `atlasScale` / `atlasCell` | 0~1.6（默认 0.75）/ 0.55 格每米 / −1=按本色自动 |
| 配色 | `SCHEMES` + 随机和声 | 晴日原味 |

---

## 6. 怎么"看见"看不见的东西

调试口（`window.__lab`）：

```js
__lab.info()            // three 版本 / 三角面 / draw call / program / 尺寸 / 全部参数 / 错误
__lab.renderNow()       // 强制渲染一帧
__lab.sample(x, y)      // 取一个像素（设备像素坐标）
__lab.patch(x, y, w, h) // 取一块区域统计
__lab.setView(θ, φ, r)  // 摆机位
__lab.setParam(k, v) / setPerturb / setAtlasOn / setWobbleOn / setOutlinePx / setCatMul / setGrain / setOutline
__lab.setScheme(name) / setTint(hs, sm, lm) / tintCount() / atlasInfo() / setAtlasCell(i)
__lab.updateOutlines()  // 手动重算描边宽度（相机/缩放变了才会自动算）
```

**验证一层效果的通用手法**（同一机位、同区域、开关那一层）：

| 想确认 | 看什么 | 判据 |
|---|---|---|
| 色相有没有漂 | 区域内像素转 HSL 求 H 均值 | 开关该层时 H 变化应 < 0.3° |
| 会不会发黑 | meanLum / 暗部像素数（L<50）/ 最暗值 | 不该增加、不该下降 |
| 这层有没有用 | sd（局部对比度） | 开关前后差多少 = 这层的贡献 |
| 描边粗细 | 整帧"墨线像素"数（墨色 ≈ 亮度 52，取 L<70） | 线性可加 |

**踩过的坑**：

- **webview 截图抓不到 WebGL 图层** → 必须先 `drawImage` 到 2D 中转 canvas（`preserveDrawingBuffer: true`），纸纹也是叠在这张中转上的。
- **拿截图和 canvas 直接逐块对比会误判**：截图里含 DOM HUD，比出来"整幅画都变了"。避开 HUD 覆盖区再比。
- **指标口径要跟状态一起锁**：`L<70` 这种阈值会被纸纹的提亮层污染（墨线从 52 被抬到 70 以上）。关掉纸纹再量。
- **改测量区域前先确认窗口尺寸**：窗口中途改过没恢复，按 2560 算的区域全错位，得出"参数完全一样"的假结论。
- **`let`/`const` 的 TDZ**：初始化序列里早于声明就被调用的变量（`placeCamera()` 早于它下面定义的函数）会让整个 module 静默不跑。跨节引用的辅助对象要么提到最前，要么在函数内部新建。
- **`read_file` 单次上限 1000 行**，这个文件 1100+ 行 → 永远"部分读取" → `file_edit` / `write_file` / `replace_line` 全被拒。改它只能用 `sed`（BSD sed：换行写 `\n`，替换串里的 `&` 写 `\&`，`a\` 后面接真换行；改完一律 grep 复核）。

---

## 7. 性能与结构预算

当前（1280×860 窗口）：**14216 三角面 / 102 draw call / 4 program**（这是当前配置/机位下的读数；默认视角 + 默认参数是 13520 / 92 / 4 —— 视锥剔除会让物件进出视野，数会跟着变）。

- draw call 里 **每件物体占 2 个**（本体 + 描边壳，69 个描边网格）。这是最明显的优化点：描边可以和本体合并成 InstancedMesh，或用一个"法线外推"的 shader 变体走同一批材质。
- 纸纹是两次全屏 `fillRect`，跟分辨率线性相关，可忽略。
- 图集 9 张 512² + mipmap ≈ 十几 MB 显存，可接受；如果要上移动端，可以降到 256²。
- 噪声全是值噪声（每像素 5~8 次 8 点插值），在移动端是主要开销；`atten` 把远处的高频关掉就是在省这部分。

---

## 8. 往游戏里搬的时候

- **坐标系先定死**：玩法需要"沿街道的里程 s + 横向偏移 d"这套路网坐标（见 `LEARNING.md`），而画风现在的噪声/肌理用的是世界坐标。两者不冲突——物体按 (s, d) 摆位，纹理仍按世界坐标采样，天然连成一片、跨物体不断开。
- **物体生成**：房子/栅栏/邮箱都走 `piece(geo, mat, { pos, rot, cat })` 这条路（自动带描边 + 自动配色登记 + 自动分配肌理格）。描边宽度**不再**由建件时写死的系数决定，而是 `outlinePx` × 该类别倍率（§4.6 的"屏幕等宽 + 分层"）。程序化生成时只改几何参数，画风层不用动。
- **还没定的画风项**：草地肌理现在是看得见的规律编织纹（`C_LEAF` 3.2× → 0.57 m 一个循环），要不要改密度/换成"草簇"图；色块层（C）要不要救（拉 `bodyAmp` 到 1.0~1.2 看极限，或退成只留轻微斑驳 + 交界印痕）；描边线宽与分层的最终档位。

---

## 9. 参考点速查

| 主题 | 位置（按小节名搜） |
|---|---|
| 可调参数 `P` / 共享 uniform `U` | `可调参数` |
| 量化表与 `GRADIENT_PARS` | `toon 材质 + 画风补丁` |
| `ATLAS_MIX` / 图集加载 / 九格分配 | `水粉肌理图集 gouache-atlas.png` |
| 描边材质 / `piece()` / 类别 `CAT` | `makeOutlineMat` / `piece` / `描边类别` |
| 纸纹 tile 生成与合成 | `纸纹颗粒` / `composite()` |
| `updateOutlines()` 屏幕等宽 | `描边宽度：目标屏幕线宽` |
| 自动配色 | `自动配色` |
| 调试口 | `调试口（给 agent 采样用）` |
