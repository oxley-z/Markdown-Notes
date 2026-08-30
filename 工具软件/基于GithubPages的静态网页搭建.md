搭建一个**完全免费、无需购买服务器**且**每天定时自动运行 Python 脚本并刷新网页**的系统，业内最标准、最稳定的方案是使用 **GitHub Actions + GitHub Pages**。

工作流架构：

- **代码仓库（GitHub Repository）**：存放 Python 抓取脚本。
    
- **定时任务（GitHub Actions）**：相当于免费的云端服务器，每天在设定的时间自动唤醒虚拟机，拉取数据运行脚本并生成 `index.html`。
    
- **静态托管（GitHub Pages）**：自动将生成的 `index.html` 部署上线，生成一个永久可访问的公网 HTTPS 网址。
    

### 第一步：准备本地项目文件

在本地电脑上创建一个专门的文件夹（例如 `fund-dashboard`），并在该目录下准备好以下文件：

#### 1. 抓取与生成脚本：`main.py`

将你的完整 Python 脚本命名为 `main.py` 放在该目录下。

> **关键细节调整**：确保脚本生成的 HTML 文件名设为 **`index.html`**（或者在运行时指定 `--out index.html`），因为静态网站服务默认将 `index.html` 作为主页渲染。

#### 2. Python 依赖清单：`requirements.txt`

在同级目录下新建一个 `requirements.txt` 文件，写入脚本依赖的第三方库：

Plaintext

```
akshare
pandas
requests
beautifulsoup4
```

### 第二步：编写 GitHub Actions 自动化工作流

在项目根目录下创建多级文件夹与配置文件： `.github/workflows/update.yml`

在 `update.yml` 中填入以下配置：

YAML

```yaml
name: Daily Fund Dashboard Update

on:
  schedule:
    # 每天北京时间 22:30 执行
    # GitHub Actions 使用 UTC
    - cron: '30 14 * * *'

  # 支持 GitHub 网页手动执行
  workflow_dispatch:

permissions:
  contents: read
  pages: write
  id-token: write

concurrency:
  group: "pages"
  cancel-in-progress: false

jobs:
  build-and-deploy:
    runs-on: ubuntu-latest

    steps:
      # 1. 拉取代码
      - name: Checkout repository
        uses: actions/checkout@v4

      # 2. 安装 Python
      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: '3.10'
          cache: 'pip'

      # 3. 安装依赖
      - name: Install dependencies
        run: |
          python -m pip install --upgrade pip
          pip install -r requirements.txt

      # 4. 运行基金数据抓取脚本
      - name: Run script to generate HTML
        run: |
          python main.py --out index.html

      # 5. 配置 GitHub Pages
      - name: Setup Pages
        uses: actions/configure-pages@v5

      # 6. 上传整个静态网站
      - name: Upload artifact
        uses: actions/upload-pages-artifact@v3
        with:
          path: '.'

      # 7. 发布到 GitHub Pages
      - name: Deploy to GitHub Pages
        id: deployment
        uses: actions/deploy-pages@v4
```

### 第三步：推送到 GitHub 并开启 Pages 服务

1. **新建 GitHub 仓库**：
    
    - 登录 GitHub，点击右上角 **New repository**。
        
    - 填写仓库名称（例如 `fund-tracker`），选择 **Public**（公开仓库享受无限额度的免费 GitHub Pages 与 Actions 构建时长）。
        
2. **将本地代码推送到远程仓库**： 在本地文件夹终端下运行：
    
    Bash
    
    ```
    git init
    git add .
    git commit -m "feat: initial fund dashboard auto-sync"
    git branch -M main
    git remote add origin https://github.com/<你的GitHub用户名>/fund-tracker.git
    git push -u origin main
    ```
    
3. **配置 GitHub Pages 部署源**：
    
    - 进入该仓库的 **Settings** -> **Pages**。
        
    - 在 **Build and deployment** 下方的 **Source** 下拉框中，选择 **`GitHub Actions`**。
        
4. **开启 Actions 权限**：
    
    - 在仓库的 **Settings** -> **Actions** -> **General**。
        
    - 滑动至底部 **Workflow permissions**，勾选 **`Read and write permissions`** 并点击 Save。
        

### 第四步：测试与日常访问

1. **手动测试触发**：
    
    - 进入仓库上方的 **Actions** 标签页。
        
    - 在左侧点击 `Daily Fund Dashboard Update`，右侧点击 **Run workflow** -> **Run workflow**。
        
    - 查看绿色对勾构建日志，完成时会输出公开访问链接（格式通常为 `https://<用户名>.github.io/<仓库名>/`）。
        
2. **自动运行机制**：
    
    - 每天到了设定的定时触发节点（如 UTC 14:30 / 北京时间 22:30，当天场内基金净值和外盘指数基本披露完毕），GitHub 会自动运行虚拟机执行脚本，更新最新数据并重新发布网页。
        
    - 你只需收藏 Pages 生成的网址，随时随地在手机或电脑浏览器中直接查看最新看板。