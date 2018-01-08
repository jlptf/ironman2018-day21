## Day 21 - 常見問題與建議 (2)

### 本日共賞

* 超出資源限制 (Exceeding CPU/Memory Limit)
* 資源配額

### 希望你知道

* [與 k8s 溝通: kubectl](https://ithelp.ithome.com.tw/articles/10193502)
* [yaml](https://ithelp.ithome.com.tw/articles/10193509)

<br/>

#### 超出資源限制

k8s 可以讓管理者限制配置到 Pod 或 Container 的資源 (CPU, Memory)，而使用者 (或開發者) 不會特別知道這些限制。一般來說都會等到發現部署失敗後才會察覺到有資源的限制。

底下為 error-limit-resource.yaml 內容

```
# error-limit-resource.yaml

---
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
    name: err-limit-nginx
spec:
    replicas: 1
    template:
      metadata:
        labels: 
          app: nginx
      spec:
        containers:
        - name: nginx
          image: nginx
          ports:
          - containerPort: 80
          resources:
            requests:
              memory: 100Gi    <=== 請分配 100G 的記憶體給我！
```

這裡指定需要分配 100G 的 memory，透過 kubectl 部署，應該能理解正常來說都不會成功吧！

> 一般人的機器應該不會超過 100G 的記憶體吧，有神機請自行調高後再測試

```bash
$ kubectl apply -f error-limit-resource.yaml
deployment "err-limit-nginx" configured
```

接著觀察 Pod

```bash
$ kubectl get pods
NAME                              READY     STATUS    RESTARTS   AGE
err-limit-nginx-b4548b69d-ll665   0/1       Pending   0          1m
```

這時候發現 Pod 並沒有運作，接著再觀察 `pods/err-limit-nginx-b4548b69d-ll665` 內的 `Event` 

> 記得將 err-limit-nginx-b4548b69d-ll665 改成正確的名稱

```bash
$ kubectl describe pods/err-limit-nginx-b4548b69d-ll665
...
Events:
  Type     Reason            Age               From               Message
  ----     ------            ----              ----               -------
  Warning  FailedScheduling  46s (x8 over 1m)  default-scheduler  No nodes are available that match all of the predicates: Insufficient memory (1).
...
```

這裡就會顯示 `Insufficient memory`，很清楚的說明 Pod 無法成功建立的原因是因為記憶體不足。

> 也有可能是因為 CPU 不足，都會顯示在 Event 內

如果想查看預設的資源限制可以使用

```bash
$ kubectl get limitrange [namespace]
```

如果是查詢 default 內的資源限制則 `[namespace]` 可不填

```bash
$ kubectl get limitrange
No resources found.
```

> minikube 預設沒有任何資源限制，但若是使用雲端方案 (GCP, AWS, ...) 等等，就會發現預設是有資源限制的。

*建議方案*

針對超出資源限制的問題，建議以下方式處理

1. 請系統管理者修改資源限制
2. 降低部署物件所需要的資源
3. 擴充硬體設備 (一般開發者較無法達成)

<br/>

#### 資源配額

與資源限制很類似，k8s 允許管理者指定資源配額 (Resource Quotas)。例如 Pod 在某個命名空間的最大運行數、最多能有幾個 Secret 物件或是 Memory, CPU 最多能使用多少等等。

> 更多說明請參考 [官方網站](https://kubernetes.io/docs/concepts/policy/resource-quotas/) 

也正是因為如此，在部署物件到 k8s 的時候，可能就會面臨資源配額不足的問題。接著我們來實際模擬一下，首先建立 quotas.yaml 模擬資源配額如下

```
# quotas.yaml

---
apiVersion: v1
kind: ResourceQuota
metadata:
  name: resource-quota
spec:
  hard:
    pods: 1   <=== 只准運行一個 Pod
```

這裡我們指定只能運行一個 Pod，並部署到 k8s 預設命名空間 default 內

```bash
$ kubectl apply -f quotas.yaml
resourcequota "resource-quota" created
```

> 建議測試前先將 default 內運行的 Deployment 與 Pod 刪除

接著編輯 quotas-deploy.yaml 

```
# quotas-deploy.yaml

---
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
    name: quotas-nginx
spec:
    replicas: 2  <=== 指定運行兩個 Pod，很明顯違反最多只能運行一個 Pod 的規範
    template:
      metadata:
        labels: 
          app: nginx
      spec:
        containers:
        - name: nginx
          image: nginx
          ports:
          - containerPort: 80
```

並部署到 k8s 

```bash
$ kubectl apply -f quotas-deploy.yaml
deployment "quotas-nginx" created
```

接著查看 Deployment 與 Pod 狀態

```bash
$ kubectl get deploy
NAME           DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
quotas-nginx   2         1         1            1           1m

$ kubectl get po
NAME                           READY     STATUS    RESTARTS   AGE
quotas-nginx-75f4785b7-98762   1/1       Running   0          1m
```

雖然我們指定運行兩個 Pod (DESIRED: 2)，但是只有一個 Pod 正確運行。於是再查看 Deployment 內容

```bash
$ kubectl describe deployment/quotas-nginx
...
NewReplicaSet:     quotas-nginx-75f4785b7 (1/2 replicas created)  <=== 只建立 1 個
Events:
  Type    Reason             Age   From                   Message
  ----    ------             ----  ----                   -------
  Normal  ScalingReplicaSet  3m    deployment-controller  Scaled up replica set quotas-nginx-75f4785b7 to 2
...
```

這裡發現 `(1/2 replicas created)`，的確只有一個 Pod 被建立，繼續追蹤 `NewReplicaSet: quotas-nginx-75f4785b7`

```bash
$ kubectl describe replicaset/quotas-nginx-75f4785b7   <=== 記得換名字喔
Events:
  Type     Reason            Age               From                   Message
  ----     ------            ----              ----                   -------
  Normal   SuccessfulCreate  6m                replicaset-controller  Created pod: quotas-nginx-75f4785b7-98762
  Warning  FailedCreate      1m (x17 over 6m)  replicaset-controller  Error creating: pods "quotas-nginx-75f4785b7-" is forbidden: exceeded quota: resource-quota, requested: pods=1, used: pods=1, limited: pods=1
```

終於在 replicaset 中的 Event 發現 `exceeded quota`。就像我們預設的情況一樣，我們給環境射了一個最多只能運行一個 Pod 的限制，所以當你想要運行兩個 Pod 的時候就會違反了規則。

*建議方案*

遇到這種情況，可以考慮使用下面方式處理

1. 請系統管理者修改資源配額
2. 刪除已部署且不必要的資源

本文與部署檔案同步發表於 [https://jlptf.github.io/ironman2018-day21/](https://jlptf.github.io/ironman2018-day21/)