# chatgpt-gijiroku-cleaner (Zenn CLI)

這個資料夾已初始化為 `zenn-cli` 專案，可用來管理 Zenn 文章。

## 已完成設定

- 初始化 Zenn 專案（`npx zenn init`）
- 建立 `articles/` 資料夾
- 生成第一篇技術文章
- 安裝 `zenn-cli`
- 設定 npm scripts

## 常用指令

```bash
# 啟動本地預覽（固定 http://localhost:3004）
npm run preview

# 新增文章草稿
npm run new
```

## 發布流程

1. 編輯 `articles/*.md`
2. 確認 Front Matter（如 `title`、`topics`、`published`）
3. Commit 並 push 到已連接 Zenn 的 GitHub 儲存庫分支

參考官方文件：
- [Zenn CLI 使用指南](https://zenn.dev/zenn/articles/zenn-cli-guide)