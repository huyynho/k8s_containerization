# Buổi 11: Autoscaling trong Kubernetes v1.35

## Chủ đề

**Autoscaling trong Kubernetes: Manual Scale, Horizontal Pod Autoscaler, Resource Metrics, Memory/CPU Scaling và giới thiệu Node Autoscaling trong môi trường on-premise**

---

## Bối cảnh của buổi học

Ở các buổi trước, đã lần lượt đi qua các nền tảng quan trọng:

- Buổi 01: Kiến trúc Kubernetes
- Buổi 02: Triển khai cluster Kubernetes bằng kubeadm, containerd và Calico
- Buổi 03: Workloads: Pod, ReplicaSet, Deployment, DaemonSet, StatefulSet, Job, CronJob
- Buổi 04: Networking và Service
- Buổi 05: ConfigMap, Secret, Volume, PV, PVC, StorageClass, CSI NFS
- Buổi 06: Helm, MetalLB và Ingress NGINX Controller
- Buổi 07: Application lifecycle, health check, rolling update, rollback và self-healing
- Buổi 08: Resource management và scheduling cơ bản
- Buổi 09: Kubernetes security cơ bản
- Buổi 10: Observability và troubleshooting

Tới buổi này, đã đủ kiến thức để hiểu câu hỏi rất thực tế trong vận hành Kubernetes:

> Khi tải của ứng dụng tăng lên, Kubernetes có thể tự tăng số lượng Pod không?  
> Khi tải giảm xuống, Kubernetes có thể tự giảm số lượng Pod không?  
> Muốn autoscaling hoạt động ổn định thì manifest phải viết như thế nào?

Buổi 11 sẽ tập trung vào **autoscaling ở cấp workload**, chủ yếu là **Horizontal Pod Autoscaler**, viết tắt là **HPA**.

Trong môi trường cloud như AWS, Azure, Google Cloud, autoscaling thường bao gồm cả autoscaling node. Tuy nhiên, trong khóa học này, cluster đang chạy on-premise bằng kubeadm trên Ubuntu 24.04, nên phần node autoscaling sẽ chỉ được giới thiệu ở mức kiến trúc và giới hạn thực tế. Lab chính sẽ tập trung vào autoscaling Pod.

---

## Môi trường lab sử dụng trong bài

Cluster Kubernetes sử dụng trong bài:

| Thành phần | Số lượng | Hệ điều hành | Ghi chú |
|---|---:|---|---|
| Control Plane Node | 1 | Ubuntu 24.04 | Bootstrap bằng kubeadm |
| Worker Node | 2 | Ubuntu 24.04 | Chạy workload |
| NFS Server | 1 | Ubuntu 24.04 | Đã dùng từ Buổi 05 |
| Container Runtime | containerd |  | Cấu hình `SystemdCgroup = true` |
| CNI | Calico |  | Đã triển khai từ Buổi 02 |
| Ingress Controller | Ingress NGINX |  | Đã triển khai từ Buổi 06 |
| LoadBalancer on-premise | MetalLB |  | Đã triển khai từ Buổi 06 |
| Metrics | Metrics Server |  | Đã giới thiệu ở Buổi 10 |

Namespace lab trong bài:

```bash
kubectl create namespace lesson-11-autoscaling
```

---

## Mục tiêu sau buổi học

Sau buổi học này, cần đạt được các mục tiêu sau:

- Hiểu scaling trong Kubernetes là gì
- Phân biệt horizontal scaling và vertical scaling
- Biết scale Deployment thủ công bằng `kubectl scale`
- Hiểu HPA là gì và hoạt động như thế nào
- Hiểu vì sao HPA cần Metrics Server
- Hiểu quan hệ giữa HPA và `resources.requests`
- Tạo HPA dựa trên CPU utilization
- Tạo HPA dựa trên memory utilization
- Tạo tải giả lập để quan sát HPA scale up và scale down
- Đọc trạng thái HPA bằng `kubectl get hpa` và `kubectl describe hpa`
- Hiểu các lỗi thường gặp khi HPA không hoạt động
- Hiểu giới hạn của Cluster Autoscaler trong môi trường on-premise

---

## Những kiến thức mới được mở khóa trong buổi 11

Trong buổi này, chúng ta sẽ mở khóa các khái niệm mới sau:

- Manual scaling
- Horizontal scaling
- Vertical scaling
- HorizontalPodAutoscaler
- Resource metrics
- Metrics API
- CPU utilization target
- Memory utilization target
- HPA behavior
- Scale up policy
- Scale down policy
- Stabilization window
- Cluster Autoscaler ở mức khái niệm
- Vertical Pod Autoscaler ở mức khái niệm

---

## Giới hạn phạm vi của buổi học

Buổi này **không đi sâu** vào các nội dung sau:

- KEDA
- Prometheus Adapter
- Custom metrics
- External metrics
- Cluster Autoscaler production deployment
- Karpenter
- VPA production deployment
- Autoscaling theo message queue
- Autoscaling node trong AWS, Azure, GCP
- FinOps và capacity planning chuyên sâu

Các nội dung này phù hợp cho phần nâng cao sau khi đã nắm chắc HPA cơ bản.

---

# Phần 1: Scaling trong Kubernetes là gì?

## 1.1. Vấn đề thực tế

Giả sử có một ứng dụng web đang chạy bằng Deployment với 2 Pod.

Vào giờ bình thường, 2 Pod xử lý request ổn định.

Nhưng đến giờ cao điểm, số lượng request tăng mạnh. Nếu vẫn chỉ có 2 Pod, ứng dụng có thể gặp các vấn đề:

- CPU tăng cao
- latency tăng
- request bị timeout
- container bị quá tải
- user cảm nhận ứng dụng chậm
- hệ thống mất ổn định

Một cách xử lý thủ công là tăng số lượng Pod từ 2 lên 5:

```bash
kubectl scale deployment web-app --replicas=5
```

Nhưng cách này có vấn đề:

- Người vận hành phải theo dõi liên tục
- Phản ứng có thể chậm hơn tải thực tế
- Dễ quên scale down sau giờ cao điểm
- Gây lãng phí tài nguyên nếu giữ replica quá cao
- Không phù hợp với workload có tải biến động liên tục

Autoscaling sinh ra để giải quyết bài toán này.

---

## 1.2. Scaling là gì?

Scaling là quá trình thay đổi năng lực xử lý của hệ thống.

Trong Kubernetes, scaling có thể hiểu theo nhiều lớp:

| Loại scaling | Ý nghĩa | Ví dụ |
|---|---|---|
| Horizontal scaling | Tăng hoặc giảm số lượng Pod | Từ 2 Pod lên 5 Pod |
| Vertical scaling | Tăng hoặc giảm CPU/RAM cho Pod | Từ 256Mi RAM lên 512Mi RAM |
| Node scaling | Tăng hoặc giảm số lượng node trong cluster | Từ 2 worker lên 5 worker |
| Event-driven scaling | Scale theo sự kiện bên ngoài | Số message trong queue tăng |

Trong buổi này, trọng tâm là **horizontal scaling** bằng HPA.

---

## 1.3. Horizontal scaling

Horizontal scaling nghĩa là tăng số lượng bản sao của ứng dụng.

Ví dụ:

```text
Ban đầu:

web-app
├── Pod 1
└── Pod 2

Sau khi scale up:

web-app
├── Pod 1
├── Pod 2
├── Pod 3
├── Pod 4
└── Pod 5
```

Trong Kubernetes, các workload như Deployment, ReplicaSet, StatefulSet có thể scale số lượng replica.

DaemonSet không phù hợp với HPA vì DaemonSet được thiết kế để chạy một Pod trên mỗi node phù hợp, không phải để scale theo số replica tùy ý.

---

## 1.4. Vertical scaling

Vertical scaling nghĩa là tăng tài nguyên cho Pod.

Ví dụ:

```yaml
resources:
  requests:
    cpu: 100m
    memory: 128Mi
  limits:
    cpu: 500m
    memory: 512Mi
```

Tăng lên:

```yaml
resources:
  requests:
    cpu: 300m
    memory: 512Mi
  limits:
    cpu: "1"
    memory: 1Gi
```

Vertical scaling hữu ích khi ứng dụng cần thêm tài nguyên trên mỗi Pod.

Tuy nhiên, trong Kubernetes truyền thống, thay đổi CPU/RAM của Pod thường dẫn đến việc Pod phải được recreate. Kubernetes v1.35 đã có các cải tiến liên quan đến resize container resource in-place, nhưng trong khóa học cơ bản này, chúng ta chỉ giới thiệu khái niệm và chưa lab production với VPA.

---

# Phần 2: Manual scaling

Trước khi học autoscaling tự động, cần hiểu scaling thủ công.

## 2.1. Tạo Deployment mẫu

Tạo file:

```bash
mkdir -p ~/k8s-course/lesson-11
cd ~/k8s-course/lesson-11
vi 01-manual-scale-deployment.yaml
```

Nội dung manifest:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: manual-scale-web
  namespace: lesson-11-autoscaling
  labels:
    app: manual-scale-web
spec:
  replicas: 2
  selector:
    matchLabels:
      app: manual-scale-web
  template:
    metadata:
      labels:
        app: manual-scale-web
    spec:
      containers:
        - name: nginx
          image: nginx:1.27-alpine
          ports:
            - containerPort: 80
          resources:
            requests:
              cpu: 50m
              memory: 64Mi
            limits:
              cpu: 200m
              memory: 128Mi
```

Apply manifest:

```bash
kubectl apply -f 01-manual-scale-deployment.yaml
```

Kiểm tra:

```bash
kubectl get deployment -n lesson-11-autoscaling
kubectl get pod -n lesson-11-autoscaling -l app=manual-scale-web -o wide
```

---

## 2.2. Giải thích manifest

### `kind: Deployment`

```yaml
kind: Deployment
```

Deployment quản lý ReplicaSet và Pod.

Trong buổi 03, đã học Deployment dùng cho workload stateless. Ở bài này, Deployment là đối tượng được scale.

---

### `replicas: 2`

```yaml
replicas: 2
```

Kubernetes sẽ duy trì 2 Pod cho ứng dụng này.

Nếu một Pod lỗi, Deployment thông qua ReplicaSet sẽ tạo lại Pod mới để đảm bảo đủ 2 replica.

---

### `resources.requests`

```yaml
resources:
  requests:
    cpu: 50m
    memory: 64Mi
```

Đây là phần cực kỳ quan trọng.

`requests` cho scheduler biết Pod cần tối thiểu bao nhiêu tài nguyên để được xếp lịch lên node.

Đối với HPA theo CPU hoặc memory utilization, `requests` còn là mẫu số để tính phần trăm sử dụng.

Ví dụ:

```yaml
requests:
  cpu: 100m
```

Nếu container đang dùng 50m CPU, utilization là:

```text
50m / 100m = 50%
```

Nếu không khai báo `requests.cpu`, HPA sẽ không thể tính CPU utilization theo phần trăm một cách đúng nghĩa.

---

### `resources.limits`

```yaml
limits:
  cpu: 200m
  memory: 128Mi
```

`limits` đặt trần sử dụng tài nguyên.

- CPU vượt limit có thể bị throttling
- Memory vượt limit có thể bị OOMKilled

Trong autoscaling, không nên đặt limit quá thấp so với request nếu workload có burst traffic, vì Pod có thể bị throttling trước khi HPA kịp scale out.

---

## 2.3. Scale thủ công bằng kubectl

Scale từ 2 Pod lên 4 Pod:

```bash
kubectl scale deployment manual-scale-web \
  -n lesson-11-autoscaling \
  --replicas=4
```

Kiểm tra:

```bash
kubectl get deployment manual-scale-web -n lesson-11-autoscaling
kubectl get pod -n lesson-11-autoscaling -l app=manual-scale-web
```

Scale xuống 1 Pod:

```bash
kubectl scale deployment manual-scale-web \
  -n lesson-11-autoscaling \
  --replicas=1
```

---

## 2.4. Quan sát ReplicaSet thay đổi số lượng Pod

```bash
kubectl get rs -n lesson-11-autoscaling
```

Khi scale Deployment, Kubernetes không tạo Deployment mới.

Deployment sẽ điều chỉnh số replica mong muốn trên ReplicaSet hiện tại.

---

## 2.5. Bài học từ manual scaling

Manual scaling phù hợp khi:

- Người vận hành biết rõ thời điểm cần scale
- Tải tăng theo lịch cố định
- Cần xử lý sự cố nhanh
- Muốn can thiệp thủ công trong maintenance window

Manual scaling không phù hợp khi:

- Tải thay đổi liên tục
- Không có người theo dõi 24/7
- Cần phản ứng tự động
- Muốn tối ưu tài nguyên theo thời gian thực

Vì vậy, chúng ta cần HPA.

---

# Phần 3: Horizontal Pod Autoscaler là gì?

## 3.1. Định nghĩa

Horizontal Pod Autoscaler, viết tắt là HPA, là một Kubernetes API resource và controller dùng để tự động điều chỉnh số lượng replica của workload dựa trên metric.

Ví dụ:

```text
Nếu CPU trung bình của Pod vượt 50%, tăng số Pod lên.

Nếu CPU giảm thấp trong một khoảng thời gian, giảm số Pod xuống.
```

HPA thường được dùng với:

- Deployment
- ReplicaSet
- StatefulSet

Không dùng với:

- DaemonSet
- Pod đơn lẻ
- Job/CronJob theo cách thông thường

---

## 3.2. HPA không trực tiếp tạo Pod

Đây là điểm rất dễ hiểu nhầm.

HPA không trực tiếp tạo Pod.

Luồng hoạt động đúng là:

```text
HPA
  └── thay đổi spec.replicas của Deployment
        └── Deployment điều chỉnh ReplicaSet
              └── ReplicaSet tạo hoặc xóa Pod
```

HPA chỉ điều chỉnh số replica mong muốn.

Các controller khác sẽ thực hiện phần còn lại.

---

## 3.3. HPA cần metric từ đâu?

HPA cần metric để ra quyết định.

Đối với CPU và memory cơ bản, HPA thường lấy metric từ:

```text
Metrics Server
  └── metrics.k8s.io API
        └── HPA Controller
```

Luồng tổng quát:

```text
Kubelet trên từng node
  └── cung cấp metric sử dụng tài nguyên của Pod/container
        └── Metrics Server thu thập
              └── Kubernetes API expose metrics.k8s.io
                    └── HPA controller đọc metric
                          └── quyết định scale Deployment
```

Nếu Metrics Server chưa hoạt động, HPA sẽ không có dữ liệu để tính toán.

---

## 3.4. Kiểm tra Metrics Server

Trong Buổi 10, đã học Metrics Server. Ở bài này chỉ kiểm tra lại.

```bash
kubectl get deployment metrics-server -n kube-system
kubectl get pod -n kube-system -l k8s-app=metrics-server
```

Kiểm tra API metric:

```bash
kubectl top nodes
kubectl top pods -A
```

Nếu lệnh `kubectl top` trả về dữ liệu CPU/RAM, Metrics Server đã hoạt động.

Ví dụ output:

```text
NAME        CPU(cores)   CPU%   MEMORY(bytes)   MEMORY%
master-01   180m         9%     1450Mi          38%
worker-01   120m         6%     980Mi           25%
worker-02   110m         5%     910Mi           23%
```

---

## 3.5. Cài Metrics Server nếu chưa có

Nếu cluster chưa cài Metrics Server, có thể cài bằng manifest chính thức:

```bash
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```

Trong môi trường kubeadm lab, nếu Metrics Server không đọc được kubelet do vấn đề TLS certificate, có thể patch thêm tham số `--kubelet-insecure-tls`.

```bash
kubectl patch deployment metrics-server -n kube-system \
  --type='json' \
  -p='[
    {
      "op": "add",
      "path": "/spec/template/spec/containers/0/args/-",
      "value": "--kubelet-insecure-tls"
    }
  ]'
```

Chờ rollout:

```bash
kubectl rollout status deployment/metrics-server -n kube-system
```

Kiểm tra lại:

```bash
kubectl top nodes
kubectl top pods -A
```

> Lưu ý vận hành: `--kubelet-insecure-tls` phù hợp cho lab kubeadm khi kubelet certificate chưa được chuẩn hóa SAN đầy đủ. Với production, nên cấu hình certificate đúng thay vì bỏ kiểm tra TLS.

---

# Phần 4: Cách HPA tính toán replica

## 4.1. Công thức cơ bản

Công thức tư duy đơn giản của HPA:

```text
desiredReplicas = ceil(currentReplicas × currentMetricValue / desiredMetricValue)
```

Ví dụ:

```text
currentReplicas = 2
current CPU average = 100%
target CPU average = 50%

desiredReplicas = ceil(2 × 100 / 50)
desiredReplicas = ceil(4)
desiredReplicas = 4
```

Nếu CPU trung bình đang gấp đôi mục tiêu, HPA có xu hướng tăng số replica gấp đôi.

---

## 4.2. Ví dụ dễ hiểu

Giả sử Deployment có 2 Pod.

HPA cấu hình:

```yaml
minReplicas: 2
maxReplicas: 6
target CPU: 50%
```

Tình huống 1:

```text
CPU trung bình hiện tại = 50%
```

Kết quả:

```text
Không scale
```

Tình huống 2:

```text
CPU trung bình hiện tại = 100%
```

Kết quả dự kiến:

```text
2 Pod × 100 / 50 = 4 Pod
```

Tình huống 3:

```text
CPU trung bình hiện tại = 25%
```

Kết quả dự kiến:

```text
2 Pod × 25 / 50 = 1 Pod
```

Nhưng nếu `minReplicas: 2`, HPA sẽ không scale xuống dưới 2 Pod.

---

## 4.3. HPA không scale liên tục từng giây

HPA là control loop chạy định kỳ.

Nó không phải cơ chế real-time từng millisecond.

Mặc định, HPA controller trong kube-controller-manager kiểm tra metric theo chu kỳ. Vì vậy khi tạo tải, cần chờ một khoảng thời gian mới thấy replica thay đổi.

---

## 4.4. HPA có cơ chế chống flapping

Nếu metric dao động nhẹ quanh target, HPA không nhất thiết scale ngay.

Ví dụ target CPU là 50%.

Nếu CPU dao động:

```text
48%, 51%, 49%, 52%
```

Nếu cứ scale lên/xuống liên tục thì hệ thống sẽ rất bất ổn.

Vì vậy HPA có tolerance và stabilization window để hạn chế flapping.

---

## 4.5. Tại sao `resources.requests` rất quan trọng với HPA?

Giả sử container khai báo:

```yaml
resources:
  requests:
    cpu: 100m
```

Nếu container đang dùng 60m CPU:

```text
CPU utilization = 60m / 100m = 60%
```

Nếu HPA target là 50%, HPA thấy Pod đang cao hơn mục tiêu.

Nếu container không có `requests.cpu`, HPA không thể tính utilization theo phần trăm request.

Khi đó HPA có thể báo lỗi kiểu:

```text
missing request for cpu
```

Vì vậy, muốn dùng HPA dựa trên CPU/memory utilization, workload nên khai báo `resources.requests` rõ ràng.

---

# Phần 5: Lab 1 - HPA theo CPU

## 5.1. Mục tiêu lab

Trong lab này, sẽ:

- Deploy một ứng dụng web tiêu thụ CPU
- Tạo Service ClusterIP cho ứng dụng
- Tạo HPA dựa trên CPU utilization
- Tạo Pod load generator để bắn request liên tục
- Quan sát HPA scale up
- Dừng load generator
- Quan sát HPA scale down

---

## 5.2. Tạo Deployment và Service

Tạo file:

```bash
vi 02-cpu-hpa-app.yaml
```

Nội dung:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: cpu-demo
  namespace: lesson-11-autoscaling
  labels:
    app: cpu-demo
spec:
  replicas: 1
  selector:
    matchLabels:
      app: cpu-demo
  template:
    metadata:
      labels:
        app: cpu-demo
    spec:
      containers:
        - name: php-apache
          image: registry.k8s.io/hpa-example
          ports:
            - containerPort: 80
          resources:
            requests:
              cpu: 100m
              memory: 64Mi
            limits:
              cpu: 500m
              memory: 128Mi
---
apiVersion: v1
kind: Service
metadata:
  name: cpu-demo
  namespace: lesson-11-autoscaling
  labels:
    app: cpu-demo
spec:
  type: ClusterIP
  selector:
    app: cpu-demo
  ports:
    - name: http
      port: 80
      targetPort: 80
```

Apply:

```bash
kubectl apply -f 02-cpu-hpa-app.yaml
```

Kiểm tra:

```bash
kubectl get deployment,pod,svc -n lesson-11-autoscaling -l app=cpu-demo
```

---

## 5.3. Giải thích manifest Deployment

### `image: registry.k8s.io/hpa-example`

```yaml
image: registry.k8s.io/hpa-example
```

Đây là image demo thường được dùng để kiểm thử HPA.

Ứng dụng này có thể tiêu thụ CPU khi nhận request liên tục, phù hợp cho lab autoscaling.

---

### `replicas: 1`

```yaml
replicas: 1
```

Ban đầu chỉ chạy 1 Pod.

Khi tải tăng, HPA sẽ tăng số replica.

---

### `requests.cpu: 100m`

```yaml
requests:
  cpu: 100m
```

Pod yêu cầu 100 millicore CPU.

Nếu HPA target CPU là 50%, nghĩa là mỗi Pod nên duy trì mức trung bình khoảng:

```text
50% × 100m = 50m CPU
```

Nếu Pod dùng cao hơn mức này, HPA sẽ xem xét scale up.

---

### `limits.cpu: 500m`

```yaml
limits:
  cpu: 500m
```

Container được phép burst tối đa đến 500m CPU.

Trong lab, limit này giúp container có đủ room để tạo CPU usage, nhưng vẫn tránh chiếm toàn bộ CPU node.

---

## 5.4. Giải thích manifest Service

```yaml
kind: Service
spec:
  type: ClusterIP
  selector:
    app: cpu-demo
```

Service tạo endpoint nội bộ để Pod load generator có thể gọi đến ứng dụng bằng DNS name:

```text
http://cpu-demo.lesson-11-autoscaling.svc.cluster.local
```

Hoặc ngắn hơn nếu cùng namespace:

```text
http://cpu-demo
```

---

## 5.5. Tạo HPA theo CPU bằng YAML

Tạo file:

```bash
vi 03-cpu-hpa.yaml
```

Nội dung:

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: cpu-demo-hpa
  namespace: lesson-11-autoscaling
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: cpu-demo
  minReplicas: 1
  maxReplicas: 6
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 50
```

Apply:

```bash
kubectl apply -f 03-cpu-hpa.yaml
```

Kiểm tra:

```bash
kubectl get hpa -n lesson-11-autoscaling
kubectl describe hpa cpu-demo-hpa -n lesson-11-autoscaling
```

---

## 5.6. Giải thích manifest HPA

### `apiVersion: autoscaling/v2`

```yaml
apiVersion: autoscaling/v2
```

Sử dụng HPA API version `autoscaling/v2`.

Version này hỗ trợ nhiều loại metric hơn và có thể cấu hình behavior chi tiết hơn so với autoscaling/v1.

---

### `kind: HorizontalPodAutoscaler`

```yaml
kind: HorizontalPodAutoscaler
```

Định nghĩa một đối tượng HPA.

HPA sẽ chạy như một controller loop ở control plane để quan sát metric và điều chỉnh số replica.

---

### `scaleTargetRef`

```yaml
scaleTargetRef:
  apiVersion: apps/v1
  kind: Deployment
  name: cpu-demo
```

Đây là workload mà HPA sẽ scale.

Trong ví dụ này, HPA scale Deployment tên `cpu-demo`.

HPA không scale Service, không scale Pod trực tiếp, cũng không scale Node.

---

### `minReplicas`

```yaml
minReplicas: 1
```

Số Pod tối thiểu.

Ngay cả khi CPU rất thấp, HPA sẽ không giảm xuống dưới 1 Pod.

---

### `maxReplicas`

```yaml
maxReplicas: 6
```

Số Pod tối đa.

Ngay cả khi CPU rất cao, HPA cũng không scale vượt quá 6 Pod.

Đây là cơ chế bảo vệ cluster khỏi việc scale quá nhiều gây hết tài nguyên.

---

### `metrics`

```yaml
metrics:
  - type: Resource
```

HPA sử dụng resource metric.

Resource metric phổ biến là:

- CPU
- memory

---

### `averageUtilization: 50`

```yaml
averageUtilization: 50
```

Mục tiêu là giữ CPU trung bình của các Pod ở khoảng 50% so với `requests.cpu`.

Ví dụ mỗi Pod request 100m CPU, 50% tương đương khoảng 50m CPU.

---

## 5.7. Tạo HPA bằng kubectl autoscale

Ngoài YAML, có thể tạo HPA nhanh bằng lệnh:

```bash
kubectl autoscale deployment cpu-demo \
  -n lesson-11-autoscaling \
  --min=1 \
  --max=6 \
  --cpu=50%
```

Tuy nhiên trong giáo trình, nên ưu tiên YAML để hiểu rõ object được tạo.

Có thể xem YAML được sinh ra bằng dry-run:

```bash
kubectl autoscale deployment cpu-demo \
  -n lesson-11-autoscaling \
  --min=1 \
  --max=6 \
  --cpu=50% \
  --dry-run=client \
  -o yaml
```

---

## 5.8. Tạo load generator

Tạo một Pod để gửi request liên tục vào Service:

```bash
kubectl run cpu-load-generator \
  -n lesson-11-autoscaling \
  --image=busybox:1.36 \
  --restart=Never \
  -- /bin/sh -c "while true; do wget -q -O- http://cpu-demo; done"
```

Kiểm tra Pod load generator:

```bash
kubectl get pod cpu-load-generator -n lesson-11-autoscaling
```

---

## 5.9. Quan sát HPA scale up

Mở một terminal khác và chạy:

```bash
kubectl get hpa -n lesson-11-autoscaling -w
```

Terminal khác:

```bash
kubectl get deployment cpu-demo -n lesson-11-autoscaling -w
```

Kiểm tra Pod:

```bash
kubectl get pod -n lesson-11-autoscaling -l app=cpu-demo -o wide
```

Sau một thời gian, số replica có thể tăng từ 1 lên nhiều Pod.

Ví dụ:

```text
NAME           REFERENCE             TARGETS    MINPODS   MAXPODS   REPLICAS   AGE
cpu-demo-hpa   Deployment/cpu-demo    120%/50%   1         6         3          3m
```

Ý nghĩa:

```text
CPU hiện tại trung bình khoảng 120%
Target là 50%
HPA đã scale Deployment lên 3 replica
```

---

## 5.10. Dừng load generator

Xóa Pod load generator:

```bash
kubectl delete pod cpu-load-generator -n lesson-11-autoscaling
```

Tiếp tục quan sát:

```bash
kubectl get hpa -n lesson-11-autoscaling -w
kubectl get deployment cpu-demo -n lesson-11-autoscaling -w
```

HPA thường không scale down ngay lập tức.

Lý do là HPA có stabilization window để tránh scale down quá nhanh khi traffic vừa giảm tạm thời.

---

## 5.11. Kiểm tra event của HPA

```bash
kubectl describe hpa cpu-demo-hpa -n lesson-11-autoscaling
```

Quan sát các phần:

```text
Metrics
Conditions
Events
```

Một HPA healthy thường có các condition như:

```text
AbleToScale
ScalingActive
ScalingLimited
```

Ý nghĩa tổng quát:

| Condition | Ý nghĩa |
|---|---|
| AbleToScale | HPA có thể lấy và cập nhật scale target |
| ScalingActive | HPA đang có metric hợp lệ để tính toán |
| ScalingLimited | HPA bị giới hạn bởi minReplicas hoặc maxReplicas |

---

# Phần 6: Lab 2 - HPA theo memory

## 6.1. Mục tiêu lab

Trong lab này, sẽ:

- Tạo một app Python đơn giản có endpoint cấp phát memory
- Tạo Deployment có `requests.memory`
- Tạo HPA dựa trên memory utilization
- Gọi endpoint để tăng memory usage
- Quan sát HPA scale up

---

## 6.2. Tạo ConfigMap chứa code Python

Tạo file:

```bash
vi 04-memory-app-configmap.yaml
```

Nội dung:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: memory-app-code
  namespace: lesson-11-autoscaling
data:
  app.py: |
    from http.server import BaseHTTPRequestHandler, HTTPServer
    from urllib.parse import urlparse, parse_qs
    import os

    allocated_memory = []

    class Handler(BaseHTTPRequestHandler):
        def do_GET(self):
            parsed = urlparse(self.path)

            if parsed.path == "/":
                message = "memory autoscaling demo\n"
                self.send_response(200)
                self.end_headers()
                self.wfile.write(message.encode())
                return

            if parsed.path == "/allocate":
                query = parse_qs(parsed.query)
                mb = int(query.get("mb", ["10"])[0])

                # Allocate memory and keep reference in global list
                allocated_memory.append(bytearray(mb * 1024 * 1024))

                message = f"allocated {mb} MiB, total chunks: {len(allocated_memory)}\n"
                self.send_response(200)
                self.end_headers()
                self.wfile.write(message.encode())
                return

            if parsed.path == "/status":
                message = f"pid={os.getpid()}, chunks={len(allocated_memory)}\n"
                self.send_response(200)
                self.end_headers()
                self.wfile.write(message.encode())
                return

            self.send_response(404)
            self.end_headers()

    if __name__ == "__main__":
        port = 8080
        server = HTTPServer(("0.0.0.0", port), Handler)
        print(f"memory app listening on port {port}")
        server.serve_forever()
```

Apply:

```bash
kubectl apply -f 04-memory-app-configmap.yaml
```

---

## 6.3. Giải thích ConfigMap

```yaml
kind: ConfigMap
```

ConfigMap được dùng để lưu source code demo `app.py`.

Ở production, thường không nên mount source code ứng dụng bằng ConfigMap theo cách này. Tuy nhiên trong lab, cách này giúp không cần build Docker image riêng.

---

```python
allocated_memory = []
```

Danh sách này giữ các block memory đã cấp phát.

Nếu không giữ reference, Python garbage collector có thể dọn memory, làm lab khó quan sát.

---

```python
allocated_memory.append(bytearray(mb * 1024 * 1024))
```

Mỗi lần gọi `/allocate?mb=10`, app sẽ cấp phát thêm khoảng 10 MiB memory.

---

## 6.4. Tạo Deployment và Service cho memory app

Tạo file:

```bash
vi 05-memory-app-deployment.yaml
```

Nội dung:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: memory-demo
  namespace: lesson-11-autoscaling
  labels:
    app: memory-demo
spec:
  replicas: 1
  selector:
    matchLabels:
      app: memory-demo
  template:
    metadata:
      labels:
        app: memory-demo
    spec:
      containers:
        - name: memory-app
          image: python:3.12-alpine
          command:
            - python
            - /app/app.py
          ports:
            - containerPort: 8080
          resources:
            requests:
              cpu: 50m
              memory: 64Mi
            limits:
              cpu: 300m
              memory: 256Mi
          volumeMounts:
            - name: app-code
              mountPath: /app
      volumes:
        - name: app-code
          configMap:
            name: memory-app-code
---
apiVersion: v1
kind: Service
metadata:
  name: memory-demo
  namespace: lesson-11-autoscaling
  labels:
    app: memory-demo
spec:
  type: ClusterIP
  selector:
    app: memory-demo
  ports:
    - name: http
      port: 8080
      targetPort: 8080
```

Apply:

```bash
kubectl apply -f 05-memory-app-deployment.yaml
```

Kiểm tra:

```bash
kubectl get pod,svc -n lesson-11-autoscaling -l app=memory-demo
```

---

## 6.5. Giải thích manifest memory app

### `image: python:3.12-alpine`

```yaml
image: python:3.12-alpine
```

Dùng image Python có sẵn để chạy script được mount từ ConfigMap.

---

### `command`

```yaml
command:
  - python
  - /app/app.py
```

Container chạy file `/app/app.py`.

File này đến từ ConfigMap `memory-app-code`.

---

### `requests.memory: 64Mi`

```yaml
requests:
  memory: 64Mi
```

HPA memory utilization sẽ lấy memory usage hiện tại chia cho `requests.memory`.

Nếu HPA target là 60%, thì mục tiêu là:

```text
60% × 64Mi = khoảng 38Mi
```

Khi app dùng memory cao hơn mức này, HPA có thể scale up.

---

### `limits.memory: 256Mi`

```yaml
limits:
  memory: 256Mi
```

Nếu container dùng quá 256Mi, container có thể bị OOMKilled.

Trong lab, ta cần cấp phát memory vừa đủ để kích hoạt HPA nhưng không vượt quá limit quá nhanh.

---

## 6.6. Tạo HPA theo memory

Tạo file:

```bash
vi 06-memory-hpa.yaml
```

Nội dung:

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: memory-demo-hpa
  namespace: lesson-11-autoscaling
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: memory-demo
  minReplicas: 1
  maxReplicas: 5
  metrics:
    - type: Resource
      resource:
        name: memory
        target:
          type: Utilization
          averageUtilization: 60
```

Apply:

```bash
kubectl apply -f 06-memory-hpa.yaml
```

Kiểm tra:

```bash
kubectl get hpa memory-demo-hpa -n lesson-11-autoscaling
```

---

## 6.7. Tạo load để tăng memory

Tạo Pod curl tạm thời:

```bash
kubectl run memory-client \
  -n lesson-11-autoscaling \
  --image=curlimages/curl:8.10.1 \
  --restart=Never \
  --command -- sh -c 'for i in $(seq 1 20); do curl -s http://memory-demo:8080/allocate?mb=10; sleep 2; done; sleep 3600'
```

Quan sát log:

```bash
kubectl logs -n lesson-11-autoscaling memory-client
```

Quan sát memory usage:

```bash
kubectl top pod -n lesson-11-autoscaling
```

Quan sát HPA:

```bash
kubectl get hpa -n lesson-11-autoscaling -w
```

Kiểm tra Deployment:

```bash
kubectl get deployment memory-demo -n lesson-11-autoscaling -w
```

---

## 6.8. Lưu ý về lab memory HPA

Memory HPA đôi khi khó quan sát hơn CPU HPA vì:

- Memory không giảm nhanh như CPU
- App có thể giữ memory lâu
- HPA scale down có stabilization window
- Memory usage phân bổ không đều giữa Pod cũ và Pod mới
- Nếu gọi Service, request có thể không vào đều tất cả Pod

Vì vậy, CPU HPA thường là lab dễ quan sát hơn. Memory HPA giúp hiểu thêm rằng HPA không chỉ scale theo CPU.

---

# Phần 7: HPA behavior

## 7.1. Vì sao cần cấu hình behavior?

Nếu chỉ dùng HPA mặc định, Kubernetes vẫn có logic scale up/scale down hợp lý.

Tuy nhiên trong production, ta thường muốn kiểm soát kỹ hơn:

- Scale up nhanh hay chậm?
- Scale down nhanh hay chậm?
- Có nên chờ vài phút trước khi scale down không?
- Mỗi lần scale tối đa bao nhiêu Pod?
- Có muốn tắt scale down tạm thời không?

Đó là lý do có `behavior`.

---

## 7.2. Ví dụ HPA behavior

Tạo file:

```bash
vi 07-cpu-hpa-with-behavior.yaml
```

Nội dung:

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: cpu-demo-hpa
  namespace: lesson-11-autoscaling
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: cpu-demo
  minReplicas: 1
  maxReplicas: 6
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 50
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 0
      policies:
        - type: Pods
          value: 2
          periodSeconds: 30
        - type: Percent
          value: 100
          periodSeconds: 30
      selectPolicy: Max
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
        - type: Pods
          value: 1
          periodSeconds: 60
      selectPolicy: Min
```

Apply:

```bash
kubectl apply -f 07-cpu-hpa-with-behavior.yaml
```

---

## 7.3. Giải thích `scaleUp`

```yaml
scaleUp:
  stabilizationWindowSeconds: 0
```

Khi tải tăng, cho phép scale up ngay, không cần chờ stabilization window.

Điều này phù hợp với ứng dụng web cần phản ứng nhanh khi traffic tăng.

---

```yaml
policies:
  - type: Pods
    value: 2
    periodSeconds: 30
```

Trong mỗi 30 giây, HPA có thể thêm tối đa 2 Pod theo policy này.

---

```yaml
  - type: Percent
    value: 100
    periodSeconds: 30
```

Trong mỗi 30 giây, HPA có thể tăng tối đa 100% số Pod hiện tại.

Ví dụ đang có 3 Pod, có thể tăng thêm tối đa 3 Pod.

---

```yaml
selectPolicy: Max
```

Nếu có nhiều policy, chọn policy cho phép scale nhiều nhất.

Tức là scale up sẽ phản ứng nhanh hơn.

---

## 7.4. Giải thích `scaleDown`

```yaml
scaleDown:
  stabilizationWindowSeconds: 300
```

Khi tải giảm, HPA sẽ cân nhắc các đề xuất scale down trong vòng 300 giây.

Điều này giúp tránh việc vừa giảm Pod xong thì traffic tăng lại ngay.

---

```yaml
policies:
  - type: Pods
    value: 1
    periodSeconds: 60
```

Mỗi 60 giây chỉ giảm tối đa 1 Pod.

---

```yaml
selectPolicy: Min
```

Chọn policy giảm ít nhất.

Scale down sẽ thận trọng hơn scale up.

Đây là tư duy khá phổ biến trong production:

```text
Scale up nhanh để bảo vệ user experience.
Scale down chậm để tránh dao động và giảm rủi ro.
```

---

# Phần 8: HPA và Deployment manifest - vấn đề cần biết

## 8.1. Có nên giữ `spec.replicas` khi dùng HPA không?

Khi đã dùng HPA, cần cẩn thận với trường:

```yaml
spec:
  replicas: 3
```

Nếu HPA đang scale Deployment lên 5 Pod, nhưng sau đó người vận hành apply lại manifest có `replicas: 3`, Deployment có thể bị kéo về 3 replica.

Điều này có thể gây conflict giữa GitOps/manual apply và HPA.

Trong production, có hai hướng xử lý:

### Hướng 1: Giữ `replicas` làm giá trị ban đầu

Phù hợp khi mới triển khai.

Sau đó HPA điều chỉnh số replica.

Nhưng cần hiểu rằng apply lại manifest có thể ảnh hưởng replica.

### Hướng 2: Bỏ `spec.replicas` khỏi Deployment manifest khi đã giao cho HPA quản lý

Phù hợp với GitOps hoặc production manifest đã ổn định.

Khi đó HPA là nơi quản lý số replica thực tế.

---

## 8.2. HPA và rolling update

HPA có thể hoạt động cùng rolling update.

Ví dụ:

- Deployment đang có 5 replica do HPA scale lên
- Người vận hành update image
- Deployment thực hiện rolling update
- HPA tiếp tục giám sát metric

Tuy nhiên cần chú ý:

- Pod mới cần readinessProbe tốt
- Resource request phải giống hoặc hợp lý
- Không đổi tên container nếu HPA đang dùng containerResource metric
- Không đặt `maxUnavailable` quá cao nếu workload đang chịu tải

---

## 8.3. HPA và readinessProbe

HPA có tính đến trạng thái Ready của Pod trong quá trình tính toán.

Nếu Pod chưa Ready, metric của Pod đó có thể chưa được dùng như Pod ổn định.

Vì vậy, readinessProbe từ Buổi 07 rất quan trọng với autoscaling.

Một Pod mới scale up cần Ready trước khi Service gửi traffic vào nó.

---

## 8.4. HPA và resource limit

Một lỗi thiết kế thường gặp:

```yaml
requests:
  cpu: 100m
limits:
  cpu: 100m
```

Request bằng limit không sai tuyệt đối, nhưng nếu app cần burst CPU ngắn hạn, container có thể bị throttle mạnh.

Trong nhiều ứng dụng web, có thể để:

```yaml
requests:
  cpu: 100m
limits:
  cpu: 500m
```

Tức là:

- Scheduler tính capacity dựa trên 100m
- App có thể burst đến 500m
- HPA có tín hiệu CPU để scale out khi tải tăng

Không có con số cố định cho mọi ứng dụng. Cần đo đạc bằng observability.

---

# Phần 9: Lab 3 - HPA với Ingress

## 9.1. Mục tiêu lab

Trong lab này, sẽ kết hợp kiến thức từ Buổi 06 và Buổi 11:

- Publish ứng dụng CPU demo qua Ingress
- Bắn request từ bên ngoài cluster
- Quan sát HPA scale up

Lab này giúp hiểu rằng:

```text
Ingress chỉ là đường vào HTTP/HTTPS.
HPA scale Deployment phía sau Service.
Hai cơ chế này độc lập nhưng phối hợp với nhau.
```

---

## 9.2. Tạo Ingress cho CPU demo

Điều kiện:

- Ingress NGINX Controller đã chạy
- MetalLB đã cấp IP cho Ingress Controller
- Máy client/laptop có thể resolve domain lab

Ví dụ domain lab:

```text
cpu-demo.k8s.local
```

Nếu chưa có DNS, có thể thêm vào file hosts trên máy client:

```text
<INGRESS_EXTERNAL_IP> cpu-demo.k8s.local
```

Xem IP của Ingress Controller:

```bash
kubectl get svc -n ingress-nginx
```

Tạo file:

```bash
vi 08-cpu-demo-ingress.yaml
```

Nội dung:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: cpu-demo-ingress
  namespace: lesson-11-autoscaling
spec:
  ingressClassName: nginx
  rules:
    - host: cpu-demo.k8s.local
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: cpu-demo
                port:
                  number: 80
```

Apply:

```bash
kubectl apply -f 08-cpu-demo-ingress.yaml
```

Kiểm tra:

```bash
kubectl get ingress -n lesson-11-autoscaling
```

Test:

```bash
curl http://cpu-demo.k8s.local
```

---

## 9.3. Tạo tải từ bên ngoài cluster

Từ máy client hoặc một VM khác cùng mạng:

```bash
while true; do curl -s http://cpu-demo.k8s.local > /dev/null; done
```

Hoặc chạy nhiều terminal song song để tạo tải mạnh hơn.

Quan sát:

```bash
kubectl get hpa -n lesson-11-autoscaling -w
kubectl get pod -n lesson-11-autoscaling -l app=cpu-demo -o wide -w
```

---

## 9.4. Ý nghĩa thực tế

Trong production, traffic thường đi theo luồng:

```text
User
  └── DNS
        └── Load Balancer / MetalLB IP
              └── Ingress Controller
                    └── Service
                          └── Pod
```

HPA không nằm trên đường đi của traffic.

HPA quan sát metric và thay đổi số Pod phía sau Service.

---

# Phần 10: Cluster Autoscaler trong môi trường on-premise

## 10.1. HPA scale Pod, không scale Node

Một hiểu nhầm phổ biến:

> Có HPA rồi thì cluster tự có thêm node khi thiếu tài nguyên?

Không đúng.

HPA chỉ scale số lượng replica của workload.

Nếu cluster không đủ CPU/RAM để chạy thêm Pod, Pod mới sẽ rơi vào trạng thái `Pending`.

Ví dụ:

```text
0/2 nodes are available: insufficient cpu
```

Lúc này HPA đã làm đúng việc của nó: tăng desired replicas.

Nhưng scheduler không thể đặt Pod mới vì không còn node đủ tài nguyên.

---

## 10.2. Node autoscaling là gì?

Node autoscaling là cơ chế tự động thêm hoặc bớt node trong cluster.

Ví dụ:

```text
HPA muốn tăng từ 3 Pod lên 10 Pod.
Cluster hiện tại chỉ đủ tài nguyên cho 5 Pod.
Node Autoscaler phát hiện Pod Pending.
Node Autoscaler yêu cầu cloud provider tạo thêm VM.
Node mới join cluster.
Scheduler đặt Pod Pending lên node mới.
```

Trong cloud, cơ chế này thường tích hợp với:

- AWS Auto Scaling Group
- Azure Virtual Machine Scale Sets
- Google Managed Instance Groups
- Karpenter trên AWS
- Cluster Autoscaler

---

## 10.3. Vì sao on-premise khó autoscale node?

Cluster trong khóa học chạy bằng kubeadm trên các VM Ubuntu 24.04.

Node là VM đã được tạo sẵn.

Kubernetes không tự biết cách:

- Tạo thêm VM trên VMware/Hyper-V/OpenStack
- Cài Ubuntu lên VM mới
- Cài containerd
- Cài kubelet
- Join node bằng kubeadm
- Cấu hình network, DNS, firewall
- Gắn node vào đúng subnet/VLAN

Do đó, trên on-premise, node autoscaling cần tích hợp thêm với nền tảng hạ tầng bên dưới.

Ví dụ:

- VMware vSphere automation
- OpenStack Nova automation
- Terraform
- Cluster API
- Morpheus / private cloud automation
- custom provisioning system

Trong khóa học cơ bản này, chúng ta chỉ dừng ở mức hiểu kiến trúc.

---

## 10.4. Tình huống mô phỏng thiếu tài nguyên

Có thể thử scale Deployment lên số replica cao:

```bash
kubectl scale deployment cpu-demo \
  -n lesson-11-autoscaling \
  --replicas=50
```

Quan sát:

```bash
kubectl get pod -n lesson-11-autoscaling -l app=cpu-demo
kubectl describe pod <pending-pod-name> -n lesson-11-autoscaling
```

Có thể thấy lỗi dạng:

```text
Insufficient cpu
Insufficient memory
```

Đưa về số thấp:

```bash
kubectl scale deployment cpu-demo \
  -n lesson-11-autoscaling \
  --replicas=1
```

Nếu đang có HPA, HPA có thể điều chỉnh lại số replica theo metric.

---

# Phần 11: Vertical Pod Autoscaler và In-place Pod Resize

## 11.1. VPA là gì?

Vertical Pod Autoscaler, viết tắt là VPA, là cơ chế tự động khuyến nghị hoặc điều chỉnh CPU/RAM request cho Pod.

So sánh:

| Cơ chế | Scale cái gì? |
|---|---|
| HPA | Số lượng Pod |
| VPA | CPU/RAM request của Pod |
| Cluster Autoscaler | Số lượng node |

Ví dụ HPA:

```text
Từ 2 Pod lên 5 Pod
```

Ví dụ VPA:

```text
Pod từ request 100m CPU / 128Mi RAM
lên request 300m CPU / 512Mi RAM
```

---

## 11.2. Vì sao không lab VPA trong buổi này?

VPA không phải component mặc định được cài sẵn trong Kubernetes core.

Để dùng VPA cần cài thêm các component riêng.

Ngoài ra, VPA và HPA có thể xung đột nếu cùng điều chỉnh dựa trên CPU/memory.

Ví dụ:

- HPA dùng CPU utilization
- VPA lại thay đổi CPU request
- Khi request thay đổi, phần trăm utilization cũng thay đổi
- Kết quả autoscaling có thể khó dự đoán nếu không thiết kế cẩn thận

Vì vậy, với khóa cơ bản, chúng ta tập trung vào HPA.

---

## 11.3. In-place Pod Resize ở Kubernetes v1.35

Kubernetes v1.35 có các cải tiến liên quan đến việc resize resource của container mà không nhất thiết phải recreate Pod trong một số trường hợp.

Tuy nhiên, đây là chủ đề nâng cao.

Trong giáo trình cơ bản này, chỉ cần nhớ:

- HPA scale số lượng Pod
- VPA liên quan đến tăng/giảm CPU/RAM cho Pod
- In-place resize là hướng phát triển giúp vertical scaling mượt hơn
- Không nên đưa vào lab cơ bản nếu chưa hiểu rõ resource management và autoscaling

---

# Phần 12: Troubleshooting HPA

## 12.1. HPA hiển thị `<unknown>`

Ví dụ:

```text
NAME           REFERENCE             TARGETS         MINPODS   MAXPODS   REPLICAS
cpu-demo-hpa   Deployment/cpu-demo    <unknown>/50%   1         6         1
```

Nguyên nhân thường gặp:

- Metrics Server chưa chạy
- Metrics API chưa sẵn sàng
- Pod chưa có metric
- Pod chưa Ready
- Resource request chưa được khai báo
- HPA trỏ sai Deployment
- Workload không có Pod matching selector

Kiểm tra:

```bash
kubectl top nodes
kubectl top pods -n lesson-11-autoscaling
kubectl describe hpa cpu-demo-hpa -n lesson-11-autoscaling
kubectl describe deployment cpu-demo -n lesson-11-autoscaling
```

---

## 12.2. Lỗi missing request for cpu

Triệu chứng trong `kubectl describe hpa`:

```text
missing request for cpu
```

Nguyên nhân:

```yaml
resources:
  requests:
    cpu: ...
```

chưa được khai báo trong container.

Cách sửa:

```yaml
resources:
  requests:
    cpu: 100m
    memory: 64Mi
```

Apply lại Deployment:

```bash
kubectl apply -f 02-cpu-hpa-app.yaml
```

---

## 12.3. HPA không scale up dù CPU cao

Các nguyên nhân có thể:

- CPU chưa cao đủ lâu
- HPA chưa lấy được metric mới
- Pod bị CPU throttling do limit quá thấp
- HPA đã chạm `maxReplicas`
- Load không thực sự vào Pod
- Service selector sai
- Load generator bị lỗi
- App quá nhẹ, request không tạo đủ CPU

Kiểm tra:

```bash
kubectl get hpa -n lesson-11-autoscaling
kubectl describe hpa cpu-demo-hpa -n lesson-11-autoscaling
kubectl top pod -n lesson-11-autoscaling
kubectl get endpointslice -n lesson-11-autoscaling
kubectl logs cpu-load-generator -n lesson-11-autoscaling
```

---

## 12.4. HPA không scale down

Nguyên nhân thường gặp:

- HPA có scaleDown stabilization window
- Metric chưa giảm
- Load generator vẫn chạy
- Pod mới vẫn đang dùng CPU/memory
- HPA đang giữ `minReplicas`
- Memory app giữ memory không release

Kiểm tra:

```bash
kubectl get hpa -n lesson-11-autoscaling -w
kubectl get pod -n lesson-11-autoscaling
kubectl top pod -n lesson-11-autoscaling
kubectl describe hpa cpu-demo-hpa -n lesson-11-autoscaling
```

---

## 12.5. Pod mới bị Pending sau khi HPA scale up

Nguyên nhân thường gặp:

- Node không đủ CPU
- Node không đủ memory
- PVC không bind được
- Taint/toleration không phù hợp
- nodeSelector/nodeAffinity quá chặt
- PodSecurity hoặc admission policy chặn

Kiểm tra:

```bash
kubectl get pod -n lesson-11-autoscaling
kubectl describe pod <pod-name> -n lesson-11-autoscaling
kubectl get nodes
kubectl describe nodes
```

Nếu lỗi là:

```text
Insufficient cpu
```

Có nghĩa cluster không đủ CPU theo request để schedule thêm Pod.

---

## 12.6. HPA scale quá nhanh hoặc quá chậm

Kiểm tra phần `behavior`:

```bash
kubectl get hpa cpu-demo-hpa -n lesson-11-autoscaling -o yaml
```

Các trường cần xem:

```yaml
behavior:
  scaleUp:
  scaleDown:
```

Điều chỉnh:

- `stabilizationWindowSeconds`
- `policies`
- `selectPolicy`

---

# Phần 13: Best practices khi dùng HPA

## 13.1. Luôn khai báo requests

HPA theo CPU/memory utilization cần `resources.requests`.

Ví dụ:

```yaml
resources:
  requests:
    cpu: 100m
    memory: 128Mi
  limits:
    cpu: 500m
    memory: 512Mi
```

Không có request, autoscaling sẽ thiếu cơ sở tính toán.

---

## 13.2. Không đặt maxReplicas quá cao nếu cluster nhỏ

Trong lab 2 worker node, không nên đặt:

```yaml
maxReplicas: 100
```

Nếu HPA scale quá nhiều Pod, cluster có thể đầy tài nguyên và tạo nhiều Pod Pending.

Nên chọn maxReplicas dựa trên:

- Capacity của node
- requests của Pod
- Số lượng workload khác
- Mức tải dự kiến
- Khả năng backend/database chịu tải

---

## 13.3. Scale app không đồng nghĩa backend chịu được tải

Nếu web app scale từ 2 lên 20 Pod nhưng database vẫn chỉ chịu được 100 connection, hệ thống vẫn có thể lỗi.

Khi thiết kế autoscaling cần tính toàn bộ chuỗi:

```text
Ingress
Service
Pod
Database
Cache
Message queue
External API
Storage
```

HPA chỉ giải quyết một phần bài toán.

---

## 13.4. ReadinessProbe là bắt buộc với app production

Khi HPA scale up, Pod mới cần thời gian khởi động.

Nếu không có readinessProbe, Service có thể gửi traffic vào Pod chưa sẵn sàng.

Nên dùng:

```yaml
readinessProbe:
  httpGet:
    path: /ready
    port: 8080
  initialDelaySeconds: 5
  periodSeconds: 5
```

---

## 13.5. Không dùng CPU làm metric duy nhất cho mọi ứng dụng

CPU phù hợp với:

- web app CPU-bound
- API service xử lý logic nhiều
- worker xử lý compute

CPU không phù hợp tuyệt đối với:

- app I/O bound
- app chờ database
- app chờ external API
- queue worker cần scale theo số message
- realtime service cần scale theo connection

Với các trường hợp nâng cao, có thể cần custom metrics hoặc event-driven autoscaling.

---

## 13.6. Scale down nên thận trọng

Scale up thường cần nhanh để bảo vệ user experience.

Scale down nên chậm hơn để tránh dao động.

Ví dụ:

```yaml
behavior:
  scaleDown:
    stabilizationWindowSeconds: 300
```

Điều này giúp workload không bị scale lên/xuống liên tục khi traffic dao động.

---

# Phần 14: Lab tổng hợp cuối buổi

## 14.1. Mục tiêu

Học viên cần tự triển khai một ứng dụng autoscaling hoàn chỉnh gồm:

- Namespace riêng
- Deployment
- Service
- Ingress
- Resource requests/limits
- HPA theo CPU
- HPA behavior
- Load generator
- Quan sát scale up/scale down
- Troubleshooting nếu HPA không hoạt động

---

## 14.2. Yêu cầu bài lab

Tạo namespace:

```bash
kubectl create namespace autoscaling-final-lab
```

Triển khai app với yêu cầu:

- Tên Deployment: `final-hpa-web`
- Image: `registry.k8s.io/hpa-example`
- Replicas ban đầu: 1
- Request CPU: 100m
- Limit CPU: 500m
- Request memory: 64Mi
- Limit memory: 128Mi
- Service ClusterIP tên `final-hpa-web`
- Ingress host: `final-hpa.k8s.local`
- HPA minReplicas: 1
- HPA maxReplicas: 8
- Target CPU utilization: 50%
- Scale up cho phép tăng nhanh
- Scale down giữ stabilization window 300 giây

---

## 14.3. Manifest tham khảo

Tạo file:

```bash
vi 09-final-autoscaling-lab.yaml
```

Nội dung:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: final-hpa-web
  namespace: autoscaling-final-lab
  labels:
    app: final-hpa-web
spec:
  replicas: 1
  selector:
    matchLabels:
      app: final-hpa-web
  template:
    metadata:
      labels:
        app: final-hpa-web
    spec:
      containers:
        - name: web
          image: registry.k8s.io/hpa-example
          ports:
            - containerPort: 80
          resources:
            requests:
              cpu: 100m
              memory: 64Mi
            limits:
              cpu: 500m
              memory: 128Mi
---
apiVersion: v1
kind: Service
metadata:
  name: final-hpa-web
  namespace: autoscaling-final-lab
  labels:
    app: final-hpa-web
spec:
  type: ClusterIP
  selector:
    app: final-hpa-web
  ports:
    - name: http
      port: 80
      targetPort: 80
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: final-hpa-web
  namespace: autoscaling-final-lab
spec:
  ingressClassName: nginx
  rules:
    - host: final-hpa.k8s.local
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: final-hpa-web
                port:
                  number: 80
---
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: final-hpa-web
  namespace: autoscaling-final-lab
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: final-hpa-web
  minReplicas: 1
  maxReplicas: 8
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 50
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 0
      policies:
        - type: Percent
          value: 100
          periodSeconds: 30
        - type: Pods
          value: 2
          periodSeconds: 30
      selectPolicy: Max
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
        - type: Pods
          value: 1
          periodSeconds: 60
      selectPolicy: Min
```

Apply:

```bash
kubectl apply -f 09-final-autoscaling-lab.yaml
```

---

## 14.4. Kiểm tra toàn bộ tài nguyên

```bash
kubectl get deployment,svc,ingress,hpa -n autoscaling-final-lab
kubectl get pod -n autoscaling-final-lab -o wide
```

Kiểm tra DNS/hosts:

```text
<INGRESS_EXTERNAL_IP> final-hpa.k8s.local
```

Test:

```bash
curl http://final-hpa.k8s.local
```

---

## 14.5. Tạo tải

Từ trong cluster:

```bash
kubectl run final-load-generator \
  -n autoscaling-final-lab \
  --image=busybox:1.36 \
  --restart=Never \
  -- /bin/sh -c "while true; do wget -q -O- http://final-hpa-web; done"
```

Hoặc từ bên ngoài:

```bash
while true; do curl -s http://final-hpa.k8s.local > /dev/null; done
```

Quan sát:

```bash
kubectl get hpa -n autoscaling-final-lab -w
kubectl get deployment final-hpa-web -n autoscaling-final-lab -w
kubectl get pod -n autoscaling-final-lab -l app=final-hpa-web -o wide -w
```

---

## 14.6. Dừng tải và quan sát scale down

```bash
kubectl delete pod final-load-generator -n autoscaling-final-lab
```

Quan sát:

```bash
kubectl get hpa -n autoscaling-final-lab -w
```

Hỏi:

- Vì sao HPA không scale down ngay?
- `stabilizationWindowSeconds: 300` có tác dụng gì?
- Nếu muốn scale down nhanh hơn thì chỉnh ở đâu?
- Nếu app bị traffic spike ngắn 10 giây thì có nên scale ngay không?

---

## 14.7. Bài tập lỗi cố ý

### Lỗi 1: Xóa CPU request

Sửa Deployment, bỏ phần:

```yaml
requests:
  cpu: 100m
```

Apply lại.

Quan sát:

```bash
kubectl describe hpa final-hpa-web -n autoscaling-final-lab
```

Học viên cần tìm ra lỗi liên quan đến missing CPU request.

---

### Lỗi 2: Đặt maxReplicas quá thấp

Đặt:

```yaml
maxReplicas: 2
```

Tạo tải lớn.

Quan sát:

```bash
kubectl describe hpa final-hpa-web -n autoscaling-final-lab
```

Học viên cần nhận ra HPA bị giới hạn bởi `maxReplicas`.

---

### Lỗi 3: Service selector sai

Đổi Service selector thành:

```yaml
selector:
  app: wrong-label
```

Tạo tải qua Ingress.

Quan sát:

```bash
kubectl get endpointslice -n autoscaling-final-lab
kubectl describe ingress final-hpa-web -n autoscaling-final-lab
```

Học viên cần nhận ra traffic không vào Pod, HPA không có tải thật để scale.

---

# Phần 15: Checklist vận hành HPA

Khi triển khai HPA cho một ứng dụng, cần kiểm tra:

- Metrics Server đã hoạt động chưa?
- `kubectl top nodes` có dữ liệu không?
- `kubectl top pods` có dữ liệu không?
- Deployment có khai báo `resources.requests.cpu` không?
- Deployment có khai báo `resources.requests.memory` nếu scale theo memory không?
- HPA trỏ đúng Deployment chưa?
- `minReplicas` có phù hợp không?
- `maxReplicas` có vượt capacity cluster không?
- App có readinessProbe chưa?
- Service selector có đúng không?
- Ingress có route đúng Service không?
- Backend/database có chịu được số Pod tăng lên không?
- Có behavior scale down để tránh flapping không?
- Có dashboard/monitoring để theo dõi sau khi scale không?

---

# Phần 16: Câu hỏi kiểm tra cuối buổi

## Câu 1

HPA scale đối tượng nào trực tiếp nhất?

A. Node  
B. Service  
C. Deployment thông qua scale subresource  
D. Ingress  

Đáp án đúng: C

---

## Câu 2

Vì sao HPA theo CPU utilization cần `resources.requests.cpu`?

A. Vì request là giới hạn CPU tối đa  
B. Vì HPA dùng request làm cơ sở tính phần trăm utilization  
C. Vì kubelet không chạy nếu thiếu request  
D. Vì Service cần request để load balance  

Đáp án đúng: B

---

## Câu 3

Nếu HPA có `minReplicas: 2`, CPU rất thấp, HPA có scale xuống 1 Pod không?

A. Có  
B. Không  
C. Tùy Service type  
D. Tùy Ingress Controller  

Đáp án đúng: B

---

## Câu 4

Nếu HPA muốn scale thêm Pod nhưng Pod mới bị Pending vì `Insufficient cpu`, nguyên nhân chính là gì?

A. Service selector sai  
B. Cluster không đủ tài nguyên CPU theo request  
C. Ingress chưa có TLS  
D. ConfigMap sai key  

Đáp án đúng: B

---

## Câu 5

HPA có tự tạo thêm worker node trong cluster kubeadm on-premise không?

A. Có, mặc định Kubernetes sẽ tạo VM mới  
B. Có, nếu dùng ClusterIP  
C. Không, HPA chỉ scale workload replica  
D. Có, nếu Deployment có nhiều replica  

Đáp án đúng: C

---

## Câu 6

`scaleDown.stabilizationWindowSeconds` dùng để làm gì?

A. Tăng tốc scale down  
B. Tránh scale down quá nhanh khi metric dao động  
C. Xóa Pod lỗi ngay lập tức  
D. Tăng CPU limit cho Pod  

Đáp án đúng: B

---

## Câu 7

HPA lấy CPU/memory metric cơ bản từ đâu?

A. CoreDNS  
B. kube-proxy  
C. Metrics Server thông qua metrics.k8s.io API  
D. Ingress Controller  

Đáp án đúng: C

---

## Câu 8

Loại workload nào không phù hợp với HPA theo replica?

A. Deployment  
B. StatefulSet  
C. ReplicaSet  
D. DaemonSet  

Đáp án đúng: D

---

# Phần 17: Dọn dẹp lab

Xóa namespace lab chính:

```bash
kubectl delete namespace lesson-11-autoscaling
```

Xóa namespace lab tổng hợp:

```bash
kubectl delete namespace autoscaling-final-lab
```

Kiểm tra:

```bash
kubectl get ns
```

---

# Phần 18: Tổng kết buổi học

Trong buổi này, đã học cách Kubernetes tự động mở rộng workload bằng HPA.

Các điểm quan trọng cần nhớ:

- Scaling có nhiều loại: horizontal, vertical, node scaling
- HPA là autoscaling ở cấp workload replica
- HPA không trực tiếp tạo Pod, mà điều chỉnh replica của workload
- Metrics Server là thành phần quan trọng cho HPA CPU/memory
- `resources.requests` là nền tảng để HPA tính utilization
- CPU HPA là lab dễ quan sát nhất
- Memory HPA hoạt động được nhưng cần hiểu đặc tính memory không giảm nhanh như CPU
- HPA scale up thường nên nhanh
- HPA scale down nên thận trọng để tránh flapping
- HPA không tự động thêm node trong môi trường on-premise
- Node autoscaling cần tích hợp với hạ tầng bên dưới
- Autoscaling không thay thế capacity planning và observability

Sau buổi này, đã hiểu cách vận hành workload linh hoạt hơn khi tải tăng/giảm.

---

# Phần 19: Chuẩn bị cho buổi tiếp theo

Sau khi học autoscaling, bước tiếp theo hợp lý là:

```text
Backup, Restore và Disaster Recovery trong Kubernetes
```

Lý do:

- Ứng dụng đã biết deploy
- Đã biết expose ra ngoài
- Đã biết storage
- Đã biết security
- Đã biết troubleshooting
- Đã biết autoscaling

Tiếp theo cần học cách bảo vệ workload và dữ liệu:

- Backup manifest
- Backup namespace
- Backup etcd
- Backup/restore PVC
- Velero
- Disaster recovery mindset

---

# Tài liệu tham khảo chính thức

- Kubernetes Documentation - Autoscaling Workloads: https://kubernetes.io/docs/concepts/workloads/autoscaling/
- Kubernetes Documentation - Horizontal Pod Autoscaling: https://kubernetes.io/docs/concepts/workloads/autoscaling/horizontal-pod-autoscale/
- Kubernetes Documentation - HorizontalPodAutoscaler Walkthrough: https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale-walkthrough/
- Kubernetes Documentation - kubectl autoscale: https://kubernetes.io/docs/reference/kubectl/generated/kubectl_autoscale/
- Kubernetes Documentation - Node Autoscaling: https://kubernetes.io/docs/concepts/cluster-administration/node-autoscaling/
- Kubernetes Autoscaler GitHub Repository: https://github.com/kubernetes/autoscaler
