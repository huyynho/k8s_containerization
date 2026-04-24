# BUỔI 02 - TRIỂN KHAI CỤM KUBERNETES BẰNG KUBEADM, CONTAINERD VÀ CALICO

## Thông tin bài học

**Khóa học:** Kubernetes nền tảng  
**Buổi:** 02  
**Chủ đề:** Triển khai cụm Kubernetes 1 control plane node và 2 worker node  
**Phiên bản mục tiêu:** Kubernetes v1.35  
**Hệ điều hành:** Ubuntu Server 24.04  
**Container Runtime:** containerd  
**CNI:** Calico  
**Công cụ bootstrap:** kubeadm  
**Mô hình:** On-premise lab, 3 VM  

---

## Vị trí của buổi học trong toàn bộ khóa Kubernetes

Ở buổi 01,  đã được học về kiến trúc Kubernetes ở mức tổng quan:

- Kubernetes cluster
- Control Plane
- Worker Node
- kube-apiserver
- etcd
- kube-scheduler
- kube-controller-manager
- kubelet
- container runtime
- CNI
- CoreDNS
- Pod
- Namespace
- Static Pod
- kubectl

Buổi 02 sẽ biến kiến thức kiến trúc đó thành một cụm Kubernetes thật.

Mục tiêu của buổi này không phải là triển khai ứng dụng phức tạp, cũng chưa phải là học Deployment, Service, Volume, Ingress hay Scheduler nâng cao. Mục tiêu chính là giúp  hiểu được:

- Một cụm Kubernetes được bootstrap như thế nào.
- kubeadm làm gì khi chạy `kubeadm init`.
- Vì sao cần container runtime.
- Vì sao cần CNI.
- Vì sao sau khi `kubeadm init` xong node thường ở trạng thái `NotReady`.
- Vì sao cần `--pod-network-cidr`.
- Vì sao nên cấu hình `--control-plane-endpoint` ngay cả khi chỉ có một control plane node.
- Vì sao cần `--apiserver-cert-extra-sans`.
- Cách dùng `--dry-run` để xem trước hành động của kubeadm trước khi thay đổi hệ thống.

---

## Mục tiêu sau buổi học

Sau buổi học này,  có thể:

- Chuẩn bị 3 VM Ubuntu 24.04 để cài Kubernetes.
- Cài đặt containerd làm container runtime.
- Cấu hình containerd dùng systemd cgroup driver.
- Cài đặt kubeadm, kubelet và kubectl từ repository Kubernetes v1.35.
- Bootstrap control plane node bằng `kubeadm init`.
- Giải thích được các flag quan trọng:
  - `--pod-network-cidr`
  - `--apiserver-cert-extra-sans`
  - `--dry-run`
  - `--control-plane-endpoint`
  - `--cri-socket`
  - `--kubernetes-version`
- Cài đặt Calico làm CNI.
- Join 2 worker node vào cluster.
- Kiểm tra trạng thái cluster sau khi triển khai.
- Tạo Pod kiểm thử đơn giản để xác nhận cluster đã hoạt động.

---

## Giới hạn kiến thức của buổi này

Để đảm bảo lộ trình học có thứ tự, buổi này chỉ sử dụng các khái niệm đã học từ buổi 01 và các khái niệm mới cần thiết cho việc dựng cluster.

### Được sử dụng trong buổi này

- Node
- Control Plane
- Worker Node
- Pod
- Namespace
- Static Pod
- kubelet
- container runtime
- CNI
- CoreDNS
- kube-proxy
- kubectl
- kubeadm
- containerd
- Calico

### Chưa sử dụng trong buổi này

- Deployment
- ReplicaSet
- Service
- Ingress
- ConfigMap
- Secret
- Volume
- PersistentVolume
- PersistentVolumeClaim
- StorageClass
- Rolling Update
- Rollback
- Liveness Probe
- Readiness Probe
- Node Affinity
- Taint và Toleration nâng cao
- RBAC nâng cao

Một số thành phần như `kube-proxy`, `CoreDNS`, `ConfigMap` có thể xuất hiện trong output của cluster, nhưng ở buổi này chỉ quan sát ở mức nhận diện. Các bài sau sẽ phân tích sâu hơn.

---

# PHẦN 1 - MÔ HÌNH TRIỂN KHAI LAB

## Sơ đồ tổng quan

```text
+---------------------------------------------------------------+
|                    Kubernetes Cluster v1.35                   |
|                                                               |
|  Control Plane Endpoint: k8s-api.k8s.local:6443               |
|  Pod CIDR:               192.168.0.0/16                       |
|  Service CIDR:           10.96.0.0/12                         |
|                                                               |
|  +-----------------------+                                    |
|  | k8s-master-01         |                                    |
|  | IP: 10.10.10.11       |                                    |
|  | Role: Control Plane   |                                    |
|  | containerd            |                                    |
|  | kubelet               |                                    |
|  | kubeadm               |                                    |
|  | kubectl               |                                    |
|  +-----------------------+                                    |
|                                                               |
|  +-----------------------+      +-----------------------+     |
|  | k8s-worker-01         |      | k8s-worker-02         |     |
|  | IP: 10.10.10.21       |      | IP: 10.10.10.22       |     |
|  | Role: Worker          |      | Role: Worker          |     |
|  | containerd            |      | containerd            |     |
|  | kubelet               |      | kubelet               |     |
|  | kubeadm               |      | kubeadm               |     |
|  +-----------------------+      +-----------------------+     |
|                                                               |
+---------------------------------------------------------------+
```

---

## Bảng thông tin VM

| Hostname | IP Address | Vai trò | OS | Ghi chú |
|---|---:|---|---|---|
| `k8s-master-01` | `10.10.10.11` | Control Plane | Ubuntu 24.04 | Chạy kube-apiserver, etcd, scheduler, controller-manager |
| `k8s-worker-01` | `10.10.10.21` | Worker Node | Ubuntu 24.04 | Chạy workload |
| `k8s-worker-02` | `10.10.10.22` | Worker Node | Ubuntu 24.04 | Chạy workload |

---

## Yêu cầu tài nguyên tối thiểu cho lab

| Thành phần | CPU | RAM | Disk |
|---|---:|---:|---:|
| Control Plane | 2 vCPU | 4 GB | 40 GB |
| Worker Node | 2 vCPU | 4 GB | 40 GB |

Trong môi trường lớp học, mỗi VM nên có ít nhất 2 vCPU và 4 GB RAM để tránh lỗi do thiếu tài nguyên khi kéo image, chạy Calico, CoreDNS và Pod kiểm thử.

---

## Quy hoạch địa chỉ mạng

| Loại mạng | CIDR / Giá trị | Ý nghĩa |
|---|---|---|
| Node Network | `10.10.10.0/24` | Mạng thật của các VM |
| Control Plane Endpoint | `k8s-api.k8s.local:6443` | Endpoint ổn định để truy cập Kubernetes API |
| Pod Network | `192.168.0.0/16` | Dải IP cấp cho Pod |
| Service Network | `10.96.0.0/12` | Dải IP ảo dành cho Service, giữ mặc định của kubeadm |

Ở buổi này chưa học Service, nhưng vẫn cần biết kubeadm mặc định tạo một dải IP riêng cho Service. Dải này là `10.96.0.0/12`.

---

## Vì sao chọn Pod CIDR là 192.168.0.0/16?

Calico manifest truyền thống mặc định dùng dải Pod network `192.168.0.0/16`.

Vì đây là bài lab đầu tiên, ta chọn:

```text
192.168.0.0/16
```

Lý do:

- Phù hợp với mặc định phổ biến của Calico.
- Giảm rủi ro cấu hình sai CNI.
- Dễ quan sát khi so sánh Node IP và Pod IP.
- Không trùng với Node Network `10.10.10.0/24`.

Điểm quan trọng nhất là Pod CIDR không được trùng với mạng thật của các node. Nếu Pod network trùng với host network, cluster có thể gặp lỗi định tuyến rất khó debug.

---

# PHẦN 2 - NHẮC LẠI KIẾN TRÚC TRƯỚC KHI TRIỂN KHAI

## kubeadm là gì?

`kubeadm` là công cụ dùng để bootstrap Kubernetes cluster.

Nói đơn giản, kubeadm giúp ta dựng phần xương sống của cluster.

Khi chạy `kubeadm init` trên control plane node, kubeadm sẽ thực hiện nhiều bước quan trọng:

- Kiểm tra điều kiện hệ thống trước khi cài.
- Tạo certificate cho control plane.
- Tạo kubeconfig cho các component.
- Tạo static Pod manifest cho:
  - kube-apiserver
  - etcd
  - kube-scheduler
  - kube-controller-manager
- Khởi động kubelet.
- Chờ control plane hoạt động.
- Tạo bootstrap token để worker node join cluster.
- Cài các add-on cơ bản như CoreDNS và kube-proxy.

---

## kubelet nằm ở đâu trong quá trình bootstrap?

Ở buổi 01, ta đã học kubelet là agent chạy trên mỗi node.

Trong quá trình bootstrap:

```text
kubeadm tạo cấu hình
        |
        v
kubelet đọc cấu hình
        |
        v
kubelet tạo static Pod cho control plane
        |
        v
containerd chạy container thực tế
```

Trên control plane node, kubelet sẽ đọc các file manifest trong thư mục:

```text
/etc/kubernetes/manifests
```

Các file này chính là static Pod manifest của control plane.

Sau khi `kubeadm init` thành công, ta có thể thấy:

```bash
sudo ls -l /etc/kubernetes/manifests
```

Output kỳ vọng:

```text
etcd.yaml
kube-apiserver.yaml
kube-controller-manager.yaml
kube-scheduler.yaml
```

Điểm cần nhớ:

> kubeadm không trực tiếp chạy kube-apiserver. kubeadm tạo manifest. kubelet đọc manifest. containerd chạy container.

---

## containerd nằm ở đâu?

containerd là container runtime.

Kubernetes không tự chạy container trực tiếp. kubelet gọi xuống container runtime thông qua CRI, sau đó container runtime mới tạo container thực tế.

Luồng đơn giản:

```text
kube-apiserver
     |
     v
kubelet trên node
     |
     v
CRI socket
     |
     v
containerd
     |
     v
container thực tế
```

Trong bài lab này, socket của containerd là:

```text
unix:///run/containerd/containerd.sock
```

Đây là lý do trong lệnh `kubeadm init` ta có thể chỉ rõ:

```bash
--cri-socket=unix:///run/containerd/containerd.sock
```

---

## Calico nằm ở đâu?

Calico là CNI plugin.

Sau khi cluster được bootstrap bằng kubeadm, các Pod hệ thống như CoreDNS có thể chưa chạy được ngay vì cluster chưa có Pod network.

Trạng thái thường gặp sau khi `kubeadm init` nhưng chưa cài CNI:

```text
k8s-master-01   NotReady
```

Lý do:

- kubelet đã chạy.
- control plane đã chạy.
- Nhưng Pod network chưa được cài.
- CoreDNS chưa thể chạy ổn định.
- Node chưa hoàn tất điều kiện network.

Sau khi cài Calico, node mới chuyển sang:

```text
Ready
```

---

# PHẦN 3 - CHUẨN BỊ HỆ THỐNG TRÊN TẤT CẢ NODE

Các bước trong phần này thực hiện trên cả 3 VM:

- `k8s-master-01`
- `k8s-worker-01`
- `k8s-worker-02`

---

## Bước 1: Cấu hình hostname

Trên master node:

```bash
sudo hostnamectl set-hostname k8s-master-01
```

Trên worker node 01:

```bash
sudo hostnamectl set-hostname k8s-worker-01
```

Trên worker node 02:

```bash
sudo hostnamectl set-hostname k8s-worker-02
```

Kiểm tra:

```bash
hostnamectl
```

---

## Bước 2: Cấu hình file hosts

Thực hiện trên cả 3 node:

```bash
sudo tee -a /etc/hosts > /dev/null <<EOF
10.10.10.11 k8s-master-01
10.10.10.21 k8s-worker-01
10.10.10.22 k8s-worker-02
10.10.10.11 k8s-api.k8s.local
EOF
```

Giải thích:

```text
10.10.10.11 k8s-master-01
```

Dòng này ánh xạ hostname của master node về IP thật của master.

```text
10.10.10.21 k8s-worker-01
10.10.10.22 k8s-worker-02
```

Hai dòng này giúp các node phân giải được hostname của nhau.

```text
10.10.10.11 k8s-api.k8s.local
```

Dòng này ánh xạ control plane endpoint về IP master hiện tại.

Ở mô hình một control plane node, endpoint này trỏ trực tiếp về master. Nếu sau này nâng cấp thành HA, ta có thể chuyển `k8s-api.k8s.local` sang IP của Load Balancer mà không cần thay đổi cách worker node kết nối đến API Server.

Kiểm tra phân giải tên:

```bash
ping -c 3 k8s-master-01
ping -c 3 k8s-worker-01
ping -c 3 k8s-worker-02
ping -c 3 k8s-api.k8s.local
```

---

## Bước 3: Tắt swap

Kubernetes yêu cầu swap phải được tắt, trừ khi cụm được cấu hình rõ ràng để hỗ trợ swap. Với bài lab nền tảng, ta tắt swap để tránh lỗi preflight.

Tắt swap ngay lập tức:

```bash
sudo swapoff -a
```

Vô hiệu hóa swap sau reboot:

```bash
sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab
```

Kiểm tra:

```bash
free -h
swapon --show
```

Kết quả mong muốn:

```text
swapon --show
```

Không trả về dòng nào.

---

## Bước 4: Load kernel modules cần thiết

Thực hiện trên cả 3 node:

```bash
sudo tee /etc/modules-load.d/k8s.conf > /dev/null <<EOF
overlay
br_netfilter
EOF
```

Load module ngay:

```bash
sudo modprobe overlay
sudo modprobe br_netfilter
```

Kiểm tra:

```bash
lsmod | grep overlay
lsmod | grep br_netfilter
```

Giải thích:

```text
overlay
```

Module này liên quan đến overlay filesystem, thường được container runtime sử dụng để quản lý layer của container image.

```text
br_netfilter
```

Module này giúp Linux bridge traffic đi qua netfilter/iptables. Kubernetes networking thường cần module này để xử lý traffic giữa Pod, Service và node.

---

## Bước 5: Cấu hình sysctl cho Kubernetes networking

Thực hiện trên cả 3 node:

```bash
sudo tee /etc/sysctl.d/k8s.conf > /dev/null <<EOF
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward = 1
EOF
```

Apply cấu hình:

```bash
sudo sysctl --system
```

Kiểm tra:

```bash
sysctl net.bridge.bridge-nf-call-iptables
sysctl net.bridge.bridge-nf-call-ip6tables
sysctl net.ipv4.ip_forward
```

Kết quả mong muốn:

```text
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward = 1
```

Giải thích:

```text
net.bridge.bridge-nf-call-iptables = 1
```

Cho phép traffic đi qua Linux bridge được xử lý bởi iptables.

```text
net.bridge.bridge-nf-call-ip6tables = 1
```

Tương tự dòng trên nhưng dành cho IPv6.

```text
net.ipv4.ip_forward = 1
```

Cho phép node forward packet IPv4. Đây là yêu cầu quan trọng để Pod network hoạt động.

---

# PHẦN 4 - CÀI ĐẶT CONTAINERD

Các bước trong phần này thực hiện trên cả 3 node.

---

## Bước 1: Cài đặt containerd

```bash
sudo apt-get update
sudo apt-get install -y containerd
```

Kiểm tra version:

```bash
containerd --version
```

Kiểm tra trạng thái service:

```bash
sudo systemctl status containerd
```

---

## Bước 2: Tạo file cấu hình mặc định cho containerd

```bash
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml > /dev/null
```

File cấu hình chính:

```text
/etc/containerd/config.toml
```

---

## Bước 3: Cấu hình containerd dùng systemd cgroup driver

```bash
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml
```

Restart containerd:

```bash
sudo systemctl restart containerd
sudo systemctl enable containerd
```

Kiểm tra:

```bash
sudo grep -i SystemdCgroup /etc/containerd/config.toml
```

Kết quả mong muốn:

```text
SystemdCgroup = true
```

---

## Vì sao cần systemd cgroup driver?

Ubuntu 24.04 dùng systemd làm init system.

Nếu kubelet dùng systemd cgroup driver nhưng containerd dùng cgroupfs, hệ thống có thể gặp tình trạng mismatch cgroup driver. Điều này có thể gây lỗi quản lý tài nguyên và làm node không ổn định dưới tải.

Với kubeadm từ Kubernetes v1.22 trở lên, nếu không cấu hình riêng `cgroupDriver`, kubeadm mặc định dùng `systemd` cho kubelet. Vì vậy containerd cũng nên được cấu hình dùng systemd cgroup driver.

---

## Bước 4: Kiểm tra CRI socket

```bash
sudo ls -l /run/containerd/containerd.sock
```

Kết quả mong muốn:

```text
srw-rw---- 1 root root ... /run/containerd/containerd.sock
```

Socket này sẽ được kubelet/kubeadm sử dụng để giao tiếp với containerd.

---

# PHẦN 5 - CÀI ĐẶT KUBEADM, KUBELET VÀ KUBECTL

Các bước trong phần này thực hiện trên cả 3 node.

---

## Vai trò của từng công cụ

| Công cụ | Vai trò |
|---|---|
| kubeadm | Bootstrap cluster, init control plane, join worker node |
| kubelet | Agent chạy trên mỗi node |
| kubectl | CLI dùng để giao tiếp với Kubernetes API Server |

Trong production, worker node không nhất thiết phải có `kubectl`, nhưng trong lab nên cài trên tất cả node để tiện kiểm tra.

---

## Bước 1: Cài đặt package phụ thuộc

```bash
sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl gpg
```

---

## Bước 2: Thêm Kubernetes v1.35 repository

```bash
sudo mkdir -p -m 755 /etc/apt/keyrings
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.35/deb/Release.key \
  | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
```

Tạo file repository:

```bash
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.35/deb/ /' \
  | sudo tee /etc/apt/sources.list.d/kubernetes.list
```

Cập nhật package index:

```bash
sudo apt-get update
```

---

## Bước 3: Cài đặt kubelet, kubeadm và kubectl

```bash
sudo apt-get install -y kubelet kubeadm kubectl
```

Pin version để tránh tự động nâng cấp ngoài ý muốn:

```bash
sudo apt-mark hold kubelet kubeadm kubectl
```

Kiểm tra version:

```bash
kubeadm version
kubectl version --client
kubelet --version
```

---

## Vì sao cần apt-mark hold?

Kubernetes cluster yêu cầu version giữa các component phải được kiểm soát cẩn thận.

Nếu hệ điều hành tự động nâng `kubelet`, `kubeadm`, `kubectl` lên version mới hơn mà người quản trị chưa có kế hoạch upgrade cluster, có thể gây lệch version không mong muốn.

Trong lab và production, nên chủ động upgrade Kubernetes theo quy trình, không để package tự nâng cấp ngẫu nhiên.

---

# PHẦN 6 - KHỞI TẠO CONTROL PLANE BẰNG KUBEADM INIT

Phần này chỉ thực hiện trên node:

```text
k8s-master-01
```

---

## Bước 1: Kiểm tra image cần dùng cho control plane

```bash
sudo kubeadm config images list
```

Có thể pull trước image:

```bash
sudo kubeadm config images pull \
  --cri-socket=unix:///run/containerd/containerd.sock
```

Lệnh này giúp kiểm tra sớm việc node có thể kéo image từ registry hay không.

---

## Bước 2: Chạy kubeadm init với dry-run

Trước khi init thật, dùng `--dry-run` để xem kubeadm sẽ làm gì.

```bash
K8S_VERSION=$(kubeadm version -o short)

sudo kubeadm init \
  --kubernetes-version=${K8S_VERSION} \
  --pod-network-cidr=192.168.0.0/16 \
  --apiserver-cert-extra-sans=k8s-api.k8s.local,10.10.10.11 \
  --control-plane-endpoint=k8s-api.k8s.local:6443 \
  --cri-socket=unix:///run/containerd/containerd.sock \
  --dry-run
```

---

## Giải thích lệnh dry-run

```bash
K8S_VERSION=$(kubeadm version -o short)
```

Lấy version của kubeadm đang cài trên máy, ví dụ:

```text
v1.35.0
```

Việc dùng biến này giúp lệnh init bám sát version kubeadm đang dùng.

```bash
sudo kubeadm init
```

Bắt đầu quá trình bootstrap control plane.

```bash
--kubernetes-version=${K8S_VERSION}
```

Chỉ rõ version Kubernetes dùng cho control plane.

```bash
--pod-network-cidr=192.168.0.0/16
```

Chỉ định dải IP dành cho Pod network.

Khi flag này được cấu hình, control plane có thể tự động cấp Pod CIDR cho từng node. Đây là thông tin quan trọng để CNI như Calico thiết lập network cho Pod.

```bash
--apiserver-cert-extra-sans=k8s-api.k8s.local,10.10.10.11
```

Thêm DNS name và IP vào Subject Alternative Names của certificate cho kube-apiserver.

Nếu sau này truy cập API Server bằng `k8s-api.k8s.local`, certificate của API Server phải hợp lệ với tên đó. Nếu không, client có thể gặp lỗi TLS kiểu:

```text
x509: certificate is valid for ..., not k8s-api.k8s.local
```

```bash
--control-plane-endpoint=k8s-api.k8s.local:6443
```

Đặt endpoint ổn định cho control plane.

Trong lab một master, endpoint này trỏ về IP master. Trong production HA, endpoint này thường trỏ về Load Balancer đặt phía trước nhiều control plane node.

```bash
--cri-socket=unix:///run/containerd/containerd.sock
```

Chỉ rõ kubeadm dùng containerd thông qua CRI socket này.

```bash
--dry-run
```

Chỉ hiển thị các bước kubeadm sẽ thực hiện, không thay đổi hệ thống thật.

Đây là flag rất quan trọng trong đào tạo và triển khai thực tế vì giúp người quản trị kiểm tra trước cấu hình.

---

## Bước 3: Chạy kubeadm init thật

Sau khi dry-run không báo lỗi nghiêm trọng, chạy lệnh init thật:

```bash
K8S_VERSION=$(kubeadm version -o short)

sudo kubeadm init \
  --kubernetes-version=${K8S_VERSION} \
  --pod-network-cidr=192.168.0.0/16 \
  --apiserver-cert-extra-sans=k8s-api.k8s.local,10.10.10.11 \
  --control-plane-endpoint=k8s-api.k8s.local:6443 \
  --cri-socket=unix:///run/containerd/containerd.sock
```

Nếu thành công, kubeadm sẽ hiển thị thông báo tương tự:

```text
Your Kubernetes control-plane has initialized successfully!
```

Đồng thời kubeadm sẽ in ra lệnh join worker node, ví dụ:

```text
kubeadm join k8s-api.k8s.local:6443 --token <token> \
    --discovery-token-ca-cert-hash sha256:<hash>
```

Cần lưu lại lệnh này để dùng ở worker node.

---

## Bước 4: Cấu hình kubeconfig cho user hiện tại

Trên master node:

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

Giải thích:

```bash
mkdir -p $HOME/.kube
```

Tạo thư mục chứa file kubeconfig.

```bash
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
```

Copy admin kubeconfig do kubeadm tạo ra vào thư mục của user hiện tại.

```bash
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

Đổi owner file kubeconfig để user hiện tại có thể dùng `kubectl` mà không cần sudo.

---

## Bước 5: Kiểm tra node sau khi init

```bash
kubectl get nodes
```

Kết quả có thể là:

```text
NAME            STATUS     ROLES           AGE   VERSION
k8s-master-01   NotReady   control-plane   1m    v1.35.x
```

Trạng thái `NotReady` lúc này là bình thường nếu chưa cài CNI.

Kiểm tra Pod hệ thống:

```bash
kubectl get pods -n kube-system
```

Có thể thấy CoreDNS chưa Running hoàn toàn.

Nguyên nhân:

```text
Cluster chưa có Pod network.
```

---

## Bước 6: Quan sát static Pod manifest của control plane

```bash
sudo ls -l /etc/kubernetes/manifests
```

Output kỳ vọng:

```text
etcd.yaml
kube-apiserver.yaml
kube-controller-manager.yaml
kube-scheduler.yaml
```

Xem manifest kube-apiserver:

```bash
sudo cat /etc/kubernetes/manifests/kube-apiserver.yaml
```

Một số dòng đáng chú ý:

```yaml
spec:
  containers:
  - command:
    - kube-apiserver
    - --advertise-address=10.10.10.11
    - --secure-port=6443
```

Giải thích:

```yaml
- kube-apiserver
```

Container này chạy tiến trình kube-apiserver.

```yaml
- --advertise-address=10.10.10.11
```

Địa chỉ API Server quảng bá cho các component khác.

```yaml
- --secure-port=6443
```

Port HTTPS của kube-apiserver.

Điểm cần nhấn mạnh:

> Control plane component trong kubeadm cluster chạy dưới dạng static Pod do kubelet quản lý.

---

# PHẦN 7 - PHÂN TÍCH CÁC FLAG QUAN TRỌNG CỦA KUBEADM INIT

## Flag 1: --pod-network-cidr

Cú pháp trong bài lab:

```bash
--pod-network-cidr=192.168.0.0/16
```

Ý nghĩa:

- Xác định dải IP dành cho Pod.
- Cho phép control plane cấp Pod CIDR cho từng node.
- CNI plugin dựa vào thông tin này để thiết lập network.
- Phải không được trùng với Node Network.

Ví dụ sai:

```text
Node Network: 192.168.0.0/24
Pod CIDR:     192.168.0.0/16
```

Trường hợp này bị overlap, dễ gây lỗi routing.

Ví dụ đúng:

```text
Node Network: 10.10.10.0/24
Pod CIDR:     192.168.0.0/16
```

Hai dải này tách biệt nhau.

---

## Flag 2: --apiserver-cert-extra-sans

Cú pháp trong bài lab:

```bash
--apiserver-cert-extra-sans=k8s-api.k8s.local,10.10.10.11
```

Ý nghĩa:

- Thêm DNS name hoặc IP vào certificate của kube-apiserver.
- Giúp client có thể truy cập API Server bằng endpoint tùy chỉnh mà không lỗi TLS.
- Nên khai báo tất cả tên hoặc IP mà quản trị viên dự kiến dùng để truy cập API Server.

Ví dụ nên thêm:

```text
k8s-api.k8s.local
10.10.10.11
```

Trong production có thể thêm:

```text
k8s-api.company.local
k8s-api.company.vn
IP Load Balancer
FQDN Load Balancer
```

Nếu quên flag này, sau khi cluster đã init, việc sửa certificate API Server sẽ phức tạp hơn so với khai báo đúng ngay từ đầu.

---

## Flag 3: --dry-run

Cú pháp trong bài lab:

```bash
--dry-run
```

Ý nghĩa:

- Cho kubeadm chạy ở chế độ mô phỏng.
- Không ghi thay đổi thật vào hệ thống.
- Giúp kiểm tra trước các bước kubeadm sẽ thực hiện.
- Rất hữu ích khi dạy học, viết tài liệu hoặc chuẩn bị triển khai production.

Nên dùng `--dry-run` trước khi chạy lệnh init thật.

Ví dụ quy trình an toàn:

```text
1. Chạy kubeadm init --dry-run
2. Đọc output
3. Sửa lỗi preflight nếu có
4. Chạy kubeadm init thật
```

---

## Flag 4: --control-plane-endpoint

Cú pháp trong bài lab:

```bash
--control-plane-endpoint=k8s-api.k8s.local:6443
```

Ý nghĩa:

- Đặt endpoint ổn định cho Kubernetes API Server.
- Worker node sẽ join cluster thông qua endpoint này.
- Kubeconfig cũng có thể dùng endpoint này để truy cập cluster.
- Rất quan trọng nếu sau này muốn nâng cấp sang mô hình HA control plane.

Trong mô hình lab:

```text
k8s-api.k8s.local -> 10.10.10.11
```

Trong mô hình HA sau này:

```text
k8s-api.k8s.local -> IP Load Balancer
```

Ví dụ:

```text
10.10.10.100 k8s-api.k8s.local
```

Trong đó `10.10.10.100` là IP của HAProxy, Keepalived, F5 hoặc Load Balancer khác.

Điểm quan trọng:

> Nên khai báo `--control-plane-endpoint` ngay từ lúc init cluster, kể cả khi hiện tại chỉ có một control plane node.

---

## Flag 5: --cri-socket

Cú pháp trong bài lab:

```bash
--cri-socket=unix:///run/containerd/containerd.sock
```

Ý nghĩa:

- Chỉ rõ container runtime mà kubeadm/kubelet sẽ dùng.
- Trong bài này runtime là containerd.
- Nếu node có nhiều runtime hoặc kubeadm không tự phát hiện đúng, flag này giúp tránh nhầm lẫn.

---

## Flag 6: --kubernetes-version

Cú pháp trong bài lab:

```bash
--kubernetes-version=${K8S_VERSION}
```

Ý nghĩa:

- Chỉ rõ version control plane sẽ dùng.
- Giúp tránh kubeadm tự chọn version không đúng kỳ vọng.
- Phù hợp khi lab yêu cầu bám sát Kubernetes v1.35.

---

# PHẦN 8 - CÀI ĐẶT CALICO CNI

Phần này thực hiện trên master node sau khi đã cấu hình kubeconfig.

---

## Bước 1: Tải manifest Calico

```bash
curl https://raw.githubusercontent.com/projectcalico/calico/v3.31.4/manifests/calico.yaml -O
```

Kiểm tra file:

```bash
ls -lh calico.yaml
```

---

## Bước 2: Kiểm tra nhanh Pod CIDR trong manifest Calico

Vì bài lab dùng Pod CIDR mặc định `192.168.0.0/16`, ta có thể apply manifest trực tiếp.

Có thể kiểm tra nhanh:

```bash
grep -n "CALICO_IPV4POOL_CIDR" calico.yaml
```

Trong manifest Calico, biến `CALICO_IPV4POOL_CIDR` có thể đang được comment hoặc dùng giá trị mặc định. Với kubeadm và Pod CIDR `192.168.0.0/16`, cấu hình mặc định phù hợp với bài lab này.

Nếu trong một môi trường khác ta dùng Pod CIDR khác, ví dụ:

```text
172.16.0.0/16
```

thì cần kiểm tra kỹ cách Calico nhận CIDR từ kubeadm hoặc tùy chỉnh manifest tương ứng.

---

## Bước 3: Apply Calico manifest

```bash
kubectl apply -f calico.yaml
```

---

## Bước 4: Theo dõi Pod hệ thống

```bash
kubectl get pods -n kube-system -w
```

Chờ đến khi các Pod quan trọng Running:

```text
calico-kube-controllers-xxxxx   Running
calico-node-xxxxx               Running
coredns-xxxxx                   Running
kube-proxy-xxxxx                Running
```

Thoát chế độ watch bằng:

```text
Ctrl + C
```

---

## Bước 5: Kiểm tra trạng thái node

```bash
kubectl get nodes -o wide
```

Kết quả mong muốn sau khi Calico hoạt động:

```text
NAME            STATUS   ROLES           AGE   VERSION   INTERNAL-IP
k8s-master-01   Ready    control-plane   10m   v1.35.x   10.10.10.11
```

Lúc này master node đã Ready, nhưng worker node chưa xuất hiện vì chưa join cluster.

---

# PHẦN 9 - JOIN WORKER NODE VÀO CLUSTER

Phần này thực hiện trên:

- `k8s-worker-01`
- `k8s-worker-02`

---

## Bước 1: Sử dụng lệnh join do kubeadm init sinh ra

Ví dụ lệnh join:

```bash
sudo kubeadm join k8s-api.k8s.local:6443 \
  --token <token> \
  --discovery-token-ca-cert-hash sha256:<hash> \
  --cri-socket=unix:///run/containerd/containerd.sock
```

Trong đó:

```text
k8s-api.k8s.local:6443
```

Là endpoint của Kubernetes API Server.

```text
--token <token>
```

Token dùng để worker node bootstrap vào cluster.

```text
--discovery-token-ca-cert-hash sha256:<hash>
```

Hash dùng để xác thực CA certificate của control plane, giúp worker node tránh join nhầm vào cluster không tin cậy.

```text
--cri-socket=unix:///run/containerd/containerd.sock
```

Chỉ rõ worker node dùng containerd làm runtime.

---

## Bước 2: Nếu mất lệnh join thì tạo lại

Trên master node:

```bash
kubeadm token create --print-join-command
```

Lệnh này sẽ in ra join command mới.

Nếu muốn xem token hiện có:

```bash
kubeadm token list
```

---

## Bước 3: Kiểm tra node sau khi join

Trên master node:

```bash
kubectl get nodes -o wide
```

Kết quả mong muốn:

```text
NAME            STATUS   ROLES           AGE   VERSION   INTERNAL-IP
k8s-master-01   Ready    control-plane   20m   v1.35.x   10.10.10.11
k8s-worker-01   Ready    <none>          3m    v1.35.x   10.10.10.21
k8s-worker-02   Ready    <none>          3m    v1.35.x   10.10.10.22
```

Giải thích:

```text
ROLES: control-plane
```

Node này là control plane node.

```text
ROLES: <none>
```

Đây là worker node. Việc hiển thị `<none>` là bình thường, không phải lỗi.

---

# PHẦN 10 - KIỂM TRA CLUSTER SAU TRIỂN KHAI

## Kiểm tra toàn bộ Pod hệ thống

```bash
kubectl get pods -A -o wide
```

Output kỳ vọng:

```text
NAMESPACE     NAME                                      READY   STATUS    NODE
kube-system   calico-kube-controllers-xxxxx             1/1     Running   k8s-master-01
kube-system   calico-node-xxxxx                         1/1     Running   k8s-master-01
kube-system   calico-node-xxxxx                         1/1     Running   k8s-worker-01
kube-system   calico-node-xxxxx                         1/1     Running   k8s-worker-02
kube-system   coredns-xxxxx                             1/1     Running   k8s-worker-01
kube-system   coredns-xxxxx                             1/1     Running   k8s-worker-02
kube-system   etcd-k8s-master-01                        1/1     Running   k8s-master-01
kube-system   kube-apiserver-k8s-master-01              1/1     Running   k8s-master-01
kube-system   kube-controller-manager-k8s-master-01     1/1     Running   k8s-master-01
kube-system   kube-proxy-xxxxx                          1/1     Running   k8s-master-01
kube-system   kube-proxy-xxxxx                          1/1     Running   k8s-worker-01
kube-system   kube-proxy-xxxxx                          1/1     Running   k8s-worker-02
kube-system   kube-scheduler-k8s-master-01              1/1     Running   k8s-master-01
```

Không cần học chi tiết tất cả Pod này ở buổi 02, nhưng  cần nhận diện được:

- `kube-apiserver`
- `etcd`
- `kube-scheduler`
- `kube-controller-manager`
- `kube-proxy`
- `coredns`
- `calico-node`
- `calico-kube-controllers`

---

## Kiểm tra runtime trên node

Trên master node:

```bash
kubectl get nodes -o custom-columns=NAME:.metadata.name,RUNTIME:.status.nodeInfo.containerRuntimeVersion,KUBELET:.status.nodeInfo.kubeletVersion,OS:.status.nodeInfo.osImage
```

Output ví dụ:

```text
NAME            RUNTIME                KUBELET   OS
k8s-master-01   containerd://1.7.x      v1.35.x   Ubuntu 24.04.x LTS
k8s-worker-01   containerd://1.7.x      v1.35.x   Ubuntu 24.04.x LTS
k8s-worker-02   containerd://1.7.x      v1.35.x   Ubuntu 24.04.x LTS
```

Điểm cần nhấn mạnh:

> Cluster đang dùng containerd làm container runtime, không dùng Docker Engine.

---

## Kiểm tra thông tin cluster

```bash
kubectl cluster-info
```

Output ví dụ:

```text
Kubernetes control plane is running at https://k8s-api.k8s.local:6443
CoreDNS is running at ...
```

Nếu output hiển thị `k8s-api.k8s.local:6443`, nghĩa là control plane endpoint đã được sử dụng đúng.

---

# PHẦN 11 - MANIFEST KIỂM THỬ POD ĐƠN GIẢN

Ở buổi 02, ta chỉ dùng Pod đơn lẻ để kiểm tra cluster. Chưa dùng Deployment.

---

## File: pod-nginx-basic.yaml

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-basic
  namespace: default
  labels:
    app: nginx-basic
spec:
  containers:
    - name: nginx
      image: nginx:1.27
      ports:
        - containerPort: 80
```

---

## Giải thích manifest

```yaml
apiVersion: v1
```

Khai báo API version của đối tượng Pod. Pod thuộc core API group nên dùng `v1`.

```yaml
kind: Pod
```

Khai báo loại object là Pod.

```yaml
metadata:
  name: nginx-basic
```

Tên của Pod là `nginx-basic`.

```yaml
namespace: default
```

Pod được tạo trong namespace `default`.

```yaml
labels:
  app: nginx-basic
```

Gắn label cho Pod. Ở buổi này label chỉ dùng để nhận diện, chưa dùng cho Service hay selector nâng cao.

```yaml
spec:
```

Phần mô tả trạng thái mong muốn của Pod.

```yaml
containers:
  - name: nginx
```

Pod này có một container tên là `nginx`.

```yaml
image: nginx:1.27
```

Container sử dụng image `nginx:1.27`.

```yaml
ports:
  - containerPort: 80
```

Khai báo container lắng nghe port 80. Dòng này mang tính mô tả metadata cho Kubernetes. Ở buổi này ta chưa expose ứng dụng ra ngoài bằng Service.

---

## Apply manifest

```bash
kubectl apply -f pod-nginx-basic.yaml
```

Kiểm tra:

```bash
kubectl get pod nginx-basic -o wide
```

Output ví dụ:

```text
NAME          READY   STATUS    RESTARTS   AGE   IP               NODE
nginx-basic   1/1     Running   0          30s   192.168.x.x      k8s-worker-01
```

Quan sát các thông tin:

```text
IP
```

Đây là Pod IP do Calico cấp.

```text
NODE
```

Đây là node mà Pod đang chạy.

---

## Kiểm tra Pod bằng describe

```bash
kubectl describe pod nginx-basic
```

Các phần cần quan sát:

```text
Node:
```

Pod đang chạy trên node nào.

```text
IP:
```

Pod được cấp IP nào.

```text
Events:
```

Các sự kiện trong quá trình Kubernetes tạo Pod, pull image, start container.

---

## Kiểm tra container bên trong Pod

```bash
kubectl exec nginx-basic -- nginx -v
```

Output ví dụ:

```text
nginx version: nginx/1.27.x
```

Lệnh này xác nhận container bên trong Pod đã chạy được.

---

## Xóa Pod kiểm thử

```bash
kubectl delete pod nginx-basic
```

Kiểm tra:

```bash
kubectl get pods
```

---

# PHẦN 12 - MANIFEST KIỂM THỬ POD-TO-POD NETWORK

Phần này dùng thêm một Pod tạm để kiểm tra Pod network. Vẫn chưa dùng Service.

---

## File: pod-curl-test.yaml

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: curl-test
  namespace: default
  labels:
    app: curl-test
spec:
  containers:
    - name: curl
      image: curlimages/curl:8.10.1
      command:
        - sleep
        - "3600"
```

---

## Giải thích manifest

```yaml
apiVersion: v1
kind: Pod
```

Đây vẫn là Pod đơn lẻ.

```yaml
metadata:
  name: curl-test
```

Tên Pod là `curl-test`.

```yaml
image: curlimages/curl:8.10.1
```

Pod sử dụng image có sẵn công cụ `curl`.

```yaml
command:
  - sleep
  - "3600"
```

Thay vì chạy xong rồi thoát, container sẽ sleep 3600 giây để  có thời gian exec vào kiểm tra.

---

## Apply hai Pod kiểm thử

Tạo lại Pod nginx nếu đã xóa:

```bash
kubectl apply -f pod-nginx-basic.yaml
kubectl apply -f pod-curl-test.yaml
```

Kiểm tra:

```bash
kubectl get pods -o wide
```

Lấy Pod IP của nginx:

```bash
NGINX_POD_IP=$(kubectl get pod nginx-basic -o jsonpath='{.status.podIP}')
echo $NGINX_POD_IP
```

Dùng Pod curl gọi trực tiếp Pod IP của nginx:

```bash
kubectl exec curl-test -- curl -s http://$NGINX_POD_IP
```

Nếu trả về HTML của nginx, Pod network đã hoạt động.

Lưu ý:

> Đây chỉ là kiểm tra trực tiếp Pod IP. Trong các buổi sau, khi học Service, ta sẽ không truy cập ứng dụng theo cách này trong mô hình thực tế.

---

## Dọn dẹp Pod kiểm thử

```bash
kubectl delete -f pod-curl-test.yaml
kubectl delete -f pod-nginx-basic.yaml
```

---

# PHẦN 13 - QUY TRÌNH RESET CLUSTER KHI LAB LỖI

Trong lớp học,  có thể init sai hoặc join sai. Ta cần biết cách reset node.

## Reset trên node cần làm lại

```bash
sudo kubeadm reset -f
```

Dọn cấu hình CNI:

```bash
sudo rm -rf /etc/cni/net.d
sudo rm -rf /var/lib/cni
```

Dọn kubeconfig của user hiện tại nếu là master node:

```bash
rm -rf $HOME/.kube
```

Restart containerd và kubelet:

```bash
sudo systemctl restart containerd
sudo systemctl restart kubelet
```

Kiểm tra:

```bash
sudo crictl ps -a
```

Nếu cần xóa container/image còn sót trong lab, có thể dùng thêm:

```bash
sudo crictl rm $(sudo crictl ps -aq) 2>/dev/null || true
sudo crictl rmi --prune
```

Cẩn thận:

> Không dùng các lệnh reset/dọn dẹp này trên production nếu chưa hiểu rõ tác động.

---

# PHẦN 14 - CÁC LỖI THƯỜNG GẶP

## Lỗi 1: Swap chưa tắt

Thông báo có thể gặp:

```text
[ERROR Swap]: running with swap on is not supported
```

Cách xử lý:

```bash
sudo swapoff -a
sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab
```

Kiểm tra:

```bash
swapon --show
```

---

## Lỗi 2: container runtime chưa chạy

Thông báo có thể gặp:

```text
container runtime is not running
```

Kiểm tra:

```bash
sudo systemctl status containerd
sudo ls -l /run/containerd/containerd.sock
```

Khởi động lại:

```bash
sudo systemctl restart containerd
```

---

## Lỗi 3: kubelet không khởi động ổn định

Kiểm tra log:

```bash
sudo journalctl -u kubelet -xe
```

Các nguyên nhân phổ biến:

- Swap chưa tắt.
- containerd chưa chạy.
- CRI socket sai.
- CNI chưa cài.
- cgroup driver mismatch.

---

## Lỗi 4: Node NotReady sau khi init

Kiểm tra:

```bash
kubectl describe node k8s-master-01
```

Nếu thấy liên quan đến network plugin, kiểm tra Calico:

```bash
kubectl get pods -n kube-system
kubectl logs -n kube-system -l k8s-app=calico-node
```

Nguyên nhân thường gặp:

- Chưa cài Calico.
- Calico Pod chưa Running.
- Pod CIDR sai hoặc overlap với Node Network.
- Node không forward được packet do thiếu sysctl.

---

## Lỗi 5: Worker join không được do không phân giải được endpoint

Thông báo có thể gặp:

```text
lookup k8s-api.k8s.local: no such host
```

Cách xử lý:

Trên worker node, kiểm tra:

```bash
cat /etc/hosts
ping -c 3 k8s-api.k8s.local
```

Nếu thiếu, thêm:

```bash
echo "10.10.10.11 k8s-api.k8s.local" | sudo tee -a /etc/hosts
```

---

## Lỗi 6: Lỗi certificate khi truy cập API Server qua endpoint

Thông báo có thể gặp:

```text
x509: certificate is valid for ..., not k8s-api.k8s.local
```

Nguyên nhân:

- Khi init cluster, chưa thêm `k8s-api.k8s.local` vào `--apiserver-cert-extra-sans`.

Cách phòng tránh:

```bash
--apiserver-cert-extra-sans=k8s-api.k8s.local,10.10.10.11
```

Nếu cluster đã init sai, trong lab cách đơn giản nhất là reset và init lại cho đúng.

---

# PHẦN 15 - LAB TỔNG HỢP

## Mục tiêu lab

Triển khai một cụm Kubernetes gồm:

- 1 control plane node
- 2 worker node
- Ubuntu 24.04
- containerd runtime
- Calico CNI
- kubeadm bootstrap
- Control Plane Endpoint: `k8s-api.k8s.local:6443`
- Pod CIDR: `192.168.0.0/16`

---

## Bài lab 1: Chuẩn bị tất cả node

Thực hiện trên cả 3 node:

```bash
sudo swapoff -a
sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab

sudo tee /etc/modules-load.d/k8s.conf > /dev/null <<EOF
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter

sudo tee /etc/sysctl.d/k8s.conf > /dev/null <<EOF
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward = 1
EOF

sudo sysctl --system
```

Cài containerd:

```bash
sudo apt-get update
sudo apt-get install -y containerd

sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml > /dev/null

sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml

sudo systemctl restart containerd
sudo systemctl enable containerd
```

Cài kubeadm, kubelet, kubectl:

```bash
sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl gpg

sudo mkdir -p -m 755 /etc/apt/keyrings

curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.35/deb/Release.key \
  | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.35/deb/ /' \
  | sudo tee /etc/apt/sources.list.d/kubernetes.list

sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```

---

## Bài lab 2: Init control plane với dry-run

Trên `k8s-master-01`:

```bash
K8S_VERSION=$(kubeadm version -o short)

sudo kubeadm init \
  --kubernetes-version=${K8S_VERSION} \
  --pod-network-cidr=192.168.0.0/16 \
  --apiserver-cert-extra-sans=k8s-api.k8s.local,10.10.10.11 \
  --control-plane-endpoint=k8s-api.k8s.local:6443 \
  --cri-socket=unix:///run/containerd/containerd.sock \
  --dry-run
```

Yêu cầu :

- Xác định kubeadm đang kiểm tra những gì.
- Xác định kubeadm sẽ tạo các component nào.
- Xác định các static Pod manifest sẽ được tạo ở đâu.
- Xác định flag nào liên quan đến Pod network.
- Xác định flag nào liên quan đến certificate của API Server.
- Xác định flag nào liên quan đến endpoint ổn định của control plane.

---

## Bài lab 3: Init control plane thật

Trên `k8s-master-01`:

```bash
K8S_VERSION=$(kubeadm version -o short)

sudo kubeadm init \
  --kubernetes-version=${K8S_VERSION} \
  --pod-network-cidr=192.168.0.0/16 \
  --apiserver-cert-extra-sans=k8s-api.k8s.local,10.10.10.11 \
  --control-plane-endpoint=k8s-api.k8s.local:6443 \
  --cri-socket=unix:///run/containerd/containerd.sock
```

Cấu hình kubeconfig:

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

Kiểm tra:

```bash
kubectl get nodes
kubectl get pods -n kube-system
```

---

## Bài lab 4: Cài Calico

Trên `k8s-master-01`:

```bash
curl https://raw.githubusercontent.com/projectcalico/calico/v3.31.4/manifests/calico.yaml -O
kubectl apply -f calico.yaml
```

Theo dõi:

```bash
kubectl get pods -n kube-system -w
```

Kiểm tra node:

```bash
kubectl get nodes -o wide
```

---

## Bài lab 5: Join worker node

Trên master, nếu cần tạo lại join command:

```bash
kubeadm token create --print-join-command
```

Trên từng worker node, chạy join command với `sudo`, ví dụ:

```bash
sudo kubeadm join k8s-api.k8s.local:6443 \
  --token <token> \
  --discovery-token-ca-cert-hash sha256:<hash> \
  --cri-socket=unix:///run/containerd/containerd.sock
```

Trên master, kiểm tra:

```bash
kubectl get nodes -o wide
```

---

## Bài lab 6: Kiểm tra Pod đơn lẻ

Tạo file:

```bash
cat > pod-nginx-basic.yaml <<'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: nginx-basic
  namespace: default
  labels:
    app: nginx-basic
spec:
  containers:
    - name: nginx
      image: nginx:1.27
      ports:
        - containerPort: 80
EOF
```

Apply:

```bash
kubectl apply -f pod-nginx-basic.yaml
```

Kiểm tra:

```bash
kubectl get pod nginx-basic -o wide
kubectl describe pod nginx-basic
kubectl exec nginx-basic -- nginx -v
```

Dọn dẹp:

```bash
kubectl delete -f pod-nginx-basic.yaml
```

---

# PHẦN 16 - CÂU HỎI KIỂM TRA CUỐI BUỔI

## Câu hỏi lý thuyết

### Câu 1

`kubeadm init` có trực tiếp chạy kube-apiserver không?

Gợi ý trả lời:

Không. kubeadm tạo static Pod manifest cho kube-apiserver. kubelet đọc manifest đó và gọi container runtime để chạy container kube-apiserver.

---

### Câu 2

Vì sao sau khi `kubeadm init` xong, node có thể ở trạng thái `NotReady`?

Gợi ý trả lời:

Vì chưa cài CNI plugin. Khi chưa có Pod network, node chưa hoàn tất điều kiện network và CoreDNS cũng chưa thể chạy ổn định.

---

### Câu 3

`--pod-network-cidr` dùng để làm gì?

Gợi ý trả lời:

Dùng để khai báo dải IP dành cho Pod network. Nếu được cấu hình, control plane có thể cấp Pod CIDR cho từng node và CNI có thể dựa vào đó để thiết lập mạng cho Pod.

---

### Câu 4

Vì sao nên dùng `--control-plane-endpoint` ngay cả khi chỉ có một master node?

Gợi ý trả lời:

Vì endpoint này tạo ra một địa chỉ ổn định cho Kubernetes API Server. Worker node và kubeconfig sẽ dùng endpoint này. Nếu sau này nâng cấp sang HA, có thể trỏ endpoint sang Load Balancer.

---

### Câu 5

`--apiserver-cert-extra-sans` giải quyết vấn đề gì?

Gợi ý trả lời:

Nó thêm DNS name hoặc IP vào certificate của API Server, giúp client truy cập API Server bằng endpoint tùy chỉnh mà không gặp lỗi TLS certificate.

---

### Câu 6

Vì sao cần tắt swap trong bài lab này?

Gợi ý trả lời:

Vì kubeadm/kubelet mặc định yêu cầu swap tắt trong mô hình triển khai thông thường. Nếu swap bật, kubeadm preflight có thể báo lỗi và dừng quá trình init/join.

---

### Câu 7

containerd có vai trò gì trong Kubernetes?

Gợi ý trả lời:

containerd là container runtime. kubelet giao tiếp với containerd thông qua CRI để tạo, chạy, dừng container thực tế.

---

### Câu 8

Calico có vai trò gì trong cluster?

Gợi ý trả lời:

Calico là CNI plugin. Nó cung cấp Pod network để các Pod có thể có IP và giao tiếp với nhau trong cluster.

---

## Câu hỏi thực hành

### Bài 1

Hãy kiểm tra cluster hiện tại đang dùng container runtime nào.

Lệnh gợi ý:

```bash
kubectl get nodes -o custom-columns=NAME:.metadata.name,RUNTIME:.status.nodeInfo.containerRuntimeVersion
```

---

### Bài 2

Hãy tìm các static Pod manifest của control plane.

Lệnh gợi ý:

```bash
sudo ls -l /etc/kubernetes/manifests
```

---

### Bài 3

Hãy kiểm tra endpoint API Server trong kubeconfig.

Lệnh gợi ý:

```bash
kubectl config view --minify
```

---

### Bài 4

Hãy tạo Pod nginx đơn lẻ và xác định Pod đó chạy trên worker node nào.

Lệnh gợi ý:

```bash
kubectl apply -f pod-nginx-basic.yaml
kubectl get pod nginx-basic -o wide
```

---

# PHẦN 17 - TÓM TẮT KIẾN THỨC BUỔI 02

Sau buổi 02,  cần nắm chắc các ý sau:

- kubeadm là công cụ bootstrap Kubernetes cluster.
- kubeadm init tạo control plane node.
- kubeadm join đưa worker node vào cluster.
- kubeadm tạo static Pod manifest cho control plane.
- kubelet là thành phần trực tiếp đọc static Pod manifest.
- containerd là runtime chạy container thực tế.
- Calico là CNI plugin cung cấp Pod network.
- Cluster có thể `NotReady` nếu chưa có CNI.
- `--pod-network-cidr` khai báo dải IP Pod.
- `--apiserver-cert-extra-sans` bổ sung DNS/IP vào certificate của API Server.
- `--control-plane-endpoint` tạo endpoint ổn định cho API Server.
- `--dry-run` giúp kiểm tra trước hành động của kubeadm.
- `--cri-socket` chỉ rõ container runtime socket.
- Không nên để Pod CIDR trùng với Node Network.
- Nên dùng endpoint dạng DNS ngay từ đầu để thuận lợi cho mở rộng HA sau này.

---

# PHẦN 18 - KIẾN THỨC ĐƯỢC MỞ KHÓA SAU BUỔI 02

Từ sau buổi này, các bài tiếp theo có thể sử dụng các khái niệm sau:

- kubeadm
- kubelet bootstrap
- kubeadm init
- kubeadm join
- containerd
- CRI socket
- CNI
- Calico
- Pod CIDR
- Service CIDR ở mức nhận diện
- Control Plane Endpoint
- API Server certificate SAN
- Static Pod manifest
- Node Ready / NotReady
- Pod IP
- Pod chạy trên node nào
- `kubectl get`
- `kubectl describe`
- `kubectl apply`
- `kubectl delete`
- `kubectl exec`

Các bài sau vẫn chưa nên mặc định dùng Deployment, Service, Volume, ConfigMap, Secret nếu chưa có buổi học riêng giới thiệu các chủ đề đó.

---

# PHỤ LỤC A - CHECKLIST TRIỂN KHAI NHANH

## Trên tất cả node

```bash
sudo swapoff -a
sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab

sudo tee /etc/modules-load.d/k8s.conf > /dev/null <<EOF
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter

sudo tee /etc/sysctl.d/k8s.conf > /dev/null <<EOF
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward = 1
EOF

sudo sysctl --system

sudo apt-get update
sudo apt-get install -y containerd

sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml > /dev/null
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml
sudo systemctl restart containerd
sudo systemctl enable containerd

sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl gpg

sudo mkdir -p -m 755 /etc/apt/keyrings

curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.35/deb/Release.key \
  | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.35/deb/ /' \
  | sudo tee /etc/apt/sources.list.d/kubernetes.list

sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```

---

## Trên master node

```bash
K8S_VERSION=$(kubeadm version -o short)

sudo kubeadm init \
  --kubernetes-version=${K8S_VERSION} \
  --pod-network-cidr=192.168.0.0/16 \
  --apiserver-cert-extra-sans=k8s-api.k8s.local,10.10.10.11 \
  --control-plane-endpoint=k8s-api.k8s.local:6443 \
  --cri-socket=unix:///run/containerd/containerd.sock
```

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

```bash
curl https://raw.githubusercontent.com/projectcalico/calico/v3.31.4/manifests/calico.yaml -O
kubectl apply -f calico.yaml
```

---

## Trên worker node

```bash
sudo kubeadm join k8s-api.k8s.local:6443 \
  --token <token> \
  --discovery-token-ca-cert-hash sha256:<hash> \
  --cri-socket=unix:///run/containerd/containerd.sock
```

---

## Kiểm tra cuối

```bash
kubectl get nodes -o wide
kubectl get pods -A -o wide
kubectl cluster-info
```

---

# PHỤ LỤC B - TÀI LIỆU THAM KHẢO

- Kubernetes v1.35 Documentation - Installing kubeadm  
  https://v1-35.docs.kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/

- Kubernetes v1.35 Documentation - Creating a cluster with kubeadm  
  https://v1-35.docs.kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/

- Kubernetes v1.35 Documentation - kubeadm init reference  
  https://v1-35.docs.kubernetes.io/docs/reference/setup-tools/kubeadm/kubeadm-init/

- Kubernetes Documentation - Container Runtimes  
  https://kubernetes.io/docs/setup/production-environment/container-runtimes/

- containerd CRI Plugin Configuration  
  https://containerd.io/docs/2.1/cri/config/

- Calico Documentation - Installing on on-premises deployments  
  https://docs.tigera.io/calico/latest/getting-started/kubernetes/self-managed-onprem/onpremises
