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

## Deployment

ReplicaSetを管理するリソース。本番環境で利用されることが多い。

Deploymentを作成する: `kubectl apply --filename <マニフェストファイル> --namespace default`

Deploymentが作成できているか確認：`kubectl get deployment --namespace default`

Podが作成されているか確認：`kubectl get pod --namespace default`

replicasetが作成されているか確認：`kubectl get replicaset --namespace default`

deploymentを更新して再度適用する：

```bash
# マニフェストファイルを更新(作成と同じ)
kubectl apply --filename <マニフェストファイル> --namespace default
# レプリカセットは新しいものに置き換わる
kubectl get replicaset --namespace default
# NAME                          DESIRED   CURRENT   READY   AGE
# nginx-deployment-575b7dcc5c   0         0         0       108s
# nginx-deployment-5d694b9887   3         3         3       11s
```

deploymentの詳細情報を確認する：`kubectl describe deployment <Deployment名> --namespace default`

Deployment名は`nginx-deployment`が相当する。

### Deploymentのstrategy

- `Recreate` : 古いPodを削除してから新しいPodを作成する
  - 参考ファイルは`deployment-recreate.yml`
  - 以下の操作で実験する
    1. `kubectl apply --filename deployment-recreate.yml --namespace default` ('nginxのimageを1.24.0に設定しておく)
    2. 別ターミナルを開いて`kubectl get pod --watch --namespace default`
    3. `kubectl apply --filename deployment-recreate.yml --namespace default` ('nginxのimageを1.25.0に設定しておく)
    4. 1.24.0のPodが削除され、1.25.0のPodが作成される
  - 一気にPodを更新するため、一時的にサービスが停止するが更新は早い

``` bash
$ kubectl get pod --watch --namespace default
# NAME                                READY   STATUS    RESTARTS   AGE
# nginx-deployment-575b7dcc5c-9n6gq   1/1     Running   0          48s
# nginx-deployment-575b7dcc5c-bc7tz   1/1     Running   0          48s
# nginx-deployment-575b7dcc5c-bvd6c   1/1     Running   0          48s
# nginx-deployment-575b7dcc5c-cjmrh   1/1     Running   0          48s
# nginx-deployment-575b7dcc5c-dkzj4   1/1     Running   0          48s
# nginx-deployment-575b7dcc5c-mbzjv   1/1     Running   0          48s
# nginx-deployment-575b7dcc5c-pslcw   1/1     Running   0          48s
# nginx-deployment-575b7dcc5c-rbtc9   1/1     Running   0          48s
# nginx-deployment-575b7dcc5c-v4hmv   1/1     Running   0          48s
# nginx-deployment-575b7dcc5c-x8sg7   1/1     Running   0          48s
# nginx-deployment-575b7dcc5c-cjmrh   1/1     Terminating   0          76s
# nginx-deployment-575b7dcc5c-9n6gq   1/1     Terminating   0          76s
# nginx-deployment-575b7dcc5c-mbzjv   1/1     Terminating   0          76s
# nginx-deployment-575b7dcc5c-pslcw   1/1     Terminating   0          76s
# nginx-deployment-575b7dcc5c-v4hmv   1/1     Terminating   0          76s
# nginx-deployment-575b7dcc5c-x8sg7   1/1     Terminating   0          76s
# nginx-deployment-575b7dcc5c-bc7tz   1/1     Terminating   0          76s
# nginx-deployment-575b7dcc5c-bvd6c   1/1     Terminating   0          76s
# nginx-deployment-575b7dcc5c-dkzj4   1/1     Terminating   0          76s
# nginx-deployment-575b7dcc5c-rbtc9   1/1     Terminating   0          76s
# nginx-deployment-575b7dcc5c-x8sg7   0/1     Completed     0          76s
# nginx-deployment-575b7dcc5c-cjmrh   0/1     Completed     0          76s
# nginx-deployment-575b7dcc5c-pslcw   0/1     Completed     0          76s
# nginx-deployment-575b7dcc5c-dkzj4   0/1     Completed     0          76s
# nginx-deployment-575b7dcc5c-9n6gq   0/1     Completed     0          76s
# nginx-deployment-575b7dcc5c-bvd6c   0/1     Completed     0          76s
# nginx-deployment-575b7dcc5c-bc7tz   0/1     Completed     0          76s
# nginx-deployment-575b7dcc5c-rbtc9   0/1     Completed     0          76s
# nginx-deployment-575b7dcc5c-mbzjv   0/1     Completed     0          76s
# nginx-deployment-575b7dcc5c-v4hmv   0/1     Completed     0          76s
# nginx-deployment-575b7dcc5c-dkzj4   0/1     Completed     0          76s
# nginx-deployment-575b7dcc5c-dkzj4   0/1     Completed     0          76s
# nginx-deployment-575b7dcc5c-cjmrh   0/1     Completed     0          76s
# nginx-deployment-575b7dcc5c-cjmrh   0/1     Completed     0          76s
# nginx-deployment-575b7dcc5c-mbzjv   0/1     Completed     0          76s
# nginx-deployment-575b7dcc5c-mbzjv   0/1     Completed     0          76s
# nginx-deployment-575b7dcc5c-rbtc9   0/1     Completed     0          76s
# nginx-deployment-575b7dcc5c-rbtc9   0/1     Completed     0          76s
# nginx-deployment-575b7dcc5c-9n6gq   0/1     Completed     0          76s
# nginx-deployment-575b7dcc5c-9n6gq   0/1     Completed     0          76s
# nginx-deployment-575b7dcc5c-x8sg7   0/1     Completed     0          76s
# nginx-deployment-575b7dcc5c-x8sg7   0/1     Completed     0          76s
# nginx-deployment-575b7dcc5c-bvd6c   0/1     Completed     0          76s
# nginx-deployment-575b7dcc5c-bvd6c   0/1     Completed     0          76s
# nginx-deployment-575b7dcc5c-bc7tz   0/1     Completed     0          76s
# nginx-deployment-575b7dcc5c-bc7tz   0/1     Completed     0          76s
# nginx-deployment-575b7dcc5c-v4hmv   0/1     Completed     0          76s
# nginx-deployment-575b7dcc5c-v4hmv   0/1     Completed     0          76s
# nginx-deployment-575b7dcc5c-pslcw   0/1     Completed     0          77s
# nginx-deployment-575b7dcc5c-pslcw   0/1     Completed     0          77s
# nginx-deployment-5d694b9887-9xlqx   0/1     Pending       0          0s
# nginx-deployment-5d694b9887-74rfn   0/1     Pending       0          0s
# nginx-deployment-5d694b9887-hd4jz   0/1     Pending       0          0s
# nginx-deployment-5d694b9887-9xlqx   0/1     Pending       0          0s
# nginx-deployment-5d694b9887-74rfn   0/1     Pending       0          0s
# nginx-deployment-5d694b9887-tmr8k   0/1     Pending       0          0s
# nginx-deployment-5d694b9887-6df7l   0/1     Pending       0          0s
# nginx-deployment-5d694b9887-6lm9w   0/1     Pending       0          0s
# nginx-deployment-5d694b9887-pjgwc   0/1     Pending       0          0s
# nginx-deployment-5d694b9887-hd4jz   0/1     Pending       0          0s
# nginx-deployment-5d694b9887-6df7l   0/1     Pending       0          0s
# nginx-deployment-5d694b9887-pjgwc   0/1     Pending       0          0s
# nginx-deployment-5d694b9887-6lm9w   0/1     Pending       0          0s
# nginx-deployment-5d694b9887-tmr8k   0/1     Pending       0          0s
# nginx-deployment-5d694b9887-w55r8   0/1     Pending       0          0s
# nginx-deployment-5d694b9887-dlld5   0/1     Pending       0          0s
# nginx-deployment-5d694b9887-pxxgs   0/1     Pending       0          0s
# nginx-deployment-5d694b9887-9xlqx   0/1     ContainerCreating   0          0s
# nginx-deployment-5d694b9887-dlld5   0/1     Pending             0          0s
# nginx-deployment-5d694b9887-pxxgs   0/1     Pending             0          0s
# nginx-deployment-5d694b9887-w55r8   0/1     Pending             0          0s
# nginx-deployment-5d694b9887-74rfn   0/1     ContainerCreating   0          0s
# nginx-deployment-5d694b9887-hd4jz   0/1     ContainerCreating   0          0s
# nginx-deployment-5d694b9887-6df7l   0/1     ContainerCreating   0          0s
# nginx-deployment-5d694b9887-w55r8   0/1     ContainerCreating   0          0s
# nginx-deployment-5d694b9887-tmr8k   0/1     ContainerCreating   0          0s
# nginx-deployment-5d694b9887-dlld5   0/1     ContainerCreating   0          0s
# nginx-deployment-5d694b9887-6lm9w   0/1     ContainerCreating   0          0s
# nginx-deployment-5d694b9887-pjgwc   0/1     ContainerCreating   0          0s
# nginx-deployment-5d694b9887-pxxgs   0/1     ContainerCreating   0          0s
# nginx-deployment-5d694b9887-6df7l   1/1     Running             0          0s
# nginx-deployment-5d694b9887-74rfn   1/1     Running             0          0s
# nginx-deployment-5d694b9887-hd4jz   1/1     Running             0          0s
# nginx-deployment-5d694b9887-tmr8k   1/1     Running             0          0s
# nginx-deployment-5d694b9887-9xlqx   1/1     Running             0          0s
# nginx-deployment-5d694b9887-dlld5   1/1     Running             0          0s
# nginx-deployment-5d694b9887-6lm9w   1/1     Running             0          0s
# nginx-deployment-5d694b9887-pjgwc   1/1     Running             0          0s
# nginx-deployment-5d694b9887-pxxgs   1/1     Running             0          0s
# nginx-deployment-5d694b9887-w55r8   1/1     Running             0          0s
```

- `RollingUpdate` : 古いPodを削除しながら新しいPodを作成する
  - 参考ファイルは`deployment-rollingupdate.yml`
  - 以下の操作で実験する
    1. `kubectl apply --filename deployment-rollingupdate.yml --namespace default` ('nginxのimageを1.24.0に設定しておく)
    2. 別ターミナルを開いて`kubectl get pod --watch --namespace default`
    3. `kubectl apply --filename deployment-rollingupdate.yml --namespace default` ('nginxのimageを1.25.0に設定しておく)
    4. 1.24.0のPodが削除され、1.25.0のPodが作成される
  - 一時的にサービスが停止することなく更新が行われる
  - 途中でPodの数が倍になる(maxSurge: 100%)の設定のため

``` bash
kubectl get pod --watch --namespace default
# NAME                               READY   STATUS    RESTARTS   AGE
# nginx-deployment-fb7c9b74d-6x728   0/1     Pending   0          0s
# nginx-deployment-fb7c9b74d-6x728   0/1     Pending   0          0s
# nginx-deployment-fb7c9b74d-l6fjs   0/1     Pending   0          0s
# nginx-deployment-fb7c9b74d-7phwq   0/1     Pending   0          0s
# nginx-deployment-fb7c9b74d-5hhnx   0/1     Pending   0          0s
# nginx-deployment-fb7c9b74d-xvb68   0/1     Pending   0          0s
# nginx-deployment-fb7c9b74d-7phwq   0/1     Pending   0          0s
# nginx-deployment-fb7c9b74d-l6fjs   0/1     Pending   0          0s
# nginx-deployment-fb7c9b74d-t9642   0/1     Pending   0          0s
# nginx-deployment-fb7c9b74d-6x728   0/1     ContainerCreating   0          0s
# nginx-deployment-fb7c9b74d-gc6rq   0/1     Pending             0          0s
# nginx-deployment-fb7c9b74d-ttbzg   0/1     Pending             0          0s
# nginx-deployment-fb7c9b74d-xvb68   0/1     Pending             0          0s
# nginx-deployment-fb7c9b74d-7bcw8   0/1     Pending             0          0s
# nginx-deployment-fb7c9b74d-5hhnx   0/1     Pending             0          0s
# nginx-deployment-fb7c9b74d-mj7t9   0/1     Pending             0          0s
# nginx-deployment-fb7c9b74d-gc6rq   0/1     Pending             0          0s
# nginx-deployment-fb7c9b74d-t9642   0/1     Pending             0          0s
# nginx-deployment-fb7c9b74d-7phwq   0/1     ContainerCreating   0          0s
# nginx-deployment-fb7c9b74d-mj7t9   0/1     Pending             0          0s
# nginx-deployment-fb7c9b74d-ttbzg   0/1     Pending             0          0s
# nginx-deployment-fb7c9b74d-7bcw8   0/1     Pending             0          0s
# nginx-deployment-fb7c9b74d-l6fjs   0/1     ContainerCreating   0          0s
# nginx-deployment-fb7c9b74d-7bcw8   0/1     ContainerCreating   0          0s
# nginx-deployment-fb7c9b74d-xvb68   0/1     ContainerCreating   0          0s
# nginx-deployment-fb7c9b74d-5hhnx   0/1     ContainerCreating   0          0s
# nginx-deployment-fb7c9b74d-mj7t9   0/1     ContainerCreating   0          0s
# nginx-deployment-fb7c9b74d-t9642   0/1     ContainerCreating   0          0s
# nginx-deployment-fb7c9b74d-gc6rq   0/1     ContainerCreating   0          0s
# nginx-deployment-fb7c9b74d-ttbzg   0/1     ContainerCreating   0          0s
# nginx-deployment-fb7c9b74d-ttbzg   1/1     Running             0          0s
# nginx-deployment-fb7c9b74d-7bcw8   1/1     Running             0          0s
# nginx-deployment-fb7c9b74d-xvb68   1/1     Running             0          0s
# nginx-deployment-fb7c9b74d-7phwq   1/1     Running             0          0s
# nginx-deployment-fb7c9b74d-5hhnx   1/1     Running             0          0s
# nginx-deployment-fb7c9b74d-mj7t9   1/1     Running             0          0s
# nginx-deployment-fb7c9b74d-t9642   1/1     Running             0          0s
# nginx-deployment-fb7c9b74d-6x728   1/1     Running             0          0s
# nginx-deployment-fb7c9b74d-gc6rq   1/1     Running             0          0s
# nginx-deployment-fb7c9b74d-l6fjs   1/1     Running             0          0s
# nginx-deployment-849fbfc6db-49tl8   0/1     Pending             0          0s
# nginx-deployment-849fbfc6db-49tl8   0/1     Pending             0          0s
# nginx-deployment-849fbfc6db-mfdk7   0/1     Pending             0          0s
# nginx-deployment-849fbfc6db-bdcz8   0/1     Pending             0          0s
# nginx-deployment-849fbfc6db-sh4hk   0/1     Pending             0          0s
# nginx-deployment-849fbfc6db-kbtlh   0/1     Pending             0          0s
# nginx-deployment-849fbfc6db-krvk5   0/1     Pending             0          0s
# nginx-deployment-849fbfc6db-jhr62   0/1     Pending             0          0s
# nginx-deployment-849fbfc6db-mfdk7   0/1     Pending             0          0s
# nginx-deployment-849fbfc6db-bdcz8   0/1     Pending             0          0s
# nginx-deployment-fb7c9b74d-ttbzg    1/1     Terminating         0          24s
# nginx-deployment-849fbfc6db-rlcml   0/1     Pending             0          0s
# nginx-deployment-849fbfc6db-7lsfs   0/1     Pending             0          0s
# nginx-deployment-849fbfc6db-fvzz2   0/1     Pending             0          0s
# nginx-deployment-849fbfc6db-kbtlh   0/1     Pending             0          0s
# nginx-deployment-849fbfc6db-krvk5   0/1     Pending             0          0s
# nginx-deployment-849fbfc6db-jhr62   0/1     Pending             0          0s
# nginx-deployment-849fbfc6db-sh4hk   0/1     Pending             0          0s
# nginx-deployment-fb7c9b74d-l6fjs    1/1     Terminating         0          24s
# nginx-deployment-849fbfc6db-49tl8   0/1     ContainerCreating   0          0s
# nginx-deployment-849fbfc6db-rlcml   0/1     Pending             0          0s
# nginx-deployment-849fbfc6db-7lsfs   0/1     Pending             0          0s
# nginx-deployment-849fbfc6db-fvzz2   0/1     Pending             0          0s
# nginx-deployment-849fbfc6db-mfdk7   0/1     ContainerCreating   0          0s
# nginx-deployment-849fbfc6db-kbtlh   0/1     ContainerCreating   0          0s
# nginx-deployment-849fbfc6db-krvk5   0/1     ContainerCreating   0          0s
# nginx-deployment-849fbfc6db-bdcz8   0/1     ContainerCreating   0          0s
# nginx-deployment-849fbfc6db-sh4hk   0/1     ContainerCreating   0          0s
# nginx-deployment-849fbfc6db-jhr62   0/1     ContainerCreating   0          0s
# nginx-deployment-849fbfc6db-fvzz2   0/1     ContainerCreating   0          0s
# nginx-deployment-849fbfc6db-rlcml   0/1     ContainerCreating   0          0s
# nginx-deployment-849fbfc6db-7lsfs   0/1     ContainerCreating   0          0s
# nginx-deployment-849fbfc6db-rlcml   1/1     Running             0          0s
# nginx-deployment-849fbfc6db-krvk5   1/1     Running             0          0s
# nginx-deployment-849fbfc6db-49tl8   1/1     Running             0          0s
# nginx-deployment-fb7c9b74d-6x728    1/1     Terminating         0          24s
# nginx-deployment-849fbfc6db-bdcz8   1/1     Running             0          0s
# nginx-deployment-849fbfc6db-sh4hk   1/1     Running             0          0s
# nginx-deployment-fb7c9b74d-7phwq    1/1     Terminating         0          24s
# nginx-deployment-fb7c9b74d-5hhnx    1/1     Terminating         0          24s
# nginx-deployment-849fbfc6db-jhr62   1/1     Running             0          0s
# nginx-deployment-849fbfc6db-fvzz2   1/1     Running             0          0s
# nginx-deployment-849fbfc6db-mfdk7   1/1     Running             0          0s
# nginx-deployment-849fbfc6db-kbtlh   1/1     Running             0          0s
# nginx-deployment-849fbfc6db-7lsfs   1/1     Running             0          0s
# nginx-deployment-fb7c9b74d-xvb68    1/1     Terminating         0          25s
# nginx-deployment-fb7c9b74d-7bcw8    1/1     Terminating         0          25s
# nginx-deployment-fb7c9b74d-t9642    1/1     Terminating         0          25s
# nginx-deployment-fb7c9b74d-mj7t9    1/1     Terminating         0          25s
# nginx-deployment-fb7c9b74d-gc6rq    1/1     Terminating         0          25s
# nginx-deployment-fb7c9b74d-ttbzg    0/1     Completed           0          34s
# nginx-deployment-fb7c9b74d-l6fjs    0/1     Completed           0          34s
# nginx-deployment-fb7c9b74d-ttbzg    0/1     Completed           0          34s
# nginx-deployment-fb7c9b74d-ttbzg    0/1     Completed           0          34s
# nginx-deployment-fb7c9b74d-l6fjs    0/1     Completed           0          34s
# nginx-deployment-fb7c9b74d-l6fjs    0/1     Completed           0          34s
# nginx-deployment-fb7c9b74d-6x728    0/1     Completed           0          35s
# nginx-deployment-fb7c9b74d-5hhnx    0/1     Completed           0          35s
# nginx-deployment-fb7c9b74d-7phwq    0/1     Completed           0          35s
# nginx-deployment-fb7c9b74d-xvb68    0/1     Completed           0          35s
# nginx-deployment-fb7c9b74d-7bcw8    0/1     Completed           0          35s
# nginx-deployment-fb7c9b74d-t9642    0/1     Completed           0          35s
# nginx-deployment-fb7c9b74d-mj7t9    0/1     Completed           0          35s
# nginx-deployment-fb7c9b74d-gc6rq    0/1     Completed           0          35s
# nginx-deployment-fb7c9b74d-7phwq    0/1     Completed           0          35s
# nginx-deployment-fb7c9b74d-7phwq    0/1     Completed           0          35s
# nginx-deployment-fb7c9b74d-7bcw8    0/1     Completed           0          35s
# nginx-deployment-fb7c9b74d-7bcw8    0/1     Completed           0          35s
# nginx-deployment-fb7c9b74d-xvb68    0/1     Completed           0          35s
# nginx-deployment-fb7c9b74d-xvb68    0/1     Completed           0          35s
# nginx-deployment-fb7c9b74d-mj7t9    0/1     Completed           0          35s
# nginx-deployment-fb7c9b74d-mj7t9    0/1     Completed           0          35s
# nginx-deployment-fb7c9b74d-5hhnx    0/1     Completed           0          35s
# nginx-deployment-fb7c9b74d-5hhnx    0/1     Completed           0          35s
# nginx-deployment-fb7c9b74d-6x728    0/1     Completed           0          35s
# nginx-deployment-fb7c9b74d-6x728    0/1     Completed           0          35s
# nginx-deployment-fb7c9b74d-t9642    0/1     Completed           0          35s
# nginx-deployment-fb7c9b74d-t9642    0/1     Completed           0          35s
# nginx-deployment-fb7c9b74d-gc6rq    0/1     Completed           0          36s
# nginx-deployment-fb7c9b74d-gc6rq    0/1     Completed           0          36s
```
