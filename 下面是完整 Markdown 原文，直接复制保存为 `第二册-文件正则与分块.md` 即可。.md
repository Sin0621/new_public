下面是完整 Markdown 原文，直接复制保存为 `第二册-文件正则与分块.md` 即可。  
  
---  
  
# 《Python Web 项目实战》第二册：文件、正则与分块  
  
| 项目 | 内容 |  
|---|---|  
| 所属系列 | 《Python Web 项目实战》 |  
| 册数 | 第二册（对应开发阶段二：Day 3-4） |  
| 主题 | 文件读写 · 异常处理 · 正则表达式 · 文本分块 · 生成器 |  
| 前置要求 | 完成第一册（配置 / 日志 / 异常 / 路由骨架） |  
| 版本 | v1.0（2026-10-21） |  
  
> **本册定位**：第一册搭好了“空房子”，这一册开始往里搬东西——**把用户的文件读进来、洗干净、切成小块**。  
> 这是整个 AI 知识库项目的**入口**，也是最容易出 bug 的地方（编码、异常、大文件）。  
>  
> **读法同第一册**：需求 → A 版能跑 → 逐行 → 发现毛病 → B 版 → 3 句话总结。每段代码都请亲手敲。  
  
---  
  
## 目录  
  
- [第 0 章 本册地图](#第-0-章-本册地图)  
- [第 1 章 打开与读取文件](#第-1-章-打开与读取文件)  
- [第 2 章 接住错误：try / except](#第-2-章-接住错误try--except)  
- [第 3 章 正则表达式（re）](#第-3-章-正则表达式re)  
- [第 4 章 文本清洗：把正则组装起来](#第-4-章-文本清洗把正则组装起来)  
- [第 5 章 文本分块（chunk）](#第-5-章-文本分块chunk)  
- [第 6 章 生成器：大文件不能一次读进内存](#第-6-章-生成器大文件不能一次读进内存)  
- [第 7 章 组装成模块](#第-7-章-组装成模块)  
- [第 8 章 给它写测试](#第-8-章-给它写测试)  
- [第 9 章 自测（8 题）](#第-9-章-自测8-题)  
- [附录 A 本册知识地图](#附录-a-本册知识地图)  
- [附录 B 完整文件清单](#附录-b-完整文件清单)  
  
---  
  
# 第 0 章 本册地图  
  
学完这一册，你会拥有：  
  
```  
用户文件 (.txt/.md)  →  读进来（第1章）  →  出错能兜住（第2章）  
                     →  用正则洗干净（第3、4章）  →  切成小块（第5章）  
                     →  大文件也不炸（第6章）  →  组装成模块（第7章）  →  有测试（第8章）  
```  
  
最终产出两个文件：  
  
| 文件 | 职责 |  
|---|---|  
| `app/services/loader.py` | 把文件读成字符串，出错时抛明确异常 |  
| `app/services/splitter.py` | 清洗文本 + 切分成块 |  
  
---  
  
# 第 1 章 打开与读取文件  
  
## 1.1 需求  
  
用户上传一个 `.txt` 或 `.md`，我要把里面的**文字**读成一个 Python 字符串。  
  
## 1.2 先认识 `open()` 的三个参数  
  
```python  
open(文件路径, mode="r", encoding="utf-8")  
```  
  
| 参数 | 含义 | 常用值 |  
|---|---|---|  
| 路径 | 文件在哪 | `"data/a.txt"` |  
| `mode` | 打开方式 | `"r"` 读 / `"w"` 覆盖写 / `"a"` 追加写 / `"rb"` 二进制读（PDF 用） |  
| `encoding` | 用什么编码解读字节 | `"utf-8"`（99% 情况）/ `"gbk"`（老中文文件） |  
  
> **编码是什么？** 文件在硬盘上存的是一串数字（字节），`encoding` 是“翻译规则”，告诉 Python 用哪本字典把这些数字翻成文字。用错字典 → 乱码或报错。  
  
## 1.3 A 版：最原始的读法（能跑，但危险）  
  
```python  
f = open("data/a.txt", "r", encoding="utf-8")   # ① 打开文件，拿到一个"文件对象"  
content = f.read()                               # ② 一次性把全部内容读成字符串  
f.close()                                        # ③ 关闭文件  
  
print(content)  
print(len(content), "个字")  
```  
  
**逐行讲：**  
  
| 代码 | 含义 |  
|---|---|  
| `open(...)` | 打开文件，返回一个**文件对象**（可以理解成“书的把手”） |  
| `"r"` | read 模式，只读不写 |  
| `f.read()` | 从当前位置读到末尾，返回一个字符串 |  
| `f.close()` | 关闭文件，释放系统资源 |  
  
**文件对象还有这些读法：**  
  
```python  
f.read()        # 全部读成一个字符串  
f.readline()    # 读一行  
f.readlines()   # 读所有行，返回列表 ["第一行\n", "第二行\n"]  
for line in f:  # 一行一行遍历（最省内存，第 6 章细讲）  
    ...  
```  
  
## 1.4 A 版的致命毛病  
  
**② 和 ③ 之间只要出错，`close()` 就永远不会执行。**  
  
```python  
f = open("data/a.txt")  
content = f.read()  
x = 1 / 0          # 炸了！  
f.close()          # 这行永远到不了 → 文件一直被占用  
```  
  
后果：文件句柄泄漏，Windows 上甚至删不掉这个文件。  
  
## 1.5 B 版：`with` 语句（自动关闭）  
  
```python  
with open("data/a.txt", "r", encoding="utf-8") as f:  
    content = f.read()  
  
print(content)  
```  
  
**语法拆解：**  
  
```  
with  open(...)  as  f  :  
     ↑打开文件      ↑给它起名叫 f  
```  
  
`with` 的含义是：**“在这个缩进块里用 f，出了这个块，不管你是正常结束还是中途报错，都自动帮你 close()。”**  
  
> **它的正式名字叫“上下文管理器”**。你现在只需要知道：**凡是打开文件，一律用 `with`，不要手动 close。**  
  
## 1.6 用 `pathlib` 处理路径  
  
字符串拼路径有个坑：Windows 用 `\`，Mac/Linux 用 `/`，写死会跨平台出错。  
  
```python  
from pathlib import Path  
  
p = Path("data") / "a.txt"       # 用 / 拼路径，自动适配系统  
print(p)                          # data/a.txt  
print(p.exists())                 # 文件存在吗？True/False  
print(p.suffix)                   # 扩展名，".txt"  
print(p.stem)                     # 去掉扩展名的文件名，"a"  
print(p.read_text(encoding="utf-8"))    # 一行读完全部内容（open 的语法糖）  
```  
  
| 写法 | 含义 |  
|---|---|  
| `Path("data") / "a.txt"` | 路径拼接，比 `"data" + "/" + "a.txt"` 安全 |  
| `.exists()` | 是否存在，返回布尔 |  
| `.suffix` | 后缀名（带点） |  
| `.read_text(encoding=...)` | 读全部文本，**内部就是 with open**，所以不用担心忘记关 |  
| `.write_text("内容", encoding=...)` | 写文件，同样自动关闭 |  
  
### 本章只需记住 3 句话  
  
1. `open(路径, "r", encoding="utf-8")` 打开，`.read()` 读全部。  
2. **打开文件一定用 `with`**，出了缩进块自动关闭，报错也关。  
3. 路径用 `Path("a") / "b.txt"` 拼，不手写斜杠。  
  
---  
  
# 第 2 章 接住错误：`try / except`  
  
## 2.1 需求  
  
用户可能：传一个不存在的文件、传一个 GBK 编码的文件、传一个没有权限的文件。  
**这些都不能让整个服务崩掉**，而要返回一句人话。  
  
## 2.2 先看会崩成什么样  
  
```python  
with open("不存在.txt", encoding="utf-8") as f:  
    print(f.read())  
```  
  
运行结果：  
  
```  
FileNotFoundError: [Errno 2] No such file or directory: '不存在.txt'  
```  
  
程序直接终止。**如果你在 Web 服务里这么写，这个请求会返回 500，用户看到一堆英文堆栈。**  
  
## 2.3 `try / except` 的基本结构  
  
```python  
try:  
    # 可能会出错的代码  
    with open("不存在.txt", encoding="utf-8") as f:  
        content = f.read()  
except FileNotFoundError:  
    # 出错时走这里，程序不会崩  
    print("文件没找到")  
```  
  
**执行逻辑**：`try` 里的代码正常跑完 → 跳过 `except`；中途抛出 `FileNotFoundError` → 立刻跳到 `except`，继续往下执行。  
  
## 2.4 完整写法：`as e` / `else` / `finally`  
  
```python  
try:  
    with open("data/a.txt", encoding="utf-8") as f:  
        content = f.read()  
except FileNotFoundError as e:          # as e：把异常对象接住，e 就是它  
    print("文件不存在：", e)  
except UnicodeDecodeError as e:         # 可以写多个 except，按顺序匹配  
    print("编码不对：", e)  
except OSError as e:                    # 更宽的一类（权限、磁盘满等）  
    print("系统错误：", e)  
else:  
    print("一切正常，读到", len(content), "个字")   # 只有没出错才执行  
finally:  
    print("不管成功失败，这句一定执行")              # 收尾用  
```  
  
**执行顺序表：**  
  
| 情况 | 走哪些分支 |  
|---|---|  
| 成功 | `try` → `else` → `finally` |  
| 出错 | `try`（中途跳出）→ 对应 `except` → `finally` |  
  
> **`except` 的匹配顺序很重要**：从上往下匹配，**先写具体的，后写宽泛的**。如果把 `except Exception` 写在最前面，后面的都轮不到了。  
  
## 2.5 两个必须知道的坑  
  
**坑 1：不要写“裸 except”**  
  
```python  
try:  
    ...  
except:              # ❌ 什么都接，连 Ctrl+C 都接住，bug 会被吞掉  
    pass  
```  
  
**坑 2：不要把错误吞掉**  
  
```python  
except Exception:  
    pass             # ❌ 出错了啥也不说，以后永远查不出原因  
```  
  
正确做法：**要么处理它（给用户提示），要么往上抛（让上层处理），但一定要留痕（记日志）。**  
  
## 2.6 B 版：自定义异常，让错误有“身份”  
  
```python  
class FileParseError(Exception):  
    """文件读取或解析失败。"""  
  
    def __init__(self, path: str, reason: str):  
        self.path = path  
        self.reason = reason  
        super().__init__(f"{path} 解析失败: {reason}")   # 父类负责生成错误信息  
  
  
def load_text(path: str) -> str:  
    p = Path(path)  
  
    if not p.exists():  
        raise FileParseError(str(p), "文件不存在")  
  
    try:  
        return p.read_text(encoding="utf-8")  
    except UnicodeDecodeError:  
        # 有些老文件是 gbk 编码，换一本字典再试一次  
        try:  
            return p.read_text(encoding="gbk")  
        except UnicodeDecodeError as e:  
            raise FileParseError(str(p), "无法识别文件编码") from e  
    except OSError as e:  
        raise FileParseError(str(p), str(e)) from e  
```  
  
**新知识逐条：**  
  
| 代码 | 含义 |  
|---|---|  
| `class FileParseError(Exception)` | 定义一个自己的异常类型（第一册第 3 章讲过） |  
| `raise FileParseError(...)` | **主动抛出**这个异常 |  
| `from e` | 把“原始异常”挂在新异常上，形成链条。好处：日志里能看到真正的根因 |  
| `return` 在 `try` 里 | 一旦 return，`finally` 仍会执行（这是 Python 的保证） |  
  
**为什么要自定义异常？**  
  
因为 `FileNotFoundError` 太宽泛——它可能来自任何地方。而 `FileParseError` 是你自己的类型，上层可以精准地只接住“文件解析失败”这一类，其他错误照常报。  
  
> 在完整项目里，把 `class FileParseError(Exception)` 改成 `class FileParseError(AppError)`，就能接上第一册的全局异常处理器，返回统一的 JSON。**只改这一处，其他代码一行不动。**  
  
## 2.7 `raise` 与 `except` 的关系（一张图记住）  
  
```  
底层出错（FileNotFoundError）  
      ↓  被 except 接住  
你的代码决定：① 自己处理（返回默认值/给提示）  
             ② 包装后重抛（raise FileParseError(...) from e）  ← 推荐  
             ③ 原样重抛（raise）  
```  
  
### 本章只需记住 3 句话  
  
1. `try` 里放可能出错的代码，`except 具体错误类型 as e` 接住它。  
2. **异常要么处理、要么包装后重抛，绝不能 `pass` 吞掉**。  
3. 自定义异常 = 继承 `Exception`；`raise ... from e` 保留根因。  
  
---  
  
# 第 3 章 正则表达式（`re`）  
  
## 3.1 需求  
  
从网页或文档里洗掉这些东西：  
  
```  
<p>Python 是一门<strong>很好</strong>的语言。</p>  
  
  
  
它有很多用途。  
```  
  
目标结果：  
  
```  
Python 是一门很好的语言。  
它有很多用途。  
```  
  
要干掉：HTML 标签、连续空格、3 个以上的空行。  
  
## 3.2 什么是正则  
  
**正则表达式 = 一种描述“文本长什么样”的小语言。**  
  
它不描述“具体内容”，而描述“模式”。比如：  
  
- `"\d"` 表示“一个数字”  
- `"\s"` 表示“一个空白字符”  
- `"<[^>]+>"` 表示“从 `<` 开始、到 `>` 结束的一小段”  
  
有了模式，就能做三件事：  
  
| 函数 | 作用 | 记忆 |  
|---|---|---|  
| `re.findall(模式, 文本)` | 找出所有匹配的片段，返回列表 | **找** |  
| `re.sub(模式, 替换成什么, 文本)` | 把匹配到的都替换掉 | **换** |  
| `re.split(模式, 文本)` | 按匹配到的位置切开 | **切** |  
  
先背这三个，够用 90% 的场景。  
  
## 3.3 A 版：不用正则，用 `str.replace` 硬干  
  
```python  
text = "<p>Python 是一门<strong>很好</strong>的语言。</p>"  
  
text = text.replace("<p>", "")  
text = text.replace("</p>", "")  
text = text.replace("<strong>", "")  
text = text.replace("</strong>", "")  
```  
  
**毛病一眼可见**：标签有几百种（`<div>`、`<span class="x">`、`<a href="...">`），你不可能一个个列完。而且 `<span class="x">` 这种带属性的，`replace` 根本处理不了。  
  
## 3.4 B 版：一条正则搞定  
  
```python  
import re  
  
text = "<p>Python 是一门<strong>很好</strong>的语言。</p>"  
clean = re.sub(r"<[^>]+>", "", text)  
print(clean)      # Python 是一门很好的语言。  
```  
  
**一行，所有标签全没了。**  
  
## 3.5 逐字符拆解 `<[^>]+>`  
  
这是本册最重要的一个正则，必须彻底看懂：  
  
| 片段 | 含义 | 读法 |  
|---|---|---|  
| `<` | 匹配一个真正的 `<` 字符 | “一个小于号” |  
| `[^>]` | 匹配**除 `>` 以外**的任意一个字符。`[]` 是“字符集合”，`^` 在集合里表示“除了” | “不是大于号的任意字符” |  
| `+` | 前面的东西出现 **1 次或多次** | “一个或多个” |  
| `>` | 匹配一个真正的 `>` | “一个大于号” |  
  
合起来读：**“一个 `<`，跟着一串不是 `>` 的字符，最后是一个 `>`”** —— 这不就是 HTML 标签吗？  
  
> **为什么用 `[^>]` 而不是 `.*`？** 因为 `.*` 是“贪婪”的，会一路吃到最后一个 `>`。对于 `<p>a</p><p>b</p>`，`<.*>` 会把整段都吃掉。而 `[^>]+` 遇到第一个 `>` 就停，**天然不会越界**——这是写正则最实用的一个技巧。  
  
## 3.6 必背的元字符表  
  
| 符号 | 含义 | 例子 |  
|---|---|---|  
| `\d` | 一个数字 | `\d{3}` = 3 个数字 |  
| `\w` | 字母/数字/下划线/汉字 | `\w+` = 一串词 |  
| `\s` | 一个空白（空格/Tab/换行） | `\s+` = 一个或多个空白 |  
| `.` | 任意一个字符（不含换行） | `a.c` 匹配 `abc`、`a1c` |  
| `+` | 前面 1 次或多次 | `a+` = `a`、`aa`、`aaa` |  
| `*` | 前面 0 次或多次 | `a*` = 空、`a`、`aa` |  
| `?` | 前面 0 次或 1 次 | `https?` = `http` 或 `https` |  
| `{n}` / `{n,}` / `{n,m}` | 次数范围 | `\d{3,5}` = 3 到 5 位数字 |  
| `[]` | 字符集合，任选其一 | `[abc]` = a 或 b 或 c |  
| `[^]` | 除了集合里的 | `[^0-9]` = 非数字 |  
| `()` | 分组 | `(ab)+` = `ab` 重复 |  
| `\|` | 或 | `cat\|dog` = cat 或 dog |  
  
## 3.7 `r"..."` 是什么？为什么正则前面都加 `r`  
  
```python  
r"<[^>]+>"      # 前面多一个 r  
```  
  
`r` = **raw string（原始字符串）**，意思是“**里面的反斜杠不要转义**”。  
  
如果不加 `r`，你写 `"\d"`，Python 会先处理转义，`\d` 可能变成别的东西；加了 `r`，`\d` 就原样交给正则引擎。  
  
**规则：写正则永远加 `r`。** 不用理解细节，当成强制习惯。  
  
## 3.8 调试正则的正确姿势  
  
新手写正则几乎不可能一次写对。**标准流程：先用 `findall` 看它到底匹配了什么，再拿去 `sub`。**  
  
```python  
text = '<a href="/x">链接</a><p>段落</p>'  
  
print(re.findall(r"<[^>]+>", text))  
# ['<a href="/x">', '</a>', '<p>', '</p>']   ← 先看这个对不对  
  
print(re.sub(r"<[^>]+>", "", text))  
# 链接段落  
```  
  
> 这个小习惯能帮你省掉 80% 的调试时间：**先验证模式，再做替换。**  
  
## 3.9 常用正则配方（背下来直接用）  
  
| 需求 | 正则 |  
|---|---|  
| 去 HTML 标签 | `r"<[^>]+>"` |  
| 多个空格压成一个 | `r"[ \t\u3000]+"` （`\u3000` 是全角空格） |  
| 3 个以上换行压成 2 个 | `r"\n{3,}"` |  
| 提取网址 | `r"https?://[^\s]+"` |  
| 提取邮箱 | `r"[\w.+-]+@[\w-]+\.[\w.]+"` |  
| 提取 Markdown 标题 | `r"^#{1,6}\s+.+$"`（需配合 `re.MULTILINE`） |  
  
### 本章只需记住 3 句话  
  
1. 正则三件套：`findall` 找、`sub` 换、`split` 切。  
2. 正则一律写 `r"..."`；`<[^>]+>` 是“任意标签”的万能配方。  
3. 写完先 `findall` 验证，再拿去 `sub`。  
  
---  
  
# 第 4 章 文本清洗：把正则组装起来  
  
## 4.1 清洗的四步  
  
一份从网页或 PDF 抓来的文本，通常要过四道工序：  
  
| 步骤 | 目的 | 正则 |  
|---|---|---|  
| 1. 去标签 | 干掉 HTML/XML | `r"<[^>]+>"` |  
| 2. 去代码块 | Markdown 代码围栏对检索是噪音 | `r"```.*?```"`（`.*?` 非贪婪，见下） |  
| 3. 压空白 | 连续空格 → 一个 | `r"[ \t\u3000]+"` |  
| 4. 压空行 | 3 个换行 → 2 个 | `r"\n{3,}"` |  
  
## 4.2 `.*?` 和 `.*` 的区别（重要）  
  
```python  
text = "```python\nprint(1)\n```中间的话```js\nx\n```"  
  
re.findall(r"```.*```", text)     # 贪婪：一口气吃到最后一个 ```，把中间的话也吞了  
re.findall(r"```.*?```", text)    # 非贪婪：吃到最近的 ``` 就停  
```  
  
**`?` 加在 `*` 或 `+` 后面 = “尽量少匹配”**。处理成对符号（引号、括号、代码块）时，几乎都要加 `?`。  
  
## 4.3 完整清洗函数  
  
```python  
import re  
  
# re.compile：把正则"预编译"成对象，重复使用时更快，也更清晰  
HTML_TAG = re.compile(r"<[^>]+>")  
CODE_BLOCK = re.compile(r"```.*?```", re.DOTALL)   # DOTALL：让 . 也能匹配换行  
SPACES = re.compile(r"[ \t\u3000]+")  
BLANK_LINES = re.compile(r"\n{3,}")  
  
  
def clean_text(text: str) -> str:  
    """清洗文本：去标签、去代码块、压空白、压空行。"""  
    text = HTML_TAG.sub("", text)        # 去 HTML 标签  
    text = CODE_BLOCK.sub("", text)      # 去 Markdown 代码块  
    text = SPACES.sub(" ", text)         # 连续空格 → 一个  
    text = BLANK_LINES.sub("\n\n", text) # 3+ 换行 → 2 个  
    return text.strip()                  # 去掉首尾空白  
```  
  
**逐行讲：**  
  
| 代码 | 含义 |  
|---|---|  
| `re.compile(模式)` | 把正则编译成一个对象，之后用 `对象.sub(...)` |  
| `re.DOTALL` | 让 `.` 也能匹配换行符（默认不匹配）。用于跨行的代码块 |  
| `HTML_TAG.sub("", text)` | 把匹配到的全部替换成空字符串 = 删除 |  
| `.strip()` | 去掉字符串首尾的空格和换行（字符串自带方法，不用 import） |  
  
## 4.4 动手验证  
  
~~~python  
raw = """  
<h1>标题</h1>  
<p>正文   有很多    空格</p>  
  
  
  
```python  
print("这段代码应该被删掉")  
```  
结尾  
"""  
  
print(clean_text(raw))  
~~~  
  
输出：  
  
```  
标题  
正文 有很多 空格  
  
结尾  
```  
  
### 本章只需记住 3 句话  
  
1. 清洗 = 一串 `re.sub` 按顺序过，每步只干一件事。  
2. `.*?` 非贪婪，处理成对符号必加 `?`。  
3. `re.compile` 预编译，跨行匹配加 `re.DOTALL`。  
  
---  
  
# 第 5 章 文本分块（chunk）  
  
## 5.1 为什么要分块  
  
两个硬约束：  
  
1. **模型有上下文长度上限**，不可能一次塞 10 万字。  
2. **检索要精准**：用户问“部署方式”，系统应该只把讲部署的那几段送去给模型，而不是整本书。  
  
所以要把长文切成 300~800 字的小块（chunk），每块单独存、单独检索。  
  
## 5.2 先设计数据结构（重要方法）  
  
写代码前，**先在纸上画出数据长什么样**：  
  
```python  
chunk = {  
    "index": 0,                    # 第几块  
    "text": "Python 是一门...",     # 内容  
    "char_count": 42,              # 字数  
}  
```  
  
一个文档 → 一个 `list`，里面装多个 `dict`：  
  
```python  
chunks = [chunk0, chunk1, chunk2, ...]  
```  
  
> **这就是“拿到需求就能写”的第一步**：先定数据结构。`list[dict]` 这个形状一定，后面遍历、存储、检索都顺了。  
  
## 5.3 A 版：定长硬切  
  
```python  
def split_fixed(text: str, size: int = 300, overlap: int = 50) -> list[dict]:  
    chunks = []  
    start = 0  
    index = 0  
  
    while start < len(text):                     # 只要还没切到头  
        piece = text[start : start + size]       # ① 切片  
        chunks.append({  
            "index": index,  
            "text": piece,  
            "char_count": len(piece),  
        })  
        start += size - overlap                  # ② 往前挪，但留 overlap 个字重叠  
        index += 1  
  
    return chunks  
```  
  
**三个新知识：**  
  
| 代码 | 含义 |  
|---|---|  
| `text[start : start + size]` | **切片**：取从 start 到 start+size（不含）的一段 |  
| `list[dict]` | 类型注解：一个列表，元素是字典 |  
| `start += size - overlap` | 每次前进 `size-overlap` 步，这样相邻两块有 `overlap` 个字重合 |  
  
**为什么要重叠（overlap）？**  
一句话可能正好被切在两块的交界处。留 50 字重叠，能保证**每个完整句子至少在某一块里是完整的**。  
  
**动手看效果：**  
  
```python  
text = "一二三四五六七八九十" * 10      # 100 个字  
chunks = split_fixed(text, size=20, overlap=5)  
for c in chunks:  
    print(c["index"], c["text"])  
```  
  
## 5.4 A 版的毛病  
  
**硬切不看句子边界**，可能切出：  
  
```  
..."Python 是一门很" | "好的语言，适合..."  
```  
  
前半块以“很”结尾，后半块以“好的”开头——**语义被劈开了**，检索质量直接下降。  
  
## 5.5 B 版：先分句，再按句子装箱  
  
思路换成：“**句子是原子，不能拆**；把句子一个个往箱子里装，装不下就换箱子。”  
  
```python  
import re  
  
# 在中文/英文句末标点后面切开（?<= 是"零宽断言"，见下）  
SENTENCE_SPLIT = re.compile(r"(?<=[。！？!?\.])")  
  
  
def split_sentences(text: str) -> list[str]:  
    """把文本切成句子列表。"""  
    parts = SENTENCE_SPLIT.split(text)  
    return [p.strip() for p in parts if p.strip()]   # 去掉空白项  
  
  
def split_text(text: str, size: int = 300, overlap: int = 50) -> list[dict]:  
    """按句子边界切块，相邻块保留 overlap 个字符重叠。"""  
    sentences = split_sentences(text)  
  
    chunks: list[dict] = []  
    buf = ""                     # 当前正在装的箱子  
  
    for s in sentences:  
        if len(buf) + len(s) <= size:      # 装得下 → 继续装  
            buf += s  
        else:                              # 装不下 → 封箱，开新箱  
            if buf:  
                chunks.append({"index": len(chunks), "text": buf, "char_count": len(buf)})  
            # 新箱子先放入上一箱的尾巴（重叠部分），再放新句子  
            buf = (buf[-overlap:] if overlap else "") + s  
  
    if buf:                                # 最后别忘了把没封的箱子收走  
        chunks.append({"index": len(chunks), "text": buf, "char_count": len(buf)})  
  
    return chunks  
```  
  
**逐段讲：**  
  
| 代码 | 含义 |  
|---|---|  
| `re.compile(r"(?<=[。！？!?\.])")` | `(?<=X)` 是**零宽断言**：只在“某个位置”匹配，不消耗字符。这里表示“在句号等的**后面**切一刀” |  
| `SENTENCE_SPLIT.split(text)` | 按这个位置切开，返回句子列表 |  
| `[p.strip() for p in parts if p.strip()]` | **列表推导式**：遍历、清洗、过滤三合一。等价于一个 for 循环 + append |  
| `buf[-overlap:]` | 切片取最后 overlap 个字符（负数索引 = 从末尾数） |  
| `if len(buf) + len(s) <= size` | 判断“再加这句话会不会超” |  
| `if buf:` 在最后 | 循环结束时箱子可能还有内容，必须补收 |  
  
**关于零宽断言 `(?<=...)`**：你现在不用会写，**当成配方记住**——“在 A 后面切一刀”就是 `(?<=A)`。  
  
## 5.6 一个已知的边界情况  
  
如果**单句话本身就超过 size**（比如一整段没有标点），B 版会切出一个超长块。真实项目里的处理是：对超长句再退回定长硬切。  
  
**这告诉我们一个方法论**：先写主干逻辑，再列“已知不完美的地方”，后面按需补。不要一开始追求完美。  
  
### 本章只需记住 3 句话  
  
1. 分块 = **先定数据结构（`list[dict]`），再写循环**。  
2. 硬切简单但破句；**按句子装箱 + overlap 重叠**才不丢语义。  
3. `(?<=A)` = 在 A 后面切；`buf[-n:]` = 取最后 n 个字符。  
  
---  
  
# 第 6 章 生成器：大文件不能一次读进内存  
  
## 6.1 需求  
  
用户上传一个 500MB 的日志文件。`f.read()` 会一次性把 500MB 塞进内存，服务器直接 OOM（内存爆掉）。  
  
## 6.2 A 版：一次全读  
  
```python  
with open("big.log", encoding="utf-8") as f:  
    lines = f.readlines()          # ❌ 500MB 全进内存，变成一个大列表  
  
for line in lines:  
    process(line)  
```  
  
**毛病**：内存占用 = 文件大小。文件越大越危险。  
  
## 6.3 B 版：`yield` 逐个产出  
  
```python  
def read_lines(path: str):  
    """逐行读取文件，每次只产出 1 行。"""  
    with open(path, encoding="utf-8") as f:  
        for line in f:                 # 文件对象本身就是"一行一行读"的  
            yield line.rstrip("\n")    # 产出一行，然后暂停  
  
  
for line in read_lines("big.log"):  
    process(line)  
```  
  
**内存占用永远只有 1 行**，跟文件多大无关。  
  
## 6.4 `yield` 到底是什么意思  
  
对比一下就懂了：  
  
```python  
def f_return():  
    return [1, 2, 3]        # 一次性造好整个列表，全部返回  
  
def f_yield():  
    yield 1                 # 产出一个，暂停  
    yield 2                 # 下次被要的时候，从这继续  
    yield 3  
```  
  
| | `return` | `yield` |  
|---|---|---|  
| 返回什么 | 一个值（或整个列表） | 一个**生成器对象** |  
| 执行方式 | 一次跑完 | **用一次跑一步，跑完暂停，记住位置** |  
| 内存 | 全部结果同时在内存 | 一次只有一个值 |  
| 用几次 | 一次 | 可以 `for` 循环反复取 |  
  
**比喻**：`return` 是“把一整锅饭端上来”；`yield` 是“盛一碗给你，你吃完了我再盛下一碗”。  
  
**关键理解**：  
  
```python  
gen = f_yield()        # 此时函数体内的代码一行都没执行！  
print(gen)             # <generator object ...>  
next(gen)              # 1    ← 执行到第一个 yield，停住  
next(gen)              # 2    ← 从上次停的地方继续  
```  
  
`for` 循环会自动帮你反复调 `next()`，直到取完。  
  
## 6.5 生成器在本项目里的三个位置  
  
| 位置 | 用生成器做什么 |  
|---|---|  
| 读大文件 | 逐行 yield，不占内存 |  
| 批量嵌入（第 3 阶段） | 每次 yield 一批 chunk 去调 API，避免一次发太多 |  
| 流式返回答案（第 8 阶段） | LLM 每吐一个字就 yield 给前端，实现打字机效果 |  
  
> **记住这个模式**：**“数据量大”或“要边算边给”的地方，就是 `yield` 出场的地方。**  
  
### 本章只需记住 3 句话  
  
1. `yield` = 产出一个值就暂停，下次从暂停处继续。  
2. 生成器内存只占一份，适合大文件、大批量、流式输出。  
3. `for line in f` 本身就是逐行读，比 `readlines()` 省内存。  
  
---  
  
# 第 7 章 组装成模块  
  
## 7.1 `app/services/loader.py`  
  
```python  
"""文件加载模块：把各种来源的文件读成纯文本字符串。"""  
from pathlib import Path  
  
from app.core.logging import get_logger  
  
logger = get_logger(__name__)  
  
# 支持的扩展名（set 查得快，O(1)）  
SUPPORTED_SUFFIXES = {".txt", ".md", ".markdown"}  
  
  
class FileParseError(Exception):  
    """文件读取或解析失败。项目里可改为继承 AppError。"""  
  
    def __init__(self, path: str, reason: str):  
        self.path = path  
        self.reason = reason  
        super().__init__(f"{path} 解析失败: {reason}")  
  
  
def load_text(path: str | Path) -> str:  
    """读取文本文件，返回字符串。  
  
    参数:  
        path: 文件路径，字符串或 Path 都行  
    返回:  
        文件的全部文本内容  
    异常:  
        FileParseError: 文件不存在、格式不支持、编码无法识别  
    """  
    p = Path(path)  
  
    # ---- 前置检查：把"可预期的问题"提前拦住，错误信息更友好 ----  
    if not p.exists():  
        raise FileParseError(str(p), "文件不存在")  
    if p.suffix.lower() not in SUPPORTED_SUFFIXES:  
        raise FileParseError(str(p), f"不支持的格式 {p.suffix}，仅支持 {SUPPORTED_SUFFIXES}")  
  
    # ---- 读取：先 utf-8，失败再试 gbk ----  
    try:  
        text = p.read_text(encoding="utf-8")  
    except UnicodeDecodeError:  
        logger.warning("utf-8 解码失败，改用 gbk 重试: %s", p)  
        try:  
            text = p.read_text(encoding="gbk")  
        except UnicodeDecodeError as e:  
            raise FileParseError(str(p), "无法识别文件编码") from e  
    except OSError as e:  
        raise FileParseError(str(p), str(e)) from e  
  
    logger.info("加载成功 | file=%s | chars=%d", p.name, len(text))  
    return text  
  
  
def iter_lines(path: str | Path):  
    """逐行读取大文件（生成器），适合超大文件。"""  
    p = Path(path)  
    if not p.exists():  
        raise FileParseError(str(p), "文件不存在")  
    with p.open(encoding="utf-8", errors="ignore") as f:  
        for line in f:  
            line = line.rstrip("\n")  
            if line:  
                yield line  
```  
  
**这段代码用到的、你已经学过的全部东西：**  
  
| 知识点 | 出现位置 |  
|---|---|  
| 文件读写 | `p.read_text()`、`p.open()` |  
| `with` | `iter_lines` 里 |  
| 异常处理 | 三层 `try/except` |  
| 自定义异常 | `FileParseError` |  
| 数据结构 | `SUPPORTED_SUFFIXES` 用 set |  
| 生成器 | `iter_lines` 里 `yield` |  
| 日志 | `logger.info / warning` |  
| 类型注解 | `str \| Path`、`-> str` |  
| 字符串方法 | `.lower()`、`.rstrip()` |  
  
## 7.2 `app/services/splitter.py`  
  
```python  
"""文本清洗与分块模块。"""  
import re  
  
from app.core.logging import get_logger  
  
logger = get_logger(__name__)  
  
# 预编译正则（模块加载时编译一次，后续复用）  
HTML_TAG = re.compile(r"<[^>]+>")  
CODE_BLOCK = re.compile(r"```.*?```", re.DOTALL)  
SPACES = re.compile(r"[ \t\u3000]+")  
BLANK_LINES = re.compile(r"\n{3,}")  
SENTENCE_SPLIT = re.compile(r"(?<=[。！？!?\.])")  
  
  
def clean_text(text: str) -> str:  
    """清洗文本：去 HTML 标签、去代码块、压缩空白和空行。"""  
    text = HTML_TAG.sub("", text)  
    text = CODE_BLOCK.sub("", text)  
    text = SPACES.sub(" ", text)  
    text = BLANK_LINES.sub("\n\n", text)  
    return text.strip()  
  
  
def split_sentences(text: str) -> list[str]:  
    """按句末标点切句，过滤空白项。"""  
    return [p.strip() for p in SENTENCE_SPLIT.split(text) if p.strip()]  
  
  
def split_text(text: str, size: int = 300, overlap: int = 50) -> list[dict]:  
    """把长文本切成带重叠的块。  
  
    参数:  
        text: 原始文本  
        size: 每块目标最大字数  
        overlap: 相邻块的重叠字数  
    返回:  
        [{"index": 0, "text": "...", "char_count": 42}, ...]  
    """  
    if not text:  
        return []  
    if overlap >= size:  
        raise ValueError("overlap 必须小于 size")  
  
    sentences = split_sentences(text)  
    chunks: list[dict] = []  
    buf = ""  
  
    for s in sentences:  
        if len(buf) + len(s) <= size:  
            buf += s  
        else:  
            if buf:  
                chunks.append({"index": len(chunks), "text": buf, "char_count": len(buf)})  
            buf = (buf[-overlap:] if overlap else "") + s  
  
    if buf:  
        chunks.append({"index": len(chunks), "text": buf, "char_count": len(buf)})  
  
    logger.info("分块完成 | 原文 %d 字 | 共 %d 块", len(text), len(chunks))  
    return chunks  
  
  
def process(text: str, size: int = 300, overlap: int = 50) -> list[dict]:  
    """一步到位：清洗 + 分块。这是给上层用的"门面函数"。"""  
    return split_text(clean_text(text), size=size, overlap=overlap)  
```  
  
> **`process()` 这个小函数体现了“门面模式”**：上层不需要知道“先 clean 再 split”的顺序，只调一个 `process()` 就行。**以后需求变了（比如加一步去页眉），只改这里，上层不动。**  
  
---  
  
# 第 8 章 给它写测试  
  
## 8.1 `pytest` 三条规则  
  
1. 文件名以 `test_` 开头  
2. 函数名以 `test_` 开头  
3. 用 `assert` 断言（不用 `unittest` 那些类）  
  
```python  
def test_add():  
    assert 1 + 1 == 2  
```  
  
运行：`pytest -v`  
  
## 8.2 `tmp_path`：pytest 白送的临时目录  
  
测试文件读写时，不能写死真实路径。`tmp_path` 是 pytest 自动创建的**临时文件夹**，每个测试一个，跑完自动清理。  
  
```python  
def test_load_text_ok(tmp_path):  
    # tmp_path 是一个 Path 对象，指向一个空的临时目录  
    p = tmp_path / "a.txt"  
    p.write_text("你好世界", encoding="utf-8")  
  
    assert load_text(p) == "你好世界"  
```  
  
## 8.3 `pytest.raises`：断言“应该抛异常”  
  
```python  
import pytest  
  
def test_missing_file():  
    with pytest.raises(FileParseError):     # 断言：这段代码必须抛 FileParseError  
        load_text("根本不存在的文件.txt")  
```  
  
如果它没抛异常，测试就失败。**这用来测“错误分支”，很多人会漏掉。**  
  
## 8.4 完整测试文件 `tests/test_splitter.py`  
  
```python  
"""文本清洗与分块的单元测试。"""  
import pytest  
  
from app.services.loader import FileParseError, load_text  
from app.services.splitter import clean_text, process, split_text  
  
  
# ---------------- 清洗 ----------------  
def test_clean_removes_html_tags():  
    assert clean_text("<p>你好</p>") == "你好"  
  
  
def test_clean_compresses_blank_lines():  
    assert clean_text("a\n\n\n\nb") == "a\n\nb"  
  
  
# ---------------- 分块 ----------------  
def test_split_empty_returns_empty_list():  
    assert split_text("") == []  
  
  
def test_split_single_chunk_when_short():  
    chunks = split_text("很短的一句话。", size=300)  
    assert len(chunks) == 1  
    assert chunks[0]["index"] == 0  
  
  
def test_split_respects_size_roughly():  
    text = "这是一句话。" * 100          # 600 字  
    chunks = split_text(text, size=100, overlap=20)  
    # 每块不应明显超过 size（允许单句超出，但这里句子短，所以能卡住）  
    assert all(c["char_count"] <= 120 for c in chunks)  
    assert len(chunks) > 1  
  
  
def test_split_has_overlap():  
    text = "一二三四五六七八九十" * 20  
    chunks = split_text(text, size=40, overlap=10)  
    # 前一块的后 10 个字，应该出现在后一块的开头  
    assert chunks[0]["text"][-10:] in chunks[1]["text"]  
  
  
def test_overlap_must_be_less_than_size():  
    with pytest.raises(ValueError):  
        split_text("abc", size=10, overlap=10)  
  
  
# ---------------- 加载 ----------------  
def test_load_missing_file():  
    with pytest.raises(FileParseError):  
        load_text("no_such_file.txt")  
  
  
def test_load_unsupported_suffix(tmp_path):  
    p = tmp_path / "a.pdf"  
    p.write_text("x", encoding="utf-8")  
    with pytest.raises(FileParseError):  
        load_text(p)  
  
  
# ---------------- 端到端 ----------------  
def test_process_clean_and_split(tmp_path):  
    p = tmp_path / "doc.md"  
    p.write_text("<h1>标题</h1>" + "正文内容。" * 50, encoding="utf-8")  
  
    result = process(load_text(p), size=60, overlap=10)  
  
    assert len(result) >= 1  
    assert "<h1>" not in result[0]["text"]      # 标签已被清掉  
```  
  
**运行：**  
  
```bash  
pytest -v --cov=app --cov-report=term-missing  
```  
  
**一个重要的测试思路：只测“行为”，不测“实现”。**  
比如 `test_split_has_overlap` 不关心你怎么切，只验证“重叠确实存在”。这样以后你优化算法，测试照样过。  
  
---  
  
# 第 9 章 自测（8 题）  
  
1. `with open(...) as f:` 相比手动 `close()` 解决了什么问题？  
2. `except FileNotFoundError as e` 里的 `as e` 是什么？  
3. 为什么 `except Exception: pass` 是坏习惯？  
4. `r"<[^>]+>"` 里 `[^>]` 和 `>` 分别是什么意思？  
5. `.*` 和 `.*?` 有什么区别？什么时候必须用后者？  
6. 写正则时为什么前面要加 `r`？  
7. `yield` 和 `return` 最本质的区别是什么？  
8. 分块为什么要留 `overlap`？  
  
<details>  
<summary>答案要点（先自己答，再点开）</summary>  
  
1. 出了缩进块自动关闭，即使中途报错也会执行，避免文件句柄泄漏。  
2. 把捕获到的异常对象绑定到变量 `e`，用于读取错误信息。  
3. 它会吞掉所有错误（包括程序中断），bug 无迹可查，之后完全无法排查。  
4. `[^>]` = 除了 `>` 以外的任意字符；`>` 就是字面的大于号。合起来匹配一个标签。  
5. `.*` 贪婪，会吃到最后一个可能的结束符；`.*?` 非贪婪，吃到最近的。处理成对符号（代码块、引号）时必须用 `?`。  
6. `r` 是原始字符串，禁止 Python 处理反斜杠转义，把 `\d`、`\s` 原样交给正则引擎。  
7. `return` 一次返回全部结果并结束函数；`yield` 产出一个值就暂停，下次从暂停处继续（惰性、省内存）。  
8. 防止一句话正好被切在块边界而语义断裂，保证每个完整句子至少出现在某一块中。  
  
</details>  
  
---  
  
# 附录 A 本册知识地图  
  
| 知识点 | 本册落点 |  
|---|---|  
| 文件读写 | 第 1 章 `open` / `with` / `Path` |  
| 异常处理 | 第 2 章 `try/except/else/finally` + 自定义异常 |  
| 正则 | 第 3、4 章 `findall/sub/split` + 元字符 |  
| 数据结构 | 第 5 章 `list[dict]`、set 查重 |  
| 高级特性 | 第 6 章 生成器 `yield`；第 5 章 列表推导式 |  
| 函数结构 | 第 5 章 默认参数、关键字参数 |  
| OOP | 第 2 章 自定义异常继承 |  
| 日志 | 第 7 章 全流程打点 |  
| 测试调试 | 第 8 章 pytest + tmp_path + raises |  
  
---  
  
# 附录 B 完整文件清单  
  
```  
insight-agent/  
├── app/  
│   ├── core/  
│   │   └── logging.py          # 第一册：get_logger  
│   └── services/  
│       ├── __init__.py         # 空文件，标记这是一个包  
│       ├── loader.py           # 本册 7.1  
│       └── splitter.py         # 本册 7.2  
└── tests/  
    ├── __init__.py  
    └── test_splitter.py        # 本册 8.4  
```  
  
**运行顺序：**  
  
```bash  
# 1) 建目录  
mkdir -p app/services tests  
  
# 2) 粘贴代码，创建空的 __init__.py  
touch app/__init__.py app/services/__init__.py tests/__init__.py  
  
# 3) 跑测试  
pytest -v  
  
# 4) 手动试一下  
python -c "  
from app.services.splitter import process  
print(process('<p>你好</p>这是一句测试。' * 20, size=50, overlap=10))  
"  
```  
  
---  
  
# 下一册预告  
  
**第三册：数据库与检索**。会用到：  
  
- `SQLAlchemy`：把 Python 类映射成数据库表（OOP 真正派上用场）  
- `Session`：一次数据库对话的开关  
- 增删改查四件套 + 事务回滚  
- 把上一册切好的 chunk 存进去，再按关键词查出来  
  
> 建议：**先把第 7 章的两个 `.py` 文件敲出来，跑第 8 章的测试，全绿了再往下走。**  
  
---  
  
**文档结束** · 版本 v1.0 · 2026-10-21  
