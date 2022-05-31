---
title: "Git 全コミットのAuthor/Committerを書き換える方法"
emoji: "📘"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["git"]
published: true
---

## filter-branch を利用する

```bash
git filter-branch -f --env-filter \
    "GIT_AUTHOR_NAME=''; \
     GIT_AUTHOR_EMAIL=''; \
     GIT_COMMITTER_NAME=''; \
     GIT_COMMITTER_EMAIL='';" \
    --tag-name-filter cat -- --all
```

- `--tag-name-filter cat` でタグを付け替えてくれます。
- `-- --all` で全ブランチを対象にします。あらかじめ対象のブランチはローカルで追跡しておくようにしましょう。
- リモートリポジトリへ push する際は `force push` が必要です。

https://git-scm.com/docs/git-filter-branch

### 過去を改竄したいときって？

基本的には組織のプロジェクトで上記を実行するのはおすすめできません。というより、普通はプロテクトされているでしょうからできないと思います。

個人プロジェクトにおいて、例えば「メールアドレスが変わったけど GitHub アカウントに古いメールアドレスを紐づけておくのが嫌だ」のようなケースで利用するのがいいと思います。もちろんそれでも慎重に。
