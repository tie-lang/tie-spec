# spec/ —— tie 语言语法规范（文档工程）

规范正文是 **tdoc 文档源**（tofflib 的统一输入格式），由 tofflib 门面装载成块流后
渲染为 PDF。产物：`tie-20262.pdf`（版本 2026.2），仓库根另存一份 `tie.pdf` 作最新别名。

## 文件

| 文件 | 职责 |
| --- | --- |
| `tie-20262.tdoc` | 文档源：封面、目录、17 章正文、代码块、表格、提示段 |
| `build.tie` | 构建入口：`load_file` 装载 → 门面渲染 PDF → 生成后结构自检 |
| `make.tsh.tie` | 构建脚本（tshell）：编译 build.tie 并运行，报告退出码 |
| `speckit.tie` | 早先的 **docx** 版渲染工具（PDF 路线取代后仅作参考） |

文档源由 tofflib 的 `src/doc/load.tie` 解析：块形状（`h1`…`h9`/`p`/`ul`/`tbl`/`tblw`/
`code`/`note`/`cover`/`toc`/`page` 等）与 `src/doc/model.tie` 的协议一一对应。
表格列宽写作 **twips**（20 twips = 1 pt，与 docx 的 `dxa` 同单位），PDF 后端按 1/20 换算。

## 字体（授权是选型的第一判据）

两款字体均为 **SIL OFL 1.1**：可商用、可嵌入产物、可再分发。

| 槽 | 字体 | 轮廓 | 说明 |
| --- | --- | --- | --- |
| 0 正文 | `NotoSansSC-VF.ttf`（思源系） | glyf | 含完整汉字与符号 |
| 1 代码 | `JetBrainsMono-Regular.ttf` | glyf | 等宽；仅 642 字形、无汉字、缺 `→`，**必须回退到槽 0** |

**不采用** `simhei` / `微软雅黑` / `等线`：它们随 Windows 授权，把字形嵌入对外分发的文档
属于再分发，有法律风险。

**轮廓类型必须匹配嵌入方式**：本库按 CIDFontType2 + FontFile2 嵌入，要求 **TrueType 轮廓**
（`glyf` + `loca`）。名为 `Noto Sans SC (TrueType).otf` 的静态版实际是 **CFF 轮廓**——
按上述方式嵌入会让阅读器把 CFF 当 TrueType 解析、**画出错误的字形**，而结构、GID、
宽度全部"正常"。`font_load` 因此带硬性守卫：缺 `glyf`/`loca` 直接拒绝并给出可操作提示。

## 生成方式

```sh
# 在仓库根执行（字体路径相对仓库根）
../tshell/_local_only/bin/tsh_main.exe -f source/20262/make.tsh.tie
```

手工两步：

```sh
tiec --no-cache source/20262/build.tie -o source/20262/build.exe
./source/20262/build.exe
```

> **务必加 `--no-cache`**：编译器有产物缓存，而 tofflib 与文档源改动频繁，
> 旧缓存会让人误判"改完仍失败"。

## 自检（构建的一部分）

`build.exe` 生成后**读回产物**逐层校验：`verify`（xref 偏移）、`objects`（对象与流长度）、
`content`（GID 在字体范围内、notdef 占比）、`layout`（文本定位不出页）、`outline`
（用到的字形在 `loca` 里真有轮廓）。

**结构自检不足以保证"看得见"**。本工程遇到过下列全部通过结构检查的缺陷：CFF/TrueType
轮廓错配（字形画错）、字体子集 `loca` 少一项（全文整体错位一个字）、缺 `/ToUnicode`
（视觉正常但复制/搜索出错）、宽度错千倍（换行与字距同时失效）。因此最终验收**必须包含
真实渲染**。

## 真实渲染验收

```sh
# 仅用于验收，不参与构建；期望 77 页、约 10.4 万字形
python -c "import pymupdf; d=pymupdf.open('tie-20262.pdf'); print(d.page_count)"
python -c "import pymupdf; d=pymupdf.open('tie-20262.pdf'); d[1].get_pixmap(dpi=110).save('p2.png')"
```

## 已知局限

* 目录页码是文本，未加内部跳转链接（需 `/Annots` 与目标页引用）。
* 未生成 `/Outlines` 书签，阅读器侧栏目录为空。
* 正文槽把 ASCII 连字符映射到 U+2011：正文里的 `-` 提取出来是非断行连字符，视觉无差别，
  但按 ASCII `-` 搜索正文匹配不到；代码槽映射正常。

## 约束（严格遵守）

* **不使用任何外部脚本语言**：构建编排一律用 **tshell**，文档生成一律用 **tofflib 自身**。
* **二进制落盘必须走 `byte_write`**：`file_write` 在 Windows 上遇 NUL 字节截断
  （实测写 4 字节只落 3 字节），而嵌入字体满含 NUL。
* **路径用绝对路径**（tshell 的 `exec_code` 不提供 shell 的复合命令语义）；tshell 输出用 ASCII。
* **文档源里的字符串按 td 规则转义**：引号写 `\"`，反斜杠写 `\\`。代码块用 **here-doc 形态**
  书写：开引号后立即换行，内容每行统一带 12 个空格前缀，装载器按**固定 12 列**剥离，
  因此代码自身的相对缩进（函数体内的几行）原样保留，而书写位置不影响内容。
  长段落用 `\` 续行：换行与续行缩进都不进入文本，可按可读宽度折行。
  代码示例不需要 `lb()`/`rb()` 拼花括号——tdoc 是数据，花括号就是普通字符。
