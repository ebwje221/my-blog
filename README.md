這是一份完整的 Hugo 建站與部署指南，彙整了我們從安裝到除錯的所有重點。這份筆記專為 Ubuntu 24.04 使用者設計，並包含了解決您過程中遇到所有錯誤的具體方案。

---

### 流程總覽 (The Big Picture)

1. **環境準備：** 安裝 Git 與 Hugo Extended 版本。
    
2. **建站與主題：** 建立網站骨架，下載主題 (Theme)。
    
3. **本地開發：** 撰寫文章，使用 `localhost` 預覽。
    
4. **雲端連結：** 建立 GitHub 儲存庫 (Repository)，設定自動化部署 (GitHub Actions)。
    
5. **除錯與上線：** 修正權限與版本問題，正式公開網站。
    

---

### 詳細步驟說明

#### 第一階段：環境安裝 (Installation)

在 Ubuntu 24.04 上，為了避免版本過舊或功能缺失，**強烈建議**使用 Snap 安裝 Extended 版本。

```Bash
# 1. 更新系統並安裝 Git
sudo apt update
sudo apt install git

# 2. 設定 Git 使用者資訊 (若未設定，Commit 會失敗)
git config --global user.email "<your-email@example.com>"
git config --global user.name "<Your Name>"

# 3. 安裝 Hugo (務必指定 extended 頻道以支援 SCSS)
sudo snap install hugo --channel=extended
```

> **常見問題解法：**
> 
> - **Snap Warnings (Bad plugs...):** 安裝時若出現 `warning: ... bad plugs`，通常是系統其他 Snap 軟體的提示，與 Hugo 無關，**可忽略**。
>     
> - **檢查版本：** 執行 `hugo version`。務必確認看到 `extended` 字樣。
>     

---

#### 第二階段：建立網站與設定主題 (Site Setup)

```Bash
# 1. 建立新網站
hugo new site <site-name> # 例如: my-blog

# 2. 進入資料夾並初始化 Git
cd <site-name>
git init

# 3. 下載佈景主題 (以 Submodule 方式管理較佳)
# 語法: git submodule add <theme-url> themes/<theme-name>
git submodule add https://github.com/adityatelange/hugo-PaperMod.git themes/PaperMod

# 4. 修改設定檔 hugo.toml
# 使用編輯器開啟 (如 nano 或 vscode)
nano hugo.toml
```

**`hugo.toml` 建議內容：**

```Ini, TOML
baseURL = 'https://<username>.github.io/<repo-name>/' # 上線前務必改為真實網址，否則樣式會跑版
languageCode = 'zh-tw'
title = '<My Website Title>'
theme = 'PaperMod' # 指定剛下載的主題名稱

[params]
  defaultTheme = 'auto'
```

---

#### 第三階段：內容撰寫與預覽 (Content & Preview)

```Bash
# 1. 新增文章
hugo new content posts/<filename>.md

# 2. 啟動本地預覽伺服器
# -D 參數是用來顯示草稿 (Draft)，因為新文章預設 draft: true
hugo server -D
```

> **預覽網址：** 打開瀏覽器前往 `http://localhost:1313/`

---

#### 第四階段：GitHub 部署設定 (Deployment)

這是最關鍵的一步，讓 GitHub 自動幫您生成網站。

**1. 建立 GitHub Actions 設定檔**

```Bash
mkdir -p .github/workflows
nano .github/workflows/hugo.yaml
```

hugo.yaml 內容 (已修正 Snap 與版本錯誤的最終版)：

此版本解決了 "Snap not found" 與 "PaperMod 版本不相容" 的問題。

```YAML
name: Deploy Hugo site to Pages

on:
  push:
    branches: ["main"]
  workflow_dispatch:

permissions:
  contents: read
  pages: write
  id-token: write

concurrency:
  group: "pages"
  cancel-in-progress: false

defaults:
  run:
    shell: bash

jobs:
  build:
    runs-on: ubuntu-latest
    env:
      # 重要：PaperMod 要求較新版本，這裡指定為 0.146.0
      HUGO_VERSION: 0.146.0
      HUGO_ENV: production
      HUGO_ENABLE_GIT_INFO: "true"
    steps:
      - name: Install Hugo CLI
        run: |
          wget -O ${{ runner.temp }}/hugo.deb https://github.com/gohugoio/hugo/releases/download/v${HUGO_VERSION}/hugo_extended_${HUGO_VERSION}_linux-amd64.deb
          sudo dpkg -i ${{ runner.temp }}/hugo.deb
      
      - name: Checkout
        uses: actions/checkout@v4
        with:
          submodules: recursive
          fetch-depth: 0

      - name: Setup Pages
        id: pages
        uses: actions/configure-pages@v5

      - name: Install Node.js dependencies
        run: "[[ -f package-lock.json || -f npm-shrinkwrap.json ]] && npm ci || true"

      - name: Build with Hugo
        env:
          HUGO_ENVIRONMENT: production
          HUGO_ENV: production
        run: |
          hugo \
            --gc \
            --minify \
            --baseURL "${{ steps.pages.outputs.base_url }}/"

      - name: Upload artifact
        uses: actions/upload-pages-artifact@v3
        with:
          path: ./public

  deploy:
    environment:
      name: github-pages
      url: ${{ steps.deployment.outputs.page_url }}
    runs-on: ubuntu-latest
    needs: build
    steps:
      - name: Deploy to GitHub Pages
        id: deployment
        uses: actions/deploy-pages@v4
```

**2. 連接 GitHub 儲存庫並推送**

```Bash
# 1. 將本地分支改名為 main (符合現代標準)
git branch -M main

# 2. 設定遠端倉庫地址
git remote add origin https://github.com/<username>/<repo-name>.git
# 若設定錯誤，使用 git remote set-url origin <url> 修改

# 3. 讓電腦記住密碼 (避免每次都要輸入 Token)
git config --global credential.helper store

# 4. 推送程式碼
git add .
git commit -m "Initial commit"
git push -u origin main
```

> **遇到 403 Forbidden 錯誤解法：**
> 
> - **原因：** GitHub 不再支援使用登入密碼進行 Git 操作。
>     
> - **解法：** 必須去 GitHub Settings -> Developer settings -> Personal access tokens (Classic) 申請一組 Token。
>     
> - **權限：** Token 必須勾選 `repo` 和 `workflow`。
>     
> - **操作：** 當終端機詢問 Password 時，貼上該 Token。
>     

---

#### 第五階段：GitHub Pages 啟用 (Activation)

若推送成功但網站未顯示 (404)，通常是這裡沒設定。

1. 前往 GitHub 儲存庫頁面 -> **Settings**。
    
2. 左側選單點選 **Pages**。
    
3. 在 **Build and deployment** 區塊：
    
    - 將 Source 改為 **`GitHub Actions`**。
        
4. 回到 **Actions** 頁面，若有失敗的項目，點擊 "Re-run all jobs"。
    

---

### 日常維護流程 (Routine)

以後您只需要重複這三個步驟：

1. **新增/修改：** `hugo new content ...` 或直接編輯檔案。
    
2. **預覽：** `hugo server -D` 確認沒問題。
    
3. **上傳：**
    
    ```Bash
    git add .
    git commit -m "<說明這次改了什麼>"
    git push
    ```
    

---

### 推薦的 Hugo 主題 (Stable & Lightweight)

除了您目前使用的 **PaperMod** (極快、功能多、SEO 強) 之外，以下幾個也是社群公認穩定且適合個人網站的選擇：

| **主題名稱**  | **特色簡介**                                         | **適合對象**             | **範例指令**                                                                               |
| ------------- | ---------------------------------------------------- | ------------------------ | ------------------------------------------------------------------------------------------ |
| **Mainroad**  | 經典的部落格佈局，有側邊欄，閱讀舒適，非常穩定。     | 喜歡傳統部落格結構者     | `git submodule add https://github.com/vimux/mainroad themes/mainroad`                      |
| **Ananke**    | Hugo 官方教學預設主題，極度可靠，幾乎不會壞。        | 初學者、追求絕對穩定者   | `git submodule add https://github.com/theNewDynamic/gohugo-theme-ananke.git themes/ananke` |
| **Bear Blog** | 極簡主義極致，無 CSS 框架，容量極小，速度最快。      | 極簡主義者、文字創作者   | `git submodule add https://github.com/janraasch/hugo-bearblog.git themes/bearblog`         |
| **Blowfish**  | 現代化設計，功能強大 (基於 Tailwind CSS)，外觀炫麗。 | 喜歡現代設計與豐富功能者 | `git submodule add https://github.com/nunocoracao/blowfish.git themes/blowfish`            |

_(註：若要更換主題，請重新下載對應的 submodule，並修改 `hugo.toml` 中的 `theme` 欄位)_

References: 
1. [mathegape](http://mathagape.com/posts/hugo-github-pages/)
2. [github](https://github.com/1000copy/zigcc.github.io)
