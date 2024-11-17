# kubectlコマンド チートシート

## k8sクラスタ構築

`kind create cluster`

## クラスタ接続確認

`kubectl cluster-info --context kind-kind`

## マニフェスト適用

`kubectl apply -f <マニフェストファイル>`

## ノード確認

`kubectl get node`

## リソースの情報を取得する

`kubectl get pod --namespace default`

- podを指定する
`kubectl get pod <Pod名> --namespace default`
- yaml形式で出力
`kubectl get pod --output yaml --namespace default`
- jqと組み合わせた例(コンテナイメージを取得)
`kubectl get pod <Pod名> --output json --namespace default | jq 'spec.containers[].image'`

## リソースの詳細情報を取得する

`kubectl describe pod <Pod名> --namespace default`

## コンテナのログを取得する

`kubectl logs <Pod名> --namespace default`

- コンテナが複数ある場合
`kubectl logs <Pod名> -c <コンテナ名> --namespace default`

## 特定のDeploymentに紐づくPodのログを参照する

`kubectl logs deploy/<Deployment名>`

## ラベルを指定して参照するPodを絞り込む

`kubectl get pod --selector(-1) <ラベル名>=<値>`

## デバッグ用のサイドカーコンテナを立ち上げる

`kubectl debug stdin --tty <デバッグ対象のPod名> --image=<デバッグ用コンテナのImage> --target=<デバッグ対象のコンテナ名>`

```bash
kubectl debug --stdin --tty myapp --image=curlimages/curl:8.4.0 --target=hello-server  --namespace default -- sh
# 実行結果
# Targeting container "hello-server". If you don't see processes from this container it may be because the container runtime doesn't support this feature.
# Defaulting debug container name to debugger-bvj6n.
# If you don't see a command prompt, try pressing enter.
# ~ $ exit
# Session ended, the ephemeral container will not be restarted but may be reattached using 'kubectl attach myapp -c debugger-bvj6n -i -t' if it is still running
```

## コンテナを即座に実行する

`kubectl run <Pod名> --image=<イメージ名>`

`busybox`というPodを起動し、`nslookup`を実行して終了する例

```bash
kubectl --namespace default run busybox --image=busybox:1.36.1 --rm --stdin --tty --restart=Never --command -- nslookup google.com
```

メモ：`--stdin` `--tty`は省略して`-it`とも書ける。
`--rm` : コンテナが終了したらPodを削除する
`--stdin(-i)` : 標準入力に渡す
`--tty(-t)` : 疑似端末を割り当てる
`--restart=Never` : コンテナが終了したら再起動しない（デフォルトでは常に再起動する）
`--command --` : `--`の後に渡される引数をコマンドとして解釈する

## コンテナにログインする

`kubectl exec --it <Pod名> -- <コマンド名>`

### execの利用例 : アプリケーションがインターネットからアクセスできなくなったときにクラスタ内からIPアドレスでアクセスできるか確認することで切り分けに役立つ

1. ログイン用のPodを起動
`kubectl --namespace default run curlpod --image=curlimages/curl:8.4.0 --command -- /bin/sh -c "while true; do sleep infinity; done"`

2. 作成を確認
`kubectl get pod curlpod`

3. すでに起動しているmyappというPodのIPアドレスを取得
`kubectl get pod myapp -o jsonpath='{.status.podIP}'`
⇒ `10.244.0.7`

4. １で起動したcurlpodにログイン
`kubectl --namespace default exec -it curlpod -- /bin/sh`

5. curlコマンドでmyappにアクセス

```bash
curl 10.244.0.7:8080
# 実行結果
# Hello, world!
```

## port-forwardでアプリケーションにアクセス

`kubectl port-forward <Pod名> <転送先ポート番号名>:<転送先ポート番号>`

PodにはK8sクラスタ内部用のIPアドレスが割り当てられる。
何もしないとクラスタ外からアクセスできない。

**5555をクラスタ内の8080に割り当て：**
`kubectl port-forward myapp 5555:8080 --namespace default`
**別のターミナルからアクセス：**`curl localhost:5555`

## リソースの削除

`kubectl delete <リソース名>`

kubectlにはPodを再起動するコマンドがないため、`kubectl delete`で代替することが多い。

## Tips : 自動補完について

公式ドキュメント
<https://kubernetes.io/docs/tasks/tools/install-kubectl-linux/#enable-shell-autocompletion>

### bashの場合

```bash
sudo apt-get install bash-completion
source /usr/share/bash-completion/bash_completion
echo 'source <(kubectl completion bash)' >>~/.bashrc
source ~/.bashrc
```

## Tips : kubectlのエイリアス

`kubectl`を`k`として使う

```bash
echo 'alias k=kubectl' >>~/.bashrc
source ~/.bashrc
```

## ReplicaSet

Podを複数起動するためのリソース

`kubectl apply --filename <マニフェストファイル> --namespace default`

確認：`kubectl get pod --namespace default`

ReplicaSetのリソースを直接参照する。: `kubectl get replicaset --namespace default`

```text
NAME         DESIRED   CURRENT   READY   AGE
httpserver   3         3         3       2m16s
```

削除：`kubectl delete replicaset <ReplicaSet名> --namespace default`
