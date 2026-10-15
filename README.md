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

## 記事一覧

| ファイル | タイトル |
|---|---|
| [54636fbe69ee1a](articles/54636fbe69ee1a.md) | Loop Engineeringへ入門 デスクトップを常時起動の実行土台を作った |
| [7ed324ea68dc1c](articles/7ed324ea68dc1c.md) | 2026年に実務1年目で転職活動してわかったこと |
| [1c3eaa134df25c](articles/1c3eaa134df25c.md) | 新人がSVN→Git 移行と CI/CD 構築した半年間の話 |
| [8675c315dd5175](articles/8675c315dd5175.md) | 「何がしたいの？」に答えられなかったので、キャリアの軸を決めた |
| [dd3b3491634389](articles/dd3b3491634389.md) | IaaSが落ちた深夜-クエリを書かずに調査する ― Observability Agent |
| [21c2511bb9516b](articles/21c2511bb9516b.md) | SAAに合格したけど全然身になってなかった話 |
| [9b6f00aad4d0ec](articles/9b6f00aad4d0ec.md) | 【リアーキテクチャ】Azure VMからVMSSに移行した話 |
| [k8s-basics-kind](articles/k8s-basics-kind.md) | 今更Kubernetesへ入門する アーキテクチャと自己修復 |
| [email-codex-guide](articles/email-codex-guide.md) | （下書き）メールはどう届く？送信ボタンから受信トレイに現れるまで |
| [how-email-reaches-inbox](articles/how-email-reaches-inbox.md) | （下書き）メールを送信してから相手の受信トレイに届くまで |

公開済みの記事はファイル名が URL の slug なので変えない（変えると別の記事になる）。新しい記事は題名に沿った slug で作る。
