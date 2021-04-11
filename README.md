# Day07-scale
## 事前作業
本日の演習問題では、クラスタモニタリング機能を利用します。
CRCでは使用可能なメモリの値がデフォルトで9GB(9216MiB)になっていますが、クラスタモニタリングを有効にするには、14GiB(14336MiB)以上でないと動作しないためメモリ割り当てを変更します。

```
$ crc config set memory 14336
```

クラスタモニタリング機能を有効化します。
```
$ crc config set enable-cluster-monitoring true
```

CRCを起動します。
```
$ crc start
```

また、MacBookにwatchコマンドをインストールしていない場合はインストールしてください。
```
$ brew install watch
```
## レプリケーションコントローラー
レプリケーションコントローラーは、ポッドの指定された数のレプリカが常に実行されるよう保証します。

レプリケーションコントローラーは、ポッドが強制終了された場合、または管理者によって明示的に削除された場合に、ポッドをインスタンス化します。

同様に、必要な数よりも多いポッドが動作していた場合、必要に応じてポッドを削除して、指定されたレプリカの件数に合わせます。

レプリケーションコントローラーの定義には主に次のものがあります。 

- 必要とされるレプリカ数
- レプリケートされたポッドを作成するためのポッド定義
- 管理されたポッドを特定するためのセレクター

セレクターはラベルのセットであり、レプリケーションコントローラーによって管理されるすべてのポッドが一致する必要があります。

```
apiVersion: apps/v1 kind: Deployment ...output omitted... spec:
  replicas: 1　①
  selector:
    matchLabels:
      app: scale　②
  strategy: {}
  template:　③
metadata:
      labels:
        app: scale　④
    spec:
      containers:
...output omitted...
```

- ①実行するポッドのコピー (またはレプリカ) 数を指定し
- ②レプリケーションコントローラーは、サービスがセレクターを使用して負荷分散するポッドを検出するのと同様に、セレクターを使用して実行中のポッドの数をカウントします。
- ③コントローラーが作成するポッドのテンプレートです。
- ④レプリケーションコントローラーによって作成されるポッドのラベルは、セレクターのものと一致する必要があります。

### アプリケーションのレプリカ数の変更
DeploymentリソースやReplicationControllerリソースのレプリカ数は`kubectl scale`コマンドを使用して動的に変更可能です。
```
$ kubectl get deployment
NAME   READY UP-TO-DATE AVAILABLE AGE
scale  1/1   1          1         8h

$ kubectl scale --replicas 5 deployment scale
```

### ポッドのオートスケーリング
Kubernetesでは、アプリケーションポッドの現在の負荷に基づいて、HorizontalPodAutoscalerというリソースタイプを使用してオートスケーリングすることができます。

HorizontalPodAutoscaler リソースを作成するには、次の例のように kubectl autoscale コマンドを使用する方法があります。

```
$ kubectl autoscale deployment hello --min 1 --max 10 --cpu-percent 80
```

前述のコマンドでは HorizontalPodAutoscaler リソースが作成されます。

これは hello デプロイ設定のレプリカの数を変更し、ポッドが要求されたCPU合計使用量の 80% 未満に保たれます。

HorizontalPodAutoscaler リソースの最小値または最大値には、負荷の急増に適応し、OpenShift クラスターの過負荷を避ける役割があります。

アプリケーションの負荷の変化が速すぎる場合、ユーザーリクエストの急増に対処するために予備のポッドをいくつか保持しておくことが推奨されます。

反対に、ポッドの数が多すぎる場合、すべてのクラスターの空き容量を使い果たし、同じ OpenShift クラスターを共有する他 のアプリケーションに影響を与える可能性があります。

現在のプロジェクトの HorizontalPodAutoscaler リソースに関する情報を取得するには、kubectl get コマンド と kubectl describe コマンドを使用します。

次に例を示します。
```
$ kubectl get hpa frontend
NAME      REFERENCE                              TARGET  CURRENT  MINPODS  MAXPODS  AGE
frontend  DeploymentConfig/myapp/frontend/scale  80%     59%      1        10       8d

$ kubectl describe hpa frontend
Name:                     frontend
Namespace:                myapp
Labels:                   <none>
CreationTimestamp:        Mon, 26 Jul 2017 21:13:47 -0400
Reference:                Deployment/myapp/frontend/scale
Target CPU utilization:   80%
Current CPU utilization:  59%
Min pods:                 1
Max pods:                 10
```
## 演習問題１
- Namespace：scaleを作成
- loadtestというアプリケーションPodを作成
- Pod数を5に増やす
- Pod数を1に減らす
- HPAを定義
- アプリケーションに負荷を与える
- Podが自動的に増えることを確認
- scalingというアプリケーションPodを作成
- Pod数を3に増やす
- Podを増やして正常に負荷分散されていることを確認


```
$ kubectl create namespace scale
$ kubectl config set-context crc-developer --namespace=scale
$ kubectl config use-context crc-developer
```

下記のマニフェストをコピーし、`loadtest.yaml`というファイルに保存します。
```
apiVersion: v1
items:
- apiVersion: apps/v1
  kind: Deployment
  metadata:
    labels:
      app: loadtest
    name: loadtest
  spec:
    replicas: 1
    selector:
      matchLabels:
        app: loadtest
    strategy: {}
    template:
      metadata:
        labels:
          app: loadtest
      spec:
        containers:
        - image: nakanakau/loadtest:latest
          name: loadtest
          resources:
            requests:
              cpu: "25m"
              memory: 25Mi
            limits:
              cpu: "100m"
              memory: 100Mi
  status: {}

- apiVersion: v1
  kind: Service
  metadata:
    labels:
      app: loadtest
    name: loadtest
  spec:
    ports:
    - port: 80
      protocol: TCP
      targetPort: 8080
    selector:
      app: loadtest
  status:
    loadBalancer: {}

- apiVersion: route.openshift.io/v1
  kind: Route
  metadata:
    annotations:
      haproxy.router.openshift.io/timeout: 120s
    labels:
      app: loadtest
    name: loadtest
  spec:
    host: ""
    port:
      targetPort: 8080
    subdomain: ""
    to:
      kind: ""
      name: loadtest
      weight: null
  status:
    ingress: null
kind: List
metadata: {}
```

上記マニフェストからリソースを作成します。
```
$ kubectl apply -f loadtest.yaml
```
Podを5つに増やします。
```
$ kubectl scale --replicas 5 deployment loadtest
```
Podが増えたことを確認します。
```
$ kubectl get pod -w
NAME                       READY   STATUS    RESTARTS   AGE
loadtest-978ccb589-cqpnc   1/1     Running   0          23s
loadtest-978ccb589-pnhzt   1/1     Running   0          23s
loadtest-978ccb589-qlg4m   1/1     Running   0          23s
loadtest-978ccb589-qpzx8   1/1     Running   0          23s
loadtest-978ccb589-qv82h   1/1     Running   0          2m50s
```
Podを1つに戻します。
```
$ kubectl scale --replicas 1 deployment loadtest
NAME                       READY   STATUS        RESTARTS   AGE
loadtest-978ccb589-cqpnc   1/1     Terminating   0          86s
loadtest-978ccb589-pnhzt   1/1     Terminating   0          86s
loadtest-978ccb589-qlg4m   1/1     Terminating   0          86s
loadtest-978ccb589-qpzx8   1/1     Terminating   0          86s
loadtest-978ccb589-qv82h   1/1     Running       0          3m53s
```
Horizontal Pod Autoscalerを定義します。

最小Pod：2、最大Pod：10、CPU閾値を50%と定義します。
```
$ kubectl autoscale deployment loadtest --min 2 --max 10 --cpu-percent 50
```
アプリケーションにアクセスするためのアドレスを確認します。
```
$ kubectl get route loadtest
NAME       HOST/PORT                         PATH   SERVICES   PORT   TERMINATION   WILDCARD
loadtest   loadtest-scale.apps-crc.testing          loadtest   8080                 None
```

アプリケーションにアクセスします。
下記のURLは、loadtestアプリケーションに負荷をかけるためのものです。
```
$ curl -X GET http://loadtest-scale.apps-crc.testing/api/loadtest/v1/cpu/1
```
`command+N`で新規ターミナルを起動し、メトリクスを確認します。
```
$ watch kubectl get hpa loadtest
```
下記のように、TARGETSのCPU使用率が上昇し、REPLICAS列の値が徐々にスケールアウトしていくことが確認できます。
```
Every 2.0s: kubectl get hpa loadtest                                                               AKFVFZT2Z8LYWP.local: Mon Apr 12 03:18:47 2021

NAME       REFERENCE             TARGETS    MINPODS   MAXPODS   REPLICAS   AGE
loadtest   Deployment/loadtest   192%/50%   2         10        8          12m
```

下記のマニフェストをコピーし、`scaling.yaml`というファイルで保存します。
```
apiVersion: v1
items:
- apiVersion: apps/v1
  kind: Deployment
  metadata:
    labels:
      app: scaling
    name: scaling
  spec:
    replicas: 1
    selector:
      matchLabels:
        app: scaling
    strategy: {}
    template:
      metadata:
        labels:
          app: scaling
      spec:
        containers:
        - image: nakanakau/scaling:latest
          name: scaling
  status: {}

- apiVersion: v1
  kind: Service
  metadata:
    labels:
      app: scaling
    name: scaling
  spec:
    ports:
    - port: 80
      protocol: TCP
      targetPort: 8080
    selector:
      app: scaling
  status:
    loadBalancer: {}

- apiVersion: route.openshift.io/v1
  kind: Route
  metadata:
    annotations:
      haproxy.router.openshift.io/timeout: 120s
    labels:
      app: scaling
    name: scaling
  spec:
    host: ""
    port:
      targetPort: 8080
    subdomain: ""
    to:
      kind: ""
      name: scaling
      weight: null
  status:
    ingress: null
kind: List
metadata: {}
```
上記のマニフェストからリソースを作成します。
```
$ kubectl apply -f scaling.yaml
```
routeを確認してアプリケーションにアクセスします。
```
$ kubectl get route scaling
NAME      HOST/PORT                        PATH   SERVICES   PORT   TERMINATION   WILDCARD
scaling   scaling-scale.apps-crc.testing          scaling    8080                 None
```
```
$ curl -s http://scaling-scale.apps-crc.testing
Server IP: 10.217.0.87 
```
Podに割り振られたCluster IPが表示されます。

Podを3に増やし、再度アプリケーションにアクセスします。
```
$ kubectl scale --replicas 3 deployment scaling
```
下記はscalingアプリケーションに30回のアクセスを試行するものです。
```
$ for x in {1..30}; do curl -s http://scaling-scale.apps-crc.testing; done;
Server IP: 10.217.0.87 
Server IP: 10.217.0.88 
Server IP: 10.217.0.89 
・・・
```
増やしたPodに対してもアクセスが分散されていることが確認できます。