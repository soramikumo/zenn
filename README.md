# Zenn Contents

[mi_queue の Zenn](https://zenn.dev/mikku_mac) の記事を管理するリポジトリ。GitHub 連携で、`main` ブランチに push すると Zenn に反映される。

## 構成

| 場所 | 中身 |
|---|---|
| `articles/<slug>.md` | 記事。ファイル名が記事 URL の slug になる |
| `books/` | 本 |
| `images/<slug>/` | 記事の画像の控え |

## 使い方

```bash
npm install
npx zenn new:article --slug my-article-slug --title "記事タイトル" --type tech --emoji 📮
npx zenn preview          # http://localhost:8000 でプレビュー
```

- slug は 12〜50 字の半角英小文字・数字・ハイフン・アンダースコア。
- `published: false` のまま push すると下書き、`true` にして push すると公開。
- 公開済みの記事は Zenn のエディタではなく、このリポジトリで直す。
