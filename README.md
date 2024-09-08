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

- `kubectl get <リソース名>`でリソースの情報を取得
- `--namespace(-n)`オプションでNamespaceを指定して実行
  - 省略可能
- `default` Namespaceはクラスタ作成時に自動で作成される
  - 実運用では使わない
- `kubectl get pod <Pod名>`でPodなど特定のリソースを指定して実行可能

```bash
$ kubectl get pod myapp --namespace default
NAME    READY   STATUS    RESTARTS   AGE
myapp   1/1     Running   0          5d23h
```

- `--output(-o)`オプションを使うとリソースの情報をさまざまな方法で取得可能

<details>
<summary>IPアドレスやNode情報を取得できる(wide)</summary>

```bash
$ kubectl get pod myapp --output wide --namespace default
NAME    READY   STATUS    RESTARTS   AGE     IP           NODE                 NOMINATED NODE   READINESS GATES
myapp   1/1     Running   0          5d23h   10.244.0.5   kind-control-plane   <none>           <none>
```

</details>

<details><summary>YAML形式でリソースの情報を取得できる(yaml)</summary>

```yaml
$ kubectl get pod myapp --output yaml --namespace default
apiVersion: v1
kind: Pod
metadata:
  annotations:
    kubectl.kubernetes.io/last-applied-configuration: |
      {"apiVersion":"v1","kind":"Pod","metadata":{"annotations":{},"labels":{"app":"myapp"},"name":"myapp","namespace":"default"},"spec":{"containers":[{"image":"blux2/hello-server:1.0","name":"hello-server","ports":[{"containerPort":8080}]}]}}
  creationTimestamp: "2024-08-15T15:02:31Z"
  labels:
    app: myapp
  name: myapp
  namespace: default
  resourceVersion: "2981"
  uid: 8f268551-e44d-46c2-b4d8-85954d4f439b
spec:
  containers:
  - image: blux2/hello-server:1.0
    imagePullPolicy: IfNotPresent
    name: hello-server
    ports:
    - containerPort: 8080
      protocol: TCP
    resources: {}
    terminationMessagePath: /dev/termination-log
    terminationMessagePolicy: File
    volumeMounts:
    - mountPath: /var/run/secrets/kubernetes.io/serviceaccount
      name: kube-api-access-m6ww4
      readOnly: true
  dnsPolicy: ClusterFirst
  enableServiceLinks: true
  nodeName: kind-control-plane
  preemptionPolicy: PreemptLowerPriority
  priority: 0
  restartPolicy: Always
  schedulerName: default-scheduler
  securityContext: {}
  serviceAccount: default
  serviceAccountName: default
  terminationGracePeriodSeconds: 30
  tolerations:
  - effect: NoExecute
    key: node.kubernetes.io/not-ready
    operator: Exists
    tolerationSeconds: 300
  - effect: NoExecute
    key: node.kubernetes.io/unreachable
    operator: Exists
    tolerationSeconds: 300
  volumes:
  - name: kube-api-access-m6ww4
    projected:
      defaultMode: 420
      sources:
      - serviceAccountToken:
          expirationSeconds: 3607
          path: token
      - configMap:
          items:
          - key: ca.crt
            path: ca.crt
          name: kube-root-ca.crt
      - downwardAPI:
          items:
          - fieldRef:
              apiVersion: v1
              fieldPath: metadata.namespace
            path: namespace
status:
  conditions:
  - lastProbeTime: null
    lastTransitionTime: "2024-08-15T15:02:39Z"
    status: "True"
    type: PodReadyToStartContainers
  - lastProbeTime: null
    lastTransitionTime: "2024-08-15T15:02:31Z"
    status: "True"
    type: Initialized
  - lastProbeTime: null
    lastTransitionTime: "2024-08-15T15:02:39Z"
    status: "True"
    type: Ready
  - lastProbeTime: null
    lastTransitionTime: "2024-08-15T15:02:39Z"
    status: "True"
    type: ContainersReady
  - lastProbeTime: null
    lastTransitionTime: "2024-08-15T15:02:31Z"
    status: "True"
    type: PodScheduled
  containerStatuses:
  - containerID: containerd://29fd348fdd36fb72736fa125c58109d3a15769a75672ff371fd5ab44a98476b7
    image: docker.io/blux2/hello-server:1.0
    imageID: docker.io/blux2/hello-server@sha256:35ab584cbe96a15ad1fb6212824b3220935d6ac9d25b3703ba259973fac5697d
    lastState: {}
    name: hello-server
    ready: true
    restartCount: 0
    started: true
    state:
      running:
        startedAt: "2024-08-15T15:02:39Z"
  hostIP: 172.18.0.2
  hostIPs:
  - ip: 172.18.0.2
  phase: Running
  podIP: 10.244.0.5
  podIPs:
  - ip: 10.244.0.5
  qosClass: BestEffort
  startTime: "2024-08-15T15:02:31Z"
```

</details>

yaml形式での出力は、lessなどと合わせて検索に利用するために使うことが多い

```bash
kubectl get pod myapp -o yaml -n default | less
```

自分がapplyしたマニュフェストとの差分を見る際に便利

```bash
kubectl get pod myapp -o yaml -n default > pod.yml
```

- マニュフェストをjson形式で出力し、欲しい情報のみ取得する

```bash
$ kubectl get pod myapp --output jsonpath='{.spec.containers[].image}'
blux2/hello-server:1.0
```

```bash
$ kubectl get pod myapp --output json --namespace default | jq '.spec.containers[].image'
"blux2/hello-server:1.0"
```

### リソースの詳細

`kubectl get`より詳しい情報が欲しい場合

```bash
kubectl describe pod myapp
```

### コンテナのログの取得

```bash
kubectl logs <Pod名>
```

```bash
$ kubectl logs myapp -c hello-server -n default
2024/08/15 15:02:39 Starting server on port 8080

```

### ログレベル変更

```bash
kubectl get pod <Pod名> --v=<ログレベル>
```

<details><summary>ログレベル7</summary>

kubectlとkube-apiserverはクライアントとAPIの関係性のため、--v=7でログを確認するとRESTのリクエストが確認できる
```bash
$ kubectl get pod --v~7 --namespace default7
error: unknown flag: --v~7
See 'kubectl get --help' for usage.
MacBook-Air:bbf-kubernetes taiga$
MacBook-Air:bbf-kubernetes taiga$ kubectl get pod --v=7 --namespace default7
I0825 12:15:18.503770   24779 loader.go:395] Config loaded from file:  /Users/taiga/.kube/config
I0825 12:15:18.507613   24779 round_trippers.go:463] GET https://127.0.0.1:55931/api?timeout=32s
I0825 12:15:18.507621   24779 round_trippers.go:469] Request Headers:
I0825 12:15:18.507628   24779 round_trippers.go:473]     Accept: application/json;g=apidiscovery.k8s.io;v=v2beta1;as=APIGroupDiscoveryList,application/json
I0825 12:15:18.507634   24779 round_trippers.go:473]     User-Agent: kubectl/v1.29.2 (darwin/arm64) kubernetes/4b8e819
I0825 12:15:18.524496   24779 round_trippers.go:574] Response Status: 200 OK in 16 milliseconds
I0825 12:15:18.526759   24779 round_trippers.go:463] GET https://127.0.0.1:55931/apis?timeout=32s
I0825 12:15:18.526766   24779 round_trippers.go:469] Request Headers:
I0825 12:15:18.526772   24779 round_trippers.go:473]     User-Agent: kubectl/v1.29.2 (darwin/arm64) kubernetes/4b8e819
I0825 12:15:18.526777   24779 round_trippers.go:473]     Accept: application/json;g=apidiscovery.k8s.io;v=v2beta1;as=APIGroupDiscoveryList,application/json
I0825 12:15:18.528646   24779 round_trippers.go:574] Response Status: 200 OK in 1 milliseconds
I0825 12:15:18.542668   24779 round_trippers.go:463] GET https://127.0.0.1:55931/api/v1/namespaces/default7/pods?limit=500
I0825 12:15:18.542684   24779 round_trippers.go:469] Request Headers:
I0825 12:15:18.542691   24779 round_trippers.go:473]     Accept: application/json;as=Table;v=v1;g=meta.k8s.io,application/json;as=Table;v=v1beta1;g=meta.k8s.io,application/json
I0825 12:15:18.542697   24779 round_trippers.go:473]     User-Agent: kubectl/v1.29.2 (darwin/arm64) kubernetes/4b8e819
I0825 12:15:18.546712   24779 round_trippers.go:574] Response Status: 200 OK in 4 milliseconds
No resources found in default7 namespace.
```

</details>

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

### デバッグ用のサイドカーコンテナの立ち上げ

コンテナの軽量化により、デバッグに必要なツールやシェルが入っていない場合のため

```bash
kubectl debug --stdin --tty <デバッグ対象Pod名> --image=<デバッグ用のコンテナのイメージ> --target=<デバッグ対象のコンテナ名>
```

#### curlのデバッグ用コンテナの立ち上げ

```bash
$ kubectl debug --stdin --tty myapp --image=curlimages/curl:8.4.0 --target=hello-server --namespace default -- sh
Targeting container "hello-server". If you don't see processes from this container it may be because the container runtime doesn't support this feature.
Defaulting debug container name to debugger-zk4b7.
If you don't see a command prompt, try pressing enter.

~ $ curl localhost:8080
Hello, world!~ $
~ $
```

デバッグ対象のPodと同じネットワーク、同じボリュームをそれぞれローカルネットワーク、ローカルボリュームとして参照できるため便利。

#### コンテナを即座に実行する

busyboxというPodを起動し、nslookupコマンドを実行したら終了する

```bash
kubectl --namespace default run busybox --image=busybox:1.36.1 --rm --stdin --tty --restart=Never --command -- nslookup google.com
Server:		10.96.0.10
Address:	10.96.0.10:53

Non-authoritative answer:
Name:	google.com
Address: 142.251.42.142

Non-authoritative answer:
Name:	google.com
Address: 2404:6800:4004:825::200e

pod "busybox" deleted
```

- `--rm`：実行が完了したらPodを削除する
- `-stdin(-i)`：オプションで標準入力に渡す
- `--tty()-t`：オプションで擬似端末を割り当てる
- `--restart=Never`：Podの再起動ポリシーをNeverに設定する。コンテナが終了しても再起動を行わない。デフォルトでは常に再起動するポリシーのため、↑のようにコマンドを一度だけ実行する場合はこの設定を入れる必要がある。
- `--command -- ""`：--の後に渡される拡張引数の一つ目が引数ではなくコマンドとして扱われる。

`--stdin`と`--tty`は２つを同時に省略して`-it`とだけ書く場合が多い

### コンテナにログインする

```bash
kubectl exec --stdin --tty <Pod名> -- <コマンド名>
```

`/bin/sh`を渡すことでコンテナにシェルが入っていれば直接ログインが可能。

ただし、シェルが入っていないことも多いためどんなPodに対しても使えるわけではない。

#### コンテナを作成してコマンドを実行する検証

```bash
$ kubectl --namespace default run curlpod --image=curlimages/curl:8.4.0 --command -- /bin/sh -c "while true; do sleep infinity;done;"
pod/curlpod created

$ kubectl get pod -n default
NAME      READY   STATUS    RESTARTS   AGE
curlpod   1/1     Running   0          5m45s
myapp     1/1     Running   0          16d

myappコンテナのIPアドレスを取得
$ kubectl get pod myapp -o wide -n default
NAME    READY   STATUS    RESTARTS   AGE   IP           NODE                 NOMINATED NODE   READINESS GATES
myapp   1/1     Running   0          16d   10.244.0.5   kind-control-plane   <none>           <none>

curlpodに接続し、取得したIPにcurlを実行
$ kubectl -n default exec -it curlpod -- /bin/sh
~ $ curl 10.244.0.5:8080
Hello, world!
```

アプリケーションがインターネット上から繋がらなくなった際に、クラスタ内からIPアドレスでアクセスできるか確認することで問題の切り分けが可能。

### port-forwardでアプリケーションにアクセス

何も設定しないとクラスタ内部のIPに対してアクセスができないが、port-forwardで手軽に接続することも可能。

```bash
kubectl port-forward <Pod名> <転送先ポート番号>:<転送元ポート番号>
```

ローカル環境とk8sクラスタ間でポートフォワーディングを設定

- 5555:8080⇨ローカルマシンの5555ポートへのアクセスをPodの8080に転送
  - ローカルマシンのポート5555をリッスン
  - Podのポート8080にトラフィックの転送
  - Podからのレスポンスをローカルマシンの5555で受け取り

```bash
$ kubectl port-forward myapp 5555:8080 -n default
Forwarding from 127.0.0.1:5555 -> 8080
Forwarding from [::1]:5555 -> 8080

他のターミナルから実施
$ curl localhost:5555
Hello, world!

$ kubectl port-forward myapp 5555:8080 -n default
Forwarding from 127.0.0.1:5555 -> 8080
Forwarding from [::1]:5555 -> 8080
Handling connection for 5555
```

転送先ポート(↑5555)は省略することも可能(ランダムで選ばれる)

指定する場合はローカルで使用されていないポートを選ぶ

### 障害を直すためのコマンド

※環境に変更を加えるため本番環境では慎重に実行する

#### リソースマニュフェストを直接修正する

```bash
kubectl edit
```

簡単に修正できる反面、修正履歴を残しにくいため推奨されていない。

ローカル環境を使うケースでもなるべく修正前のマニュフェストを保存しておき、修正後のマニュフェストをapplyする方が望ましい。

```bash
$ kubectl edit pod myapp -n default
Edit cancelled, no changes made.
```

#### リソースを削除する


