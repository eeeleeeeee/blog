# My Blog

用 [MkDocs Material](https://squidfunk.github.io/mkdocs-material/) 建的個人部落格，部署到 GitHub Pages。

## 環境需求

| 工具 | 版本 | 安裝方式（Windows） |
|------|------|---------------------|
| Python | 3.12.x | 見下方 Python 安裝 |
| Git | 2.x | `scoop install git` |

> Python 3.13 / 3.14 也許能跑，但 3.12 是目前最穩的 LTS 級版本，新環境一律用這個。

### Python 安裝（Windows / scoop）

`python312` 在 scoop 的 `versions` bucket 裡，預設沒加，要先加 bucket：

```powershell
scoop bucket add versions
scoop install versions/python312

# 把 `python` 指向 3.12（如果系統已經裝過其他版本）
scoop reset python312
python --version   # 應顯示 Python 3.12.x
```

## 第一次設定

```powershell
git clone <repo-url> blog
cd blog

# 用 Python 3.12 建虛擬環境（不依賴系統的 python 預設版）
& "$HOME\scoop\apps\python312\current\python.exe" -m venv .venv

# 啟用 venv（後續每次新開 PowerShell 都要做）
.\.venv\Scripts\Activate.ps1

# 安裝相依套件
pip install -r requirements.txt
```

> 如果 `Activate.ps1` 跳「無法載入指令碼」，先解開 PowerShell 執行原則：
> ```powershell
> Set-ExecutionPolicy -Scope CurrentUser -ExecutionPolicy RemoteSigned
> ```

## 常用指令

| 指令 | 用途 |
|------|------|
| `mkdocs serve` | 本機預覽（含 live reload），預設網址 `http://127.0.0.1:8000/blog/` |
| `mkdocs build` | 產生靜態檔到 `site/`（部署由 GitHub Actions 處理，平常不用手動跑） |
| `mkdocs --version` | 確認 mkdocs 安裝正確 |

## 寫一篇新文章

在 `docs/blog/posts/` 底下建一個 `.md` 檔，最上面要有 front matter：

```markdown
---
date: 2026-05-10
categories:
  - 雜記
tags:
  - 起手式
---

# 文章標題

文章內文。

<!-- more -->

`<!-- more -->` 之前的內容會出現在列表頁的摘要，之後的要點進文章才看得到。
```

`date` 是必填，否則 blog plugin 會跳錯。

## 專案結構

```
blog/
├── .github/workflows/    # GitHub Actions 部署設定（待新增）
├── .venv/                # Python 虛擬環境（不進 git）
├── docs/                 # 所有 markdown 內容
│   ├── index.md          # 首頁
│   └── blog/
│       ├── index.md      # 部落格總覽頁
│       └── posts/        # 文章
├── site/                 # mkdocs build 產物（不進 git）
├── .gitignore
├── mkdocs.yml            # MkDocs 主設定
├── requirements.txt      # Python 相依套件
└── README.md
```

## 已知問題與避雷

### click 8.3 之後 `mkdocs serve` 的 live reload 預設失效

`requirements.txt` 已把 `click` 鎖在 `<8.3`。如果手動升級了 click 導致版本超過，會發生：

- 改檔不會自動 rebuild
- 啟動 log 看不到 `Watching paths for changes: ...`

解法：重新跑 `pip install -r requirements.txt`，或啟動加 `--livereload` 旗標。
