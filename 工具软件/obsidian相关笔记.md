# 插件
## Custom Attachment location

**核心功能**：规范化附件的名字和路径，自定义图片名字，自动转移图片到对应文件夹。

## Consistent Attachments and Links

**核心功能**：将图片wiki链接转换成标准的 markdown 相对路径链接，让笔记更有普适性。

## 多平台文章&消息同步

[笔记同步助手教程](https://www.bijitongbu.site/tutorials/)

# Clearing Unused Images

**核心功能**：清除没有被笔记链接的图片。

# 常见问题解决

## obsidian的图片显示不兼容Typora

### 单个笔记整理图片

一共只需要用到两个插件：Custom Attachment location 和 Consistent Attachments and Links。

1. 先用 Custom Attachment location 插件设置图片的名字和储存位置；然后收集当前笔记的附件。
	打开obsidian命令面板输入：
	`Custom Attachment Location: Collect attachments in current note`

2. 用 Consistent Attachments and Links 插件将 wiki 图片链接转换为标准的 markdown 链接  
	`Consistent Attachments and Links: Replace All Wiki Embeds with Markdown Embeds in Current note`
	此时变成了这样：叹号+中括号+小括号（小括号内只包含了附件的名字）：  
	`![test 1-20250317165307.png](test%201-20250317165307.png)`

3. 最后用 Consistent Attachments and Links 插件将链接转换为相对路径
	`Consistent Attachments and Links: Convert All Embed Paths to Relative in Current Note  `
	最后变成了这样：叹号+中括号+很长的小括号（小括号内包含了附件的路径）：  
	`![test 1-20250317165307.png](assets/test%201/test%201-20250317165307.png)`

### 批量整理笔记图片

与单个文件的转换类似，建议先尝试整理单个笔记确保没问题了再批量整理，批量操作前记得备份。

依然只需要用到两个插件：Custom Attachment location 和 Consistent Attachments and Links。

1. 用 Custom Attachment location 插件设置图片的名字和储存位置；然后收集整个库的附件。  
    `Custom Attachment Location: Collect attachments in entire vault`  
    
2. 用 Consistent Attachments and Links 插件将 wiki 图片链接转换为标准的 markdown 链接  
    `Consistent Attachments and Links: Replace All Wiki Embeds with Markdown Embeds`  
    
3. 最后用 Consistent Attachments and Links 插件将链接转换为相对路径  
    `Consistent Attachments and Links: Convert All Embed Paths to Relative`

