## 标题
# 一级标题
## 二级标题
### 三级标题
#### 四级标题
##### 五级标题
###### 六级标题

## 列表
- 1.有序列表
- 无序列表

## 连接
[百度连接](https://www.baidu.com)
![](图片的网络连接)

## 引用
>引用内容

## 粗/斜体
*斜体*
**粗体**

## 代办清单
- [x] 已完成项目1
  - [x] 已完成事项
  - [ ] 代办事项
- [ ] 代办项目2

## 表格
| Tables        | Are           | Cool   |
| ------------- |:-------------:| ------:|
| col 3 is      | zhongxinduiqi |youduiqi|
| col 2 is      | centered      |   $12  |
| zebra stripes | are neat      |    $1  |

## 绘制流程图

```
graph TD
    A[矩形图] -->B(圆角矩形图)
    B --> C{菱形图}
    C -->|关系one| D[Laptop]
    C -->|two| E[iPhone]
    C -->|three| F[Car]
```
## 序列图
```
sequenceDiagram
    loop every day
        Alice->John: Hello John, how are you?
        John->Alice: Great!
    end
```

## 甘特图
```
gantt
dateFormat YYYY-MM-DD
title 产品计划表
section 初期阶段
明确需求: 2016-03-01, 10d
section 中期阶段
跟进开发: 2016-03-11, 15d
section 后期阶段
走查测试: 2016-03-20, 9d
```
## 数学公式

inline math: `$\dfrac{
\tfrac{1}{2}[1-(\tfrac{1}{2})^n] }{
1-\tfrac{1}{2} } = s_n$`.

## 代码引用
`helloworld`

```
第一段
第二段
第三段
```

## 格式转换

### html
- 1.MdCharm
	选择 ‘File’, ‘Export to…’，勾选 ‘HTML’, 点击 ‘Browser…’ 选择导出目录并输入导出的文件名，点击 ‘OK’，即可将当前的 Markdown 文档转换为 HTML 文档。如果不满意 HTML 文档的样式，可以在设置中自定义 CSS。
- 2.Pandoc
	参考 Installing 安装 Pandoc。
	打开命令行，进入文档所在目录：`cd /path/to/file/`

	执行下面的命令，将 Markdown 转换为 HTML：'pandoc -o hello.html hello.md'

	默认的转换，只是将 Markdown 内容转换为 HTML 标签，所以只能看到浏览器的默认样式。

	可以执行下面的命令，为导出的 HTML 添加自定义样式：

	`pandoc -o hello.html -c style.css hello.md`

```
	style.css 仍然是以 <link> 的方式关联到 HTML 文档中的，所以在发布的时候需要将 CSS 一同发布出去。
```

### 转换为 PDF 文档
- 1.MdCharm
	与导出 HTML 文档类似，选择 ‘File’, ‘Export to…’，勾选 ‘PDF’, 点击 ‘Browser…’ 选择导出目录并输入导出的文件名，点击 ‘OK’，即可将当前的 Markdown 文档转换为 PDF 文档。

	如果不满意 PDF 文档的样式，可以在设置中自定义 CSS。

- 2.Pandoc
	使用 Pandoc 导出 PDF 文档，需要先安装某个 LaTeX 引擎（参考 Creating a PDF）。然后执行命令：

	`pandoc -o hello.pdf hello.md`

	当然，也可以通过` -c style.css `来指定样式文件。

- 3.Chrome
	在将 Markdown 转换为 HTML 文档 之后，可以通过 Chrome 浏览器 打开它。选择 ‘打印’（Ctrl+P），然后更改 ‘目标打印机’ 为 ‘另存为 PDF’，再进行一些设置后，即可保存为 PDF 文档。

#### 转换为 Word 文档
- 1.复制粘贴

	在导出为 HTML 文档之后，可以（在浏览器中）手动复制 HTML 页面的内容，然后粘贴到 Word 文档中，保存即可。

- 2.Pandoc
	执行下面的命令，即可将 Markdown 文档转换为 Word 文档：

	`pandoc -o hello.docx hello.md`

