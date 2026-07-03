# Markdown 中插入空格
```markdown
&emsp;          全角空格
&ensp;          半角空格
&nbsp;          不换行空格
&thinsp;        细小空格
```
例1：

	我&emsp;他

显示效果为

---

**我&ensp;他**

---

例2：

	我&emsp;他

显示效果为

---

**我&ensp;他**

---

例3：

	我&nbsp;他

显示效果为

---

**我&nbsp;他**

---

例4：

	我&thinsp;他

显示效果为

---

**我&thinsp;他**

---

# Markdown 中插入空行
```markdown
<br />        方式1
&nbsp;        方式2
```

例1：

	我<br /> 
	他

显示效果为

---

**我<br /> 
他**

---

例2：

	我
	&nbsp;
	他

显示效果为

---

 **我
&nbsp;
他**

---

# Markdown 中文字左、中、右对齐
```html
<center>Markdown</center>

<p align = "left">Markdown</p>

<p align="right" >Markdown</p>

```

显示效果为

---

<center>Markdown</center>

<p align = "left">Markdown</p>

<p align="right">Markdown</p>

---

# 上下角标

```markdown
H<sub>2</sub>O
x<sup>2</sup>+y<sup>2</sup>
```

显示效果为

---

H<sub>2</sub>O

x<sup>2</sup>+y<sup>2</sup>

# 设置字体

## 颜色文字

```markdown
<font color=red>我是红色</font>
```

效果如下：

<font color=red>我是红色</font>

## 设置字体

```markdown
```

# 跳转

## <span id="jump">文章内跳转</span>

1. 定义一个锚点 ` <span id="jump">跳转到的地方</span>`
2. 跳转语法 ` [点击跳转`](#jump)

示例：

跳转至[文章内跳转](#jump)

## 文章间跳转

### md写法

[跳转至git_note](../工具软件/Git/git_note.md#撤销与删除)

```html
[跳转至gitnote](./git_note,md#撤销与删除)
```

### html写法

```md
<a href="https://github.com/BackMountainDevil/The-C-Programming-Language#the-c-programming-language">返回目录</a>
```

<a href="https://github.com/BackMountainDevil/The-C-Programming-Language#the-c-programming-language">The-C-Programming-Language</a>

# 单元格合并

```html
<table>
    <tr>
        <td>类别</td>
        <td>名称</td>
    </tr>
    <tr>
        <td rowspan="2">颜色</td>
        <td>红色</td>
    </tr>
    <tr>
        <td>黄色</td>
    </tr>
    <tr>
        <td colspan="2">姓氏</td>
    </tr>
    <tr>
        <td>王</td>
        <td>张</td>
    </tr>
</table>
```

效果：

<table>
    <tr>
        <td>类别</td>
        <td>名称</td>
    </tr>
    <tr>
        <td rowspan="2">颜色</td>
        <td>红色</td>
    </tr>
    <tr>
        <td>黄色</td>
    </tr>
    <tr>
        <td colspan="2">姓氏</td>
    </tr>
    <tr>
        <td>王</td>
        <td>张</td>
    </tr>
</table>
# html加粗

```bash
<b>a</b>
```

效果：

未加粗：a

加粗：<b>a</b>

# 插入音频

## 网易云音乐外链

```html
<iframe frameborder="no" border="0" marginwidth="0" marginheight="0" width=330 height=86 src="//music.163.com/outchain/player?type=2&id=33367876&auto=1&height=66"></iframe>
```

## MP3音频文件

```html
<audio src="mp3文件链接地址"></audio>
```

# 插入视频

```html
<video src="视频链接"></video>
```

```html
<iframe height=498 width=510 src="视频链接">
```
