# Buổi 03 - Kubernetes Workloads: Pod, ReplicaSet, Deployment, DaemonSet, StatefulSet, Job và CronJob

## 1. Mục tiêu buổi học

Sau buổi học này,  cần nắm được cách Kubernetes mô hình hóa ứng dụng thông qua các workload resource.

Trong buổi 01,  đã biết Kubernetes có control plane, worker node, kubelet, container runtime, Pod và các thành phần hệ thống như kube-apiserver, scheduler, controller-manager, etcd, CoreDNS, CNI.

Trong buổi 02,  đã triển khai được một cụm Kubernetes bằng `kubeadm`, sử dụng `containerd` làm container runtime và Calico làm CNI.

Buổi 03 bắt đầu đi vào câu hỏi rất thực tế:

> Sau khi đã có cluster, làm sao để chạy ứng dụng trên Kubernetes một cách đúng đắn?

Mục tiêu cụ thể:

- Hiểu khái niệm workload trong Kubernetes.
- Hiểu Pod là đơn vị chạy ứng dụng nhỏ nhất.
- Phân biệt Pod single-container và multi-container.
- Hiểu vì sao không nên vận hành ứng dụng production chỉ bằng Pod đơn lẻ.
- Hiểu vai trò của ReplicaSet trong việc duy trì số lượng Pod mong muốn.
- Hiểu vai trò của Deployment trong việc quản lý ứng dụng stateless.
- Hiểu DaemonSet dùng để chạy Pod trên mỗi node hoặc một nhóm node.
- Hiểu StatefulSet dùng cho ứng dụng cần định danh ổn định.
- Hiểu Job dùng cho tác vụ chạy một lần đến khi hoàn tất.
- Hiểu CronJob dùng cho tác vụ chạy theo lịch.
- Biết đọc, viết và giải thích các manifest cơ bản cho từng loại workload.
- Biết kiểm tra trạng thái workload bằng `kubectl`.

## 2. Phạm vi kiến thức của buổi này

### 2.1. Được phép sử dụng từ các buổi trước

Trong bài này, chúng ta được phép sử dụng các kiến thức đã học:

- Cluster
- Node
- Control Plane
- Worker Node
- kube-apiserver
- kubelet
- kube-scheduler
- controller-manager
- container runtime
- CNI
- Pod
- Namespace ở mức cơ bản
- kubectl
- Manifest YAML cơ bản
- kubeadm cluster 1 master node và 2 worker node

### 2.2. Kiến thức mới được mở khóa trong buổi này

Buổi này sẽ mở khóa thêm các khái niệm:

- Workload
- Container trong Pod
- Pod template
- Label
- Selector
- OwnerReference ở mức khái niệm
- ReplicaSet
- Deployment
- DaemonSet
- StatefulSet
- Job
- CronJob
- Headless Service ở mức tối thiểu để minh họa StatefulSet
- Restart policy ở mức cơ bản
- Resource requests và limits ở mức nhập môn
- Controller reconciliation ở mức nhập môn

### 2.3. Các nội dung chưa đi sâu trong buổi này

Để giữ đúng tiến trình khóa học, bài này chưa đi sâu vào:

- Service networking đầy đủ
- ClusterIP, NodePort, LoadBalancer
- Ingress
- ConfigMap
- Secret
- PersistentVolume, PersistentVolumeClaim, StorageClass
- Probe: liveness, readiness, startup
- Rolling update và rollback chuyên sâu
- Autoscaling
- Taints, tolerations, affinity nâng cao
- RBAC
- NetworkPolicy
- Helm

Một số khái niệm có thể được nhắc đến ở mức tối thiểu nếu bắt buộc để manifest chạy được, nhưng chưa xem là nội dung chính của bài.

## 3. Môi trường lab sử dụng trong bài

Bài lab giả định  đã có cụm Kubernetes từ buổi 02.

| Thành phần | Giá trị mẫu |
|---|---|
| Kubernetes version | v1.35 |
| Bootstrap tool | kubeadm |
| OS | Ubuntu Server 24.04 |
| Container runtime | containerd |
| CNI | Calico |
| Control plane | 1 master node |
| Worker | 2 worker node |
| Namespace lab | `lab-workloads` |

Mô hình node tham khảo:

| Node | Vai trò | Ví dụ hostname |
|---|---|---|
| Master | Control plane | `k8s-master-01` |
| Worker 1 | Worker node | `k8s-worker-01` |
| Worker 2 | Worker node | `k8s-worker-02` |

Kiểm tra cluster trước khi bắt đầu:

```bash
kubectl get nodes -o wide
```

Kết quả kỳ vọng:

```text
NAME            STATUS   ROLES           AGE   VERSION
k8s-master-01   Ready    control-plane    ...   v1.35.x
k8s-worker-01   Ready    <none>           ...   v1.35.x
k8s-worker-02   Ready    <none>           ...   v1.35.x
```

Tạo thư mục làm bài lab:

```bash
mkdir -p ~/k8s-course/buoi-03-workloads
cd ~/k8s-course/buoi-03-workloads
```

Tạo namespace dùng riêng cho buổi học:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: lab-workloads
```

Lưu file:

```bash
cat <<'EOF' > 00-namespace.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: lab-workloads
EOF
```

Apply namespace:

```bash
kubectl apply -f 00-namespace.yaml
kubectl get namespace lab-workloads
```

## 4. Workload trong Kubernetes là gì?

Trong Kubernetes, workload là ứng dụng hoặc tác vụ chạy trên cluster.

Workload có thể là:

- Một web application chạy liên tục.
- Một API backend chạy nhiều bản sao.
- Một logging agent chạy trên tất cả node.
- Một database cần định danh ổn định.
- Một batch job chạy một lần rồi kết thúc.
- Một scheduled job chạy mỗi phút, mỗi giờ hoặc mỗi ngày.

Kubernetes không chỉ chạy container. Kubernetes quản lý trạng thái mong muốn của ứng dụng.

Ví dụ:

> Tôi muốn luôn có 3 Pod chạy ứng dụng web.

Kubernetes sẽ cố gắng duy trì trạng thái đó. Nếu một Pod bị xóa, controller sẽ tạo Pod mới. Nếu số lượng Pod thấp hơn mong muốn, controller sẽ tạo thêm Pod. Nếu số lượng Pod nhiều hơn mong muốn, controller sẽ xóa bớt Pod.

Đây là điểm khác biệt lớn giữa việc chạy container bằng lệnh thủ công và chạy ứng dụng bằng Kubernetes.

## 5. Desired State và Actual State

Kubernetes hoạt động dựa trên mô hình rất quan trọng:

- Desired State: trạng thái mong muốn do người quản trị khai báo.
- Actual State: trạng thái thực tế đang tồn tại trong cluster.
- Controller: thành phần liên tục so sánh hai trạng thái này và điều chỉnh hệ thống.

Ví dụ manifest Deployment khai báo:

```yaml
replicas: 3
```

Ý nghĩa:

> Tôi muốn có 3 Pod chạy ứng dụng này.

Nếu thực tế chỉ còn 2 Pod, controller sẽ tạo thêm 1 Pod.

Nếu thực tế có 4 Pod, controller sẽ xóa bớt 1 Pod.

Luồng xử lý đơn giản:

```text
User
 |
 | kubectl apply -f manifest.yaml
 v
kube-apiserver
 |
 | lưu object vào etcd
 v
etcd
 |
 | controller quan sát object
 v
Controller
 |
 | tạo / xóa / cập nhật Pod
 v
kubelet trên worker node
 |
 | gọi container runtime
 v
container chạy trên node
```

## 6. Label và Selector

Trước khi học ReplicaSet, Deployment, DaemonSet và StatefulSet,  phải hiểu Label và Selector.

### 6.1. Label là gì?

Label là cặp key-value gắn vào Kubernetes object.

Ví dụ:

```yaml
metadata:
  labels:
    app: web
    environment: lab
```

Label giúp Kubernetes và người quản trị phân loại object.

Một Pod có thể có label:

```yaml
app: web
tier: frontend
version: v1
```

### 6.2. Selector là gì?

Selector dùng để chọn các object có label phù hợp.

Ví dụ:

```yaml
selector:
  matchLabels:
    app: web
```

Ý nghĩa:

> Workload này sẽ quản lý các Pod có label `app=web`.

### 6.3. Vì sao Label và Selector rất quan trọng?

Các controller như ReplicaSet, Deployment, DaemonSet và StatefulSet không quản lý Pod bằng tên Pod.

Chúng quản lý Pod thông qua label selector.

Ví dụ ReplicaSet có selector:

```yaml
selector:
  matchLabels:
    app: rs-web
```

Thì template của Pod phải có label tương ứng:

```yaml
template:
  metadata:
    labels:
      app: rs-web
```

Nếu selector và label không khớp, Kubernetes sẽ từ chối manifest hoặc workload không quản lý đúng Pod.

Đây là một trong những lỗi rất phổ biến khi học Kubernetes.

## 7. Pod

## 7.1. Pod là gì?

Pod là đơn vị triển khai nhỏ nhất trong Kubernetes.

Một Pod có thể chứa:

- Một container.
- Nhiều container có liên quan chặt chẽ với nhau.

Các container trong cùng một Pod:

- Được đặt cùng node.
- Được schedule cùng nhau.
- Chia sẻ network namespace.
- Có thể giao tiếp với nhau qua `localhost`.
- Có thể chia sẻ volume nếu được khai báo.

Ví dụ đơn giản:

```text
Pod
 └── Container nginx
```

Ví dụ multi-container:

```text
Pod
 ├── Container web
 └── Container sidecar
```

Điểm cần nhớ:

> Pod không phải là VM. Pod là một logical host nhỏ để chạy một hoặc nhiều container liên quan chặt chẽ.

## 7.2. Khi nào dùng Pod đơn lẻ?

Pod đơn lẻ thường dùng để:

- Học tập.
- Debug.
- Chạy thử container.
- Kiểm tra image.
- Minh họa cách Kubernetes chạy container.

Trong production, hiếm khi tạo Pod đơn lẻ trực tiếp, vì Pod đơn lẻ không có controller cấp cao quản lý vòng đời.

Nếu xóa một Pod đơn lẻ, Kubernetes không tự tạo lại Pod đó.

## 7.3. Manifest Pod single-container

Tạo file:

```bash
cat <<'EOF' > 01-pod-single-container.yaml
apiVersion: v1
kind: Pod
metadata:
  name: single-nginx
  namespace: lab-workloads
  labels:
    app: single-nginx
    lesson: buoi-03
spec:
  containers:
    - name: nginx
      image: nginx:1.27-alpine
      ports:
        - containerPort: 80
      resources:
        requests:
          cpu: "50m"
          memory: "64Mi"
        limits:
          cpu: "200m"
          memory: "128Mi"
EOF
```

Apply manifest:

```bash
kubectl apply -f 01-pod-single-container.yaml
```

Kiểm tra Pod:

```bash
kubectl get pods -n lab-workloads
kubectl get pod single-nginx -n lab-workloads -o wide
kubectl describe pod single-nginx -n lab-workloads
```

Xem log:

```bash
kubectl logs single-nginx -n lab-workloads
```

Truy cập thử Pod bằng port-forward:

```bash
kubectl port-forward pod/single-nginx -n lab-workloads 8080:80
```

Mở terminal khác trên máy đang chạy `kubectl`:

```bash
curl http://127.0.0.1:8080
```

Dừng port-forward bằng `Ctrl + C`.

## 7.4. Giải thích manifest Pod single-container

### `apiVersion: v1`

Pod thuộc core API group nên dùng `apiVersion: v1`.

### `kind: Pod`

Khai báo object cần tạo là Pod.

### `metadata.name`

```yaml
name: single-nginx
```

Tên Pod trong namespace `lab-workloads`.

### `metadata.namespace`

```yaml
namespace: lab-workloads
```

Pod được tạo trong namespace riêng của bài lab.

### `metadata.labels`

```yaml
labels:
  app: single-nginx
  lesson: buoi-03
```

Gắn label cho Pod. Buổi này dùng label để học cách workload controller chọn Pod.

### `spec.containers`

```yaml
containers:
  - name: nginx
```

Danh sách container trong Pod.

Ở ví dụ này chỉ có một container tên `nginx`.

### `image`

```yaml
image: nginx:1.27-alpine
```

Image container được kubelet yêu cầu container runtime kéo về và chạy.

### `ports.containerPort`

```yaml
ports:
  - containerPort: 80
```

Khai báo container lắng nghe port 80.

Lưu ý:

- Trường này chủ yếu mang tính khai báo metadata trong Pod.
- Nó không tự động expose ứng dụng ra ngoài cluster.
- Cách expose bằng Service sẽ học ở buổi Networking/Service.

### `resources.requests`

```yaml
requests:
  cpu: "50m"
  memory: "64Mi"
```

Khai báo lượng tài nguyên tối thiểu Pod yêu cầu.

Scheduler dựa vào `requests` để quyết định node nào đủ tài nguyên để chạy Pod.

### `resources.limits`

```yaml
limits:
  cpu: "200m"
  memory: "128Mi"
```

Khai báo giới hạn tài nguyên tối đa container được dùng.

Ở buổi này chỉ cần hiểu ở mức cơ bản:

- `requests`: dùng để đặt chỗ tài nguyên khi schedule.
- `limits`: dùng để giới hạn tài nguyên khi container chạy.

Quản lý tài nguyên chi tiết sẽ học ở buổi riêng.

## 7.5. Thử xóa Pod đơn lẻ

Xóa Pod:

```bash
kubectl delete pod single-nginx -n lab-workloads
```

Kiểm tra lại:

```bash
kubectl get pods -n lab-workloads
```

Kết quả kỳ vọng:

```text
No resources found in lab-workloads namespace.
```

Kết luận:

> Pod đơn lẻ bị xóa thì không tự được tạo lại. Muốn duy trì số lượng Pod, cần dùng controller như ReplicaSet hoặc Deployment.

Apply lại để tiếp tục lab nếu cần:

```bash
kubectl apply -f 01-pod-single-container.yaml
```

## 8. Pod multi-container

## 8.1. Khi nào một Pod có nhiều container?

Multi-container Pod dùng khi các container có quan hệ rất chặt chẽ và cần chạy cùng nhau.

Ví dụ phổ biến:

- Main container chạy ứng dụng chính.
- Sidecar container ghi log, đồng bộ file hoặc hỗ trợ ứng dụng chính.
- Proxy container đứng cạnh ứng dụng chính.
- Init-like helper container chuẩn bị dữ liệu cho app.

Điểm quan trọng:

> Không nên gom nhiều ứng dụng độc lập vào cùng một Pod chỉ vì muốn tiết kiệm YAML.

Một Pod multi-container nên đại diện cho một đơn vị ứng dụng logic.

## 8.2. Ví dụ multi-container Pod

Trong ví dụ này:

- Container `web` chạy nginx.
- Container `content-generator` chạy busybox, định kỳ ghi file `index.html`.
- Hai container chia sẻ một volume tạm tên `shared-html`.
- Nginx đọc nội dung từ volume này và phục vụ qua HTTP.

Sơ đồ:

```text
Pod: multi-container-web

+------------------------------+
| Container: web               |
| nginx đọc /usr/share/nginx/html
+------------------------------+
              ^
              | shared volume
              v
+------------------------------+
| Container: content-generator |
| ghi /shared/index.html       |
+------------------------------+
```

Tạo file:

```bash
cat <<'EOF' > 02-pod-multi-container.yaml
apiVersion: v1
kind: Pod
metadata:
  name: multi-container-web
  namespace: lab-workloads
  labels:
    app: multi-container-web
    lesson: buoi-03
spec:
  volumes:
    - name: shared-html
      emptyDir: {}
  containers:
    - name: web
      image: nginx:1.27-alpine
      ports:
        - containerPort: 80
      volumeMounts:
        - name: shared-html
          mountPath: /usr/share/nginx/html
      resources:
        requests:
          cpu: "50m"
          memory: "64Mi"
        limits:
          cpu: "200m"
          memory: "128Mi"

    - name: content-generator
      image: busybox:1.36
      command:
        - /bin/sh
        - -c
      args:
        - |
          while true; do
            echo "<h1>Hello from multi-container Pod</h1><p>Generated at $(date)</p>" > /shared/index.html
            sleep 5
          done
      volumeMounts:
        - name: shared-html
          mountPath: /shared
      resources:
        requests:
          cpu: "10m"
          memory: "32Mi"
        limits:
          cpu: "100m"
          memory: "64Mi"
EOF
```

Apply manifest:

```bash
kubectl apply -f 02-pod-multi-container.yaml
```

Kiểm tra:

```bash
kubectl get pod multi-container-web -n lab-workloads
kubectl describe pod multi-container-web -n lab-workloads
```

Xem log của từng container:

```bash
kubectl logs multi-container-web -n lab-workloads -c web
kubectl logs multi-container-web -n lab-workloads -c content-generator
```

Truy cập thử:

```bash
kubectl port-forward pod/multi-container-web -n lab-workloads 8081:80
```

Mở terminal khác:

```bash
curl http://127.0.0.1:8081
```

Lặp lại nhiều lần:

```bash
watch -n 2 'curl -s http://127.0.0.1:8081'
```

## 8.3. Giải thích manifest multi-container Pod

### `spec.volumes`

```yaml
volumes:
  - name: shared-html
    emptyDir: {}
```

Tạo volume tạm trong vòng đời của Pod.

`emptyDir` được tạo khi Pod được gán vào node và tồn tại cho đến khi Pod bị xóa khỏi node.

Trong bài này, `emptyDir` dùng để minh họa việc chia sẻ dữ liệu giữa các container trong cùng Pod.

Lưu ý:

- `emptyDir` không phải persistent storage.
- Nếu Pod bị xóa và tạo lại, dữ liệu trong `emptyDir` mất.
- Persistent storage sẽ học ở buổi riêng.

### Container `web`

```yaml
- name: web
  image: nginx:1.27-alpine
```

Container chính chạy nginx.

### `volumeMounts` của container `web`

```yaml
volumeMounts:
  - name: shared-html
    mountPath: /usr/share/nginx/html
```

Mount volume `shared-html` vào thư mục mà nginx dùng để phục vụ web.

### Container `content-generator`

```yaml
- name: content-generator
  image: busybox:1.36
```

Container phụ chạy busybox.

### `command` và `args`

```yaml
command:
  - /bin/sh
  - -c
args:
  - |
    while true; do
      echo "<h1>Hello from multi-container Pod</h1><p>Generated at $(date)</p>" > /shared/index.html
      sleep 5
    done
```

Container này chạy vòng lặp vô hạn, cứ 5 giây ghi lại file `index.html`.

### `volumeMounts` của container `content-generator`

```yaml
volumeMounts:
  - name: shared-html
    mountPath: /shared
```

Mount cùng volume `shared-html` vào thư mục `/shared`.

Như vậy:

- Container `content-generator` ghi vào `/shared/index.html`.
- Container `web` đọc từ `/usr/share/nginx/html/index.html`.
- Hai đường dẫn khác nhau nhưng cùng trỏ vào một volume.

## 8.4. Những điểm cần nhớ về multi-container Pod

Một multi-container Pod phù hợp khi các container:

- Cần chạy cùng node.
- Cần chia sẻ network namespace.
- Cần chia sẻ volume.
- Có vòng đời gắn chặt với nhau.
- Cùng phục vụ một đơn vị ứng dụng logic.

Không nên dùng multi-container Pod để chạy các service độc lập như:

- Nginx frontend
- API backend
- Database

Nếu các thành phần này có vòng đời, scale và update khác nhau, nên tách thành workload riêng.

## 9. ReplicaSet

## 9.1. ReplicaSet là gì?

ReplicaSet là workload controller dùng để duy trì một số lượng Pod bản sao ổn định.

Ví dụ:

> Luôn duy trì 3 Pod chạy nginx.

Nếu một Pod bị xóa, ReplicaSet tạo Pod mới.

Nếu số lượng Pod nhiều hơn mong muốn, ReplicaSet xóa bớt.

ReplicaSet quản lý Pod thông qua:

- `spec.replicas`
- `spec.selector`
- `spec.template`

Sơ đồ:

```text
ReplicaSet
 ├── desired replicas: 3
 ├── selector: app=rs-web
 └── template:
      └── Pod nginx

Actual Pods:
 ├── rs-web-xxxxx
 ├── rs-web-yyyyy
 └── rs-web-zzzzz
```

## 9.2. Có nên dùng ReplicaSet trực tiếp không?

Trong thực tế, thường không tạo ReplicaSet trực tiếp.

Thay vào đó, dùng Deployment.

Deployment sẽ tự tạo và quản lý ReplicaSet.

Tuy nhiên, học ReplicaSet là rất cần thiết vì:

- Deployment hoạt động thông qua ReplicaSet.
- ReplicaSet giúp hiểu cơ chế replica.
- ReplicaSet giúp hiểu label selector.
- ReplicaSet giúp hiểu self-healing cơ bản.

## 9.3. Manifest ReplicaSet

Tạo file:

```bash
cat <<'EOF' > 03-replicaset.yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: rs-web
  namespace: lab-workloads
  labels:
    app: rs-web
    lesson: buoi-03
spec:
  replicas: 3
  selector:
    matchLabels:
      app: rs-web
  template:
    metadata:
      labels:
        app: rs-web
        lesson: buoi-03
    spec:
      containers:
        - name: nginx
          image: nginx:1.27-alpine
          ports:
            - containerPort: 80
          resources:
            requests:
              cpu: "50m"
              memory: "64Mi"
            limits:
              cpu: "200m"
              memory: "128Mi"
EOF
```

Apply manifest:

```bash
kubectl apply -f 03-replicaset.yaml
```

Kiểm tra:

```bash
kubectl get rs -n lab-workloads
kubectl get pods -n lab-workloads -l app=rs-web -o wide
```

Kết quả kỳ vọng:

```text
NAME     DESIRED   CURRENT   READY   AGE
rs-web   3         3         3       ...
```

## 9.4. Giải thích manifest ReplicaSet

### `apiVersion: apps/v1`

ReplicaSet thuộc API group `apps`.

### `kind: ReplicaSet`

Khai báo object là ReplicaSet.

### `spec.replicas`

```yaml
replicas: 3
```

Số lượng Pod mong muốn.

### `spec.selector`

```yaml
selector:
  matchLabels:
    app: rs-web
```

ReplicaSet sẽ quản lý các Pod có label `app=rs-web`.

### `spec.template`

```yaml
template:
  metadata:
    labels:
      app: rs-web
```

Template dùng để tạo Pod mới khi cần.

Pod được tạo từ template phải có label khớp với selector.

### Mối quan hệ giữa selector và template label

Đoạn này rất quan trọng:

```yaml
selector:
  matchLabels:
    app: rs-web
template:
  metadata:
    labels:
      app: rs-web
```

Nếu selector là `app=rs-web` thì Pod template cũng phải có `app=rs-web`.

Nếu không khớp, ReplicaSet không thể quản lý đúng Pod.

## 9.5. Thử self-healing với ReplicaSet

Liệt kê Pod:

```bash
kubectl get pods -n lab-workloads -l app=rs-web
```

Xóa một Pod bất kỳ:

```bash
kubectl delete pod <ten-pod-rs-web> -n lab-workloads
```

Quan sát lại:

```bash
kubectl get pods -n lab-workloads -l app=rs-web -w
```

Kết quả:

- Pod bị xóa chuyển sang `Terminating`.
- ReplicaSet tạo Pod mới.
- Số lượng Pod vẫn quay về 3.

Kết luận:

> ReplicaSet không giữ một Pod cụ thể sống mãi. ReplicaSet giữ đúng số lượng Pod mong muốn.

## 9.6. Scale ReplicaSet

Scale lên 5 replicas:

```bash
kubectl scale rs rs-web -n lab-workloads --replicas=5
```

Kiểm tra:

```bash
kubectl get rs rs-web -n lab-workloads
kubectl get pods -n lab-workloads -l app=rs-web
```

Scale xuống 2 replicas:

```bash
kubectl scale rs rs-web -n lab-workloads --replicas=2
```

Kiểm tra:

```bash
kubectl get pods -n lab-workloads -l app=rs-web
```

Đưa về 3 replicas:

```bash
kubectl scale rs rs-web -n lab-workloads --replicas=3
```

## 9.7. Vấn đề cần biết về ReplicaSet

### Không nên cập nhật ứng dụng bằng cách sửa ReplicaSet trực tiếp

ReplicaSet không phải lựa chọn tốt để rollout version ứng dụng.

Ví dụ muốn đổi image từ `nginx:1.27-alpine` sang image khác, nên dùng Deployment.

### Không dùng selector trùng nhau

Nếu hai ReplicaSet có selector chồng lấn, chúng có thể tranh nhau quản lý Pod.

Đây là lỗi thiết kế rất nguy hiểm.

### Không sửa trực tiếp Pod do ReplicaSet tạo

Pod do ReplicaSet quản lý nên được xem là disposable.

Nếu muốn thay đổi cấu hình, thay đổi ở workload object, không sửa trực tiếp từng Pod.

## 10. Deployment

## 10.1. Deployment là gì?

Deployment là workload controller cấp cao dùng phổ biến nhất cho ứng dụng stateless.

Deployment quản lý:

- ReplicaSet
- Pod template
- Số lượng replicas
- Quá trình update ứng dụng
- Lịch sử revision ở mức cơ bản

Sơ đồ:

```text
Deployment
 └── ReplicaSet
      ├── Pod
      ├── Pod
      └── Pod
```

Khi tạo Deployment, Kubernetes thường tạo ReplicaSet bên dưới. ReplicaSet sau đó tạo Pod.

## 10.2. Khi nào dùng Deployment?

Dùng Deployment cho:

- Web frontend stateless.
- API backend stateless.
- Microservice stateless.
- Worker service chạy lâu dài và có thể nhân bản.
- Ứng dụng không yêu cầu danh tính cố định cho từng Pod.

Không nên dùng Deployment cho:

- Database cần identity ổn định.
- Clustered application cần thứ tự khởi động cụ thể.
- Agent bắt buộc chạy trên mọi node.
- Batch job chạy xong rồi dừng.

## 10.3. Manifest Deployment

Tạo file:

```bash
cat <<'EOF' > 04-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: deploy-web
  namespace: lab-workloads
  labels:
    app: deploy-web
    lesson: buoi-03
spec:
  replicas: 3
  revisionHistoryLimit: 5
  selector:
    matchLabels:
      app: deploy-web
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 1
  template:
    metadata:
      labels:
        app: deploy-web
        lesson: buoi-03
    spec:
      containers:
        - name: nginx
          image: nginx:1.27-alpine
          ports:
            - containerPort: 80
          resources:
            requests:
              cpu: "50m"
              memory: "64Mi"
            limits:
              cpu: "200m"
              memory: "128Mi"
EOF
```

Apply manifest:

```bash
kubectl apply -f 04-deployment.yaml
```

Kiểm tra:

```bash
kubectl get deployment -n lab-workloads
kubectl get rs -n lab-workloads -l app=deploy-web
kubectl get pods -n lab-workloads -l app=deploy-web -o wide
```

Xem trạng thái rollout:

```bash
kubectl rollout status deployment/deploy-web -n lab-workloads
```

## 10.4. Giải thích manifest Deployment

### `kind: Deployment`

Khai báo workload là Deployment.

### `spec.replicas`

```yaml
replicas: 3
```

Deployment mong muốn có 3 Pod chạy ứng dụng.

### `spec.revisionHistoryLimit`

```yaml
revisionHistoryLimit: 5
```

Giữ lại tối đa 5 revision cũ để hỗ trợ rollback.

Rolling update và rollback sẽ được học kỹ hơn ở buổi chuyên sâu sau.

### `spec.selector`

```yaml
selector:
  matchLabels:
    app: deploy-web
```

Deployment chọn Pod thông qua label `app=deploy-web`.

### `spec.strategy`

```yaml
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxSurge: 1
    maxUnavailable: 1
```

Cấu hình chiến lược update.

Ở buổi này chỉ cần hiểu cơ bản:

- `RollingUpdate`: cập nhật dần dần.
- `maxSurge: 1`: được phép tạo thêm tối đa 1 Pod vượt số replicas mong muốn trong quá trình update.
- `maxUnavailable: 1`: cho phép tối đa 1 Pod không sẵn sàng trong quá trình update.

Phần này sẽ được học kỹ trong buổi Rolling Update và Rollback.

### `spec.template`

```yaml
template:
  metadata:
    labels:
      app: deploy-web
```

Pod template dùng để tạo Pod.

Deployment không trực tiếp chạy container. Deployment tạo ReplicaSet, ReplicaSet tạo Pod từ template này.

## 10.5. Quan sát Deployment tạo ReplicaSet và Pod

Chạy:

```bash
kubectl get deployment,rs,pod -n lab-workloads -l app=deploy-web
```

Kết quả sẽ có dạng:

```text
deployment.apps/deploy-web
replicaset.apps/deploy-web-xxxxxxxxxx
pod/deploy-web-xxxxxxxxxx-aaaaa
pod/deploy-web-xxxxxxxxxx-bbbbb
pod/deploy-web-xxxxxxxxxx-ccccc
```

Nhận xét:

- Tên Pod bắt đầu bằng tên Deployment.
- Sau đó là hash của ReplicaSet.
- Cuối cùng là suffix ngẫu nhiên của Pod.

## 10.6. Scale Deployment

Scale lên 5 replicas:

```bash
kubectl scale deployment deploy-web -n lab-workloads --replicas=5
```

Kiểm tra:

```bash
kubectl get deployment deploy-web -n lab-workloads
kubectl get pods -n lab-workloads -l app=deploy-web
```

Scale xuống 2 replicas:

```bash
kubectl scale deployment deploy-web -n lab-workloads --replicas=2
```

Đưa về 3 replicas:

```bash
kubectl scale deployment deploy-web -n lab-workloads --replicas=3
```

## 10.7. Vấn đề cần biết về Deployment

### Deployment phù hợp nhất với stateless application

Ứng dụng stateless là ứng dụng không phụ thuộc vào dữ liệu cục bộ của từng Pod.

Nếu Pod bị xóa và Pod khác được tạo lại, ứng dụng vẫn hoạt động bình thường.

### Không sửa trực tiếp ReplicaSet do Deployment tạo

ReplicaSet bên dưới Deployment do Deployment quản lý.

Nếu cần thay đổi, sửa Deployment.

### Không sửa trực tiếp Pod do Deployment tạo

Pod là kết quả của Deployment và ReplicaSet.

Nếu sửa trực tiếp Pod, thay đổi có thể mất khi Pod bị tạo lại.

### Deployment không tự expose ứng dụng ra ngoài

Deployment chỉ quản lý Pod.

Muốn truy cập ổn định đến Pod cần Service. Service sẽ học ở buổi Networking/Service.

Trong bài này, nếu cần test nhanh, ta dùng `kubectl port-forward`:

```bash
kubectl port-forward deployment/deploy-web -n lab-workloads 8082:80
```

Test:

```bash
curl http://127.0.0.1:8082
```

## 11. DaemonSet

## 11.1. DaemonSet là gì?

DaemonSet đảm bảo mỗi node phù hợp sẽ chạy một bản sao của Pod.

Dùng DaemonSet khi cần chạy agent hoặc daemon ở cấp node.

Ví dụ:

- Log collector trên mỗi node.
- Monitoring agent trên mỗi node.
- Storage daemon trên mỗi node.
- Network component trên mỗi node.
- Security agent trên mỗi node.

Sơ đồ với cluster 2 worker:

```text
DaemonSet: node-agent

k8s-master-01   control-plane    thường không chạy workload do taint
k8s-worker-01   worker           node-agent Pod
k8s-worker-02   worker           node-agent Pod
```

Lưu ý:

Trong cluster kubeadm, master node thường có taint để không schedule workload thông thường lên control plane. Vì vậy DaemonSet trong bài này có thể chỉ chạy trên 2 worker node.

## 11.2. Manifest DaemonSet

Tạo file:

```bash
cat <<'EOF' > 05-daemonset.yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: ds-node-agent
  namespace: lab-workloads
  labels:
    app: ds-node-agent
    lesson: buoi-03
spec:
  selector:
    matchLabels:
      app: ds-node-agent
  template:
    metadata:
      labels:
        app: ds-node-agent
        lesson: buoi-03
    spec:
      containers:
        - name: node-agent
          image: busybox:1.36
          command:
            - /bin/sh
            - -c
          args:
            - |
              while true; do
                echo "Node agent is running on $(hostname) at $(date)"
                sleep 30
              done
          resources:
            requests:
              cpu: "10m"
              memory: "32Mi"
            limits:
              cpu: "100m"
              memory: "64Mi"
EOF
```

Apply manifest:

```bash
kubectl apply -f 05-daemonset.yaml
```

Kiểm tra:

```bash
kubectl get daemonset -n lab-workloads
kubectl get pods -n lab-workloads -l app=ds-node-agent -o wide
```

Kết quả kỳ vọng:

- Nếu master node có taint mặc định, có thể thấy 2 Pod trên 2 worker.
- Nếu master node cho phép schedule workload, có thể thấy 3 Pod trên cả 3 node.

Xem log một Pod:

```bash
kubectl logs <ten-pod-ds-node-agent> -n lab-workloads
```

## 11.3. Giải thích manifest DaemonSet

### `kind: DaemonSet`

Khai báo workload là DaemonSet.

### Không có `replicas`

DaemonSet không dùng `replicas`.

Số Pod phụ thuộc vào số node phù hợp.

Nếu có 2 worker phù hợp, DaemonSet tạo 2 Pod.

Nếu thêm worker node thứ ba, DaemonSet sẽ tự tạo thêm Pod trên worker mới.

### `spec.selector`

```yaml
selector:
  matchLabels:
    app: ds-node-agent
```

DaemonSet dùng selector để quản lý Pod có label `app=ds-node-agent`.

### `spec.template`

Pod template mô tả Pod agent sẽ chạy trên các node phù hợp.

### Vì sao dùng `busybox`

Ở bài này, `busybox` chỉ dùng để tạo một container nhẹ chạy vòng lặp và ghi log.

Trong thực tế, DaemonSet thường chạy các agent thật như:

- Fluent Bit
- Promtail
- Node Exporter
- Calico node
- Storage plugin node agent

## 11.4. Vấn đề cần biết về DaemonSet

### DaemonSet chạy theo node, không chạy theo số replicas

Không scale DaemonSet bằng `kubectl scale` như Deployment.

DaemonSet scale theo số node phù hợp.

### DaemonSet thường dùng cho hạ tầng

DaemonSet phù hợp với workload cấp node.

Không nên dùng DaemonSet cho web application thông thường.

### Control plane node có thể không chạy DaemonSet

Nếu control plane node có taint, DaemonSet không chạy trên đó trừ khi Pod có toleration phù hợp.

Taints và tolerations sẽ học kỹ ở buổi Scheduler nâng cao.

## 12. StatefulSet

## 12.1. StatefulSet là gì?

StatefulSet dùng để quản lý ứng dụng stateful.

StatefulSet cung cấp:

- Tên Pod ổn định.
- Thứ tự tạo Pod ổn định.
- Thứ tự xóa Pod ổn định.
- Danh tính mạng ổn định khi kết hợp với headless Service.
- Khả năng gắn storage ổn định khi kết hợp với PersistentVolumeClaim.

Ví dụ tên Pod của StatefulSet:

```text
sts-web-0
sts-web-1
sts-web-2
```

Khác với Deployment, Pod của StatefulSet không có suffix ngẫu nhiên.

## 12.2. Khi nào dùng StatefulSet?

Dùng StatefulSet cho:

- Database.
- Message queue cần identity ổn định.
- Distributed system cần thứ tự node.
- Clustered application cần hostname ổn định.
- Ứng dụng cần storage gắn với từng replica.

Ví dụ:

- PostgreSQL cluster
- MongoDB replica set
- Kafka
- ZooKeeper
- Elasticsearch

Trong bài này, chúng ta chỉ minh họa identity ổn định của StatefulSet, chưa triển khai persistent storage.

Storage sẽ học ở buổi riêng.

## 12.3. Vì sao StatefulSet cần Service?

StatefulSet cần trường:

```yaml
serviceName: sts-web-headless
```

Trường này trỏ tới một Service dùng để quản lý định danh mạng cho Pod.

Để tránh đi quá sâu vào Service ở buổi này, ta chỉ dùng một headless Service tối thiểu.

Headless Service có:

```yaml
clusterIP: None
```

Nó không tạo một virtual IP kiểu Service thông thường, mà chủ yếu phục vụ DNS record cho Pod phía sau.

Service sẽ được học kỹ ở buổi Networking/Service.

## 12.4. Manifest StatefulSet tối thiểu

Tạo file:

```bash
cat <<'EOF' > 06-statefulset.yaml
apiVersion: v1
kind: Service
metadata:
  name: sts-web-headless
  namespace: lab-workloads
  labels:
    app: sts-web
    lesson: buoi-03
spec:
  clusterIP: None
  selector:
    app: sts-web
  ports:
    - name: http
      port: 80
      targetPort: 80
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: sts-web
  namespace: lab-workloads
  labels:
    app: sts-web
    lesson: buoi-03
spec:
  serviceName: sts-web-headless
  replicas: 2
  selector:
    matchLabels:
      app: sts-web
  template:
    metadata:
      labels:
        app: sts-web
        lesson: buoi-03
    spec:
      containers:
        - name: nginx
          image: nginx:1.27-alpine
          ports:
            - name: http
              containerPort: 80
          resources:
            requests:
              cpu: "50m"
              memory: "64Mi"
            limits:
              cpu: "200m"
              memory: "128Mi"
EOF
```

Apply manifest:

```bash
kubectl apply -f 06-statefulset.yaml
```

Kiểm tra:

```bash
kubectl get statefulset -n lab-workloads
kubectl get pods -n lab-workloads -l app=sts-web -o wide
```

Kết quả kỳ vọng:

```text
sts-web-0
sts-web-1
```

## 12.5. Giải thích manifest StatefulSet

### Headless Service

```yaml
apiVersion: v1
kind: Service
metadata:
  name: sts-web-headless
```

Tạo Service tên `sts-web-headless`.

Ở buổi này chỉ cần hiểu đây là Service phục vụ định danh mạng cho StatefulSet.

### `clusterIP: None`

```yaml
clusterIP: None
```

Khai báo đây là headless Service.

### `selector`

```yaml
selector:
  app: sts-web
```

Service chọn các Pod có label `app=sts-web`.

### `kind: StatefulSet`

```yaml
kind: StatefulSet
```

Khai báo workload là StatefulSet.

### `serviceName`

```yaml
serviceName: sts-web-headless
```

StatefulSet dùng Service này để cung cấp định danh mạng ổn định.

### `replicas`

```yaml
replicas: 2
```

Tạo 2 Pod:

```text
sts-web-0
sts-web-1
```

### `selector` và `template`

Giống ReplicaSet và Deployment, StatefulSet cũng dùng selector để quản lý Pod.

```yaml
selector:
  matchLabels:
    app: sts-web
template:
  metadata:
    labels:
      app: sts-web
```

Hai phần này phải khớp nhau.

## 12.6. Kiểm tra identity ổn định

Liệt kê Pod:

```bash
kubectl get pods -n lab-workloads -l app=sts-web
```

Xóa Pod `sts-web-0`:

```bash
kubectl delete pod sts-web-0 -n lab-workloads
```

Quan sát:

```bash
kubectl get pods -n lab-workloads -l app=sts-web -w
```

Pod mới vẫn tên là:

```text
sts-web-0
```

Kết luận:

> StatefulSet duy trì danh tính ổn định cho từng replica.

## 12.7. Kiểm tra DNS nội bộ của StatefulSet

Chạy Pod tạm để kiểm tra DNS:

```bash
kubectl run dns-test -n lab-workloads --image=busybox:1.36 --rm -it --restart=Never -- sh
```

Trong shell của Pod `dns-test`, chạy:

```bash
nslookup sts-web-0.sts-web-headless.lab-workloads.svc.cluster.local
nslookup sts-web-1.sts-web-headless.lab-workloads.svc.cluster.local
```

Thoát khỏi Pod:

```bash
exit
```

Nếu DNS hoạt động,  sẽ thấy tên Pod StatefulSet có thể resolve được.

Lưu ý:

- Nội dung này chỉ dùng để minh họa StatefulSet identity.
- DNS và Service sẽ học chi tiết ở buổi networking.

## 12.8. Vấn đề cần biết về StatefulSet

### StatefulSet không tự làm ứng dụng trở thành database cluster

StatefulSet chỉ cung cấp cơ chế định danh, ordering và quản lý Pod.

Ứng dụng bên trong vẫn phải hỗ trợ clustering hoặc replication.

Ví dụ:

- StatefulSet không tự biến PostgreSQL đơn lẻ thành PostgreSQL HA.
- StatefulSet không tự cấu hình MongoDB replica set.
- StatefulSet không tự cấu hình Kafka cluster.

### StatefulSet thường cần persistent storage

Trong production, StatefulSet thường đi với:

```yaml
volumeClaimTemplates:
```

Nhưng phần này cần kiến thức PersistentVolume, PersistentVolumeClaim và StorageClass.

Vì vậy bài này chưa dùng PVC để tránh vượt tiến trình.

### StatefulSet khác Deployment ở identity

Deployment tạo Pod tên ngẫu nhiên:

```text
deploy-web-xxxx-yyyy
```

StatefulSet tạo Pod tên ổn định:

```text
sts-web-0
sts-web-1
```

## 13. Job

## 13.1. Job là gì?

Job dùng để chạy tác vụ một lần đến khi hoàn tất.

Khác với Deployment:

- Deployment chạy ứng dụng lâu dài.
- Job chạy task, hoàn tất rồi dừng.

Ví dụ Job:

- Backup database.
- Generate report.
- Import dữ liệu.
- Migration database schema.
- Batch processing.
- Chạy một script xử lý dữ liệu.

## 13.2. Manifest Job thành công

Tạo file:

```bash
cat <<'EOF' > 07-job-success.yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: job-hello
  namespace: lab-workloads
  labels:
    app: job-hello
    lesson: buoi-03
spec:
  completions: 1
  parallelism: 1
  backoffLimit: 3
  ttlSecondsAfterFinished: 300
  template:
    metadata:
      labels:
        app: job-hello
    spec:
      restartPolicy: Never
      containers:
        - name: hello
          image: busybox:1.36
          command:
            - /bin/sh
            - -c
          args:
            - |
              echo "Job started at $(date)"
              sleep 5
              echo "Job completed successfully at $(date)"
EOF
```

Apply:

```bash
kubectl apply -f 07-job-success.yaml
```

Kiểm tra:

```bash
kubectl get jobs -n lab-workloads
kubectl get pods -n lab-workloads -l app=job-hello
```

Xem log:

```bash
kubectl logs -n lab-workloads -l app=job-hello
```

Kết quả kỳ vọng:

```text
Job started at ...
Job completed successfully at ...
```

## 13.3. Giải thích manifest Job

### `apiVersion: batch/v1`

Job thuộc API group `batch`.

### `kind: Job`

Khai báo workload là Job.

### `spec.completions`

```yaml
completions: 1
```

Số lần task cần hoàn tất thành công.

### `spec.parallelism`

```yaml
parallelism: 1
```

Số Pod được chạy song song.

Ví dụ:

- `completions: 5`
- `parallelism: 2`

Ý nghĩa: cần 5 lần hoàn tất thành công, nhưng chỉ chạy tối đa 2 Pod cùng lúc.

### `spec.backoffLimit`

```yaml
backoffLimit: 3
```

Số lần retry trước khi Job bị xem là failed.

### `ttlSecondsAfterFinished`

```yaml
ttlSecondsAfterFinished: 300
```

Sau khi Job hoàn tất, Kubernetes có thể tự dọn Job sau 300 giây.

Trường này giúp môi trường lab không bị quá nhiều Job cũ.

### `restartPolicy`

```yaml
restartPolicy: Never
```

Job yêu cầu `restartPolicy` là `Never` hoặc `OnFailure`.

Không dùng `Always` cho Job.

## 13.4. Manifest Job thất bại để quan sát retry

Tạo file:

```bash
cat <<'EOF' > 08-job-fail.yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: job-fail
  namespace: lab-workloads
  labels:
    app: job-fail
    lesson: buoi-03
spec:
  backoffLimit: 2
  ttlSecondsAfterFinished: 300
  template:
    metadata:
      labels:
        app: job-fail
    spec:
      restartPolicy: Never
      containers:
        - name: fail
          image: busybox:1.36
          command:
            - /bin/sh
            - -c
          args:
            - |
              echo "This job will fail intentionally"
              exit 1
EOF
```

Apply:

```bash
kubectl apply -f 08-job-fail.yaml
```

Quan sát:

```bash
kubectl get jobs -n lab-workloads
kubectl get pods -n lab-workloads -l app=job-fail
kubectl describe job job-fail -n lab-workloads
```

Kết quả:

- Job tạo Pod.
- Pod thất bại.
- Job retry theo `backoffLimit`.
- Sau khi vượt quá số lần retry, Job chuyển trạng thái failed.

## 13.5. Vấn đề cần biết về Job

### Job không dành cho ứng dụng chạy mãi

Nếu container trong Job chạy vô hạn, Job sẽ không hoàn tất.

### Job giữ Pod đã hoàn tất để xem log

Sau khi Job hoàn thành, Pod có thể ở trạng thái `Completed`.

Điều này hữu ích để xem log sau khi tác vụ chạy xong.

### Cần dọn Job cũ

Nếu tạo nhiều Job, cluster có thể có nhiều Pod `Completed`.

Có thể dọn bằng:

```bash
kubectl delete job job-hello -n lab-workloads
kubectl delete job job-fail -n lab-workloads
```

Hoặc dùng:

```yaml
ttlSecondsAfterFinished: 300
```

## 14. CronJob

## 14.1. CronJob là gì?

CronJob dùng để tạo Job theo lịch định kỳ.

Ví dụ:

- Chạy backup mỗi ngày lúc 01:00.
- Generate report mỗi giờ.
- Gửi email nhắc việc mỗi sáng.
- Đồng bộ dữ liệu mỗi 5 phút.
- Dọn file tạm mỗi đêm.

CronJob tương tự crontab trong Linux, nhưng thay vì chạy command trực tiếp trên một server, nó tạo Job trong Kubernetes.

## 14.2. Cron format cơ bản

CronJob dùng format:

```text
* * * * *
| | | | |
| | | | +--- day of week
| | | +----- month
| | +------- day of month
| +--------- hour
+----------- minute
```

Ví dụ:

| Schedule | Ý nghĩa |
|---|---|
| `*/1 * * * *` | Mỗi phút |
| `0 * * * *` | Mỗi giờ tại phút 0 |
| `0 1 * * *` | Mỗi ngày lúc 01:00 |
| `0 1 * * 0` | Mỗi Chủ nhật lúc 01:00 |

## 14.3. Manifest CronJob

Tạo file:

```bash
cat <<'EOF' > 09-cronjob.yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: cronjob-hello
  namespace: lab-workloads
  labels:
    app: cronjob-hello
    lesson: buoi-03
spec:
  schedule: "*/1 * * * *"
  timeZone: "Asia/Ho_Chi_Minh"
  concurrencyPolicy: Forbid
  successfulJobsHistoryLimit: 3
  failedJobsHistoryLimit: 1
  jobTemplate:
    spec:
      backoffLimit: 2
      ttlSecondsAfterFinished: 300
      template:
        metadata:
          labels:
            app: cronjob-hello
        spec:
          restartPolicy: Never
          containers:
            - name: hello
              image: busybox:1.36
              command:
                - /bin/sh
                - -c
              args:
                - |
                  echo "CronJob executed at $(date)"
EOF
```

Apply:

```bash
kubectl apply -f 09-cronjob.yaml
```

Kiểm tra:

```bash
kubectl get cronjob -n lab-workloads
kubectl get jobs -n lab-workloads
```

Đợi khoảng 1 đến 2 phút rồi kiểm tra lại:

```bash
kubectl get jobs -n lab-workloads
kubectl get pods -n lab-workloads -l app=cronjob-hello
```

Xem log Pod mới nhất:

```bash
kubectl logs -n lab-workloads -l app=cronjob-hello --tail=20
```

## 14.4. Giải thích manifest CronJob

### `kind: CronJob`

Khai báo workload là CronJob.

### `schedule`

```yaml
schedule: "*/1 * * * *"
```

Chạy mỗi phút.

Trong production không nên chạy quá dày nếu task nặng.

### `timeZone`

```yaml
timeZone: "Asia/Ho_Chi_Minh"
```

Chỉ định timezone cho lịch chạy.

Điều này giúp lịch chạy rõ ràng hơn trong môi trường Việt Nam.

### `concurrencyPolicy`

```yaml
concurrencyPolicy: Forbid
```

Không cho phép Job mới chạy nếu Job trước đó chưa xong.

Các giá trị thường gặp:

| Giá trị | Ý nghĩa |
|---|---|
| `Allow` | Cho phép chạy chồng nhiều Job |
| `Forbid` | Không chạy Job mới nếu Job cũ chưa xong |
| `Replace` | Job mới thay thế Job cũ đang chạy |

### `successfulJobsHistoryLimit`

```yaml
successfulJobsHistoryLimit: 3
```

Giữ lại tối đa 3 Job thành công gần nhất.

### `failedJobsHistoryLimit`

```yaml
failedJobsHistoryLimit: 1
```

Giữ lại tối đa 1 Job thất bại gần nhất.

### `jobTemplate`

```yaml
jobTemplate:
  spec:
    template:
      spec:
        containers:
```

CronJob không trực tiếp tạo Pod.

CronJob tạo Job.

Job tạo Pod.

Sơ đồ:

```text
CronJob
 └── Job
      └── Pod
```

## 14.5. Suspend CronJob

Tạm dừng CronJob:

```bash
kubectl patch cronjob cronjob-hello -n lab-workloads -p '{"spec":{"suspend":true}}'
```

Kiểm tra:

```bash
kubectl get cronjob cronjob-hello -n lab-workloads
```

Bật lại:

```bash
kubectl patch cronjob cronjob-hello -n lab-workloads -p '{"spec":{"suspend":false}}'
```

## 14.6. Vấn đề cần biết về CronJob

### CronJob có thể tạo nhiều Job nếu lịch chạy quá dày

Nếu task chạy lâu hơn chu kỳ schedule, có thể xảy ra chạy chồng.

Dùng `concurrencyPolicy: Forbid` để tránh tình huống này.

### CronJob không phù hợp cho workload chạy liên tục

Nếu ứng dụng cần chạy liên tục, dùng Deployment, StatefulSet hoặc DaemonSet tùy trường hợp.

### Cần kiểm soát lịch sử Job

Nếu không giới hạn history, cluster có thể tích tụ nhiều Job và Pod Completed.

Nên cấu hình:

```yaml
successfulJobsHistoryLimit:
failedJobsHistoryLimit:
ttlSecondsAfterFinished:
```

## 15. Bảng so sánh các workload

| Workload | Dùng để làm gì | Có controller duy trì không | Tên Pod ổn định không | Chạy liên tục không |
|---|---|---|---|---|
| Pod | Chạy container trực tiếp | Không, nếu tạo Pod đơn lẻ | Có tên do user đặt, nhưng không được tái tạo nếu xóa | Có thể |
| ReplicaSet | Duy trì số lượng Pod | Có | Không | Có |
| Deployment | Ứng dụng stateless | Có | Không | Có |
| DaemonSet | Chạy agent trên mỗi node | Có | Không | Có |
| StatefulSet | Ứng dụng stateful cần identity | Có | Có | Có |
| Job | Task chạy đến khi hoàn tất | Có | Không | Không |
| CronJob | Job chạy theo lịch | Có | Không | Không, chỉ chạy theo lịch |

## 16. Cách chọn workload phù hợp

### 16.1. Web/API stateless

Dùng:

```text
Deployment
```

Ví dụ:

- Nginx web frontend
- Node.js API
- Java Spring Boot API
- Python FastAPI backend

### 16.2. Agent chạy trên mỗi node

Dùng:

```text
DaemonSet
```

Ví dụ:

- Log collector
- Node monitoring
- Security agent
- Network node plugin

### 16.3. Database hoặc ứng dụng cần identity

Dùng:

```text
StatefulSet
```

Ví dụ:

- PostgreSQL
- MongoDB
- Kafka
- ZooKeeper

Lưu ý: cần học thêm storage trước khi triển khai production.

### 16.4. Task chạy một lần

Dùng:

```text
Job
```

Ví dụ:

- Database migration
- Import data
- Batch report

### 16.5. Task chạy theo lịch

Dùng:

```text
CronJob
```

Ví dụ:

- Backup hằng ngày
- Dọn log định kỳ
- Report định kỳ

## 17. Các vấn đề quan trọng cần biết khi làm việc với workload

## 17.1. Pod là disposable

Pod nên được xem là có thể bị xóa và tạo lại bất cứ lúc nào.

Không nên lưu dữ liệu quan trọng chỉ trong filesystem của container hoặc `emptyDir`.

## 17.2. Không nên dùng naked Pod cho production

Naked Pod là Pod được tạo trực tiếp, không nằm dưới controller như Deployment, ReplicaSet, DaemonSet, StatefulSet hoặc Job.

Nếu naked Pod bị xóa, Kubernetes không tự tạo lại.

## 17.3. Selector phải khớp với template label

Đây là lỗi thường gặp nhất.

Sai:

```yaml
selector:
  matchLabels:
    app: web
template:
  metadata:
    labels:
      app: api
```

Đúng:

```yaml
selector:
  matchLabels:
    app: web
template:
  metadata:
    labels:
      app: web
```

## 17.4. Không dùng selector chồng lấn

Không nên có hai controller cùng chọn một nhóm Pod.

Ví dụ nguy hiểm:

```yaml
ReplicaSet A selector: app=web
ReplicaSet B selector: app=web
```

Hai controller có thể tranh nhau quản lý Pod.

## 17.5. Không sửa trực tiếp Pod do controller tạo

Nếu Pod được tạo bởi Deployment, ReplicaSet, DaemonSet hoặc StatefulSet, không nên sửa trực tiếp Pod.

Hãy sửa workload object.

## 17.6. Restart policy khác nhau theo workload

| Workload | restartPolicy thường dùng |
|---|---|
| Deployment | `Always` hoặc bỏ trống |
| ReplicaSet | `Always` hoặc bỏ trống |
| DaemonSet | `Always` hoặc bỏ trống |
| StatefulSet | `Always` hoặc bỏ trống |
| Job | `Never` hoặc `OnFailure` |
| CronJob | `Never` hoặc `OnFailure` trong jobTemplate |

Với Deployment, ReplicaSet, DaemonSet, StatefulSet, nếu bỏ trống `restartPolicy`, Kubernetes mặc định là `Always`.

## 17.7. Số replicas không đồng nghĩa với số node

Cluster có 2 worker nhưng Deployment có thể chạy 5 replicas.

Kubernetes có thể đặt nhiều Pod trên cùng một node nếu tài nguyên đủ.

Ví dụ:

```bash
kubectl scale deployment deploy-web -n lab-workloads --replicas=5
kubectl get pods -n lab-workloads -l app=deploy-web -o wide
```

 sẽ thấy có thể có nhiều Pod nằm trên cùng một worker.

## 17.8. Trạng thái lỗi thường gặp của Pod

| Trạng thái | Ý nghĩa thường gặp |
|---|---|
| `Pending` | Pod chưa được schedule hoặc chưa kéo image xong |
| `ContainerCreating` | Kubelet đang tạo container |
| `Running` | Pod đang chạy |
| `Completed` | Pod của Job đã chạy xong |
| `CrashLoopBackOff` | Container chạy rồi crash lặp lại |
| `ImagePullBackOff` | Không kéo được image |
| `ErrImagePull` | Lỗi pull image ban đầu |
| `Terminating` | Pod đang bị xóa |

Lệnh kiểm tra quan trọng:

```bash
kubectl describe pod <pod-name> -n lab-workloads
kubectl logs <pod-name> -n lab-workloads
kubectl get events -n lab-workloads --sort-by=.lastTimestamp
```

## 18. Lab tổng hợp buổi 03

## 18.1. Mục tiêu lab

Sau lab này,  phải:

- Tạo được namespace lab.
- Tạo được Pod single-container.
- Tạo được Pod multi-container.
- Tạo được ReplicaSet và quan sát self-healing.
- Tạo được Deployment và quan sát Deployment tạo ReplicaSet.
- Tạo được DaemonSet và quan sát Pod chạy trên các node.
- Tạo được StatefulSet và quan sát identity ổn định.
- Tạo được Job thành công và Job thất bại.
- Tạo được CronJob chạy mỗi phút.
- Biết dọn toàn bộ tài nguyên sau lab.

## 18.2. Chuẩn bị

```bash
cd ~/k8s-course/buoi-03-workloads
kubectl get nodes -o wide
kubectl get pods -A
```

Apply namespace:

```bash
kubectl apply -f 00-namespace.yaml
```

## 18.3. Lab 1 - Pod single-container

```bash
kubectl apply -f 01-pod-single-container.yaml
kubectl get pod single-nginx -n lab-workloads -o wide
kubectl describe pod single-nginx -n lab-workloads
```

Test nhanh:

```bash
kubectl port-forward pod/single-nginx -n lab-workloads 8080:80
```

Terminal khác:

```bash
curl http://127.0.0.1:8080
```

Dừng port-forward bằng `Ctrl + C`.

## 18.4. Lab 2 - Pod multi-container

```bash
kubectl apply -f 02-pod-multi-container.yaml
kubectl get pod multi-container-web -n lab-workloads
kubectl logs multi-container-web -n lab-workloads -c content-generator
```

Test:

```bash
kubectl port-forward pod/multi-container-web -n lab-workloads 8081:80
```

Terminal khác:

```bash
curl http://127.0.0.1:8081
```

## 18.5. Lab 3 - ReplicaSet self-healing

```bash
kubectl apply -f 03-replicaset.yaml
kubectl get rs -n lab-workloads
kubectl get pods -n lab-workloads -l app=rs-web
```

Xóa một Pod:

```bash
kubectl delete pod <ten-pod-rs-web> -n lab-workloads
```

Quan sát:

```bash
kubectl get pods -n lab-workloads -l app=rs-web -w
```

Dừng watch bằng `Ctrl + C`.

## 18.6. Lab 4 - Deployment

```bash
kubectl apply -f 04-deployment.yaml
kubectl get deployment -n lab-workloads
kubectl get rs -n lab-workloads -l app=deploy-web
kubectl get pods -n lab-workloads -l app=deploy-web -o wide
kubectl rollout status deployment/deploy-web -n lab-workloads
```

Scale:

```bash
kubectl scale deployment deploy-web -n lab-workloads --replicas=5
kubectl get pods -n lab-workloads -l app=deploy-web -o wide
```

Đưa về 3:

```bash
kubectl scale deployment deploy-web -n lab-workloads --replicas=3
```

## 18.7. Lab 5 - DaemonSet

```bash
kubectl apply -f 05-daemonset.yaml
kubectl get daemonset -n lab-workloads
kubectl get pods -n lab-workloads -l app=ds-node-agent -o wide
```

Xem log:

```bash
kubectl logs -n lab-workloads -l app=ds-node-agent --tail=20
```

Câu hỏi cho :

- Vì sao DaemonSet có thể không chạy trên master node?
- Nếu thêm một worker node mới, điều gì sẽ xảy ra với DaemonSet?

## 18.8. Lab 6 - StatefulSet

```bash
kubectl apply -f 06-statefulset.yaml
kubectl get statefulset -n lab-workloads
kubectl get pods -n lab-workloads -l app=sts-web -o wide
```

Xóa Pod:

```bash
kubectl delete pod sts-web-0 -n lab-workloads
kubectl get pods -n lab-workloads -l app=sts-web -w
```

Dừng watch bằng `Ctrl + C`.

Kiểm tra DNS nội bộ:

```bash
kubectl run dns-test -n lab-workloads --image=busybox:1.36 --rm -it --restart=Never -- sh
```

Trong Pod:

```bash
nslookup sts-web-0.sts-web-headless.lab-workloads.svc.cluster.local
nslookup sts-web-1.sts-web-headless.lab-workloads.svc.cluster.local
exit
```

## 18.9. Lab 7 - Job

Job thành công:

```bash
kubectl apply -f 07-job-success.yaml
kubectl get jobs -n lab-workloads
kubectl get pods -n lab-workloads -l app=job-hello
kubectl logs -n lab-workloads -l app=job-hello
```

Job thất bại:

```bash
kubectl apply -f 08-job-fail.yaml
kubectl get jobs -n lab-workloads
kubectl get pods -n lab-workloads -l app=job-fail
kubectl describe job job-fail -n lab-workloads
```

## 18.10. Lab 8 - CronJob

```bash
kubectl apply -f 09-cronjob.yaml
kubectl get cronjob -n lab-workloads
```

Đợi 1 đến 2 phút:

```bash
kubectl get jobs -n lab-workloads
kubectl get pods -n lab-workloads -l app=cronjob-hello
kubectl logs -n lab-workloads -l app=cronjob-hello --tail=20
```

Tạm dừng CronJob:

```bash
kubectl patch cronjob cronjob-hello -n lab-workloads -p '{"spec":{"suspend":true}}'
kubectl get cronjob cronjob-hello -n lab-workloads
```

## 18.11. Lab 9 - Quan sát toàn bộ workload

```bash
kubectl get all -n lab-workloads
```

Lưu ý:

`kubectl get all` không hiển thị tất cả mọi loại object trong Kubernetes, nhưng đủ để quan sát nhiều workload cơ bản trong bài này.

Kiểm tra riêng StatefulSet:

```bash
kubectl get statefulset -n lab-workloads
```

Kiểm tra riêng CronJob:

```bash
kubectl get cronjob -n lab-workloads
```

## 18.12. Cleanup toàn bộ lab

Cách nhanh nhất:

```bash
kubectl delete namespace lab-workloads
```

Kiểm tra:

```bash
kubectl get namespace lab-workloads
```

Nếu muốn xóa từng file:

```bash
kubectl delete -f 09-cronjob.yaml
kubectl delete -f 08-job-fail.yaml
kubectl delete -f 07-job-success.yaml
kubectl delete -f 06-statefulset.yaml
kubectl delete -f 05-daemonset.yaml
kubectl delete -f 04-deployment.yaml
kubectl delete -f 03-replicaset.yaml
kubectl delete -f 02-pod-multi-container.yaml
kubectl delete -f 01-pod-single-container.yaml
kubectl delete -f 00-namespace.yaml
```

## 19. Câu hỏi kiểm tra cuối buổi

### Câu 1

Pod là gì trong Kubernetes?

Gợi ý trả lời:

- Pod là đơn vị triển khai nhỏ nhất.
- Pod chứa một hoặc nhiều container.
- Các container trong cùng Pod chia sẻ network namespace và có thể chia sẻ volume.

### Câu 2

Vì sao không nên dùng Pod đơn lẻ cho production?

Gợi ý trả lời:

- Nếu Pod bị xóa, Kubernetes không tự tạo lại.
- Không có controller duy trì desired state.
- Nên dùng Deployment, StatefulSet, DaemonSet hoặc Job tùy workload.

### Câu 3

ReplicaSet duy trì Pod bằng cách nào?

Gợi ý trả lời:

- Dựa vào `replicas`, `selector` và `template`.
- Nếu số Pod ít hơn desired replicas, ReplicaSet tạo thêm Pod.
- Nếu số Pod nhiều hơn desired replicas, ReplicaSet xóa bớt.

### Câu 4

Deployment khác ReplicaSet ở điểm nào?

Gợi ý trả lời:

- Deployment là cấp cao hơn.
- Deployment quản lý ReplicaSet.
- Deployment hỗ trợ declarative update và revision history.
- Thực tế nên dùng Deployment cho ứng dụng stateless thay vì tạo ReplicaSet trực tiếp.

### Câu 5

DaemonSet phù hợp với loại workload nào?

Gợi ý trả lời:

- Workload cần chạy trên mỗi node hoặc một nhóm node.
- Ví dụ log collector, monitoring agent, storage daemon, network component.

### Câu 6

StatefulSet khác Deployment ở điểm nào?

Gợi ý trả lời:

- StatefulSet tạo Pod có tên ổn định như `app-0`, `app-1`.
- StatefulSet phù hợp với ứng dụng stateful.
- StatefulSet thường đi với headless Service và persistent storage.
- Deployment phù hợp hơn cho ứng dụng stateless.

### Câu 7

Job và CronJob khác nhau thế nào?

Gợi ý trả lời:

- Job chạy task một lần đến khi hoàn tất.
- CronJob tạo Job theo lịch định kỳ.
- CronJob giống crontab nhưng chạy trong Kubernetes.

### Câu 8

Vì sao selector và label trong template phải khớp nhau?

Gợi ý trả lời:

- Controller dùng selector để tìm Pod cần quản lý.
- Nếu selector không khớp label, controller không quản lý đúng Pod.
- Kubernetes có thể từ chối manifest hoặc workload hoạt động sai.

### Câu 9

`restartPolicy` của Job nên là gì?

Gợi ý trả lời:

- `Never` hoặc `OnFailure`.
- Không dùng `Always` cho Job.

### Câu 10

Nếu Deployment có 5 replicas nhưng cluster chỉ có 2 worker node thì có chạy được không?

Gợi ý trả lời:

- Có thể chạy nếu tài nguyên đủ.
- Nhiều Pod có thể được schedule trên cùng một node.
- Số replicas không bắt buộc bằng số node.

## 20. Tổng kết buổi học

Sau buổi này,  đã hiểu các workload resource quan trọng nhất trong Kubernetes.

Các điểm cần nhớ:

- Pod là đơn vị chạy container nhỏ nhất.
- Pod có thể chạy một hoặc nhiều container.
- Pod đơn lẻ không phù hợp cho production.
- ReplicaSet duy trì số lượng Pod mong muốn.
- Deployment là workload phổ biến nhất cho ứng dụng stateless.
- DaemonSet chạy Pod trên mỗi node hoặc nhóm node.
- StatefulSet dùng cho workload cần identity ổn định.
- Job dùng cho tác vụ chạy một lần đến khi hoàn tất.
- CronJob dùng cho Job chạy theo lịch.
- Label và selector là nền tảng để controller quản lý Pod.
- Không sửa trực tiếp Pod do controller tạo.
- Chọn đúng workload là bước đầu tiên để thiết kế ứng dụng Kubernetes đúng.

## 21. Gợi ý buổi tiếp theo

Buổi tiếp theo nên học về:

```text
Service và cơ chế truy cập ứng dụng trong Kubernetes
```

Lý do:

Ở buổi này,  đã chạy được Pod, Deployment, StatefulSet và các workload khác. Nhưng ứng dụng mới chỉ được test tạm bằng `kubectl port-forward`.

Để ứng dụng có endpoint ổn định trong cluster hoặc từ bên ngoài cluster, cần học Service.

Các nội dung phù hợp cho buổi sau:

- Service là gì?
- ClusterIP
- NodePort
- LoadBalancer trong môi trường on-premise
- Headless Service
- Endpoint và EndpointSlice
- DNS nội bộ trong Kubernetes
- Truy cập ứng dụng giữa các Pod
- Truy cập ứng dụng từ bên ngoài cluster

## 22. Nguồn tham khảo chính thức

- Kubernetes v1.35 Documentation - Workloads: https://v1-35.docs.kubernetes.io/docs/concepts/workloads/
- Kubernetes v1.35 Documentation - Pods: https://v1-35.docs.kubernetes.io/docs/concepts/workloads/pods/
- Kubernetes v1.35 Documentation - ReplicaSet: https://v1-35.docs.kubernetes.io/docs/concepts/workloads/controllers/replicaset/
- Kubernetes v1.35 Documentation - Deployment: https://v1-35.docs.kubernetes.io/docs/concepts/workloads/controllers/deployment/
- Kubernetes v1.35 Documentation - DaemonSet: https://v1-35.docs.kubernetes.io/docs/concepts/workloads/controllers/daemonset/
- Kubernetes v1.35 Documentation - StatefulSet: https://v1-35.docs.kubernetes.io/docs/concepts/workloads/controllers/statefulset/
- Kubernetes v1.35 Documentation - Job: https://v1-35.docs.kubernetes.io/docs/concepts/workloads/controllers/job/
- Kubernetes v1.35 Documentation - CronJob: https://v1-35.docs.kubernetes.io/docs/concepts/workloads/controllers/cron-jobs/
