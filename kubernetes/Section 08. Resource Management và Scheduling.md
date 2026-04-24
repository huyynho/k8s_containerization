# BUỔI 08: RESOURCE MANAGEMENT VÀ SCHEDULING CƠ BẢN TRONG KUBERNETES v1.35

## 1. Mục tiêu của buổi học

Ở các buổi trước, đã biết cách triển khai cluster Kubernetes bằng kubeadm, tạo Pod, Deployment, Service, ConfigMap, Secret, Volume, PVC, cài Helm, MetalLB, Ingress NGINX Controller và publish ứng dụng ra ngoài bằng HTTP/HTTPS.

Đến buổi này, chúng ta bắt đầu đi vào một nhóm kiến thức rất quan trọng trong vận hành Kubernetes thực tế:

- Làm sao Kubernetes biết một Pod cần bao nhiêu CPU và RAM?
- Làm sao scheduler quyết định Pod nên chạy trên node nào?
- Vì sao có Pod bị `Pending` dù cluster vẫn còn node?
- Vì sao container bị `OOMKilled`?
- Vì sao ứng dụng bị chậm dù không bị restart?
- Vì sao một workload production không nên chạy kiểu “không requests, không limits”?
- Làm sao ép Pod chạy vào một nhóm node cụ thể?
- Làm sao ngăn Pod thường chạy vào node chuyên dụng?
- Làm sao cordon/drain node để bảo trì?

Sau buổi học này, cần nắm được:

- Khái niệm `capacity` và `allocatable` của Node.
- Ý nghĩa của `resources.requests` và `resources.limits`.
- Đơn vị CPU, memory trong Kubernetes.
- Cách Kubernetes phân loại Pod theo QoS: `Guaranteed`, `Burstable`, `BestEffort`.
- Cách kube-scheduler chọn node cho Pod.
- Cách dùng node label và `nodeSelector`.
- Cách dùng node affinity cơ bản.
- Cách dùng pod anti-affinity cơ bản để phân tán replica.
- Cách dùng taint và toleration.
- Cách dùng `cordon`, `uncordon`, `drain` để bảo trì node.
- Cách phân tích lỗi Pod bị `Pending` do thiếu resource hoặc sai rule scheduling.

---

## 2. Phạm vi kiến thức của buổi 08

### 2.1. Những kiến thức đã được phép sử dụng

Buổi này được phép sử dụng các kiến thức đã học từ buổi 01 đến buổi 07:

- Cluster, Node, Control Plane, Worker Node.
- kube-apiserver, kubelet, kube-scheduler, kube-controller-manager.
- Pod, ReplicaSet, Deployment, DaemonSet, StatefulSet, Job, CronJob.
- Namespace.
- Label và selector ở mức workload/service.
- Service `ClusterIP`, `NodePort`, Headless, ExternalName.
- ConfigMap, Secret.
- Volume, PV, PVC, StorageClass, CSI NFS.
- Helm, MetalLB, Ingress NGINX Controller.
- Readiness probe, liveness probe, startup probe.
- Rolling update, rollback, self-healing.

### 2.2. Những kiến thức mới được mở khóa trong buổi này

Buổi này sẽ mở khóa các khái niệm mới:

- Node capacity.
- Node allocatable.
- Resource request.
- Resource limit.
- CPU unit.
- Memory unit.
- Ephemeral storage request/limit ở mức giới thiệu.
- Pod QoS class.
- Scheduler filtering.
- Scheduler scoring.
- Node label cho scheduling.
- `nodeSelector`.
- `nodeAffinity`.
- `podAntiAffinity` cơ bản.
- Taint.
- Toleration.
- Cordon.
- Drain.
- Uncordon.
- LimitRange và ResourceQuota ở mức nền tảng.

### 2.3. Những nội dung chưa đi sâu trong buổi này

Để giữ đúng tiến trình kiến thức, buổi này chưa đi sâu vào:

- Horizontal Pod Autoscaler.
- Vertical Pod Autoscaler.
- Cluster Autoscaler.
- Pod Priority và Preemption nâng cao.
- Pod Disruption Budget.
- Topology Spread Constraints nâng cao.
- Scheduler Profile.
- Custom Scheduler.
- Dynamic Resource Allocation.
- Device Plugin cho GPU.
- Multi-tenancy security nâng cao.

Các nội dung này sẽ phù hợp hơn ở các buổi sau.

---

## 3. Mô hình lab sử dụng trong buổi học

Bài lab tiếp tục sử dụng cluster Kubernetes đã triển khai từ các buổi trước.

```text
+-----------------------------+
| Kubernetes Cluster v1.35     |
| kubeadm + containerd + Calico|
+-----------------------------+

Control Plane Node:
- k8s-master-01
- Ubuntu Server 24.04
- kube-apiserver
- etcd
- kube-scheduler
- kube-controller-manager
- kubelet
- containerd

Worker Node 1:
- k8s-worker-01
- Ubuntu Server 24.04
- kubelet
- containerd

Worker Node 2:
- k8s-worker-02
- Ubuntu Server 24.04
- kubelet
- containerd

NFS Server:
- nfs-server-01
- Ubuntu Server 24.04
- Đã dùng trong buổi 05 cho NFS CSI
```

Trong buổi này, NFS Server không phải thành phần trọng tâm. Nó vẫn tồn tại trong môi trường lab, nhưng nội dung chính tập trung vào resource và scheduling.

Kiểm tra trạng thái cluster trước khi bắt đầu:

```bash
kubectl get nodes -o wide
```

Kết quả mong đợi:

```text
NAME            STATUS   ROLES           AGE   VERSION   INTERNAL-IP
k8s-master-01   Ready    control-plane    ...   v1.35.x   10.10.10.11
k8s-worker-01   Ready    <none>           ...   v1.35.x   10.10.10.21
k8s-worker-02   Ready    <none>           ...   v1.35.x   10.10.10.22
```

---

# PHẦN 1: RESOURCE MANAGEMENT TRONG KUBERNETES

---

## 4. Vì sao cần Resource Management?

Khi chạy container thủ công bằng Docker hoặc containerd, nếu không giới hạn tài nguyên, container có thể dùng rất nhiều CPU hoặc RAM của host.

Trong Kubernetes, nếu nhiều Pod cùng chạy trên một node nhưng không khai báo tài nguyên rõ ràng, cluster sẽ gặp nhiều vấn đề:

- Scheduler không biết Pod thật sự cần bao nhiêu tài nguyên.
- Pod có thể được xếp quá nhiều vào một node.
- Một container dùng quá nhiều memory có thể làm node bị áp lực.
- Ứng dụng quan trọng có thể bị ảnh hưởng bởi ứng dụng kém quan trọng.
- Khi node thiếu tài nguyên, kubelet phải evict Pod.
- Khi container vượt memory limit, container có thể bị `OOMKilled`.

Resource Management giúp Kubernetes trả lời 2 câu hỏi:

```text
Pod này cần tối thiểu bao nhiêu tài nguyên để được schedule?
Pod này được phép dùng tối đa bao nhiêu tài nguyên khi chạy?
```

Hai câu hỏi này tương ứng với:

```yaml
resources:
  requests:
    cpu: ...
    memory: ...
  limits:
    cpu: ...
    memory: ...
```

---

## 5. Node Capacity và Node Allocatable

Mỗi node trong Kubernetes có một lượng tài nguyên vật lý hoặc ảo hóa nhất định.

Ví dụ một worker node có:

```text
CPU:    2 vCPU
Memory: 4 GiB RAM
Disk:   50 GiB
```

Nhưng Kubernetes không thể cấp toàn bộ tài nguyên đó cho Pod, vì node còn cần chạy:

- Operating system.
- kubelet.
- containerd.
- kube-proxy.
- Calico agent.
- Các system daemon khác.

Vì vậy Kubernetes phân biệt:

### Capacity

`Capacity` là tổng tài nguyên mà node có.

### Allocatable

`Allocatable` là phần tài nguyên có thể cấp cho Pod.

Kiểm tra thông tin node:

```bash
kubectl describe node k8s-worker-01
```

Tìm các phần:

```text
Capacity:
  cpu:                2
  memory:             4023892Ki
  pods:               110

Allocatable:
  cpu:                2
  memory:             3921492Ki
  pods:               110
```

Trong thực tế production, có thể cấu hình để kubelet reserve tài nguyên cho system và Kubernetes components. Nội dung này sẽ được nhắc lại trong phần vận hành nâng cao.

---

## 6. Resource Requests

`requests` là lượng tài nguyên tối thiểu mà Kubernetes dùng để quyết định scheduling.

Ví dụ:

```yaml
resources:
  requests:
    cpu: "100m"
    memory: "128Mi"
```

Ý nghĩa:

```text
Container này cần tối thiểu 100 millicpu và 128 MiB RAM để chạy ổn định.
```

Khi scheduler xếp Pod lên node, scheduler sẽ kiểm tra node còn đủ tài nguyên `allocatable` chưa được request hay không.

Điểm cực kỳ quan trọng:

```text
Scheduler dựa vào requests, không dựa vào mức sử dụng thực tế tại thời điểm đó.
```

Ví dụ:

```text
Node có 2 CPU allocatable.
Đã có 4 Pod, mỗi Pod request 500m CPU.
Tổng request = 2000m = 2 CPU.

Dù thực tế 4 Pod đó đang idle, scheduler vẫn xem node này đã hết CPU để schedule thêm Pod mới có CPU request.
```

Đây là lý do sizing `requests` rất quan trọng.

---

## 7. Resource Limits

`limits` là lượng tài nguyên tối đa container được phép sử dụng.

Ví dụ:

```yaml
resources:
  limits:
    cpu: "500m"
    memory: "256Mi"
```

Ý nghĩa:

```text
Container này được phép dùng tối đa 500 millicpu và 256 MiB RAM.
```

Tuy nhiên, CPU limit và memory limit có hành vi khác nhau.

### 7.1. CPU limit

CPU là tài nguyên có thể chia sẻ theo thời gian.

Nếu container vượt CPU limit, nó thường không bị kill ngay. Thay vào đó, container bị throttle.

```text
CPU throttle = container bị làm chậm lại vì vượt mức CPU được phép dùng.
```

Ví dụ:

```text
Ứng dụng cần burst lên 2 CPU trong vài giây.
Nhưng limit chỉ là 500m.
Ứng dụng không chết, nhưng phản hồi có thể chậm bất thường.
```

Trong production, đặt CPU limit quá thấp có thể làm ứng dụng bị chậm khó hiểu.

### 7.2. Memory limit

Memory khác CPU. Memory không thể throttle theo cách giống CPU.

Nếu container vượt memory limit, container có thể bị kill và trạng thái thường thấy là:

```text
OOMKilled
```

Ví dụ:

```text
Container limit memory = 256Mi.
Ứng dụng dùng đến 300Mi.
Kernel/cgroup kill process.
Kubernetes ghi nhận container bị OOMKilled.
```

Kiểm tra Pod bị OOMKilled:

```bash
kubectl describe pod <pod-name> -n <namespace>
```

Tìm phần:

```text
Last State: Terminated
Reason:     OOMKilled
Exit Code:  137
```

---

## 8. Đơn vị CPU trong Kubernetes

CPU trong Kubernetes có thể khai báo bằng đơn vị core hoặc millicore.

Ví dụ:

```yaml
cpu: "1"
```

Nghĩa là 1 CPU core.

```yaml
cpu: "500m"
```

Nghĩa là 500 millicpu, tương đương 0.5 CPU core.

```yaml
cpu: "100m"
```

Nghĩa là 0.1 CPU core.

Bảng tham khảo:

| Giá trị | Ý nghĩa |
|---|---|
| `1` | 1 CPU core |
| `500m` | 0.5 CPU core |
| `250m` | 0.25 CPU core |
| `100m` | 0.1 CPU core |
| `50m` | 0.05 CPU core |

Trong lab, chúng ta thường dùng:

```yaml
cpu: "100m"
cpu: "250m"
cpu: "500m"
```

---

## 9. Đơn vị Memory trong Kubernetes

Memory thường dùng các đơn vị:

```text
Ki, Mi, Gi
```

Ví dụ:

```yaml
memory: "128Mi"
memory: "256Mi"
memory: "1Gi"
```

Trong đó:

```text
1Gi = 1024Mi
1Mi = 1024Ki
```

Cần phân biệt rất kỹ:

```yaml
memory: "400Mi"
```

với:

```yaml
memory: "400m"
```

`400Mi` là 400 mebibytes.

`400m` trong memory là 0.4 bytes, gần như chắc chắn là cấu hình sai.

Trong bài giảng và lab, nên dùng rõ ràng:

```yaml
memory: "128Mi"
memory: "256Mi"
memory: "512Mi"
memory: "1Gi"
```

---

## 10. Manifest Deployment có requests và limits

Tạo namespace cho buổi học:

```bash
kubectl create namespace lab08-resource-scheduling
```

Tạo file:

```bash
nano 01-deployment-with-resources.yaml
```

Nội dung:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-resource-demo
  namespace: lab08-resource-scheduling
  labels:
    app: web-resource-demo
spec:
  replicas: 2
  selector:
    matchLabels:
      app: web-resource-demo
  template:
    metadata:
      labels:
        app: web-resource-demo
    spec:
      containers:
        - name: nginx
          image: nginx:1.27-alpine
          ports:
            - containerPort: 80
          resources:
            requests:
              cpu: "100m"
              memory: "128Mi"
            limits:
              cpu: "500m"
              memory: "256Mi"
```

Apply manifest:

```bash
kubectl apply -f 01-deployment-with-resources.yaml
```

Kiểm tra Pod:

```bash
kubectl get pods -n lab08-resource-scheduling -o wide
```

Kiểm tra Deployment:

```bash
kubectl get deployment -n lab08-resource-scheduling
```

---

## 11. Giải thích manifest Deployment có resources

### apiVersion

```yaml
apiVersion: apps/v1
```

Deployment thuộc API group `apps/v1`.

### kind

```yaml
kind: Deployment
```

Khai báo object là Deployment.

### metadata.name

```yaml
metadata:
  name: web-resource-demo
```

Tên Deployment là `web-resource-demo`.

### metadata.namespace

```yaml
namespace: lab08-resource-scheduling
```

Deployment được tạo trong namespace riêng của buổi học.

### replicas

```yaml
replicas: 2
```

Deployment cần duy trì 2 Pod replica.

### selector

```yaml
selector:
  matchLabels:
    app: web-resource-demo
```

Deployment quản lý các Pod có label:

```yaml
app: web-resource-demo
```

### template.metadata.labels

```yaml
template:
  metadata:
    labels:
      app: web-resource-demo
```

Pod được tạo ra sẽ có label `app=web-resource-demo`.

Label này phải khớp với selector của Deployment.

### containers.name

```yaml
containers:
  - name: nginx
```

Pod có một container tên `nginx`.

### image

```yaml
image: nginx:1.27-alpine
```

Container dùng image `nginx:1.27-alpine`.

### ports.containerPort

```yaml
ports:
  - containerPort: 80
```

Khai báo container lắng nghe port 80.

Lưu ý:

```text
containerPort chỉ là metadata để mô tả port ứng dụng trong container.
Nó không tự expose ứng dụng ra ngoài cluster.
```

### resources.requests.cpu

```yaml
resources:
  requests:
    cpu: "100m"
```

Container request 100 millicpu.

Scheduler sẽ dùng giá trị này để tính xem node nào còn đủ CPU để chạy Pod.

### resources.requests.memory

```yaml
memory: "128Mi"
```

Container request 128Mi memory.

Scheduler sẽ dùng giá trị này để tính xem node nào còn đủ memory để chạy Pod.

### resources.limits.cpu

```yaml
limits:
  cpu: "500m"
```

Container được phép dùng tối đa 500 millicpu.

Nếu vượt mức này, container có thể bị CPU throttling.

### resources.limits.memory

```yaml
memory: "256Mi"
```

Container được phép dùng tối đa 256Mi memory.

Nếu vượt mức này, container có thể bị `OOMKilled`.

---

## 12. Kiểm tra resource request của Pod

Xem Pod được schedule vào node nào:

```bash
kubectl get pods -n lab08-resource-scheduling -o wide
```

Xem chi tiết Pod:

```bash
kubectl describe pod <pod-name> -n lab08-resource-scheduling
```

Tìm phần:

```text
Containers:
  nginx:
    Requests:
      cpu:        100m
      memory:     128Mi
    Limits:
      cpu:        500m
      memory:     256Mi
```

Xem resource đang được request trên node:

```bash
kubectl describe node k8s-worker-01
```

Tìm phần:

```text
Allocated resources:
  Resource           Requests      Limits
  cpu                ...           ...
  memory             ...           ...
```

Lưu ý:

```text
Allocated resources là tổng requests/limits đã được khai báo bởi các Pod trên node.
Nó không nhất thiết phản ánh usage thực tế tại thời điểm đó.
```

---

## 13. Pod QoS Class

Kubernetes phân loại Pod thành 3 QoS class:

- `Guaranteed`
- `Burstable`
- `BestEffort`

QoS class ảnh hưởng đến thứ tự Pod có thể bị evict khi node bị áp lực tài nguyên.

### 13.1. Guaranteed

Pod thuộc QoS `Guaranteed` khi:

- Mọi container đều khai báo CPU request và CPU limit.
- Mọi container đều khai báo memory request và memory limit.
- CPU request bằng CPU limit.
- Memory request bằng memory limit.

Ví dụ:

```yaml
resources:
  requests:
    cpu: "500m"
    memory: "256Mi"
  limits:
    cpu: "500m"
    memory: "256Mi"
```

Đây là class ổn định nhất, ít bị evict nhất khi node pressure.

### 13.2. Burstable

Pod thuộc QoS `Burstable` khi:

- Có ít nhất một request hoặc limit CPU/memory.
- Nhưng không đủ điều kiện để thành `Guaranteed`.

Ví dụ:

```yaml
resources:
  requests:
    cpu: "100m"
    memory: "128Mi"
  limits:
    cpu: "500m"
    memory: "256Mi"
```

Đây là kiểu rất phổ biến cho workload thông thường.

### 13.3. BestEffort

Pod thuộc QoS `BestEffort` khi:

- Không khai báo CPU request.
- Không khai báo CPU limit.
- Không khai báo memory request.
- Không khai báo memory limit.

Ví dụ:

```yaml
containers:
  - name: nginx
    image: nginx:1.27-alpine
```

Pod `BestEffort` dễ bị evict nhất khi node thiếu tài nguyên.

---

## 14. Lab: Quan sát QoS Class của Pod

Tạo file:

```bash
nano 02-qos-examples.yaml
```

Nội dung:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: qos-besteffort
  namespace: lab08-resource-scheduling
spec:
  containers:
    - name: nginx
      image: nginx:1.27-alpine
      ports:
        - containerPort: 80
---
apiVersion: v1
kind: Pod
metadata:
  name: qos-burstable
  namespace: lab08-resource-scheduling
spec:
  containers:
    - name: nginx
      image: nginx:1.27-alpine
      ports:
        - containerPort: 80
      resources:
        requests:
          cpu: "100m"
          memory: "128Mi"
        limits:
          cpu: "500m"
          memory: "256Mi"
---
apiVersion: v1
kind: Pod
metadata:
  name: qos-guaranteed
  namespace: lab08-resource-scheduling
spec:
  containers:
    - name: nginx
      image: nginx:1.27-alpine
      ports:
        - containerPort: 80
      resources:
        requests:
          cpu: "250m"
          memory: "256Mi"
        limits:
          cpu: "250m"
          memory: "256Mi"
```

Apply:

```bash
kubectl apply -f 02-qos-examples.yaml
```

Kiểm tra QoS:

```bash
kubectl get pod qos-besteffort -n lab08-resource-scheduling -o jsonpath='{.status.qosClass}{"\n"}'
kubectl get pod qos-burstable -n lab08-resource-scheduling -o jsonpath='{.status.qosClass}{"\n"}'
kubectl get pod qos-guaranteed -n lab08-resource-scheduling -o jsonpath='{.status.qosClass}{"\n"}'
```

Kết quả mong đợi:

```text
BestEffort
Burstable
Guaranteed
```

---

## 15. Resource request quá lớn và lỗi Pending

Một lỗi rất hay gặp là Pod request nhiều tài nguyên hơn khả năng còn lại của cluster.

Tạo file:

```bash
nano 03-pod-too-large.yaml
```

Nội dung:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-too-large
  namespace: lab08-resource-scheduling
spec:
  containers:
    - name: nginx
      image: nginx:1.27-alpine
      resources:
        requests:
          cpu: "100"
          memory: "100Gi"
        limits:
          cpu: "100"
          memory: "100Gi"
```

Apply:

```bash
kubectl apply -f 03-pod-too-large.yaml
```

Kiểm tra trạng thái:

```bash
kubectl get pod pod-too-large -n lab08-resource-scheduling
```

Kết quả thường thấy:

```text
NAME            READY   STATUS    RESTARTS   AGE
pod-too-large   0/1     Pending   0          ...
```

Describe Pod:

```bash
kubectl describe pod pod-too-large -n lab08-resource-scheduling
```

Tìm phần Events:

```text
Warning  FailedScheduling  default-scheduler  0/3 nodes are available: insufficient cpu, insufficient memory.
```

Ý nghĩa:

```text
Scheduler không tìm thấy node nào đủ tài nguyên theo requests của Pod.
Pod chưa được bind vào node nào nên nằm ở Pending.
```

Xóa Pod sau khi quan sát:

```bash
kubectl delete pod pod-too-large -n lab08-resource-scheduling
```

---

## 16. Lưu ý về In-place Pod Resize trong Kubernetes v1.35

Từ Kubernetes v1.35, tính năng In-place Pod Resize đã ở trạng thái stable.

Ý tưởng của tính năng này là cho phép điều chỉnh CPU/memory requests và limits của container trong một Pod đang chạy mà không nhất thiết phải xóa và tạo lại Pod.

Tuy nhiên, trong giáo trình này, chúng ta chưa đưa In-place Pod Resize vào lab chính vì:

- Đây là nội dung nâng cao hơn so với mục tiêu buổi 08.
- Cần hiểu kỹ `/resize` subresource.
- Cần hiểu rõ runtime và kubelet xử lý resize như thế nào.
- Với workload production, đa số vận hành vẫn thay đổi resource qua Deployment template và rollout.

Trong buổi này, chỉ cần nhớ:

```text
Kubernetes v1.35 có hỗ trợ In-place Pod Resize ở trạng thái stable, nhưng lab chính vẫn dùng cách khai báo resources trong manifest workload.
```

---

# PHẦN 2: KUBE-SCHEDULER VÀ CÁCH POD ĐƯỢC XẾP VÀO NODE

---

## 17. kube-scheduler làm gì?

Khi tạo Pod, ban đầu Pod chưa có node.

```text
Pod created -> chưa có .spec.nodeName -> scheduler quan sát -> chọn node phù hợp -> bind Pod vào node
```

kube-scheduler là thành phần control plane chịu trách nhiệm chọn node cho Pod chưa được schedule.

Luồng đơn giản:

```text
User apply manifest
        |
        v
kube-apiserver lưu Pod object
        |
        v
kube-scheduler thấy Pod chưa có node
        |
        v
Scheduler lọc node phù hợp
        |
        v
Scheduler chấm điểm node phù hợp
        |
        v
Scheduler bind Pod vào node được chọn
        |
        v
kubelet trên node đó tạo container
```

---

## 18. Scheduler Filtering và Scoring

Scheduler hoạt động theo 2 bước chính:

### 18.1. Filtering

Filtering là bước loại bỏ các node không phù hợp.

Ví dụ node bị loại nếu:

- Không đủ CPU request.
- Không đủ memory request.
- Không match `nodeSelector`.
- Không thỏa node affinity required.
- Có taint mà Pod không có toleration phù hợp.
- Node đang `SchedulingDisabled` do bị cordon.

Sau filtering, scheduler có danh sách các node khả thi.

Nếu danh sách rỗng:

```text
Pod Pending
```

### 18.2. Scoring

Scoring là bước chấm điểm các node còn lại.

Ví dụ scheduler có thể ưu tiên node:

- Ít tải hơn.
- Phù hợp với preferred node affinity hơn.
- Có image đã được pull sẵn.
- Phân bổ Pod hợp lý hơn.

Node có điểm cao nhất sẽ được chọn.

Nếu nhiều node có điểm bằng nhau, scheduler có thể chọn một node trong số đó.

---

## 19. Không nên dùng nodeName trong workload thông thường

Pod có trường:

```yaml
spec:
  nodeName: k8s-worker-01
```

Trường này ép Pod chạy trên một node cụ thể và bỏ qua scheduler.

Trong vận hành thông thường, không nên dùng `nodeName` vì:

- Bỏ qua logic scheduler.
- Có thể làm Pod fail nếu node không đủ tài nguyên.
- Không linh hoạt khi node maintenance.
- Không phù hợp với tư duy declarative và tự động hóa của Kubernetes.

Nên dùng:

- `nodeSelector` cho trường hợp đơn giản.
- `nodeAffinity` cho trường hợp cần rule linh hoạt hơn.
- `taints/tolerations` để dành node cho workload đặc biệt.

---

# PHẦN 3: NODE LABEL VÀ NODESELECTOR

---

## 20. Node label là gì?

Node cũng có label giống Pod, Deployment, Service.

Xem label của node:

```bash
kubectl get nodes --show-labels
```

Output có thể rất dài.

Xem gọn hơn:

```bash
kubectl get nodes -L kubernetes.io/hostname
```

Thêm label cho worker node:

```bash
kubectl label node k8s-worker-01 nodepool=app
kubectl label node k8s-worker-02 nodepool=infra
```

Kiểm tra:

```bash
kubectl get nodes -L nodepool
```

Kết quả mong đợi:

```text
NAME            STATUS   ROLES           AGE   VERSION   NODEPOOL
k8s-master-01   Ready    control-plane    ...   v1.35.x
k8s-worker-01   Ready    <none>           ...   v1.35.x   app
k8s-worker-02   Ready    <none>           ...   v1.35.x   infra
```

---

## 21. nodeSelector

`nodeSelector` là cách đơn giản nhất để yêu cầu Pod chạy trên node có label cụ thể.

Ví dụ:

```yaml
nodeSelector:
  nodepool: app
```

Ý nghĩa:

```text
Pod chỉ được schedule lên node có label nodepool=app.
```

Nếu không có node nào thỏa điều kiện, Pod sẽ `Pending`.

---

## 22. Manifest Deployment dùng nodeSelector

Tạo file:

```bash
nano 04-deployment-nodeselector.yaml
```

Nội dung:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-nodeselector-demo
  namespace: lab08-resource-scheduling
  labels:
    app: web-nodeselector-demo
spec:
  replicas: 2
  selector:
    matchLabels:
      app: web-nodeselector-demo
  template:
    metadata:
      labels:
        app: web-nodeselector-demo
    spec:
      nodeSelector:
        nodepool: app
      containers:
        - name: nginx
          image: nginx:1.27-alpine
          ports:
            - containerPort: 80
          resources:
            requests:
              cpu: "100m"
              memory: "128Mi"
            limits:
              cpu: "500m"
              memory: "256Mi"
```

Apply:

```bash
kubectl apply -f 04-deployment-nodeselector.yaml
```

Kiểm tra Pod:

```bash
kubectl get pods -n lab08-resource-scheduling -l app=web-nodeselector-demo -o wide
```

Kết quả mong đợi:

```text
Các Pod được schedule vào k8s-worker-01 vì node này có label nodepool=app.
```

---

## 23. Giải thích phần nodeSelector trong manifest

Phần quan trọng:

```yaml
spec:
  nodeSelector:
    nodepool: app
```

Lưu ý vị trí:

```text
nodeSelector nằm trong spec của Pod template.
```

Trong Deployment, Pod template nằm tại:

```yaml
spec:
  template:
    spec:
      nodeSelector:
        nodepool: app
```

Không đặt `nodeSelector` ở cấp Deployment spec bên ngoài.

Sai:

```yaml
spec:
  replicas: 2
  nodeSelector:
    nodepool: app
```

Đúng:

```yaml
spec:
  template:
    spec:
      nodeSelector:
        nodepool: app
```

Vì scheduler schedule Pod, không schedule trực tiếp Deployment.

---

## 24. Lab lỗi: nodeSelector không match node nào

Tạo file:

```bash
nano 05-deployment-bad-nodeselector.yaml
```

Nội dung:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-bad-nodeselector
  namespace: lab08-resource-scheduling
spec:
  replicas: 1
  selector:
    matchLabels:
      app: web-bad-nodeselector
  template:
    metadata:
      labels:
        app: web-bad-nodeselector
    spec:
      nodeSelector:
        nodepool: gpu
      containers:
        - name: nginx
          image: nginx:1.27-alpine
          resources:
            requests:
              cpu: "100m"
              memory: "128Mi"
            limits:
              cpu: "500m"
              memory: "256Mi"
```

Apply:

```bash
kubectl apply -f 05-deployment-bad-nodeselector.yaml
```

Kiểm tra:

```bash
kubectl get pods -n lab08-resource-scheduling -l app=web-bad-nodeselector
```

Pod sẽ `Pending`.

Describe Pod:

```bash
kubectl describe pod <pod-name> -n lab08-resource-scheduling
```

Có thể thấy event tương tự:

```text
0/3 nodes are available: node(s) didn't match Pod's node affinity/selector.
```

Ý nghĩa:

```text
Không có node nào có label nodepool=gpu.
Scheduler không thể bind Pod vào node.
```

Xóa Deployment lỗi:

```bash
kubectl delete deployment web-bad-nodeselector -n lab08-resource-scheduling
```

---

# PHẦN 4: NODE AFFINITY CƠ BẢN

---

## 25. nodeAffinity là gì?

`nodeAffinity` cũng dùng để điều khiển Pod chạy trên node nào dựa vào node label, nhưng mạnh hơn `nodeSelector`.

So sánh nhanh:

| Cơ chế | Đặc điểm |
|---|---|
| `nodeSelector` | Đơn giản, chỉ match label dạng key=value |
| `nodeAffinity` | Linh hoạt hơn, hỗ trợ toán tử `In`, `NotIn`, `Exists`, `DoesNotExist`, `Gt`, `Lt` |

Node affinity có hai kiểu thường gặp:

### requiredDuringSchedulingIgnoredDuringExecution

Đây là rule bắt buộc.

```text
Nếu không có node nào thỏa rule, Pod sẽ Pending.
```

### preferredDuringSchedulingIgnoredDuringExecution

Đây là rule ưu tiên.

```text
Scheduler cố gắng chọn node thỏa rule, nhưng nếu không có thì vẫn có thể schedule Pod lên node khác.
```

Cụm từ `IgnoredDuringExecution` nghĩa là:

```text
Nếu node label thay đổi sau khi Pod đã được schedule, Kubernetes không tự động đuổi Pod khỏi node đó chỉ vì rule không còn đúng.
```

---

## 26. Manifest Deployment dùng required nodeAffinity

Tạo thêm label cho node:

```bash
kubectl label node k8s-worker-01 disk=ssd
kubectl label node k8s-worker-02 disk=hdd
```

Kiểm tra:

```bash
kubectl get nodes -L nodepool,disk
```

Tạo file:

```bash
nano 06-deployment-required-node-affinity.yaml
```

Nội dung:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-required-affinity
  namespace: lab08-resource-scheduling
  labels:
    app: web-required-affinity
spec:
  replicas: 2
  selector:
    matchLabels:
      app: web-required-affinity
  template:
    metadata:
      labels:
        app: web-required-affinity
    spec:
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
              - matchExpressions:
                  - key: disk
                    operator: In
                    values:
                      - ssd
      containers:
        - name: nginx
          image: nginx:1.27-alpine
          ports:
            - containerPort: 80
          resources:
            requests:
              cpu: "100m"
              memory: "128Mi"
            limits:
              cpu: "500m"
              memory: "256Mi"
```

Apply:

```bash
kubectl apply -f 06-deployment-required-node-affinity.yaml
```

Kiểm tra:

```bash
kubectl get pods -n lab08-resource-scheduling -l app=web-required-affinity -o wide
```

Kết quả mong đợi:

```text
Pod được schedule vào node có label disk=ssd.
```

---

## 27. Giải thích nodeAffinity required

Phần quan trọng:

```yaml
affinity:
  nodeAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
      nodeSelectorTerms:
        - matchExpressions:
            - key: disk
              operator: In
              values:
                - ssd
```

Giải thích:

```yaml
affinity:
```

Khai báo các rule affinity/anti-affinity cho Pod.

```yaml
nodeAffinity:
```

Rule liên quan đến node.

```yaml
requiredDuringSchedulingIgnoredDuringExecution:
```

Điều kiện bắt buộc khi scheduling.

```yaml
nodeSelectorTerms:
```

Danh sách các nhóm điều kiện chọn node.

```yaml
matchExpressions:
```

Danh sách biểu thức match label.

```yaml
key: disk
```

Kiểm tra label key `disk` trên node.

```yaml
operator: In
```

Giá trị label phải nằm trong danh sách `values`.

```yaml
values:
  - ssd
```

Node phải có:

```text
disk=ssd
```

---

## 28. Manifest Deployment dùng preferred nodeAffinity

Tạo file:

```bash
nano 07-deployment-preferred-node-affinity.yaml
```

Nội dung:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-preferred-affinity
  namespace: lab08-resource-scheduling
  labels:
    app: web-preferred-affinity
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web-preferred-affinity
  template:
    metadata:
      labels:
        app: web-preferred-affinity
    spec:
      affinity:
        nodeAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
            - weight: 80
              preference:
                matchExpressions:
                  - key: nodepool
                    operator: In
                    values:
                      - app
      containers:
        - name: nginx
          image: nginx:1.27-alpine
          ports:
            - containerPort: 80
          resources:
            requests:
              cpu: "100m"
              memory: "128Mi"
            limits:
              cpu: "500m"
              memory: "256Mi"
```

Apply:

```bash
kubectl apply -f 07-deployment-preferred-node-affinity.yaml
```

Kiểm tra:

```bash
kubectl get pods -n lab08-resource-scheduling -l app=web-preferred-affinity -o wide
```

Giải thích:

```text
Scheduler sẽ ưu tiên node có nodepool=app.
Nhưng nếu node đó không đủ tài nguyên, scheduler vẫn có thể chọn node khác.
```

---

# PHẦN 5: POD ANTI-AFFINITY CƠ BẢN

---

## 29. Vì sao cần Pod Anti-Affinity?

Giả sử một ứng dụng có 2 replica.

Nếu cả 2 replica đều chạy trên cùng một worker node, khi node đó lỗi thì toàn bộ ứng dụng có thể bị gián đoạn.

Pod anti-affinity giúp yêu cầu hoặc khuyến nghị Kubernetes không đặt các Pod giống nhau trên cùng một topology domain.

Trong lab này, topology domain đơn giản nhất là node hostname:

```yaml
topologyKey: kubernetes.io/hostname
```

Nghĩa là:

```text
Cố gắng không đặt các Pod cùng app trên cùng một node.
```

---

## 30. Manifest Deployment dùng preferred podAntiAffinity

Tạo file:

```bash
nano 08-deployment-pod-anti-affinity.yaml
```

Nội dung:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-anti-affinity
  namespace: lab08-resource-scheduling
  labels:
    app: web-anti-affinity
spec:
  replicas: 2
  selector:
    matchLabels:
      app: web-anti-affinity
  template:
    metadata:
      labels:
        app: web-anti-affinity
    spec:
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
            - weight: 100
              podAffinityTerm:
                labelSelector:
                  matchExpressions:
                    - key: app
                      operator: In
                      values:
                        - web-anti-affinity
                topologyKey: kubernetes.io/hostname
      containers:
        - name: nginx
          image: nginx:1.27-alpine
          ports:
            - containerPort: 80
          resources:
            requests:
              cpu: "100m"
              memory: "128Mi"
            limits:
              cpu: "500m"
              memory: "256Mi"
```

Apply:

```bash
kubectl apply -f 08-deployment-pod-anti-affinity.yaml
```

Kiểm tra:

```bash
kubectl get pods -n lab08-resource-scheduling -l app=web-anti-affinity -o wide
```

Kỳ vọng:

```text
Hai Pod có xu hướng được phân tán sang hai worker node khác nhau.
```

Lưu ý:

```text
Vì đây là preferred rule nên scheduler cố gắng phân tán, nhưng không bắt buộc tuyệt đối.
```

---

## 31. Giải thích podAntiAffinity

Phần quan trọng:

```yaml
affinity:
  podAntiAffinity:
    preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 100
        podAffinityTerm:
          labelSelector:
            matchExpressions:
              - key: app
                operator: In
                values:
                  - web-anti-affinity
          topologyKey: kubernetes.io/hostname
```

Giải thích:

```yaml
podAntiAffinity:
```

Khai báo rule không muốn Pod nằm gần các Pod khác có label nhất định.

```yaml
preferredDuringSchedulingIgnoredDuringExecution:
```

Rule ưu tiên, không bắt buộc.

```yaml
weight: 100
```

Độ ưu tiên cao nhất trong thang 1 đến 100.

```yaml
labelSelector:
```

Chọn các Pod hiện có để so sánh.

```yaml
key: app
operator: In
values:
  - web-anti-affinity
```

Rule áp dụng với Pod có label:

```text
app=web-anti-affinity
```

```yaml
topologyKey: kubernetes.io/hostname
```

Topology domain là hostname của node.

Nói đơn giản:

```text
Scheduler sẽ cố gắng không đặt 2 Pod app=web-anti-affinity trên cùng một node.
```

---

# PHẦN 6: TAINT VÀ TOLERATION

---

## 32. Taint và Toleration là gì?

Nếu `nodeSelector` và `nodeAffinity` là cách Pod “chọn” node, thì taint là cách node “từ chối” Pod.

Tư duy dễ nhớ:

```text
nodeSelector / nodeAffinity: Pod hút về node phù hợp.
taint: Node đẩy Pod không phù hợp ra xa.
toleration: Pod chịu được taint đó.
```

Một node có taint sẽ không nhận Pod, trừ khi Pod có toleration phù hợp.

Ví dụ:

```bash
kubectl taint nodes k8s-worker-02 dedicated=infra:NoSchedule
```

Ý nghĩa:

```text
Node k8s-worker-02 chỉ nhận Pod nào tolerates taint dedicated=infra:NoSchedule.
```

---

## 33. Các effect phổ biến của taint

### NoSchedule

```text
Pod mới không được schedule lên node nếu không có toleration phù hợp.
Pod cũ đang chạy không bị đuổi khỏi node.
```

### PreferNoSchedule

```text
Scheduler cố gắng không schedule Pod lên node đó, nhưng không tuyệt đối.
```

### NoExecute

```text
Pod mới không được schedule lên node.
Pod đang chạy cũng có thể bị evict nếu không có toleration phù hợp.
```

Trong lab cơ bản, chúng ta dùng `NoSchedule`.

---

## 34. Taint mặc định trên control plane node

Cluster kubeadm thường taint control plane node để workload thường không chạy trên master/control plane.

Kiểm tra:

```bash
kubectl describe node k8s-master-01 | grep -i taints
```

Có thể thấy:

```text
Taints: node-role.kubernetes.io/control-plane:NoSchedule
```

Ý nghĩa:

```text
Pod ứng dụng thông thường không chạy trên control plane node, trừ khi có toleration phù hợp.
```

Đây là lý do cluster 1 master + 2 worker thường chỉ schedule workload lên worker node.

---

## 35. Lab: Taint một worker node

Gắn taint cho worker-02:

```bash
kubectl taint nodes k8s-worker-02 dedicated=infra:NoSchedule
```

Kiểm tra:

```bash
kubectl describe node k8s-worker-02 | grep -i taints
```

Kết quả:

```text
Taints: dedicated=infra:NoSchedule
```

---

## 36. Lab lỗi: Pod muốn vào node bị taint nhưng không có toleration

Tạo file:

```bash
nano 09-pod-taint-without-toleration.yaml
```

Nội dung:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-without-toleration
  namespace: lab08-resource-scheduling
spec:
  nodeSelector:
    nodepool: infra
  containers:
    - name: nginx
      image: nginx:1.27-alpine
      resources:
        requests:
          cpu: "100m"
          memory: "128Mi"
        limits:
          cpu: "500m"
          memory: "256Mi"
```

Apply:

```bash
kubectl apply -f 09-pod-taint-without-toleration.yaml
```

Kiểm tra:

```bash
kubectl get pod pod-without-toleration -n lab08-resource-scheduling
```

Pod sẽ `Pending`.

Describe:

```bash
kubectl describe pod pod-without-toleration -n lab08-resource-scheduling
```

Event có thể thấy:

```text
node(s) had untolerated taint {dedicated: infra}
```

Giải thích:

```text
Pod yêu cầu chạy trên nodepool=infra.
k8s-worker-02 có nodepool=infra nhưng lại bị taint dedicated=infra:NoSchedule.
Pod không có toleration nên scheduler không được đặt Pod vào node này.
```

Xóa Pod lỗi:

```bash
kubectl delete pod pod-without-toleration -n lab08-resource-scheduling
```

---

## 37. Manifest Pod có toleration phù hợp

Tạo file:

```bash
nano 10-pod-with-toleration.yaml
```

Nội dung:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-with-toleration
  namespace: lab08-resource-scheduling
spec:
  nodeSelector:
    nodepool: infra
  tolerations:
    - key: "dedicated"
      operator: "Equal"
      value: "infra"
      effect: "NoSchedule"
  containers:
    - name: nginx
      image: nginx:1.27-alpine
      resources:
        requests:
          cpu: "100m"
          memory: "128Mi"
        limits:
          cpu: "500m"
          memory: "256Mi"
```

Apply:

```bash
kubectl apply -f 10-pod-with-toleration.yaml
```

Kiểm tra:

```bash
kubectl get pod pod-with-toleration -n lab08-resource-scheduling -o wide
```

Kết quả mong đợi:

```text
Pod chạy trên k8s-worker-02.
```

---

## 38. Giải thích toleration

Phần quan trọng:

```yaml
tolerations:
  - key: "dedicated"
    operator: "Equal"
    value: "infra"
    effect: "NoSchedule"
```

Giải thích:

```yaml
key: "dedicated"
```

Toleration match với taint có key `dedicated`.

```yaml
operator: "Equal"
```

Yêu cầu value phải bằng nhau.

```yaml
value: "infra"
```

Toleration match với taint có value `infra`.

```yaml
effect: "NoSchedule"
```

Toleration match với taint effect `NoSchedule`.

Taint trên node là:

```text
dedicated=infra:NoSchedule
```

Toleration trên Pod là:

```text
dedicated=infra:NoSchedule
```

Vì khớp nhau nên Pod được phép schedule vào node.

Lưu ý rất quan trọng:

```text
Toleration không ép Pod chạy vào node bị taint.
Toleration chỉ cho phép Pod chịu được taint đó.
Muốn hướng Pod vào node cụ thể, cần kết hợp thêm nodeSelector hoặc nodeAffinity.
```

---

## 39. Xóa taint sau lab

Sau khi lab xong, xóa taint để tránh ảnh hưởng các bài sau:

```bash
kubectl taint nodes k8s-worker-02 dedicated=infra:NoSchedule-
```

Dấu `-` ở cuối dùng để remove taint.

Kiểm tra:

```bash
kubectl describe node k8s-worker-02 | grep -i taints
```

Nếu không còn taint:

```text
Taints: <none>
```

---

# PHẦN 7: CORDON, UNCORDON VÀ DRAIN NODE

---

## 40. Vì sao cần cordon/drain?

Trong vận hành Kubernetes, sẽ có lúc cần bảo trì node:

- Update OS.
- Reboot worker node.
- Thay disk.
- Kiểm tra hardware.
- Cập nhật container runtime.
- Nâng cấp kubelet.

Không nên tắt node đột ngột khi workload đang chạy.

Kubernetes cung cấp các thao tác:

- `cordon`: đánh dấu node không nhận Pod mới.
- `drain`: đuổi workload ra khỏi node một cách có kiểm soát.
- `uncordon`: cho node nhận Pod mới trở lại.

---

## 41. Cordon node

Cordon worker-01:

```bash
kubectl cordon k8s-worker-01
```

Kiểm tra:

```bash
kubectl get nodes
```

Kết quả:

```text
k8s-worker-01   Ready,SchedulingDisabled
```

Ý nghĩa:

```text
Node vẫn Ready, Pod cũ vẫn chạy, nhưng scheduler không đặt Pod mới lên node này.
```

Tạo thử Deployment mới:

```bash
kubectl create deployment cordon-test --image=nginx:1.27-alpine -n lab08-resource-scheduling --replicas=3
```

Kiểm tra:

```bash
kubectl get pods -n lab08-resource-scheduling -l app=cordon-test -o wide
```

Pod mới sẽ không được schedule vào `k8s-worker-01`.

---

## 42. Uncordon node

Cho node nhận Pod trở lại:

```bash
kubectl uncordon k8s-worker-01
```

Kiểm tra:

```bash
kubectl get nodes
```

Kết quả:

```text
k8s-worker-01   Ready
```

Dọn Deployment test:

```bash
kubectl delete deployment cordon-test -n lab08-resource-scheduling
```

---

## 43. Drain node

`drain` dùng để chuẩn bị node trước khi bảo trì.

Câu lệnh lab:

```bash
kubectl drain k8s-worker-01 --ignore-daemonsets --delete-emptydir-data
```

Giải thích:

```bash
kubectl drain k8s-worker-01
```

Yêu cầu Kubernetes evict workload khỏi node `k8s-worker-01`.

```bash
--ignore-daemonsets
```

DaemonSet Pod thường được quản lý đặc biệt và không bị drain theo cách workload thường. Flag này yêu cầu bỏ qua DaemonSet.

```bash
--delete-emptydir-data
```

Cho phép xóa dữ liệu trong `emptyDir` khi Pod bị evict.

Cảnh báo:

```text
Không chạy drain bừa trên production nếu chưa hiểu workload, replica, storage, PDB và downtime impact.
```

Sau khi drain, kiểm tra node:

```bash
kubectl get nodes
```

Node sẽ ở trạng thái:

```text
Ready,SchedulingDisabled
```

Pod thuộc Deployment sẽ được tạo lại trên node khác nếu còn tài nguyên.

Sau khi bảo trì xong:

```bash
kubectl uncordon k8s-worker-01
```

---

# PHẦN 8: LIMITRANGE VÀ RESOURCEQUOTA Ở MỨC CƠ BẢN

---

## 44. Vì sao cần LimitRange và ResourceQuota?

Trong môi trường nhiều team dùng chung Kubernetes cluster, nếu không kiểm soát tài nguyên theo namespace, một team có thể tạo quá nhiều Pod hoặc request quá nhiều CPU/RAM.

Hai object quan trọng:

### LimitRange

Dùng để đặt default request/limit hoặc giới hạn min/max cho container trong namespace.

Ví dụ:

```text
Nếu user tạo Pod mà quên resources, namespace tự gán default request/limit.
```

### ResourceQuota

Dùng để giới hạn tổng tài nguyên được sử dụng trong namespace.

Ví dụ:

```text
Namespace team-a chỉ được request tối đa 4 CPU và 8Gi memory.
```

Trong buổi này, chúng ta chỉ học ở mức nền tảng để hiểu cách bảo vệ cluster khỏi workload không kiểm soát.

---

## 45. Manifest LimitRange

Tạo file:

```bash
nano 11-limitrange.yaml
```

Nội dung:

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: default-container-limits
  namespace: lab08-resource-scheduling
spec:
  limits:
    - type: Container
      defaultRequest:
        cpu: "100m"
        memory: "128Mi"
      default:
        cpu: "500m"
        memory: "256Mi"
      min:
        cpu: "50m"
        memory: "64Mi"
      max:
        cpu: "1"
        memory: "1Gi"
```

Apply:

```bash
kubectl apply -f 11-limitrange.yaml
```

Kiểm tra:

```bash
kubectl describe limitrange default-container-limits -n lab08-resource-scheduling
```

Ý nghĩa:

```text
Container không khai báo resources có thể được gán default request/limit.
Container không được nhỏ hơn min hoặc lớn hơn max đã cấu hình.
```

---

## 46. Manifest ResourceQuota

Tạo file:

```bash
nano 12-resourcequota.yaml
```

Nội dung:

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: namespace-compute-quota
  namespace: lab08-resource-scheduling
spec:
  hard:
    requests.cpu: "2"
    requests.memory: "2Gi"
    limits.cpu: "4"
    limits.memory: "4Gi"
    pods: "20"
```

Apply:

```bash
kubectl apply -f 12-resourcequota.yaml
```

Kiểm tra:

```bash
kubectl describe resourcequota namespace-compute-quota -n lab08-resource-scheduling
```

Ý nghĩa:

```yaml
requests.cpu: "2"
```

Tổng CPU requests trong namespace không vượt quá 2 CPU.

```yaml
requests.memory: "2Gi"
```

Tổng memory requests trong namespace không vượt quá 2Gi.

```yaml
limits.cpu: "4"
```

Tổng CPU limits trong namespace không vượt quá 4 CPU.

```yaml
limits.memory: "4Gi"
```

Tổng memory limits trong namespace không vượt quá 4Gi.

```yaml
pods: "20"
```

Namespace không được có quá 20 Pod.

---

## 47. Lưu ý khi dùng ResourceQuota trong lab

ResourceQuota có thể làm một số manifest ở phần sau bị reject nếu tổng request/limit vượt quota.

Nếu cần dọn quota để tiếp tục lab khác:

```bash
kubectl delete resourcequota namespace-compute-quota -n lab08-resource-scheduling
kubectl delete limitrange default-container-limits -n lab08-resource-scheduling
```

Trong production, ResourceQuota rất hữu ích cho môi trường nhiều namespace/team.

Trong lớp học, cần nhớ rằng ResourceQuota là một cơ chế admission control ở namespace level.

---

# PHẦN 9: LAB TỔNG HỢP BUỔI 08

---

## 48. Mục tiêu lab tổng hợp

Trong lab tổng hợp, sẽ triển khai một ứng dụng web với các yêu cầu:

- Chạy bằng Deployment.
- Có 2 replica.
- Có resources requests/limits.
- Có readiness/liveness probe đã học từ buổi 07.
- Chỉ chạy trên nhóm node `nodepool=app`.
- Cố gắng phân tán replica trên nhiều node bằng pod anti-affinity.
- Expose bằng Service `NodePort` để kiểm tra nhanh.
- Quan sát scheduler placement.
- Phân tích resource request trên node.

---

## 49. Chuẩn bị node label

Đảm bảo worker node có label:

```bash
kubectl label node k8s-worker-01 nodepool=app --overwrite
kubectl label node k8s-worker-02 nodepool=app --overwrite
```

Kiểm tra:

```bash
kubectl get nodes -L nodepool
```

Kết quả:

```text
k8s-worker-01   nodepool=app
k8s-worker-02   nodepool=app
```

---

## 50. Manifest lab tổng hợp

Tạo file:

```bash
nano 13-final-lab-resource-scheduling.yaml
```

Nội dung:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: production-like-web
  namespace: lab08-resource-scheduling
  labels:
    app: production-like-web
spec:
  replicas: 2
  selector:
    matchLabels:
      app: production-like-web
  template:
    metadata:
      labels:
        app: production-like-web
    spec:
      nodeSelector:
        nodepool: app
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
            - weight: 100
              podAffinityTerm:
                labelSelector:
                  matchExpressions:
                    - key: app
                      operator: In
                      values:
                        - production-like-web
                topologyKey: kubernetes.io/hostname
      containers:
        - name: nginx
          image: nginx:1.27-alpine
          ports:
            - containerPort: 80
          resources:
            requests:
              cpu: "100m"
              memory: "128Mi"
            limits:
              cpu: "500m"
              memory: "256Mi"
          readinessProbe:
            httpGet:
              path: /
              port: 80
            initialDelaySeconds: 5
            periodSeconds: 10
          livenessProbe:
            httpGet:
              path: /
              port: 80
            initialDelaySeconds: 15
            periodSeconds: 20
---
apiVersion: v1
kind: Service
metadata:
  name: production-like-web-svc
  namespace: lab08-resource-scheduling
spec:
  type: NodePort
  selector:
    app: production-like-web
  ports:
    - name: http
      protocol: TCP
      port: 80
      targetPort: 80
      nodePort: 30088
```

Apply:

```bash
kubectl apply -f 13-final-lab-resource-scheduling.yaml
```

---

## 51. Kiểm tra kết quả lab tổng hợp

Kiểm tra Deployment:

```bash
kubectl get deployment production-like-web -n lab08-resource-scheduling
```

Kiểm tra Pod:

```bash
kubectl get pods -n lab08-resource-scheduling -l app=production-like-web -o wide
```

Kỳ vọng:

```text
Có 2 Pod Running.
Pod có xu hướng nằm trên 2 worker khác nhau.
```

Kiểm tra Service:

```bash
kubectl get svc production-like-web-svc -n lab08-resource-scheduling
```

Truy cập từ máy ngoài cluster:

```bash
curl http://<Worker-Node-IP>:30088
```

Ví dụ:

```bash
curl http://10.10.10.21:30088
curl http://10.10.10.22:30088
```

---

## 52. Giải thích manifest lab tổng hợp

### nodeSelector

```yaml
nodeSelector:
  nodepool: app
```

Chỉ cho Pod chạy trên node có label:

```text
nodepool=app
```

### podAntiAffinity

```yaml
podAntiAffinity:
  preferredDuringSchedulingIgnoredDuringExecution:
    - weight: 100
```

Scheduler cố gắng không đặt các Pod cùng app trên cùng một node.

### topologyKey

```yaml
topologyKey: kubernetes.io/hostname
```

Phạm vi phân tán là từng node hostname.

### resources.requests

```yaml
requests:
  cpu: "100m"
  memory: "128Mi"
```

Scheduler dùng giá trị này để quyết định node có đủ tài nguyên hay không.

### resources.limits

```yaml
limits:
  cpu: "500m"
  memory: "256Mi"
```

Giới hạn mức tài nguyên tối đa container được phép sử dụng.

### readinessProbe

```yaml
readinessProbe:
  httpGet:
    path: /
    port: 80
```

Kubernetes chỉ đưa Pod vào endpoint của Service nếu readiness probe thành công.

### livenessProbe

```yaml
livenessProbe:
  httpGet:
    path: /
    port: 80
```

Nếu liveness probe fail nhiều lần, kubelet restart container.

### Service NodePort

```yaml
type: NodePort
nodePort: 30088
```

Expose ứng dụng qua port 30088 trên các node.

---

# PHẦN 10: TROUBLESHOOTING RESOURCE VÀ SCHEDULING

---

## 53. Pod Pending do thiếu CPU/Memory

Triệu chứng:

```bash
kubectl get pods -n lab08-resource-scheduling
```

```text
pod-name   0/1   Pending
```

Kiểm tra:

```bash
kubectl describe pod <pod-name> -n lab08-resource-scheduling
```

Event thường gặp:

```text
insufficient cpu
insufficient memory
```

Hướng xử lý:

- Giảm `requests` nếu đang khai báo quá cao.
- Tăng tài nguyên node.
- Thêm worker node.
- Xóa workload không cần thiết.
- Kiểm tra ResourceQuota.

---

## 54. Pod Pending do nodeSelector sai

Event:

```text
node(s) didn't match Pod's node affinity/selector
```

Kiểm tra node label:

```bash
kubectl get nodes --show-labels
kubectl get nodes -L nodepool,disk
```

Hướng xử lý:

- Sửa label trên node.
- Sửa `nodeSelector` trong manifest.
- Dùng `--overwrite` nếu cần cập nhật label.

Ví dụ:

```bash
kubectl label node k8s-worker-01 nodepool=app --overwrite
```

---

## 55. Pod Pending do taint không có toleration

Event:

```text
node(s) had untolerated taint
```

Kiểm tra taint:

```bash
kubectl describe node <node-name> | grep -i taints
```

Hướng xử lý:

- Thêm toleration phù hợp vào Pod.
- Hoặc xóa taint nếu không còn cần.

Xóa taint:

```bash
kubectl taint nodes <node-name> key=value:NoSchedule-
```

---

## 56. Container OOMKilled

Triệu chứng:

```bash
kubectl get pod <pod-name> -n <namespace>
```

Có thể thấy restart count tăng.

Describe:

```bash
kubectl describe pod <pod-name> -n <namespace>
```

Tìm:

```text
Reason: OOMKilled
Exit Code: 137
```

Nguyên nhân:

```text
Container vượt memory limit.
```

Hướng xử lý:

- Tăng memory limit.
- Tối ưu ứng dụng.
- Kiểm tra memory leak.
- Đặt request/limit phù hợp hơn.

---

## 57. Ứng dụng chậm nhưng không restart

Một nguyên nhân có thể là CPU throttling.

Dấu hiệu:

- Ứng dụng phản hồi chậm.
- Không bị OOMKilled.
- Không restart.
- CPU limit đặt quá thấp.

Hướng xử lý:

- Kiểm tra CPU limit.
- Tăng CPU limit nếu cần.
- Với workload latency-sensitive, cân nhắc không đặt CPU limit quá thấp.
- Dùng monitoring ở buổi observability để xác nhận CPU throttling.

---

## 58. Node SchedulingDisabled

Kiểm tra:

```bash
kubectl get nodes
```

Thấy:

```text
Ready,SchedulingDisabled
```

Nguyên nhân:

- Node đã bị cordon.
- Node đã bị drain.

Hướng xử lý:

```bash
kubectl uncordon <node-name>
```

---

# PHẦN 11: BEST PRACTICES CƠ BẢN

---

## 59. Best practices về requests và limits

### Luôn khai báo requests cho workload quan trọng

Nếu không khai báo requests, scheduler không có cơ sở tốt để xếp Pod.

Khuyến nghị:

```yaml
resources:
  requests:
    cpu: "100m"
    memory: "128Mi"
```

### Không đặt memory limit quá sát usage thực tế

Nếu ứng dụng thường dùng 240Mi, không nên đặt limit 256Mi quá sát.

Nên để buffer.

### Cẩn thận với CPU limit

CPU limit quá thấp có thể gây throttling.

Với một số workload production, có thể đặt CPU request nhưng cân nhắc kỹ CPU limit.

### Không để production workload ở BestEffort

Pod BestEffort dễ bị evict nhất khi node pressure.

### Dùng QoS Guaranteed cho workload đặc biệt quan trọng

Ví dụ:

- Core service rất quan trọng.
- Workload latency-sensitive.
- Thành phần hạ tầng cần ổn định.

Tuy nhiên, Guaranteed thường làm giảm mức linh hoạt sử dụng tài nguyên.

---

## 60. Best practices về scheduling

### Dùng label có ý nghĩa

Ví dụ:

```bash
nodepool=app
nodepool=infra
disk=ssd
environment=production
zone=dc-a
```

Tránh label mơ hồ như:

```bash
fast=true
special=yes
```

### Không dùng nodeName nếu không thật sự cần

Ưu tiên:

- `nodeSelector`
- `nodeAffinity`
- `taints/tolerations`

### Dùng taint cho node chuyên dụng

Ví dụ:

```bash
kubectl taint nodes k8s-worker-02 dedicated=infra:NoSchedule
```

Sau đó workload hạ tầng có toleration phù hợp.

### Kết hợp taint/toleration với nodeSelector hoặc affinity

Toleration chỉ cho phép Pod chịu được taint.

Nó không ép Pod vào node đó.

Muốn ép hoặc hướng Pod vào node, cần thêm:

```yaml
nodeSelector:
  nodepool: infra
```

hoặc:

```yaml
affinity:
  nodeAffinity:
    ...
```

### Cẩn thận khi drain node

Trước khi drain production node, cần kiểm tra:

- Workload có đủ replica không?
- Có Pod dùng local storage không?
- Có StatefulSet không?
- Có DaemonSet không?
- Có PodDisruptionBudget không?
- Drain có gây downtime không?

---

# PHẦN 12: CHECKLIST CUỐI BUỔI

---

## 61. Checklist kiến thức cần nắm

Sau buổi này, cần trả lời được:

- `requests` khác `limits` như thế nào?
- Scheduler dùng `requests` hay `limits` để quyết định placement?
- CPU `500m` nghĩa là gì?
- Memory `256Mi` nghĩa là gì?
- Vì sao `400m` memory là sai?
- Pod `Guaranteed` cần điều kiện gì?
- Pod `Burstable` là gì?
- Pod `BestEffort` nguy hiểm ở điểm nào?
- Vì sao Pod bị `Pending` do thiếu resource?
- kube-scheduler làm gì?
- Filtering và scoring khác nhau như thế nào?
- `nodeSelector` dùng khi nào?
- `nodeAffinity` khác `nodeSelector` ở đâu?
- Taint khác nodeAffinity như thế nào?
- Toleration có ép Pod chạy vào node bị taint không?
- Cordon khác drain như thế nào?
- Khi nào cần uncordon node?

---

## 62. Câu hỏi kiểm tra cuối buổi

### Câu 1

Trong Kubernetes, scheduler sử dụng trường nào để kiểm tra node có đủ tài nguyên chạy Pod hay không?

A. `resources.limits`

B. `resources.requests`

C. `containerPort`

D. `replicas`

Đáp án: B

---

### Câu 2

CPU `250m` tương đương bao nhiêu CPU core?

A. 0.025 core

B. 0.25 core

C. 2.5 core

D. 25 core

Đáp án: B

---

### Câu 3

Pod nào dễ bị evict nhất khi node bị áp lực tài nguyên?

A. Guaranteed

B. Burstable

C. BestEffort

D. StatefulSet

Đáp án: C

---

### Câu 4

Nếu một container vượt memory limit, hiện tượng nào thường xảy ra?

A. CPU throttling

B. OOMKilled

C. Service đổi IP

D. Pod tự đổi namespace

Đáp án: B

---

### Câu 5

`nodeSelector` dùng để làm gì?

A. Expose Pod ra ngoài cluster

B. Chọn node dựa trên node label

C. Tạo PersistentVolume

D. Cấu hình Ingress Controller

Đáp án: B

---

### Câu 6

Toleration có ý nghĩa gì?

A. Ép Pod chạy vào node cụ thể

B. Cho phép Pod chịu được taint phù hợp trên node

C. Xóa taint khỏi node

D. Tăng CPU limit cho Pod

Đáp án: B

---

### Câu 7

Câu lệnh nào dùng để ngăn node nhận Pod mới nhưng không đuổi Pod cũ?

A. `kubectl drain`

B. `kubectl cordon`

C. `kubectl delete node`

D. `kubectl taint pod`

Đáp án: B

---

### Câu 8

Câu lệnh nào dùng để cho node nhận Pod mới trở lại sau khi cordon?

A. `kubectl uncordon`

B. `kubectl undrain`

C. `kubectl restart node`

D. `kubectl rollout undo`

Đáp án: A

---

### Câu 9

Trong scheduler, bước Filtering có nhiệm vụ gì?

A. Chấm điểm tất cả node

B. Loại bỏ các node không phù hợp

C. Restart container lỗi

D. Tạo Service endpoint

Đáp án: B

---

### Câu 10

Nếu Pod có `nodeSelector: nodepool=gpu` nhưng không node nào có label này, Pod sẽ thường ở trạng thái nào?

A. Running

B. Completed

C. Pending

D. CrashLoopBackOff

Đáp án: C

---

# PHẦN 13: DỌN DẸP LAB

---

## 63. Xóa tài nguyên lab

Nếu muốn xóa toàn bộ tài nguyên buổi 08:

```bash
kubectl delete namespace lab08-resource-scheduling
```

Xóa label test trên node nếu muốn đưa node về trạng thái ban đầu:

```bash
kubectl label node k8s-worker-01 nodepool- disk-
kubectl label node k8s-worker-02 nodepool- disk-
```

Đảm bảo không còn taint lab:

```bash
kubectl taint nodes k8s-worker-02 dedicated=infra:NoSchedule- || true
```

Kiểm tra lại node:

```bash
kubectl get nodes -L nodepool,disk
kubectl describe node k8s-worker-02 | grep -i taints
```

---

# PHẦN 14: TỔNG KẾT BUỔI 08

---

## 64. Tổng kết

Trong buổi này, đã học cách Kubernetes quản lý tài nguyên và điều khiển placement của Pod.

Các điểm quan trọng nhất:

- `requests` là cơ sở để scheduler quyết định node có đủ tài nguyên hay không.
- `limits` là giới hạn tối đa container được phép sử dụng.
- CPU vượt limit thường bị throttling.
- Memory vượt limit có thể gây `OOMKilled`.
- Kubernetes phân loại Pod thành `Guaranteed`, `Burstable`, `BestEffort`.
- kube-scheduler chọn node qua hai bước chính: filtering và scoring.
- `nodeSelector` là cách đơn giản để chọn node theo label.
- `nodeAffinity` linh hoạt hơn `nodeSelector`.
- `podAntiAffinity` giúp phân tán Pod để tăng availability.
- Taint giúp node từ chối Pod không phù hợp.
- Toleration giúp Pod được phép chạy trên node có taint phù hợp.
- Cordon ngăn node nhận Pod mới.
- Drain di chuyển workload khỏi node để bảo trì.
- LimitRange và ResourceQuota giúp kiểm soát tài nguyên theo namespace.

Sau buổi 08, đã có nền tảng tốt để bước sang các nội dung tiếp theo như Security, Observability, Troubleshooting, Autoscaling và vận hành workload production-like.

---

# PHẦN 15: NGUỒN THAM KHẢO CHÍNH THỨC

---

## 65. Kubernetes Documentation

- Kubernetes v1.35 Documentation - Resource Management for Pods and Containers  
  https://v1-35.docs.kubernetes.io/docs/concepts/configuration/manage-resources-containers/

- Kubernetes v1.35 Documentation - Pod Quality of Service Classes  
  https://v1-35.docs.kubernetes.io/docs/concepts/workloads/pods/pod-qos/

- Kubernetes v1.35 Documentation - Kubernetes Scheduler  
  https://v1-35.docs.kubernetes.io/docs/concepts/scheduling-eviction/kube-scheduler/

- Kubernetes Documentation - Assigning Pods to Nodes  
  https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/

- Kubernetes v1.35 Documentation - Taints and Tolerations  
  https://v1-35.docs.kubernetes.io/docs/concepts/scheduling-eviction/taint-and-toleration/

