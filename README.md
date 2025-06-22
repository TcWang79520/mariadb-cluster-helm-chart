# MariaDB Helm Chart 部署範例

這是一套使用 **Helm Chart 部署 MariaDB 1 + N (主從複寫)** 的範例專案，適合自建 K8s 環境 (例如 **minikube**) 進行測試與學習。

---

## 💻 環境需求

- **安裝並啟用 Kubernetes 環境**
  - 建議使用 [minikube](https://minikube.sigs.k8s.io/docs/start/)
  - 範例啟動 minikube：
    ```bash
    minikube start --cpus=2 --memory=4096
    ```
- **安裝 helm**
  - 官方教學: [Helm 官方](https://helm.sh/docs/intro/install/)

---

## 🚀 部署方式

```bash
# 建立命名空間 (可選)
kubectl create namespace mariadb

# 部署
helm upgrade --install mydb ./my-mariadb --namespace mariadb

# 查看部署狀態
kubectl get pods -n mariadb
kubectl get pvc -n mariadb
kubectl get svc -n mariadb
```

---

## 📝 主要檔案

- `values.yaml` → 配置 MariaDB image、replica 數、密碼、儲存等
- `templates/statefulset-primary.yaml` → 主資料庫 StatefulSet
- `templates/statefulset-replica.yaml` → 從資料庫 StatefulSet
- `templates/configmap.yaml` → 初始化 SQL + my.cnf 設定
- `templates/service.yaml` → 主從對應的 Service

---

## 🔍 常見錯誤與解決辦法

### ❗ 無法建立 PVC / Pod 一直卡在 `ContainerCreating`

查詢：

```bash
kubectl describe statefulset mydb-my-mariadb-primary -n mariadb
kubectl get events -n mariadb --sort-by=.metadata.creationTimestamp | tail -20
kubectl describe pvc -n mariadb
```

解決：

- 確認 **StorageClass 存在**：`kubectl get storageclass`
- 確認 minikube 啟動正常：`minikube status`
- 如果 minikube 重啟後出現錯誤：
  ```bash
  minikube start
  minikube update-context
  ```

---

### ❗ Replica pod 連不上 primary，顯示 DNS NXDOMAIN 或 Unknown host

查詢：

```bash
kubectl exec -it <replica-pod> -- sh
# 或
kubectl run debug --rm -it --image=busybox:1.35 -- sh
nslookup mydb-my-mariadb-primary-0.mydb-my-mariadb-primary.mariadb.svc.cluster.local
```

解決：

- 確認 Service 名稱與 Headless Service 正確
- 確認 StatefulSet 的 `serviceName` 對應正確

---

### ❗ 主從 server\_id 相同，導致複寫失敗

症狀：

```
Fatal error: The slave I/O thread stops because master and slave have equal MariaDB server ids
```

查詢：

```sql
SHOW VARIABLES LIKE 'server_id';
```

解決：

- 確保 `values.yaml` 分別給主與從不同的 `serverId`
- `ConfigMap` 正確掛載至 `/etc/mysql/conf.d/my.cnf`

---

### ❗ Pod 啟動正常但資料不一致（從沒有同步表或資料）

查詢：

```sql
SHOW SLAVE STATUS\G
```

看：

- `Last_SQL_Error`
- `Slave_IO_Running` / `Slave_SQL_Running`

解決：

- 確認主資料庫有正確建立資料表
- 手動重啟 slave 複寫：
  ```sql
  STOP SLAVE;
  START SLAVE;
  ```

---

## 🧭 其他常用指令

```bash
# 查看 Helm 渲染結果
helm template mydb ./my-mariadb --namespace mariadb

# 刪除部署
helm uninstall mydb --namespace mariadb

# 清除 PVC 資源
kubectl delete pvc -n mariadb --all
```

---

## 💡 建議

✅ 這套 Chart 適合在 **學習 / 測試環境**，若要用於正式環境，請考慮：

- 使用 **mariadb-galera** 或專用的 Operator
- 搭配外部 HAProxy / Nginx 或 cloud LB 進行高可用切換
- 加入資安強化：密碼加密、TLS、資源限制等

---

## 🗂 專案結構範例

```
my-mariadb/
 ├── charts/
 ├── templates/
 │    ├── configmap.yaml
 │    ├── service.yaml
 │    ├── statefulset-primary.yaml
 │    ├── statefulset-replica.yaml
 ├── values.yaml
 ├── Chart.yaml
 └── .gitignore
```

