# Buổi 07 - Application Lifecycle, Health Check, Rolling Update, Rollback và Self-healing trong Kubernetes v1.35

## 1. Mục tiêu của buổi học

Ở các buổi trước,  đã biết Kubernetes cluster được hình thành như thế nào, workload được định nghĩa ra sao, Service giúp truy cập Pod như thế nào, storage được cấp phát ra sao và cách đưa ứng dụng ra bên ngoài bằng Ingress.

Buổi này chuyển trọng tâm từ “triển khai ứng dụng” sang “vận hành vòng đời ứng dụng”. Đây là phần rất quan trọng khi đi vào môi trường production, vì một ứng dụng không chỉ cần chạy được mà còn cần:

- Khởi động đúng cách.
- Chỉ nhận traffic khi thật sự sẵn sàng.
- Tự restart khi bị treo hoặc không còn khỏe.
- Update phiên bản mới mà hạn chế downtime.
- Rollback nhanh khi phiên bản mới lỗi.
- Tự phục hồi khi container hoặc Pod gặp sự cố.

Sau buổi này,  cần nắm được:

- Pod lifecycle ở mức vận hành.
- Container states và Pod phases.
- Ý nghĩa của `restartPolicy`.
- Sự khác nhau giữa `startupProbe`, `readinessProbe` và `livenessProbe`.
- Cách Service loại bỏ Pod chưa sẵn sàng ra khỏi danh sách endpoint.
- Cách Deployment thực hiện rolling update.
- Ý nghĩa của `maxSurge`, `maxUnavailable`, `minReadySeconds`, `progressDeadlineSeconds`, `revisionHistoryLimit`.
- Cách theo dõi rollout.
- Cách rollback Deployment.
- Cách quan sát cơ chế self-healing của Kubernetes.

---

## 2. Phạm vi kiến thức của buổi 07

### 2.1. Kiến thức đã được phép sử dụng từ các buổi trước

Buổi này được xây dựng dựa trên các kiến thức đã học:

- Cluster Kubernetes gồm control plane và worker node.
- Pod, ReplicaSet, Deployment.
- Label và selector.
- Namespace.
- Service loại `ClusterIP` và `NodePort`.
- Endpoint/EndpointSlice ở mức quan sát Service trỏ đến Pod.
- ConfigMap.
- Secret ở mức đã học, nhưng buổi này không tập trung vào Secret.
- Volume mount từ ConfigMap.
- Helm, MetalLB, Ingress NGINX Controller đã học ở buổi 06.

### 2.2. Kiến thức mới được mở khóa trong buổi này

Các khái niệm mới của buổi 07:

- Pod phase.
- Container state.
- `restartPolicy`.
- Startup probe.
- Readiness probe.
- Liveness probe.
- Container lifecycle hook `preStop`.
- Graceful termination.
- Deployment rollout.
- Deployment rollback.
- Revision history.
- Deployment progress deadline.
- Self-healing.

### 2.3. Những nội dung chưa đi sâu trong buổi này

Để giữ đúng tiến trình giáo trình, buổi này chưa đi sâu vào:

- Resource request và limit.
- QoS class.
- HPA/autoscaling.
- PodDisruptionBudget.
- RBAC.
- SecurityContext.
- NetworkPolicy.
- Observability stack như Prometheus, Grafana, Loki.
- Argo CD/GitOps.

Các nội dung trên sẽ phù hợp hơn ở các buổi sau.

---

## 3. Mô hình lab sử dụng trong buổi học

Buổi này tiếp tục sử dụng cluster đã triển khai từ các buổi trước.

### 3.1. Mô hình node

```text
+----------------------+       +----------------------+       +----------------------+
| master-01            |       | worker-01            |       | worker-02            |
| Ubuntu 24.04         |       | Ubuntu 24.04         |       | Ubuntu 24.04         |
| Control Plane        |       | Worker Node          |       | Worker Node          |
| kube-apiserver       |       | kubelet              |       | kubelet              |
| etcd                 |       | containerd           |       | containerd           |
| scheduler            |       | Calico               |       | Calico               |
| controller-manager   |       |                      |       |                      |
+----------------------+       +----------------------+       +----------------------+
```

Nếu buổi 05 có thêm NFS Server và buổi 06 có MetalLB/Ingress, các thành phần đó vẫn có thể tồn tại trong môi trường lab. Tuy nhiên, buổi 07 không bắt buộc dùng NFS. Ingress chỉ được dùng ở phần mở rộng.

### 3.2. Namespace lab

Toàn bộ tài nguyên của buổi học sẽ đặt trong namespace riêng:

```bash
kubectl create namespace k8s-b07
```

Kiểm tra:

```bash
kubectl get ns k8s-b07
```

Kết quả mong đợi:

```text
NAME      STATUS   AGE
k8s-b07   Active   ...
```

---

## 4. Tại sao cần học Application Lifecycle?

Trong môi trường Docker thông thường, khi container chạy được là ta thường xem như ứng dụng đã được triển khai xong. Nhưng trong Kubernetes, “chạy được” chưa đủ.

Một ứng dụng production cần trả lời được các câu hỏi sau:

- Ứng dụng đã khởi động xong chưa?
- Ứng dụng có đang sống không?
- Ứng dụng đã sẵn sàng nhận request chưa?
- Nếu container bị treo thì ai restart?
- Nếu Pod bị xóa thì ai tạo lại?
- Nếu node hỏng thì workload có được chạy lại ở node khác không?
- Nếu cập nhật image lỗi thì làm sao rollback?
- Nếu phiên bản mới chưa sẵn sàng thì Kubernetes có tiếp tục gửi traffic vào nó không?

Kubernetes giải quyết các câu hỏi này bằng nhiều cơ chế phối hợp với nhau:

- `kubelet` theo dõi container trên từng node.
- `Deployment Controller` duy trì số lượng Pod mong muốn.
- `ReplicaSet` giữ số replica thực tế.
- `Service` và `EndpointSlice` quyết định Pod nào được nhận traffic.
- `Probe` giúp kubelet biết container đã started, ready hoặc live chưa.
- `RollingUpdate` giúp thay thế Pod cũ bằng Pod mới có kiểm soát.
- `Rollback` giúp quay lại revision trước nếu rollout lỗi.

---

## 5. Pod lifecycle

### 5.1. Pod không phải là máy ảo

Một lỗi tư duy phổ biến của người mới học Kubernetes là xem Pod giống như một máy ảo nhỏ. Cách hiểu này không chính xác.

Pod là một đơn vị chạy workload có vòng đời ngắn hơn VM rất nhiều. Pod có thể bị xóa và tạo lại bất kỳ lúc nào khi:

- Node gặp sự cố.
- Deployment cần rollout phiên bản mới.
- Liveness probe thất bại.
- User xóa Pod thủ công.
- ReplicaSet cần cân bằng lại số lượng Pod.
- Pod bị evict do tài nguyên node không đủ.

Vì vậy, khi thiết kế ứng dụng trên Kubernetes, ta không nên phụ thuộc vào một Pod cụ thể. Thay vào đó, nên để controller như Deployment, StatefulSet, DaemonSet quản lý vòng đời của Pod.

### 5.2. Pod phase

Pod phase là trạng thái tổng quát của Pod trong vòng đời.

Các phase thường gặp:

| Pod phase | Ý nghĩa |
|---|---|
| `Pending` | Pod đã được API Server chấp nhận nhưng chưa chạy hoàn chỉnh. Có thể đang chờ scheduler chọn node, đang pull image hoặc đang tạo container. |
| `Running` | Pod đã được gán vào node, container đã được tạo và ít nhất một container đang chạy hoặc đang khởi động lại. |
| `Succeeded` | Tất cả container trong Pod đã kết thúc thành công và sẽ không restart. Thường gặp với Job. |
| `Failed` | Tất cả container đã kết thúc, ít nhất một container kết thúc lỗi và không được restart tiếp. |
| `Unknown` | Kubernetes không lấy được trạng thái Pod, thường do mất liên lạc với node. |

Lưu ý quan trọng:

`STATUS` hiển thị bởi `kubectl get pod` không hoàn toàn giống với `phase` trong Kubernetes API. Ví dụ `CrashLoopBackOff`, `ImagePullBackOff`, `Terminating` là các trạng thái hiển thị để người vận hành dễ hiểu, không phải lúc nào cũng là Pod phase chính thức.

Kiểm tra phase của một Pod:

```bash
kubectl get pod <pod-name> -n k8s-b07 -o jsonpath='{.status.phase}'
```

Kiểm tra đầy đủ trạng thái:

```bash
kubectl describe pod <pod-name> -n k8s-b07
```

### 5.3. Container state

Bên trong một Pod, Kubernetes theo dõi trạng thái từng container.

Có ba container state chính:

| Container state | Ý nghĩa |
|---|---|
| `Waiting` | Container chưa chạy. Có thể đang pull image, chờ volume, chờ secret/config, hoặc bị lỗi image. |
| `Running` | Container đang chạy. |
| `Terminated` | Container đã kết thúc, có exit code và lý do kết thúc. |

Quan sát container state:

```bash
kubectl describe pod <pod-name> -n k8s-b07
```

Hoặc dùng JSONPath:

```bash
kubectl get pod <pod-name> -n k8s-b07 \
  -o jsonpath='{.status.containerStatuses[*].state}'
```

### 5.4. restartPolicy

`restartPolicy` quy định kubelet có restart container trong Pod hay không khi container kết thúc.

Các giá trị chính:

| restartPolicy | Ý nghĩa |
|---|---|
| `Always` | Luôn restart container nếu container dừng. Đây là giá trị phổ biến với Deployment. |
| `OnFailure` | Chỉ restart nếu container kết thúc với exit code khác 0. Phù hợp với Job. |
| `Never` | Không restart container. |

Ví dụ Pod dùng `restartPolicy: Always`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: restart-always-demo
  namespace: k8s-b07
spec:
  restartPolicy: Always
  containers:
    - name: app
      image: busybox:1.36
      command: ["sh", "-c", "echo Starting; sleep 10; exit 1"]
```

Giải thích các dòng quan trọng:

```yaml
restartPolicy: Always
```

Dòng này yêu cầu kubelet restart container khi container kết thúc. Vì container trong ví dụ `exit 1`, kubelet sẽ restart container nhiều lần.

```yaml
command: ["sh", "-c", "echo Starting; sleep 10; exit 1"]
```

Container cố ý kết thúc lỗi sau 10 giây để  quan sát số lần restart.

Triển khai:

```bash
kubectl apply -f restart-always-demo.yaml
```

Quan sát:

```bash
kubectl get pod restart-always-demo -n k8s-b07 -w
```

Sau một thời gian, có thể thấy số `RESTARTS` tăng lên.

Dọn dẹp:

```bash
kubectl delete pod restart-always-demo -n k8s-b07
```

---

## 6. Health check trong Kubernetes

### 6.1. Vấn đề thực tế

Không phải cứ process còn chạy là ứng dụng còn khỏe.

Một ứng dụng có thể gặp các tình huống:

- Process vẫn chạy nhưng bị deadlock.
- HTTP server mở port nhưng dependency backend chưa sẵn sàng.
- Ứng dụng cần 60 giây để warm up cache.
- Ứng dụng khởi động chậm, nếu kiểm tra quá sớm sẽ bị restart liên tục.
- Database tạm thời không kết nối được, app không nên nhận traffic nhưng cũng chưa cần restart.

Kubernetes dùng probe để xử lý các tình huống này.

### 6.2. Ba loại probe chính

| Probe | Mục đích | Khi fail thì điều gì xảy ra? |
|---|---|---|
| `startupProbe` | Kiểm tra ứng dụng đã khởi động xong chưa. | Nếu fail quá ngưỡng, kubelet kill container và restart theo `restartPolicy`. Khi startupProbe chưa thành công, liveness/readiness chưa chạy. |
| `readinessProbe` | Kiểm tra container đã sẵn sàng nhận traffic chưa. | Pod bị đánh dấu NotReady và bị loại khỏi EndpointSlice của Service. Container không bị restart chỉ vì readiness fail. |
| `livenessProbe` | Kiểm tra container còn sống/khỏe không. | Nếu fail quá ngưỡng, kubelet restart container. |

Cách nhớ nhanh:

```text
startupProbe   = App đã khởi động xong chưa?
readinessProbe = App đã sẵn sàng nhận traffic chưa?
livenessProbe  = App còn sống khỏe không, hay cần restart?
```

### 6.3. Các cơ chế kiểm tra của probe

Kubernetes hỗ trợ nhiều cơ chế probe. Trong buổi này tập trung vào ba cơ chế dễ hiểu nhất:

| Cơ chế | Ý nghĩa | Ví dụ sử dụng |
|---|---|---|
| `httpGet` | kubelet gửi HTTP request đến container. | Web app, REST API. |
| `tcpSocket` | kubelet kiểm tra port TCP có mở không. | Database, TCP service đơn giản. |
| `exec` | kubelet chạy command bên trong container. | Kiểm tra file, process, script nội bộ. |

Ví dụ `httpGet` probe:

```yaml
readinessProbe:
  httpGet:
    path: /ready
    port: 8080
  initialDelaySeconds: 5
  periodSeconds: 5
```

Ví dụ `tcpSocket` probe:

```yaml
livenessProbe:
  tcpSocket:
    port: 8080
  initialDelaySeconds: 10
  periodSeconds: 10
```

Ví dụ `exec` probe:

```yaml
livenessProbe:
  exec:
    command:
      - cat
      - /tmp/healthy
  periodSeconds: 5
```

### 6.4. Các tham số quan trọng của probe

| Tham số | Ý nghĩa |
|---|---|
| `initialDelaySeconds` | Số giây chờ sau khi container start rồi mới bắt đầu probe. |
| `periodSeconds` | Chu kỳ kiểm tra. |
| `timeoutSeconds` | Thời gian chờ tối đa cho mỗi lần probe. |
| `successThreshold` | Số lần thành công liên tiếp để xem là success. Với liveness/startup thường để 1. |
| `failureThreshold` | Số lần fail liên tiếp trước khi Kubernetes xem probe là thất bại. |

Ví dụ:

```yaml
livenessProbe:
  httpGet:
    path: /healthz
    port: 8080
  initialDelaySeconds: 10
  periodSeconds: 5
  timeoutSeconds: 2
  failureThreshold: 3
```

Cách hiểu:

- Sau khi container chạy 10 giây, kubelet bắt đầu probe.
- Cứ 5 giây kiểm tra một lần.
- Mỗi lần chờ tối đa 2 giây.
- Nếu thất bại 3 lần liên tiếp, container bị xem là không khỏe và sẽ bị restart.

---

## 7. Demo application cho toàn bộ buổi học

Để lab rõ ràng, ta sẽ tạo một ứng dụng HTTP nhỏ bằng Python.

Ứng dụng này có các endpoint:

| Endpoint | Mục đích |
|---|---|
| `/` | Trả về thông tin app, version, hostname. |
| `/startup` | Thành công sau khi app đã chạy đủ thời gian startup delay. |
| `/ready` | Trả về 200 nếu app ready, 503 nếu app không ready. |
| `/healthz` | Trả về 200 nếu app live, 500 nếu app bị đánh dấu unhealthy. |
| `/break-readiness` | Cố ý làm readiness fail. |
| `/fix-readiness` | Khôi phục readiness. |
| `/break-liveness` | Cố ý làm liveness fail. |
| `/exit` | Cố ý cho process thoát với exit code 1. |
| `/sleep` | Giả lập request xử lý chậm. |

### 7.1. Tạo ConfigMap chứa code ứng dụng

Tạo file `01-configmap-app.yaml`:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: lifecycle-demo-app
  namespace: k8s-b07
data:
  app.py: |
    import os
    import sys
    import time
    import signal
    import socket
    from http.server import BaseHTTPRequestHandler, HTTPServer

    APP_VERSION = os.getenv("APP_VERSION", "v1")
    STARTUP_DELAY = int(os.getenv("STARTUP_DELAY", "0"))
    PORT = int(os.getenv("PORT", "8080"))

    started_at = time.time()
    readiness_ok = True
    liveness_ok = True

    def handle_sigterm(signum, frame):
        print("Received SIGTERM. Start graceful shutdown...", flush=True)
        time.sleep(5)
        print("Graceful shutdown completed.", flush=True)
        sys.exit(0)

    signal.signal(signal.SIGTERM, handle_sigterm)

    class Handler(BaseHTTPRequestHandler):
        def _send(self, code, body):
            self.send_response(code)
            self.send_header("Content-Type", "text/plain")
            self.end_headers()
            self.wfile.write(body.encode("utf-8"))

        def log_message(self, format, *args):
            print("%s - - [%s] %s" % (self.client_address[0], self.log_date_time_string(), format % args), flush=True)

        def do_GET(self):
            global readiness_ok
            global liveness_ok

            uptime = int(time.time() - started_at)
            hostname = socket.gethostname()

            if self.path == "/":
                body = f"Hello from lifecycle-demo {APP_VERSION}\nhostname={hostname}\nuptime={uptime}s\n"
                self._send(200, body)

            elif self.path == "/startup":
                if uptime >= STARTUP_DELAY:
                    self._send(200, f"startup ok after {uptime}s\n")
                else:
                    self._send(503, f"starting, uptime={uptime}s, need={STARTUP_DELAY}s\n")

            elif self.path == "/ready":
                if readiness_ok:
                    self._send(200, "ready\n")
                else:
                    self._send(503, "not ready\n")

            elif self.path == "/healthz":
                if liveness_ok:
                    self._send(200, "live\n")
                else:
                    self._send(500, "unhealthy\n")

            elif self.path == "/break-readiness":
                readiness_ok = False
                self._send(200, "readiness is now broken\n")

            elif self.path == "/fix-readiness":
                readiness_ok = True
                self._send(200, "readiness is now fixed\n")

            elif self.path == "/break-liveness":
                liveness_ok = False
                self._send(200, "liveness is now broken\n")

            elif self.path == "/exit":
                self._send(200, "process will exit with code 1\n")
                sys.stdout.flush()
                os._exit(1)

            elif self.path == "/sleep":
                time.sleep(10)
                self._send(200, "slow response completed\n")

            else:
                self._send(404, "not found\n")

    print(f"Starting lifecycle-demo {APP_VERSION} on port {PORT}, startup delay={STARTUP_DELAY}s", flush=True)
    server = HTTPServer(("0.0.0.0", PORT), Handler)
    server.serve_forever()
```

Triển khai:

```bash
kubectl apply -f 01-configmap-app.yaml
```

Kiểm tra:

```bash
kubectl get configmap lifecycle-demo-app -n k8s-b07
```

### 7.2. Giải thích các phần quan trọng

```yaml
data:
  app.py: |
```

ConfigMap lưu nội dung file `app.py`. Ở Deployment phía sau, file này sẽ được mount vào container để chạy ứng dụng.

```python
APP_VERSION = os.getenv("APP_VERSION", "v1")
STARTUP_DELAY = int(os.getenv("STARTUP_DELAY", "0"))
```

Ứng dụng đọc version và thời gian startup delay từ environment variable. Khi update `APP_VERSION`, Deployment sẽ tạo rollout mới.

```python
readiness_ok = True
liveness_ok = True
```

Hai biến này dùng để giả lập trạng thái ready/live của app.

```python
elif self.path == "/break-readiness":
    readiness_ok = False
```

Endpoint này cố ý làm readiness probe thất bại.

```python
elif self.path == "/break-liveness":
    liveness_ok = False
```

Endpoint này cố ý làm liveness probe thất bại. Sau một số lần probe fail, kubelet sẽ restart container.

```python
signal.signal(signal.SIGTERM, handle_sigterm)
```

Khi Pod bị xóa hoặc rollout thay thế, container sẽ nhận SIGTERM. Ứng dụng bắt tín hiệu này để giả lập graceful shutdown.

---

## 8. Manifest Deployment dùng startupProbe, readinessProbe và livenessProbe

Tạo file `02-deployment-lifecycle-demo.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: lifecycle-demo
  namespace: k8s-b07
  labels:
    app: lifecycle-demo
spec:
  replicas: 3
  revisionHistoryLimit: 5
  progressDeadlineSeconds: 120
  minReadySeconds: 5
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 1
  selector:
    matchLabels:
      app: lifecycle-demo
  template:
    metadata:
      labels:
        app: lifecycle-demo
    spec:
      terminationGracePeriodSeconds: 30
      containers:
        - name: app
          image: python:3.12-alpine
          imagePullPolicy: IfNotPresent
          command: ["python", "/app/app.py"]
          env:
            - name: APP_VERSION
              value: "v1"
            - name: STARTUP_DELAY
              value: "10"
            - name: PORT
              value: "8080"
          ports:
            - name: http
              containerPort: 8080
          startupProbe:
            httpGet:
              path: /startup
              port: http
            periodSeconds: 3
            timeoutSeconds: 2
            failureThreshold: 10
          readinessProbe:
            httpGet:
              path: /ready
              port: http
            initialDelaySeconds: 3
            periodSeconds: 5
            timeoutSeconds: 2
            failureThreshold: 2
            successThreshold: 1
          livenessProbe:
            httpGet:
              path: /healthz
              port: http
            initialDelaySeconds: 5
            periodSeconds: 5
            timeoutSeconds: 2
            failureThreshold: 3
          lifecycle:
            preStop:
              exec:
                command: ["sh", "-c", "echo preStop hook started; sleep 10"]
          volumeMounts:
            - name: app-code
              mountPath: /app
      volumes:
        - name: app-code
          configMap:
            name: lifecycle-demo-app
```

Triển khai:

```bash
kubectl apply -f 02-deployment-lifecycle-demo.yaml
```

Theo dõi Pod:

```bash
kubectl get pod -n k8s-b07 -l app=lifecycle-demo -w
```

Kiểm tra Deployment:

```bash
kubectl get deployment lifecycle-demo -n k8s-b07
```

Kiểm tra ReplicaSet:

```bash
kubectl get rs -n k8s-b07 -l app=lifecycle-demo
```

---

## 9. Giải thích manifest Deployment

### 9.1. Số lượng replica

```yaml
replicas: 3
```

Deployment yêu cầu luôn có 3 Pod chạy ứng dụng. Nếu một Pod bị xóa, ReplicaSet sẽ tạo lại Pod mới để đưa số lượng Pod về 3.

Đây là nền tảng của self-healing ở cấp Pod.

### 9.2. Giữ lịch sử revision

```yaml
revisionHistoryLimit: 5
```

Kubernetes giữ tối đa 5 ReplicaSet cũ để có thể rollback. Nếu không cần rollback quá sâu, không nên giữ quá nhiều revision cũ.

### 9.3. Giới hạn thời gian rollout

```yaml
progressDeadlineSeconds: 120
```

Nếu Deployment không tiến triển trong 120 giây, Kubernetes đánh dấu rollout là failed. Điều này rất hữu ích khi image lỗi, readiness fail hoặc Pod mới không thể available.

### 9.4. Pod mới phải Ready ổn định trong một khoảng thời gian

```yaml
minReadySeconds: 5
```

Pod mới phải ở trạng thái Ready ít nhất 5 giây mới được xem là Available. Tham số này giúp tránh tình huống Pod vừa Ready đã bị xem là ổn, nhưng ngay sau đó lại fail.

### 9.5. RollingUpdate strategy

```yaml
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxSurge: 1
    maxUnavailable: 1
```

Deployment sẽ update theo kiểu rolling update.

Với `replicas: 3`:

- `maxSurge: 1` cho phép tạo tối đa 1 Pod vượt quá số replica mong muốn. Trong quá trình update có thể có tối đa 4 Pod.
- `maxUnavailable: 1` cho phép tối đa 1 Pod không available trong quá trình update. Như vậy luôn cố gắng duy trì ít nhất 2 Pod available.

### 9.6. Selector và template label

```yaml
selector:
  matchLabels:
    app: lifecycle-demo
```

Deployment dùng selector này để tìm các Pod thuộc quyền quản lý của nó.

```yaml
template:
  metadata:
    labels:
      app: lifecycle-demo
```

Pod template phải có label khớp với selector. Nếu selector và label không khớp, Deployment không thể quản lý Pod đúng cách.

### 9.7. Graceful termination

```yaml
terminationGracePeriodSeconds: 30
```

Khi Pod bị xóa, kubelet cho container tối đa 30 giây để shutdown trước khi bị force kill.

### 9.8. Startup probe

```yaml
startupProbe:
  httpGet:
    path: /startup
    port: http
  periodSeconds: 3
  timeoutSeconds: 2
  failureThreshold: 10
```

Ý nghĩa:

- Kubelet gọi endpoint `/startup`.
- Mỗi 3 giây kiểm tra một lần.
- Cho phép fail tối đa 10 lần.
- Tổng thời gian chờ khởi động xấp xỉ 30 giây.
- Trong lúc startupProbe chưa thành công, readinessProbe và livenessProbe chưa chạy.

Vì app được cấu hình `STARTUP_DELAY=10`, startupProbe này đủ rộng để app khởi động xong.

### 9.9. Readiness probe

```yaml
readinessProbe:
  httpGet:
    path: /ready
    port: http
  initialDelaySeconds: 3
  periodSeconds: 5
  timeoutSeconds: 2
  failureThreshold: 2
  successThreshold: 1
```

Ý nghĩa:

- Kubelet kiểm tra endpoint `/ready`.
- Nếu fail 2 lần liên tiếp, Pod bị đánh dấu NotReady.
- Pod NotReady sẽ bị loại khỏi EndpointSlice của Service.
- Container không bị restart chỉ vì readinessProbe fail.

Readiness probe trả lời câu hỏi: “Pod này đã nên nhận traffic chưa?”

### 9.10. Liveness probe

```yaml
livenessProbe:
  httpGet:
    path: /healthz
    port: http
  initialDelaySeconds: 5
  periodSeconds: 5
  timeoutSeconds: 2
  failureThreshold: 3
```

Ý nghĩa:

- Kubelet kiểm tra endpoint `/healthz`.
- Nếu fail 3 lần liên tiếp, kubelet restart container.

Liveness probe trả lời câu hỏi: “Container này còn sống khỏe không, hay cần restart?”

### 9.11. preStop hook

```yaml
lifecycle:
  preStop:
    exec:
      command: ["sh", "-c", "echo preStop hook started; sleep 10"]
```

Khi Pod bị terminate, kubelet chạy hook này trước khi dừng container. Trong thực tế, `preStop` thường dùng để:

- Chờ load balancer ngưng gửi traffic.
- Gửi tín hiệu deregister đến service registry.
- Cho app thời gian hoàn tất request đang xử lý.
- Delay nhẹ để EndpointSlice cập nhật xong trước khi process thật sự dừng.

---

## 10. Tạo Service cho ứng dụng

Tạo file `03-service-lifecycle-demo.yaml`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: lifecycle-demo
  namespace: k8s-b07
  labels:
    app: lifecycle-demo
spec:
  type: ClusterIP
  selector:
    app: lifecycle-demo
  ports:
    - name: http
      port: 80
      targetPort: http
```

Triển khai:

```bash
kubectl apply -f 03-service-lifecycle-demo.yaml
```

Kiểm tra:

```bash
kubectl get svc lifecycle-demo -n k8s-b07
```

Kiểm tra EndpointSlice:

```bash
kubectl get endpointslice -n k8s-b07 -l kubernetes.io/service-name=lifecycle-demo
```

Hoặc xem Endpoint truyền thống:

```bash
kubectl get endpoints lifecycle-demo -n k8s-b07
```

Kiểm tra truy cập Service từ trong cluster:

```bash
kubectl run curl-test -n k8s-b07 --rm -it --restart=Never \
  --image=curlimages/curl:8.10.1 --command -- \
  curl -s http://lifecycle-demo
```

Kết quả mong đợi:

```text
Hello from lifecycle-demo v1
hostname=lifecycle-demo-xxxxxxxxxx-yyyyy
uptime=...
```

---

## 11. Lab 01 - Quan sát Pod lifecycle

### 11.1. Mục tiêu

Quan sát quá trình Pod đi từ lúc được tạo đến lúc Running/Ready.

### 11.2. Theo dõi Pod

Chạy lệnh:

```bash
kubectl get pod -n k8s-b07 -l app=lifecycle-demo -w
```

Trong một terminal khác, xóa một Pod để quan sát Pod mới được tạo lại:

```bash
POD=$(kubectl get pod -n k8s-b07 -l app=lifecycle-demo -o jsonpath='{.items[0].metadata.name}')
kubectl delete pod $POD -n k8s-b07
```

Quan sát kết quả:

```text
NAME                              READY   STATUS              RESTARTS   AGE
lifecycle-demo-xxxxx-yyyyy         0/1     ContainerCreating   0          2s
lifecycle-demo-xxxxx-yyyyy         0/1     Running             0          5s
lifecycle-demo-xxxxx-yyyyy         1/1     Running             0          18s
```

### 11.3. Phân tích

Trong quá trình này:

- Pod mới được ReplicaSet tạo ra.
- Scheduler gán Pod vào một worker node.
- Kubelet trên node đó pull image nếu cần.
- Kubelet tạo container thông qua containerd.
- Startup probe chạy trước.
- Sau khi startup probe thành công, readiness probe và liveness probe bắt đầu chạy.
- Khi readiness probe thành công, Pod chuyển sang Ready.
- Service có thể gửi traffic đến Pod Ready.

### 11.4. Kiểm tra phase và container state

Lấy tên Pod:

```bash
POD=$(kubectl get pod -n k8s-b07 -l app=lifecycle-demo -o jsonpath='{.items[0].metadata.name}')
echo $POD
```

Xem Pod phase:

```bash
kubectl get pod $POD -n k8s-b07 -o jsonpath='{.status.phase}'
echo
```

Xem chi tiết:

```bash
kubectl describe pod $POD -n k8s-b07
```

Xem logs:

```bash
kubectl logs $POD -n k8s-b07
```

---

## 12. Lab 02 - Readiness probe và Service endpoint

### 12.1. Mục tiêu

Chứng minh rằng readiness probe fail không restart container, nhưng làm Pod bị loại khỏi Service endpoint.

### 12.2. Xem endpoint ban đầu

```bash
kubectl get pod -n k8s-b07 -l app=lifecycle-demo -o wide
kubectl get endpoints lifecycle-demo -n k8s-b07
```

Nếu có 3 Pod Ready, Service sẽ có 3 endpoint tương ứng.

### 12.3. Cố ý làm một Pod NotReady

Chọn một Pod:

```bash
POD=$(kubectl get pod -n k8s-b07 -l app=lifecycle-demo -o jsonpath='{.items[0].metadata.name}')
echo $POD
```

Gọi endpoint phá readiness bên trong Pod:

```bash
kubectl exec -n k8s-b07 $POD -- wget -qO- http://127.0.0.1:8080/break-readiness
```

Theo dõi Pod:

```bash
kubectl get pod -n k8s-b07 -l app=lifecycle-demo -w
```

Sau một thời gian ngắn, Pod có thể chuyển thành `0/1` Ready:

```text
NAME                              READY   STATUS    RESTARTS   AGE
lifecycle-demo-xxxxx-yyyyy         0/1     Running   0          5m
```

### 12.4. Kiểm tra endpoint sau khi readiness fail

```bash
kubectl get endpoints lifecycle-demo -n k8s-b07
```

Số endpoint nên giảm đi. Pod NotReady không còn được Service chọn để nhận traffic.

Kiểm tra chi tiết EndpointSlice:

```bash
kubectl get endpointslice -n k8s-b07 \
  -l kubernetes.io/service-name=lifecycle-demo -o yaml
```

Tìm phần `conditions.ready`.

### 12.5. Kiểm tra container không bị restart

```bash
kubectl get pod $POD -n k8s-b07
```

Cột `RESTARTS` không tăng chỉ vì readiness fail.

Đây là khác biệt rất quan trọng giữa readiness probe và liveness probe.

### 12.6. Khôi phục readiness

```bash
kubectl exec -n k8s-b07 $POD -- wget -qO- http://127.0.0.1:8080/fix-readiness
```

Theo dõi lại:

```bash
kubectl get pod $POD -n k8s-b07 -w
```

Kiểm tra endpoint:

```bash
kubectl get endpoints lifecycle-demo -n k8s-b07
```

Pod sẽ được đưa trở lại endpoint khi readiness probe thành công.

### 12.7. Kết luận

Readiness probe dùng để bảo vệ traffic.

Nếu app chưa sẵn sàng, Kubernetes không nhất thiết restart app. Kubernetes chỉ cần ngưng gửi request đến app đó.

---

## 13. Lab 03 - Liveness probe và container restart

### 13.1. Mục tiêu

Chứng minh rằng liveness probe fail sẽ làm kubelet restart container.

### 13.2. Chọn Pod

```bash
POD=$(kubectl get pod -n k8s-b07 -l app=lifecycle-demo -o jsonpath='{.items[0].metadata.name}')
echo $POD
```

Xem số restart hiện tại:

```bash
kubectl get pod $POD -n k8s-b07
```

### 13.3. Cố ý làm liveness fail

```bash
kubectl exec -n k8s-b07 $POD -- wget -qO- http://127.0.0.1:8080/break-liveness
```

Theo dõi Pod:

```bash
kubectl get pod $POD -n k8s-b07 -w
```

Sau khoảng vài lần probe, cột `RESTARTS` sẽ tăng.

Xem event:

```bash
kubectl describe pod $POD -n k8s-b07
```

Có thể thấy event liên quan đến liveness probe failed và container restart.

Xem logs của container hiện tại:

```bash
kubectl logs $POD -n k8s-b07
```

Xem logs của container trước khi restart:

```bash
kubectl logs $POD -n k8s-b07 --previous
```

### 13.4. Phân tích

Luồng xử lý:

```text
App bị đánh dấu unhealthy
        |
        v
/healthz trả về HTTP 500
        |
        v
livenessProbe fail nhiều lần liên tiếp
        |
        v
kubelet restart container
        |
        v
container chạy lại từ đầu trong cùng Pod
```

Lưu ý:

- Liveness probe không tạo Pod mới.
- Kubelet restart container bên trong Pod.
- Nếu container restart thành công, Pod vẫn có thể tiếp tục phục vụ.

---

## 14. Lab 04 - Container process exit và restartPolicy

### 14.1. Mục tiêu

Quan sát kubelet restart container khi process chính của container thoát lỗi.

### 14.2. Gọi endpoint làm process exit

Chọn Pod:

```bash
POD=$(kubectl get pod -n k8s-b07 -l app=lifecycle-demo -o jsonpath='{.items[0].metadata.name}')
echo $POD
```

Gọi `/exit`:

```bash
kubectl exec -n k8s-b07 $POD -- wget -qO- http://127.0.0.1:8080/exit
```

Có thể lệnh `kubectl exec` sẽ bị ngắt vì process bên trong container thoát.

Theo dõi:

```bash
kubectl get pod $POD -n k8s-b07 -w
```

Xem chi tiết:

```bash
kubectl describe pod $POD -n k8s-b07
```

### 14.3. Phân tích

Vì Pod thuộc Deployment, Pod template mặc định dùng `restartPolicy: Always`. Khi process chính trong container exit code 1, kubelet restart container.

Điểm cần phân biệt:

- Container crash → kubelet restart container.
- Pod bị xóa → ReplicaSet tạo Pod mới.
- Node lỗi → control plane tạo Pod thay thế trên node khác nếu có thể.

---

## 15. Lab 05 - Graceful termination khi xóa Pod

### 15.1. Mục tiêu

Hiểu quá trình Pod termination, `preStop` hook và `terminationGracePeriodSeconds`.

### 15.2. Theo dõi Pod

Mở terminal 1:

```bash
kubectl get pod -n k8s-b07 -l app=lifecycle-demo -w
```

Mở terminal 2:

```bash
POD=$(kubectl get pod -n k8s-b07 -l app=lifecycle-demo -o jsonpath='{.items[0].metadata.name}')
kubectl logs $POD -n k8s-b07 -f
```

Mở terminal 3 và xóa Pod:

```bash
kubectl delete pod $POD -n k8s-b07
```

### 15.3. Quan sát

Trong logs có thể thấy app nhận SIGTERM:

```text
Received SIGTERM. Start graceful shutdown...
Graceful shutdown completed.
```

Trong terminal watch Pod, Pod cũ chuyển sang `Terminating` và Pod mới được tạo để thay thế.

### 15.4. Luồng termination

```text
kubectl delete pod
        |
        v
API Server đánh dấu Pod đang terminating
        |
        v
EndpointSlice cập nhật, Pod không còn là endpoint ready cho Service
        |
        v
kubelet chạy preStop hook
        |
        v
container nhận SIGTERM
        |
        v
app có thời gian shutdown trong terminationGracePeriodSeconds
        |
        v
nếu quá thời gian grace period, container bị SIGKILL
```

### 15.5. Vì sao graceful termination quan trọng?

Trong production, nếu app bị kill ngay lập tức:

- Request đang xử lý có thể bị ngắt.
- Transaction có thể dang dở.
- Client thấy lỗi 502/503.
- Log hoặc buffer chưa kịp flush.
- App chưa kịp deregister khỏi dependency bên ngoài.

Kubernetes không tự làm cho ứng dụng graceful nếu ứng dụng không xử lý SIGTERM đúng cách. App nên bắt SIGTERM và shutdown sạch.

---

## 16. Rolling update trong Deployment

### 16.1. Rolling update là gì?

Rolling update là quá trình thay thế dần Pod cũ bằng Pod mới.

Thay vì dừng toàn bộ ứng dụng rồi chạy phiên bản mới, Kubernetes làm theo kiểu:

```text
Tạo một phần Pod mới
        |
        v
Chờ Pod mới Ready
        |
        v
Giảm một phần Pod cũ
        |
        v
Tiếp tục lặp lại đến khi toàn bộ Pod là phiên bản mới
```

### 16.2. Khi nào Deployment tạo rollout mới?

Deployment chỉ tạo rollout mới khi `.spec.template` thay đổi.

Ví dụ các thay đổi tạo rollout mới:

- Đổi image.
- Đổi environment variable trong Pod template.
- Đổi command/args.
- Đổi probe.
- Đổi label/annotation trong Pod template.
- Đổi volume mount trong Pod template.

Các thay đổi không tạo rollout mới:

- Scale số replicas.
- Đổi field ngoài Pod template nhưng không ảnh hưởng template.

### 16.3. Vai trò của ReplicaSet trong rollout

Khi Deployment rollout:

```text
Deployment
   |
   +-- ReplicaSet cũ  -> Pod phiên bản cũ
   |
   +-- ReplicaSet mới -> Pod phiên bản mới
```

Deployment controller scale up ReplicaSet mới và scale down ReplicaSet cũ theo strategy đã cấu hình.

---

## 17. Lab 06 - Rolling update thành công

### 17.1. Mục tiêu

Update app từ version `v1` sang `v2` và quan sát Deployment tạo ReplicaSet mới.

### 17.2. Xem trạng thái ban đầu

```bash
kubectl get deployment lifecycle-demo -n k8s-b07
kubectl get rs -n k8s-b07 -l app=lifecycle-demo
kubectl get pod -n k8s-b07 -l app=lifecycle-demo
```

Kiểm tra version hiện tại:

```bash
kubectl run curl-test -n k8s-b07 --rm -it --restart=Never \
  --image=curlimages/curl:8.10.1 --command -- \
  curl -s http://lifecycle-demo
```

Kết quả có `v1`.

### 17.3. Update APP_VERSION sang v2

Dùng lệnh:

```bash
kubectl set env deployment/lifecycle-demo APP_VERSION=v2 -n k8s-b07
```

Lệnh này thay đổi environment variable trong Pod template, do đó sẽ trigger rollout.

### 17.4. Theo dõi rollout

```bash
kubectl rollout status deployment/lifecycle-demo -n k8s-b07
```

Theo dõi ReplicaSet:

```bash
kubectl get rs -n k8s-b07 -l app=lifecycle-demo -w
```

Theo dõi Pod:

```bash
kubectl get pod -n k8s-b07 -l app=lifecycle-demo -w
```

### 17.5. Kiểm tra rollout history

```bash
kubectl rollout history deployment/lifecycle-demo -n k8s-b07
```

Kết quả có thể tương tự:

```text
deployment.apps/lifecycle-demo
REVISION  CHANGE-CAUSE
1         <none>
2         <none>
```

### 17.6. Kiểm tra version sau update

```bash
kubectl run curl-test -n k8s-b07 --rm -it --restart=Never \
  --image=curlimages/curl:8.10.1 --command -- \
  curl -s http://lifecycle-demo
```

Kết quả mong đợi có `v2`.

### 17.7. Ghi change cause

Để lịch sử rollout dễ hiểu hơn, có thể annotate Deployment:

```bash
kubectl annotate deployment/lifecycle-demo -n k8s-b07 \
  kubernetes.io/change-cause="Update APP_VERSION from v1 to v2" --overwrite
```

Xem lại:

```bash
kubectl rollout history deployment/lifecycle-demo -n k8s-b07
```

Lưu ý: annotation này phục vụ quan sát lịch sử, không tự rollback hay thay đổi Pod.

---

## 18. Lab 07 - Rolling update bị lỗi do image sai

### 18.1. Mục tiêu

Tạo lỗi rollout để  thấy Kubernetes giữ lại Pod cũ, rollout không hoàn tất và có thể rollback.

### 18.2. Set image sai

```bash
kubectl set image deployment/lifecycle-demo app=python:3.12-alpine-not-exist -n k8s-b07
```

Ghi change cause:

```bash
kubectl annotate deployment/lifecycle-demo -n k8s-b07 \
  kubernetes.io/change-cause="Bad rollout: wrong image tag" --overwrite
```

### 18.3. Theo dõi rollout

```bash
kubectl rollout status deployment/lifecycle-demo -n k8s-b07
```

Có thể thấy rollout bị kẹt vì image pull lỗi.

Trong terminal khác:

```bash
kubectl get pod -n k8s-b07 -l app=lifecycle-demo -w
```

Có thể thấy Pod mới ở trạng thái `ImagePullBackOff` hoặc `ErrImagePull`.

Xem ReplicaSet:

```bash
kubectl get rs -n k8s-b07 -l app=lifecycle-demo
```

Xem Deployment:

```bash
kubectl describe deployment lifecycle-demo -n k8s-b07
```

### 18.4. Phân tích

Vì Deployment dùng:

```yaml
maxUnavailable: 1
maxSurge: 1
```

Kubernetes không kill toàn bộ Pod cũ cùng lúc. Một số Pod phiên bản cũ vẫn tiếp tục phục vụ traffic. Đây là một lợi ích lớn của rolling update.

Nếu không có readiness probe, một Pod mới có thể được xem là ready quá sớm. Vì vậy rolling update production thường cần readiness probe tốt.

---

## 19. Lab 08 - Rollback Deployment

### 19.1. Mục tiêu

Rollback Deployment về revision trước khi rollout lỗi.

### 19.2. Xem lịch sử rollout

```bash
kubectl rollout history deployment/lifecycle-demo -n k8s-b07
```

Xem chi tiết một revision:

```bash
kubectl rollout history deployment/lifecycle-demo -n k8s-b07 --revision=2
```

### 19.3. Rollback về revision trước

Rollback về revision gần nhất ổn định:

```bash
kubectl rollout undo deployment/lifecycle-demo -n k8s-b07
```

Hoặc rollback về revision cụ thể:

```bash
kubectl rollout undo deployment/lifecycle-demo -n k8s-b07 --to-revision=2
```

Theo dõi:

```bash
kubectl rollout status deployment/lifecycle-demo -n k8s-b07
```

Kiểm tra image:

```bash
kubectl get deployment lifecycle-demo -n k8s-b07 \
  -o jsonpath='{.spec.template.spec.containers[0].image}'
echo
```

Kiểm tra version:

```bash
kubectl run curl-test -n k8s-b07 --rm -it --restart=Never \
  --image=curlimages/curl:8.10.1 --command -- \
  curl -s http://lifecycle-demo
```

### 19.4. Phân tích

Rollback của Deployment chủ yếu rollback phần Pod template.

Điều này có nghĩa:

- Image có thể quay lại version cũ.
- Env trong Pod template có thể quay lại version cũ.
- Probe trong Pod template có thể quay lại version cũ.
- Scale replicas không nhất thiết là một revision rollback theo cách giống Pod template.

### 19.5. Bài học production

Trong môi trường thật:

- Không nên update image bằng tag `latest`.
- Nên dùng tag version rõ ràng hoặc image digest.
- Nên có readiness probe trước khi rolling update.
- Nên theo dõi `kubectl rollout status` trong pipeline CI/CD.
- Nên rollback ngay khi rollout stuck hoặc error rate tăng.

---

## 20. Lab 09 - Rolling update bị kẹt do readiness fail

### 20.1. Mục tiêu

Chứng minh rằng nếu Pod mới không Ready, Deployment rollout sẽ không hoàn tất.

### 20.2. Cập nhật manifest làm app version v3 nhưng readiness ban đầu fail

Ta tạo một Deployment patch thêm biến môi trường `APP_VERSION=v3`. Sau đó sẽ phá readiness để thấy ảnh hưởng. Tuy nhiên, để minh họa rõ rollout kẹt vì readiness, ta dùng một biến môi trường mới và một command sửa nhỏ không cần thay code chính.

Cách đơn giản hơn trong lab: set startup delay quá cao nhưng startupProbe quá thấp sẽ làm Pod mới restart liên tục. Đây là lỗi gần giống app khởi động chậm nhưng probe cấu hình sai.

Patch Deployment:

```bash
kubectl patch deployment lifecycle-demo -n k8s-b07 --type='json' -p='[
  {"op":"replace","path":"/spec/template/spec/containers/0/env/0/value","value":"v3"},
  {"op":"replace","path":"/spec/template/spec/containers/0/env/1/value","value":"120"},
  {"op":"replace","path":"/spec/template/spec/containers/0/startupProbe/failureThreshold","value":5}
]'
```

Giải thích:

- `APP_VERSION` đổi thành `v3` để tạo rollout mới.
- `STARTUP_DELAY` đổi thành 120 giây.
- `startupProbe.failureThreshold=5`, `periodSeconds=3`, tổng thời gian chờ khoảng 15 giây.
- App cần 120 giây mới startup xong, nhưng startupProbe chỉ cho khoảng 15 giây, nên container sẽ bị restart liên tục.

Theo dõi:

```bash
kubectl rollout status deployment/lifecycle-demo -n k8s-b07
```

Xem Pod:

```bash
kubectl get pod -n k8s-b07 -l app=lifecycle-demo -w
```

Xem describe:

```bash
POD=$(kubectl get pod -n k8s-b07 -l app=lifecycle-demo -o jsonpath='{.items[-1:].metadata.name}')
kubectl describe pod $POD -n k8s-b07
```

### 20.3. Rollback

```bash
kubectl rollout undo deployment/lifecycle-demo -n k8s-b07
kubectl rollout status deployment/lifecycle-demo -n k8s-b07
```

### 20.4. Bài học

Startup probe không phải cấu hình càng chặt càng tốt. Nếu app khởi động chậm, startup probe phải đủ rộng để app có thời gian initialization.

Sai startup probe có thể làm app không bao giờ khởi động được.

---

## 21. Self-healing trong Kubernetes

### 21.1. Self-healing không phải phép màu

Kubernetes self-healing nghĩa là Kubernetes cố gắng đưa trạng thái thực tế về trạng thái mong muốn.

Ví dụ:

- Deployment muốn 3 replicas.
- Hiện tại chỉ còn 2 Pod.
- ReplicaSet tạo thêm 1 Pod.

Hoặc:

- Container trong Pod bị crash.
- Kubelet restart container theo `restartPolicy`.

Hoặc:

- Pod phía sau Service không Ready.
- Service ngưng gửi traffic đến Pod đó.

Nhưng Kubernetes không tự sửa được mọi lỗi.

Ví dụ Kubernetes không tự sửa được:

- Code ứng dụng sai logic.
- Database mất dữ liệu.
- Secret sai password.
- ConfigMap cấu hình sai endpoint backend.
- Storage backend NFS/Ceph/SAN bị lỗi nặng.
- Image không tồn tại trong registry.

Kubernetes giúp phát hiện, cô lập, restart, replace và rollback. Nhưng nguyên nhân gốc vẫn cần người vận hành hoặc pipeline xử lý.

### 21.2. Các tầng self-healing

| Tầng | Thành phần xử lý | Ví dụ |
|---|---|---|
| Container | kubelet | Container crash thì restart. |
| Pod replica | ReplicaSet/Deployment controller | Pod bị xóa thì tạo Pod mới. |
| Node failure | Control plane + controllers | Pod trên node lỗi được thay thế ở node khác nếu điều kiện cho phép. |
| Traffic | Service/EndpointSlice | Pod NotReady bị loại khỏi endpoint. |
| Storage | PV controller / CSI | Volume có thể detach/attach lại tùy loại storage. |

---

## 22. Lab 10 - Self-healing khi Pod bị xóa

### 22.1. Mục tiêu

Chứng minh Deployment duy trì số replica mong muốn.

### 22.2. Xem số Pod hiện tại

```bash
kubectl get deployment lifecycle-demo -n k8s-b07
kubectl get pod -n k8s-b07 -l app=lifecycle-demo
```

### 22.3. Xóa một Pod

```bash
POD=$(kubectl get pod -n k8s-b07 -l app=lifecycle-demo -o jsonpath='{.items[0].metadata.name}')
kubectl delete pod $POD -n k8s-b07
```

Theo dõi:

```bash
kubectl get pod -n k8s-b07 -l app=lifecycle-demo -w
```

### 22.4. Phân tích

Khi Pod bị xóa:

- ReplicaSet phát hiện số Pod thực tế thấp hơn mong muốn.
- ReplicaSet tạo Pod mới.
- Pod mới được scheduler gán vào node.
- Kubelet tạo container.
- Probes kiểm tra app.
- Khi readiness thành công, Pod mới nhận traffic.

Đây là self-healing ở cấp Pod replica.

---

## 23. Lab 11 - Self-healing khi container bị crash

### 23.1. Mục tiêu

Chứng minh kubelet restart container trong cùng Pod khi process chính exit lỗi.

### 23.2. Gọi endpoint `/exit`

```bash
POD=$(kubectl get pod -n k8s-b07 -l app=lifecycle-demo -o jsonpath='{.items[0].metadata.name}')
kubectl exec -n k8s-b07 $POD -- wget -qO- http://127.0.0.1:8080/exit || true
```

Theo dõi restart:

```bash
kubectl get pod $POD -n k8s-b07 -w
```

Xem trạng thái container:

```bash
kubectl describe pod $POD -n k8s-b07
```

### 23.3. Phân tích

Đây là self-healing ở cấp container.

Khác với Lab 10:

- Lab 10: Pod bị xóa, Pod mới được tạo.
- Lab 11: Container trong Pod restart, Pod object vẫn là Pod cũ.

---

## 24. Lab 12 - Optional: Publish ứng dụng qua Ingress để quan sát traffic

Phần này dùng lại kiến thức buổi 06. Nếu môi trường đã có MetalLB và Ingress NGINX Controller, có thể publish app ra ngoài.

### 24.1. Tạo Ingress HTTP

Tạo file `04-ingress-lifecycle-demo.yaml`:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: lifecycle-demo
  namespace: k8s-b07
spec:
  ingressClassName: nginx
  rules:
    - host: lifecycle.lab.k8s.local
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: lifecycle-demo
                port:
                  number: 80
```

Triển khai:

```bash
kubectl apply -f 04-ingress-lifecycle-demo.yaml
```

Kiểm tra:

```bash
kubectl get ingress lifecycle-demo -n k8s-b07
```

Thêm record DNS hoặc `/etc/hosts` trên máy client:

```text
<INGRESS_EXTERNAL_IP> lifecycle.lab.k8s.local
```

Test:

```bash
curl http://lifecycle.lab.k8s.local
```

### 24.2. Quan sát khi một Pod NotReady

Phá readiness một Pod:

```bash
POD=$(kubectl get pod -n k8s-b07 -l app=lifecycle-demo -o jsonpath='{.items[0].metadata.name}')
kubectl exec -n k8s-b07 $POD -- wget -qO- http://127.0.0.1:8080/break-readiness
```

Gửi nhiều request:

```bash
for i in $(seq 1 20); do curl -s http://lifecycle.lab.k8s.local | grep hostname; done
```

Pod NotReady sẽ không nên xuất hiện trong danh sách trả lời traffic qua Service.

---

## 25. Manifest tổng hợp cuối buổi

Nếu muốn triển khai nhanh toàn bộ lab chính, có thể dùng một file duy nhất `b07-lifecycle-full.yaml`.

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: k8s-b07
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: lifecycle-demo-app
  namespace: k8s-b07
data:
  app.py: |
    import os
    import sys
    import time
    import signal
    import socket
    from http.server import BaseHTTPRequestHandler, HTTPServer

    APP_VERSION = os.getenv("APP_VERSION", "v1")
    STARTUP_DELAY = int(os.getenv("STARTUP_DELAY", "0"))
    PORT = int(os.getenv("PORT", "8080"))

    started_at = time.time()
    readiness_ok = True
    liveness_ok = True

    def handle_sigterm(signum, frame):
        print("Received SIGTERM. Start graceful shutdown...", flush=True)
        time.sleep(5)
        print("Graceful shutdown completed.", flush=True)
        sys.exit(0)

    signal.signal(signal.SIGTERM, handle_sigterm)

    class Handler(BaseHTTPRequestHandler):
        def _send(self, code, body):
            self.send_response(code)
            self.send_header("Content-Type", "text/plain")
            self.end_headers()
            self.wfile.write(body.encode("utf-8"))

        def log_message(self, format, *args):
            print("%s - - [%s] %s" % (self.client_address[0], self.log_date_time_string(), format % args), flush=True)

        def do_GET(self):
            global readiness_ok
            global liveness_ok

            uptime = int(time.time() - started_at)
            hostname = socket.gethostname()

            if self.path == "/":
                body = f"Hello from lifecycle-demo {APP_VERSION}\nhostname={hostname}\nuptime={uptime}s\n"
                self._send(200, body)
            elif self.path == "/startup":
                if uptime >= STARTUP_DELAY:
                    self._send(200, f"startup ok after {uptime}s\n")
                else:
                    self._send(503, f"starting, uptime={uptime}s, need={STARTUP_DELAY}s\n")
            elif self.path == "/ready":
                if readiness_ok:
                    self._send(200, "ready\n")
                else:
                    self._send(503, "not ready\n")
            elif self.path == "/healthz":
                if liveness_ok:
                    self._send(200, "live\n")
                else:
                    self._send(500, "unhealthy\n")
            elif self.path == "/break-readiness":
                readiness_ok = False
                self._send(200, "readiness is now broken\n")
            elif self.path == "/fix-readiness":
                readiness_ok = True
                self._send(200, "readiness is now fixed\n")
            elif self.path == "/break-liveness":
                liveness_ok = False
                self._send(200, "liveness is now broken\n")
            elif self.path == "/exit":
                self._send(200, "process will exit with code 1\n")
                sys.stdout.flush()
                os._exit(1)
            elif self.path == "/sleep":
                time.sleep(10)
                self._send(200, "slow response completed\n")
            else:
                self._send(404, "not found\n")

    print(f"Starting lifecycle-demo {APP_VERSION} on port {PORT}, startup delay={STARTUP_DELAY}s", flush=True)
    server = HTTPServer(("0.0.0.0", PORT), Handler)
    server.serve_forever()
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: lifecycle-demo
  namespace: k8s-b07
  labels:
    app: lifecycle-demo
spec:
  replicas: 3
  revisionHistoryLimit: 5
  progressDeadlineSeconds: 120
  minReadySeconds: 5
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 1
  selector:
    matchLabels:
      app: lifecycle-demo
  template:
    metadata:
      labels:
        app: lifecycle-demo
    spec:
      terminationGracePeriodSeconds: 30
      containers:
        - name: app
          image: python:3.12-alpine
          imagePullPolicy: IfNotPresent
          command: ["python", "/app/app.py"]
          env:
            - name: APP_VERSION
              value: "v1"
            - name: STARTUP_DELAY
              value: "10"
            - name: PORT
              value: "8080"
          ports:
            - name: http
              containerPort: 8080
          startupProbe:
            httpGet:
              path: /startup
              port: http
            periodSeconds: 3
            timeoutSeconds: 2
            failureThreshold: 10
          readinessProbe:
            httpGet:
              path: /ready
              port: http
            initialDelaySeconds: 3
            periodSeconds: 5
            timeoutSeconds: 2
            failureThreshold: 2
            successThreshold: 1
          livenessProbe:
            httpGet:
              path: /healthz
              port: http
            initialDelaySeconds: 5
            periodSeconds: 5
            timeoutSeconds: 2
            failureThreshold: 3
          lifecycle:
            preStop:
              exec:
                command: ["sh", "-c", "echo preStop hook started; sleep 10"]
          volumeMounts:
            - name: app-code
              mountPath: /app
      volumes:
        - name: app-code
          configMap:
            name: lifecycle-demo-app
---
apiVersion: v1
kind: Service
metadata:
  name: lifecycle-demo
  namespace: k8s-b07
  labels:
    app: lifecycle-demo
spec:
  type: ClusterIP
  selector:
    app: lifecycle-demo
  ports:
    - name: http
      port: 80
      targetPort: http
```

Triển khai:

```bash
kubectl apply -f b07-lifecycle-full.yaml
kubectl rollout status deployment/lifecycle-demo -n k8s-b07
```

---

## 26. Lab tổng hợp cuối buổi

### 26.1. Yêu cầu lab

 cần thực hiện đầy đủ các bước sau:

- Tạo namespace `k8s-b07`.
- Tạo ConfigMap chứa app Python.
- Tạo Deployment 3 replicas.
- Cấu hình `startupProbe`, `readinessProbe`, `livenessProbe`.
- Cấu hình rolling update strategy.
- Tạo Service `ClusterIP`.
- Kiểm tra app qua Service.
- Làm readiness fail và quan sát EndpointSlice.
- Làm liveness fail và quan sát container restart.
- Update app từ v1 sang v2.
- Tạo rollout lỗi bằng image sai.
- Rollback về revision ổn định.
- Xóa một Pod và quan sát self-healing.

### 26.2. Checklist kết quả

 hoàn thành lab khi chứng minh được:

```bash
kubectl get deployment lifecycle-demo -n k8s-b07
```

Có `READY` là `3/3`.

```bash
kubectl get pod -n k8s-b07 -l app=lifecycle-demo
```

Có 3 Pod Running.

```bash
kubectl get endpoints lifecycle-demo -n k8s-b07
```

Có endpoint tương ứng với các Pod Ready.

```bash
kubectl rollout history deployment/lifecycle-demo -n k8s-b07
```

Có nhiều revision.

```bash
kubectl run curl-test -n k8s-b07 --rm -it --restart=Never \
  --image=curlimages/curl:8.10.1 --command -- \
  curl -s http://lifecycle-demo
```

Trả về version app hiện tại.

---

## 27. Troubleshooting thường gặp

### 27.1. Pod bị `ImagePullBackOff`

Kiểm tra:

```bash
kubectl describe pod <pod-name> -n k8s-b07
```

Nguyên nhân thường gặp:

- Sai tên image.
- Sai tag.
- Không truy cập được registry.
- Registry cần authentication nhưng chưa cấu hình imagePullSecret.

Trong bài này, lỗi image sai được dùng có chủ đích để học rollback.

### 27.2. Pod Running nhưng không Ready

Kiểm tra:

```bash
kubectl describe pod <pod-name> -n k8s-b07
kubectl logs <pod-name> -n k8s-b07
```

Nguyên nhân thường gặp:

- Readiness endpoint trả lỗi.
- App chưa khởi động xong.
- Sai path hoặc port trong readinessProbe.
- Dependency backend chưa sẵn sàng.

### 27.3. Pod restart liên tục

Kiểm tra:

```bash
kubectl get pod <pod-name> -n k8s-b07
kubectl describe pod <pod-name> -n k8s-b07
kubectl logs <pod-name> -n k8s-b07 --previous
```

Nguyên nhân thường gặp:

- App crash.
- Liveness probe cấu hình sai.
- Startup probe quá ngắn.
- ConfigMap/Secret sai.
- App không bắt được dependency cần thiết.

### 27.4. Rollout bị stuck

Kiểm tra:

```bash
kubectl rollout status deployment/lifecycle-demo -n k8s-b07
kubectl describe deployment lifecycle-demo -n k8s-b07
kubectl get rs -n k8s-b07 -l app=lifecycle-demo
kubectl get pod -n k8s-b07 -l app=lifecycle-demo
```

Nguyên nhân thường gặp:

- Image lỗi.
- Pod mới không Ready.
- Probe sai.
- Node không đủ tài nguyên.
- PVC không mount được.

Cách xử lý nhanh:

```bash
kubectl rollout undo deployment/lifecycle-demo -n k8s-b07
```

### 27.5. Service không gửi traffic đến Pod

Kiểm tra:

```bash
kubectl get svc lifecycle-demo -n k8s-b07
kubectl get endpoints lifecycle-demo -n k8s-b07
kubectl get pod -n k8s-b07 -l app=lifecycle-demo --show-labels
```

Nguyên nhân thường gặp:

- Service selector không khớp Pod label.
- Pod chưa Ready.
- Readiness probe fail.
- Sai `targetPort`.

---

## 28. Best practices cho production

### 28.1. Luôn cấu hình readiness probe cho ứng dụng nhận traffic

Readiness probe giúp tránh gửi traffic đến Pod chưa sẵn sàng.

Đây là probe gần như bắt buộc với web app, API, backend service.

### 28.2. Dùng liveness probe cẩn thận

Liveness probe sai có thể gây cascading failure.

Không nên dùng liveness probe để kiểm tra dependency bên ngoài một cách quá cứng nhắc. Ví dụ, nếu database chậm tạm thời mà liveness fail, Kubernetes restart hàng loạt app, khiến hệ thống tệ hơn.

Liveness nên kiểm tra sức khỏe nội tại của process.

### 28.3. Dùng startup probe cho ứng dụng khởi động chậm

Nếu app cần nhiều thời gian để warm up, migrate cache, load model hoặc load dữ liệu, hãy dùng startup probe.

Không nên tăng `initialDelaySeconds` của liveness quá lớn để thay thế startup probe trong các app khởi động chậm.

### 28.4. Không dùng image tag `latest` cho production

Nên dùng:

```text
myapp:1.2.3
```

Hoặc image digest:

```text
myapp@sha256:...
```

Điều này giúp rollback và audit dễ hơn.

### 28.5. Đặt `progressDeadlineSeconds`

Nếu không đặt deadline hợp lý, pipeline hoặc người vận hành có thể chờ rollout quá lâu mà không biết rollout đã stuck.

### 28.6. Dùng `minReadySeconds` cho app nhạy cảm

Với app dễ “Ready sớm rồi chết”, `minReadySeconds` giúp Deployment chỉ xem Pod là available khi Pod Ready ổn định trong một khoảng thời gian.

### 28.7. Ứng dụng phải xử lý SIGTERM

Kubernetes gửi SIGTERM khi terminate container. App cần:

- Ngừng nhận request mới.
- Hoàn tất request đang chạy.
- Flush log/metrics.
- Đóng connection sạch.
- Thoát trước khi hết grace period.

---

## 29. Câu hỏi kiểm tra cuối buổi

### Câu 1

`readinessProbe` fail thì Kubernetes sẽ làm gì?

A. Restart container ngay lập tức.  
B. Xóa Pod và tạo Pod mới.  
C. Đánh dấu Pod NotReady và loại Pod khỏi endpoint của Service.  
D. Rollback Deployment.

Đáp án đúng: C.

### Câu 2

`livenessProbe` fail quá `failureThreshold` thì điều gì xảy ra?

A. Container bị restart bởi kubelet.  
B. Deployment rollback.  
C. Service đổi ClusterIP.  
D. Node bị drain.

Đáp án đúng: A.

### Câu 3

`startupProbe` có tác dụng gì?

A. Scale Pod theo CPU.  
B. Cho ứng dụng khởi động chậm có thêm thời gian trước khi liveness/readiness được thực thi.  
C. Tạo Service endpoint.  
D. Tự động tạo Ingress.

Đáp án đúng: B.

### Câu 4

Deployment rollout mới được trigger khi nào?

A. Khi `.spec.template` thay đổi.  
B. Khi chỉ scale replicas từ 3 lên 5.  
C. Khi xem logs Pod.  
D. Khi tạo Service mới.

Đáp án đúng: A.

### Câu 5

Với Deployment `replicas: 3`, `maxSurge: 1`, `maxUnavailable: 1`, trong rolling update có thể có tối đa bao nhiêu Pod tồn tại cùng lúc?

A. 2  
B. 3  
C. 4  
D. 5

Đáp án đúng: C.

### Câu 6

Rollback Deployment bằng lệnh nào?

A. `kubectl delete deployment`  
B. `kubectl rollout undo deployment/<name>`  
C. `kubectl restart deployment/<name>`  
D. `kubectl apply rollback`

Đáp án đúng: B.

### Câu 7

Khi Pod thuộc Deployment bị xóa thủ công, thành phần nào đảm bảo Pod mới được tạo lại?

A. CoreDNS  
B. kube-proxy  
C. ReplicaSet/Deployment controller  
D. Ingress Controller

Đáp án đúng: C.

### Câu 8

Vì sao không nên cấu hình liveness probe kiểm tra database quá cứng nhắc?

A. Vì liveness probe không hỗ trợ HTTP.  
B. Vì database chậm tạm thời có thể làm app bị restart hàng loạt, gây cascading failure.  
C. Vì liveness chỉ chạy một lần duy nhất.  
D. Vì liveness chỉ dùng cho Job.

Đáp án đúng: B.

---

## 30. Dọn dẹp lab

Nếu muốn xóa toàn bộ tài nguyên của buổi 07:

```bash
kubectl delete namespace k8s-b07
```

Kiểm tra:

```bash
kubectl get ns k8s-b07
```

Nếu namespace chưa xóa xong ngay, chờ thêm một lúc:

```bash
kubectl get namespace k8s-b07 -w
```

---

## 31. Tổng kết buổi học

Trong buổi này,  đã học cách Kubernetes vận hành vòng đời ứng dụng thay vì chỉ tạo Pod cho chạy.

Các điểm quan trọng cần nhớ:

- Pod có lifecycle riêng và có thể bị thay thế bất kỳ lúc nào.
- Container có trạng thái `Waiting`, `Running`, `Terminated`.
- Pod có phase `Pending`, `Running`, `Succeeded`, `Failed`, `Unknown`.
- `startupProbe` dùng cho app khởi động chậm.
- `readinessProbe` quyết định Pod có được nhận traffic hay không.
- `livenessProbe` quyết định container có cần restart hay không.
- Readiness fail không restart container.
- Liveness fail có thể restart container.
- Deployment rolling update thay thế Pod cũ bằng Pod mới có kiểm soát.
- `maxSurge` cho phép tạo thêm Pod vượt desired replicas.
- `maxUnavailable` giới hạn số Pod unavailable trong update.
- Rollback giúp quay về revision ổn định.
- Self-healing là cơ chế đưa trạng thái thực tế về trạng thái mong muốn.
- Kubernetes không sửa được lỗi ứng dụng, nhưng giúp restart, replace, isolate và rollback tốt hơn.

Sau buổi này,  đã có nền tảng vững để đi tiếp sang buổi về resource management và scheduling cơ bản.

---

## 32. Tài liệu tham khảo chính thức

- Kubernetes v1.35 Documentation - Pod Lifecycle: https://v1-35.docs.kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/
- Kubernetes v1.35 Documentation - Liveness, Readiness, and Startup Probes: https://v1-35.docs.kubernetes.io/docs/concepts/configuration/liveness-readiness-startup-probes/
- Kubernetes v1.35 Documentation - Configure Liveness, Readiness and Startup Probes: https://v1-35.docs.kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/
- Kubernetes v1.35 Documentation - Deployments: https://v1-35.docs.kubernetes.io/docs/concepts/workloads/controllers/deployment/
- Kubernetes v1.35 Documentation - Kubernetes Self-Healing: https://v1-35.docs.kubernetes.io/docs/concepts/architecture/self-healing/

