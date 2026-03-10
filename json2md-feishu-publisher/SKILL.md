---
name: json2md-feishu-publisher
description: 将json格式文档转为更加便于展示和阅读的markdown文档，并将markdown文档同步发布在飞书文档中。当用户提出"将数据在飞书上展示"、"将数据发布到飞书"时使用。
version: 1.1.0
metadata:
  author: xuzhaopu
---
# Json2md Feishu Publisher
将json格式文档转为更加便于展示和阅读的markdown格式文档，并将markdown文档同步发布在飞书文档中。

## 步骤
Step1: 理解json文档的结构，生成json转markdown的python脚本；
Step2: 执行python脚本，将json文档转为同名的markdown文档；
Step3: 确定分享邮箱，使用feishu-mcp，创建一个飞书云文档展示markdown文档内容，并将该文档的编辑权限分享给指定用户。

### Step1:
1. 创建以时间戳命名的raw文件夹，并将其路径保存为变量供后续步骤使用
```bash
RAW_DIR="raw_$(date +%Y%m%d_%H%M%S)"
mkdir "$RAW_DIR"
echo "Output directory: $RAW_DIR"
```
2. 读取json文件内容，理解文档结构；
3. 生成用于转markdown的python脚本，要求如下：
  * 脚本名称为json-to-md.py
  * 脚本的输入为json文件，输出为md文件
  * json中的array of object需要用markdown的表格展示
  * 如果数据中有图片，采用`![](图片URL)`的标准markdown图片语法用于显示
  * python脚本保存在 `$RAW_DIR` 目录中；
**注意**：
  * 单元格内容中的中文引号、管道符等特殊字符，必须进行转义，确保单元格内容不会影响表格结构；
  * 确保使用UTF-8编码

### Step2:
在 Step1 创建的 `$RAW_DIR` 目录中执行转换，将 `<filename>` 替换为实际的 json 文件名（不含扩展名）：
```bash
python "$RAW_DIR/json-to-md.py" .json "$RAW_DIR/.md"
```
markdown 文档保存在 `$RAW_DIR` 目录中，与 json 文件同名。

### Step3:
#### 3.0 确定分享邮箱
按以下优先级确定要分享的邮箱地址：

1. **用户在对话中明确提供了邮箱**（如"分享给 xxx@xxx.com"）→ 直接使用该邮箱；
2. **读取环境变量 `FEISHU_SHARE_EMAIL`**：
   ```bash
   echo $FEISHU_SHARE_EMAIL
   ```
   如果输出非空 → 使用该邮箱；
3. **以上均无** → 在继续之前，向用户提问：
   > "请问需要将飞书文档的编辑权限分享给哪个邮箱？"
   等待用户回复后再继续。

#### 3.1 创建并分享飞书文档
1. 使用feishu-mcp，创建一个飞书云文档展示markdown文档内容，根据文档内容命名该飞书文档；
2. 将该文档的编辑权限分享给上一步确定的邮箱对应的用户。
