# Buổi 10: Observability và Troubleshooting trong Kubernetes v1.35

## 1. Mục tiêu của buổi học

Ở các buổi trước, đã biết cách triển khai cụm Kubernetes bằng `kubeadm`, chạy workload, expose ứng dụng bằng Service/Ingress, gắn cấu hình bằng ConfigMap/Secret, dùng PVC/StorageClass, cấu hình probes, rolling update, rollback, resources, scheduling và security cơ bản.

Từ buổi này, trọng tâm chuyển sang năng lực vận hành:

- Quan sát trạng thái của cluster và ứng dụng.
- Đọc đúng triệu chứng lỗi.
- Biết bắt đầu debug từ đâu.
- Biết dùng đúng lệnh `kubectl` cho từng lớp lỗi.
- Biết phân biệt lỗi ở Pod, container image, config, resource, Service, DNS, Ingress, storage hoặc node.
- Biết cài và kiểm tra Metrics Server để sử dụng `kubectl top`.
- Biết xây dựng tư duy troubleshooting có hệ thống thay vì đoán mò.

Sau buổi học này, cần trả lời được các câu hỏi sau:

- Khi Pod bị `Pending`, cần kiểm tra gì trước?
- Khi Pod bị `ImagePullBackOff`, nguyên nhân thường nằm ở đâu?
- Khi Pod bị `CrashLoopBackOff`, đọc log như thế nào?
- Khi Pod bị `CreateContainerConfigError`, lỗi thường liên quan đến resource nào?
- Khi Service không truy cập được, làm sao kiểm tra selector và EndpointSlice?
- Khi Ingress trả về 404 hoặc 503, cần kiểm tra theo thứ tự nào?
- Metrics Server dùng để làm gì và không nên dùng để làm gì?
- Khác nhau giữa logs, events và metrics là gì?

---

## 2. Vị trí của buổi 10 trong lộ trình khóa học

Các khái niệm đã được mở khóa ở các buổi trước:

- Cluster, Node, Control Plane, Worker Node
- kubeadm, containerd, Calico CNI
- Pod, ReplicaSet, Deployment, DaemonSet, StatefulSet, Job, CronJob
- Service: ClusterIP, NodePort, Headless, ExternalName
- Ingress NGINX Controller
- ConfigMap, Secret
- Volume, PV, PVC, StorageClass, CSI NFS
- Helm, MetalLB
- Probes, rolling update, rollback, self-healing
- Resource requests/limits, QoS, scheduling cơ bản
- RBAC, ServiceAccount, SecurityContext, Pod Security Standards, NetworkPolicy

Buổi này không tập trung giới thiệu object mới phức tạp. Buổi này dùng lại toàn bộ kiến thức đã học để hình thành năng lực vận hành và xử lý sự cố.

---

## 3. Mô hình lab sử dụng trong buổi học

Cụm Kubernetes sử dụng trong khóa học:

```text
+-----------------------------+
|        control-plane        |
| master-01                   |
| Ubuntu 24.04                |
| kubeadm                     |
| containerd                  |
| kube-apiserver              |
| kube-scheduler              |
| kube-controller-manager     |
| etcd                        |
+--------------+--------------+
               |
               |
+--------------+--------------+
|                             |
| worker-01                  worker-02
| Ubuntu 24.04               Ubuntu 24.04
| kubelet                    kubelet
| kube-proxy                 kube-proxy
| containerd                 containerd
| Calico CNI                 Calico CNI
+-----------------------------+

+-----------------------------+
| nfs-01                      |
| Ubuntu 24.04                |
| NFS Server                  |
| Dùng cho các bài storage    |
+-----------------------------+
```

Các thành phần đã có từ các buổi trước:

- Kubernetes v1.35
- CNI: Calico
- Container runtime: containerd
- Ingress Controller: ingress-nginx
- MetalLB
- NFS CSI Driver
- StorageClass: `nfs-csi`

Trong buổi này, ta sẽ bổ sung:

- Metrics Server
- Một namespace riêng cho lab troubleshooting
- Một số workload cố tình lỗi để học cách phân tích

---

## 4. Observability là gì?

Observability là khả năng hiểu trạng thái bên trong của hệ thống dựa trên các tín hiệu mà hệ thống phát ra.

Trong Kubernetes, các tín hiệu quan trọng gồm:

- Logs
- Events
- Metrics
- Traces

Trong phạm vi buổi này, ta tập trung vào ba nhóm chính:

- Logs
- Events
- Metrics

Traces chỉ được giới thiệu ở mức khái niệm vì thường cần thêm hệ sinh thái như OpenTelemetry, Jaeger, Tempo hoặc APM platform. Phần đó phù hợp với các buổi observability nâng cao.

---

## 5. Phân biệt Logs, Events và Metrics

### 5.1. Logs

Logs là thông tin do ứng dụng hoặc container ghi ra.

Ví dụ:

```text
2026-04-24T10:00:01Z INFO Starting web server
2026-04-24T10:00:05Z ERROR Cannot connect to database
```

Trong Kubernetes, best practice là ứng dụng ghi log ra:

- `stdout`
- `stderr`

Sau đó kubelet và container runtime sẽ xử lý log ở cấp node. Người vận hành có thể xem log bằng:

```bash
kubectl logs
```

Ví dụ:

```bash
kubectl logs pod/web-abc123 -n demo
```

Nếu Pod có nhiều container:

```bash
kubectl logs pod/web-abc123 -n demo -c app
```

Nếu muốn xem log liên tục:

```bash
kubectl logs -f pod/web-abc123 -n demo
```

Nếu container đã crash và được restart, có thể xem log của lần chạy trước:

```bash
kubectl logs pod/web-abc123 -n demo --previous
```

### 5.2. Events

Events là các sự kiện Kubernetes ghi nhận về object.

Ví dụ:

- Pod được scheduled lên node nào
- Image pull thành công hay thất bại
- Container start thất bại
- PVC không bind được
- Probe fail
- Scheduler không tìm được node phù hợp

Xem event bằng:

```bash
kubectl get events -n demo --sort-by=.lastTimestamp
```

Hoặc xem event liên quan trực tiếp đến Pod bằng:

```bash
kubectl describe pod <pod-name> -n demo
```

Events thường trả lời câu hỏi:

```text
Kubernetes đã cố làm gì với object này?
Việc đó thành công hay thất bại?
Nếu thất bại thì Kubernetes ghi lý do gì?
```

### 5.3. Metrics

Metrics là số đo định lượng của hệ thống.

Ví dụ:

- CPU đang sử dụng
- Memory đang sử dụng
- Số lượng Pod
- Số request
- Latency
- Error rate

Trong bài này, ta dùng Metrics Server để xem CPU/memory cơ bản qua:

```bash
kubectl top nodes
kubectl top pods
```

Lưu ý quan trọng:

Metrics Server không phải là hệ thống monitoring đầy đủ kiểu Prometheus. Metrics Server chủ yếu phục vụ autoscaling pipeline và các lệnh như `kubectl top`.

---

## 6. Tư duy troubleshooting theo lớp

Khi gặp lỗi trong Kubernetes, không nên debug ngẫu nhiên. Nên đi theo lớp:

```text
User / Client
   |
   v
Ingress
   |
   v
Service
   |
   v
EndpointSlice
   |
   v
Pod
   |
   v
Container
   |
   v
Image / Config / Secret / Volume / Resource / Node
```

Khi một ứng dụng không truy cập được từ bên ngoài, thứ tự kiểm tra nên là:

```text
DNS / hosts file
Ingress rule
Ingress Controller
Service
EndpointSlice
Pod readiness
Container logs
Config / Secret / PVC
Node / CNI / NetworkPolicy
```

Khi một Pod không chạy được, thứ tự kiểm tra nên là:

```text
kubectl get pod
kubectl describe pod
kubectl logs
kubectl logs --previous
kubectl get events
kubectl get deployment/rs
kubectl top pod/node
kubectl describe node
```

---

## 7. Các lệnh quan sát cơ bản

### 7.1. Quan sát node

```bash
kubectl get nodes -o wide
```

Xem chi tiết một node:

```bash
kubectl describe node worker-01
```

Xem label của node:

```bash
kubectl get nodes --show-labels
```

Xem tài nguyên node nếu đã có Metrics Server:

```bash
kubectl top nodes
```

### 7.2. Quan sát namespace

```bash
kubectl get ns
```

Xem tất cả resource trong namespace:

```bash
kubectl get all -n <namespace>
```

### 7.3. Quan sát Pod

```bash
kubectl get pods -n <namespace>
```

Xem thêm node, Pod IP, trạng thái:

```bash
kubectl get pods -n <namespace> -o wide
```

Xem chi tiết Pod:

```bash
kubectl describe pod <pod-name> -n <namespace>
```

Xem YAML thực tế của Pod:

```bash
kubectl get pod <pod-name> -n <namespace> -o yaml
```

### 7.4. Quan sát Deployment và ReplicaSet

```bash
kubectl get deploy,rs,pod -n <namespace>
```

Xem rollout:

```bash
kubectl rollout status deployment/<deployment-name> -n <namespace>
```

Xem lịch sử rollout:

```bash
kubectl rollout history deployment/<deployment-name> -n <namespace>
```

### 7.5. Quan sát Service và EndpointSlice

```bash
kubectl get svc -n <namespace>
```

```bash
kubectl get endpointslice -n <namespace>
```

Xem chi tiết Service:

```bash
kubectl describe svc <service-name> -n <namespace>
```

### 7.6. Quan sát Ingress

```bash
kubectl get ingress -n <namespace>
```

```bash
kubectl describe ingress <ingress-name> -n <namespace>
```

Xem log Ingress Controller:

```bash
kubectl logs -n ingress-nginx deploy/ingress-nginx-controller
```

Nếu ingress-nginx-controller có nhiều Pod:

```bash
kubectl get pod -n ingress-nginx
kubectl logs -n ingress-nginx <ingress-nginx-pod-name>
```

### 7.7. Quan sát event toàn namespace

```bash
kubectl get events -n <namespace> --sort-by=.lastTimestamp
```

Xem event toàn cluster:

```bash
kubectl get events -A --sort-by=.lastTimestamp
```

---

## 8. Cài Metrics Server

### 8.1. Metrics Server dùng để làm gì?

Metrics Server thu thập CPU/memory resource metrics từ kubelet trên các node và expose qua Kubernetes Metrics API.

Sau khi có Metrics Server, có thể dùng:

```bash
kubectl top nodes
kubectl top pods
```

Metrics Server là nền tảng cần thiết cho các nội dung autoscaling ở buổi sau, đặc biệt là HPA.

### 8.2. Cài đặt Metrics Server bằng manifest chính thức

Trên node có quyền `kubectl`, chạy:

```bash
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```

Kiểm tra:

```bash
kubectl get deployment metrics-server -n kube-system
kubectl get pod -n kube-system -l k8s-app=metrics-server
```

Kiểm tra APIService:

```bash
kubectl get apiservice | grep metrics
```

Nếu hoạt động tốt, sau một lúc có thể chạy:

```bash
kubectl top nodes
kubectl top pods -A
```

### 8.3. Lỗi thường gặp trên kubeadm lab on-premise

Trong môi trường kubeadm lab, Metrics Server có thể gặp lỗi liên quan đến kubelet serving certificate.

Triệu chứng thường thấy:

```bash
kubectl top nodes
```

Kết quả:

```text
error: Metrics API not available
```

Kiểm tra log:

```bash
kubectl logs -n kube-system deploy/metrics-server
```

Có thể thấy lỗi dạng:

```text
x509: cannot validate certificate
```

Với môi trường lab, có thể patch thêm flag:

```bash
kubectl -n kube-system patch deployment metrics-server --type='json' -p='[
  {
    "op": "add",
    "path": "/spec/template/spec/containers/0/args/-",
    "value": "--kubelet-insecure-tls"
  },
  {
    "op": "add",
    "path": "/spec/template/spec/containers/0/args/-",
    "value": "--kubelet-preferred-address-types=InternalIP,Hostname,InternalDNS,ExternalDNS,ExternalIP"
  }
]'
```

Kiểm tra rollout:

```bash
kubectl rollout status deployment/metrics-server -n kube-system
```

Kiểm tra lại:

```bash
kubectl top nodes
kubectl top pods -A
```

Giải thích:

```yaml
--kubelet-insecure-tls
```

Cho phép Metrics Server bỏ qua việc xác thực CA của kubelet serving certificate.

```yaml
--kubelet-preferred-address-types=InternalIP,Hostname,InternalDNS,ExternalDNS,ExternalIP
```

Ưu tiên dùng địa chỉ `InternalIP` của node khi Metrics Server kết nối đến kubelet.

Lưu ý production:

- Không nên dùng `--kubelet-insecure-tls` trong production nếu có thể tránh.
- Nên cấu hình kubelet serving certificate chuẩn và được ký bởi CA phù hợp.
- Metrics Server không thay thế Prometheus/Grafana cho monitoring dài hạn.

---

## 9. Chuẩn bị namespace lab

Tạo namespace:

```bash
kubectl create namespace k8s-lab-observability
```

Kiểm tra:

```bash
kubectl get ns k8s-lab-observability
```

Từ đây về sau, các manifest lab sẽ dùng namespace:

```text
k8s-lab-observability
```

---

# Phần 1: Lab quan sát ứng dụng khỏe mạnh

## 10. Triển khai ứng dụng web bình thường

Tạo file:

```bash
vi 01-healthy-web.yaml
```

Nội dung:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: healthy-web
  namespace: k8s-lab-observability
  labels:
    app: healthy-web
spec:
  replicas: 2
  selector:
    matchLabels:
      app: healthy-web
  template:
    metadata:
      labels:
        app: healthy-web
    spec:
      containers:
        - name: nginx
          image: nginx:1.27
          ports:
            - containerPort: 80
          resources:
            requests:
              cpu: "50m"
              memory: "64Mi"
            limits:
              cpu: "200m"
              memory: "128Mi"
          readinessProbe:
            httpGet:
              path: /
              port: 80
            initialDelaySeconds: 3
            periodSeconds: 5
          livenessProbe:
            httpGet:
              path: /
              port: 80
            initialDelaySeconds: 10
            periodSeconds: 10
---
apiVersion: v1
kind: Service
metadata:
  name: healthy-web-svc
  namespace: k8s-lab-observability
spec:
  type: ClusterIP
  selector:
    app: healthy-web
  ports:
    - name: http
      port: 80
      targetPort: 80
```

Apply:

```bash
kubectl apply -f 01-healthy-web.yaml
```

Kiểm tra:

```bash
kubectl get deploy,rs,pod,svc,endpointslice -n k8s-lab-observability
```

Kỳ vọng:

```text
deployment.apps/healthy-web
replicaset.apps/healthy-web-xxxxx
pod/healthy-web-xxxxx
service/healthy-web-svc
endpointslice.discovery.k8s.io/healthy-web-svc-xxxxx
```

---

## 11. Giải thích manifest

### 11.1. Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
```

Khai báo workload dạng Deployment.

Deployment đã được học ở Buổi 03 và Buổi 07. Deployment quản lý ReplicaSet, ReplicaSet quản lý Pod.

```yaml
replicas: 2
```

Yêu cầu Kubernetes duy trì 2 Pod chạy ứng dụng.

```yaml
selector:
  matchLabels:
    app: healthy-web
```

Deployment dùng selector này để tìm các Pod thuộc về nó.

```yaml
template:
  metadata:
    labels:
      app: healthy-web
```

Pod được tạo ra sẽ có label `app=healthy-web`.

Label này rất quan trọng vì:

- Deployment dùng để quản lý Pod.
- Service dùng để chọn Pod.
- Troubleshooting Service thường bắt đầu từ selector và label.

### 11.2. Container

```yaml
image: nginx:1.27
```

Container chạy image `nginx:1.27`.

```yaml
ports:
  - containerPort: 80
```

Khai báo container listen port 80. Trường này chủ yếu mang ý nghĩa metadata và giúp manifest dễ hiểu.

### 11.3. Resource requests/limits

```yaml
resources:
  requests:
    cpu: "50m"
    memory: "64Mi"
  limits:
    cpu: "200m"
    memory: "128Mi"
```

`requests` là lượng tài nguyên scheduler dùng để tính toán node có đủ chỗ chạy Pod hay không.

`limits` là ngưỡng tối đa container được dùng.

Nếu container vượt memory limit, có thể bị OOMKilled.

### 11.4. Readiness Probe

```yaml
readinessProbe:
  httpGet:
    path: /
    port: 80
```

Readiness Probe cho biết Pod đã sẵn sàng nhận traffic hay chưa.

Nếu readiness fail, Pod vẫn có thể chạy nhưng sẽ bị loại khỏi endpoint của Service.

### 11.5. Liveness Probe

```yaml
livenessProbe:
  httpGet:
    path: /
    port: 80
```

Liveness Probe cho biết container còn sống khỏe hay không.

Nếu liveness fail liên tục, kubelet sẽ restart container.

### 11.6. Service

```yaml
kind: Service
```

Service cung cấp endpoint ổn định để truy cập các Pod.

```yaml
selector:
  app: healthy-web
```

Service chọn các Pod có label `app=healthy-web`.

```yaml
port: 80
targetPort: 80
```

`port` là port của Service.

`targetPort` là port trên Pod/container mà Service forward traffic đến.

---

## 12. Quan sát ứng dụng

Xem Pod:

```bash
kubectl get pod -n k8s-lab-observability -o wide
```

Xem chi tiết Pod:

```bash
POD=$(kubectl get pod -n k8s-lab-observability -l app=healthy-web -o jsonpath='{.items[0].metadata.name}')
kubectl describe pod $POD -n k8s-lab-observability
```

Xem logs:

```bash
kubectl logs $POD -n k8s-lab-observability
```

Xem Service:

```bash
kubectl get svc healthy-web-svc -n k8s-lab-observability
```

Xem EndpointSlice:

```bash
kubectl get endpointslice -n k8s-lab-observability -l kubernetes.io/service-name=healthy-web-svc
```

Test nội bộ bằng Pod tạm:

```bash
kubectl run curl-test \
  -n k8s-lab-observability \
  --image=curlimages/curl:8.10.1 \
  --rm -it --restart=Never \
  -- curl -I http://healthy-web-svc
```

Nếu thành công:

```text
HTTP/1.1 200 OK
```

---

# Phần 2: Debug lỗi ImagePullBackOff

## 13. Tạo workload lỗi image

Tạo file:

```bash
vi 02-image-pull-error.yaml
```

Nội dung:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: image-error
  namespace: k8s-lab-observability
  labels:
    app: image-error
spec:
  replicas: 1
  selector:
    matchLabels:
      app: image-error
  template:
    metadata:
      labels:
        app: image-error
    spec:
      containers:
        - name: app
          image: nginx:this-tag-does-not-exist
          ports:
            - containerPort: 80
```

Apply:

```bash
kubectl apply -f 02-image-pull-error.yaml
```

Kiểm tra:

```bash
kubectl get pod -n k8s-lab-observability -l app=image-error
```

Có thể thấy trạng thái:

```text
ImagePullBackOff
```

Hoặc:

```text
ErrImagePull
```

---

## 14. Debug ImagePullBackOff

Lấy Pod name:

```bash
POD=$(kubectl get pod -n k8s-lab-observability -l app=image-error -o jsonpath='{.items[0].metadata.name}')
```

Describe Pod:

```bash
kubectl describe pod $POD -n k8s-lab-observability
```

Xem phần Events ở cuối output.

Thông thường sẽ thấy lỗi dạng:

```text
Failed to pull image "nginx:this-tag-does-not-exist"
manifest unknown
Back-off pulling image
```

Xem event toàn namespace:

```bash
kubectl get events -n k8s-lab-observability --sort-by=.lastTimestamp
```

Kết luận:

```text
Pod đã được scheduler đặt lên node.
Kubelet trên node cố pull image.
Image/tag không tồn tại.
Container chưa start được.
Vì vậy kubectl logs chưa có dữ liệu ứng dụng.
```

Sửa lỗi:

```bash
kubectl set image deployment/image-error app=nginx:1.27 -n k8s-lab-observability
```

Kiểm tra rollout:

```bash
kubectl rollout status deployment/image-error -n k8s-lab-observability
```

Kiểm tra Pod:

```bash
kubectl get pod -n k8s-lab-observability -l app=image-error
```

---

## 15. Checklist lỗi ImagePullBackOff

Khi gặp `ImagePullBackOff`, cần kiểm tra:

- Tên image có đúng không?
- Tag có tồn tại không?
- Node có ra Internet hoặc registry được không?
- Registry private có cần imagePullSecret không?
- Secret pull image có đúng namespace không?
- DNS của node có resolve được registry không?
- Firewall/proxy có chặn không?
- Container runtime có pull được image thủ công bằng `crictl` không?

Lệnh hữu ích trên worker node:

```bash
sudo crictl pull nginx:1.27
```

Nếu node chưa có `crictl`, có thể cài từ package hoặc dùng công cụ phù hợp theo môi trường lab.

---

# Phần 3: Debug lỗi CrashLoopBackOff

## 16. Tạo workload crash liên tục

Tạo file:

```bash
vi 03-crashloop.yaml
```

Nội dung:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: crashloop-demo
  namespace: k8s-lab-observability
  labels:
    app: crashloop-demo
spec:
  replicas: 1
  selector:
    matchLabels:
      app: crashloop-demo
  template:
    metadata:
      labels:
        app: crashloop-demo
    spec:
      containers:
        - name: app
          image: busybox:1.36
          command:
            - /bin/sh
            - -c
            - |
              echo "Application is starting..."
              echo "Simulating fatal error..."
              exit 1
```

Apply:

```bash
kubectl apply -f 03-crashloop.yaml
```

Kiểm tra:

```bash
kubectl get pod -n k8s-lab-observability -l app=crashloop-demo
```

Sau một lúc có thể thấy:

```text
CrashLoopBackOff
```

---

## 17. Debug CrashLoopBackOff

Lấy Pod name:

```bash
POD=$(kubectl get pod -n k8s-lab-observability -l app=crashloop-demo -o jsonpath='{.items[0].metadata.name}')
```

Xem trạng thái:

```bash
kubectl describe pod $POD -n k8s-lab-observability
```

Xem log hiện tại:

```bash
kubectl logs $POD -n k8s-lab-observability
```

Xem log lần chạy trước:

```bash
kubectl logs $POD -n k8s-lab-observability --previous
```

Kết quả dự kiến:

```text
Application is starting...
Simulating fatal error...
```

Kết luận:

```text
Image pull thành công.
Container đã start.
Process chính trong container exit với mã lỗi.
Kubelet restart container theo restartPolicy của Pod do Deployment tạo ra.
Sau nhiều lần crash, Kubernetes hiển thị CrashLoopBackOff.
```

Sửa lỗi bằng cách đổi command để container chạy lâu:

```bash
kubectl patch deployment crashloop-demo \
  -n k8s-lab-observability \
  --type='json' \
  -p='[
    {
      "op": "replace",
      "path": "/spec/template/spec/containers/0/command",
      "value": ["/bin/sh", "-c", "echo Application fixed; sleep 3600"]
    }
  ]'
```

Kiểm tra:

```bash
kubectl rollout status deployment/crashloop-demo -n k8s-lab-observability
kubectl get pod -n k8s-lab-observability -l app=crashloop-demo
```

---

## 18. Checklist lỗi CrashLoopBackOff

Khi gặp `CrashLoopBackOff`, cần kiểm tra:

- `kubectl logs`
- `kubectl logs --previous`
- Exit code trong `kubectl describe pod`
- Biến môi trường có thiếu không?
- ConfigMap/Secret có đúng không?
- Ứng dụng có connect được database/API không?
- Liveness Probe có quá gắt không?
- App có cần thời gian startup dài hơn không?
- Memory có bị OOMKilled không?

Nếu thấy:

```text
Reason: OOMKilled
```

Thì cần kiểm tra memory limit và mức tiêu thụ memory của ứng dụng.

---

# Phần 4: Debug lỗi Pending do thiếu tài nguyên

## 19. Tạo Pod yêu cầu tài nguyên quá lớn

Tạo file:

```bash
vi 04-pending-resource.yaml
```

Nội dung:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pending-big-resource
  namespace: k8s-lab-observability
  labels:
    app: pending-big-resource
spec:
  containers:
    - name: app
      image: nginx:1.27
      resources:
        requests:
          cpu: "100"
          memory: "500Gi"
        limits:
          cpu: "100"
          memory: "500Gi"
```

Apply:

```bash
kubectl apply -f 04-pending-resource.yaml
```

Kiểm tra:

```bash
kubectl get pod pending-big-resource -n k8s-lab-observability
```

Kỳ vọng:

```text
Pending
```

---

## 20. Debug Pending

Describe Pod:

```bash
kubectl describe pod pending-big-resource -n k8s-lab-observability
```

Xem phần Events.

Có thể thấy:

```text
0/2 nodes are available: insufficient cpu, insufficient memory
```

Kết luận:

```text
Pod chưa được schedule lên node.
Container chưa start.
Không có logs ứng dụng.
Lỗi nằm ở scheduling/resource request.
```

Xem tài nguyên node:

```bash
kubectl describe nodes
```

Nếu có Metrics Server:

```bash
kubectl top nodes
```

Lưu ý:

- Scheduler dựa vào `requests`, không dựa vào mức usage hiện tại.
- Một node có thể đang dùng ít CPU thực tế nhưng vẫn không nhận Pod nếu tổng requests đã vượt allocatable.
- `kubectl top` cho biết usage hiện tại.
- `kubectl describe node` cho biết allocated resources theo requests/limits.

Sửa lỗi bằng cách giảm requests:

```bash
kubectl delete pod pending-big-resource -n k8s-lab-observability
```

Tạo lại Pod hợp lý:

```bash
cat <<'EOF' > 04-pending-resource-fixed.yaml
apiVersion: v1
kind: Pod
metadata:
  name: pending-big-resource-fixed
  namespace: k8s-lab-observability
spec:
  containers:
    - name: app
      image: nginx:1.27
      resources:
        requests:
          cpu: "50m"
          memory: "64Mi"
        limits:
          cpu: "200m"
          memory: "128Mi"
EOF

kubectl apply -f 04-pending-resource-fixed.yaml
```

Kiểm tra:

```bash
kubectl get pod pending-big-resource-fixed -n k8s-lab-observability -o wide
```

---

## 21. Checklist lỗi Pending

Khi Pod bị `Pending`, cần kiểm tra:

- Scheduler có báo thiếu CPU/memory không?
- Có nodeSelector sai không?
- Có nodeAffinity không match không?
- Có taint mà Pod không có toleration không?
- PVC có đang Pending không?
- StorageClass có tồn tại không?
- Node có Ready không?
- Có dùng hostPort gây conflict không?
- Có PodDisruption hoặc admission policy chặn không?

Lệnh quan trọng nhất:

```bash
kubectl describe pod <pod-name> -n <namespace>
```

---

# Phần 5: Debug lỗi CreateContainerConfigError

## 22. Tạo Pod tham chiếu Secret không tồn tại

Tạo file:

```bash
vi 05-missing-secret.yaml
```

Nội dung:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: missing-secret-demo
  namespace: k8s-lab-observability
  labels:
    app: missing-secret-demo
spec:
  containers:
    - name: app
      image: busybox:1.36
      command:
        - /bin/sh
        - -c
        - |
          echo "DB_PASSWORD=$DB_PASSWORD"
          sleep 3600
      env:
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: app-db-secret
              key: db-password
```

Apply:

```bash
kubectl apply -f 05-missing-secret.yaml
```

Kiểm tra:

```bash
kubectl get pod missing-secret-demo -n k8s-lab-observability
```

Có thể thấy:

```text
CreateContainerConfigError
```

---

## 23. Debug CreateContainerConfigError

Describe Pod:

```bash
kubectl describe pod missing-secret-demo -n k8s-lab-observability
```

Xem phần Events.

Có thể thấy:

```text
Error: secret "app-db-secret" not found
```

Kết luận:

```text
Image đã có thể pull.
Pod đã có thể được schedule.
Nhưng kubelet không thể tạo container config vì manifest tham chiếu Secret không tồn tại.
```

Kiểm tra Secret:

```bash
kubectl get secret -n k8s-lab-observability
```

Tạo Secret còn thiếu:

```bash
kubectl create secret generic app-db-secret \
  -n k8s-lab-observability \
  --from-literal=db-password='P@ssw0rd123'
```

Xóa Pod để kubelet tạo lại:

```bash
kubectl delete pod missing-secret-demo -n k8s-lab-observability
kubectl apply -f 05-missing-secret.yaml
```

Kiểm tra:

```bash
kubectl get pod missing-secret-demo -n k8s-lab-observability
kubectl logs missing-secret-demo -n k8s-lab-observability
```

---

## 24. Checklist lỗi CreateContainerConfigError

Khi gặp `CreateContainerConfigError`, cần kiểm tra:

- ConfigMap có tồn tại không?
- Secret có tồn tại không?
- Key trong ConfigMap/Secret có đúng không?
- Volume mount có tham chiếu đúng tên volume không?
- ServiceAccount có tồn tại không?
- ImagePullSecret có đúng namespace không?
- Field trong manifest có sai cấu trúc không?

Lệnh quan trọng:

```bash
kubectl describe pod <pod-name> -n <namespace>
```

---

# Phần 6: Debug Service không có Endpoint

## 25. Tạo Service selector sai

Tạo file:

```bash
vi 06-service-no-endpoint.yaml
```

Nội dung:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: svc-debug-web
  namespace: k8s-lab-observability
  labels:
    app: svc-debug-web
spec:
  replicas: 2
  selector:
    matchLabels:
      app: svc-debug-web
  template:
    metadata:
      labels:
        app: svc-debug-web
    spec:
      containers:
        - name: nginx
          image: nginx:1.27
          ports:
            - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: svc-debug-web-svc
  namespace: k8s-lab-observability
spec:
  type: ClusterIP
  selector:
    app: wrong-label
  ports:
    - name: http
      port: 80
      targetPort: 80
```

Apply:

```bash
kubectl apply -f 06-service-no-endpoint.yaml
```

Kiểm tra:

```bash
kubectl get pod -n k8s-lab-observability -l app=svc-debug-web
kubectl get svc svc-debug-web-svc -n k8s-lab-observability
kubectl get endpointslice -n k8s-lab-observability -l kubernetes.io/service-name=svc-debug-web-svc
```

Test:

```bash
kubectl run curl-test \
  -n k8s-lab-observability \
  --image=curlimages/curl:8.10.1 \
  --rm -it --restart=Never \
  -- curl -I --max-time 5 http://svc-debug-web-svc
```

Có thể lỗi timeout hoặc connection failure.

---

## 26. Debug Service selector

Xem Service:

```bash
kubectl describe svc svc-debug-web-svc -n k8s-lab-observability
```

Xem selector:

```bash
kubectl get svc svc-debug-web-svc -n k8s-lab-observability -o yaml
```

Xem label của Pod:

```bash
kubectl get pod -n k8s-lab-observability --show-labels
```

Kết luận:

```text
Service selector là app=wrong-label.
Pod label là app=svc-debug-web.
Service không chọn được Pod nào.
Vì vậy EndpointSlice không có endpoint backend.
```

Sửa Service selector:

```bash
kubectl patch svc svc-debug-web-svc \
  -n k8s-lab-observability \
  -p '{"spec":{"selector":{"app":"svc-debug-web"}}}'
```

Kiểm tra EndpointSlice:

```bash
kubectl get endpointslice -n k8s-lab-observability -l kubernetes.io/service-name=svc-debug-web-svc
```

Test lại:

```bash
kubectl run curl-test \
  -n k8s-lab-observability \
  --image=curlimages/curl:8.10.1 \
  --rm -it --restart=Never \
  -- curl -I http://svc-debug-web-svc
```

---

## 27. Checklist debug Service

Khi Service không truy cập được:

- Service có tồn tại không?
- Service selector có match label Pod không?
- Pod có Running không?
- Pod có Ready không?
- EndpointSlice có endpoint không?
- `targetPort` có đúng với port app đang listen không?
- NetworkPolicy có chặn không?
- kube-proxy có chạy không?
- CoreDNS có resolve được service name không?

Lệnh quan trọng:

```bash
kubectl describe svc <service-name> -n <namespace>
kubectl get endpointslice -n <namespace>
kubectl get pod -n <namespace> --show-labels
```

---

# Phần 7: Debug DNS trong Kubernetes

## 28. Kiểm tra CoreDNS

CoreDNS chạy trong namespace `kube-system`:

```bash
kubectl get deploy coredns -n kube-system
kubectl get pod -n kube-system -l k8s-app=kube-dns
```

Xem Service kube-dns:

```bash
kubectl get svc kube-dns -n kube-system
```

Xem log CoreDNS:

```bash
kubectl logs -n kube-system deployment/coredns
```

Nếu có nhiều container hoặc nhiều Pod:

```bash
kubectl get pod -n kube-system -l k8s-app=kube-dns
kubectl logs -n kube-system <coredns-pod-name>
```

---

## 29. Dùng Pod tạm để test DNS

Chạy Pod tạm:

```bash
kubectl run dns-test \
  -n k8s-lab-observability \
  --image=busybox:1.36 \
  --rm -it --restart=Never \
  -- nslookup healthy-web-svc
```

Test FQDN đầy đủ:

```bash
kubectl run dns-test \
  -n k8s-lab-observability \
  --image=busybox:1.36 \
  --rm -it --restart=Never \
  -- nslookup healthy-web-svc.k8s-lab-observability.svc.cluster.local
```

Test từ Pod có sẵn:

```bash
kubectl exec -n k8s-lab-observability deploy/healthy-web -- cat /etc/resolv.conf
```

Kết quả thường có dạng:

```text
search k8s-lab-observability.svc.cluster.local svc.cluster.local cluster.local
nameserver <cluster-dns-ip>
options ndots:5
```

Giải thích:

- `nameserver` thường là ClusterIP của Service `kube-dns`.
- `search` giúp Pod resolve tên ngắn trong namespace.
- Service `healthy-web-svc` trong cùng namespace có thể gọi bằng tên ngắn.
- Service khác namespace nên gọi bằng FQDN hoặc `<service>.<namespace>`.

---

## 30. Checklist debug DNS

Khi DNS trong Pod lỗi:

- CoreDNS Pod có Running không?
- Service `kube-dns` có tồn tại không?
- Pod có `/etc/resolv.conf` đúng không?
- NetworkPolicy có chặn egress tới DNS không?
- Calico có hoạt động không?
- Node có vấn đề network không?
- Service name có đúng namespace không?
- Có gọi nhầm short name giữa hai namespace không?

Lệnh quan trọng:

```bash
kubectl get pod -n kube-system -l k8s-app=kube-dns
kubectl logs -n kube-system deployment/coredns
kubectl exec <pod> -n <namespace> -- cat /etc/resolv.conf
```

---

# Phần 8: Debug Ingress HTTP/HTTPS

## 31. Tạo Ingress cho healthy-web

Giả sử từ Buổi 06 đã có:

- MetalLB
- Ingress NGINX Controller
- DNS hoặc file hosts trỏ domain về IP LoadBalancer của Ingress Controller

Kiểm tra IP Ingress Controller:

```bash
kubectl get svc -n ingress-nginx
```

Ví dụ:

```text
ingress-nginx-controller   LoadBalancer   10.96.x.x   192.168.56.240   80:xxxxx/TCP,443:xxxxx/TCP
```

Giả sử domain lab:

```text
healthy.k8s.local
```

Trên máy client hoặc máy admin, thêm vào `/etc/hosts`:

```text
192.168.56.240 healthy.k8s.local
```

Tạo file:

```bash
vi 07-ingress-healthy-web.yaml
```

Nội dung:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: healthy-web-ingress
  namespace: k8s-lab-observability
spec:
  ingressClassName: nginx
  rules:
    - host: healthy.k8s.local
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: healthy-web-svc
                port:
                  number: 80
```

Apply:

```bash
kubectl apply -f 07-ingress-healthy-web.yaml
```

Kiểm tra:

```bash
kubectl get ingress -n k8s-lab-observability
kubectl describe ingress healthy-web-ingress -n k8s-lab-observability
```

Test:

```bash
curl -I http://healthy.k8s.local
```

---

## 32. Debug Ingress 404

Nếu truy cập Ingress bị 404, thường kiểm tra:

```bash
kubectl get ingress -n k8s-lab-observability
kubectl describe ingress healthy-web-ingress -n k8s-lab-observability
```

Kiểm tra domain:

```bash
curl -v http://healthy.k8s.local
```

Nếu dùng IP trực tiếp:

```bash
curl -v http://192.168.56.240
```

Có thể bị default backend hoặc 404 vì request không có Host header đúng.

Test với Host header:

```bash
curl -I -H "Host: healthy.k8s.local" http://192.168.56.240
```

Kết luận:

```text
Ingress routing HTTP dựa vào host/path.
Nếu request không có Host header đúng rule, có thể trả 404.
```

---

## 33. Debug Ingress 503

Nếu Ingress trả 503, thường backend Service không có endpoint hoặc Pod không Ready.

Kiểm tra:

```bash
kubectl describe ingress healthy-web-ingress -n k8s-lab-observability
kubectl get svc healthy-web-svc -n k8s-lab-observability
kubectl get endpointslice -n k8s-lab-observability -l kubernetes.io/service-name=healthy-web-svc
kubectl get pod -n k8s-lab-observability -l app=healthy-web
```

Xem log Ingress Controller:

```bash
kubectl logs -n ingress-nginx deploy/ingress-nginx-controller
```

Checklist Ingress 503:

- Ingress backend service name có đúng không?
- Service port có đúng không?
- Service có EndpointSlice không?
- Pod có Ready không?
- Readiness probe có fail không?
- NetworkPolicy có chặn Ingress Controller tới backend không?

---

## 34. Tạo HTTPS Ingress bằng TLS Secret self-signed

Tạo certificate self-signed cho lab:

```bash
openssl req -x509 -nodes -days 365 \
  -newkey rsa:2048 \
  -keyout healthy.k8s.local.key \
  -out healthy.k8s.local.crt \
  -subj "/CN=healthy.k8s.local/O=K8S Lab"
```

Tạo TLS Secret:

```bash
kubectl create secret tls healthy-web-tls \
  -n k8s-lab-observability \
  --cert=healthy.k8s.local.crt \
  --key=healthy.k8s.local.key
```

Tạo file:

```bash
vi 08-ingress-healthy-web-https.yaml
```

Nội dung:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: healthy-web-ingress
  namespace: k8s-lab-observability
spec:
  ingressClassName: nginx
  tls:
    - hosts:
        - healthy.k8s.local
      secretName: healthy-web-tls
  rules:
    - host: healthy.k8s.local
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: healthy-web-svc
                port:
                  number: 80
```

Apply:

```bash
kubectl apply -f 08-ingress-healthy-web-https.yaml
```

Test:

```bash
curl -k -I https://healthy.k8s.local
```

Dùng `-k` vì certificate self-signed chưa được client trust.

---

## 35. Checklist debug HTTPS Ingress

Khi HTTPS lỗi:

- Secret TLS có đúng namespace với Ingress không?
- Secret type có phải `kubernetes.io/tls` không?
- Secret có đủ `tls.crt` và `tls.key` không?
- Host trong Ingress có khớp với domain truy cập không?
- Certificate CN/SAN có khớp domain không?
- Client có trust CA không?
- Ingress Controller có nhận config mới không?
- Port 443 của Ingress Controller có expose ra ngoài không?

Lệnh kiểm tra Secret:

```bash
kubectl get secret healthy-web-tls -n k8s-lab-observability -o yaml
```

Kiểm tra certificate từ client:

```bash
openssl s_client -connect healthy.k8s.local:443 -servername healthy.k8s.local
```

---

# Phần 9: Debug bằng kubectl exec và ephemeral debug container

## 36. Sử dụng kubectl exec

`kubectl exec` dùng để chạy lệnh bên trong container đang chạy.

Ví dụ:

```bash
kubectl exec -n k8s-lab-observability deploy/healthy-web -- nginx -v
```

Vào shell:

```bash
kubectl exec -n k8s-lab-observability -it deploy/healthy-web -- /bin/bash
```

Nếu image không có bash:

```bash
kubectl exec -n k8s-lab-observability -it deploy/healthy-web -- /bin/sh
```

Kiểm tra file trong container:

```bash
kubectl exec -n k8s-lab-observability deploy/healthy-web -- ls -la /usr/share/nginx/html
```

Lưu ý:

- `exec` chỉ dùng được khi container đang Running.
- Nếu container crash quá nhanh, nên dùng `kubectl logs --previous`.
- Không nên sửa thủ công dữ liệu trong container production rồi xem đó là giải pháp lâu dài.
- Mọi thay đổi phải được đưa về manifest, image hoặc config chuẩn.

---

## 37. Debug Pod có image tối giản

Nhiều image production rất tối giản, không có:

- `sh`
- `bash`
- `curl`
- `nslookup`
- `ping`
- `ps`
- `netstat`

Khi đó có thể dùng Pod debug tạm:

```bash
kubectl run net-debug \
  -n k8s-lab-observability \
  --image=nicolaka/netshoot \
  --rm -it --restart=Never \
  -- /bin/bash
```

Bên trong Pod debug:

```bash
curl -I http://healthy-web-svc
nslookup healthy-web-svc
dig healthy-web-svc.k8s-lab-observability.svc.cluster.local
```

Nếu không muốn dùng `nicolaka/netshoot`, có thể dùng image nhẹ hơn:

```bash
kubectl run curl-test \
  -n k8s-lab-observability \
  --image=curlimages/curl:8.10.1 \
  --rm -it --restart=Never \
  -- sh
```

Lưu ý:

- Với môi trường bị hạn chế Internet, cần mirror image về private registry.
- Nếu NetworkPolicy đang chặn egress/ingress, Pod debug cũng có thể bị chặn theo policy.

---

## 38. Giới thiệu kubectl debug

`kubectl debug` hỗ trợ tạo ephemeral container để debug Pod đang chạy, đặc biệt hữu ích khi container chính thiếu công cụ debug.

Ví dụ:

```bash
POD=$(kubectl get pod -n k8s-lab-observability -l app=healthy-web -o jsonpath='{.items[0].metadata.name}')

kubectl debug -n k8s-lab-observability -it $POD \
  --image=nicolaka/netshoot \
  --target=nginx \
  -- /bin/bash
```

Giải thích:

```bash
kubectl debug
```

Tạo môi trường debug gắn với Pod đang có.

```bash
--image=nicolaka/netshoot
```

Dùng image có nhiều công cụ network troubleshooting.

```bash
--target=nginx
```

Chỉ định container mục tiêu trong Pod.

Lưu ý:

- Ephemeral container là công cụ debug, không phải cách triển khai app.
- Không nên lạm dụng trong production nếu không có quy trình kiểm soát.
- RBAC nên giới hạn quyền sử dụng `pods/ephemeralcontainers`.

---

# Phần 10: Debug lỗi liên quan đến PVC và Storage

## 39. Kiểm tra PVC Pending

Từ Buổi 05, đã biết PVC/StorageClass/NFS CSI.

Khi PVC bị Pending:

```bash
kubectl get pvc -n <namespace>
```

Describe PVC:

```bash
kubectl describe pvc <pvc-name> -n <namespace>
```

Kiểm tra StorageClass:

```bash
kubectl get storageclass
kubectl describe storageclass nfs-csi
```

Kiểm tra CSI driver:

```bash
kubectl get pod -A | grep -i nfs
kubectl get csidriver
```

Kiểm tra event:

```bash
kubectl get events -n <namespace> --sort-by=.lastTimestamp
```

Checklist PVC Pending:

- StorageClass có tồn tại không?
- `storageClassName` trong PVC có đúng không?
- CSI driver có Running không?
- NFS Server có reachable không?
- NFS export có đúng không?
- Worker node có package `nfs-common` không?
- NetworkPolicy/firewall có chặn không?
- Access mode có phù hợp không?

---

## 40. Debug Pod stuck ContainerCreating do volume mount

Nếu Pod bị:

```text
ContainerCreating
```

Quá lâu, cần kiểm tra:

```bash
kubectl describe pod <pod-name> -n <namespace>
```

Nếu lỗi volume mount, Events có thể báo:

```text
MountVolume.SetUp failed
```

Hoặc:

```text
Unable to attach or mount volumes
```

Với NFS, vào worker node kiểm tra:

```bash
showmount -e <nfs-server-ip>
```

Kiểm tra mount thủ công:

```bash
sudo mkdir -p /mnt/test-nfs
sudo mount -t nfs <nfs-server-ip>:/srv/nfs/k8s /mnt/test-nfs
df -h | grep test-nfs
sudo umount /mnt/test-nfs
```

Nếu mount thủ công cũng lỗi, vấn đề nằm ngoài Kubernetes:

- NFS Server
- Firewall
- Export permission
- Network routing
- Package client

---

# Phần 11: Debug lỗi RBAC và permission

## 41. Triệu chứng lỗi RBAC

Một số lỗi thường gặp:

```text
Error from server (Forbidden)
```

Ví dụ:

```text
pods is forbidden: User "system:serviceaccount:demo:app-sa" cannot list resource "pods"
```

Lệnh kiểm tra quyền:

```bash
kubectl auth can-i list pods -n k8s-lab-observability
```

Kiểm tra với ServiceAccount cụ thể:

```bash
kubectl auth can-i list pods \
  -n k8s-lab-observability \
  --as=system:serviceaccount:k8s-lab-observability:default
```

Kiểm tra RoleBinding:

```bash
kubectl get role,rolebinding -n k8s-lab-observability
```

Describe RoleBinding:

```bash
kubectl describe rolebinding <rolebinding-name> -n k8s-lab-observability
```

Checklist RBAC:

- ServiceAccount có đúng namespace không?
- Role có đúng verbs không?
- Role có đúng resources không?
- RoleBinding có bind đúng subject không?
- Dùng Role hay ClusterRole có phù hợp không?
- App có đang dùng đúng ServiceAccount không?

---

# Phần 12: Debug node và system components

## 42. Kiểm tra system pods

```bash
kubectl get pod -n kube-system -o wide
```

Các Pod quan trọng:

- CoreDNS
- kube-proxy
- Calico
- Metrics Server
- kube-apiserver
- kube-controller-manager
- kube-scheduler
- etcd

Với kubeadm, các control plane components thường chạy dạng static Pod trên master node.

Kiểm tra static pod manifest trên master:

```bash
sudo ls -l /etc/kubernetes/manifests/
```

Thường có:

```text
etcd.yaml
kube-apiserver.yaml
kube-controller-manager.yaml
kube-scheduler.yaml
```

Xem log control plane component qua kubectl:

```bash
kubectl logs -n kube-system kube-apiserver-master-01
kubectl logs -n kube-system kube-scheduler-master-01
kubectl logs -n kube-system kube-controller-manager-master-01
kubectl logs -n kube-system etcd-master-01
```

Tên Pod thực tế có thể khác tùy hostname.

---

## 43. Kiểm tra kubelet trên node

Trên node cần kiểm tra:

```bash
sudo systemctl status kubelet
```

Xem log kubelet:

```bash
sudo journalctl -u kubelet -f
```

Xem log containerd:

```bash
sudo systemctl status containerd
sudo journalctl -u containerd -f
```

Kiểm tra disk:

```bash
df -h
```

Kiểm tra memory:

```bash
free -h
```

Kiểm tra process:

```bash
top
```

Kiểm tra network interface:

```bash
ip a
ip route
```

Kiểm tra DNS node:

```bash
resolvectl status
```

Hoặc:

```bash
cat /etc/resolv.conf
```

---

## 44. Node NotReady

Khi node `NotReady`:

```bash
kubectl get nodes
kubectl describe node <node-name>
```

Kiểm tra trên node:

```bash
sudo systemctl status kubelet
sudo journalctl -u kubelet -n 100 --no-pager
sudo systemctl status containerd
sudo crictl ps
sudo crictl images
```

Checklist Node NotReady:

- kubelet có chạy không?
- containerd có chạy không?
- Node có hết disk không?
- Node có hết memory không?
- CNI có lỗi không?
- Calico Pod trên node đó có Running không?
- Node có đổi IP/hostname không?
- API Server có reachable từ node không?
- Time/NTP có lệch quá nhiều không?
- Certificate kubelet có hết hạn không?

---

# Phần 13: Bảng triệu chứng lỗi thường gặp

## 45. Bảng lỗi Pod phổ biến

| Triệu chứng | Ý nghĩa thường gặp | Lệnh kiểm tra chính |
|---|---|---|
| `Pending` | Chưa schedule được hoặc PVC chưa bind | `kubectl describe pod` |
| `ContainerCreating` | Đang tạo container hoặc mount volume | `kubectl describe pod` |
| `ImagePullBackOff` | Pull image thất bại | `kubectl describe pod` |
| `ErrImagePull` | Lỗi pull image lần đầu | `kubectl describe pod` |
| `CrashLoopBackOff` | Container start rồi crash lặp lại | `kubectl logs --previous` |
| `CreateContainerConfigError` | Sai config trước khi tạo container | `kubectl describe pod` |
| `RunContainerError` | Runtime không chạy được container | `kubectl describe pod`, kubelet logs |
| `OOMKilled` | Container vượt memory limit | `kubectl describe pod`, `kubectl top pod` |
| `Completed` | Pod chạy xong, thường gặp với Job | `kubectl logs` |
| `Terminating` lâu | Finalizer, volume detach, preStop, grace period | `kubectl describe pod` |

---

## 46. Bảng lỗi Service/Ingress phổ biến

| Triệu chứng | Nguyên nhân thường gặp | Lệnh kiểm tra |
|---|---|---|
| Service timeout | Không có endpoint, targetPort sai, NetworkPolicy chặn | `kubectl get endpointslice` |
| Service resolve DNS fail | CoreDNS lỗi, gọi sai namespace | `nslookup`, CoreDNS logs |
| Ingress 404 | Host/path không match rule | `kubectl describe ingress` |
| Ingress 503 | Backend Service không có endpoint | `kubectl get endpointslice` |
| HTTPS warning | Self-signed cert hoặc cert không trusted | `openssl s_client` |
| HTTPS sai cert | Secret TLS sai hoặc host mismatch | `kubectl get secret`, `describe ingress` |
| NodePort không vào được | Firewall node, sai port, Pod chưa Ready | `kubectl get svc`, node firewall |

---

# Phần 14: Playbook troubleshooting theo tình huống

## 47. Tình huống 1: Người dùng báo web không truy cập được qua domain

Thứ tự kiểm tra:

```bash
nslookup healthy.k8s.local
```

```bash
curl -v http://healthy.k8s.local
```

```bash
kubectl get ingress -n k8s-lab-observability
kubectl describe ingress healthy-web-ingress -n k8s-lab-observability
```

```bash
kubectl get svc healthy-web-svc -n k8s-lab-observability
kubectl get endpointslice -n k8s-lab-observability -l kubernetes.io/service-name=healthy-web-svc
```

```bash
kubectl get pod -n k8s-lab-observability -l app=healthy-web -o wide
```

```bash
kubectl logs -n ingress-nginx deploy/ingress-nginx-controller
```

Kết luận theo từng trường hợp:

- DNS không resolve: lỗi DNS ngoài cluster hoặc hosts file.
- Ingress không có rule: lỗi manifest Ingress.
- Service không có endpoint: lỗi selector hoặc Pod NotReady.
- Pod không Ready: kiểm tra readinessProbe/logs.
- Ingress Controller không chạy: kiểm tra namespace `ingress-nginx`.

---

## 48. Tình huống 2: Deployment rollout xong nhưng user báo lỗi 503

Kiểm tra rollout:

```bash
kubectl rollout status deployment/healthy-web -n k8s-lab-observability
kubectl rollout history deployment/healthy-web -n k8s-lab-observability
```

Kiểm tra Pod:

```bash
kubectl get pod -n k8s-lab-observability -l app=healthy-web
```

Kiểm tra readiness:

```bash
POD=$(kubectl get pod -n k8s-lab-observability -l app=healthy-web -o jsonpath='{.items[0].metadata.name}')
kubectl describe pod $POD -n k8s-lab-observability
```

Kiểm tra EndpointSlice:

```bash
kubectl get endpointslice -n k8s-lab-observability -l kubernetes.io/service-name=healthy-web-svc -o yaml
```

Nếu Pod Running nhưng không Ready, Service sẽ không đưa Pod vào endpoint nhận traffic.

Rollback nếu cần:

```bash
kubectl rollout undo deployment/healthy-web -n k8s-lab-observability
```

---

## 49. Tình huống 3: Pod bị OOMKilled

Kiểm tra Pod:

```bash
kubectl get pod -n <namespace>
kubectl describe pod <pod-name> -n <namespace>
```

Tìm:

```text
Reason: OOMKilled
Exit Code: 137
```

Kiểm tra metrics:

```bash
kubectl top pod -n <namespace>
```

Kiểm tra resource limit:

```bash
kubectl get pod <pod-name> -n <namespace> -o yaml | grep -A10 resources:
```

Hướng xử lý:

- Tăng memory limit nếu app thật sự cần thêm memory.
- Tối ưu memory usage của app.
- Kiểm tra memory leak.
- Đặt requests/limits hợp lý.
- Không đặt limit quá thấp theo cảm tính.

---

## 50. Tình huống 4: Pod không ra Internet được

Kiểm tra từ Pod debug:

```bash
kubectl run net-debug \
  -n k8s-lab-observability \
  --image=nicolaka/netshoot \
  --rm -it --restart=Never \
  -- /bin/bash
```

Trong Pod:

```bash
ip route
nslookup google.com
curl -I https://example.com
```

Kiểm tra CoreDNS:

```bash
kubectl logs -n kube-system deployment/coredns
```

Kiểm tra NetworkPolicy:

```bash
kubectl get networkpolicy -A
```

Kiểm tra Calico:

```bash
kubectl get pod -n calico-system
```

Hoặc nếu Calico được cài ở namespace khác:

```bash
kubectl get pod -A | grep -i calico
```

Kiểm tra node NAT/routing:

```bash
ip route
sudo iptables -t nat -L -n -v
```

Lưu ý:

- Trên kubeadm on-premise, egress Internet phụ thuộc node network, CNI và NAT/routing.
- Nếu NetworkPolicy default deny egress được áp dụng, Pod có thể không ra Internet dù node ra Internet bình thường.

---

# Phần 15: Lab tổng hợp cuối buổi

## 51. Mục tiêu lab tổng hợp

Học viên sẽ xử lý một chuỗi lỗi theo thứ tự:

- ImagePullBackOff
- CrashLoopBackOff
- Service không có endpoint
- Ingress 404
- Ingress 503
- Metrics Server không hoạt động
- PVC/volume mount lỗi ở mức phân tích

Giảng viên có thể chia lớp thành nhóm. Mỗi nhóm nhận một lỗi, phải trình bày:

- Triệu chứng
- Lệnh đã dùng
- Output quan trọng
- Nguyên nhân gốc
- Cách sửa
- Bài học vận hành

---

## 52. Tạo app tổng hợp có lỗi

Tạo file:

```bash
vi 09-final-broken-app.yaml
```

Nội dung:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: final-broken-web
  namespace: k8s-lab-observability
  labels:
    app: final-broken-web
spec:
  replicas: 2
  selector:
    matchLabels:
      app: final-broken-web
  template:
    metadata:
      labels:
        app: final-broken-web
    spec:
      containers:
        - name: web
          image: nginx:wrong-version
          ports:
            - containerPort: 80
          resources:
            requests:
              cpu: "50m"
              memory: "64Mi"
            limits:
              cpu: "200m"
              memory: "128Mi"
---
apiVersion: v1
kind: Service
metadata:
  name: final-broken-web-svc
  namespace: k8s-lab-observability
spec:
  type: ClusterIP
  selector:
    app: final-wrong-label
  ports:
    - name: http
      port: 80
      targetPort: 8080
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: final-broken-web-ingress
  namespace: k8s-lab-observability
spec:
  ingressClassName: nginx
  rules:
    - host: final.k8s.local
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: final-broken-web-svc
                port:
                  number: 80
```

Apply:

```bash
kubectl apply -f 09-final-broken-app.yaml
```

Yêu cầu sửa theo thứ tự:

```bash
kubectl get pod -n k8s-lab-observability -l app=final-broken-web
kubectl describe pod <pod-name> -n k8s-lab-observability
```

Sửa image:

```bash
kubectl set image deployment/final-broken-web web=nginx:1.27 -n k8s-lab-observability
```

Kiểm tra rollout:

```bash
kubectl rollout status deployment/final-broken-web -n k8s-lab-observability
```

Kiểm tra Service endpoint:

```bash
kubectl get endpointslice -n k8s-lab-observability -l kubernetes.io/service-name=final-broken-web-svc
kubectl get pod -n k8s-lab-observability --show-labels
```

Sửa selector:

```bash
kubectl patch svc final-broken-web-svc \
  -n k8s-lab-observability \
  -p '{"spec":{"selector":{"app":"final-broken-web"}}}'
```

Kiểm tra targetPort:

```bash
kubectl get svc final-broken-web-svc -n k8s-lab-observability -o yaml
```

Sửa targetPort:

```bash
kubectl patch svc final-broken-web-svc \
  -n k8s-lab-observability \
  --type='json' \
  -p='[
    {
      "op": "replace",
      "path": "/spec/ports/0/targetPort",
      "value": 80
    }
  ]'
```

Test nội bộ:

```bash
kubectl run curl-test \
  -n k8s-lab-observability \
  --image=curlimages/curl:8.10.1 \
  --rm -it --restart=Never \
  -- curl -I http://final-broken-web-svc
```

Test Ingress:

```bash
curl -I -H "Host: final.k8s.local" http://<INGRESS_LOADBALANCER_IP>
```

Nếu có DNS/hosts:

```bash
curl -I http://final.k8s.local
```

---

## 53. Phiên bản manifest đã sửa hoàn chỉnh

Tạo file:

```bash
vi 10-final-fixed-app.yaml
```

Nội dung:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: final-fixed-web
  namespace: k8s-lab-observability
  labels:
    app: final-fixed-web
spec:
  replicas: 2
  selector:
    matchLabels:
      app: final-fixed-web
  template:
    metadata:
      labels:
        app: final-fixed-web
    spec:
      containers:
        - name: web
          image: nginx:1.27
          ports:
            - containerPort: 80
          resources:
            requests:
              cpu: "50m"
              memory: "64Mi"
            limits:
              cpu: "200m"
              memory: "128Mi"
          readinessProbe:
            httpGet:
              path: /
              port: 80
            initialDelaySeconds: 3
            periodSeconds: 5
          livenessProbe:
            httpGet:
              path: /
              port: 80
            initialDelaySeconds: 10
            periodSeconds: 10
---
apiVersion: v1
kind: Service
metadata:
  name: final-fixed-web-svc
  namespace: k8s-lab-observability
spec:
  type: ClusterIP
  selector:
    app: final-fixed-web
  ports:
    - name: http
      port: 80
      targetPort: 80
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: final-fixed-web-ingress
  namespace: k8s-lab-observability
spec:
  ingressClassName: nginx
  rules:
    - host: final.k8s.local
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: final-fixed-web-svc
                port:
                  number: 80
```

Apply:

```bash
kubectl apply -f 10-final-fixed-app.yaml
```

Kiểm tra:

```bash
kubectl get deploy,rs,pod,svc,ingress,endpointslice -n k8s-lab-observability
```

Test:

```bash
curl -I -H "Host: final.k8s.local" http://<INGRESS_LOADBALANCER_IP>
```

---

# Phần 16: Bộ lệnh vận hành nhanh

## 54. Pod troubleshooting quick commands

```bash
kubectl get pod -A
kubectl get pod -n <namespace> -o wide
kubectl describe pod <pod-name> -n <namespace>
kubectl logs <pod-name> -n <namespace>
kubectl logs <pod-name> -n <namespace> --previous
kubectl exec -it <pod-name> -n <namespace> -- /bin/sh
kubectl get events -n <namespace> --sort-by=.lastTimestamp
```

## 55. Deployment troubleshooting quick commands

```bash
kubectl get deploy,rs,pod -n <namespace>
kubectl describe deploy <deployment-name> -n <namespace>
kubectl rollout status deployment/<deployment-name> -n <namespace>
kubectl rollout history deployment/<deployment-name> -n <namespace>
kubectl rollout undo deployment/<deployment-name> -n <namespace>
```

## 56. Service troubleshooting quick commands

```bash
kubectl get svc -n <namespace>
kubectl describe svc <service-name> -n <namespace>
kubectl get endpointslice -n <namespace>
kubectl get pod -n <namespace> --show-labels
```

## 57. Ingress troubleshooting quick commands

```bash
kubectl get ingress -n <namespace>
kubectl describe ingress <ingress-name> -n <namespace>
kubectl get svc -n ingress-nginx
kubectl logs -n ingress-nginx deploy/ingress-nginx-controller
curl -I -H "Host: <domain>" http://<ingress-ip>
```

## 58. Metrics troubleshooting quick commands

```bash
kubectl get deployment metrics-server -n kube-system
kubectl logs -n kube-system deploy/metrics-server
kubectl get apiservice | grep metrics
kubectl top nodes
kubectl top pods -A
```

## 59. Node troubleshooting quick commands

```bash
kubectl get nodes -o wide
kubectl describe node <node-name>
sudo systemctl status kubelet
sudo journalctl -u kubelet -f
sudo systemctl status containerd
sudo journalctl -u containerd -f
df -h
free -h
ip route
```

---

# Phần 17: Best practices vận hành

## 60. Best practices về log

Ứng dụng nên ghi log ra stdout/stderr.

Không nên chỉ ghi log vào file bên trong container nếu không có sidecar hoặc logging agent thu thập.

Log nên có:

- Timestamp
- Log level
- Request ID hoặc correlation ID
- Service name
- Error message rõ ràng
- Không ghi password/token/secret ra log

Ví dụ log tốt:

```json
{
  "timestamp": "2026-04-24T10:00:01Z",
  "level": "error",
  "service": "payment-api",
  "request_id": "req-123456",
  "message": "database connection timeout"
}
```

## 61. Best practices về manifest để dễ troubleshooting

Manifest nên có label chuẩn:

```yaml
labels:
  app.kubernetes.io/name: web
  app.kubernetes.io/component: frontend
  app.kubernetes.io/part-of: demo-app
  app.kubernetes.io/managed-by: kubectl
```

Nên khai báo:

- `resources.requests`
- `resources.limits`
- `readinessProbe`
- `livenessProbe`
- `startupProbe` nếu app khởi động lâu
- `terminationGracePeriodSeconds`
- `securityContext`
- ConfigMap/Secret rõ ràng
- Service selector khớp label Pod
- Ingress host rõ ràng

## 62. Best practices về troubleshooting

Không nên xóa Pod ngay khi chưa xem log/event.

Thứ tự nên làm:

```text
1. get
2. describe
3. logs
4. events
5. kiểm tra object liên quan
6. test network nội bộ
7. kiểm tra node nếu cần
8. mới sửa manifest
```

Với lỗi production:

- Ghi nhận thời điểm lỗi.
- Lưu output quan trọng.
- Kiểm tra thay đổi gần nhất.
- Xem rollout history.
- Rollback nếu ảnh hưởng dịch vụ.
- Sau khi ổn định mới root cause analysis.

---

# Phần 18: Câu hỏi kiểm tra cuối buổi

## 63. Câu hỏi lý thuyết

### Câu 1

Sự khác nhau giữa logs, events và metrics là gì?

Gợi ý trả lời:

- Logs do ứng dụng/container ghi ra.
- Events do Kubernetes ghi nhận về object.
- Metrics là số đo định lượng như CPU/memory.

### Câu 2

Vì sao Pod bị `Pending` thường không có logs ứng dụng?

Gợi ý trả lời:

Vì Pod chưa được schedule hoặc container chưa được tạo/chạy, nên chưa có process ứng dụng ghi log.

### Câu 3

Khi gặp `ImagePullBackOff`, lệnh quan trọng nhất đầu tiên là gì?

Gợi ý trả lời:

```bash
kubectl describe pod <pod-name> -n <namespace>
```

### Câu 4

Khi gặp `CrashLoopBackOff`, tại sao nên dùng `--previous`?

Gợi ý trả lời:

Vì container có thể đã crash và restart. `--previous` giúp xem log của lần chạy trước đó.

### Câu 5

Service không có endpoint thường do những nguyên nhân nào?

Gợi ý trả lời:

- Selector không match label Pod.
- Pod không Ready.
- Pod không tồn tại.
- Namespace sai.
- Readiness probe fail.

### Câu 6

Ingress trả 404 khác gì Ingress trả 503?

Gợi ý trả lời:

- 404 thường là host/path không match rule.
- 503 thường là backend Service không có endpoint hoặc backend không sẵn sàng.

### Câu 7

Metrics Server có nên dùng thay Prometheus không?

Gợi ý trả lời:

Không. Metrics Server chủ yếu phục vụ Metrics API, `kubectl top` và autoscaling pipeline. Monitoring dài hạn nên dùng hệ thống như Prometheus/Grafana hoặc giải pháp observability tương đương.

---

## 64. Bài tập thực hành về nhà

### Bài 1

Tạo một Deployment dùng image sai tag để sinh lỗi `ImagePullBackOff`.

Yêu cầu:

- Chụp output `kubectl get pod`.
- Chụp phần Events trong `kubectl describe pod`.
- Sửa image và chứng minh Pod Running.

### Bài 2

Tạo một Pod bị `CrashLoopBackOff`.

Yêu cầu:

- Xem log hiện tại.
- Xem log bằng `--previous`.
- Giải thích vì sao container crash.
- Sửa lại command để Pod chạy ổn định.

### Bài 3

Tạo Service selector sai.

Yêu cầu:

- Chứng minh Service không có EndpointSlice backend.
- Sửa selector.
- Test truy cập Service bằng Pod `curl`.

### Bài 4

Tạo Ingress HTTP cho một Service.

Yêu cầu:

- Test bằng domain đúng.
- Test bằng IP không có Host header.
- Giải thích vì sao kết quả khác nhau.

### Bài 5

Cài Metrics Server và chạy:

```bash
kubectl top nodes
kubectl top pods -A
```

Yêu cầu:

- Ghi lại lỗi nếu có.
- Nếu phải dùng `--kubelet-insecure-tls`, giải thích vì sao chỉ nên dùng trong lab.

---

# Phần 19: Dọn dẹp lab

Xóa namespace lab:

```bash
kubectl delete namespace k8s-lab-observability
```

Nếu muốn giữ Metrics Server cho buổi sau, không xóa Metrics Server.

Nếu muốn xóa Metrics Server:

```bash
kubectl delete -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```

---

# Phần 20: Tổng kết buổi học

Trong buổi này, đã học cách quan sát và xử lý lỗi Kubernetes theo hướng vận hành thực tế.

Các ý chính cần nhớ:

- `kubectl get` cho biết trạng thái tổng quan.
- `kubectl describe` thường là lệnh quan trọng nhất khi Pod lỗi.
- `kubectl logs` đọc log container hiện tại.
- `kubectl logs --previous` dùng khi container đã crash/restart.
- `kubectl get events --sort-by=.lastTimestamp` giúp xem timeline lỗi.
- `kubectl top` cần Metrics Server.
- `ImagePullBackOff` thường liên quan image/registry/network/secret pull image.
- `CrashLoopBackOff` thường liên quan process ứng dụng crash.
- `Pending` thường liên quan scheduler, resource, taint/toleration, PVC.
- `CreateContainerConfigError` thường liên quan ConfigMap/Secret/volume/config thiếu hoặc sai.
- Service không có endpoint thường do selector sai hoặc Pod NotReady.
- Ingress 404 thường do host/path không match.
- Ingress 503 thường do backend không sẵn sàng.
- Troubleshooting tốt phải đi theo lớp, không đoán mò.

Buổi tiếp theo có thể tiếp tục với Autoscaling, đặc biệt là HPA, vì Buổi 10 đã mở khóa Metrics Server và `kubectl top`.

---

# Tài liệu tham khảo chính thức

- Kubernetes v1.35 Documentation - Observability: https://v1-35.docs.kubernetes.io/docs/concepts/cluster-administration/observability/
- Kubernetes v1.35 Documentation - Logging Architecture: https://v1-35.docs.kubernetes.io/docs/concepts/cluster-administration/logging/
- Kubernetes v1.35 Documentation - Troubleshooting Applications: https://v1-35.docs.kubernetes.io/docs/tasks/debug/debug-application/
- Kubernetes v1.35 Documentation - Debug Pods: https://v1-35.docs.kubernetes.io/docs/tasks/debug/debug-application/debug-pods/
- Kubernetes v1.35 Documentation - Debug Services: https://v1-35.docs.kubernetes.io/docs/tasks/debug/debug-application/debug-service/
- Kubernetes v1.35 Documentation - Get a Shell to a Running Container: https://v1-35.docs.kubernetes.io/docs/tasks/debug/debug-application/get-shell-running-container/
- Kubernetes v1.35 Documentation - Determine the Reason for Pod Failure: https://v1-35.docs.kubernetes.io/docs/tasks/debug/debug-application/determine-reason-pod-failure/
- Kubernetes SIGs Metrics Server: https://github.com/kubernetes-sigs/metrics-server
