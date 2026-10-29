[English](README.md) | **简体中文**

\# illustrator Master-Copyist



> \*\*一句话让 AI 在 Illustrator 里复刻任何图。\*\*

>

> 可能是目前最强的开源免费、Agent 主导的 Adobe Illustrator 复刻 Skill。



\*\*illustrator Master-Copyist\*\* 是一个面向科研示意图的「栅格 → 可编辑矢量」重建技能包。它把一张 PNG/JPG 拆成 \*\*可编辑活字文本层、独立线稿层、独立上色/几何层、可删除背景层\*\*，全部落在 Adobe Illustrator 原生对象上——不是描边字，不是位图，不是白块盖字。





\---



\## 目录



\- \[它是什么](#它是什么)

\- \[它能生成什么](#它能生成什么)

\- \[它能做什么](#它能做什么)

\- \[核心特性](#核心特性)

\- \[环境依赖](#环境依赖)

\- \[快速开始](#快速开始)

\- \[工作流程与原理](#工作流程与原理)

&#x20; - \[路线 A：栅格重建（Raster Reconstruction）](#路线-a栅格重建raster-reconstruction)

&#x20; - \[路线 B：线稿重建（AniLines 一笔几何）](#路线-b线稿重建anilines-一笔几何)

&#x20; - \[填充边界与背景（强制）](#填充边界与背景强制)

&#x20; - \[交付与验收](#交付与验收)

\- \[工具包 toolkit](#工具包-toolkit)

\- \[自迭代机制](#自迭代机制)

\- \[实测数据与性能](#实测数据与性能)

\- \[已知限制与披露](#已知限制与披露)

\- \[目录结构](#目录结构)

\- \[故障排查速查](#故障排查速查)

\- \[致谢](#致谢)

\- \[许可](#许可)



\---



\## 它是什么



`illustrator Master-Copyist` 是一个 \*\*Agent 可直接加载的 Skill\*\*，同时支持 Codex、Claude Code、OpenCode。它不依赖云端矢量化 API，核心脚本全部本地运行，自带 Python 运行时、VTracer 可执行文件、AniLines 线稿模型权重，通过共享的 Windows/macOS CLI 与 Adobe Illustrator 原生通信。



它解决的核心问题是：



> \*\*科研图不能只“看起来像”，必须“还能改”。\*\*



传统矢量化工具输出的是死路径：文字变轮廓、图层糊成一团、背景烘焙进画面，投稿返修时无法改一个字。本 Skill 的路线是 \*\*「文字与图形分离」\*\*：



1\. 先把每个标签转写成结构化 manifest（真实字体、字号、基线、颜色、锚点）；

2\. 只从副本中擦掉文字，保留图形；

3\. 对剩余图形做保真描摹；

4\. 把真实 `<text>` 按原基线注回；

5\. 在 Illustrator 里生成原生 TextFrame + 矢量路径；

6\. 最后叠加独立 `AniLines lineart` 线稿层，并做线稿定界、透明底、可删背景层。



它明确 \*\*不适用\*\* 于：用描摹恢复实验数据、把统计图当示意图重画、对每张上传图片无差别套用本流程。统计图应使用原始矢量导出与真实数据；描摹和颜色量化不能恢复数值测量。



\---



\## 它能生成什么



交付的 `.ai` 文档图层结构（自上而下）：



| 图层 | 内容 | 说明 |

| --- | --- | --- |

| `AniLines lineart` | 独立一笔线稿层 | AniLines 神经网络提取 + 字形级文字过滤 + 一笔几何装配，可单独开关/改色/删除 |

| `Editable text` | 独立活字层 | 每个标签是原生 TextFrame，字体、字号、颜色、基线可编辑 |

| `图层 1`（trace geometry） | 独立上色/几何层 | VTracer 保真描摹的矢量路径；SVG 导入后常合并为一个 GroupItem，`pathItems` 仍可数到全部轮廓 |

| `Background` | 可删背景层 | 源图页面色的单个矩形；隐藏或删除后画板全透明 |



同时产出：



\- `figure.ai` — Illustrator 原生文档

\- `figure.svg` — 结构化 SVG（path + 真实 `<text>`，无位图/外链/脚本）

\- `figure.png` — 白底预览（200% 导出）

\- `figure\_nobg.png` — 透明底预览

\- `qa.json` / `candidate\_comparison.csv` / `playback.json` — 审计记录

\- `lineart\_strokes.json` — 一笔线稿几何，可在 Illustrator 里脚本重绘

\- `text\_manifest.json` — 逐标签文字规格



\*\*自带抠图能力\*\*：`fill\_matte.py` 按线稿几何生成 alpha 遮罩，`vectorize\_cut.py` 保留 alpha 直接喂 VTracer，透明区域不产生路径，页面背景按构造消失。`verify\_fill\_bounds.py` 检查溢出、背景色差、四角 alpha、页色残留；`visual\_check\_bg.py` 在黑底与洋红底上做双底色目视验收。



\---



\## 它能做什么



1\. \*\*将 AI 生成的图片转换成矢量图\*\*，便于二次编辑、排版与投稿。

2\. \*\*将网上公开图片转成无背景矢量图\*\*，用于素材收藏、临摹与结构参考（注意原图版权）。

3\. \*\*复刻论文/综述机制图\*\*：多面板、箭头、细胞、受体、通路、色块芯片、徽标数字，全部落成可编辑对象。

4\. \*\*重建已有 SVG\*\*：校验、准备、注入活字、原生导入，保留原矢量结构。

5\. \*\*从零绘制新示意图\*\*：给定明确内容，直接写结构化 SVG（path + 基本形状 + 真实 `<text>`，箭头用 polygon），再走原生绘制。

6\. \*\*线稿单独交付\*\*：AniLines 提取一笔线稿，作为独立图层叠加到已有 `figure.ai`，不破坏原 artwork。

7\. \*\*透明底/换底交付\*\*：线稿定界 + 可删 Background 层，图形可直接 key 到任意背景。

8\. \*\*批量/可恢复绘制\*\*：调度器分批写入、定期保存 job 自有 `figure.ai`、导出 PNG，中断后按完整路径恢复。



\---



\## 核心特性



\### 1. 文字与图形彻底分离



\- 标签走 manifest / live-text 路径，绝不把字母描成轮廓再用白块盖住。

\- 线稿去字用 \*\*字形级连通域过滤\*\*（`lineart\_text\_filter.py`）：只删 bbox 完全落在文字墨迹框内的连通域，跨框的引线、边框、细胞轮廓一律保留。\*\*从不做整框平填擦除\*\*——那会损坏穿过文字框的图形。

\- 擦除阶段有 \*\*残字闭环\*\*：擦除 → 扫描残字簇 → 并框/归位 → 重擦 → 复扫，直到只剩图形碎片。五轮实测把 03\_1 从 11 轮迭代降到 3 轮。



\### 2. 线稿是填充边界的唯一权威



\- \*\*不按颜色判断背景\*\*。页面可能是渐变、照片底、米色、粉色；取样页色只是逐图便利手段，不是规则。

\- `fill\_matte.py` 二值化线稿 → 方向性闭运算封 1–3 px 发丝缝 → 从画布四边 flood 判外部 → 写出 RGBA，线稿围出的图形外 alpha=0。

\- 方向性闭运算很关键：各向同性核大到能封画框角，也会封掉面板间 5–10 px 白沟，整页变不透明。

\- `restore\_ink\_alpha.py` 必跑：虚线分隔、虚线标注框、开口轮廓围不成闭环，会被 flood 判成外部而整条消失；按去字线稿把墨迹处 alpha 回写 255。033 实测救回 52,853 px。



\### 3. 原生 TextFrame，不是描边字



\- 每个标签是 Illustrator 原生文本框，字体、字号、颜色、基线、锚点可改。

\- 字距恒为 0。负字距会让 Illustrator 同时用 JSX runtime 和 SVG 导入双渲染同一文本框（双影）。宽度不匹配时按 `fs × 目标宽 / 渲染宽` 重标字号，绝不用负 tracking。

\- 字体选择区分 regular/bold 与 Black/Light；缺字体在准备阶段停止，不会把错误字体写进文档。

\- 度量口径用 PIL 实测（含 kerning），不用 fontTools pen 累进（不含 kerning，会系统性偏窄 7–8%）。



\### 4. 独立线稿层（AniLines）



\- 内置 AniLines（MIT，gyrojeff/AniLines）basic/detail 双模型，权重各 69 MB，随包 vendored 在 `third\_party/anilines/`。

\- 默认 `--scale 2` 细节模式：输入先 Lanczos 2× 上采样再推理，细线不被网络抹掉。实测笔画数 778 → 1400，闭环 97 → 286，中位线宽 2.00 → 1.41 px。

\- 一笔几何装配：骨架化 → Rutovitz 交叉数判节点 → 节点聚类 → 链提取 → 切向连续合并；闭环保持 `closed=true`，“一圆一笔”由构造保证。

\- 墨迹取色修正：骨架常偏离墨迹 1–2 px，逐点取 5×5（回退 9×9）窗口最暗像素、按笔画取中位；最暗仍 ≥250 的笔画视为无墨跳过，不画成页面色（避免白线在白底消失、在深底刺眼）。



\### 5. 保 alpha 描摹 + 可删背景



\- `vectorize\_cut.py` 直喂 RGBA 给 pinned VTracer CLI，参数与 faithful 档逐项一致（4× LANCZOS、spline、speckle 4、precision 8、gradient-step 4、simplify 0.6、128 色），再 `scale(0.25)` 回原尺寸、恢复 viewBox。透明区不产生路径。

\- `import\_to\_illustrator.py --bg-color` 在最底层建可删 `Background` 层；`--png-nobg` 另导透明底。

\- `verify\_fill\_bounds.py` 四项判据 + `visual\_check\_bg.py` 双底色目视，缺一不可。



\### 6. 自迭代机制



每轮任务结束自动复盘高价值可复用脚本，归类归档到 `toolkit/`，并更新 `toolkit/调用指引手册.md`。越用越强、越快、越便宜。



\### 7. 完全本地、无矢量化 API Key



核心脚本不上传图片、不要求矢量化 API key。依赖安装是独立显式步骤，不是每图下载。模型与 Adobe 访问由用户负责；把图片给云端模型看并非完全离线。



\---



\## 环境依赖



\### 运行时（随包自带）



| 项 | 说明 |

| --- | --- |

| Python | \*\*Skill 自带 `.venv`\*\*：portable CPython 3.13.14，full-install 布局，无需外部 Python，任意工作目录可用 |

| Windows 解释器 | `.venv/Scripts/python.exe` |

| macOS 解释器 | `.venv/bin/python` |

| CLI 入口 | `skill\_dir/scripts/figure\_cli.py` |

| 调用前缀 | 所有命令加 `-X utf8`；路径含空格时引号包裹；PowerShell 引号路径前加 `\&` |



> 发布包通常直接附带 `.venv`。若从源码分发且未附带，运行一次 `python -X utf8 scripts/setup\_environment.py --with-vtracer` 创建 `.venv` 并按 `requirements.lock` 安装。\*\*不要\*\*往 `.venv` 里 pip install，也不要用系统 Python；新机器缺包时从另一台机器的 `.venv/Lib/site-packages` 连同 `.dist-info` 一起复制。



\### Python 包分层



| 层 | 包 | 使用方 |

| --- | --- | --- |

| Core | Pillow、numpy、defusedxml、fontTools、opencv-python-headless (cv2)、pip | figure\_cli / figurelib / toolkit stages 01–05 |

| Line-art (AniLines) | \*\*torch 2.14.1+cpu\*\*、scipy、scikit-image、networkx、sympy、mpmath、jinja2、markupsafe、fsspec、filelock、imageio、tifffile、lazy-loader、packaging、typing-extensions | `scripts/lineart\_\*.py` stages 1–3（extract / text\_filter / strokes） |

| Bundled 非 Python | `bin/vtracer.exe`（VTracer 1.0.0-alpha.4，pinned SHA256）、`third\_party/anilines/`（代码 + 双 69 MB 权重） | 矢量化后端、AniLines 模型 |

| External（用户侧） | Adobe Illustrator 2026 + COM bridge（PowerShell 5.1）/ macOS AppleScript | `lineart\_overlay.py native`、原生绘制、交付 |



\### Adobe 兼容性



| 环境 | 实现 | 证据 |

| --- | --- | --- |

| Windows + Illustrator 27.2.0 | 原生路径、活字、独立文本层、AI 保存、PNG 导出、恢复 | 真机测试，含用户目视验收的论文图复刻 |

| Windows 其他 Illustrator 版本 | 枚举注册的 COM 版本，逐项测试所需能力 | 未逐版本测试 |

| macOS Intel / Apple silicon | 发现运行中的 app bundle，AppleScript + 共享 JSX；双架构 pinned VTracer | 适配器契约检查；尚无 macOS 真机结果 |

| Linux | SVG 准备、文字清理、描摹、QA | 无原生 Illustrator 连接 |



本机实测 Illustrator 路径（2026-10-29）：



```text

D:\\Program Files (x86)\\Adobe IIIustrator\\Adobe Illustrator 2026\\Support Files\\Contents\\Windows\\Illustrator.exe

```



目录名 `IIIustrator` 是 \*\*三个 i\*\*（Adobe 安装包自身拼写），按 `Illustrator` 搜不到。版本 30.0.0，ProgID `Illustrator.Application`。



macOS 首次连接可能需要在「系统设置 → 隐私与安全性 → 自动化」给终端/Agent 授权；适配器只选已运行的 bundle，不会另拉一个版本。



\### 检查命令



```bash

<python> -X utf8 <cli> doctor

<python> -X utf8 <cli> illustrator-doctor

```



`illustrator-doctor` 报告实际版本、活动文档、画板坐标与所需能力。新系统/新 Adobe 版本先跑一次可丢弃文档测试：



```bash

<python> -X utf8 <cli> native-test --create-document --out-dir <new-test-directory>

```



\---



\## 快速开始



\### 0. 放置 Skill（按 Agent 选一个目录）



| Agent | 项目目录 | 调用 |

| --- | --- | --- |

| Codex | `<project>/.agents/skills/illustrator-Master Copyist/` | `$illustrator-Master Copyist` 或显式工作流请求 |

| Claude Code | `<project>/.claude/skills/illustrator-Master Copyist/` | `/illustrator-Master Copyist` |

| OpenCode | `<project>/.opencode/skills/illustrator-Master Copyist/` | 加载命名 skill；也支持项目 `.agents/skills` / `.claude/skills` |



每个 Agent 只选一个发现位置，避免重复条目。脚本不会改 Agent 设置、不写 hooks/MCP、不注册全局 skill。



\### 1. 环境自检



```bash

<python> -X utf8 <cli> doctor

<python> -X utf8 <cli> illustrator-doctor

```



\### 2. 选输入路线



\- \*\*新示意图\*\*：写结构化 SVG（path + 基本形状 + 真实 `<text>`，箭头用 polygon）。

\- \*\*已有矢量 SVG\*\*：直接 `validate` + `prepare`。

\- \*\*栅格复刻\*\*：走路线 A，先读 `references/workflow.md` 与 `references/input-and-validation.md`。

\- \*\*线稿重建\*\*：走路线 B，读 `references/lineart-pipeline.md`。



\### 3. 最小栅格复刻命令链



```bash

\# 1) 转写 + 量测 + manifest（toolkit Stage 1–2）

<python> -X utf8 toolkit/01\_detect/detect\_text\_lines.py --image <orig.png> --out-dir <work>/lines --overlay

<python> -X utf8 toolkit/02\_manifest/local\_match.py --image <orig.png> --spec <work>/spec.json --out <work>/located.json --overlay <work>/locate\_check.png

<python> -X utf8 toolkit/02\_manifest/grow\_boxes.py --image <orig.png> --in <work>/located.json --out <work>/grown.json --gap 12

<python> -X utf8 toolkit/02\_manifest/fit\_sizes.py --in <work>/grown.json --out <work>/text\_manifest.internal.json --colors <work>/colors.json

<python> -X utf8 toolkit/02\_manifest/conv\_manifest.py --in <work>/text\_manifest.internal.json --out <work>/text\_manifest.json --image <orig.png>

<python> -X utf8 toolkit/02\_manifest/verify\_text\_fit.py --manifest <work>/text\_manifest.json --out <work>/fit\_report.json --tol 2.5



\# 2) 擦字 + 残字闭环（toolkit Stage 3）

<python> -X utf8 toolkit/03\_erase/erase05.py --source <orig.png> --manifest <work>/text\_manifest.json --out <work>/text\_clean.png --passes 2

<python> -X utf8 toolkit/03\_erase/scan\_remnants.py --image <work>/text\_clean.png --manifest <work>/text\_manifest.json --apply

\# 有归位 → 重跑 erase05 → 再 scan，直到只剩图形碎片



\# 3) 线稿提取 + 字形级去字（skill lineart 管线）

<python> -X utf8 scripts/lineart\_extract.py <orig.png> <work>/la\_raw.png --mode detail

<python> -X utf8 scripts/lineart\_text\_filter.py --lineart <work>/la\_raw.png --text-manifest <work>/text\_manifest.json --measure-from <orig.png> --out <work>/la\_clean.png



\# 4) 线稿定界 → 透明底 → 墨迹回写 → 保 alpha 描摹

<python> -X utf8 toolkit/03\_erase/fill\_matte.py --source <work>/text\_clean.png --lineart <work>/la\_clean.png --out <work>/fig\_cut.png --shrink 2

<python> -X utf8 toolkit/03\_erase/restore\_ink\_alpha.py --cut <work>/fig\_cut.png --lineart <work>/la\_clean.png --out <work>/fig\_cut\_ink.png --report <work>/ink.json

<python> -X utf8 toolkit/04\_fixup/vectorize\_cut.py --image <work>/fig\_cut\_ink.png --out <work>/geom.svg --vtracer <skill>/bin/vtracer.exe

<python> -X utf8 toolkit/04\_fixup/split\_compound\_paths.py --in <work>/geom.svg --out <work>/fixed\_geom.svg



\# 5) 注入活字

<python> -X utf8 <skill>/scripts/figure\_cli.py merge-text --svg <work>/fixed\_geom.svg --text-manifest <work>/text\_manifest.json --output <work>/figure\_master.svg



\# 6) 原生导入交付 + 活字分层

<python> -X utf8 toolkit/05\_delivery/import\_to\_illustrator.py --svg <work>/figure\_master.svg --ai <work>/figure\_final.ai --png <work>/figure\_final.png --png-nobg <work>/figure\_nobg.png --bg-color "#fefefe" --skill-dir "<skill 目录>"

<python> -X utf8 toolkit/05\_delivery/split\_text\_layer.py --ai <work>/figure\_final.ai --out-ai <work>/figure\_final.ai --out-png <work>/figure\_final.png --skill-dir "<skill 目录>"



\# 7) 线稿叠加层（必要收尾）

<python> -X utf8 scripts/lineart\_strokes.py <work>/la\_clean.png <orig.png> <work>/strokes.json --scale 2 --smooth 0.8

<python> -X utf8 scripts/lineart\_overlay.py native --strokes <work>/strokes.json --figure-ai <work>/figure\_final.ai --out-ai <work>/final.ai --out-png <work>/final.png --viewbox 1536 1024



\# 8) 验收

<python> -X utf8 toolkit/05\_delivery/verify\_fill\_bounds.py --png <work>/final.png --png-nobg <work>/figure\_nobg.png --matte <work>/fig\_cut\_ink.png --source <orig.png> --text-manifest <work>/text\_manifest.json --tol 3

<python> -X utf8 toolkit/05\_delivery/visual\_check\_bg.py --png-nobg <work>/figure\_nobg.png --out-dir <work>/vis

```



> `--viewbox W H` 在非 1536×1024 画板时必须传。`import\_to\_illustrator.py` 必须显式传 `--skill-dir`，否则默认值指向原作者机器路径，`import figurelib` 直接失败。



\---



\## 工作流程与原理



\### 总览



```text

原图

&#x20;├─ Stage 1  文字行检测 / 模板匹配 / 局部 F1 定位

&#x20;├─ Stage 2  manifest：内容 + 框 + 字号 + 基线 + 颜色

&#x20;├─ Stage 3  擦字 ⇄ 残字闭环

&#x20;├─ Stage 4  线稿提取 → 字形级去字 → 一笔几何

&#x20;├─ Stage 5  线稿定界 → 墨迹回写 → 保 alpha 描摹 → 拆分/修整

&#x20;├─ Stage 6  merge-text 注入活字

&#x20;├─ Stage 7  原生导入 → Background 层 → 活字分层

&#x20;├─ Stage 8  AniLines lineart 独立层叠加

&#x20;└─ Stage 9  数值验收 + 双底色目视验收

```



\### 路线 A：栅格重建（Raster Reconstruction）



适用于：AI 生成图、论文截图、网上公开示意图、多面板机制图。



\*\*核心原则：先转写、再擦字、后描摹、最后注活字。\*\*



1\. \*\*转写与量测\*\*

&#x20;  - 读图只用于读 \*\*内容\*\*，不用于读坐标。Read 工具会缩放大图，目测坐标可偏 20–100 px；所有放置、字号、擦除目标一律来自程序量测的连通域墨迹框。

&#x20;  - 行检测（`detect\_text\_lines.py`）产出 `lines.json` 与 1:1 标注裁片。

&#x20;  - 已知文本时优先模板匹配（`match\_text\_template.py`）；纹理密集机制图改用局部窗口 + 二值重叠 F1 定位（`local\_match.py`）。033 实测：模板匹配 69 条里 60 条低分错位，`local\_match` 仅 3 条低分。

&#x20;  - 定位框必须过 `grow\_boxes.py`：模板外框比源墨迹窄 6–12%，不补会在首尾留半个字母。

&#x20;  - 多字符按 \*\*宽度\*\* 拟合字号（行宽与源一致），1–2 字符按 \*\*高度\*\* 拟合（徽标数字宽窄差异大）。

&#x20;  - `verify\_text\_fit.py` 是擦字前最后一道闸：逐条量 `dw/dh/dx/dbase`。033 定稿中位 `|dw|=0.1`、`|dx|=0.0`、`|dbase|=0.0`。



2\. \*\*擦字\*\*

&#x20;  - 首选 `erase05.py`：掩膜级擦除，只改字形掩膜像素，框内其余像素原样保留，不会擦出矩形鬼影。

&#x20;  - 纯白底、密集小标签可用 `erase\_text\_v2.py`：底类自动路由（flat → 平色 / artwork → inpaint），支持强制 inpaint ID 与有界残字轮。

&#x20;  - 贴底纹、渐变边、色块白字必须用 `erase05.py`；白字取色不能采样，一律给 `#ffffff`；极性看局部对比，不看全局。

&#x20;  - 残字闭环：`scan\_remnants.py` 扫描 → 并框/归位 → 重擦 → 复扫，2–3 轮。



3\. \*\*描摹\*\*

&#x20;  - `faithful` 档：VTracer 1.x、4× Lanczos、最多 128 色、filter-speckle=4、color-precision=8、gradient-step=4、simplify=0.6、optimize=0；重采样图上限 32 MP。

&#x20;  - 同绕向复合路径拆分（`split\_compound\_paths.py`）解决 QA 的 “Ambiguous nonzero compound fill”。

&#x20;  - 局部鬼影可用 `delete\_ghost\_paths.py` 补救，但优先走残字闭环根治。



4\. \*\*注入活字\*\*

&#x20;  - `merge-text` 把 manifest 里的每个 run 变成真实 `<text>`，按原基线放置。

&#x20;  - 所有标签原本在图形上方时，给独立 `Editable text` 层；交错文本保持原绘制顺序。

&#x20;  - `letter\_spacing` 恒 0；`font\_size ≤ 45` 且 `≤ 框高/0.72`。



5\. \*\*原生导入\*\*

&#x20;  - `import\_to\_illustrator.py`：`taskkill Illustrator` → sleep ≥70 s → 启动 → `app.open(master.svg)` → 建 `Background` 层 → `export PNG 200%` → `saveAs(.ai)`，单次调用全链。

&#x20;  - 启动必须 `DEVNULL + DETACHED`，否则子进程继承 stdout 管道，后台包装进程写完文件仍不退出。

&#x20;  - SVG 导入路线保持画板 1:1；batch-runtime 绘制会加 0.72 缩放白边，已弃用。



\### 路线 B：线稿重建（AniLines 一笔几何）



适用于：需要独立线稿层、需要一笔几何作为绘图源、需要叠加到已有 `figure.ai` 的场景。



```text

original.png

&#x20;├─ Stage 1  lineart\_extract.py   AniLines basic/detail → lineart.png

&#x20;├─ Stage 2  lineart\_text\_filter.py  字形级去字 → clean.png

&#x20;├─ Stage 3  lineart\_strokes.py  一笔装配 → strokes.json

&#x20;├─ Stage 4a lineart\_overlay.py preview   本地合成预览

&#x20;└─ Stage 4b lineart\_overlay.py native    独立 AniLines lineart 层

```



\*\*Stage 1 — AniLines 提取\*\*



```bash

<python> -X utf8 scripts/lineart\_extract.py <image.png> <lineart.png> --mode detail --scale 2

```



\- `basic`：笔画最少，适合一笔分解。

\- `detail`：保留细纹理，默认。

\- `--scale 2`：全卷积网络，输入先 Lanczos 2× 上采样再推理，细线不被 Nyquist 抹掉；三阶段默认 `--scale 2`，`--scale 1` 为旧快速路径。输出坐标始终回原图空间。

\- 权重：`third\_party/anilines/weights/anilines\_{basic,detail}.pth`（各 69 MB，MIT，gyrojeff/AniLines）。重装：从 `https://hf-mirror.com/gyrojeff/AniLines/resolve/main/` 下载 `basic.pth` / `detail.pth`，加 `anilines\_` 前缀。



\*\*Stage 2 — 字形级文字过滤\*\*



```bash

<python> -X utf8 scripts/lineart\_text\_filter.py \\

&#x20; --lineart <lineart.png> \\

&#x20; --text <text\_spec.json> \\

&#x20; \[--text-manifest <text\_manifest.json> --measure-from <original.png>] \\

&#x20; --out <cleaned.png>

```



算法：二值化线稿 → 8 连通域标记 → \*\*只删 bbox 完全落在一个文字墨迹框内（+3 px 容差）的连通域\*\*。跨框的引线、标签芯片、细胞轮廓一律保留。无 inpaint、无平填、不损坏穿越图形。



文字框来源（并集）：



\- `text\_spec.json` — `lines\[].bbox`，实测紧致墨迹框（首选）。

\- `text\_manifest.json` + `--measure-from` — `runs\[].erase\_bbox` 常被外扩 7–50 px，只作 \*\*搜索窗\*\*，在窗内从原图重新量紧致墨迹框。



> 已知限制：Stage 2 会保留贴图形的字形碎片（完全落入规则）。可接受——替代方案（整框擦除）会损坏图形，更糟。



\*\*Stage 3 — 一笔几何装配\*\*



```bash

<python> -X utf8 scripts/lineart\_strokes.py <cleaned.png> <original.png> <strokes.json> \[--scale 2] \[--smooth 0.8]

```



二值化 → 闭 1 px 裂缝 + 面积去噪 → 骨架化 → Rutovitz 交叉数判节点 → 节点聚类 → 链提取 → 切向连续合并 → 每笔折线（closed flag、宽度、从原图采样的墨色）。



\- “一圆一笔”由构造保证：纯环保持 `closed=true`，跨节点碎片按切向连续合并（`parts>1`）。

\- `--scale 2` 消费 2× 线稿，距离变换宽度减半回原单位，给出亚像素宽度估计。

\- `--smooth` 沿路径轻高斯，去骨架阶梯，让折线贴合原曲线。

\- 墨色采样：逐中心线点取 5×5（回退 9×9）窗口最暗像素，按笔画取中位；最暗仍 ≥250 视为无墨跳过，打印计数。`--keep-blank` 保留，`--blank-lum` 调阈值。



\*\*Stage 4a — 本地预览\*\*



```bash

<python> -X utf8 scripts/lineart\_overlay.py preview \\

&#x20; --strokes <strokes.json> --figure-png <figure.png> --out <preview.png> \\

&#x20; \[--mode full] \[--viewbox W H]

```



`--mode full` 用于 SVG 导入文档（画板贴合）；默认 `batch` 匹配 skill 的 0.72 放置。非 1536×1024 画板必须传 `--viewbox W H`。



\*\*Stage 4b — 原生独立层\*\*



```bash

<python> -X utf8 scripts/lineart\_overlay.py native \\

&#x20; --strokes <strokes.json> --figure-ai <figure.ai> \\

&#x20; --out-ai <figure\_lineart.ai> --out-png <figure\_lineart.png> \\

&#x20; \[--viewbox W H]

```



打开 `figure.ai`，解析放置变换（优先 root note `scientific-vector-local|hash|left|top|scale`；无 note 的 SVG 导入文档回退到 viewBox 居中贴合），新增独立 `AniLines lineart` 层，1:1 映射到 artwork，另存新文件——\*\*原 `figure.ai` 不会被覆盖\*\*。



\*\*质量门\*\*



`lineart\_eval.py` 对原图（排除文字区）测：`edge\_recall`（原 Canny 边被 ±tol 内墨迹覆盖——细节保留）、`edge\_precision`（墨迹落在原边上——贴合）、`ink\_density`（过度提取监控）。改参数前后同有效分辨率对比。



\### 填充边界与背景（强制）



> \*\*线稿是唯一权威。不要用颜色判断背景。\*\*



每个填充对象必须留在线稿围出的标记内；页面背景绝不能烘焙进 artwork。重建后若有颜色漏进页边、或携带不透明页面大小背景路径，即不可交付——用户需要能把图 key 到另一背景。



\*\*顺序固定：先线稿，后填充边界。\*\*



1\. 完成 `AniLines lineart` 层（extract → glyph filter → strokes）。

2\. 从同一线稿派生 matte。

3\. 再描摹填充。

4\. 线稿层是独立交付物，\*\*之后不得再编辑\*\*——不改色、不擦除、不“修”笔画让填充好看。唯一允许的编辑是字形级文字过滤（标签必须是活字）。



\*\*管线步骤\*\*



```bash

\# 1) 线稿定界 → 透明底

<python> -X utf8 toolkit/03\_erase/fill\_matte.py --source <text\_clean.png> --lineart <la\_clean.png> --out <fig\_cut.png> --shrink 2



\# 2) 墨迹回写 alpha（必跑）

<python> -X utf8 toolkit/03\_erase/restore\_ink\_alpha.py --cut <fig\_cut.png> --lineart <la\_clean.png> --out <fig\_cut\_ink.png> --report <ink.json>



\# 3) 保 alpha 的 faithful 描摹

<python> -X utf8 toolkit/04\_fixup/vectorize\_cut.py --image <fig\_cut\_ink.png> --out <geom.svg> --vtracer <skill>/bin/vtracer.exe



\# 4)（可选）清页面色大块填充

<python> -X utf8 toolkit/04\_fixup/drop\_bg\_fills.py --svg <master.svg> --out <master\_cut.svg> --source <orig.png> --min-area 8000



\# 5) 交付：Background 层 + 透明底 PNG

<python> -X utf8 toolkit/05\_delivery/import\_to\_illustrator.py --svg <master\_cut.svg> --ai <figure\_final.ai> --png <figure\_final.png> --png-nobg <figure\_nobg.png> --bg-color "#fefefe" --skill-dir "<skill 目录>"

```



\*\*关键细节\*\*



\- `fill\_matte.py` 的方向性闭运算：各向同性核大到封画框角，也会封 5–10 px 面板白沟，整页变不透明。`--shrink 2`（默认）补偿 VTracer 沿 alpha 边约 2 px 的系统外扩；改前先实测。

\- `vectorize\_cut.py` 替代 `figure\_cli.py vectorize`：后者 faithful 档预处理 `convert('RGB')` 会丢 alpha、把背景写回。

\- `drop\_bg\_fills.py` 只在页面为纯色时做：按 `--source` 从原图四边取样页色，\*\*绝不写死白色\*\*；面积门槛（默认 8000 px²）不可去——只按颜色删会把细胞质、肺泡腔等内容色一起挖成透明洞（01 实测 1585 条页色路径里 605 条是内容）。页面非纯色就跳过，几何定界本身已保证不越界。

\- 文字豁免填充约束，但文本框不得自带背景补丁；标签背后的芯片/盒子属于 artwork 层，作为几何描摹，不能以文本框填充形式到达。



\*\*验收（两项都必做，互不替代）\*\*



```bash

\# 数值

<python> -X utf8 toolkit/05\_delivery/verify\_fill\_bounds.py \\

&#x20; --png <final.png> --png-nobg <nobg.png> --matte <fig\_cut\_ink.png> \\

&#x20; --source <orig.png> --text-manifest <manifest> --tol 3



\# 目视：黑底 + 洋红底两张

<python> -X utf8 toolkit/05\_delivery/visual\_check\_bg.py --png-nobg <nobg.png> --out-dir vis

```



| 指标 | 含义 | 阈值 |

| --- | --- | --- |

| `spill\_ratio` | matte 外 + 容差的不透明像素（排除文本框） | ≤ 0.10% |

| `bg\_channel\_delta` | 导出边框色 vs 源边框色 | ≤ 2 |

| `corner\_alpha` | 四角 alpha | == 0 |

| 最大页色连通域 | 是否残留页面底板 | < 8000 px²（仅做了清底时要求） |

| `white\_left\_ratio` | 信息项 | — |



\*\*`--tol` 怎么定\*\*：VTracer 沿 alpha 边系统性外扩，`fill\_matte --shrink` 只补一部分。用合成方块实测：200×200 图、中间 100×100 不透明方块，faithful 参数描回 47.5..152.6（应 50..150）→ 外扩 2.5–3 px。本机 VTracer 1.0.0-alpha.4 取 `--tol 3`；写死 2 会把描摹器偏差算成“填充越界”（033：tol=2 → spill 0.0122 FAIL；tol=3 → 0.00056 PASS）。



\*\*目视判读\*\*：黑底上图形外与面板间隙不得有色块；洋红底上任何非洋红色小块都是填充溢出。细胞质、芯片、肺泡腔这类 \*\*被自己描边围住\*\* 的内填色属于内容，允许保留——判依据是“有没有自己的描边”，不是“颜色浅不浅”。



\*\*已知软点\*\*：线稿未闭合处，区域与页边连通，被 flood 判为外部，内部填充被一并清掉。表现为某块区域凭空消失——看 `matte.json` 的 `kept\_ratio`，回线稿补笔或调大 `--seal`，\*\*不要关 matte\*\*。线条本身被清掉用 `restore\_ink\_alpha.py` 解决，两者不是一回事。



\### 交付与验收



\*\*验收清单\*\*



\- \[ ] 每个标签与重要结构对照源图：箭头、颜色、布局、细节。

\- \[ ] 原生文本数 == 审阅过的 SVG runs 数，字体正确。

\- \[ ] 栅格重建包含独立 `AniLines lineart` 层（不是合并进 artwork）。

\- \[ ] 交付真实路径 + 文本，无嵌入位图、外链、脚本。

\- \[ ] 用户已有 artwork 与原文件未被破坏。

\- \[ ] 看 \*\*Illustrator 实导 PNG（200%）\*\* 逐标签核对：无双影、无字号爆表、无残影重叠。本地路径预览不完全渲染文本，不能替代此检查。

\- \[ ] `native\_export\_complete\_visual\_review\_required` 只表示自动原生检查完成，\*\*视觉审阅仍必须做\*\*；只有 Illustrator 真正保存后才宣称原生 `.ai` 交付。

\- \[ ] `verify\_fill\_bounds.py` 四项 PASS，且两张 `visual\_check\_bg.py` 合成图已目视检查。

\- \[ ] `.ai` 带独立 `Background` 层（源页色）；隐藏后无任何不透明像素。

\- \[ ] 披露描摹差异；视觉验收不是像素级完全一致的证明。



\*\*交付物\*\*



`.ai`、`.svg`、`.png`（白底）、`.png`（透明底）、`text\_manifest.json`、`qa.json`、`playback.json`、`lineart\_strokes.json`、必要的审计记录。



\*\*图层顺序\*\*



`import\_to\_illustrator` 建 `Background`（置底）→ `split\_text\_layer` 加 `Editable text`（置顶）→ `lineart\_overlay native` 加 `AniLines lineart`（再置顶）。最终自上而下：



```text

AniLines lineart

Editable text

图层 1（trace geometry）

Background

```



两个“看起来像丢东西”的正常现象：



\- SVG 导入后图形被合并成 \*\*1 个 GroupItem\*\*（`trace geometry`），图层面板只显示 `图层 1 (1)`；不是丢图形，`pathItems` 仍能数到全部轮廓。

\- `pathItems` 数 ≠ manifest runs 数：前者是描摹路径，后者是文本框，两套独立。



\---



\## 工具包 toolkit



`toolkit/` 是五轮实图复刻沉淀下来的补全工具集，覆盖 skill 本体不包的中段与两端。\*\*每个阶段开始先看 `toolkit/` 有没有现成脚本可干或可改；从头写是最后手段（“非必要不从头开始写代码”）。在副本上工作，绝不改 `toolkit/` 原目录；适配记录写进 job log。\*\*



完整用法见 \[`toolkit/调用指引手册.md`](toolkit/调用指引手册.md)。脚本索引：



| 阶段 | 脚本 | 用途 / 何时用 |

| --- | --- | --- |

| 1 检测 | `01\_detect/detect\_text\_lines.py` | 每图第一步，产出 `lines.json` + 1:1 标注裁片 |

| 2 manifest | `02\_manifest/match\_text\_template.py` | 已知文本首选；PIL 渲染模板 + 多尺度 matchTemplate + 位置先验；低分自动标 `<-- low-score` |

| 2 定位（纹理图） | `02\_manifest/local\_match.py` | 纹理密集图默认定位器；局部窗口 + 二值重叠 F1，平坦背景拿不到分 |

| 2 补框 | `02\_manifest/grow\_boxes.py` | 定位后必跑；横向按列间隙、纵向按行墨量延伸；徽标数字 `--skip-vertical-suffix \_num` |

| 2 定稿 | `02\_manifest/fit\_sizes.py` | 多字符按宽度、1–2 字符按高度；宽高比越界回落高度；`--colors` 传字色 |

| 2 校验 | `02\_manifest/verify\_text\_fit.py` | 擦字前最后一道闸；量 `dw/dh/dx/dbase` |

| 2 库 | `02\_manifest/pil\_metrics.py` | PIL 实测度量（含 kerning），先在参考号真渲染再线性缩放 |

| 2 判体 | `02\_manifest/font\_pick.py` | 源字体不可得时先量再选；宽高一致性与字形 F1 分开报告 |

| 2 manifest | `02\_manifest/build\_manifest.py` | 行框指派 + 字号/基线/填色；输出与 merge-text 兼容 |

| 2 库 | `02\_manifest/text\_metrics.py` | pen-aware 度量/放置/底类判别（flat/artwork 路由） |

| 2 可选 | `02\_manifest/snap\_runs.py` | run 就地吸附到邻近行框；低置信用 Stage 3b 复核 |

| 2 量测 | `02\_manifest/probe.py` | 手工写框时唯一可靠来源；`--light` 亮字暗底、`--thr 175\~190` 压图标签、`--json` 直接喂 manifest |

| 2c 校正 | `02\_manifest/refine\_boxes.py` | 框被面板边框/色条/箭头污染时用；输出框取自渲染模板本身；`score<0.55` 转人工 |

| 2e 转格式 | `02\_manifest/conv\_manifest.py` | 补 canvas/font\_size/font\_family/reviewed；`erase\_bbox` 取整（浮点会让 `lineart\_text\_filter` 抛 TypeError） |

| 2 示例 | `02\_manifest/runs\_example\_03\_2.json` | 66 条 runs 实例（含粗斜体、hint） |

| 2 示例 | `02\_manifest/spec\_example\_03\_3.json` | 69 条模板匹配 spec 实例 |

| 3a 擦除（首选） | `03\_erase/erase05.py` | 掩膜级擦除；只改字形掩膜像素；`polarity=-1` 亮字暗底；`disc` 圆内约束 |

| 3a 擦除（备选） | `03\_erase/erase\_text\_v2.py` | 纯白底/密集小标签；底类自动路由；`--inpaint-ids` 强制；`--residual` 有界残字轮 |

| 3a 擦除（v1） | `03\_erase/erase\_text.py` | 三分支；白字主色平填 `--white-ids` |

| 3b 残字闭环 | `03\_erase/scan\_remnants.py` | 每图必跑 2–3 轮；残字在哪真文字就在哪；`--apply` 写回 |

| 3.6 定界 | `03\_erase/fill\_matte.py` | A 路线核心；alpha 只由线稿决定；方向性闭运算封缝；`--shrink` 抵消 VTracer 外扩 |

| 3.6b 复原 | `03\_erase/restore\_ink\_alpha.py` | fill\_matte 后必跑；救回虚线/开口轮廓；033 实测 `restored\_px=52853` |

| 4a 拆分 | `04\_fixup/split\_compound\_paths.py` | QA 报 “Ambiguous nonzero compound fill” 时必用 |

| 4b 删鬼影 | `04\_fixup/delete\_ghost\_paths.py` | 补救专用；zone 宁大勿小 |

| 4c 描摹变体 | `04\_fixup/vectorize\_cut.py` | 保 alpha 的 faithful 描摹；参数与 skill faithful 逐项一致 |

| 4d 清底 | `04\_fixup/drop\_bg\_fills.py` | 只删页面色大块；页色从 `--source` 取样；面积门槛不可去 |

| 4e 裁剪（参考） | `04\_fixup/clip\_to\_matte.py` | \*\*不用于交付\*\*：产 `<defs>` 与 `validate` 冲突 |

| 4f 线稿去字 | `04\_fixup/filter\_text\_strokes.py` | 剔除落在文字框内的线稿笔画；pad 默认 8，重影严重取 12 |

| 5 交付 | `05\_delivery/import\_to\_illustrator.py` | 原生 SVG 导入 → `.ai` + PNG（Background 层）；内置 taskkill→sleep≥70→启动→导入 |

| 5 交付（attach） | `05\_delivery/import\_attach.py` | 用户已手动开 Illustrator 时用；跳过重启直接 attach；JSX 与前者逐句一致 |

| 5b 分层 | `05\_delivery/split\_text\_layer.py` | 交付前必跑；所有 textFrame `move` 到顶层 `Editable text` |

| 5 验收（数值） | `05\_delivery/verify\_fill\_bounds.py` | spill / bg delta / corner alpha / 页色残留 |

| 5 验收（目视） | `05\_delivery/visual\_check\_bg.py` | 黑底 + 洋红底两张 |

| 8 满版拼贴 | `05\_delivery/verify\_panel\_layout.py` | 分幅覆盖率、沟槽透明率、意外孔洞、标签落点 |



\*\*实测校准值（同机同版本可直接复用）\*\*



| 量 | 实测值 | 用法 |

| --- | --- | --- |

| VTracer 沿 alpha 边外扩 | 2.5–3 px | `verify\_fill\_bounds --tol 3`；`fill\_matte --shrink` 只补 2 px |

| 模板外框相对源墨迹窄化 | 6–12%（随字串长度增加） | 定位框一律过 `grow\_boxes`，不当 `erase\_bbox` |

| 源字体 vs Arial 宽度差 | 窄约 7%（fs\_w/fs\_h 中位 0.872–0.92） | 宽度拟合后行宽正确、字高偏小，属可披露固有限制 |

| `fill\_matte` 误判开口线像素 | 约 3.4%（033：52853 / 1572864） | 不做 `restore\_ink\_alpha` 会丢整条虚线 |

| `--shrink` 后残留外扩 | 1 px 左右 | 与 `--tol 3` 配合，别压到 2 |



\---



\## 自迭代机制



每轮任务结束，自动复盘所有高价值、可复用或值得参考的脚本：



1\. 归类到 `toolkit/` 对应阶段目录；

2\. 更新 `toolkit/调用指引手册.md` 的索引与验收口径；

3\. 在 job log 记录工具适配（改了什么、为什么）。



经过五轮复刻 + 多次独立复刻实验，toolkit 已收录 \*\*24 个脚本 + 2 个示例数据\*\*，且每个收录脚本都经过 `py\_compile` + 实图冒烟；新增脚本还要求串联跑通（033 独立批：`local\_match → grow\_boxes → fit\_sizes → conv\_manifest → verify\_text\_fit`，69 条中位 `|dw|=0.1`、`|dx|=0.0`、`|dbase|=0.0`）。



历史缺陷也记在手册里（如 `scan\_remnants` 的 5 元组解包崩溃、`verify\_text\_fit` 未传 anchor 导致居中标签 dx 虚高、`font\_pick` 只用高度拟合把宽体字判差），避免下一个人重踩。



\---



\## 实测数据与性能



\### 五轮复刻总览（2026-10-26 \~ 10-27）



| 轮次 | 图 | 规格 | 文字 | 路径 | 收敛情况 |

| --- | --- | --- | --- | --- | --- |

| 1 | 01\_Introduction | 1536×1024 | 57 | 24,447 对象 | 首版目验抓 10 处偏差，二轮收敛 |

| 2 | 02\_1\_TR\_AM | 1536×1024 | 70 | 26,077 | 负字距双影排查约 3 轮收敛 |

| 3 | 02\_2\_Mo\_AM | 1536×1024 | 67 | 38,991 | 行分组 bug 修复后 2 轮收敛 |

| 4 | 03\_1\_Steady\_state | 1774×887 | 58 | 33,029 | 首轮欠佳；残字闭环后返修 11 轮 → 基本收敛 |

| 5 | 03\_2\_Disease\_dynamics | 1536×1024 | 67 | 46,329 | 闭环法 4 轮内收敛 |



共产出 \*\*305 个活字 TextFrame、约 20 万条矢量路径\*\*，画板与源图 1:1。



\### 其他独立复刻



| 图 | 规格 | 结果 |

| --- | --- | --- |

| 03\_3\_Dysfunction | 五面板 | 模板匹配 69 条全命中；`local\_match` 仅 3 条低分；定稿中位偏差 0.0–0.1 px |

| 04\_Therapy | 1536×1024 四面板 | 79 条中 8 条框被边框/色条污染，`refine\_boxes` 修正；395 字段回归 0 差异 |

| 1080×1446 六分幅满版拼贴 | 六分幅各有底色 | 标准判据结构性失效，改用版式几何 alpha、`erase\_by\_ring` 三路取色、`verify\_panel\_layout` |



\### 效率与成本



\- \*\*大型复杂综述图\*\*：接入 DeepSeek 的情况下，平均耗时约 \*\*40 分钟\*\*，成本约 \*\*2–3 RMB\*\*（用户实测口径）。

\- 自动化应“拒绝”而非“硬猜”：联合宽高拟合带拒绝阈（score>1.0 转人工），03\_2 自动指派 63/66，仅 3 条手工；纯手工映射的 02\_2 反复三轮。

\- 残字闭环把 03\_1 从 11 轮迭代降到 3 轮。

\- 验证闭环放在最便宜的层：数字扫描（残字/字号/越界）→ 渲染目验（双影/爆表/残影），两级门禁最划算。



\### 方法论教训（最高价值规则）



1\. \*\*只信程序量测的墨迹框，不信目测坐标。\*\* Read 工具缩放大图，混用 5×/2×/1× 切片会让感知位置偏 20–100 px。放置、字号、擦除目标一律来自连通域实测。

2\. \*\*字距恒 0。\*\* 负字距会让 Illustrator 双渲染同一文本框；宽度不匹配按 `fs × 目标宽 / 渲染宽` 重标字号。

3\. \*\*框用途分离。\*\* 定位框、字号目标、擦除区域是三件事。加宽擦除框拿去拟合字号会把字号吹大 20–45 pt。`fs ≤ 框高/0.72` 且 `≤ 45`。

4\. \*\*擦除要残字闭环。\*\* 擦 → 扫残字簇（宽匹配、高 ≤30 px、≥2 字符）→ 并框 → 重擦 → 复扫。一遍不收敛。

5\. \*\*绝不为去字损坏图形。\*\* 只删 bbox 完全落在文字框内的矢量路径（膨胀 ≤7 px）；宁留字形碎片，不吞箭头、颗粒、色条。

6\. \*\*残影通常是擦除层的，不是线稿层的。\*\* 标签看起来重影时，从底图同区域裁图对比；多数鬼影是描摹几何里预先存在的（字形轮廓比擦框大 5–15 px）。

7\. \*\*可靠交付：\*\* `taskkill Illustrator` → sleep ≥70 s → 启动 → `app.open(master.svg)` → `saveAs(.ai)` + `export PNG 200%`，全在一次调用内。看导出 PNG，不看路径预览。

8\. \*\*知道固有边界。\*\* 渐变量化（128 色）、字体替换、色块白字擦除成本是方法边界，披露，不当 bug。



\---



\## 已知限制与披露



| 类别 | 限制 | 说明 |

| --- | --- | --- |

| 颜色 | 渐变量化为 128 色平色 | faithful 档上限；属方法固有限制 |

| 字体 | 源字体不可得时用 Arial 近似 | 宽度拟合后行宽正确、字高偏小约 7%；属可披露差异 |

| 擦除 | 色块白字、压图文字存在“擦一半留碎片比不擦更糟”的代价 | 保守解是保留源字形、撤该处活字 |

| 残影 | 部分标签有 5–15 px 浅影（源字形比擦框大） | 每处修复约一轮“并框→重擦→重描”（约 6 分钟/组） |

| 线稿 | 开口线/虚线会被 flood 判成外部 | `restore\_ink\_alpha.py` 必跑；仍丢面时回线稿补笔或调 `--seal`，不要关 matte |

| 满版拼贴 | 多分幅各有底色时，标准判据结构性失效 | 见 `toolkit/调用指引手册.md` §8：版式几何 alpha、`erase\_by\_ring`、`verify\_panel\_layout` |

| 输入类型 | 复杂显微照片一般不适合全矢量管线 | 见 `input-and-validation.md` |

| 数据 | 描摹不能恢复实验数据/数值测量 | 统计图应使用原始矢量导出与真实数据 |

| 原生 | Windows 27.2.0 真机验证；macOS 仅适配器契约检查 | 其他版本需逐项验证；不宣称通用兼容 |

| 降级 | 无 Illustrator 时无法产出 `.ai` / 活字分层 / 原生线稿层 | 降级交付 SVG + PNG + `lineart\_strokes.json` + 线稿位图预览，并明示原因 |



\*\*降级交付\*\*（`illustrator-doctor` 报 `Open Illustrator yourself` 且确认未安装）：



\- SVG（8109 对象 / 48 活字 / 0 位图）

\- PNG（白底）

\- PNG（透明底）

\- `lineart\_strokes.json`（3330 条一笔笔画，可在 Illustrator 里脚本重绘为独立层）

\- `lineart\_layer.png`（线稿位图预览）

\- 必须向用户明示 `.ai` 未产出及原因



\---



\## 目录结构



```text

illustrator-Master Copyist/

├── SKILL.md                          # Skill 主文档（Agent 读取）

├── README.md                         # 本文档

├── LICENSE                           # 上游 MIT 许可

├── NOTICE.md                         # 来源与许可说明

├── .venv/                            # 自包含 CPython 3.13.14 + 全部依赖

├── bin/

│   └── vtracer.exe                   # VTracer 1.0.0-alpha.4（pinned SHA256）

├── third\_party/

│   └── anilines/                     # AniLines 代码 + 双 69 MB 权重（MIT）

│       └── weights/

│           ├── anilines\_basic.pth

│           └── anilines\_detail.pth

├── scripts/

│   ├── figure\_cli.py                 # 共享 CLI 入口

│   ├── figurelib/                    # SVG/几何/COM/AppleScript/调度

│   ├── setup\_environment.py          # 从源码安装依赖（可选）

│   ├── selfcheck.py                  # 包自检（不访问 Illustrator/网络）

│   ├── lineart\_extract.py            # Stage 1

│   ├── lineart\_text\_filter.py        # Stage 2

│   ├── lineart\_strokes.py            # Stage 3

│   ├── lineart\_overlay.py            # Stage 4a/4b

│   └── lineart\_eval.py               # 质量门

├── toolkit/

│   ├── 调用指引手册.md                # 工具包权威用法

│   ├── 01\_detect/

│   ├── 02\_manifest/

│   ├── 03\_erase/

│   ├── 04\_fixup/

│   └── 05\_delivery/

└── references/

&#x20;   ├── workflow.md                   # 栅格重建与恢复

&#x20;   ├── input-and-validation.md       # 输入/验收规范

&#x20;   ├── installation-and-compatibility.md

&#x20;   ├── lineart-pipeline.md           # 线稿管线

&#x20;   └── remake-test-report.md         # 五轮实测报告

```



> 以实际发布包为准；`toolkit/` 脚本清单随自迭代增长。



\---



\## 故障排查速查



| 症状 | 处理 |

| --- | --- |

| Illustrator 连接失败 | 确认 app 在同一用户会话中打开；Windows 用 PowerShell 5.1 COM bridge |

| 多版本 Illustrator 同时在跑 | Windows 用 `--prog-id Illustrator.Application.N`；macOS 用 `--app-path` 指定实际运行 `.app` |

| macOS 拒绝自动化 | 用户在「隐私与安全性 → 自动化」给终端/Agent 授权；脚本不改系统权限 |

| 缺字体 | 确认目标字体或约定替代字体，更新 SVG/manifest，开新 job |

| 浅色细胞核消失 / 背景被压平 | 比较更高色数候选，检查局部区域；白底修改只针对已识别背景对象 |

| 小闭合三次环被拒 | 解析器支持拆分单个闭合三次曲线；用更新版 skill，开新 job |

| 没有清晰放置空间 | 用空白画板/文档；只有已获授权时才 `--allow-overlap` |

| Adobe 对话框阻塞绘制 | 检查对话框，确认上一操作空闲后再开始 |

| 日志成功但图不对 | 看真实 PNG、字体面、对象层级；执行成功不等于视觉验收 |

| `ModuleNotFoundError: figurelib` | `import\_to\_illustrator.py` / `split\_text\_layer.py` 显式传 `--skill-dir` |

| 后台包装进程写完不退出 | 启动 Illustrator 用 `DEVNULL + DETACHED` |

| 图层面板一选就选中几万条路径 | 交付前跑 `split\_text\_layer.py`，活字拆到 `Editable text` 层 |

| 线稿层整片发灰/噪点 | 不要用擦字图提线稿；用原图 → `lineart\_extract` → `lineart\_text\_filter` → 去 <12 px 小连通域 |

| 白字被交付成粉紫色 | 白字一律给 `#ffffff`，不要采样（源图白字 G 通道被底色渗色拉低） |

| 中央分隔线/iNOS 框消失 | `fill\_matte` 把开口线判成外部；跑 `restore\_ink\_alpha.py`，不要关 matte |

| 面板间隙被填成不透明 | `fill\_matte` 闭运算核过大封了白沟；用方向性闭运算，检查 `--seal` |

| `spill\_ratio` 刚好 FAIL | `--tol` 按实测给 3，不要用默认 2；先量本机 VTracer 外扩 |

| 满版拼贴 corner\_alpha 非 0 | 分幅铺到画布四边，角落本来有内容；改用 `verify\_panel\_layout.py` |



\---



\## 致谢



\- 感谢基于神经网络模型的开源线稿提取模型 \*\*\[AniLines](https://github.com/gyrojeff/AniLines)\*\*（MIT，gyrojeff），本 Skill 的独立线稿层与一笔几何管线建立在其之上。

\- 感谢前辈 \*\*\[scientific-illustrator-local](https://github.com/yrui-cmd/cell-lct)\*\* 项目提供思路借鉴；本 Skill 在其基础上重构并扩展了线稿重建、填充定界、工具包与自迭代机制。

\- 感谢 \*\*DeepSeek\*\* 与 \*\*MiMo\*\* 提供超强代码能力支持。

\- 感谢 \[VTracer](https://github.com/visioncortex/vtracer)（visioncortex）提供保真矢量化后端。

\- 感谢五轮实图复刻与多次独立复刻实验中每一位逐标签核对、逐残影返修的人。



\---



\## 许可



\- 本 Skill 适配并重构了 \[yrui-cmd/cell-lct](https://github.com/yrui-cmd/cell-lct)（commit `4af98d8057f48a944141f9122da7aba83d544d28`，2026-09-06T16:20:16Z）的 MIT 许可材料。上游版权与 MIT 条款保留在 \[`LICENSE`](LICENSE)。

\- 本包 \*\*不包含\*\* 任何小米喵服务器、专有算法或权重、API key、支付流程、`cell\_no\_ai` 服务、AI 来源标记移除服务、Adobe 应用、模型或字体文件；不绕过任何服务认证。

\- \[VTracer 1.0.0-alpha.4](https://github.com/visioncortex/vtracer/releases/tag/1.0.0-alpha.4) 是可选、单独安装的依赖，遵循其自身许可；安装器只下载 pinned 官方 release asset 并校验 SHA256。

\- Python 依赖各自保留其许可。

\- 模型与 Adobe 访问由用户负责；把图片给云端模型看并非完全离线。

\- 请随 Skill 保留 `LICENSE` 与 `NOTICE.md`。



\---



\*README 基于 SKILL.md v0.2.1、`toolkit/调用指引手册.md`（2026-10-29 版）、五轮实测报告与安装/输入/线稿管线文档整理。命令与阈值以随包文档为准；不同 Illustrator 版本与机器请先跑 `doctor` / `illustrator-doctor` / `native-test` 验证。\*

