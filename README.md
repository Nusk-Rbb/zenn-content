# zenn-content

[Zenn](https://zenn.dev/) の記事を管理するリポジトリ。

## 使い方

```bash
npm install          # 初回のみ
npx zenn preview     # http://localhost:8000 でプレビュー
npx zenn new:article --slug <slug> --title "<title>" --type tech
npx zenn list:articles
```

`slug` は 12〜50 文字の小文字英数・ハイフン・アンダースコア。

## 公開のしかた

記事の frontmatter が `published: false` のうちは下書き。`true` にして push すると公開される。

```yaml
---
title: "記事タイトル"
emoji: "🔒"
type: "tech"        # tech | idea
topics: ["nix"]     # 最大5個
published: false
---
```

## 記事

| slug | タイトル | 状態 |
|---|---|---|
| `nix-supply-chain-hardening` | Nix のサプライチェーン対策 | 公開 |

## 参考

* [Zenn CLI の使い方](https://zenn.dev/zenn/articles/zenn-cli-guide)
* [GitHub リポジトリ連携](https://zenn.dev/zenn/articles/connect-to-github)
