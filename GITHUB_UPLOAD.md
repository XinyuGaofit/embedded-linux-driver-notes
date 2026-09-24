# 上传到 GitHub

如果仓库已经创建为空仓库：

```bash
git clone https://github.com/<你的用户名>/embedded-linux-driver-notes.git
cd embedded-linux-driver-notes
```

将本目录中的：

```text
README.md
docs/
.gitignore
```

复制进去，然后：

```bash
git add .
git commit -m "docs: add embedded linux driver notes"
git push
```

如果直接使用 GitHub 网页，也可以上传 `README.md` 和整个 `docs/` 目录。
