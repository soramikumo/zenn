---
title: "本当にTerraformを理解している？"
emoji: "🧱"
type: "tech"
topics: ["Terraform", "IaC", "入門"]
published: false
---

## この記事を書いたきっかけ

昨今はAIエージェントのおかげで、リプレースがしやすくなりました。それでも3割くらいはIaCできない現場があると私は思っています（大偏見）。

AIに`.tf`を書かせることはできます。ただ、applyして何が消えるかを判断するのは人間です。その判断に要るのがstateの理解です。改めてTerraformを理解しましょう。

module周りの書き方（変数の受け渡しなど）は扱いません。絞るのは2つ。Terraformが裏でしていることと、stateの扱い方です。

対象の読者
- Terraformを触ったことはあるが、stateがよく分かっていない方
- これからIaCを導入する方

※ この記事ではAI生成した文章や、資料が含まれています。私はAIぽい要素は無くそうと努力していますが、読みづらければ私の頭が悪いせいなのですみません。。

## この記事でやること

Terraformの登場人物を整理し、init・plan・applyで何が起きるか実際に動かして確かめます。そのうえでstateの置き場所とディレクトリ構成を考えます。

検証にクラウドは使いません。`local_file`はローカルにファイルを作るリソースです。手元ですぐ再現できます。

検証環境（2026-10-23 実施）:

| 項目 | バージョン |
|---|---|
| 実行マシン | Mac（Apple Silicon） |
| Terraform | 1.12.1 |
| hashicorp/local | 2.5.3 |
| hashicorp/random | 3.7.2 |

## 用語

- provider：TerraformとクラウドのAPIをつなぐプラグイン。AWSなら`hashicorp/aws`、Google Cloudなら`hashicorp/google`。今回は`hashicorp/local`を使います
- state：Terraformが作ったリソースの記録。既定では`terraform.tfstate`というJSONファイル
- backend：stateの置き場所。既定はローカル。GCSやS3にも置けます
- lineage：stateごとの固有のID。どのstateかを見分けるために使われます
- serial：stateの版数。中身が変わるたびに増えます
- ロック：同時に2つの操作がstateを書き換えないようにする仕組み
- ドリフト：Terraformの外で実リソースが変わり、stateとずれた状態

## 登場人物は3つ

Terraformが見ているのは、コード・state・実リソースの3つだけです。providerはその間に立つ通訳です。

```
コード(.tf)  ←比較→  state  ←比較→  実リソース
「こうあってほしい」  「前回作ったもの」   「今あるもの」
                                         ↑
                                provider が API で読み書きする
```

「Terraformは3つを見比べて、差分だけを実行するツール」です。

## init・plan・applyで何が起きるか

| コマンド | 読むもの | 書くもの |
|---|---|---|
| init | コード（providerとbackendの指定） | `.terraform/`（provider本体）と`.terraform.lock.hcl` |
| plan | コード・state・実リソース | 何も書かない |
| apply | コード・state・実リソース | 実リソースとstate |

使ったコードはこれだけです。

```hcl:main.tf
terraform {
  required_providers {
    local = {
      source  = "hashicorp/local"
      version = "2.5.3"
    }
  }
}

resource "local_file" "hello" {
  filename = "${path.module}/out/hello.txt"
  content  = "hello terraform\n"
}
```

### init：providerを取ってくる

```text:terraform init（実ログ・抜粋）
Initializing the backend...
Initializing provider plugins...
- Finding hashicorp/local versions matching "2.5.3"...
- Installing hashicorp/local v2.5.3...
- Installed hashicorp/local v2.5.3 (signed by HashiCorp)
Terraform has created a lock file .terraform.lock.hcl to record the provider
selections it made above.
```

init直後のディレクトリには`.terraform/`と`.terraform.lock.hcl`だけが増えています。`terraform.tfstate`はまだありません。

```text
.terraform
.terraform.lock.hcl
main.tf
```

:::message
backendを書いた場合は、init後に`.terraform/terraform.tfstate`ができます。これはbackendの設定を覚えておくファイルで、stateではありません。
:::

### plan：3つを見比べる

```text:terraform plan（実ログ・抜粋）
  # local_file.hello will be created
  + resource "local_file" "hello" {
      + content              = <<-EOT
            hello terraform
        EOT
      + filename             = "./out/hello.txt"
      + id                   = (known after apply)
    }

Plan: 1 to add, 0 to change, 0 to destroy.
```

stateがまだないため、Terraformは「何も作っていない」と判断して作成を提案します。

2回目以降のplanでは、先頭に`Refreshing state...`が出ます。providerが実リソースを読みにいっている行です。

```text:apply後のplan（実ログ）
local_file.hello: Refreshing state... [id=a0ea5669caa2a7da6e3aaf7a2fbb0bffd2689619]

No changes. Your infrastructure matches the configuration.
```

planはstateを書き換えません。実リソースを読んでも、結果は表示するだけです。

### apply：実行してstateを書く

```text:terraform apply（実ログ・抜粋）
local_file.hello: Creating...
local_file.hello: Creation complete after 0s [id=a0ea5669caa2a7da6e3aaf7a2fbb0bffd2689619]

Apply complete! Resources: 1 added, 0 changed, 0 destroyed.
```

ここで初めて`terraform.tfstate`ができます。

```json:terraform.tfstate（実物・抜粋）
{
  "version": 4,
  "terraform_version": "1.12.1",
  "serial": 1,
  "lineage": "9220f245-dbe3-329f-517f-5f83e1eb51c0",
  "resources": [
    {
      "mode": "managed",
      "type": "local_file",
      "name": "hello",
      "provider": "provider[\"registry.terraform.io/hashicorp/local\"]",
      "instances": [
        {
          "attributes": {
            "content": "hello terraform\n",
            "filename": "./out/hello.txt",
            "id": "a0ea5669caa2a7da6e3aaf7a2fbb0bffd2689619"
          }
        }
      ]
    }
  ]
}
```

`resources`に「どのproviderで何を作ったか」が入っています。`serial`と`lineage`は次の章で使います。

## stateは1つしかない

### stateがないと、Terraformは何も作っていないと判断する

`terraform.tfstate`を別の場所に移してからplanしました。ファイル（`out/hello.txt`）は消していません。

```text:stateを移したあとのplan（実ログ・抜粋）
  # local_file.hello will be created
  + resource "local_file" "hello" {

Plan: 1 to add, 0 to change, 0 to destroy.
```

すでにあるファイルを、もう一度作ろうとしています。`local_file`は上書きするだけで済みますが、クラウドならこうはいきません。たとえば同じ名前のバケットは作成エラーになり、名前の重複が許されるリソースは二重にできます。

### stateは記録であって実物ではない

今度はstateを戻し、ファイルを手で消してからplanしました。コンソールで手で消したのと同じ状況です。

```text:ファイルを手で消したあとのplan（実ログ・抜粋）
local_file.hello: Refreshing state... [id=a0ea5669caa2a7da6e3aaf7a2fbb0bffd2689619]

  # local_file.hello will be created
Plan: 1 to add, 0 to change, 0 to destroy.
```

`Refreshing state...`で実リソースを読み、消えていることに気づいて作り直そうとしています。この状態をドリフトと呼びます。

:::message
ずれの出方はproviderによって違います。`local_file`は中身を手で書き換えても「消えた」とみなし、作り直しを提案しました。クラウドのリソースは設定を変えられても、多くは`~ update in-place`で元に戻そうとします。
:::

### Terraformはstateの取り違えを拒否する

stateにはlineage（どのstateか）とserial（何版目か）が入っています。古いstateや別のstateで上書きしようとすると、Terraformが止めます。

まずserialが1の時点でstateをコピーしました。次はファイルの中身を変えてapplyします。serialは1から4に増えました。1回のapplyで1ずつ増えるとは限りません。

そのあとで、コピーしておいた古いstateを書き戻しました。

```text:古いstateを書き戻す（実ログ）
$ terraform state push state-v1.tfstate
Failed to write state: cannot import state with serial 1 over newer state with serial 4
```

lineageだけを書き換えた別のstateも拒否されます。

```text:lineageが違うstateを書き戻す（実ログ）
$ terraform state push state-other.tfstate
Failed to write state: cannot import state with lineage "00000000-0000-0000-0000-000000000000" over unrelated state with lineage "9220f245-dbe3-329f-517f-5f83e1eb51c0"
```

Terraform自身が、正しいstateは常に1つという前提で動いています。

## 1つしかないstateを守るのに要るもの

要るのは共有・ロック・履歴・機密保護の4つです。共有と履歴は名前のとおりです。ここではロックと機密を実際に見ます。

### ロック：同時にapplyすると片方が止まる

15秒かかるリソース（`local-exec`で`sleep 15`）を足しました。applyを2つ同時に走らせると、あとの方が止まります。

```text:あとから走らせたapply（実ログ・抜粋）
Error: Error acquiring the state lock

Error message: resource temporarily unavailable
Lock Info:
  ID:        2ebb3487-a10b-c3ad-c532-6ae775c4740d
  Path:      terraform.tfstate
  Operation: OperationTypeApply
  Who:       user@host
  Version:   1.12.1
```

これはローカルbackendのファイルロックです。同じPCの中でしか効きません。チームで使うなら、backend側にロックの仕組みが要ります。

### 機密：planでは隠れていてもstateには平文で入る

`random_password`で16文字のパスワードを作りました。

```text:terraform apply（実ログ・抜粋）
      + result      = (sensitive value)
```

```json:terraform.tfstate（実物・抜粋）
"result": "（実際のパスワードが平文で入っている。記事では伏せた）",
```

planの表示では隠れていますが、stateにはそのまま入ります。HashiCorp公式ドキュメントも、stateを機密情報として扱いGitのワークフローから外すよう勧めています。

## stateの置き場所を比べる

| 置き場所 | 共有 | ロック | 履歴 | 機密 |
|---|---|---|---|---|
| ローカル | × | 同じPC内だけ | × | 手元だけ |
| git | pushすれば共有できる | × | ○ | 履歴に残り、消せない |
| GCS・S3 | ○ | ○ | バージョニングで○ | IAMで見られる人を絞れる |
| HCP Terraform | ○ | ○ | ○ | 保存時に暗号化される |

- gitはapplyとコミットが別の操作です。コミットを忘れると、古いstateが残ります
- GCSはロックに対応しています。公式はバケットのバージョニングを強く勧めています
- S3はTerraform 1.10からS3だけでロックできます（`use_lockfile = true`、既定はオフ）。古い記事に出てくるDynamoDBのロックは非推奨になりました

実務の基本はオブジェクトストレージ（GCS・S3）です。迷うのはbootstrapだけで、次の章で扱います。

## ディレクトリ構成はstateの分け方

Terraformはディレクトリごとにstateを1つ持ちます。ディレクトリをどう切るかで、stateの分け方が決まります。

```
terraform/
├── bootstrap/      # stateを置くバケットを作る
├── env/
│   ├── dev/        # devのstate
│   └── prod/       # prodのstate
└── modules/        # stateを持たない部品
```

### env：環境ごとにstateを分けて、壊れる範囲を限る

stateを環境ごとに分けておけば、devでapplyが失敗してもprodは無事です。

### modules：stateを持たない部品

moduleは呼び出した側（env/devなど）のstateに入ります。module単体のstateはありません。変数の受け渡しは別の記事で扱います。

### bootstrap：stateの置き場所を作る

env/devのbackendでGCSを指定するには、先にバケットが要ります。そのバケットを作るのがbootstrapです。

問題はbootstrap自身のstateをどこに置くかです。選択肢は2つあります。

**作ったバケットに移す**

まずローカルでapplyしてバケットを作ります。次にbackendへそのバケットを書き足します。`terraform init -migrate-state`を実行すると、stateがバケットへ移ります。ロックも履歴もそろいます。

ただし、バケットが自分自身の記録を抱える形になります。金庫の鍵を金庫にしまうのと同じです。バケットが作り直しになる変更（名前やリージョンの変更）をapplyすると、stateも一緒に消えかねません。`force_destroy = true`だと中身ごと消せてしまいます。既定値（false）のままにし、`lifecycle { prevent_destroy = true }`も付けて削除を防ぎます。古い版はバージョニングで戻せるようにしておきます。

**ローカルに置いてgitにコミットする**

stateを管理対象の外へ置けるため、バケットに何が起きてもstateは残ります。引き換えにロックはなく、機密が入っていないかの確認が要ります。bootstrapは変更が少なく、applyするのも1人のことが多いです。そのためロックがなくても困りにくいでしょう。

どちらにも理由があり、チームによって選び方が分かれます。

## まとめ

- Terraformはコード・state・実リソースを見比べ、差分だけを実行するツール
- stateは1つしかなく、Terraform自身もlineageとserialで取り違えを防いでいる
- stateの置き場所はオブジェクトストレージが基本。bootstrapだけは置き場所を選ぶ

今回はローカルのファイルで確かめました。クラウドだと差分の出方が違うことがあります。次はmoduleと変数の受け渡しを書く予定です。

## 参考

- [Backend Type: s3](https://developer.hashicorp.com/terraform/language/backend/s3)
- [Backend Type: gcs](https://developer.hashicorp.com/terraform/language/backend/gcs)
- [Manage sensitive data in your configuration](https://developer.hashicorp.com/terraform/language/state/sensitive-data)
- [Terraform v1.10 CHANGELOG](https://github.com/hashicorp/terraform/blob/v1.10/CHANGELOG.md)
