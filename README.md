# kindの操作

## ローカル環境(Mac)でのセットアップ

### インストール

```bash
$ brew install kind

$ kind version
kind v0.23.0 go1.22.3 darwin/amd64
```

### クラスタ作成

DockerHubのkindest/nodeリポジトリを使用できる

```bash
$ kind create cluster --image=kindest/node:v1.29.0
Thanks for using kind! 😊
```

dockerのコンテナとしてクラスタ環境が作成される

![alt text](<s1.png>)

### 状態確認

```bash
$ kubectl cluster-info --context kind-kind
Kubernetes control plane is running at https://127.0.0.1:53902
CoreDNS is running at https://127.0.0.1:53902/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy

To further debug and diagnose cluster problems, use 'kubectl cluster-info dump'.
```

### kubectlのconfig

k8sのクラスタアップデート時に操作する可能性あり

場所：` ~/.kube/config`

```bash
$ cat ~/.kube/config
apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: 省略
    server: https://127.0.0.1:53902
  name: kind-kind
contexts: //クラスタの設定情報ごとに名前のついたコンテキストが作成される
- context:
    cluster: kind-kind
    user: kind-kind
  name: kind-kind
current-context: kind-kind
kind: Config
preferences: {}
users:
- name: kind-kind
  user:
    client-certificate-data: 省略
    client-key-data: 省略
```

複数クラスタに接続する際の接続情報もここに記載

`kubectl cluster-info --context kind-kind`の`context`は↑の定義から取得している

`kucectl config use-context`で`--context`オプションをつけないときのデフォルトコンテキストを指定可能(`kubectx`で代替可能)

### クラスタの削除

```bash
$ kind delete cluster
Deleting cluster "kind" ...
Deleted nodes: ["kind-control-plane"]
```

## k8sについて

### マニュフェスト

`.yaml/.yml`を拡張子とするYML形式のファイルで表される。

ファイルには動作させたいリソースの**仕様**を記載する。※NGINXというコンテナを動作させたい、など

JSON形式で記載することも可能

- k8sではマニュフェストというYMLファイルを使ってコンテナを起動する
- マニュフェストを起動するために**kubectl**を使う
  - kubectlコマンドでk8sクラスタと通信する

### Podについて

コンテナを起動するための最小構成リソース=Pod

Podは複数コンテナをまとめて起動できる

下記はまとめて一つのPodとして起動することが多い

- Aというサービス
- ログを転送するサービス
  - メインのサービスに付帯するプログラム=サイドカー

```sample
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  containers:
  - name: nginx
    image: nginx:1.25.3
    ports:
    - containerPort: 80
```

マニュフェストに指定できるオプションは[リファレンス](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.29/)を参考

### Namespace

リソースを作成するための場所=Namespace

- 単一クラスタ内のリソースを分離するメカニズムを提供
- リソースの名前はNamespace内では一位である必要がある

#### kube-system Namespaceについて

Control PlaneやWorker Nodeで起動するk8sのシステムコンポーネントのPodが利用するNamespace

```bash
$ kubectl get pod --namespace kube-system
NAME                                         READY   STATUS    RESTARTS   AGE
etcd-kind-control-plane                      1/1     Running   0          13s
kube-apiserver-kind-control-plane            1/1     Running   0          13s
kube-controller-manager-kind-control-plane   1/1     Running   0          13s
kube-scheduler-kind-control-plane            1/1     Running   0          13s
```

### k8sの起動確認

```bash
$ kubectl get nodes
NAME                 STATUS   ROLES           AGE   VERSION
kind-control-plane   Ready    control-plane   21m   v1.29.0
```

#### kindの場合

```bash
$ kind get clusters
kind // デフォルトで作成されるクラスタ名はkind
```

### リソースの確認

```bash
$ kubectl get pod --namespace default
No resources found in default namespace.
```

### マニュフェストの適用

```kind/myapp.yml
apiVersion: v1
kind: Pod
metadata:
  name: myapp
  labels:
    app: myapp
spec:
  containers:
  - name: hello-server
    image: blux2/hello-server:1.0
    ports:
    - containerPort: 8080
```

#### `kubectl apply`コマンドによるマニュフェストの作成

```bash
$ kubectl apply --filename kind/myapp.yml --namespace default
pod/myapp created
$ kubectl get pod --namespace default
NAME    READY   STATUS    RESTARTS   AGE
myapp   1/1     Running   0          47s
```

#### `kubectl run`コマンドによる実行 ※非推奨

```bash
$ kubectl run myapp2 --image=blux2/hello-server:1.0 --namespace default
pod/myapp2 created
```

- マニュフェストがあった方が差分が明確
- podの冗長化など高度な設定ができない
- デバッグなどの一時利用だけで使用する

## トラブルシューティング

`kubectl`でトラブルシューティングできるようになること

![alt text](<s2.png>)


### STATUSについて

```bash
$ kubectl get pod --namespace default
NAME     READY   STATUS    RESTARTS   AGE
myapp    1/1     Running   0          55m
myapp2   1/1     Running   0          46m
```

| STATUS | 内容 |
|-|-|
|Pending|k8sからPodの作成は許可されたが、ひとつ以上のコンテナが準備中。長時間この状態の場合は異常の場合あり|
|Running|Podがノードにスケジュールされ、すべてのコンテナが作成された状態。常時起動が想定されるPodであれば正常な状態|
|Completed|Podの全てのコンテナが完了した状態。再起動はしない|
|Unknown|何らかの原因でPodの状態が取得できなかった。Podが実行されるべきノードとの通信エラーで発生する|
|ErrImagePull|Imageの取得で失敗したことを表している。|
|Error|コンテナが異常終了した状態。|
|OOMKilled|コンテナがOOM(Out Of Memory)で終了した状態。Podのリソースを増やすことを検討|
|Terminating|Podが削除中の状態。繰り返す場合は異常の可能性あり|
