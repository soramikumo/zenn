---
title: "本当にTerraformを理解している？"
emoji: "🧱"
type: "tech"
topics: ["Terraform", "IaC", "入門"]
published: false
---

## この記事を書いたきっかけ

昨今はAIエージェントのおかげで、リプレースがしやすくなりました。それでも3割くらいはIaCできない現場があると私は思っています（大偏見）。

`terraform`は書くこと自体は簡単です。hclという言語でyamlと同じように人間が見やすい構造言語になっています。

なのでAIに`terraform`を書かせることは簡単です。レビューも簡単です。
ただ、applyして何が消えるかを判断するのは人間です。その判断に要るのがstateの理解です。改めてTerraformを理解しましょう。

module周りの書き方（変数の受け渡しなど）は扱いません。絞るのは2つ。Terraformが裏でしていることと、stateの扱い方です。

対象の読者
- Terraformを触ったことはあるが、stateがよく分かっていない方
- これからIaCを導入する方

※ この記事ではAI生成した文章や、資料が含まれています。私はAIぽい要素は無くそうと努力していますが、読みづらければ私の頭が悪いせいなのですみません。。

## この記事でやること

Terraformの登場人物を整理し、init・plan・applyで何が起きるか実際に動かして確かめます。そのうえでstateの置き場所とディレクトリ構成を考えます。

検証にクラウドは使いません。`local_file`はローカルにファイルを作るリソースです。手元ですぐ再現できます。使ったコードは[soramikumo/terraform-state-basics](https://github.com/soramikumo/terraform-state-basics)にあります。

検証環境（2026-10-23〜24 実施）:

| 項目 | バージョン |
|---|---|
| 実行マシン | Mac（Apple Silicon） |
| Terraform | 1.12.1 |
| hashicorp/local | 2.5.3 |

## 用語
簡単にterraformで出てくる用語の解説です。よく使うものだけ抜粋にはなります。

- provider：TerraformとクラウドのAPIをつなぐプラグイン。AWSなら`hashicorp/aws`、Google Cloudなら`hashicorp/google`。今回は`hashicorp/local`を使います
- state：Terraformが作ったリソースの記録。既定では`terraform.tfstate`というJSONファイル
- backend：stateの置き場所を指定することができます。既定はローカル。GCSやS3にも置けます
- lineage：stateごとの固有のID。どのstateかを見分けるために使われます
- serial：stateの版数。中身が変わるたびに増えます
- ロック：同時に2つの操作がstateを書き換えないようにする仕組み
- ドリフト：Terraformの外で実リソースが変わり、stateとずれた状態

## どのように状態管理しているのか

Terraformが見ているのは、
1. 手元にある宣言的コード
2. state
3. クラウド環境にある実リソース

providerはその間に立つ通訳です。

**「Terraformは3つを見比べて、差分だけを実行するツール」** でしかないのです。


![Terraformは3つを見比べる](/images/terraform-state-basics/01-cast.png)

## init・plan・applyで何が起きるか

![init・plan・applyが読むもの・書くもの](/images/terraform-state-basics/02-commands.png)

トグル planの挙動について
planは何も出力しません。実リソースを読んで、変更内容を表示するだけです。
すでにapplyしていれば、stateが生まれるので、唯一のstateと実リソースを比較しているのです。

### stateは唯一である
もし、すでにapplyして、リソースがクラウド上にできたとします。applyするとstateは生まれます。次にapply するときにはそのstateを元に、差分を検知するわけですが、ここでstateを破棄してplan・applyしようとするとどうなるでしょうか？

答えは
もう一度リソースを作り直します。
正確に言えば、クラウドリソースで同じものを作りなすということはないのです。？
→まあ正直に言えばわからなん


使ったコードは[main.tf](https://github.com/soramikumo/terraform-state-basics/blob/main/main.tf)だけです。

```hcl:main.tf
# プロバイダーの指定
terraform {
  required_providers {
    local = {
      source  = "hashicorp/local"
      version = "2.5.3"
    }
  }
}

# hello.txtを作成
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
backendには、一般的にはS3やGCSといったクラウドストレージにstateを投げるための設定を書くことが多いです。
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

## stateは1つしか存在してはいけない

実験として`terraform.tfstate`を別の場所に移してからplanしました。
**stateがないと、Terraformは何も作っていないと判断します**

```text:stateを移したあとのplan（実ログ・抜粋）
  # local_file.hello will be created
  + resource "local_file" "hello" 

Plan: 1 to add, 0 to change, 0 to destroy.
```
あら、、、？
すでにあるファイルを、もう一度作ろうとしています。`local_file`は上書きするだけで済みますが、クラウドならこうはいきません。
**同じ名前のバケットは作成エラーになり、名前の重複が許されるリソースは二重にできてしまいます。**

### stateは記録であって実物ではない

今度はstateを戻し、ファイルを手で消してからplanしました。コンソールで手で消したのと同じ状況です。

```text:ファイルを手で消したあとのplan（実ログ・抜粋）
local_file.hello: Refreshing state... [id=a0ea5669caa2a7da6e3aaf7a2fbb0bffd2689619]

  # local_file.hello will be created
Plan: 1 to add, 0 to change, 0 to destroy.
```

`Refreshing state...`で実リソースを読み、消えていることに気づいて作り直そうとしています。
この状態をドリフトと呼びます。

![stateと実物がずれたときのplan](/images/terraform-state-basics/03-drift.png)

:::message
ずれの出方はproviderによって違います。`local_file`は中身を手で書き換えても「消えた」とみなし、作り直しを提案しました。クラウドのリソースは設定を変えられても、多くは`~ update in-place`で元に戻そうとします。
:::

### Terraformはstateの取り違えを拒否する

stateには`lineage`（どのstateか）と`serial`（何版目か）が入っています。
古いstateや別のstateで上書きしようとすると、Terraformが止めてくれます

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

Terraform自身が、正しいstateは常に1つという前提で動いています。その1つとは、backendに今置いてあるstateです。今回の[main.tf](https://github.com/soramikumo/terraform-state-basics/blob/main/main.tf)はbackendを書いていません。そのため既定のlocal backendになり、同じディレクトリの`terraform.tfstate`がその1つです。最後に書き込まれたstateが、そのまま正しいstateになります。

lineageとserialで確かめているのは「同じstateの古くない版か」だけ。中身が正しいかは見ていません。

試しに、同じlineageのままserialだけ1つ上げたstateを作りました。中身の`content`は`fake`に書き換えてあります。書き戻すと、エラーなく通ります。

```text:中身を書き換えたstateを書き戻す（実ログ）
$ terraform state push forged.tfstate
$ terraform state pull | grep '"content"'
            "content": "fake\n",
```

実物のファイルは`hello terraform`のままです。それでもplanは作り直しを提案しました。

```text:書き換えたstateでのplan（実ログ・抜粋）
  # local_file.hello must be replaced
-/+ resource "local_file" "hello" {
      ~ content              = <<-EOT # forces replacement
          - fake
          + hello terraform
        EOT

Plan: 1 to add, 0 to change, 1 to destroy.
```

同じ中身のコピーを配っても、次のapplyで古くなります。全員が読み書きするstateは1か所におかなければなりません。

## 1つしかないstateを守るのに要るもの

要るのは
1. 共有
2. ロック
3. 履歴
4. 機密保護

以前にもいった通り、例外を除いてstateは基本クラウドストレージ(s3やgcs)に起きます。
ここではロックと機密を実際に見ます。

### ロック：同時にapplyすると片方が止まる

15秒かかるリソース（`local-exec`で`sleep 15`）を足しました。applyを2つ同時に走らせると、あとの方が止まります。

:::details 実際に流したコマンド
足したのは[add/slow.tf](https://github.com/soramikumo/terraform-state-basics/blob/main/add/slow.tf)です。1つ目のapplyは裏で走らせ、4秒後に2つ目を走らせました。

```sh
# 1つ目：裏で走らせる。sleep 15で15秒かかる
terraform apply -auto-approve > apply-A.log 2>&1 &

# 4秒後、1つ目がまだ動いている間に2つ目を実行
sleep 4
terraform apply -auto-approve
```

`-auto-approve`は確認の`yes`を飛ばすオプションです。裏で走らせたapplyには`yes`を打てないため付けました。ターミナルを2つ開き、1つ目が終わる前に2つ目を走らせても同じ結果になります。
:::

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

`sensitive = true`を付けた変数でパスワードを渡しました。値は`demo-pass-1234`です。

```hcl:secret/main.tf
variable "db_password" {
  type      = string
  sensitive = true
}

# 実際はDBのパスワード欄などに渡す
resource "terraform_data" "db" {
  input = var.db_password
}
```

```text:terraform apply（実ログ・抜粋）
$ terraform apply -var 'db_password=demo-pass-1234'

  + resource "terraform_data" "db" {
      + id     = (known after apply)
      + input  = (sensitive value)
      + output = (known after apply)
    }
```

```json:terraform.tfstate（実物・抜粋）
"input": {
  "value": "demo-pass-1234",
  "type": "string"
},
```

planの表示では隠れていますが、stateにはそのまま入ります。HashiCorp公式ドキュメントも、stateを機密情報として扱いGitのワークフローから外すよう勧めています。
もし管理する場合は注意が必要になります。

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

![ディレクトリ1つにstateが1つ](/images/terraform-state-basics/04-directory.png)

### env：環境ごとにstateを分けて、壊れる範囲を限る

stateを環境ごとに分けておけば、devでapplyが失敗してもprodは無事です。

### modules：stateを持たない部品

moduleは呼び出した側（env/devなど）のstateに入ります。module単体のstateはありません。変数の受け渡しは別の記事で扱います。

### bootstrap：stateの置き場所を作る

env/devのbackendでGCSを指定するには、先にバケットが要ります。そのバケットを作るのがbootstrapです。

問題はbootstrap自身のstateをどこに置くかです。選択肢は2つあります。

![bootstrapのstateの置き場所は2案](/images/terraform-state-basics/05-bootstrap.png)

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
