# Music Autotag

给本地音乐文件**自动补全音乐标签、LRC 歌词（默认双语）与封面**的工具。
先联网匹配 → 预览确认 → 再写入，绝不瞎写。

同一套后端（`autotag.py`）配了四个入口：**macOS 原生 App**、**Windows Fluent 界面**、**网页版界面**、**命令行**。

| 入口 | 适合 | 特点 |
|---|---|---|
| **macOS App**（`build/MusicAutotag.app`，约 1 MB） | 想把歌直接拖进窗口 | 原生 Objective-C + Cocoa，Liquid Glass 侧边栏、系统统一工具栏、SF Symbols |
| **Windows GUI**（`windows/gui_win.py`） | Windows 用户 | Fluent / Win11 风格，可打包成单文件 `exe`，支持拖放 |
| **网页版 GUI**（`gui.py`） | 临时用、跨平台 | 本地起服务 + 浏览器界面，零额外依赖 |
| **命令行**（`autotag.py`） | 批量、脚本、自动化 | 支持 `--dry-run` / `-y` / `--jsonl`，可接入 CI |

---

## 功能

- **标签**：标题 / 歌手 / 专辑 / 专辑歌手 / 年份 / 音轨号，只补空缺，`--force` 才覆盖
- **歌词**：生成同名 `.lrc`，**默认合并翻译**（原文行下面紧跟译文行，跨数据源按时间轴就近对齐），可 `--embed-lyrics` 内嵌进音频标签
- **封面**：**始终内嵌**（ID3 `APIC` / FLAC `Picture` / MP4 `covr` / Vorbis `METADATA_BLOCK_PICTURE`），命令行默认另存一份与音频同名的 jpg，`--no-cover-file` 可只内嵌不落文件
- **中日文友好**：假名/汉字/韩文都能正确匹配；搜索前自动清掉 `『』「」【】` 等干扰符号
- **写后回读校验**：写完立刻重开文件确认封面/标签真的进去了，结果行写 `封面内嵌✓`，失败则 `⚠封面未内嵌（原因）`
- **限流自愈**：某数据源被限流/报错即熔断，整批改用其它源；4xx/5xx 直接失败不重试

支持格式：`mp3` `flac` `m4a` `mp4` `ogg` `opus` `wav` `aiff`（可读 `ape` / `wma` 等，但写标签会跳过）。

---

## 快速开始

### 1. 命令行

```bash
cd music-autotag

# 预览（不写盘）
python3 autotag.py ~/Music/某专辑 --dry-run

# 预览 → 输入 y 确认 → 写入
python3 autotag.py ~/Music/某专辑

# 跳过确认直接写（批量/脚本）
python3 autotag.py ~/Music -j 6 -y
```

依赖只有 `mutagen`（纯 Python），仓库里的 `vendor/` 已内置，**通常不需要装任何东西**。
若确实缺：`python3 autotag.py --install-deps`。

### 2. macOS App

```bash
./build_app.sh          # 构建 → build/MusicAutotag.app（原生 ObjC，约 1 MB）
open build/MusicAutotag.app
```

- **把歌曲文件或整个文件夹拖进窗口任意位置**（也支持拖到 Dock 图标、⌘O 添加文件）
- 左侧是可折叠侧边栏＝**所选歌曲信息**（大封面、曲名、歌手、专辑/时长/来源/匹配分/文件/写入结果）；
  右侧是**歌曲列表**（多列表格：状态/曲名/歌手/专辑/时长/来源/匹配/写入内容）
- 窗口缩到 **900pt 以下自动折叠侧边栏，拉宽到 1000pt 以上自动还原**；⌘B / 工具栏按钮 / 拖动分隔条也能折叠
- 匹配完**自动选中可写入的行** → 点「写入所选」只写选中的，低匹配/失败行会标出且不会写
- 用系统自带的 `/usr/bin/python3` 调用后端，无需打包 Python，所以体积很小

### 3. Windows GUI（Fluent / Win11 风格）

```bat
cd music-autotag\windows
启动.bat                 :: 自动装可选依赖并启动（等价于 python gui_win.py）
```

可选依赖（不装也能跑，会优雅降级）：

| 包 | 作用 | 不装的后果 |
|---|---|---|
| `sv-ttk` | 真正的 Win11 Fluent 主题 | 用自带的 Fluent 近似样式 |
| `tkinterdnd2` | 把歌曲**拖进窗口** | 只能点「添加文件 / 文件夹」 |
| `pillow` | 显示封面缩略图 | 侧栏封面显示 ♪ 占位符 |

界面与 macOS 版同构（左侧信息侧边栏 + 右侧列表 + 选项 + 底部操作条），原生标题栏和菜单栏。
快捷键：`Ctrl+O` 添加文件、`Ctrl+Shift+O` 文件夹、`Ctrl+R` 匹配、`Ctrl+S` 写入、`Ctrl+A` 全选可写入、`Ctrl+B` 折叠侧栏。

### 4. 网页版 GUI

```bash
python3 gui.py                    # 自动打开 http://127.0.0.1:8422/
python3 gui.py --no-browser --port 8422
```

---

## 工作流程（预览 → 确认 → 写入）

```
[1/3] Adele - Hello.wav              Adele - Hello（Hello，0.98）
[2/3] 周杰伦 - 晴天.wav               周杰伦 - 晴天（叶惠美，0.98）
[3/3] 文藝天国 - スクールゾーン.flac   文藝天国 - スクールゾーン（スクールゾーン，0.98）

================ 匹配结果预览（共 3 个待写入）================
 #  文件                  曲名            歌手       专辑            时长  分数  来源   写入
 1  Adele - Hello.wav     Hello           Adele      Hello           4:55  0.98  QQ音乐 标签+歌词(含翻译)+封面(内嵌)
 2  周杰伦 - 晴天.wav       晴天            周杰伦     叶惠美          4:29  0.98  QQ音乐 标签+歌词+封面(内嵌)
 3  文藝天国 - スクールゾーン… スクールゾーン   文藝天国   スクールゾーン   3:31  0.98  网易云 标签+歌词(含翻译)+封面(内嵌)
--------------------------------------------------------------
低匹配  某某 - 某某.flac    匹配分 0.42 < 0.60，最接近：翻唱者 - 某某
==============================================================
确认写入以上 3 个文件？[y] 全部写入 / [s] 逐个确认 / [n] 取消（默认 n）:
```

- 非交互终端（管道/脚本）不会自动写盘，会提示改用 `-y`
- `--dry-run` 只看预览，永远不写

---

## 命令行参数

| 参数 | 说明 |
|---|---|
| `paths` | 音频文件或目录（目录递归扫描） |
| `-y`, `--yes` | 预览后直接写入，不再询问 |
| `--dry-run` | 只预览，不写盘也不询问 |
| `-j N`, `--jobs N` | 并发线程数，**1~16**（默认 4；接口是限流的，超过 8 基本只有副作用） |
| `--min-score 0.6` | 匹配阈值，低于则报「低匹配」并跳过 |
| `--force` | 忽略「已有信息」判断，强制重新匹配 |
| `--overwrite-tags` | 覆盖已存在的标签 / `.lrc` / 封面文件 |
| `--redo-all` | 即使看起来完整也重新处理 |
| `--no-translation` | 不合并翻译歌词（默认合并双语） |
| `--strip-credits` | 去掉歌词里的「作词/作曲/Written by」等行 |
| `--embed-lyrics` | 歌词同时内嵌进音频标签 |
| `--no-lyrics` / `--no-cover` / `--no-tags` | 分别关闭歌词 / 封面 / 标签 |
| `--cover-name cover.jpg` | 封面存成目录级 `cover.jpg`（整专辑目录推荐；默认与音频同名） |
| `--no-cover-file` | 封面只内嵌进标签，不保存图片文件 |
| `--no-extend` | 不从文件名补充缺失的标题/歌手/专辑 |
| `--check` | 只检查本地已有状态（标签/歌词文件/内嵌封面），不联网也不写入 |
| `--jsonl` | 逐行输出 JSON（供图形界面调用）：scan 行 + 汇总，加 `-y` 则继续写入 |
| `--install-deps` | 自动安装 mutagen 到脚本同级 `vendor/` |

---

## 匹配规则

1. **取词**：优先用已有标签；没有标签时按文件名猜 —— `歌手 - 标题`（最佳）、`歌手 - 专辑 - 标题`、`01 标题`
2. **多源搜索**：每个查询串同时问 **QQ音乐** 和 **网易云音乐**（中文官方曲库 QQ 更强，两者互补）
3. **打分**：`标题相似度×0.6 + 歌手相似度×0.35 + 专辑相似度×0.05`，再乘时长系数
   - 时长差 ≤4s 不罚；≤10s ×0.85；≤20s ×0.6；更大 ×0.35
   - 但曲名与歌手几乎完全一致时系数下限 0.75 —— 同一首歌的不同版本（专辑版/现场版）时长差几十秒不该被淘汰
4. **阈值**：低于 `--min-score`（默认 0.6）报「低匹配」并给出最接近的候选，**绝不会硬写**
5. 已有信息完整（标签 + `.lrc` + 封面）的文件默认跳过；`--force` / `--redo-all` 可强制重做

匹配不准时，把文件改名成 **`歌手 - 标题.扩展名`** 再 `--force`，命中率最高。

---

## 数据源与稳定性

| 数据源 | 用途 |
|---|---|
| QQ音乐 | 曲库信息、官方歌词、封面（中文曲目命中率最高） |
| 网易云音乐 | 曲库信息、歌词**与翻译**（英文/日文歌的双语主要来自这里） |
| iTunes Search API | 封面兜底 |

- **翻译补齐**：当前源没有翻译时，会用另一源按「歌手 + 标题」再搜一次，把翻译按时间轴（±1.2s 就近对齐）插入原文行下方 —— 两个源的时间戳格式不同（`[mm:ss.xx]` vs `[mm:ss.xxx]`）也能对上
- **熔断**：某源返回 405/500 或被限流即标记不可用并改用另一源，整批不再重试它；全部不可用时快速返回「低匹配」而不是卡住
- **快速失败**：HTTP 4xx/5xx 直接放弃（重试无意义，只会拖慢批量）
- **缓存**：同一进程内相同查询只请求一次
- **线程**：实测 4 个文件 —— 1 线程 11.7s、4 线程 7.3s、8 线程 7.1s、16 线程 6.6s，所以默认 4、上限 16

---

## 写入内容与格式对照

| 格式 | 标签 | 内嵌封面 | 内嵌歌词 |
|---|---|---|---|
| MP3 | ID3v2 (`TIT2`/`TPE1`/`TALB`/`TPE2`/`TDRC`/`TCON`/`TRCK`) | `APIC` | `USLT` |
| FLAC | Vorbis Comment | `Picture` 块 | `LYRICS` |
| M4A / MP4 | `©nam`/`©ART`/`©alb`/`aART`/`©day`/`©gen`/`trkn` | `covr` | `©lyr` |
| OGG / OPUS | Vorbis Comment | `METADATA_BLOCK_PICTURE` | `LYRICS` |
| WAV / AIFF | ID3v2（同上） | `APIC` | `USLT` |

封面格式自动嗅探（JPEG/PNG/GIF/WebP），下载内容不是图片会被拒绝，不会把错误页写进标签。

---

## 自检与调试

```bash
python3 autotag.py ~/Music --check                 # 本地体检：标签 / 歌词文件 / 内嵌封面，不联网不写入
python3 autotag.py ~/Music --jsonl --dry-run -y    # 逐行 JSON（GUI/App 用的协议）
python3 windows/gui_win.py --self-test "D:\Music"  # Windows GUI：不开窗口只跑匹配
python3 gui.py --no-browser                        # 网页版：不自动开浏览器
```

macOS App 内置两个自检开关（开发用）：

```bash
# 打印控件坐标、越界/重叠检查，并做窗口宽度扫描（验证侧边栏自动折叠）
MUSICAUTOTAG_DUMP=1 build/MusicAutotag.app/Contents/MacOS/MusicAutotag

# 自动“拖入”文件跑一遍并打印结果
MUSICAUTOTAG_TEST_PATHS="/path/a.flac,/path/b.flac" MUSICAUTOTAG_TEST_WAIT=40 \
  build/MusicAutotag.app/Contents/MacOS/MusicAutotag
```

---

## 打包

### macOS App

```bash
./build_app.sh      # → build/MusicAutotag.app（约 1 MB）
```

打包内容：`clang` 编译出的原生可执行文件（几十 KB）+ `autotag.py` + 纯 Python 的 `vendor/mutagen`；
运行时用系统自带的 `/usr/bin/python3`，不捆绑 Python 运行时，所以体积很小。构建时会剔除 `__pycache__` 并 `strip` 可执行文件。

系统要求：macOS 12+（Liquid Glass 侧边栏需要 macOS 26+，更低版本自动退回 `NSVisualEffectView`）。

### Windows exe

```bat
windows\build_exe.bat      :: → dist\MusicAutotag.exe（单文件、无黑框、图标 windows\icon.ico）
```

用的是 `windows/MusicAutotag.spec`，里面处理了三个容易踩的坑：

| 坑 | 不处理会怎样 | 处理方式 |
|---|---|---|
| `autotag.py` 里的 `mutagen.*` 是**函数内 import**，静态分析抓不到 | 打包成功但一写标签就 `ModuleNotFoundError` | 写进 `hiddenimports`，并把 `vendor/` 加进 `pathex`（否则 PyInstaller 会报 “Hidden import 'mutagen.xxx' not found” 且不进包） |
| `sv-ttk` 主题带 `sv.tcl` / `theme/*.tcl` / 精灵图 PNG | exe **丢 Fluent 主题** | `collect_all("sv_ttk")` |
| `tkinterdnd2` 带 tkdnd 二进制 | exe **拖放失效** | `collect_all("tkinterdnd2")` |

**本机不是 Windows**：仓库里带了 `.github/workflows/build-windows.yml`，推到 GitHub 后在 Actions 里手动运行 `build-windows`，
云端 Windows 会装依赖、打包、跑一次 `--self-test` 冒烟测试，再把 `MusicAutotag.exe` 作为附件给你下载。
（PyInstaller 不支持交叉编译，macOS 上无法直接产出 Windows 的 exe。）

### 图标

`icon.icns` 已转换成 `windows/icon.ico`（内含 16/24/32/48/64/128/256 七种尺寸的 PNG 负载），
同时用作 Windows 窗口图标与打包 exe 的图标；macOS 侧直接用 `icon.icns`。

---

## 项目结构

```
music-autotag/
├── autotag.py                  # 后端：匹配 / 歌词 / 封面 / 标签写入（CLI + 被所有界面复用）
├── gui.py                      # 网页版 GUI（本地 http 服务 + 单页界面）
├── build_app.sh                # 构建 macOS .app
├── app/
│   ├── main.m                  # macOS 原生 App（Objective-C + Cocoa，Liquid Glass 侧边栏）
│   └── Info.plist
├── windows/
│   ├── gui_win.py              # Windows GUI（tkinter + Fluent / Win11 风格）
│   ├── MusicAutotag.spec       # PyInstaller 打包配置
│   ├── build_exe.bat           # 一键打包 exe
│   ├── 启动.bat                 # 一键启动（自动装可选依赖）
│   ├── requirements.txt        # sv-ttk / tkinterdnd2 / pillow / pyinstaller（全可选）
│   ├── icon.ico                # 由 icon.icns 转换（7 种尺寸）
│   └── icon_256.png
├── vendor/                     # 纯 Python 的 mutagen（已内置，开箱即用）
├── icon.icns                   # 应用图标源文件
├── .github/workflows/
│   └── build-windows.yml       # 云端 Windows 打包 exe
└── build/                      # 构建产物（MusicAutotag.app 等）
```

---

## 常见问题

| 现象 | 原因 / 处理 |
|---|---|
| 提示「低匹配 0.42 < 0.60」 | 文件名太随意。改成 `歌手 - 标题.mp3` 再 `--force`；或用 `--min-score 0.5` 放宽 |
| 播放器看不到封面 | 先用 `--check` 确认文件里真有内嵌封面；FLAC 的 `PICTURE` 块 Apple Music / Quick Look 不读，用 网易云音乐 / foobar2000 / MusicBee / VLC 查看 |
| 某个源报 405 / 500 | 该源被限流，工具会自动熔断并改用另一源（stderr 提示一次），稍后自动恢复 |
| 歌多了很慢 | 保持 `-j 4`；线程加到 16 也只快一点点，因为瓶颈是对方接口 |
| 中文歌没有翻译 | 中文原唱多数没有官方翻译，属正常；英文/日文歌的双语大多能拿到 |
| Windows 上界面不像 Win11 | 没装 `sv-ttk`，`pip install sv-ttk` 后重启 |
| macOS 窗口很窄时侧边栏消失 | 这是原生行为：≤900pt 自动折叠，拉宽或按 ⌘B 展开 |
| 拖入第二批文件后第一批没了 | 不会，现在会**合并**后重新匹配；写入过程中则拒绝新拖入，避免打断写盘 |

---

## 已验证 / 未验证

**已实测**（有输出的那种验证）：

- 命令行：匹配 → 预览表 → 确认 → 写入 → 回读校验，全流程通过（mp3 / wav / flac）
- 日文曲目命中率修复：假名归一化 + 搜索词清洗后，`カワイクナイ`、`アイデンティティ`、`スクールゾーン` 等从 0.03 提升到 0.98
- 双语歌词：`Adele - Hello` 输出 53 行、原文 + 中文翻译交错
- macOS App：布局自检（无越界/无重叠）、窗口宽度扫描（1180→展开 / 900→折叠 / 640→折叠 / 1180→还原）、集成测试自动拖入并输出 `ROWS=2 ok=2 writeEnabled=1`
- Windows GUI：界面真实构造并跑通（渲染列表、自动选中、详情面板、筛选、侧栏折叠、主题切换、线程夹取全部 `UI_OK`），`--self-test` 后端 2/2
- PyInstaller：spec 在 macOS 实跑打包成功，**打包出来的程序 `--self-test` 真跑通了匹配**
- `icon.ico` 结构回读校验（7 种尺寸、PNG 负载）

**未验证**（本机没有 Windows，需要你在真机确认）：

- Windows 上的窗口图标显示、`tkinterdnd2` 拖放、`build_exe.bat` 打包结果
- `sv-ttk` 的实际外观（它需要 Tk 8.6，本机系统 Python 是 Tk 8.5，只能验证“失败时优雅退回自带样式”）

---

## 说明

- 数据来自 QQ音乐 / 网易云音乐公开接口与 iTunes Search API，**无需账号、无需 Cookie**
- 仅供整理**自己已拥有**的音乐文件，请勿用于分发受版权保护的内容
- 写入前一律需要确认（命令行默认询问，App/GUI 需点「写入所选」）；`--dry-run` 可完全不写盘
