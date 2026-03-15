# 在 Kubernetes 使用 InitContainer 部署（Helm）

本文說明如何透過 Helm 部署 Traefik 時，使用 InitContainer 從 private Git repository 載入 `cloudflarewarp` 作為 local plugin。

## 運作原理

```
┌─────────────────────────────────────────────┐
│  Traefik Pod                                │
│                                             │
│  [InitContainer: git-clone]                 │
│    ↓ clone repo → /plugins-local/src/...   │
│                                             │
│  [Container: traefik]                       │
│    ← 從 /plugins-local 讀取 plugin 原始碼  │
└─────────────────────────────────────────────┘
         ↑ 共用 emptyDir Volume
```

InitContainer 在 Traefik 啟動前執行，將 plugin 原始碼 clone 到共享 Volume；Traefik 啟動後以 `localPlugins` 模式載入。

## 前置條件

- Traefik Helm chart（官方 `traefik/traefik`）
- kubectl 可存取目標 cluster
- 若為 private repository，需有 Git 存取憑證

---

## 步驟一：建立 Git 存取 Secret（private repo 才需要）

使用 GitHub Personal Access Token（需有 `repo` 讀取權限）：

```bash
kubectl create secret generic cloudflarewarp-git-secret \
  --from-literal=token=<your-github-token> \
  -n traefik
```

> 若為 public repository，可跳過此步驟，並在後續的 `values.yaml` 中移除 Secret 相關設定。

---

## 步驟二：準備 Helm values

建立 `values.yaml`，加入以下設定：

```yaml
# values.yaml

# 1. 將 experimental.localPlugins 加入 Traefik 靜態設定
additionalArguments:
  - "--experimental.localPlugins.cloudflarewarp.moduleName=github.com/enoqv/cloudflarewarp"

# 2. 宣告共享 Volume（emptyDir）
deployment:
  additionalVolumes:
    - name: plugins-local
      emptyDir: {}

# 3. 將 Volume 掛載到 Traefik 主容器
additionalVolumeMounts:
  - name: plugins-local
    mountPath: /plugins-local

# 4. 設定 InitContainer 執行 git clone
initContainers:
  - name: git-clone-plugin
    image: alpine/git:latest
    command:
      - sh
      - -c
      - |
        git clone --depth 1 \
          https://oauth2:$(GIT_TOKEN)@github.com/enoqv/cloudflarewarp.git \
          /plugins-local/src/github.com/enoqv/cloudflarewarp
    env:
      - name: GIT_TOKEN
        valueFrom:
          secretKeyRef:
            name: cloudflarewarp-git-secret
            key: token
    volumeMounts:
      - name: plugins-local
        mountPath: /plugins-local
```

> **Public repository** 的話，`command` 可簡化為：
> ```yaml
> command:
>   - sh
>   - -c
>   - |
>     git clone --depth 1 \
>       https://github.com/enoqv/cloudflarewarp.git \
>       /plugins-local/src/github.com/enoqv/cloudflarewarp
> ```
> 並移除 `env` 及 `secretKeyRef` 區塊。

### 指定版本（建議）

生產環境建議鎖定特定 tag 或 commit，避免 upstream 異動影響服務：

```yaml
# 指定 tag
git clone --depth 1 --branch v1.3.0 \
  https://... /plugins-local/src/github.com/enoqv/cloudflarewarp

# 或指定 commit
git clone https://... /tmp/cloudflarewarp && \
  cd /tmp/cloudflarewarp && \
  git checkout <commit-sha> && \
  cp -r . /plugins-local/src/github.com/enoqv/cloudflarewarp
```

---

## 步驟三：部署 Traefik

```bash
helm repo add traefik https://traefik.github.io/charts
helm repo update

helm upgrade --install traefik traefik/traefik \
  -n traefik \
  --create-namespace \
  -f values.yaml
```

---

## 步驟四：建立 Middleware

部署完成後，建立使用 plugin 的 Middleware 資源。

**middleware.yaml**

```yaml
apiVersion: traefik.io/v1alpha1
kind: Middleware
metadata:
  name: cloudflarewarp
  namespace: default
spec:
  plugin:
    cloudflarewarp:
      disableDefault: false
      trustip:
        - 10.0.0.0/8
        - 172.16.0.0/12
        - 192.168.0.0/16
```

```bash
kubectl apply -f middleware.yaml
```

### 設定說明

| 欄位 | 類型 | 說明 |
|---|---|---|
| `disableDefault` | bool | `true` 時停用內建的 Cloudflare IP 清單，僅信任 `trustip` 指定的範圍 |
| `trustip` | []string | 額外信任的 CIDR 清單（通常加入 cluster 內部網段） |

---

## 步驟五：在 IngressRoute 套用 Middleware

```yaml
apiVersion: traefik.io/v1alpha1
kind: IngressRoute
metadata:
  name: my-app
  namespace: default
spec:
  entryPoints:
    - web
    - websecure
  routes:
    - match: Host(`example.com`)
      kind: Rule
      services:
        - name: my-app
          port: 80
      middlewares:
        - name: cloudflarewarp
          namespace: default
```

---

## 驗證

確認 plugin 載入成功：

```bash
# 查看 Traefik Pod 是否正常啟動
kubectl get pods -n traefik

# 查看 InitContainer 執行結果
kubectl logs -n traefik <traefik-pod-name> -c git-clone-plugin

# 查看 Traefik 主容器 log，確認 plugin 已載入
kubectl logs -n traefik <traefik-pod-name> | grep -i "cloudflarewarp\|plugin\|localPlugin"
```

請求通過 Cloudflare 後，response headers 應出現：

| Header | 說明 |
|---|---|
| `X-Is-Trusted: yes` | 請求來自受信任的 Cloudflare IP |
| `X-Real-IP` | 真實的使用者 IP（來自 `CF-Connecting-IP`） |
| `X-Forwarded-Proto` | 原始協定（來自 `CF-Visitor`） |

直接連線（非 Cloudflare）時：

| Header | 說明 |
|---|---|
| `X-Is-Trusted: no` | 請求不來自 Cloudflare |
| `X-Real-IP` | 直接連線的 IP |

---

## 疑難排解

**InitContainer 停在 `Init:0/1`**

```bash
kubectl describe pod -n traefik <traefik-pod-name>
kubectl logs -n traefik <traefik-pod-name> -c git-clone-plugin
```

常見原因：Secret 不存在、Token 無效、網路無法連外。

---

**Traefik 啟動後出現 `plugin not found` 或 `unknown plugin`**

確認 clone 後的目錄結構是否正確：

```bash
kubectl exec -n traefik <traefik-pod-name> -- ls /plugins-local/src/github.com/enoqv/cloudflarewarp/
```

應包含：`cloudflarewarp.go`、`go.mod`、`.traefik.yml`、`ips/` 目錄。

---

**Traefik Pod 重啟後需重新 clone**

emptyDir 在 Pod 重啟後會清空，InitContainer 每次都會重新執行，這是預期行為。若需要持久化可改用 PVC，但通常不必要，因為 clone 速度很快（`--depth 1`）。

---

**需要更新 Cloudflare IP 清單**

在 repository 執行 `bash updateCFIps.sh` 並 commit，下次 Pod 重啟時 InitContainer 即會 clone 到最新版本。
