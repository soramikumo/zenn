---
title: "今更Kubernetesへ入門する アーキテクチャと自己修復"
emoji: "☸️"
type: "tech"
topics: ["Kubernetes", "入門", "docker", ]
published: true
---

## この記事を書いたきっかけ

仕事で Kubernetes を使う予定ができました。そのために学んだことを整理、そしていくつか実験をしてみたのでその結果をまとめています。

対象の読者
- k8sに対して興味がある方
- ゆる〜く理解したい方

※ この記事ではAI生成した文章や、資料が含まれています。私はAIぽい要素は無くそうと努力していますが、読みづらければ私の頭が悪いせいなのですみません。。

## この記事でやること

kind で3ノードの Kubernetes を立て、わざと壊して「宣言した状態を保ち続ける」動きを観察します。

:::details kind とは
kind（Kubernetes IN Docker）は、Kubernetes 公式の CLI ツールです。Docker コンテナ1つを1台のノードに見立て、その中で kubeadm を使って普通のクラスタを組みます。

中身は改造していない Kubernetes です。もとは Kubernetes 自体のテスト用に作られたので、起動が速く、使い終わったら丸ごと消せます。ノードを止めたいときは `docker stop` で済むので、障害の再現にも向いています。

ローカルで動かす手段はほかにもあります。

| ツール | 特徴 |
|---|---|
| minikube | 老舗。アドオン（ingress・dashboard など）が豊富 |
| k3d | 軽量版の k3s を Docker で動かす |
| Docker Desktop | 設定のチェック1つで単一ノードが立つ |

今回は複数ノードを YAML 1枚で作れて、ノード障害を再現しやすい kind を選びました。
:::

- Pod を消す → 何秒で戻るか
- Service 経由なら Pod が入れ替わっても届くか
- 存在しないイメージに更新する → どこで止まり、どう戻すか
- ワーカーノードごと止める → 何秒で Pod が逃げるか

検証環境（2026-10-14 実施）:

| 項目 | バージョン |
|---|---|
| 実行マシン | Windows 11 の WSL2 Ubuntu 24.04（x86_64） |
| Docker | docker-ce 29.9.0 |
| kind | v0.33.0 |
| Kubernetes（kindest/node） | v1.37.0 |
| kubectl | v1.37.0 |

ノードイメージは digest まで固定しています。この記事の挙動は v1.37.0 時点のものです。

## Kubernetes の基礎

検証に入る前に、Kubernetes の全体像を順番に見ていきます。知っている人は「作った構成」まで飛ばしてください。

### なぜ Kubernetes が必要か

コンテナを1台で動かすだけなら Docker で足ります。台数が増えると、次のことを人が面倒を見ることになります。

- 落ちたコンテナを誰が戻すか
- どのマシンに置くか
- 入れ替わるコンテナにどう通信を届けるか

Kubernetes はこれを人の代わりに続けてくれます。

### Kubernetes は何をしてくれるのか

![Kubernetes はあるべき姿に近づけ続ける](/images/k8s-basics-kind/01-desired-state.png)

Kubernetes には「nginx を3つ動かしたい」のように、あるべき姿を渡します。「このマシンでこのコマンドを打つ」といった手順は書きません。

渡したあとは、Kubernetes があるべき姿と今の様子を比べ続けます。1つ落ちれば「3つあるべきなのに2つ」というずれが見つかり、自動で1つ足されます。この記事の検証は、ほぼすべてこの動きの観察です。

あるべき姿を書いたファイルを**マニフェスト**と呼びます。中身は YAML です。

@[card](https://kubernetes.io/ja/docs/concepts/overview/)

### コンテナとは

![コンテナはアプリと必要なものを1つの箱にしたもの](/images/k8s-basics-kind/02-container.png)

Kubernetes が動かすのはコンテナです。コンテナはアプリと、アプリが必要とするもの（言語の実行環境やライブラリ）を1つの箱にまとめたものです。

コンテナがないと、同じマシンに入れたアプリ同士が部品を取り合います。例えばアプリ A は Python 3.8 を、アプリ B は Python 3.12 を必要とします。箱に分けておけば、それぞれが自分の部品を持ち込めます。

用語を3つ押さえておきます。

| 用語 | 意味 | 例 |
|---|---|---|
| イメージ | 箱の設計図 | `nginx:1.27-alpine` |
| コンテナ | イメージから起動した実物 | 動いている nginx |
| コンテナランタイム | コンテナを起動・停止するソフト | containerd |

:::details Docker との関係
Docker はコンテナを作って動かすための道具一式です。Kubernetes はコンテナランタイムとして containerd などを使い、Docker 本体は必要ありません（v1.24 で dockershim が外れました）。Docker で作ったイメージはそのまま動かすことができます。
:::

@[card](https://kubernetes.io/ja/docs/concepts/containers/)

### ノードとクラスタ

![マシン1台がノード、まとめてクラスタ](/images/k8s-basics-kind/03-cluster.png)

コンテナを1台のマシンだけで動かすと、そのマシンが壊れたときに全部止まります。そこで Kubernetes は何台かのマシンを束ねて使います。

- **ノード**：束ねたマシン1台のことです。物理マシンでも VM でも構いません。
- **クラスタ**：ノードをまとめたもの全体です。

ノードには2つの役があります。

- **コントロールプレーン**：司令塔です。どこで何を動かすかを決め、見張ります。自分ではアプリを動かしません。
- **ワーカーノード**：働き手です。実際にコンテナを動かします（コンテナは Pod という箱に入れて動かします。次で説明します）。

:::message
「マスターノード」は、コントロールプレーンが載ったノードの古い呼び名です。今の公式ドキュメントでは「コントロールプレーン」と呼びます。古い記事でマスターと書いてあれば、同じものと読み替えてください。
:::

この記事の kind では、Docker コンテナ1つが1台のノードになります。コントロールプレーン1台とワーカーノード2台の、計3台で組みました。

@[card](https://kubernetes.io/ja/docs/concepts/architecture/nodes/)

### Pod：コンテナを入れる箱

![コンテナは Pod という箱に入って動く](/images/k8s-basics-kind/04-pod.png)

ノードの上で、コンテナはそのまま動くわけではありません。**Pod** という箱に入れて動かします。Kubernetes が置き場所を決めたり作り直したりする単位も、コンテナではなく Pod です。

たいていは1つの Pod に1つのコンテナです。いつも一緒に動くべきもの同士（アプリ本体とログを送る係など）は、同じ Pod に入れることもあります。(計装など)
**同じ Pod のコンテナは IP を共有し、保存領域（Volume）も共有でき、必ず同じノードで動きます。**

Pod は使い捨てです。消えると IP も一緒に消え、作り直された Pod は別の IP になります。
> この性質は、後で出てくる Service につながります。

ここまでの入れ子をまとめると次のとおりです。

```
クラスタ
 └ ノード（コントロールプレーン／ワーカーノード）
    └ Pod
       └ コンテナ（1つ以上）
```

@[card](https://kubernetes.io/ja/docs/concepts/workloads/pods/)

### Namespace：クラスタの中の部屋分け

![Namespace はクラスタの中の部屋分け](/images/k8s-basics-kind/05-namespace.png)

上の入れ子とは別に、**Namespace** という区切りがあります。クラスタの中を部屋に分け、Pod や Service をどこかの部屋に入れます。

**Namespace ごとに、使える CPU やメモリの上限（ResourceQuota）や触れる人（RBAC）を決められます。そのため、チームや開発・本番の区切りとしてよく使われます。**

@[card](https://kubernetes.io/ja/docs/concepts/overview/working-with-objects/namespaces/)

### アーキテクチャ：誰が何をしているか

![Kubernetes のアーキテクチャ](/images/k8s-basics-kind/06-architecture.png)

コントロールプレーンとワーカーノードの中身を開くと、上の図のようになります。

**コントロールプレーンの部品**

| 部品 | 役割 |
|---|---|
| kube-apiserver | すべての操作の受付窓口。kubectl もほかの部品もここに話しかける |
| etcd | クラスタの状態を保存するデータベース（キーバリューストア） |
| kube-scheduler | 新しい Pod をどのノードに置くか決める |
| kube-controller-manager | あるべき数と今の数を比べ、ずれを直し続ける |
| cloud-controller-manager | クラウドのロードバランサーなどとつなぐ。kind にはない |

**ワーカーノードの部品**^[実際にはコントロールプレーンのノードでも kubelet が動いています。②で見るように、司令塔の部品自体も Pod として動くためです。]

| 部品 | 役割 |
|---|---|
| kubelet | 自分のノードに割り当てられた Pod を、コンテナランタイムに頼んで動かす |
| kube-proxy | Service 宛ての通信を Pod へ振り分けるルールを書き込む |
| コンテナランタイム | コンテナを実際に起動・停止する（kind では containerd） |

部品どうしは直接やりとりしません。全員が kube-apiserver を通して状態を読み書きします。etcd に触れるのも kube-apiserver だけです。

@[card](https://kubernetes.io/ja/docs/concepts/overview/components/)

### Kubernetes をどう操作するのか

ここまでで、窓口が kube-apiserver だと分かりました。では、私たちはどうやって kube-apiserver に話しかけるのでしょうか？　使うのは次の2つです。

- **マニフェスト**：あるべき姿を書いた YAML ファイルです。「どのコンテナを・いくつ・どう動かすか」を書きます。
- **kubectl**：kube-apiserver に話しかけるコマンドです。マニフェストを渡したり、今の様子を聞いたりします。

マニフェストを kubectl で渡せば、あとは上の図の部品たちが動かしてくれます。

マニフェストの例です。「nginx を3つ動かし続けて」という内容で、後半で使う web.yaml から Deployment の部分を抜き出しています。

```yaml:web.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
      - name: nginx
        image: nginx:1.27-alpine
```

:::details 1行ずつの意味
| 行 | 意味 |
|---|---|
| `apiVersion: apps/v1` | 書式のバージョン。作るものの種類ごとに決まっている |
| `kind: Deployment` | kindは名詞で種類の方です。k8sリソースを選択しています(後述)|
| `metadata.name: web` | 名前。kubectl で指すときに使う |
| `spec.replicas: 3` | いくつ動かすか。ここが「3つあるべき」と指定しています |
| `spec.selector` | どの Pod を自分の担当とみなすか。ラベル `app: web` の付いた Pod が対象 |
| `spec.template` | 作る Pod のひな形。ここで付けるラベルは selector と一致させる |
| `containers[].image` | 動かすイメージ。ここでは nginx |


:::

渡すときは `kubectl apply` を使います。 -f はファイルを指定しています。

```sh
kubectl apply -f web.yaml
```

`apply` は「このファイルの内容をあるべき姿にして」という意味です。ファイルを直して同じコマンドを打てば、差分だけが反映されます。

よく使うコマンドです。

| コマンド | やること |
|---|---|
| `kubectl apply -f <ファイル>` | マニフェストを渡す（作成・更新） |
| `kubectl get pods` | Pod の一覧と状態を見る |
| `kubectl describe pod <Pod名>` | 1つの Pod の詳細とイベントを見る |
| `kubectl logs <Pod名>` | コンテナのログを見る |
| `kubectl delete -f <ファイル>` | マニフェストで作ったものを消す |

どれも裏では kube-apiserver に HTTP でリクエストを送っています。話しかける先のクラスタは `~/.kube/config` に書いてあり、kind はクラスタを作るときにここへ接続先を書き込みます。

@[card](https://kubernetes.io/ja/docs/concepts/overview/working-with-objects/)

@[card](https://kubernetes.io/ja/docs/reference/kubectl/)

### kubectl apply から Pod が動くまで

![kubectl apply から Pod が動くまで](/images/k8s-basics-kind/07-apply-flow.png)

では `kubectl apply -f web.yaml` を打ったとき、裏では何が起きているのでしょうか？　部品を順番に追います。

1. kubectl がマニフェストを kube-apiserver に送る
2. kube-apiserver が中身を確かめ、etcd に保存する
3. kube-controller-manager が「3つあるべきなのに0」と気づき、Pod を3つ登録する
4. kube-scheduler が置き場所の決まっていない Pod を見つけ、ノードを割り当てる
5. 割り当てられたノードの kubelet がそれに気づき、コンテナランタイムに起動を頼む
6. kubelet が「動いた」と kube-apiserver に報告する

:::details もう少し詳しく
kubectl に「受け付けた」と返るのは2の時点です。コマンドが返ってきても、コンテナはまだ動いていません。`kubectl get pods` で見える `Pending` は4の前、`ContainerCreating` は5の途中です。

3は実際には2段階です。Deployment を見張る係が ReplicaSet を作り、ReplicaSet を見張る係が Pod を作ります。どちらも kube-controller-manager の中にいます。この親子関係は③で実際に確かめます。
:::

@[card](https://kubernetes.io/ja/docs/concepts/architecture/controller/)

### アプリを動かす4つの部品

![アプリを動かす4つの部品](/images/k8s-basics-kind/08-workload.png)

最後に、マニフェストに書く側の部品です。Pod は前に見た箱で、ほかの3つはそれを扱うための部品です。

| 部品 | 役割 | 自分で書くか |
|---|---|---|
| Pod | コンテナを入れる箱。1つ以上のコンテナと IP を持つ | 普段は単体で書かない（Deployment の `template` に書く） |
| ReplicaSet | 同じ Pod を指定した数だけそろえる | 書かない |
| Deployment | ReplicaSet を管理し、更新と巻き戻しを担う | 書く |
| Service | 入れ替わる Pod の前に置く、変わらない名前と IP | 書く |

`kind: Pod` と書けば Pod を単体でも作れます。ただし単体の Pod は、消えたりノードごと落ちたりしても誰も作り直しません。そのため普段は Deployment の `template` に Pod のひな形を書き、作り直しは ReplicaSet に任せます。単体で作るのは、⑤の `client` のような使い捨ての確認用くらいです。

Pod は作り直すたびに IP が変わります。そのため Pod に直接つなぐのではなく、前に置いた Service につなぎます。振り分け先にする Pod の IP 一覧は、**EndpointSlice** という部品に入っています。

@[card](https://kubernetes.io/ja/docs/concepts/workloads/controllers/deployment/)

@[card](https://kubernetes.io/ja/docs/concepts/workloads/controllers/replicaset/)

@[card](https://kubernetes.io/ja/docs/concepts/services-networking/service/)

### 道具

| 道具 | 役割 |
|---|---|
| kubectl | kube-apiserver を操作する CLI。サーバーとの差は ±1 マイナーバージョンまでしか対応しない |
| マニフェスト | あるべき姿を書いた YAML |
| kind | Docker コンテナをノードに見立て、手元にクラスタを作るツール |

公式の説明は [Cluster Architecture](https://kubernetes.io/docs/concepts/architecture/) と [Kubernetes Components](https://kubernetes.io/docs/concepts/overview/components/) にあります。

## 作った構成

```
Windows 11 / WSL2 Ubuntu (docker-ce)
└─ kind クラスタ "k8s-basics"（kindest/node:v1.37.0）
   ├─ k8s-basics-control-plane … apiserver / etcd / scheduler / controller-manager
   ├─ k8s-basics-worker        … kubelet ─ Pod(nginx)
   └─ k8s-basics-worker2       … kubelet ─ Pod(nginx)
         ▲
         │ kubectl v1.37.0
       ターミナル
```

kind の設定は3ノード分のイメージを固定するだけです。

```yaml:kind-config.yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
  image: kindest/node:v1.37.0@sha256:a1ed56cfb0e7b93589bdf97c8cd566405a265939e3620fc4f5de89adff580ae5
- role: worker
  image: kindest/node:v1.37.0@sha256:a1ed56cfb0e7b93589bdf97c8cd566405a265939e3620fc4f5de89adff580ae5
- role: worker
  image: kindest/node:v1.37.0@sha256:a1ed56cfb0e7b93589bdf97c8cd566405a265939e3620fc4f5de89adff580ae5
```

アプリは nginx を3つと、その前に置く Service です。

```yaml:web.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
      - name: nginx
        image: nginx:1.27-alpine
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: web
spec:
  selector:
    app: web
  ports:
  - port: 80
    targetPort: 80
```

## やったこと

### ① クラスタを作る

```sh
kind create cluster --name k8s-basics --config kind-config.yaml
```

イメージの pull 込みで73秒かかりました。直後の様子です（実ログ・抜粋）。

```
$ docker ps
NAMES                      IMAGE                  STATUS
k8s-basics-control-plane   kindest/node:v1.37.0   Up 25 seconds
k8s-basics-worker2         kindest/node:v1.37.0   Up 25 seconds
k8s-basics-worker          kindest/node:v1.37.0   Up 25 seconds
$ kubectl get nodes
NAME                       STATUS     ROLES           AGE   VERSION
k8s-basics-control-plane   NotReady   control-plane   16s   v1.37.0
k8s-basics-worker          NotReady   <none>          3s    v1.37.0
k8s-basics-worker2         NotReady   <none>          3s    v1.37.0
```

`docker ps` に並ぶ3つのコンテナが、3台のノードです。kind ではノードを Docker コンテナで作っています。

ノードには、kubelet とコンテナランタイムが動くマシンなら何でも使えます。手元以外では、次のようなものがノードになります。

| 環境 | ノードになるもの |
|---|---|
| 手元（kind） | Docker コンテナ |
| オンプレミス | 物理サーバーや VM |
| Amazon EKS | EC2 インスタンス。Fargate を使う場合は、Pod ごとに AWS が用意するマシン |
| Google GKE | Compute Engine の VM |

作った直後は3台とも NotReady でした。Pod 同士をつなぐネットワーク（kind では kindnet）が入るまでは Ready になりません。数秒待つと Ready に変わりました。

### ② 司令塔も Pod として動いている

```
$ kubectl get pods -n kube-system -o wide（抜粋）
NAME                                               READY   STATUS    NODE
etcd-k8s-basics-control-plane                      1/1     Running   k8s-basics-control-plane
kube-apiserver-k8s-basics-control-plane            1/1     Running   k8s-basics-control-plane
kube-controller-manager-k8s-basics-control-plane   1/1     Running   k8s-basics-control-plane
kube-scheduler-k8s-basics-control-plane            1/1     Running   k8s-basics-control-plane
coredns-559f6c778d-r28sd                           1/1     Running   k8s-basics-control-plane
kube-proxy-7x546                                   1/1     Running   k8s-basics-worker
kube-proxy-qwczm                                   1/1     Running   k8s-basics-worker2
kindnet-rncpw                                      1/1     Running   k8s-basics-worker
```

「アーキテクチャ」で見た司令塔の部品が、control-plane ノードの上に Pod として並んでいます。どれも kube-system の部屋に入っています。kube-proxy と kindnet は各ノードに1つずつあります。

### ③ Deployment は ReplicaSet を作り、ReplicaSet が Pod を作る

```sh
kubectl apply -f web.yaml
```

Pod が誰に作られたかは `ownerReferences` に書いてあります（実ログ）。

```
pod/web-6bd858c6f9-k7rl2 <- ReplicaSet/web-6bd858c6f9
pod/web-6bd858c6f9-mmsc9 <- ReplicaSet/web-6bd858c6f9
pod/web-6bd858c6f9-qbnd9 <- ReplicaSet/web-6bd858c6f9
replicaset/web-6bd858c6f9 <- Deployment/web
```

3つの Pod は worker に1つ、worker2 に2つ置かれました。どこに置くかは kube-scheduler が決めるので、こちらからは指定していません。

### ④ Pod を消しても1秒で戻る

1つ消して、3つ Running に戻るまでを測りました。

```
$ kubectl delete pod/web-6bd858c6f9-k7rl2
3つ Running に戻るまで: 1.0s
```

イベントを見ると、作り直したのが ReplicaSet だと分かります（実ログ・抜粋）。

```
Normal   Killing            pod/web-6bd858c6f9-k7rl2    Stopping container nginx
Normal   SuccessfulCreate   replicaset/web-6bd858c6f9   Created pod: web-6bd858c6f9-5tk79
Normal   Scheduled          pod/web-6bd858c6f9-5tk79    Successfully assigned default/web-6bd858c6f9-5tk79 to k8s-basics-worker
Normal   Started            pod/web-6bd858c6f9-5tk79    Container started
```

「消された」と気づいて作り直したのではありません。「3つあるべきなのに2つしかない」というずれを見つけて、1つ足しています。

### ⑤ Pod の IP は変わっても Service の IP は変わらない

別の Pod（busybox）から、Service の名前 `web` で nginx に届くか確かめます。

```
$ kubectl exec client -- wget -qO- http://web | grep title
<title>Welcome to nginx!</title>
```

次に `kubectl rollout restart` で Pod を全部作り直し、IP を比べました（実ログ）。

```
before: 10.244.1.3 10.244.1.2 10.244.2.3
after : 10.244.2.5 10.244.2.6 10.244.1.4 10.244.1.2
svc IP: 10.96.102.183
```

新しい Pod の IP はどれも前と違います（10.244.1.2 は終了中の古い Pod）。Service の IP（10.96.102.183）は最初から同じです。呼ぶ側は Service の名前だけ知っていれば済みます。

### ⑥ 更新に失敗しても古い Pod は残る

動いている nginx を、存在しないイメージに更新してみます（タグの打ち間違いを想定）。

```sh
kubectl set image deploy/web nginx=nginx:9.99-doesnotexist
```

`rollout status --timeout=60s` は `error: timed out waiting for the condition` になりました。そのときの様子です（実ログ）。

```
$ kubectl get pods -l app=web
NAME                   READY   STATUS         RESTARTS   AGE
web-67bd6dbcb4-dpl24   1/1     Running        0          88s
web-67bd6dbcb4-sk675   1/1     Running        0          88s
web-67bd6dbcb4-wjnvx   1/1     Running        0          87s
web-79ccc95497-s4tjn   0/1     ErrImagePull   0          73s
```

新しい Pod は1つしか作られず、pull に失敗して止まっています。古い3つは Running のままで、この間も `wget http://web` は届きました。止まったのは更新だけで、サービスは止まっていません。

Deployment は新旧を少しずつ入れ替えるので、最初の1つが立ち上がらなければ次に進みません。

戻すのは1コマンドです。

```
$ kubectl rollout undo deploy/web
deployment.apps/web rolled back
image: nginx:1.27-alpine
```

### ⑦ ノードが落ちるとどうなる

Pod が2つ載っている worker2 を、`docker stop` で丸ごと止めました。5秒おきに様子を見ています（実ログ・抜粋）。

```
$ tolerations
node.kubernetes.io/not-ready NoExecute 300
node.kubernetes.io/unreachable NoExecute 300
$ docker stop k8s-basics-worker2
+54s ノード NotReady
+354s 別ノードで3つ Running
Normal  TaintManagerEviction  pod/web-67bd6dbcb4-dpl24  Marking for deletion Pod default/web-67bd6dbcb4-dpl24
Normal  SuccessfulCreate      replicaset/web-67bd6dbcb4 Created pod: web-67bd6dbcb4-85r9s
```

時間の内訳は次のとおりです。

1. ノードから返事がなくなり、NotReady と判定されるまで **54秒**
2. Pod に付いている toleration「ノードと連絡が取れなくても **300秒** は待つ」が切れるまで待つ
3. 切れたら Pod を退避させ、ReplicaSet が別のノードに作り直す

300秒はノードの一時的な瞬断で Pod を動かさないための既定値です。短くしたいときは Pod の `tolerationSeconds` で変更できます。

:::details なぜ Pod の削除は1秒で、ノード障害は6分なのか
Pod の削除は kube-apiserver への操作なので、その場で記録されます。コントローラーは kube-apiserver に「変化があったら教えて」という接続（watch）を張っているため、すぐ通知を受けて作り直せます。

ノードの停止は、誰も kube-apiserver に知らせません。知らせる係はそのノードの kubelet で、本人が止まっているためです。kubelet は10秒おきに「生きている」と書き込んでおり、コントロールプレーンはこれが50秒途絶えたら NotReady と判定します（確認は5秒おき）。これが54秒の内訳です。

残りの300秒は、気づいたあとにわざと待っている時間です。
:::

作り直された2つは残った worker(ノード) に置かれ、3つとも worker 1台に集まりました（control-plane にはアプリの Pod を置かないため）
> 万一、残りのノードも止まった場合には完全に停止します

```
web-67bd6dbcb4-85r9s   Running   k8s-basics-worker
web-67bd6dbcb4-sc5q2   Running   k8s-basics-worker
web-67bd6dbcb4-sk675   Running   k8s-basics-worker
```

**止まっている間、リクエストはどうなるか**
NotReady になった54秒の時点で worker2 の Pod が Service の振り分け先から外れ、残る1つに届きます。それまでの54秒は、一部が止まったノードへ送られて失敗します。今回は測っていませんが。

## 結果のまとめ

④〜⑦で壊したものと、戻るまでの時間をまとめます。

| 壊したもの | 起きたこと | 戻るまで | 効いた仕組み |
|---|---|---|---|
| Pod を1つ削除 | 別の Pod がすぐ作られた | **1.0秒** | ReplicaSet |
| Pod を全部作り直し | Pod の IP は全部変わった。Service の IP は同じ | — | Service |
| 存在しないタグに更新 | 新しい Pod が1つ止まり、古い3つは動き続けた | `rollout undo` で即 | Deployment |
| ワーカーノードを停止 | 54秒で NotReady、Pod の退避は354秒後 | **約6分** | taint と toleration（300秒） |

Pod 単体の故障なら一瞬で戻ります。ノードごと落ちたときは、既定の設定のままだと6分近く戻りませんでしたね。

## やってみて

k8sの基礎から実際に壊してどう動くかまでをまとめてみました。
k8sは固有の概念が多いように見えますが、その概念は他の技術でも使うことが多いです。

自己修復を支える仕組みはリコンサイル（reconcile）と呼ばれ、Reactのレンダリング技術でも出てきたり、serviceの振り分けはLB(L4の)のようなものだとイメージした方もいるのではないでしょうか。

k8sを学ぶことで、インフラ領域における堅牢性の高いシステムの実践的な動作を知ることができると思います。
k8sは利用者が多く、OSSで提供されているため、多くのサードパーティツールがあります。
いつか別の記事でそういったツールをまとめたいと思います。


## 参考

- [Kubernetes Components](https://kubernetes.io/docs/concepts/overview/components/)
- [Deployments](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/)
- [Service](https://kubernetes.io/docs/concepts/services-networking/service/)
- [Taints and Tolerations（taint based evictions）](https://kubernetes.io/docs/concepts/scheduling-eviction/taint-and-toleration/#taint-based-evictions)
- [Version Skew Policy](https://kubernetes.io/releases/version-skew-policy/)
- [kind Quick Start](https://kind.sigs.k8s.io/docs/user/quick-start/)
