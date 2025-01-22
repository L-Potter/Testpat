
Istio `Gateway` 資源的 YAML 文件示範，使用 `apiVersion: networking.istio.io/v1beta1`，並包含 `metadata`、`spec`、`selector` 以及設定 TLS 的 `servers` 區塊：  
https://istio.io/latest/docs/reference/config/networking/gateway/
https://istio.io/latest/zh/docs/tasks/traffic-management/ingress/ingress-control/

```yaml
apiVersion: networking.istio.io/v1beta1
kind: Gateway
metadata:
  name: my-ingress-gateway
  namespace: istio-system  # 指定命名空間，根據實際情況修改
spec:
  selector:
    istio: ingressgateway  # 這裡應該與 Istio ingress-gateway 部署的 label 匹配
  servers:
  - port:
      number: 443
      name: https
      protocol: HTTPS
    hosts:
    - "example.com"  # 替換為你的網域名稱
    tls:
      mode: SIMPLE  # 使用單向 TLS
      credentialName: ingress-cert  # 必須與 Kubernetes Secret 名稱對應
  - port:
      number: 80
      name: http
      protocol: HTTP
    hosts:
    - "example.com"
```

### 參數說明：
1. **`apiVersion: networking.istio.io/v1beta1`**  
   - 使用 Istio 的 API 版本，確保 Istio 安裝支持該版本。

2. **`kind: Gateway`**  
   - 定義 Istio Gateway 資源。

3. **`metadata.name`**  
   - 指定 Gateway 的名稱，可根據需求更改。

4. **`spec.selector`**  
   - `istio: ingressgateway` 需與 Istio Ingress Gateway Deployment 的 label 匹配，預設通常為 `istio-ingressgateway`。

5. **`servers`**  
   - `port` 定義服務監聽的端口（443 為 HTTPS，80 為 HTTP）。  
   - `hosts` 定義允許的主機（例如 `example.com` 或 `*.example.com`）。  
   - `tls` 設定 TLS 相關選項：
     - `mode: SIMPLE` 指定單向 TLS（僅加密）。
     - `credentialName` 需對應 Kubernetes Secret 內存儲的憑證。

### 建立 Kubernetes Secret：
如果使用 `credentialName: ingress-cert`，需要先手動建立對應的 Secret，例如：

```bash
kubectl create secret tls ingress-cert --cert=cert.pem --key=key.pem -n istio-system
```

### 部署方式：
儲存 YAML 文件為 `gateway.yaml`，並執行：

```bash
kubectl apply -f gateway.yaml
```

這樣就能在 Kubernetes 中建立一個 Istio Gateway，來處理 HTTPS 和 HTTP 流量。


---

在 Istio 中，`Gateway` 只是用來定義流量的入口點，還需要搭配 **`VirtualService`** 才能將進入的流量正確地路由到後端服務。  

以下是一個完整的 `VirtualService` YAML 文件示範，涵蓋您提到的 **`metadata`**, **`spec`**, **`gateways`**, **`hosts`**, **`http`**, **`route`**, **`destination`**, **`host`**, 和 **`port`**。  

### `VirtualService` YAML 範例：
```yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: my-virtualservice  # VirtualService 名稱
  namespace: istio-system  # 需與 Gateway 一致
spec:
  gateways:
  - my-ingress-gateway  # 需與 Gateway 定義的名稱一致
  hosts:
  - example.com  # 與 Gateway 的 hosts 配置保持一致
  http:
  - match:
    - uri:
        prefix: /  # 將所有請求轉發
    route:
    - destination:
        host: my-service.default.svc.cluster.local  # 內部服務名稱
        port:
          number: 80  # 內部服務的端口
```

---

### 參數說明：
1. **`metadata.name`**  
   - `my-virtualservice` 是這個 VirtualService 的名稱，可以根據需求修改。  

2. **`spec.gateways`**  
   - 這裡必須指定前面定義的 Gateway 名稱，例如 `my-ingress-gateway`。  
   - 如果要允許內部流量，也可以使用 `mesh`（如 `- mesh`）。

3. **`spec.hosts`**  
   - 這裡需與 Gateway 設定的 `hosts` 相匹配，例如 `example.com`。  
   - 可使用萬用字元，例如 `*.example.com`。

4. **`http.match`**  
   - 定義匹配條件，`prefix: /` 表示匹配所有請求。  
   - 可進一步細化，如 `/api/v1` 來路由特定 API 請求。

5. **`http.route.destination.host`**  
   - 內部的 Kubernetes 服務名稱，例如 `my-service.default.svc.cluster.local`。  
   - 應按照 `<service-name>.<namespace>.svc.cluster.local` 格式。

6. **`http.route.destination.port.number`**  
   - 內部服務的端口，例如 `80`，根據實際服務的暴露端口進行設置。

---

### 部署步驟：
1. 先確保已經建立對應的 Gateway（如 `my-ingress-gateway`）。
2. 儲存 YAML 文件為 `virtualservice.yaml`，然後執行：

   ```bash
   kubectl apply -f virtualservice.yaml
   ```

3. 驗證 VirtualService 部署是否成功：

   ```bash
   kubectl get virtualservice -n istio-system
   ```

---

這樣配置之後，外部的請求將會通過 Istio Gateway，並由 VirtualService 根據路由規則將流量轉發到 Kubernetes 內部的 `my-service` 服務。

# namespace DNS
https://kubernetes.io/zh-cn/docs/concepts/overview/working-with-objects/namespaces/#namespaces-and-dns


* https://kubernetes.io/zh-cn/docs/concepts/services-networking/dns-pod-service/
* https://godleon.github.io/blog/Kubernetes/k8s-DNS-for-Service-and-Pod/

### 1. **`destination.host` 的來源**  
在 Istio 的 `VirtualService` 中，`destination.host` 指定的是**Kubernetes 內部服務的名稱**，通常的格式為：  

```
<服務名稱>.<命名空間>.svc.cluster.local
```

例如，如果 Kubernetes 中有一個服務名稱為 `my-service`，位於 `default` 命名空間，則完整的 `destination.host` 為：  

```yaml
host: my-service.default.svc.cluster.local
```

這個名稱是 Kubernetes 自動分配的**內部 DNS 名稱**，確保 Istio 正確地將流量導向相應的 Pod。  

如果 VirtualService 只是在相同命名空間內，你可以簡寫為服務名稱（例如 `my-service`），但在跨命名空間時，建議使用完整格式來避免名稱衝突。  

---

### 2. **Gateway 使用萬用字元，而 VirtualService 需要指定完整主機名稱**  

#### **(1) Gateway 使用萬用字元的原因：**  
`Gateway` 主要用來管理**進入流量**，允許接收來自多個網域的請求，因此可以使用萬用字元 (`*.example.com`) 來接收**任意子網域的請求**，例如：  

```yaml
hosts:
- "*.example.com"
```

這樣，Gateway 可以接受所有符合 `example.com` 網域的請求，如：  

- `api.example.com`
- `shop.example.com`
- `user.example.com`

這樣設計的好處是**簡化配置**，允許多個子網域共享相同的 Gateway，減少重複配置。  

---

#### **(2) VirtualService 指定完整主機名稱的原因：**  
`VirtualService` 的作用是在流量到達 Kubernetes 內部後，根據特定的業務需求**精確地分流**，因此需要對請求進行更細緻的控制，通常指定完整的域名，例如：  

```yaml
hosts:
- "api.example.com"
```

這樣做的原因包括：  

- **精準控制：** 明確指向具體的應用（如 `api.example.com` → 轉發到 API 服務）。  
- **避免衝突：** 多個服務可能使用相同的 Gateway，但處理不同的子網域。  
- **流量細分：** 允許不同的 VirtualService 處理不同的子網域，提供更靈活的流量管理策略。  

例如，如果你的應用有以下情境：  

| 網域              | VirtualService 目標服務           |  
|------------------|-----------------------------|  
| api.example.com  | `api-service.default.svc.cluster.local` |  
| shop.example.com | `shop-service.default.svc.cluster.local` |  

你需要在 VirtualService 中分別設定不同的 `hosts`，而不是使用萬用字元。  

---

### 3. **綜合示範配置**  

#### ① `Gateway` 設定（允許萬用字元）：  
```yaml
apiVersion: networking.istio.io/v1beta1
kind: Gateway
metadata:
  name: my-ingress-gateway
  namespace: istio-system
spec:
  selector:
    istio: ingressgateway
  servers:
  - port:
      number: 443
      name: https
      protocol: HTTPS
    hosts:
    - "*.example.com"
    tls:
      mode: SIMPLE
      credentialName: ingress-cert
```

---

#### ② `VirtualService` 設定（具體路由）：  
```yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: api-virtualservice
  namespace: istio-system
spec:
  gateways:
  - my-ingress-gateway
  hosts:
  - "api.example.com"
  http:
  - match:
    - uri:
        prefix: /api
    route:
    - destination:
        host: api-service.default.svc.cluster.local
        port:
          number: 80
```

```yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: shop-virtualservice
  namespace: istio-system
spec:
  gateways:
  - my-ingress-gateway
  hosts:
  - "shop.example.com"
  http:
  - match:
    - uri:
        prefix: /shop
    route:
    - destination:
        host: shop-service.default.svc.cluster.local
        port:
          number: 80
```

---

### **總結：**
1. **Gateway 使用萬用字元**：  
   - 目的是允許所有子網域的流量進入，例如 `*.example.com`。  
   - 有助於減少配置重複，提供更靈活的入口流量管理。  

2. **VirtualService 指定完整域名**：  
   - 目的是精準控制流量，確保不同的子域名請求進入對應的服務。  
   - 避免流量誤導或衝突，並允許針對特定網域配置不同路由規則。  

如果你希望所有子網域都指向同一個後端服務，可以在 VirtualService 中也使用萬用字元，例如：  

```yaml
hosts:
- "*.example.com"
```

但這種情況下，所有子網域的請求都會導向同一個服務，適用於統一入口的應用情境（如微前端架構）。

---

在 Kubernetes + Istio 環境中，DNS 設定涉及內部（`*.cluster.local`）和外部（`example.com`）兩個不同的層面。讓我們來拆解如何完成 Gateway 用到的 `hosts` 以及 `cluster.local` 的名稱解析。  

---  

## **1. 設定 Kubernetes 內部 DNS (`.cluster.local`)**  
Kubernetes 內部服務會自動透過 **CoreDNS** 提供名稱解析，無需手動設定。當你在 Istio `VirtualService` 中指定目標主機（`destination.host`），這通常指的是 Kubernetes 內部服務名稱，例如：  

```yaml
destination:
  host: my-service.default.svc.cluster.local
  port:
    number: 80
```

**解析方式：**  
Kubernetes 內部的 DNS 會自動處理以下規則：  

- **完整格式（FQDN）：** `my-service.default.svc.cluster.local`  
- **簡化格式（同一命名空間）：** `my-service`  
- **跨命名空間時：** 需要使用 `<service-name>.<namespace>.svc.cluster.local`  

**示例：**  
如果你的 Kubernetes 內部服務名稱是 `my-service`，位於 `default` 命名空間，則你可以：  

1. 直接在 VirtualService 中使用完整名稱（推薦）：  
   ```yaml
   destination:
     host: my-service.default.svc.cluster.local
     port:
       number: 80
   ```
2. 如果 VirtualService 與目標服務位於相同命名空間，則可以簡寫：  
   ```yaml
   destination:
     host: my-service
     port:
       number: 80
   ```

**驗證 DNS 是否正常運作：**  
你可以使用 `busybox` 容器測試內部 DNS 解析：  

```bash
kubectl run busybox --image=busybox --restart=Never --rm -it -- nslookup my-service.default.svc.cluster.local
```

若解析成功，表示 Kubernetes 內部 DNS 設定正確。  

---  

## **2. 設定外部 DNS（Gateway 使用的 `hosts`）**  
在 Istio Gateway 配置的 `hosts`（如 `example.com`）需要依賴**外部 DNS 解析**，通常透過 DNS 設定來解析到 Kubernetes 集群的 Ingress IP 或 LoadBalancer IP。  

### **步驟 1：確認 Istio Ingress Gateway 的 IP**  
執行以下命令來取得 Istio Ingress Gateway 的外部 IP：  

```bash
kubectl get service istio-ingressgateway -n istio-system
```

輸出範例（取決於你的 Kubernetes 部署類型）：  

- **LoadBalancer 模式：**  
  ```plaintext
  NAME                   TYPE           CLUSTER-IP      EXTERNAL-IP      PORT(S)
  istio-ingressgateway   LoadBalancer   10.100.200.10    203.0.113.50     80:30380/TCP,443:30443/TCP
  ```
  在這種情況下，外部 IP 為 `203.0.113.50`。

- **NodePort 模式：**  
  你可能需要手動配置 Nginx 或其他反向代理來提供外部訪問。

---

### **步驟 2：設定 DNS 解析到 Ingress Gateway IP**  
使用你的 DNS 託管服務（如 Cloudflare、Route 53 或自建 DNS 伺服器），將你的域名 `example.com` 指向 Ingress Gateway 的外部 IP。  

- **如果你有權限管理 DNS，添加 A 記錄：**  

  ```
  example.com -> 203.0.113.50
  ```

- **如果沒有權限，可以手動在本地電腦的 `/etc/hosts` 添加暫時解析：**  

  ```plaintext
  203.0.113.50 example.com
  ```

確認設定後，測試域名是否解析成功：  

```bash
nslookup example.com
```

---

### **步驟 3：確認 TLS 憑證配置**  
如果你使用 HTTPS，則需要在 Kubernetes 中建立對應的 TLS 憑證，以便在 Gateway 中使用：  

```bash
kubectl create secret tls ingress-cert \
  --cert=example.com.crt \
  --key=example.com.key \
  -n istio-system
```

---

### **步驟 4：測試外部流量是否成功進入 Kubernetes**  
如果 DNS 已指向正確的 IP，並且 Gateway 和 VirtualService 正確配置，你可以使用 `curl` 測試：  

```bash
curl -H "Host: example.com" https://203.0.113.50 --insecure
```

---

## **3. 綜合示範配置**  
### **(1) Gateway 配置（允許外部流量）**
```yaml
apiVersion: networking.istio.io/v1beta1
kind: Gateway
metadata:
  name: my-ingress-gateway
  namespace: istio-system
spec:
  selector:
    istio: ingressgateway  # 與 istio-ingressgateway service label 匹配
  servers:
  - port:
      number: 443
      name: https
      protocol: HTTPS
    hosts:
    - "*.example.com"
    tls:
      mode: SIMPLE
      credentialName: ingress-cert  # 之前建立的 Secret 名稱
  - port:
      number: 80
      name: http
      protocol: HTTP
    hosts:
    - "*"
```

---

### **(2) VirtualService 配置（路由到內部服務）**
```yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: my-virtualservice
  namespace: istio-system
spec:
  gateways:
  - my-ingress-gateway
  hosts:
  - "example.com"
  http:
  - match:
    - uri:
        prefix: /
    route:
    - destination:
        host: my-service.default.svc.cluster.local
        port:
          number: 80
```

---

### **(3) Kubernetes 內部服務示範**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service
  namespace: default
spec:
  selector:
    app: my-app
  ports:
    - protocol: TCP
      port: 80
      targetPort: 8080
```

---

## **總結**
1. **內部 DNS 設定：**  
   - Kubernetes 內部自動管理 `.cluster.local` 名稱解析，無需手動設定。  
   - 使用 FQDN `my-service.default.svc.cluster.local` 確保跨命名空間解析正確。  

2. **外部 DNS 設定：**  
   - Gateway 配置 `hosts` 為域名，如 `example.com`。  
   - 透過 DNS 託管平台將該域名指向 Ingress Gateway 的外部 IP。  
   - 若無 DNS 託管權限，可手動添加 `/etc/hosts` 進行測試。  

3. **測試流量是否正常進入：**  
   - 使用 `curl` 測試 `Host` 標頭來模擬請求。  
   - 確保 VirtualService 的 `hosts` 與 Gateway 設置一致，確保流量正確轉發。  

這樣就能完成完整的 Istio Gateway 和 VirtualService 流程，並確保內外部 DNS 設定正確運作。

https://github.com/superj80820/2020-ithelp-contest/tree/master/DAY23

沒錯，這段描述的流程是在使用 `minikube tunnel` 來模擬 Kubernetes LoadBalancer 的情境下，解決 DNS 解析問題的方式。讓我們來拆解你的問題並詳細說明：

---

## **1. `minikube tunnel` 的作用**  

當你在本機使用 Minikube 部署 Istio 或其他應用時，Kubernetes 無法直接提供真正的 LoadBalancer 類型服務（因為它通常依賴雲端供應商，如 AWS、GCP）。  

Minikube 提供了一個 `minikube tunnel` 命令，**將本機端口轉發到 Kubernetes Service，並模擬 LoadBalancer 的外部 IP**。  

### **執行流程：**  

```bash
minikube tunnel
```

- 該命令會：  
  1. **分配一個模擬的外部 IP**（通常是 `127.0.0.1` 或本地網卡的 IP）；  
  2. 將該 IP 綁定到 LoadBalancer 類型的 Service；  
  3. 需要在 `/etc/hosts` 或其他 DNS 解決方案中手動配置對應的解析。  

你可以使用以下指令來取得 Minikube 指定的外部 IP：  

```bash
minikube ip
```

**示例輸出：**  

```
192.168.49.2  (如果你用的是 VirtualBox 驅動)
127.0.0.1     (如果你用的是 Docker 驅動)
```

---

## **2. `/etc/hosts` 配置**  

如果 `minikube ip` 回傳的是 `127.0.0.1`，則你需要手動將你的應用域名對應到該 IP，以便本地測試。例如，如果你的 Gateway 配置 `hosts: ["python-www"]`，你需要手動將該域名指向 Minikube：  

```bash
sudo nano /etc/hosts
```

並添加：  

```
127.0.0.1 python-www
```

這樣當你在瀏覽器或 `curl` 中訪問 `http://python-www` 時，系統會將請求指向本地的 Minikube。  

---

## **3. 為什麼 Kubernetes 內部能解析 `python-www`？**  

在 Kubernetes 內部環境中，Pod 透過 CoreDNS 解析內部服務名稱時，會自動附加 Kubernetes 的預設搜尋域（`search domain`），例如：  

```plaintext
python-www.default.svc.cluster.local.
```

這裡的搜尋順序：  

1. 如果你在 Pod 內部執行：  

   ```bash
   nslookup python-www
   ```

   解析器會依序嘗試：  
   
   ```
   python-www.default.svc.cluster.local.  # 首選（因為 namespace 預設）
   python-www.svc.cluster.local.
   python-www.cluster.local.
   ```

2. 當你在 Pod 內執行 `curl python-www`，如果該服務部署在 `default` 命名空間，Kubernetes 會自動補全 `default.svc.cluster.local`，這就是為什麼 Pod 內部無需手動指定完整的 FQDN（Fully Qualified Domain Name）。  

3. 如果 Pod 與 Service 不在同一個命名空間，你需要使用完整域名，例如：  

   ```
   python-www.my-namespace.svc.cluster.local
   ```

---

## **4. 與 NGINX Ingress Controller 的對比**  

你提到「類似使用 NGINX 來找尋 DNS 的請求」，這部分是指在外部環境中，**NGINX Ingress Controller** 充當反向代理，接收來自客戶端的請求，然後將其轉發至 Kubernetes 內部的服務。  

如果你使用 Istio Ingress Gateway，它的工作方式與 NGINX Ingress Controller 類似，但功能更強大（如流量管理、熔斷等）。  

當 NGINX 或 Istio Gateway 接收到像 `python-www` 這樣的請求時，它們需要依靠正確的 DNS 配置來決定如何處理請求：  

- **在 Kubernetes 內部：** 會補全 `.default.svc.cluster.local`。  
- **在 Kubernetes 之外：** 需要手動設定 `/etc/hosts` 或使用真實的 DNS 來解析。  

如果你的 DNS 設定錯誤，可能會發生：  

- 請求無法找到服務，導致 503 錯誤。  
- Gateway 收到請求但無法解析內部服務名稱。  

---

## **5. 檢查 DNS 設定是否正確的步驟**  

在 Minikube 上進行以下測試，確保你的 DNS 設定正確：  

1. **檢查服務的 Cluster IP：**  

   ```bash
   kubectl get svc -n default
   ```

   確保你的 `python-www` 服務存在並且 Cluster IP 正確。  

2. **使用 Pod 內部測試 DNS 解析：**  

   ```bash
   kubectl run -it --rm busybox --image=busybox -- nslookup python-www
   ```

   預期輸出：  

   ```
   Server:    10.96.0.10
   Address:   10.96.0.10#53
   
   Name:      python-www.default.svc.cluster.local
   Address:   10.104.117.30
   ```

3. **從本機測試是否能解析：**  

   ```bash
   curl -H "Host: python-www" http://127.0.0.1
   ```

   如果正確，應該返回服務的回應。  

---

## **6. 小結**  

1. **Minikube 模擬 LoadBalancer**  
   - 使用 `minikube tunnel` 來提供外部 IP。  
   - 使用 `minikube ip` 取得模擬 IP，並在 `/etc/hosts` 手動設定域名對應。  

2. **Kubernetes 內部 DNS**  
   - CoreDNS 自動補全 `.default.svc.cluster.local`，Pod 內部可直接使用短名稱訪問。  
   - 外部系統（如瀏覽器）需要手動配置 DNS 解析（`/etc/hosts` 或企業 DNS 服務）。  

3. **NGINX Ingress vs Istio Gateway**  
   - NGINX 依賴外部 DNS 設定，作為傳統反向代理。  
   - Istio Gateway 提供更多服務發現和流量管理功能。  

4. **測試並驗證設定**  
   - 使用 `nslookup` 檢查 DNS 解析。  
   - 使用 `curl` 測試實際應用程式連線。  

---

如果你有進一步的疑問，歡迎隨時詢問！

---

https://istio.io/latest/docs/ops/configuration/traffic-management/protocol-selection/


以下是一個完整的 **Kubernetes YAML 配置範例**，包含：  

1. **Web API Service**（Kubernetes Service 定義）  
2. **VirtualService**（Istio 流量管理）  
3. **Sidecar**（Istio 流量範圍控制）  

---  

### **1. 部署 Web API 的 Deployment 與 Service**  

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-api
  labels:
    app: web-api
spec:
  replicas: 2
  selector:
    matchLabels:
      app: web-api
  template:
    metadata:
      labels:
        app: web-api
    spec:
      containers:
      - name: web-api
        image: my-web-api:latest
        ports:
        - containerPort: 8080
        resources:
          limits:
            cpu: "500m"
            memory: "512Mi"
        env:
        - name: ENV
          value: "production"
---
apiVersion: v1
kind: Service
metadata:
  name: web-api
spec:
  selector:
    app: web-api
  ports:
  - name: http
    protocol: TCP
    port: 80
    targetPort: 8080
  type: ClusterIP
```

---

### **2. Istio VirtualService 設定流量路由**  

這個 VirtualService 會將來自 `example.com` 的流量路由到 `web-api` 服務的 80 端口。  

```yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: web-api-vs
spec:
  hosts:
  - "example.com"
  gateways:
  - web-api-gateway
  http:
  - match:
    - uri:
        prefix: /
    route:
    - destination:
        host: web-api.default.svc.cluster.local
        port:
          number: 80
    retries:
      attempts: 3
      perTryTimeout: 2s
      retryOn: 5xx
```

---

### **3. Istio Gateway 定義**  

這個 Gateway 允許來自外部的 HTTP 流量進入集群，並匹配 `example.com`。  

```yaml
apiVersion: networking.istio.io/v1beta1
kind: Gateway
metadata:
  name: web-api-gateway
spec:
  selector:
    istio: ingressgateway  # 使用預設 Istio Ingress Gateway
  servers:
  - port:
      number: 80
      name: http
      protocol: HTTP
    hosts:
    - "example.com"
```

---

### **4. Sidecar 配置（流量控制）**  

此 Sidecar 配置限制 Web API Pod 只能與 `web-api` 和 `istio-system` 通信，防止不必要的流量進出。  

```yaml
apiVersion: networking.istio.io/v1beta1
kind: Sidecar
metadata:
  name: web-api-sidecar
spec:
  workloadSelector:
    labels:
      app: web-api
  egress:
  - hosts:
    - "default/web-api.default.svc.cluster.local"  # 只允許訪問 web-api 服務
    - "istio-system/*"  # 允許與 Istio 控制平面通訊
```

---

### **5. 設定 Istio 自動注入 Sidecar**  

確保 Web API 部署在 `default` 命名空間中已啟用 Sidecar 自動注入：  

```bash
kubectl label namespace default istio-injection=enabled
```

如果不想使用自動注入，可以手動在 Pod 定義中加入注解來啟用 Sidecar：  

```yaml
apiVersion: v1
kind: Pod
metadata:
  annotations:
    sidecar.istio.io/inject: "true"
```

---

### **6. 更新 /etc/hosts 來模擬 DNS**（適用於 Minikube）  

若使用 Minikube，需將 `example.com` 綁定到 Minikube 的外部 IP，例如：  

```bash
minikube ip   # 獲取 Minikube IP，例如 192.168.49.2
```

然後在 `/etc/hosts` 文件中添加：  

```plaintext
192.168.49.2 example.com
```

這樣當你訪問 `http://example.com` 時，請求會被正確引導至 Istio Gateway。  

---

### **7. 驗證配置**  

部署完成後，可以執行以下命令來驗證：  

```bash
# 確認所有 Pod 運行正常
kubectl get pods -n default

# 檢查 Istio Gateway 是否啟動
kubectl get gateway -n default

# 確認 VirtualService 是否正確配置
kubectl get virtualservice -n default
```

測試服務是否正常運行：  

```bash
curl -H "Host: example.com" http://$(minikube ip)
```

如果一切正常，你應該會看到 Web API 返回的內容。  

---

### **總結**  

1. **Deployment & Service**：定義 Web API 應用，並暴露內部服務。  
2. **Istio Gateway**：允許外部訪問 `example.com`，並將流量轉發給 VirtualService。  
3. **VirtualService**：配置 HTTP 流量路由至 `web-api`，並處理重試等流量策略。  
4. **Sidecar**：限制 Web API Pod 只能與指定服務通訊，增強安全性與性能。  
5. **Minikube 設定**：手動將域名綁定到 Minikube IP，以模擬外部訪問行為。  

---

這個範例可作為一個簡單的 Istio 網格部署模板，能幫助你熟悉 Istio 的流量管理和網路治理。  

有任何問題，歡迎討論！
