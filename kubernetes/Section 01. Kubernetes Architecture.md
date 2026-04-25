# Buổi 01: Kiến trúc Kubernetes

## Thông tin bài giảng

**Khóa học:** Kubernetes Administration / Kubernetes Foundation  
**Phiên bản Kubernetes:** v1.35  
**Mô hình lab:** On-premise, triển khai bằng `kubeadm`  
**Hệ điều hành:** Ubuntu Server 24.04  
**Số lượng VM:** 3 VM  
**Vai trò node:**

| Node | Vai trò | Mô tả |
|---|---|---|
| `master-01` | Control Plane | Chạy các thành phần điều khiển chính của Kubernetes |
| `worker-01` | Worker Node | Chạy workload của ứng dụng |
| `worker-02` | Worker Node | Chạy workload của ứng dụng |

---

## Mục tiêu buổi học

Sau buổi học này, cần nắm được:

- Kubernetes là gì và vì sao cần Kubernetes.
- Kiến trúc tổng thể của một Kubernetes cluster.
- Sự khác nhau giữa Control Plane Node và Worker Node.
- Vai trò của các thành phần chính:
  - `kube-apiserver`
  - `etcd`
  - `kube-scheduler`
  - `kube-controller-manager`
  - `kubelet`
  - `kube-proxy`
  - Container Runtime
  - CNI Plugin
  - CoreDNS
- Vì sao trong mô hình `kubeadm`, các control plane component thường chạy dưới dạng Static Pod.
- `kubectl` giao tiếp với Kubernetes cluster như thế nào.
- Pod là đơn vị nhỏ nhất mà Kubernetes có thể triển khai và quản lý.
- Cách quan sát kiến trúc cluster bằng các lệnh `kubectl`, `systemctl`, `crictl`.
- Cách tạo một Pod đơn giản để minh họa luồng hoạt động của Kubernetes.

---

## Các kiến thức được phép sử dụng trong buổi này

Vì đây là buổi đầu tiên, bài giảng chỉ sử dụng các khái niệm nền tảng sau:

- Cluster
- Node
- Control Plane
- Worker Node
- Kubernetes API
- Kubernetes Object
- Namespace ở mức cơ bản
- Pod ở mức cơ bản
- Container Image ở mức cơ bản
- Container Runtime
- CNI ở mức cơ bản
- CoreDNS ở mức cơ bản
- Static Pod
- `kubectl`
- `kubeconfig`

Các buổi sau mới được sử dụng sâu hơn các khái niệm sau:

- Deployment
- ReplicaSet
- DaemonSet
- StatefulSet
- Service
- Ingress
- ConfigMap
- Secret
- PersistentVolume
- PersistentVolumeClaim
- StorageClass
- Probe
- Rolling Update
- Rollback
- Scheduling Policy nâng cao
- RBAC chi tiết

---

## Phần 1: Kubernetes là gì?

Kubernetes là một nền tảng mã nguồn mở dùng để triển khai, vận hành và quản lý các ứng dụng containerized.

Nói đơn giản hơn, nếu Docker hoặc container runtime giúp chạy một container, thì Kubernetes giúp quản lý rất nhiều container chạy trên nhiều máy chủ khác nhau.

Một hệ thống thực tế thường không chỉ có một container. Ví dụ một ứng dụng web có thể gồm:

- Frontend
- Backend API
- Database
- Cache
- Message queue
- Worker xử lý nền
- Monitoring agent
- Logging agent

Nếu chỉ chạy bằng lệnh container thủ công, người quản trị sẽ gặp rất nhiều vấn đề:

- Container chết thì ai khởi động lại?
- Server chết thì workload chuyển sang đâu?
- Cần chạy nhiều bản sao ứng dụng thì làm thế nào?
- Cần rolling update thì xử lý ra sao?
- Cần service discovery thì làm thế nào?
- Cần giới hạn tài nguyên CPU/RAM thì cấu hình ở đâu?
- Cần expose ứng dụng ra bên ngoài thì dùng cơ chế gì?

Kubernetes sinh ra để giải quyết các bài toán đó.

Tuy nhiên, trong buổi đầu tiên, ta chưa đi vào tất cả chức năng trên. Trọng tâm của buổi này là hiểu kiến trúc Kubernetes trước.

---

## Phần 2: Kubernetes Cluster là gì?

Một Kubernetes cluster là tập hợp các máy chủ cùng hoạt động để chạy ứng dụng container.

Trong mô hình lab của khóa học này, cluster gồm 3 VM:

```text
+------------------+       +------------------+
|    master-01     |       |    worker-01     |
|  Control Plane   |       |   Worker Node    |
+------------------+       +------------------+

+------------------+
|    worker-02     |
|   Worker Node    |
+------------------+
```

Trong đó:

- `master-01` là node điều khiển.
- `worker-01` và `worker-02` là nơi chạy workload ứng dụng.

Một Kubernetes cluster luôn có hai nhóm thành phần lớn:

```text
Kubernetes Cluster
├── Control Plane
│   ├── kube-apiserver
│   ├── etcd
│   ├── kube-scheduler
│   └── kube-controller-manager
│
└── Worker Nodes
    ├── kubelet
    ├── kube-proxy
    ├── container runtime
    └── pods
```

---

## Phần 3: Control Plane là gì?

Control Plane là bộ não của Kubernetes cluster.

Control Plane không trực tiếp chạy ứng dụng business như web, API hay database. Thay vào đó, nó chịu trách nhiệm quản lý trạng thái mong muốn của toàn bộ cluster.

Ví dụ, khi người quản trị yêu cầu:

> “Tôi muốn chạy một Pod nginx.”

Control Plane sẽ tiếp nhận yêu cầu, lưu trạng thái vào hệ thống, quyết định Pod nên chạy ở node nào, sau đó các thành phần trên worker node sẽ thực thi.

Trong mô hình `kubeadm` một control plane node thường chạy các thành phần sau:

```text
Control Plane Node: master-01

/etc/kubernetes/manifests/
├── kube-apiserver.yaml
├── etcd.yaml
├── kube-controller-manager.yaml
└── kube-scheduler.yaml
```

Các file YAML trên không phải là manifest ứng dụng thông thường. Đây là manifest của Static Pod dùng để chạy chính các thành phần control plane.

---

## Phần 4: Các thành phần của Control Plane

### kube-apiserver

`kube-apiserver` là cổng giao tiếp trung tâm của Kubernetes.

Tất cả yêu cầu gửi vào cluster đều đi qua API Server.

Ví dụ:

```bash
kubectl get nodes
kubectl get pods
kubectl apply -f pod.yaml
kubectl delete pod nginx-pod
```

Các lệnh này không thao tác trực tiếp với worker node. `kubectl` sẽ gửi HTTP request đến `kube-apiserver`.

Có thể hình dung API Server như cửa chính của Kubernetes:

```text
User / Admin / Tool
        |
        v
kubectl / API Client
        |
        v
kube-apiserver
        |
        v
etcd / scheduler / controller / kubelet
```

Vai trò chính của `kube-apiserver`:

- Cung cấp Kubernetes API.
- Tiếp nhận request từ `kubectl`, dashboard, CI/CD tool hoặc các component nội bộ.
- Xác thực và kiểm tra quyền truy cập.
- Kiểm tra tính hợp lệ của object.
- Ghi trạng thái mong muốn vào `etcd`.
- Là điểm giao tiếp chung cho các thành phần khác.

Trong bài đầu tiên, chỉ cần nhớ:

> Muốn nói chuyện với Kubernetes, gần như luôn phải đi qua `kube-apiserver`.

---

### etcd

`etcd` là cơ sở dữ liệu key-value của Kubernetes.

Toàn bộ trạng thái của cluster được lưu trong `etcd`.

Ví dụ:

- Có bao nhiêu node trong cluster.
- Có bao nhiêu Pod đang được khai báo.
- Pod nào đang chạy ở node nào.
- Namespace nào tồn tại.
- Config của các object.
- Trạng thái mong muốn của cluster.

Có thể hình dung `etcd` là “bộ nhớ dài hạn” của Kubernetes.

```text
kube-apiserver
      |
      v
    etcd
```

Kubernetes không lưu trạng thái chính trong file rời rạc trên từng node worker. Trạng thái cluster được quản lý tập trung thông qua API Server và lưu trong `etcd`.

Trong mô hình lab 1 master + 2 worker:

```text
master-01
├── kube-apiserver
├── etcd
├── kube-controller-manager
└── kube-scheduler
```

`etcd` chỉ chạy trên `master-01`.

Điều này có nghĩa là:

- Mô hình lab đơn giản, dễ học.
- Nhưng không phải mô hình HA.
- Nếu `master-01` hỏng hoàn toàn và không có backup `etcd`, cluster có thể mất trạng thái.

Trong môi trường production, nên có nhiều control plane node hoặc cơ chế backup `etcd` định kỳ.

---

### kube-scheduler

`kube-scheduler` là thành phần quyết định Pod sẽ chạy trên node nào.

Khi một Pod mới được tạo ra, ban đầu Pod chưa được gán vào worker node cụ thể. Scheduler sẽ quan sát các Pod chưa được gán node, sau đó chọn node phù hợp.

Ví dụ:

```text
Pod nginx-pod được tạo
        |
        v
Pod đang ở trạng thái chưa có node
        |
        v
kube-scheduler chọn worker-01
        |
        v
Pod được gán vào worker-01
```

Trong buổi này, ta chỉ cần hiểu scheduler ở mức cơ bản:

- Scheduler không chạy container.
- Scheduler chỉ chọn node.
- Sau khi chọn node, kubelet trên node đó mới là thành phần chạy Pod.

Những nội dung như nodeSelector, affinity, taint, toleration, topology spread sẽ được học ở buổi về Scheduler nâng cao.

---

### kube-controller-manager

`kube-controller-manager` chạy các controller của Kubernetes.

Controller là các vòng lặp điều khiển liên tục quan sát trạng thái thực tế của cluster và cố gắng đưa trạng thái thực tế về trạng thái mong muốn.

Ví dụ tư duy rất quan trọng:

```text
Trạng thái mong muốn:
- Cần có 1 Pod nginx chạy

Trạng thái thực tế:
- Pod nginx chưa tồn tại

Controller / thành phần liên quan:
- Phát hiện lệch trạng thái
- Thực hiện hành động để đưa cluster về trạng thái mong muốn
```

Trong buổi đầu tiên, chưa cần đi sâu từng controller, chỉ cần hiểu:

> Kubernetes vận hành theo mô hình desired state. Người quản trị khai báo trạng thái mong muốn, Kubernetes liên tục điều chỉnh hệ thống để đạt trạng thái đó.

Một số controller phổ biến sẽ học sau:

- Deployment controller
- ReplicaSet controller
- Node controller
- Job controller
- EndpointSlice controller

---

### cloud-controller-manager

`cloud-controller-manager` dùng khi Kubernetes tích hợp với cloud provider như AWS, Azure, Google Cloud hoặc OpenStack.

Trong mô hình on-premise bằng `kubeadm` với 3 VM Ubuntu 24.04, thường chưa dùng `cloud-controller-manager`.

Vì vậy trong bài này chỉ cần biết:

- Đây là component tùy chọn.
- Dùng để tích hợp Kubernetes với hạ tầng cloud bên dưới.
- Ví dụ: tạo Load Balancer cloud, đọc thông tin node từ cloud provider, quản lý route cloud.

---

## Phần 5: Worker Node là gì?

Worker Node là nơi chạy workload ứng dụng.

Trong mô hình lab:

```text
worker-01
worker-02
```

Các worker node sẽ chạy Pod của ứng dụng.

Một worker node thường có các thành phần:

```text
Worker Node
├── kubelet
├── kube-proxy
├── container runtime
└── Pods
```

Trong đó:

- `kubelet` nhận lệnh từ control plane và đảm bảo Pod chạy đúng trên node.
- `container runtime` thực sự tạo và chạy container.
- `kube-proxy` hỗ trợ network rule cho Service.
- CNI plugin cung cấp networking cho Pod.

Ở buổi này, phần Service chưa được học sâu, nên `kube-proxy` chỉ cần hiểu ở mức: thành phần hỗ trợ network rule để các Pod và Service có thể giao tiếp theo mô hình Kubernetes.

---

## Phần 6: Các thành phần trên Worker Node

### kubelet

`kubelet` là agent chạy trên mỗi node.

Mỗi node trong cluster đều có `kubelet`, bao gồm cả control plane node và worker node.

Vai trò chính của `kubelet`:

- Đăng ký node với Kubernetes API Server.
- Nhận thông tin Pod cần chạy trên node.
- Gọi container runtime để tạo container.
- Theo dõi trạng thái Pod và container.
- Báo cáo trạng thái node và Pod về API Server.

Luồng đơn giản:

```text
kube-apiserver
      |
      v
kubelet trên worker-01
      |
      v
container runtime
      |
      v
container chạy bên trong Pod
```

Một cách nói dễ nhớ:

> API Server ra lệnh ở cấp cluster, kubelet thực thi ở cấp node.

---

### Container Runtime

Container Runtime là phần mềm chịu trách nhiệm chạy container thật sự.

Ví dụ phổ biến:

- containerd
- CRI-O

Trong môi trường Kubernetes hiện đại dùng `kubeadm`, `containerd` là lựa chọn rất phổ biến.

Kubernetes không tự mình chạy container. Kubernetes giao tiếp với container runtime thông qua CRI, viết tắt của Container Runtime Interface.

Luồng đơn giản:

```text
kubelet
  |
  v
container runtime
  |
  v
container
```

Nếu thiếu container runtime, worker node không thể chạy Pod.

---

### kube-proxy

`kube-proxy` là thành phần chạy trên các node để hỗ trợ network rule cho Kubernetes Service.

Trong bài đầu tiên, chưa học Service nên chỉ cần nhớ:

- `kube-proxy` liên quan đến networking.
- `kube-proxy` giúp traffic đến Service được chuyển đến Pod phù hợp.
- Trên Linux, kube-proxy thường làm việc với iptables hoặc IPVS.

Ta sẽ quay lại `kube-proxy` kỹ hơn ở buổi Service và Networking.

---

### CNI Plugin

CNI là viết tắt của Container Network Interface.

Kubernetes cần CNI plugin để cấp network cho Pod.

Nếu chưa cài CNI plugin sau khi `kubeadm init`, nhiều Pod hệ thống như CoreDNS sẽ chưa chạy bình thường.

Một số CNI plugin phổ biến:

- Calico
- Cilium
- Flannel
- Weave Net

Trong mô hình on-premise lab, nếu đã cài Calico thì có thể thấy các Pod liên quan trong namespace `kube-system`, ví dụ:

```bash
kubectl get pods -n kube-system -o wide
```

Kết quả có thể có các Pod như:

```text
calico-kube-controllers-...
calico-node-...
```

Trong bài này, ta không đi sâu vào NetworkPolicy hay routing của CNI. Chỉ cần hiểu:

> Không có CNI, Pod networking trong cluster sẽ không hoàn chỉnh.

---

### CoreDNS

CoreDNS là thành phần DNS nội bộ của Kubernetes.

CoreDNS giúp các Pod phân giải tên miền nội bộ trong cluster.

Ví dụ sau này khi học Service, ứng dụng có thể gọi nhau bằng DNS name thay vì IP.

Trong buổi đầu, chỉ cần quan sát CoreDNS như một addon hệ thống.

Kiểm tra CoreDNS:

```bash
kubectl get pods -n kube-system -l k8s-app=kube-dns -o wide
```

Kết quả mong đợi:

```text
NAME                       READY   STATUS    RESTARTS   AGE   IP              NODE
coredns-xxxxxxxxxx-abcde   1/1     Running   0          1h    10.244.x.x      worker-01
coredns-xxxxxxxxxx-fghij   1/1     Running   0          1h    10.244.x.x      worker-02
```

Nếu CoreDNS chưa Running, cần kiểm tra CNI trước.

---

## Phần 7: Pod là gì?

Pod là đơn vị triển khai nhỏ nhất trong Kubernetes.

Một Pod có thể chứa một hoặc nhiều container. Các container trong cùng một Pod chia sẻ network namespace và có thể giao tiếp với nhau qua `localhost`.

Trong bài đầu tiên, ta dùng mô hình đơn giản nhất:

```text
Pod
└── Container nginx
```

Cần lưu ý:

- Kubernetes không chạy container trực tiếp như đơn vị quản lý cao nhất.
- Kubernetes quản lý Pod.
- Container chạy bên trong Pod.
- Pod được đặt lên Node.

Mô hình:

```text
worker-01
└── Pod nginx-pod
    └── Container nginx
```

Trong thực tế production, ta thường không tạo Pod đơn lẻ thủ công. Ta thường dùng Deployment, StatefulSet, DaemonSet hoặc Job để quản lý Pod. Tuy nhiên, vì đây là buổi đầu tiên nên ta dùng Pod đơn lẻ để quan sát kiến trúc dễ hơn.

---

## Phần 8: Namespace là gì ở mức cơ bản?

Namespace là cơ chế phân chia tài nguyên logic trong Kubernetes.

Ví dụ:

```bash
kubectl get namespaces
```

Kết quả có thể gồm:

```text
NAME              STATUS   AGE
default           Active   1h
kube-node-lease   Active   1h
kube-public       Active   1h
kube-system       Active   1h
```

Ý nghĩa cơ bản:

| Namespace | Mục đích |
|---|---|
| `default` | Namespace mặc định cho workload nếu không chỉ định namespace |
| `kube-system` | Chứa các Pod hệ thống của Kubernetes |
| `kube-public` | Namespace public, thường dùng cho thông tin có thể đọc công khai |
| `kube-node-lease` | Chứa Lease object phục vụ heartbeat của node |

Trong bài này, ta chỉ dùng namespace ở mức quan sát:

```bash
kubectl get pods -n kube-system
kubectl get pods -n default
```

Chưa đi sâu quota, RBAC, multi-tenancy hoặc isolation theo namespace.

---

## Phần 9: Static Pod trong mô hình kubeadm

Trong cluster được tạo bằng `kubeadm`, các control plane component thường chạy dưới dạng Static Pod.

Static Pod là Pod được quản lý trực tiếp bởi `kubelet` trên một node cụ thể, không phải được tạo theo cách thông thường từ API Server.

Trên control plane node, kiểm tra thư mục:

```bash
sudo ls -l /etc/kubernetes/manifests/
```

Kết quả thường có:

```text
etcd.yaml
kube-apiserver.yaml
kube-controller-manager.yaml
kube-scheduler.yaml
```

Ý nghĩa:

| File | Thành phần được chạy |
|---|---|
| `etcd.yaml` | etcd |
| `kube-apiserver.yaml` | kube-apiserver |
| `kube-controller-manager.yaml` | kube-controller-manager |
| `kube-scheduler.yaml` | kube-scheduler |

Điểm quan trọng:

- `kubelet` trên `master-01` đọc các file YAML trong `/etc/kubernetes/manifests/`.
- Sau đó `kubelet` yêu cầu container runtime chạy các Pod hệ thống đó.
- Vì vậy, ngay cả khi API Server chưa sẵn sàng lúc đầu, kubelet vẫn có thể khởi động API Server dưới dạng Static Pod.
- Sau khi API Server hoạt động, các Static Pod này được phản ánh lên API Server dưới dạng mirror Pod.

Quan sát bằng `kubectl`:

```bash
kubectl get pods -n kube-system -o wide
```

Có thể thấy:

```text
etcd-master-01
kube-apiserver-master-01
kube-controller-manager-master-01
kube-scheduler-master-01
```

Các Pod này thường có tên gắn với hostname của node control plane.

---

## Phần 10: kubectl giao tiếp với cluster như thế nào?

`kubectl` là công cụ dòng lệnh chính để làm việc với Kubernetes.

Khi chạy:

```bash
kubectl get nodes
```

Luồng xử lý về mặt kiến trúc là:

```text
kubectl
  |
  | đọc kubeconfig
  v
kube-apiserver
  |
  | đọc trạng thái cluster
  v
etcd
  |
  v
trả kết quả về kubectl
```

`kubectl` cần file kubeconfig để biết:

- Địa chỉ API Server là gì.
- Dùng certificate hoặc token nào để xác thực.
- Đang thao tác với cluster/context nào.

Với cluster tạo bằng `kubeadm`, file admin kubeconfig thường nằm tại:

```bash
/etc/kubernetes/admin.conf
```

Người dùng thường copy file này về thư mục home:

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

Kiểm tra context hiện tại:

```bash
kubectl config current-context
```

Xem thông tin cluster trong kubeconfig:

```bash
kubectl config view
```

---

## Phần 11: Luồng hoạt động khi tạo một Pod

Khi người quản trị tạo một Pod, ví dụ:

```bash
kubectl apply -f nginx-pod.yaml
```

Luồng xử lý có thể mô tả như sau:

```text
1. Người quản trị chạy kubectl apply
        |
        v
2. kubectl gửi request đến kube-apiserver
        |
        v
3. kube-apiserver kiểm tra request
        |
        v
4. kube-apiserver lưu Pod object vào etcd
        |
        v
5. kube-scheduler phát hiện Pod chưa có node
        |
        v
6. kube-scheduler chọn worker node phù hợp
        |
        v
7. kubelet trên worker node nhận thông tin Pod
        |
        v
8. kubelet gọi container runtime
        |
        v
9. container runtime pull image và chạy container
        |
        v
10. kubelet báo trạng thái Pod về kube-apiserver
```

Sơ đồ tổng quan:

```text
kubectl
  |
  v
kube-apiserver
  |
  +-------> etcd
  |
  +-------> kube-scheduler
              |
              v
          chọn node
              |
              v
          kubelet trên worker
              |
              v
       container runtime
              |
              v
          Pod / Container
```

Đây là một trong những luồng quan trọng nhất cần nắm trong buổi đầu.

---

## Phần 12: Manifest Pod đầu tiên

### File manifest hoàn chỉnh

Tạo file:

```bash
mkdir -p ~/k8s-lab/buoi-01
cd ~/k8s-lab/buoi-01
nano nginx-pod.yaml
```

Nội dung file:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-architecture-demo
  namespace: default
  labels:
    app: nginx
    lesson: architecture
spec:
  containers:
    - name: nginx
      image: nginx:1.27
      ports:
        - containerPort: 80
```

Áp dụng manifest:

```bash
kubectl apply -f nginx-pod.yaml
```

Kiểm tra Pod:

```bash
kubectl get pods -o wide
```

Kết quả mong đợi:

```text
NAME                      READY   STATUS    RESTARTS   AGE   IP            NODE
nginx-architecture-demo   1/1     Running   0          30s   10.244.x.x    worker-01
```

Node có thể là `worker-01` hoặc `worker-02`, tùy scheduler chọn.

---

### Giải thích manifest

#### `apiVersion: v1`

```yaml
apiVersion: v1
```

Dòng này khai báo version API của object.

Với Pod cơ bản, API version là `v1`.

Trong Kubernetes, mỗi loại object thuộc về một API group và version nhất định. Buổi này chỉ cần nhớ:

> Pod cơ bản dùng `apiVersion: v1`.

---

#### `kind: Pod`

```yaml
kind: Pod
```

Dòng này cho Kubernetes biết object cần tạo là Pod.

Ở buổi đầu tiên, ta dùng Pod đơn lẻ để dễ quan sát kiến trúc.

Chưa dùng Deployment ở bài này vì Deployment là workload resource cấp cao hơn và sẽ được học riêng.

---

#### `metadata`

```yaml
metadata:
  name: nginx-architecture-demo
  namespace: default
  labels:
    app: nginx
    lesson: architecture
```

`metadata` chứa thông tin định danh và mô tả object.

Trong đó:

```yaml
name: nginx-architecture-demo
```

Là tên của Pod.

```yaml
namespace: default
```

Pod được tạo trong namespace `default`.

```yaml
labels:
  app: nginx
  lesson: architecture
```

Labels là các cặp key-value dùng để gắn nhãn cho object.

Trong bài này, label chỉ dùng để quan sát và filter đơn giản. Các buổi sau sẽ học kỹ hơn về label selector.

Ví dụ filter Pod theo label:

```bash
kubectl get pods -l app=nginx
```

---

#### `spec`

```yaml
spec:
  containers:
    - name: nginx
      image: nginx:1.27
      ports:
        - containerPort: 80
```

`spec` mô tả trạng thái mong muốn của Pod.

Nói cách khác, đây là phần ta khai báo:

> Tôi muốn Pod này chạy container nào, dùng image gì, mở port nào.

---

#### `containers`

```yaml
containers:
  - name: nginx
```

Một Pod có thể có một hoặc nhiều container.

Trong bài này, Pod chỉ có một container tên là `nginx`.

---

#### `image`

```yaml
image: nginx:1.27
```

Dòng này khai báo container image cần chạy.

Kubelet sẽ yêu cầu container runtime pull image này nếu node chưa có sẵn image.

---

#### `ports`

```yaml
ports:
  - containerPort: 80
```

Dòng này khai báo container bên trong Pod lắng nghe port 80.

Ở buổi này, ta chưa expose Pod ra ngoài cluster bằng Service. Vì vậy port này chủ yếu để mô tả container port và phục vụ việc quan sát.

Service sẽ được học ở buổi networking/service riêng.

---

## Phần 13: Quan sát Pod được scheduler đặt vào node nào

Sau khi tạo Pod:

```bash
kubectl get pod nginx-architecture-demo -o wide
```

Kết quả ví dụ:

```text
NAME                      READY   STATUS    RESTARTS   AGE   IP            NODE
nginx-architecture-demo   1/1     Running   0          1m    10.244.1.10   worker-01
```

Cột `NODE` cho biết Pod đang chạy trên node nào.

Nếu Pod chạy trên `worker-01`, có thể kiểm tra container ở node đó:

```bash
ssh worker-01
sudo crictl ps | grep nginx
```

Kết quả có thể tương tự:

```text
CONTAINER           IMAGE               CREATED          STATE    NAME
xxxxxxxxxxxx        nginx               2 minutes ago    Running  nginx
```

Giải thích:

- `kubectl get pod` lấy thông tin từ API Server.
- `crictl ps` kiểm tra container thực tế trên node.
- Nếu Pod được scheduler đặt ở `worker-01`, container thật sự sẽ chạy trên `worker-01`.

Đây là điểm rất quan trọng để phân biệt:

| Công cụ | Nhìn từ đâu |
|---|---|
| `kubectl get pod` | Nhìn từ Kubernetes API |
| `crictl ps` | Nhìn trực tiếp từ container runtime trên node |

---

## Phần 14: Kiểm tra trạng thái cluster

### Kiểm tra node

```bash
kubectl get nodes -o wide
```

Kết quả ví dụ:

```text
NAME        STATUS   ROLES           AGE   VERSION   INTERNAL-IP
master-01   Ready    control-plane   1h    v1.35.x   10.10.10.11
worker-01   Ready    <none>          1h    v1.35.x   10.10.10.21
worker-02   Ready    <none>          1h    v1.35.x   10.10.10.22
```

Giải thích:

| Cột | Ý nghĩa |
|---|---|
| `NAME` | Tên node trong Kubernetes |
| `STATUS` | Trạng thái node |
| `ROLES` | Vai trò node |
| `VERSION` | Phiên bản kubelet |
| `INTERNAL-IP` | IP nội bộ của node |

Nếu node ở trạng thái `Ready`, node có thể tham gia cluster.

Nếu node ở trạng thái `NotReady`, cần kiểm tra:

- kubelet
- container runtime
- CNI
- network giữa node và API Server
- certificate
- tài nguyên CPU/RAM/disk

---

### Kiểm tra namespace hệ thống

```bash
kubectl get namespaces
```

Kết quả ví dụ:

```text
NAME              STATUS   AGE
default           Active   1h
kube-node-lease   Active   1h
kube-public       Active   1h
kube-system       Active   1h
```

Namespace `kube-system` rất quan trọng vì chứa nhiều thành phần hệ thống của cluster.

---

### Kiểm tra Pod hệ thống

```bash
kubectl get pods -n kube-system -o wide
```

Kết quả ví dụ:

```text
NAME                                READY   STATUS    NODE
etcd-master-01                      1/1     Running   master-01
kube-apiserver-master-01            1/1     Running   master-01
kube-controller-manager-master-01   1/1     Running   master-01
kube-scheduler-master-01            1/1     Running   master-01
coredns-xxxxxxxxxx-abcde            1/1     Running   worker-01
coredns-xxxxxxxxxx-fghij            1/1     Running   worker-02
kube-proxy-xxxxx                    1/1     Running   master-01
kube-proxy-yyyyy                    1/1     Running   worker-01
kube-proxy-zzzzz                    1/1     Running   worker-02
```

Nếu dùng Calico, có thể có thêm:

```text
calico-kube-controllers-...
calico-node-...
```

Giải thích nhanh:

| Pod | Vai trò |
|---|---|
| `etcd-master-01` | Lưu trạng thái cluster |
| `kube-apiserver-master-01` | Kubernetes API |
| `kube-controller-manager-master-01` | Chạy các controller |
| `kube-scheduler-master-01` | Chọn node cho Pod |
| `coredns-*` | DNS nội bộ |
| `kube-proxy-*` | Network rule cho Service |
| `calico-*` | Pod network nếu dùng Calico |

---

## Phần 15: Quan sát control plane manifest trên master

Trên node `master-01`:

```bash
sudo ls -l /etc/kubernetes/manifests/
```

Kết quả:

```text
etcd.yaml
kube-apiserver.yaml
kube-controller-manager.yaml
kube-scheduler.yaml
```

Xem file manifest của API Server:

```bash
sudo sed -n '1,120p' /etc/kubernetes/manifests/kube-apiserver.yaml
```

Có thể thấy đây là một Pod manifest.

Ví dụ rút gọn:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: kube-apiserver
  namespace: kube-system
spec:
  containers:
    - command:
        - kube-apiserver
      image: registry.k8s.io/kube-apiserver:v1.35.x
```

Ý nghĩa:

- API Server cũng được chạy như một Pod.
- Nhưng đây là Static Pod do kubelet trên master quản lý.
- Nếu file manifest bị thay đổi sai, control plane có thể lỗi.
- Không nên chỉnh sửa các file này khi chưa hiểu rõ.

---

## Phần 16: Quan sát kubelet

Kiểm tra trạng thái kubelet trên mỗi node:

```bash
sudo systemctl status kubelet
```

Kết quả mong đợi:

```text
Active: active (running)
```

Xem log kubelet:

```bash
sudo journalctl -u kubelet -f
```

Kubelet là một trong những thành phần quan trọng nhất trên mỗi node.

Nếu kubelet lỗi:

- Node có thể chuyển `NotReady`.
- Pod trên node có thể không được báo cáo trạng thái đúng.
- Node không nhận workload mới.
- Các Static Pod trên control plane có thể bị ảnh hưởng.

---

## Phần 17: Quan sát container runtime

Nếu dùng `containerd`, kiểm tra service:

```bash
sudo systemctl status containerd
```

Kiểm tra container đang chạy:

```bash
sudo crictl ps
```

Trên `master-01`, có thể thấy các container của control plane:

```bash
sudo crictl ps | egrep 'kube-apiserver|etcd|kube-scheduler|kube-controller-manager'
```

Trên worker node nơi Pod nginx đang chạy:

```bash
sudo crictl ps | grep nginx
```

Điểm cần nhấn mạnh:

> Kubernetes không thay thế container runtime. Kubernetes điều phối, còn container runtime thực thi container.

---

## Phần 18: Lab tổng hợp buổi 01

### Mục tiêu lab

Sau bài lab này, có thể:

- Xác định node nào là control plane, node nào là worker.
- Kiểm tra các Pod hệ thống của Kubernetes.
- Nhìn thấy control plane component chạy dưới dạng Static Pod.
- Tạo một Pod đơn giản.
- Quan sát Pod được scheduler đặt vào worker node.
- Kiểm tra container thực tế bằng `crictl`.
- Xóa Pod sau khi hoàn thành lab.

---

### Sơ đồ lab

```text
                +----------------------+
                |      master-01       |
                |   Control Plane      |
                |                      |
                | kube-apiserver       |
                | etcd                 |
                | kube-scheduler       |
                | controller-manager   |
                +----------+-----------+
                           |
                           | Kubernetes API
                           |
          +----------------+----------------+
          |                                 |
+---------+----------+           +----------+---------+
|     worker-01      |           |      worker-02     |
| kubelet            |           | kubelet            |
| container runtime  |           | container runtime  |
| Pod có thể chạy ở đây          | Pod có thể chạy ở đây
+--------------------+           +--------------------+
```

---

### Bước 1: Kiểm tra cluster

Chạy trên `master-01` hoặc máy có kubeconfig:

```bash
kubectl cluster-info
```

Kết quả ví dụ:

```text
Kubernetes control plane is running at https://10.10.10.11:6443
CoreDNS is running at https://10.10.10.11:6443/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy
```

Giải thích:

- Kubernetes control plane đang expose API qua port `6443`.
- CoreDNS được nhận diện là service DNS nội bộ của cluster.
- Phần Service sẽ học sau; ở đây chỉ cần biết CoreDNS là DNS addon.

---

### Bước 2: Kiểm tra danh sách node

```bash
kubectl get nodes -o wide
```

Ghi nhận:

- Tên node control plane.
- Tên hai worker node.
- IP của từng node.
- Version Kubernetes.

Câu hỏi:

- Node nào có role `control-plane`?
- Node nào đang là worker?
- Tất cả node đã `Ready` chưa?

---

### Bước 3: Kiểm tra Pod hệ thống

```bash
kubectl get pods -n kube-system -o wide
```

Ghi nhận:

- `kube-apiserver` chạy ở node nào?
- `etcd` chạy ở node nào?
- `kube-scheduler` chạy ở node nào?
- `kube-controller-manager` chạy ở node nào?
- `CoreDNS` chạy ở node nào?
- `kube-proxy` có chạy trên tất cả node không?
- Nếu dùng Calico, `calico-node` có chạy trên tất cả node không?

Câu hỏi:

- Vì sao các component control plane chạy trên `master-01`?
- Vì sao `kube-proxy` thường chạy trên nhiều node?
- Vì sao CoreDNS cần CNI hoạt động trước?

---

### Bước 4: Kiểm tra Static Pod manifest

Trên `master-01`:

```bash
sudo ls -l /etc/kubernetes/manifests/
```

Xem nội dung rút gọn của manifest API Server:

```bash
sudo grep -E 'name:|image:|command:' /etc/kubernetes/manifests/kube-apiserver.yaml
```

Kết quả ví dụ:

```text
name: kube-apiserver
image: registry.k8s.io/kube-apiserver:v1.35.x
- kube-apiserver
```

Câu hỏi:

- File này đang mô tả thành phần nào?
- Thành phần nào đọc file này để chạy API Server?
- Có nên chỉnh sửa file này tùy tiện không?

---

### Bước 5: Tạo Pod demo

Tạo thư mục lab:

```bash
mkdir -p ~/k8s-lab/buoi-01
cd ~/k8s-lab/buoi-01
```

Tạo file:

```bash
cat > nginx-pod.yaml <<'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: nginx-architecture-demo
  namespace: default
  labels:
    app: nginx
    lesson: architecture
spec:
  containers:
    - name: nginx
      image: nginx:1.27
      ports:
        - containerPort: 80
EOF
```

Apply manifest:

```bash
kubectl apply -f nginx-pod.yaml
```

Kiểm tra:

```bash
kubectl get pod nginx-architecture-demo -o wide
```

Ghi lại node mà Pod được đặt vào.

---

### Bước 6: Mô tả Pod

```bash
kubectl describe pod nginx-architecture-demo
```

Quan sát các phần:

- `Name`
- `Namespace`
- `Node`
- `Status`
- `IP`
- `Containers`
- `Image`
- `Events`

Phần `Events` rất quan trọng. Có thể thấy các sự kiện như:

```text
Scheduled
Pulling
Pulled
Created
Started
```

Giải thích:

| Event | Ý nghĩa |
|---|---|
| `Scheduled` | Scheduler đã chọn node cho Pod |
| `Pulling` | Kubelet/container runtime đang pull image |
| `Pulled` | Image đã được pull xong |
| `Created` | Container đã được tạo |
| `Started` | Container đã được chạy |

Đây là nơi có thể nhìn thấy luồng kiến trúc Kubernetes trong thực tế.

---

### Bước 7: Kiểm tra container thực tế trên worker node

Xác định Pod đang chạy ở node nào:

```bash
kubectl get pod nginx-architecture-demo -o wide
```

Ví dụ Pod chạy trên `worker-01`.

SSH vào worker đó:

```bash
ssh worker-01
```

Kiểm tra container:

```bash
sudo crictl ps | grep nginx
```

Nếu thấy container nginx, điều đó chứng minh:

- Kubernetes API ghi nhận Pod.
- Scheduler đã chọn node.
- Kubelet trên node đã nhận việc.
- Container runtime đã chạy container.

---

### Bước 8: Xóa Pod

Quay lại node có `kubectl`:

```bash
kubectl delete -f nginx-pod.yaml
```

Kiểm tra:

```bash
kubectl get pods
```

Kết quả mong đợi:

```text
No resources found in default namespace.
```

---

## Phần 19: Câu hỏi kiểm tra cuối buổi

### Câu 1

Trong Kubernetes, thành phần nào là cổng giao tiếp trung tâm của cluster?

A. `kubelet`  
B. `kube-apiserver`  
C. `containerd`  
D. `CoreDNS`  

Đáp án: B

Giải thích: `kube-apiserver` expose Kubernetes API và là điểm giao tiếp trung tâm giữa người dùng, công cụ quản trị và các thành phần nội bộ.

---

### Câu 2

Thành phần nào lưu trạng thái của Kubernetes cluster?

A. `kube-scheduler`  
B. `kube-proxy`  
C. `etcd`  
D. `CoreDNS`  

Đáp án: C

Giải thích: `etcd` là key-value store dùng để lưu dữ liệu của Kubernetes API Server.

---

### Câu 3

Khi một Pod mới được tạo nhưng chưa được gán vào node, thành phần nào sẽ chọn node cho Pod?

A. `kube-scheduler`  
B. `kubelet`  
C. `kube-proxy`  
D. `containerd`  

Đáp án: A

Giải thích: `kube-scheduler` quan sát các Pod chưa được gán node và chọn node phù hợp.

---

### Câu 4

Thành phần nào trực tiếp chạy trên mỗi node và đảm bảo Pod/container hoạt động theo trạng thái mong muốn?

A. `etcd`  
B. `kubelet`  
C. `CoreDNS`  
D. `kubectl`  

Đáp án: B

Giải thích: `kubelet` là node agent, chịu trách nhiệm làm việc với container runtime để chạy Pod trên node.

---

### Câu 5

Trong mô hình `kubeadm`, các manifest control plane thường nằm ở đâu?

A. `/var/lib/kubelet/config.yaml`  
B. `/etc/kubernetes/manifests/`  
C. `/etc/containerd/config.toml`  
D. `/var/log/pods/`  

Đáp án: B

Giải thích: kubeadm đặt các Static Pod manifest của control plane component trong `/etc/kubernetes/manifests/`.

---

### Câu 6

Pod là gì trong Kubernetes?

A. Một máy ảo chạy trong Kubernetes  
B. Một hệ điều hành tối giản  
C. Đơn vị triển khai nhỏ nhất mà Kubernetes tạo và quản lý  
D. Một loại network plugin  

Đáp án: C

Giải thích: Pod là đơn vị deployable nhỏ nhất trong Kubernetes và có thể chứa một hoặc nhiều container.

---

### Câu 7

Nếu chạy `kubectl get pods -n kube-system`, ta đang xem gì?

A. Các Pod hệ thống trong namespace `kube-system`  
B. Các VM trong cluster  
C. Các container Docker trực tiếp trên host  
D. Các image đang lưu trong registry  

Đáp án: A

Giải thích: Namespace `kube-system` thường chứa các thành phần hệ thống của Kubernetes.

---

### Câu 8

Công cụ nào có thể dùng để quan sát container thực tế đang chạy qua CRI runtime trên node?

A. `kubectl config view`  
B. `crictl ps`  
C. `kubeadm init`  
D. `kubectl api-resources`  

Đáp án: B

Giải thích: `crictl ps` dùng để xem container qua CRI runtime trên node.

---

### Câu 9

Vì sao trong bài này chưa dùng Deployment?

A. Vì Kubernetes v1.35 không còn Deployment  
B. Vì Deployment chỉ chạy trên cloud  
C. Vì buổi này chỉ tập trung vào kiến trúc nền tảng và Pod đơn lẻ để quan sát luồng hoạt động  
D. Vì Deployment không thể chạy trên kubeadm  

Đáp án: C

Giải thích: Deployment là workload resource cấp cao hơn, sẽ được học ở buổi sau khi đã nắm Pod, Node và control plane.

---

### Câu 10

Thành phần nào cung cấp DNS nội bộ cho Kubernetes cluster?

A. CoreDNS  
B. kubelet  
C. containerd  
D. etcd  

Đáp án: A

Giải thích: CoreDNS là DNS addon mặc định phổ biến trong Kubernetes.

---

## Phần 20: Tổng kết buổi học

Trong buổi này, đã học được kiến trúc nền tảng của Kubernetes.

Các ý quan trọng cần nhớ:

- Kubernetes cluster gồm Control Plane và Worker Node.
- Control Plane là bộ não điều khiển cluster.
- Worker Node là nơi chạy workload.
- `kube-apiserver` là cổng giao tiếp trung tâm.
- `etcd` lưu trạng thái cluster.
- `kube-scheduler` chọn node cho Pod.
- `kube-controller-manager` chạy các controller để duy trì desired state.
- `kubelet` chạy trên mỗi node và đảm bảo Pod hoạt động.
- Container runtime thực sự chạy container.
- CNI cung cấp network cho Pod.
- CoreDNS cung cấp DNS nội bộ.
- Pod là đơn vị triển khai nhỏ nhất trong Kubernetes.
- Trong `kubeadm`, control plane component thường chạy dưới dạng Static Pod trong `/etc/kubernetes/manifests/`.
- `kubectl` giao tiếp với cluster thông qua Kubernetes API Server.

---

## Phần 21: Chuẩn bị cho buổi tiếp theo

Sau buổi này, đã có đủ nền tảng để học tiếp về Kubernetes Object và cách quản lý object bằng YAML manifest.

Buổi tiếp theo có thể đi vào:

- Kubernetes Object Model
- `apiVersion`, `kind`, `metadata`, `spec`, `status`
- Imperative command vs declarative manifest
- Pod manifest chi tiết hơn
- Label và selector ở mức nền tảng
- Namespace thực hành cơ bản

Chưa nên đi vào Deployment ngay nếu chưa nắm vững object model và YAML manifest.

---

## Nguồn tham khảo chính

- Kubernetes v1.35 Documentation – Kubernetes Components  
  https://v1-35.docs.kubernetes.io/docs/concepts/overview/components/

- Kubernetes v1.35 Documentation – Nodes  
  https://v1-35.docs.kubernetes.io/docs/concepts/architecture/nodes/

- Kubernetes v1.35 Documentation – Pods  
  https://v1-35.docs.kubernetes.io/docs/concepts/workloads/pods/

- Kubernetes v1.35 Documentation – The Kubernetes API  
  https://v1-35.docs.kubernetes.io/docs/concepts/overview/kubernetes-api/

- Kubernetes v1.35 Documentation – The kubectl command-line tool  
  https://v1-35.docs.kubernetes.io/docs/concepts/overview/kubectl/

- Kubernetes v1.35 Documentation – Creating a cluster with kubeadm  
  https://v1-35.docs.kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/

- Kubernetes v1.35 Documentation – Static Pods  
  https://v1-35.docs.kubernetes.io/docs/tasks/configure-pod-container/static-pod/
