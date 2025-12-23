# KEDA 繁體中文文件

此資料夾包含 KEDA（Kubernetes 事件驅動自動擴縮器）的繁體中文翻譯文件，使用 [mdbook](https://rust-lang.github.io/mdBook/) 建構。

## 本地開發

### 安裝 mdbook

```bash
# 使用 Cargo 安裝
cargo install mdbook

# 或使用 Homebrew (macOS)
brew install mdbook
```

### 建構文件

```bash
cd docs
mdbook build
```

建構輸出會在 `book/` 資料夾。

### 本地預覽

```bash
cd docs
mdbook serve
```

開啟瀏覽器前往 http://localhost:3000

## 專案結構

```
docs/
├── book.toml          # mdbook 設定檔
├── README.md          # 本說明文件
└── src/
    ├── SUMMARY.md     # 目錄結構
    ├── _index.md      # 首頁
    ├── deploy.md      # 部署指南
    ├── migration.md   # 版本遷移
    ├── concepts/      # 核心概念
    ├── authentication-providers/  # 驗證提供者
    ├── scalers/       # 擴縮器
    ├── operate/       # 維運指南
    ├── reference/     # 參考資料
    └── img/           # 圖片資源
```

## 部署

可使用 GitHub Actions 自動部署到 GitHub Pages。參見 `.github/workflows/deploy-docs.yml`。
