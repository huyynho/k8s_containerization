# Buổi 04 - Kubernetes Networking và Service: ClusterIP, NodePort, Headless và ExternalName

## 1. Mục tiêu buổi học

Sau buổi học này,  cần hiểu Kubernetes networking hoạt động như thế nào ở mức nền tảng và biết cách dùng Service để giúp các Pod giao tiếp với nhau một cách ổn định.

Trong buổi 01,  đã biết kiến trúc Kubernetes gồm control plane, worker node, kubelet, kube-apiserver, kube-scheduler, controller-manager, CoreDNS, CNI và kube-proxy ở mức tổng quan.

Trong buổi 02,  đã triển khai được một cụm Kubernetes bằng `kubeadm`, sử dụng `containerd` làm container runtime và Calico làm CNI.

Trong buổi 03,  đã học các workload chính như Pod, ReplicaSet, Deployment, DaemonSet, StatefulSet, Job và CronJob.

Buổi 04 trả lời câu hỏi rất quan trọng:

> Khi các Pod có thể được tạo mới, xóa đi, scale lên, scale xuống và thay đổi IP liên tục, làm sao ứng dụng này tìm được ứng dụng kia trong Kubernetes?

Mục tiêu cụ thể:

- Hiểu Kubernetes networking model ở mức nền tảng.
- Hiểu sự khác nhau giữa Node network, Pod network và Service network.
- Hiểu vai trò của CNI, Calico, kube-proxy, CoreDNS và EndpointSlice.
- Hiểu vì sao Pod IP không phải là địa chỉ ổn định để ứng dụng phụ thuộc lâu dài.
- Hiểu Service là gì và vì sao Service tách client khỏi danh sách Pod backend.
- Hiểu selector trong Service dùng để chọn Pod backend như thế nào.
- Phân biệt rõ `containerPort`, `targetPort`, `port` và `nodePort`.
- Biết tạo và kiểm tra Service loại `ClusterIP`.
- Biết tạo và kiểm tra Service loại `NodePort`.
- Biết tạo và kiểm tra Headless Service.
- Biết tạo và kiểm tra Service loại `ExternalName`.
- Biết kiểm tra DNS nội bộ trong Kubernetes.
- Biết kiểm tra EndpointSlice để hiểu Service đang trỏ đến Pod nào.
- Biết xử lý một số lỗi Service phổ biến như selector sai, targetPort sai, DNS không resolve hoặc NodePort không truy cập được.

## 2. Phạm vi kiến thức của buổi này

### 2.1. Được phép sử dụng từ các buổi trước

Trong bài này, chúng ta được phép sử dụng các kiến thức đã học:

- Cluster
- Node
- Control Plane
- Worker Node
- kube-apiserver
- kubelet
- kube-proxy ở mức thành phần hệ thống
- CoreDNS ở mức thành phần hệ thống
- container runtime
- CNI
- Calico
- Pod
- Namespace
- Label
- Selector
- ReplicaSet
- Deployment
- StatefulSet
- kubectl
- Manifest YAML
- kubeadm cluster 1 master node và 2 worker node

### 2.2. Kiến thức mới được mở khóa trong buổi này

Buổi này mở khóa thêm các khái niệm:

- Kubernetes networking model
- Node IP
- Pod IP
- Cluster IP
- Service CIDR
- Pod CIDR
- Service discovery
- DNS name của Service
- EndpointSlice
- Service selector
- Service port
- Target port
- Node port
- Service type `ClusterIP`
- Service type `NodePort`
- Headless Service
- Service type `ExternalName`
- DNS CNAME trong Kubernetes
- Debug Service cơ bản

### 2.3. Các nội dung chưa đi sâu trong buổi này

Để giữ đúng tiến trình khóa học, bài này chưa đi sâu vào:

- Service type `LoadBalancer`
- Cloud Controller Manager
- AWS Load Balancer Controller
- Azure cloud provider integration
- GCP cloud provider integration
- MetalLB
- Ingress
- Ingress Controller
- Gateway API
- NetworkPolicy
- Service Mesh
- eBPF networking nâng cao
- Cilium thay thế kube-proxy
- kube-proxy IPVS tuning nâng cao
- IPv6 và Dual-stack chuyên sâu
- ExternalTrafficPolicy
- InternalTrafficPolicy
- SessionAffinity
- Topology Aware Routing

Lưu ý quan trọng: `LoadBalancer` không được đưa vào nội dung chính của buổi này. Trong môi trường cloud như AWS, Azure, GCP, Service type `LoadBalancer` thường cần sự phối hợp với cloud provider integration hoặc cloud-controller-manager để tạo external load balancer tương ứng. Phần này nên để riêng cho nhóm bài về Kubernetes trên public cloud hoặc triển khai bare-metal load balancer như MetalLB.

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
| Pod CIDR tham khảo | `192.168.0.0/16` |
| Service CIDR kubeadm mặc định thường gặp | `10.96.0.0/12` |
| Namespace lab | `lab-networking` |

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
NAME            STATUS   ROLES           AGE   VERSION   INTERNAL-IP
k8s-master-01   Ready    control-plane    ...   v1.35.x   10.10.10.11
k8s-worker-01   Ready    <none>           ...   v1.35.x   10.10.10.21
k8s-worker-02   Ready    <none>           ...   v1.35.x   10.10.10.22
```

Kiểm tra Pod hệ thống liên quan đến networking:

```bash
kubectl get pods -n kube-system -o wide
```

Các Pod cần chú ý:

```text
calico-node-xxxxx
calico-kube-controllers-xxxxx
coredns-xxxxx
kube-proxy-xxxxx
```

Tạo thư mục làm bài lab:

```bash
mkdir -p ~/k8s-course/buoi-04-networking-service
cd ~/k8s-course/buoi-04-networking-service
```

Tạo namespace dùng riêng cho buổi học:

```bash
cat <<'EOF' > 00-namespace.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: lab-networking
EOF
```

Apply namespace:

```bash
kubectl apply -f 00-namespace.yaml
```

Kiểm tra:

```bash
kubectl get ns lab-networking
```

## 4. Bức tranh tổng quan về networking trong Kubernetes

Networking trong Kubernetes có thể hơi khó lúc mới học vì nó không chỉ là một mạng duy nhất. Khi triển khai một cluster, ta cần phân biệt ít nhất 3 lớp địa chỉ:

- Node IP
- Pod IP
- Service IP

### 4.1. Node IP

Node IP là địa chỉ IP của máy chủ hoặc VM chạy Kubernetes node.

Ví dụ:

```text
k8s-master-01  10.10.10.11
k8s-worker-01  10.10.10.21
k8s-worker-02  10.10.10.22
```

Đây là địa chỉ thuộc mạng hạ tầng thật của lab, ví dụ VLAN management hoặc server network.

Có thể kiểm tra bằng lệnh:

```bash
kubectl get nodes -o wide
```

Node IP dùng cho các mục đích như:

- kubelet giao tiếp với control plane.
- kube-proxy nhận traffic NodePort.
- admin SSH vào node.
- các node giao tiếp với nhau.

### 4.2. Pod IP

Pod IP là địa chỉ IP được cấp cho từng Pod.

Trong cluster dùng Calico với `--pod-network-cidr=192.168.0.0/16`, Pod IP thường nằm trong dải này hoặc dải được cấu hình tương ứng trong Calico.

Ví dụ:

```text
Pod A: 192.168.10.5
Pod B: 192.168.20.9
Pod C: 192.168.30.14
```

Kiểm tra Pod IP:

```bash
kubectl get pods -A -o wide
```

Pod IP có đặc điểm:

- Mỗi Pod có IP riêng.
- Pod IP do CNI cấp phát.
- Khi Pod bị xóa và tạo lại, Pod mới có thể nhận IP khác.
- Không nên hard-code Pod IP trong ứng dụng.
- Pod IP phù hợp để quan sát và debug, nhưng không phải cơ chế service discovery bền vững cho ứng dụng.

### 4.3. Service IP hay ClusterIP

Service IP là địa chỉ IP ảo được Kubernetes cấp cho Service loại `ClusterIP`.

Ví dụ:

```text
backend-service: 10.96.120.15
```

Service IP không phải IP của một Pod cụ thể. Nó là IP ổn định đại diện cho một nhóm Pod backend.

Khi client gọi vào Service IP, Kubernetes thông qua kube-proxy và EndpointSlice sẽ chuyển traffic đến một trong các Pod phù hợp phía sau Service.

### 4.4. Vì sao cần nhiều lớp mạng như vậy?

Nếu chỉ có Pod IP, vấn đề phát sinh rất nhanh.

Ví dụ có Deployment `backend` chạy 3 Pod:

```text
backend-abc   192.168.10.10
backend-def   192.168.20.11
backend-ghi   192.168.30.12
```

Sau đó một Pod bị lỗi và được tạo lại:

```text
backend-abc   deleted
backend-xyz   192.168.40.20
```

Nếu ứng dụng frontend đang hard-code IP `192.168.10.10`, ứng dụng sẽ lỗi.

Service giải quyết vấn đề này bằng cách cung cấp một địa chỉ và một DNS name ổn định:

```text
backend-service.lab-networking.svc.cluster.local
```

Ứng dụng frontend chỉ cần gọi Service. Kubernetes sẽ tự theo dõi danh sách Pod backend phía sau.

## 5. Bốn bài toán networking trong Kubernetes

Kubernetes networking thường có 4 nhóm bài toán chính:

- Container-to-container communication
- Pod-to-Pod communication
- Pod-to-Service communication
- External-to-Service communication

### 5.1. Container-to-container communication

Trong cùng một Pod, các container chia sẻ network namespace.

Điều này có nghĩa là các container trong cùng Pod có thể giao tiếp với nhau qua `localhost`.

Ví dụ:

```text
Pod multi-container
├── container app      listen localhost:8080
└── container sidecar  call localhost:8080
```

Kiến thức này đã được mở khóa ở buổi 03 khi học Pod multi-container.

### 5.2. Pod-to-Pod communication

Mỗi Pod có một IP riêng.

Theo Kubernetes networking model, Pod cần có khả năng giao tiếp với Pod khác mà không cần NAT ở tầng ứng dụng. Trong lab dùng Calico, Calico chịu trách nhiệm cấp IP cho Pod và thiết lập đường đi để Pod ở node này có thể giao tiếp với Pod ở node khác.

Ví dụ:

```text
Pod A trên worker-01: 192.168.10.5
Pod B trên worker-02: 192.168.20.8

Pod A có thể gọi Pod B qua 192.168.20.8 nếu network policy không chặn.
```

Trong buổi này, ta chưa học NetworkPolicy, vì vậy mặc định ta giả định các Pod có thể giao tiếp với nhau nếu CNI hoạt động bình thường.

### 5.3. Pod-to-Service communication

Đây là trọng tâm của buổi học.

Thay vì Pod A gọi trực tiếp Pod B qua Pod IP, Pod A gọi Service.

Ví dụ:

```text
frontend Pod ---> backend Service ---> backend Pod 1
                              └------> backend Pod 2
                              └------> backend Pod 3
```

Service giúp client không cần biết có bao nhiêu Pod backend, tên Pod là gì, IP Pod là gì, Pod nào vừa bị thay thế.

### 5.4. External-to-Service communication

Đây là bài toán đưa traffic từ bên ngoài cluster vào ứng dụng trong cluster.

Trong buổi này, ta chỉ học cách cơ bản nhất là `NodePort`.

Ví dụ:

```text
Client bên ngoài ---> NodeIP:30080 ---> Service NodePort ---> backend Pod
```

Chưa học `LoadBalancer`, `Ingress`, `Gateway API` ở buổi này.

## 6. Các thành phần liên quan đến networking và Service

### 6.1. CNI và Calico

CNI là chuẩn plugin network cho container runtime và Kubernetes node.

Trong lab này, Calico là CNI plugin.

Calico chịu trách nhiệm chính cho:

- Cấp IP cho Pod.
- Tạo network interface cho Pod.
- Đảm bảo Pod trên các node khác nhau có thể giao tiếp.
- Chuẩn bị nền tảng để sau này học NetworkPolicy.

Kiểm tra Pod Calico:

```bash
kubectl get pods -n kube-system | grep calico
```

Ví dụ kết quả:

```text
calico-kube-controllers-xxxxx   1/1   Running
calico-node-xxxxx               1/1   Running
calico-node-yyyyy               1/1   Running
calico-node-zzzzz               1/1   Running
```

Nếu Calico lỗi, Pod có thể không được cấp IP, hoặc Pod ở node này không giao tiếp được với Pod ở node khác.

### 6.2. kube-proxy

`kube-proxy` chạy trên mỗi node.

Vai trò của kube-proxy là phản ánh thông tin Service từ Kubernetes API xuống node, từ đó cấu hình cơ chế chuyển tiếp traffic đến backend Pod.

Trong cluster Linux thông thường, kube-proxy có thể hoạt động với mode như:

- `iptables`
- `ipvs`
- `nftables`

Trong bài này, ta không đi sâu tuning kube-proxy.  chỉ cần nắm:

> kube-proxy là thành phần giúp Service IP và NodePort hoạt động trên node.

Kiểm tra kube-proxy:

```bash
kubectl get pods -n kube-system -l k8s-app=kube-proxy -o wide
```

Ví dụ kết quả:

```text
NAME               READY   STATUS    NODE
kube-proxy-abcd1   1/1     Running   k8s-master-01
kube-proxy-abcd2   1/1     Running   k8s-worker-01
kube-proxy-abcd3   1/1     Running   k8s-worker-02
```

Nếu kube-proxy lỗi, Service ClusterIP hoặc NodePort có thể không hoạt động đúng.

### 6.3. CoreDNS

CoreDNS cung cấp DNS nội bộ cho Kubernetes.

Khi tạo Service tên `backend-service` trong namespace `lab-networking`, Kubernetes có thể tạo DNS name dạng:

```text
backend-service.lab-networking.svc.cluster.local
```

Pod trong cùng namespace có thể gọi ngắn hơn:

```text
backend-service
```

Pod ở namespace khác nên gọi đầy đủ hơn:

```text
backend-service.lab-networking
```

hoặc:

```text
backend-service.lab-networking.svc.cluster.local
```

Kiểm tra CoreDNS:

```bash
kubectl get pods -n kube-system -l k8s-app=kube-dns -o wide
```

Ví dụ kết quả:

```text
NAME                       READY   STATUS    NODE
coredns-xxxxxxxxxx-aaaaa   1/1     Running   k8s-worker-01
coredns-xxxxxxxxxx-bbbbb   1/1     Running   k8s-worker-02
```

Nếu CoreDNS lỗi, ứng dụng có thể vẫn gọi được Service qua ClusterIP, nhưng gọi bằng tên DNS sẽ lỗi.

### 6.4. EndpointSlice

EndpointSlice là object lưu danh sách endpoint backend cho Service.

Khi Service có selector, Kubernetes control plane sẽ tìm các Pod match selector đó và tạo EndpointSlice tương ứng.

Ví dụ:

```text
Service backend-service
selector: app=backend

Pod backend-1: 192.168.10.5
Pod backend-2: 192.168.20.6
Pod backend-3: 192.168.30.7

EndpointSlice backend-service-xxxxx
- 192.168.10.5:80
- 192.168.20.6:80
- 192.168.30.7:80
```

Kiểm tra EndpointSlice:

```bash
kubectl get endpointslice -n lab-networking
```

hoặc:

```bash
kubectl describe endpointslice -n lab-networking
```

EndpointSlice là một trong những chìa khóa để debug Service.

Nếu Service không có endpoint, thường là do:

- selector của Service không match label của Pod;
- Pod chưa Ready;
- Pod không expose đúng port mong muốn;
- workload backend chưa chạy;
- namespace bị nhầm.

## 7. Service trong Kubernetes là gì?

Service là Kubernetes object dùng để expose một nhóm Pod qua một endpoint ổn định.

Về bản chất, Service giải quyết 3 vấn đề:

- Pod backend có thể thay đổi liên tục.
- Pod IP không ổn định.
- Client không nên tự theo dõi danh sách Pod backend.

Service giúp client chỉ cần biết:

```text
Tên Service
Port của Service
Namespace của Service
```

Thay vì phải biết:

```text
Tên từng Pod
IP từng Pod
Pod nào còn sống
Pod nào đã bị thay thế
```

### 7.1. Service chọn Pod bằng selector

Service thường dùng selector để chọn Pod backend.

Ví dụ:

```yaml
selector:
  app: backend
```

Nghĩa là Service sẽ chọn các Pod có label:

```yaml
labels:
  app: backend
```

Nếu selector không match label, Service vẫn được tạo, nhưng không có endpoint backend.

Đây là lỗi rất phổ biến khi học Kubernetes.

### 7.2. Service không tạo Pod

Cần phân biệt rõ:

- Deployment tạo và quản lý Pod.
- Service không tạo Pod.
- Service chỉ tìm Pod phù hợp thông qua selector.

Ví dụ:

```text
Deployment backend
└── tạo Pod có label app=backend

Service backend-service
└── selector app=backend
    └── trỏ đến các Pod do Deployment tạo ra
```

### 7.3. Service không thay thế CNI

Service không cấp network cho Pod.

Pod network vẫn do CNI như Calico đảm nhiệm.

Service chỉ cung cấp abstraction để client truy cập nhóm Pod backend.

## 8. Bốn khái niệm port cần phân biệt

Đây là phần cực kỳ quan trọng.

Khi làm Service,  rất dễ nhầm các trường sau:

- `containerPort`
- `targetPort`
- `port`
- `nodePort`

### 8.1. containerPort

`containerPort` nằm trong manifest Pod hoặc Pod template của Deployment.

Ví dụ:

```yaml
containers:
  - name: nginx
    image: nginx:1.27
    ports:
      - containerPort: 80
```

Ý nghĩa:

- Cho biết container dự kiến listen port nào.
- Mang tính khai báo/documentation trong Kubernetes.
- Giúp các object khác có thể tham chiếu named port nếu có đặt tên.
- Không tự động expose ứng dụng ra ngoài cluster.

### 8.2. targetPort

`targetPort` nằm trong Service.

Ví dụ:

```yaml
targetPort: 80
```

Ý nghĩa:

- Là port trên Pod backend mà Service sẽ forward traffic đến.
- Thường trùng với `containerPort`.
- Có thể là số, ví dụ `80`.
- Có thể là tên port, ví dụ `http`.

### 8.3. port

`port` nằm trong Service.

Ví dụ:

```yaml
port: 8080
```

Ý nghĩa:

- Là port mà client trong cluster dùng để gọi Service.
- Client gọi `service-name:port`.
- Service nhận traffic ở `port`, sau đó forward đến `targetPort` trên Pod backend.

Ví dụ:

```text
client Pod gọi backend-service:8080
Service forward đến backend Pod:80
```

### 8.4. nodePort

`nodePort` chỉ dùng với Service type `NodePort`.

Ví dụ:

```yaml
nodePort: 30080
```

Ý nghĩa:

- Là port mở trên mỗi Node IP.
- Client bên ngoài cluster có thể gọi `NodeIP:nodePort`.
- Dải NodePort mặc định thường là `30000-32767`.

Ví dụ:

```text
Client ngoài cluster gọi http://10.10.10.21:30080
kube-proxy trên node nhận traffic
traffic được chuyển đến một backend Pod phù hợp
```

### 8.5. Bảng tóm tắt port

| Trường | Nằm ở đâu | Ai dùng | Ý nghĩa |
|---|---|---|---|
| `containerPort` | Pod / Pod template | Container / người đọc manifest | Port ứng dụng listen trong container |
| `targetPort` | Service | Service | Port trên Pod backend nhận traffic |
| `port` | Service | Client trong cluster | Port của Service |
| `nodePort` | Service NodePort | Client ngoài cluster | Port mở trên Node IP |

Ví dụ tổng hợp:

```text
Client trong cluster gọi: backend-service:8080
                             │
                             ▼
Service port: 8080
Service targetPort: 80
                             │
                             ▼
Backend Pod containerPort: 80
```

Với NodePort:

```text
Client ngoài cluster gọi: NodeIP:30080
                             │
                             ▼
Service nodePort: 30080
Service port: 8080
Service targetPort: 80
                             │
                             ▼
Backend Pod containerPort: 80
```

## 9. Chuẩn bị ứng dụng backend dùng chung cho lab

Trước khi tạo Service, ta cần có workload backend.

Trong bài này, dùng Deployment chạy NGINX.

Lưu file `01-backend-deployment.yaml`:

```bash
cat <<'EOF' > 01-backend-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend-web
  namespace: lab-networking
  labels:
    app: backend-web
spec:
  replicas: 3
  selector:
    matchLabels:
      app: backend-web
  template:
    metadata:
      labels:
        app: backend-web
    spec:
      containers:
        - name: nginx
          image: nginx:1.27
          ports:
            - name: http
              containerPort: 80
EOF
```

Apply:

```bash
kubectl apply -f 01-backend-deployment.yaml
```

Kiểm tra:

```bash
kubectl get deployment -n lab-networking
kubectl get pods -n lab-networking -o wide
```

Kết quả kỳ vọng:

```text
NAME                          READY   STATUS    IP              NODE
backend-web-xxxxxxxxx-aaaaa    1/1     Running   192.168.x.x     k8s-worker-01
backend-web-xxxxxxxxx-bbbbb    1/1     Running   192.168.x.x     k8s-worker-02
backend-web-xxxxxxxxx-ccccc    1/1     Running   192.168.x.x     k8s-worker-01
```

### 9.1. Giải thích manifest Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
```

Khai báo object là Deployment. Kiến thức này đã học ở buổi 03.

```yaml
metadata:
  name: backend-web
  namespace: lab-networking
```

Deployment tên `backend-web`, nằm trong namespace `lab-networking`.

```yaml
labels:
  app: backend-web
```

Label ở metadata của Deployment. Label này chủ yếu giúp quản lý object Deployment.

```yaml
replicas: 3
```

Yêu cầu Deployment duy trì 3 Pod backend.

```yaml
selector:
  matchLabels:
    app: backend-web
```

Deployment quản lý các Pod có label `app=backend-web`.

```yaml
template:
  metadata:
    labels:
      app: backend-web
```

Pod được tạo ra sẽ có label `app=backend-web`.

Đây là label rất quan trọng vì Service sẽ dùng label này để tìm Pod backend.

```yaml
containers:
  - name: nginx
    image: nginx:1.27
```

Mỗi Pod chạy một container NGINX.

```yaml
ports:
  - name: http
    containerPort: 80
```

Container listen port 80, đồng thời đặt tên port là `http`.

Tên `http` này có thể được Service tham chiếu trong `targetPort`.

## 10. Service type ClusterIP

`ClusterIP` là Service type mặc định.

Nếu không khai báo `spec.type`, Kubernetes sẽ hiểu là `ClusterIP`.

ClusterIP dùng để expose ứng dụng bên trong cluster.

Ví dụ:

```text
frontend Pod ---> backend-web-service:80 ---> backend-web Pod
```

### 10.1. Khi nào dùng ClusterIP?

Dùng ClusterIP khi:

- Ứng dụng chỉ cần được truy cập từ trong cluster.
- Backend API chỉ phục vụ frontend trong cluster.
- Database hoặc cache chỉ phục vụ ứng dụng nội bộ.
- Muốn có DNS name ổn định cho một nhóm Pod.
- Muốn tách client khỏi Pod IP backend.

### 10.2. Manifest ClusterIP Service

Lưu file `02-backend-clusterip-service.yaml`:

```bash
cat <<'EOF' > 02-backend-clusterip-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: backend-web-clusterip
  namespace: lab-networking
  labels:
    app: backend-web
spec:
  type: ClusterIP
  selector:
    app: backend-web
  ports:
    - name: http
      protocol: TCP
      port: 80
      targetPort: http
EOF
```

Apply:

```bash
kubectl apply -f 02-backend-clusterip-service.yaml
```

Kiểm tra Service:

```bash
kubectl get svc -n lab-networking
```

Kết quả kỳ vọng:

```text
NAME                    TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
backend-web-clusterip   ClusterIP   10.96.x.x       <none>        80/TCP    ...
```

### 10.3. Giải thích manifest ClusterIP Service

```yaml
apiVersion: v1
kind: Service
```

Khai báo object là Service.

```yaml
metadata:
  name: backend-web-clusterip
  namespace: lab-networking
```

Service tên `backend-web-clusterip`, nằm trong namespace `lab-networking`.

```yaml
labels:
  app: backend-web
```

Label gắn lên chính Service. Label này dùng để quản lý Service, không phải selector chọn Pod.

```yaml
spec:
  type: ClusterIP
```

Khai báo Service type là ClusterIP.

Nếu bỏ dòng này, mặc định Kubernetes cũng dùng ClusterIP.

```yaml
selector:
  app: backend-web
```

Đây là phần cực kỳ quan trọng.

Service sẽ tìm các Pod có label:

```yaml
app: backend-web
```

Các Pod này được Deployment `backend-web` tạo ra ở phần trước.

```yaml
ports:
  - name: http
    protocol: TCP
    port: 80
    targetPort: http
```

Giải thích:

- `name: http`: tên của port trong Service.
- `protocol: TCP`: giao thức TCP.
- `port: 80`: client trong cluster gọi Service qua port 80.
- `targetPort: http`: Service forward traffic đến port tên `http` trong Pod backend.

Trong Deployment, container đã khai báo:

```yaml
ports:
  - name: http
    containerPort: 80
```

Vì vậy `targetPort: http` sẽ trỏ đến `containerPort: 80`.

### 10.4. Kiểm tra EndpointSlice của ClusterIP Service

Kiểm tra EndpointSlice:

```bash
kubectl get endpointslice -n lab-networking
```

Kết quả ví dụ:

```text
NAME                          ADDRESSTYPE   PORTS   ENDPOINTS
backend-web-clusterip-abcde    IPv4          80      192.168.x.x,192.168.y.y,192.168.z.z
```

Describe EndpointSlice:

```bash
kubectl describe endpointslice -n lab-networking -l kubernetes.io/service-name=backend-web-clusterip
```

Điểm cần quan sát:

```text
Ports:
  Name: http
  Port: 80
Endpoints:
  Addresses: 192.168.x.x
  Conditions:
    Ready: true
```

Nếu EndpointSlice có danh sách Pod IP, Service đã tìm được backend.

Nếu không có endpoint, cần kiểm tra selector và label.

### 10.5. Kiểm tra DNS của ClusterIP Service

Tạo Pod client dùng để test:

```bash
kubectl run test-client \
  -n lab-networking \
  --image=curlimages/curl:8.10.1 \
  --restart=Never \
  --command -- sleep 3600
```

Kiểm tra Pod client:

```bash
kubectl get pod test-client -n lab-networking
```

Exec vào Pod client:

```bash
kubectl exec -it test-client -n lab-networking -- sh
```

Từ trong Pod, gọi Service bằng tên ngắn:

```sh
curl -I http://backend-web-clusterip
```

Kết quả kỳ vọng:

```text
HTTP/1.1 200 OK
Server: nginx/1.27.x
```

Gọi bằng FQDN đầy đủ:

```sh
curl -I http://backend-web-clusterip.lab-networking.svc.cluster.local
```

Kết quả kỳ vọng:

```text
HTTP/1.1 200 OK
```

Thoát khỏi Pod:

```sh
exit
```

### 10.6. Kiểm tra Service IP ổn định khi Pod thay đổi

Xem Service IP:

```bash
kubectl get svc backend-web-clusterip -n lab-networking
```

Ví dụ:

```text
NAME                    TYPE        CLUSTER-IP     PORT(S)
backend-web-clusterip   ClusterIP   10.96.10.50    80/TCP
```

Xem Pod hiện tại:

```bash
kubectl get pods -n lab-networking -o wide -l app=backend-web
```

Xóa một Pod backend cụ thể:

```bash
kubectl delete pod -n lab-networking <ten-pod-backend>
```

Kiểm tra lại Pod:

```bash
kubectl get pods -n lab-networking -o wide -l app=backend-web
```

Pod mới có thể có IP khác.

Kiểm tra lại Service:

```bash
kubectl get svc backend-web-clusterip -n lab-networking
```

ClusterIP vẫn giữ nguyên.

Ý nghĩa:

> Pod có thể thay đổi, nhưng Service vẫn là điểm truy cập ổn định cho client.

## 11. Service type NodePort

`NodePort` dùng để expose Service ra ngoài cluster thông qua port mở trên mỗi node.

Mô hình:

```text
Client ngoài cluster
      │
      │ http://NodeIP:30080
      ▼
NodePort Service
      │
      ▼
Backend Pod
```

NodePort vẫn tạo ClusterIP phía sau. Có thể hiểu NodePort là mở rộng từ ClusterIP.

### 11.1. Khi nào dùng NodePort?

Dùng NodePort khi:

- Lab cần truy cập ứng dụng từ bên ngoài cluster.
- Môi trường on-premise chưa có LoadBalancer.
- Muốn kiểm tra nhanh ứng dụng.
- Muốn đặt một reverse proxy hoặc external load balancer thủ công phía trước các node.

Không nên lạm dụng NodePort cho production lớn nếu không có lớp quản lý truy cập phù hợp.

Trong production, thường dùng:

- LoadBalancer trên cloud.
- MetalLB trên bare-metal.
- Ingress Controller.
- Gateway API.
- Reverse proxy hoặc firewall ngoài cluster.

Các nội dung trên sẽ học ở buổi sau.

### 11.2. Manifest NodePort Service

Lưu file `03-backend-nodeport-service.yaml`:

```bash
cat <<'EOF' > 03-backend-nodeport-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: backend-web-nodeport
  namespace: lab-networking
  labels:
    app: backend-web
spec:
  type: NodePort
  selector:
    app: backend-web
  ports:
    - name: http
      protocol: TCP
      port: 80
      targetPort: http
      nodePort: 30080
EOF
```

Apply:

```bash
kubectl apply -f 03-backend-nodeport-service.yaml
```

Kiểm tra:

```bash
kubectl get svc backend-web-nodeport -n lab-networking
```

Kết quả kỳ vọng:

```text
NAME                   TYPE       CLUSTER-IP     EXTERNAL-IP   PORT(S)
backend-web-nodeport   NodePort   10.96.x.x      <none>        80:30080/TCP
```

### 11.3. Giải thích manifest NodePort Service

```yaml
apiVersion: v1
kind: Service
```

Khai báo object là Service.

```yaml
metadata:
  name: backend-web-nodeport
  namespace: lab-networking
```

Service tên `backend-web-nodeport`, nằm trong namespace `lab-networking`.

```yaml
spec:
  type: NodePort
```

Khai báo Service type là NodePort.

```yaml
selector:
  app: backend-web
```

Service chọn các Pod backend có label `app=backend-web`.

```yaml
ports:
  - name: http
    protocol: TCP
    port: 80
    targetPort: http
    nodePort: 30080
```

Giải thích:

- `port: 80`: Service port bên trong cluster.
- `targetPort: http`: forward đến named port `http` của Pod backend.
- `nodePort: 30080`: mở port 30080 trên mỗi node.

Nếu không khai báo `nodePort`, Kubernetes sẽ tự cấp một port trong dải NodePort.

Dải mặc định thường là:

```text
30000-32767
```

### 11.4. Kiểm tra truy cập NodePort từ ngoài cluster

Lấy danh sách node IP:

```bash
kubectl get nodes -o wide
```

Ví dụ:

```text
k8s-master-01   10.10.10.11
k8s-worker-01   10.10.10.21
k8s-worker-02   10.10.10.22
```

Từ máy admin hoặc máy bên ngoài cluster có route đến node IP, test:

```bash
curl -I http://10.10.10.21:30080
```

hoặc:

```bash
curl -I http://10.10.10.22:30080
```

Kết quả kỳ vọng:

```text
HTTP/1.1 200 OK
Server: nginx/1.27.x
```

Điểm cần hiểu:

- Có thể gọi vào bất kỳ node nào trong cluster qua `NodeIP:nodePort`.
- Node đó không nhất thiết phải là node đang chạy Pod backend.
- kube-proxy sẽ xử lý chuyển tiếp traffic đến endpoint phù hợp.

### 11.5. Các lỗi thường gặp với NodePort

#### Lỗi 1: Không truy cập được từ máy ngoài

Kiểm tra:

```bash
kubectl get svc backend-web-nodeport -n lab-networking
kubectl get endpointslice -n lab-networking -l kubernetes.io/service-name=backend-web-nodeport
```

Nếu Service và EndpointSlice đúng, kiểm tra tiếp:

- Máy ngoài có ping được Node IP không?
- Firewall trên Ubuntu có chặn port 30080 không?
- Security group hoặc firewall vật lý có chặn không?
- Có route từ máy ngoài đến node network không?
- kube-proxy có chạy trên node không?

Kiểm tra kube-proxy:

```bash
kubectl get pods -n kube-system -l k8s-app=kube-proxy -o wide
```

#### Lỗi 2: NodePort khai báo ngoài dải hợp lệ

Ví dụ sai:

```yaml
nodePort: 8080
```

Nếu cluster dùng dải mặc định `30000-32767`, port 8080 không hợp lệ.

Kubernetes API sẽ từ chối manifest.

#### Lỗi 3: Trùng NodePort

Nếu Service khác đã dùng `30080`, tạo Service mới cũng dùng `30080` sẽ lỗi.

Cần chọn port khác trong dải NodePort.

## 12. Headless Service

Headless Service là Service không có ClusterIP.

Khai báo bằng:

```yaml
clusterIP: None
```

Headless Service không làm load balancing qua một Service IP duy nhất.

Thay vào đó, DNS của Headless Service trả về IP của các Pod backend.

### 12.1. Khi nào dùng Headless Service?

Dùng Headless Service khi:

- Cần client nhìn thấy từng Pod backend riêng lẻ.
- Cần stable network identity cho StatefulSet.
- Cần service discovery ở mức từng Pod.
- Ứng dụng stateful tự quản lý clustering, replication hoặc leader election.

Ví dụ thường gặp:

- Database cluster.
- Kafka.
- ZooKeeper.
- Cassandra.
- Redis cluster.
- Stateful application cần định danh ổn định.

### 12.2. So sánh ClusterIP và Headless

| Tiêu chí | ClusterIP Service | Headless Service |
|---|---|---|
| Có ClusterIP | Có | Không |
| kube-proxy xử lý load balancing | Có | Không |
| DNS trả về | Service IP | Danh sách Pod IP |
| Phù hợp với | Stateless backend | Stateful workload / service discovery từng Pod |
| Khai báo | `type: ClusterIP` hoặc mặc định | `clusterIP: None` |

### 12.3. Manifest Headless Service với StatefulSet

Trong buổi 03,  đã học StatefulSet. Buổi này dùng lại StatefulSet để thấy rõ vai trò Headless Service.

Lưu file `04-headless-statefulset.yaml`:

```bash
cat <<'EOF' > 04-headless-statefulset.yaml
apiVersion: v1
kind: Service
metadata:
  name: stateful-web-headless
  namespace: lab-networking
  labels:
    app: stateful-web
spec:
  clusterIP: None
  selector:
    app: stateful-web
  ports:
    - name: http
      protocol: TCP
      port: 80
      targetPort: http
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: stateful-web
  namespace: lab-networking
spec:
  serviceName: stateful-web-headless
  replicas: 2
  selector:
    matchLabels:
      app: stateful-web
  template:
    metadata:
      labels:
        app: stateful-web
    spec:
      containers:
        - name: nginx
          image: nginx:1.27
          ports:
            - name: http
              containerPort: 80
EOF
```

Apply:

```bash
kubectl apply -f 04-headless-statefulset.yaml
```

Kiểm tra:

```bash
kubectl get svc stateful-web-headless -n lab-networking
kubectl get statefulset stateful-web -n lab-networking
kubectl get pods -n lab-networking -l app=stateful-web -o wide
```

Kết quả kỳ vọng:

```text
NAME                    TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)
stateful-web-headless   ClusterIP   None         <none>        80/TCP
```

Pod StatefulSet:

```text
NAME             READY   STATUS    IP              NODE
stateful-web-0   1/1     Running   192.168.x.x     k8s-worker-01
stateful-web-1   1/1     Running   192.168.y.y     k8s-worker-02
```

### 12.4. Giải thích manifest Headless Service

```yaml
apiVersion: v1
kind: Service
metadata:
  name: stateful-web-headless
  namespace: lab-networking
```

Tạo Service tên `stateful-web-headless`.

```yaml
spec:
  clusterIP: None
```

Đây là dòng quan trọng nhất.

`clusterIP: None` biến Service thành Headless Service.

Kubernetes sẽ không cấp ClusterIP cho Service này.

```yaml
selector:
  app: stateful-web
```

Headless Service chọn các Pod có label `app=stateful-web`.

```yaml
ports:
  - name: http
    protocol: TCP
    port: 80
    targetPort: http
```

Khai báo port cho Service và DNS SRV record nếu cần.

### 12.5. Giải thích phần StatefulSet liên quan đến networking

```yaml
kind: StatefulSet
metadata:
  name: stateful-web
```

Tạo StatefulSet tên `stateful-web`.

```yaml
spec:
  serviceName: stateful-web-headless
```

Đây là dòng rất quan trọng.

StatefulSet dùng Headless Service này để cung cấp network identity ổn định cho từng Pod.

```yaml
replicas: 2
```

Tạo 2 Pod:

```text
stateful-web-0
stateful-web-1
```

```yaml
selector:
  matchLabels:
    app: stateful-web
```

StatefulSet quản lý Pod có label `app=stateful-web`.

```yaml
template:
  metadata:
    labels:
      app: stateful-web
```

Pod do StatefulSet tạo ra sẽ có label `app=stateful-web`, nhờ đó Headless Service chọn được các Pod này.

### 12.6. Kiểm tra DNS của Headless Service

Dùng Pod `test-client` đã tạo trước đó.

Exec vào Pod:

```bash
kubectl exec -it test-client -n lab-networking -- sh
```

Kiểm tra DNS của Headless Service:

```sh
nslookup stateful-web-headless
```

Kết quả kỳ vọng có thể trả về nhiều IP Pod:

```text
Name:      stateful-web-headless.lab-networking.svc.cluster.local
Address 1: 192.168.x.x stateful-web-0.stateful-web-headless.lab-networking.svc.cluster.local
Address 2: 192.168.y.y stateful-web-1.stateful-web-headless.lab-networking.svc.cluster.local
```

Kiểm tra DNS riêng từng Pod:

```sh
nslookup stateful-web-0.stateful-web-headless.lab-networking.svc.cluster.local
nslookup stateful-web-1.stateful-web-headless.lab-networking.svc.cluster.local
```

Gọi từng Pod thông qua DNS ổn định:

```sh
curl -I http://stateful-web-0.stateful-web-headless.lab-networking.svc.cluster.local
curl -I http://stateful-web-1.stateful-web-headless.lab-networking.svc.cluster.local
```

Kết quả kỳ vọng:

```text
HTTP/1.1 200 OK
```

Thoát khỏi Pod:

```sh
exit
```

### 12.7. Điểm cần nhớ về Headless Service

Headless Service không phải là cách expose ứng dụng ra ngoài cluster.

Headless Service chủ yếu phục vụ service discovery bên trong cluster.

Nó đặc biệt hữu ích với StatefulSet vì từng Pod cần identity riêng:

```text
stateful-web-0.stateful-web-headless.lab-networking.svc.cluster.local
stateful-web-1.stateful-web-headless.lab-networking.svc.cluster.local
```

Khác với Deployment, Pod của StatefulSet có tên ổn định.

## 13. Service type ExternalName

`ExternalName` là Service đặc biệt.

Nó không dùng selector.

Nó không tạo ClusterIP.

Nó không proxy traffic.

Nó chỉ tạo DNS CNAME để ánh xạ một Service name trong Kubernetes đến một DNS name bên ngoài.

Ví dụ:

```text
external-kubernetes.lab-networking.svc.cluster.local
CNAME kubernetes.io
```

### 13.1. Khi nào dùng ExternalName?

Dùng ExternalName khi:

- Ứng dụng trong cluster cần gọi một service bên ngoài bằng tên nội bộ.
- Muốn che giấu tên DNS thật phía sau một tên Service nội bộ.
- Muốn chuẩn hóa endpoint để sau này có thể chuyển service từ ngoài cluster vào trong cluster.

Ví dụ:

```text
Ứng dụng gọi: payment-api.lab-networking.svc.cluster.local
Thực tế DNS CNAME đến: payment-api.company.com
```

Sau này nếu payment API chạy trong cluster, ta có thể đổi ExternalName thành ClusterIP Service mà ứng dụng không cần đổi cấu hình nhiều.

### 13.2. Manifest ExternalName Service

Lưu file `05-externalname-service.yaml`:

```bash
cat <<'EOF' > 05-externalname-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: external-kubernetes
  namespace: lab-networking
spec:
  type: ExternalName
  externalName: kubernetes.io
EOF
```

Apply:

```bash
kubectl apply -f 05-externalname-service.yaml
```

Kiểm tra:

```bash
kubectl get svc external-kubernetes -n lab-networking
```

Kết quả kỳ vọng:

```text
NAME                  TYPE           CLUSTER-IP   EXTERNAL-IP     PORT(S)
external-kubernetes   ExternalName   <none>       kubernetes.io   <none>
```

### 13.3. Giải thích manifest ExternalName

```yaml
apiVersion: v1
kind: Service
```

Khai báo object là Service.

```yaml
metadata:
  name: external-kubernetes
  namespace: lab-networking
```

Tạo Service tên `external-kubernetes` trong namespace `lab-networking`.

```yaml
spec:
  type: ExternalName
```

Khai báo Service type là ExternalName.

```yaml
externalName: kubernetes.io
```

DNS của Kubernetes sẽ ánh xạ Service name này đến DNS name `kubernetes.io`.

Không có selector vì Service này không trỏ đến Pod backend trong cluster.

Không có ClusterIP vì không có proxying.

### 13.4. Kiểm tra DNS của ExternalName

Exec vào Pod client:

```bash
kubectl exec -it test-client -n lab-networking -- sh
```

Kiểm tra DNS:

```sh
nslookup external-kubernetes
```

Kết quả kỳ vọng:

```text
external-kubernetes.lab-networking.svc.cluster.local canonical name = kubernetes.io
```

Có thể thử curl nếu cluster có outbound Internet:

```sh
curl -I https://external-kubernetes
```

Tuy nhiên cần lưu ý: ExternalName có thể gây vấn đề với HTTP Host header hoặc TLS certificate vì hostname client dùng là `external-kubernetes`, còn certificate bên ngoài thường cấp cho `kubernetes.io`.

Vì vậy trong lab, mục tiêu chính là hiểu DNS CNAME, không nhất thiết phải dùng ExternalName cho HTTPS production nếu chưa hiểu kỹ hostname và TLS.

Thoát khỏi Pod:

```sh
exit
```

## 14. DNS trong Kubernetes Service

Kubernetes tạo DNS record cho Service.

Với Service `backend-web-clusterip` trong namespace `lab-networking`, các cách gọi thường gặp:

```text
backend-web-clusterip
backend-web-clusterip.lab-networking
backend-web-clusterip.lab-networking.svc
backend-web-clusterip.lab-networking.svc.cluster.local
```

### 14.1. Gọi Service trong cùng namespace

Nếu Pod client nằm trong cùng namespace `lab-networking`, có thể gọi ngắn:

```bash
curl http://backend-web-clusterip
```

CoreDNS sẽ tự mở rộng theo search domain trong `/etc/resolv.conf` của Pod.

Kiểm tra trong Pod:

```bash
kubectl exec -it test-client -n lab-networking -- cat /etc/resolv.conf
```

Ví dụ:

```text
nameserver 10.96.0.10
search lab-networking.svc.cluster.local svc.cluster.local cluster.local
options ndots:5
```

Ý nghĩa:

- `nameserver`: IP của DNS service trong cluster.
- `search`: danh sách domain được thử khi gọi tên ngắn.
- `ndots:5`: ảnh hưởng cách resolver xử lý tên ngắn và FQDN.

### 14.2. Gọi Service khác namespace

Nếu Pod ở namespace khác, nên gọi:

```text
backend-web-clusterip.lab-networking
```

hoặc đầy đủ:

```text
backend-web-clusterip.lab-networking.svc.cluster.local
```

Ví dụ tạo Pod client ở namespace `default`:

```bash
kubectl run test-client-default \
  --image=curlimages/curl:8.10.1 \
  --restart=Never \
  --command -- sleep 3600
```

Gọi Service bằng tên ngắn sẽ không đúng vì khác namespace:

```bash
kubectl exec -it test-client-default -- curl -I http://backend-web-clusterip
```

Có thể lỗi resolve DNS.

Gọi đúng namespace:

```bash
kubectl exec -it test-client-default -- curl -I http://backend-web-clusterip.lab-networking
```

hoặc:

```bash
kubectl exec -it test-client-default -- curl -I http://backend-web-clusterip.lab-networking.svc.cluster.local
```

Dọn Pod test ở namespace default:

```bash
kubectl delete pod test-client-default
```

## 15. Lab tổng hợp: triển khai và kiểm tra toàn bộ Service type trong phạm vi bài học

### 15.1. Mục tiêu lab

Sau lab này,  sẽ thực hiện được:

- Tạo namespace lab.
- Tạo Deployment backend.
- Expose backend bằng ClusterIP.
- Kiểm tra Service DNS.
- Kiểm tra EndpointSlice.
- Expose backend bằng NodePort.
- Truy cập NodePort từ ngoài cluster.
- Tạo StatefulSet với Headless Service.
- Kiểm tra DNS từng Pod của StatefulSet.
- Tạo ExternalName Service.
- Kiểm tra DNS CNAME.

### 15.2. Bước 1 - Tạo namespace

```bash
kubectl apply -f 00-namespace.yaml
```

Kiểm tra:

```bash
kubectl get ns lab-networking
```

### 15.3. Bước 2 - Tạo Deployment backend

```bash
kubectl apply -f 01-backend-deployment.yaml
```

Kiểm tra:

```bash
kubectl get deployment,pod -n lab-networking -o wide
```

### 15.4. Bước 3 - Tạo ClusterIP Service

```bash
kubectl apply -f 02-backend-clusterip-service.yaml
```

Kiểm tra:

```bash
kubectl get svc backend-web-clusterip -n lab-networking
kubectl get endpointslice -n lab-networking -l kubernetes.io/service-name=backend-web-clusterip
```

Tạo client:

```bash
kubectl run test-client \
  -n lab-networking \
  --image=curlimages/curl:8.10.1 \
  --restart=Never \
  --command -- sleep 3600
```

Test DNS và HTTP:

```bash
kubectl exec -it test-client -n lab-networking -- curl -I http://backend-web-clusterip
```

### 15.5. Bước 4 - Tạo NodePort Service

```bash
kubectl apply -f 03-backend-nodeport-service.yaml
```

Kiểm tra:

```bash
kubectl get svc backend-web-nodeport -n lab-networking
```

Từ máy admin bên ngoài cluster:

```bash
curl -I http://<WORKER_NODE_IP>:30080
```

Ví dụ:

```bash
curl -I http://10.10.10.21:30080
```

### 15.6. Bước 5 - Tạo Headless Service và StatefulSet

```bash
kubectl apply -f 04-headless-statefulset.yaml
```

Kiểm tra:

```bash
kubectl get svc stateful-web-headless -n lab-networking
kubectl get statefulset stateful-web -n lab-networking
kubectl get pods -n lab-networking -l app=stateful-web -o wide
```

Test DNS:

```bash
kubectl exec -it test-client -n lab-networking -- nslookup stateful-web-headless
kubectl exec -it test-client -n lab-networking -- nslookup stateful-web-0.stateful-web-headless.lab-networking.svc.cluster.local
kubectl exec -it test-client -n lab-networking -- nslookup stateful-web-1.stateful-web-headless.lab-networking.svc.cluster.local
```

Test HTTP đến từng Pod:

```bash
kubectl exec -it test-client -n lab-networking -- curl -I http://stateful-web-0.stateful-web-headless.lab-networking.svc.cluster.local
kubectl exec -it test-client -n lab-networking -- curl -I http://stateful-web-1.stateful-web-headless.lab-networking.svc.cluster.local
```

### 15.7. Bước 6 - Tạo ExternalName Service

```bash
kubectl apply -f 05-externalname-service.yaml
```

Kiểm tra:

```bash
kubectl get svc external-kubernetes -n lab-networking
```

Test DNS:

```bash
kubectl exec -it test-client -n lab-networking -- nslookup external-kubernetes
```

Kết quả kỳ vọng là CNAME đến `kubernetes.io`.

### 15.8. Bước 7 - Quan sát toàn bộ object trong namespace

```bash
kubectl get all -n lab-networking
```

Lưu ý: `kubectl get all` không hiển thị tất cả mọi loại object. Ví dụ EndpointSlice không nằm đầy đủ trong `get all`.

Kiểm tra thêm:

```bash
kubectl get svc -n lab-networking
kubectl get endpointslice -n lab-networking
```

## 16. Bài lab lỗi cố ý: Service không có endpoint

Phần này giúp  học cách debug.

### 16.1. Tạo Service selector sai

Lưu file `06-broken-service.yaml`:

```bash
cat <<'EOF' > 06-broken-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: broken-service
  namespace: lab-networking
spec:
  type: ClusterIP
  selector:
    app: wrong-label
  ports:
    - name: http
      protocol: TCP
      port: 80
      targetPort: 80
EOF
```

Apply:

```bash
kubectl apply -f 06-broken-service.yaml
```

Kiểm tra Service:

```bash
kubectl get svc broken-service -n lab-networking
```

Kiểm tra EndpointSlice:

```bash
kubectl get endpointslice -n lab-networking -l kubernetes.io/service-name=broken-service
```

Có thể không thấy endpoint nào, hoặc EndpointSlice không có địa chỉ backend.

Test:

```bash
kubectl exec -it test-client -n lab-networking -- curl -I --connect-timeout 5 http://broken-service
```

Có thể bị timeout hoặc không kết nối được.

### 16.2. Debug lỗi selector

Xem label của Pod backend:

```bash
kubectl get pods -n lab-networking --show-labels
```

Ta thấy Pod backend có label:

```text
app=backend-web
```

Nhưng Service lại selector:

```yaml
selector:
  app: wrong-label
```

Vì vậy Service không chọn được Pod.

### 16.3. Sửa Service

Sửa selector:

```bash
kubectl patch svc broken-service -n lab-networking -p '{"spec":{"selector":{"app":"backend-web"}}}'
```

Kiểm tra lại EndpointSlice:

```bash
kubectl get endpointslice -n lab-networking -l kubernetes.io/service-name=broken-service
```

Test lại:

```bash
kubectl exec -it test-client -n lab-networking -- curl -I http://broken-service
```

Kết quả kỳ vọng:

```text
HTTP/1.1 200 OK
```

Ý nghĩa:

> Khi Service không hoạt động, đừng vội kiểm tra quá sâu. Hãy kiểm tra selector và Pod label trước.

## 17. Troubleshooting Kubernetes Service

### 17.1. Checklist debug nhanh

Khi Service lỗi, kiểm tra theo thứ tự:

```bash
kubectl get svc -n <namespace>
kubectl describe svc <service-name> -n <namespace>
kubectl get pods -n <namespace> --show-labels
kubectl get endpointslice -n <namespace> -l kubernetes.io/service-name=<service-name>
kubectl describe endpointslice -n <namespace> -l kubernetes.io/service-name=<service-name>
kubectl get pods -n kube-system -l k8s-app=kube-dns
kubectl get pods -n kube-system -l k8s-app=kube-proxy
```

### 17.2. Lỗi selector không match label

Triệu chứng:

```bash
kubectl get endpointslice -n lab-networking -l kubernetes.io/service-name=<service-name>
```

Không có endpoint phù hợp.

Nguyên nhân:

- Service selector sai.
- Pod label sai.
- Nhầm namespace.

Cách kiểm tra:

```bash
kubectl describe svc <service-name> -n <namespace>
kubectl get pods -n <namespace> --show-labels
```

### 17.3. Lỗi targetPort sai

Triệu chứng:

- Service có endpoint.
- DNS resolve được.
- Nhưng curl bị connection refused hoặc timeout.

Nguyên nhân:

- Service forward đến port mà container không listen.

Ví dụ Service sai:

```yaml
ports:
  - port: 80
    targetPort: 8080
```

Trong khi container NGINX listen port 80.

Cách kiểm tra:

```bash
kubectl describe svc <service-name> -n <namespace>
kubectl describe pod <pod-name> -n <namespace>
```

### 17.4. Lỗi DNS không resolve

Triệu chứng:

```text
Could not resolve host
```

Kiểm tra CoreDNS:

```bash
kubectl get pods -n kube-system -l k8s-app=kube-dns
kubectl logs -n kube-system -l k8s-app=kube-dns
```

Kiểm tra file DNS trong Pod:

```bash
kubectl exec -it test-client -n lab-networking -- cat /etc/resolv.conf
```

Kiểm tra gọi FQDN đầy đủ:

```bash
kubectl exec -it test-client -n lab-networking -- nslookup backend-web-clusterip.lab-networking.svc.cluster.local
```

Nếu gọi tên ngắn không được nhưng FQDN được, có thể Pod ở namespace khác hoặc search domain không như mong đợi.

### 17.5. Lỗi NodePort không truy cập được

Triệu chứng:

Từ ngoài cluster:

```bash
curl http://<NodeIP>:30080
```

không kết nối được.

Kiểm tra:

```bash
kubectl get svc backend-web-nodeport -n lab-networking
kubectl get endpointslice -n lab-networking -l kubernetes.io/service-name=backend-web-nodeport
kubectl get pods -n kube-system -l k8s-app=kube-proxy -o wide
```

Kiểm tra node network:

```bash
ping <NodeIP>
nc -vz <NodeIP> 30080
```

Trên Ubuntu node, nếu có dùng firewall:

```bash
sudo ufw status
sudo iptables -L -n | head
```

Lưu ý: tùy môi trường lab, firewall vật lý, security group, routing hoặc NAT bên ngoài có thể chặn traffic đến NodePort.

### 17.6. Lỗi ExternalName với HTTP/HTTPS

Triệu chứng:

- `nslookup` ra CNAME đúng.
- Nhưng `curl https://external-name-service` lỗi TLS hoặc trả response lạ.

Nguyên nhân:

ExternalName chỉ đổi DNS ở mức CNAME. Nó không sửa HTTP Host header hoặc TLS SNI theo ý bạn.

Ví dụ client gọi:

```text
https://external-kubernetes
```

Nhưng certificate bên ngoài cấp cho:

```text
kubernetes.io
```

Có thể phát sinh lỗi certificate mismatch.

## 18. Bảng so sánh các loại Service trong buổi này

| Service type | Có ClusterIP | Có selector | Có DNS nội bộ | Truy cập từ ngoài cluster | Use case chính |
|---|---:|---:|---:|---:|---|
| ClusterIP | Có | Thường có | Có | Không trực tiếp | Giao tiếp nội bộ giữa các ứng dụng |
| NodePort | Có | Thường có | Có | Có, qua `NodeIP:nodePort` | Lab, on-prem đơn giản, expose tạm thời |
| Headless | Không | Có hoặc không | Có | Không trực tiếp | StatefulSet, service discovery từng Pod |
| ExternalName | Không | Không | Có, dạng CNAME | Không proxy | Ánh xạ Service name đến DNS ngoài cluster |

## 19. Các lệnh kubectl quan trọng trong buổi này

Xem Service:

```bash
kubectl get svc -n lab-networking
```

Describe Service:

```bash
kubectl describe svc <service-name> -n lab-networking
```

Xem EndpointSlice:

```bash
kubectl get endpointslice -n lab-networking
```

Lọc EndpointSlice theo Service:

```bash
kubectl get endpointslice -n lab-networking -l kubernetes.io/service-name=<service-name>
```

Describe EndpointSlice:

```bash
kubectl describe endpointslice -n lab-networking -l kubernetes.io/service-name=<service-name>
```

Xem Pod IP:

```bash
kubectl get pods -n lab-networking -o wide
```

Xem label của Pod:

```bash
kubectl get pods -n lab-networking --show-labels
```

Test HTTP từ Pod client:

```bash
kubectl exec -it test-client -n lab-networking -- curl -I http://<service-name>
```

Test DNS từ Pod client:

```bash
kubectl exec -it test-client -n lab-networking -- nslookup <service-name>
```

Xem CoreDNS:

```bash
kubectl get pods -n kube-system -l k8s-app=kube-dns
```

Xem kube-proxy:

```bash
kubectl get pods -n kube-system -l k8s-app=kube-proxy -o wide
```

## 20. Bài tập thực hành thêm cho 

### Bài tập 1 - Đổi Service port

Yêu cầu:

- Tạo Service `backend-web-port8080`.
- Client gọi Service qua port `8080`.
- Service forward vào Pod port `80`.

Gợi ý manifest:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: backend-web-port8080
  namespace: lab-networking
spec:
  type: ClusterIP
  selector:
    app: backend-web
  ports:
    - name: http-alt
      protocol: TCP
      port: 8080
      targetPort: http
```

Test:

```bash
kubectl apply -f <file>.yaml
kubectl exec -it test-client -n lab-networking -- curl -I http://backend-web-port8080:8080
```

### Bài tập 2 - Không khai báo nodePort để Kubernetes tự cấp

Yêu cầu:

- Tạo Service type NodePort nhưng không khai báo `nodePort`.
- Quan sát Kubernetes tự cấp port nào.

Gợi ý:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: backend-web-auto-nodeport
  namespace: lab-networking
spec:
  type: NodePort
  selector:
    app: backend-web
  ports:
    - name: http
      protocol: TCP
      port: 80
      targetPort: http
```

Kiểm tra:

```bash
kubectl get svc backend-web-auto-nodeport -n lab-networking
```

### Bài tập 3 - Làm lỗi selector rồi sửa

Yêu cầu:

- Tạo Service selector sai.
- Kiểm tra EndpointSlice không có endpoint.
- Sửa selector bằng `kubectl patch`.
- Kiểm tra Service hoạt động lại.

### Bài tập 4 - So sánh DNS ClusterIP và Headless

Yêu cầu:

- `nslookup backend-web-clusterip`
- `nslookup stateful-web-headless`
- Ghi lại sự khác nhau giữa kết quả DNS.

Gợi ý kết luận:

- ClusterIP Service trả về một Service IP.
- Headless Service trả về danh sách Pod IP.

## 21. Cleanup lab

Nếu muốn xóa toàn bộ object trong buổi học:

```bash
kubectl delete namespace lab-networking
```

Nếu chỉ muốn xóa từng phần:

```bash
kubectl delete -f 05-externalname-service.yaml
kubectl delete -f 04-headless-statefulset.yaml
kubectl delete -f 03-backend-nodeport-service.yaml
kubectl delete -f 02-backend-clusterip-service.yaml
kubectl delete -f 01-backend-deployment.yaml
kubectl delete -f 00-namespace.yaml
```

## 22. Câu hỏi kiểm tra cuối buổi

### Câu 1

Vì sao không nên để frontend gọi trực tiếp Pod IP của backend?

Gợi ý trả lời:

Pod là tài nguyên ephemeral. Khi Pod bị xóa, recreate hoặc rollout, Pod IP có thể thay đổi. Service cung cấp endpoint ổn định để client không cần theo dõi từng Pod backend.

### Câu 2

Service dùng trường nào để chọn Pod backend?

Gợi ý trả lời:

Service thường dùng `spec.selector` để match với label của Pod.

### Câu 3

Sự khác nhau giữa `port` và `targetPort` là gì?

Gợi ý trả lời:

`port` là port của Service mà client gọi. `targetPort` là port trên Pod backend mà Service forward traffic đến.

### Câu 4

NodePort mở port ở đâu?

Gợi ý trả lời:

NodePort mở port trên mỗi Node IP trong cluster. Client bên ngoài có thể truy cập bằng `NodeIP:nodePort` nếu network và firewall cho phép.

### Câu 5

Headless Service khác ClusterIP Service ở điểm nào?

Gợi ý trả lời:

Headless Service có `clusterIP: None`, không được cấp ClusterIP, không dùng kube-proxy để load balancing qua Service IP. DNS của Headless Service trả về IP của các Pod backend.

### Câu 6

ExternalName Service có proxy traffic không?

Gợi ý trả lời:

Không. ExternalName chỉ tạo DNS CNAME trỏ đến một DNS name bên ngoài. Không có selector, không có ClusterIP và không proxy traffic.

### Câu 7

Khi Service không có endpoint, nên kiểm tra gì đầu tiên?

Gợi ý trả lời:

Kiểm tra selector của Service và label của Pod. Sau đó kiểm tra Pod có Ready không, namespace có đúng không, và EndpointSlice có được tạo không.

### Câu 8

CoreDNS có vai trò gì trong Service discovery?

Gợi ý trả lời:

CoreDNS cung cấp DNS record cho Service và Pod, giúp ứng dụng trong cluster gọi Service bằng tên thay vì IP.

### Câu 9

EndpointSlice dùng để làm gì?

Gợi ý trả lời:

EndpointSlice lưu danh sách backend endpoint của Service, thường là IP và port của các Pod match selector. kube-proxy dùng EndpointSlice làm nguồn thông tin để route traffic nội bộ.

### Câu 10

Vì sao bài này chưa dùng Service type LoadBalancer?

Gợi ý trả lời:

Vì LoadBalancer cần tích hợp với cloud provider hoặc một giải pháp load balancer ngoài như MetalLB trong môi trường bare-metal. Nội dung này nên tách sang bài riêng về expose ứng dụng nâng cao hoặc Kubernetes trên AWS/Azure/GCP.

## 23. Tổng kết buổi học

Sau buổi này,  cần nhớ các ý chính:

- Pod IP có thể thay đổi, không nên hard-code trong ứng dụng.
- Service cung cấp endpoint ổn định cho một nhóm Pod backend.
- Service thường chọn Pod thông qua selector và label.
- EndpointSlice cho biết Service đang trỏ đến backend endpoint nào.
- CoreDNS giúp Pod gọi Service bằng DNS name.
- kube-proxy giúp Service ClusterIP và NodePort hoạt động trên node.
- ClusterIP dùng cho truy cập nội bộ trong cluster.
- NodePort dùng để truy cập Service từ ngoài cluster qua `NodeIP:nodePort`.
- Headless Service không có ClusterIP, DNS trả về Pod IP, thường dùng với StatefulSet.
- ExternalName ánh xạ Service name đến DNS name bên ngoài bằng CNAME.
- Cần phân biệt rõ `containerPort`, `targetPort`, `port` và `nodePort`.

## 24. Kiến thức đã được mở khóa sau buổi 04

Sau buổi 04, các buổi tiếp theo được phép sử dụng thêm:

- Kubernetes networking model
- Node IP
- Pod IP
- Service IP
- Service CIDR
- Pod CIDR
- CoreDNS service discovery
- EndpointSlice
- ClusterIP Service
- NodePort Service
- Headless Service
- ExternalName Service
- `containerPort`
- `targetPort`
- `port`
- `nodePort`
- DNS name dạng `<service>.<namespace>.svc.cluster.local`
- Debug Service bằng selector, label và EndpointSlice

## 25. Gợi ý buổi tiếp theo

Buổi tiếp theo có thể đi vào một trong các hướng sau:

- ConfigMap và Secret.
- Volume, PersistentVolume, PersistentVolumeClaim và StorageClass.
- Probes: liveness, readiness và startup.
- Ingress và expose HTTP application.
- NetworkPolicy với Calico.

Nếu đi theo tiến trình nền tảng ứng dụng, nên học `ConfigMap` và `Secret` trước, vì ứng dụng thực tế thường cần cấu hình và thông tin nhạy cảm trước khi đi đến storage hoặc ingress.

## 26. Nguồn tham khảo chính

- Kubernetes v1.35 Documentation - Service: https://v1-35.docs.kubernetes.io/docs/concepts/services-networking/service/
- Kubernetes v1.35 Documentation - DNS for Services and Pods: https://v1-35.docs.kubernetes.io/docs/concepts/services-networking/dns-pod-service/
- Kubernetes v1.35 Documentation - EndpointSlices: https://v1-35.docs.kubernetes.io/docs/concepts/services-networking/endpoint-slices/
- Kubernetes v1.35 Documentation - Cluster Networking: https://v1-35.docs.kubernetes.io/docs/concepts/cluster-administration/networking/
- Kubernetes v1.35 Reference - kube-proxy: https://v1-35.docs.kubernetes.io/docs/reference/command-line-tools-reference/kube-proxy/

Lưu ý: Kubernetes v1.35 documentation hiện là static snapshot và không còn actively maintained. Tuy nhiên bài giảng này cố ý bám theo v1.35 để phù hợp với môi trường lab của khóa học.
