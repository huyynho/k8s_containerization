# Phần mở rộng: Triển khai cụm Kubernetes HA gồm 3 Master và 2 Worker bằng kubeadm

> **Phiên bản mục tiêu:** Kubernetes v1.35  
> **Hệ điều hành:** Ubuntu Server 24.04  
> **Container Runtime:** containerd  
> **CNI:** Calico  
> **Mô hình:** On-premise / Bare-metal / VM-based lab  
> **Cụm mở rộng:** 3 Control Plane Node + 2 Worker Node  
> **Phương pháp bootstrap:** kubeadm  
> **Vị trí trong giáo trình:** Phần mở rộng sau Buổi 12

---

## Mục tiêu của phần mở rộng

Sau khi hoàn thành phần này, có thể:

- Hiểu sự khác biệt giữa Kubernetes single control plane và HA control plane.
- Hiểu vì sao cụm 3 master cần `controlPlaneEndpoint`.
- Hiểu vai trò của Load Balancer hoặc VIP phía trước Kubernetes API Server.
- Hiểu mô hình stacked etcd và external etcd.
- Triển khai cụm Kubernetes HA gồm 3 master và 2 worker bằng kubeadm.
- Join thêm control plane node bằng `kubeadm join --control-plane`.
- Hiểu ý nghĩa của `--upload-certs` và `--certificate-key`.
- Cài đặt containerd, kubeadm, kubelet, kubectl trên Ubuntu 24.04.
- Cài đặt Calico CNI cho cụm HA.
- Kiểm tra trạng thái control plane, etcd quorum, worker node và workload sau triển khai.
- Thực hiện một số bài kiểm tra failover cơ bản cho Kubernetes API endpoint.
- Nhận diện các lỗi thường gặp khi triển khai kubeadm HA.

---

## Kiến thức đã học cần sử dụng lại

Phần mở rộng này sử dụng lại các kiến thức đã học ở các buổi trước:

- Buổi 01: Kiến trúc Kubernetes, Control Plane, Worker Node, kube-apiserver, etcd, scheduler, controller-manager, kubelet.
- Buổi 02: Bootstrap cluster bằng kubeadm, containerd, Calico, `--pod-network-cidr`, `--apiserver-cert-extra-sans`, `--control-plane-endpoint`.
- Buổi 03: Pod, Deployment và workload cơ bản.
- Buổi 04: Kubernetes Networking, Service, DNS nội bộ.
- Buổi 05: Storage và NFS Server trong mô hình lab.
- Buổi 07: Self-healing và vòng đời ứng dụng.
- Buổi 08: Scheduler và node placement.
- Buổi 10: Troubleshooting.
- Buổi 12: etcd backup/restore và disaster recovery.

---

## Phạm vi của phần mở rộng

Phần này tập trung vào triển khai cụm Kubernetes HA control plane bằng kubeadm.

Nội dung bao gồm:

- Thiết kế mô hình 3 master + 2 worker.
- Thiết kế API endpoint dùng VIP/load balancer.
- Cài đặt HAProxy và Keepalived cho môi trường lab on-premise.
- Bootstrap master đầu tiên.
- Join master thứ hai và thứ ba vào control plane.
- Join worker node.
- Cài Calico CNI.
- Kiểm tra cụm.
- Kiểm tra failover cơ bản.
- Troubleshooting.

Nội dung **không đi sâu** vào:

- External etcd cluster chạy trên node riêng.
- Multi-control-plane multi-region.
- Upgrade Kubernetes HA cluster.
- Hardening sâu cho production.
- Cloud Controller Manager trên AWS/Azure/GCP.
- Load Balancer cloud-native.

Các nội dung đó có thể tách thành các buổi nâng cao riêng.

---

# Phần 1. Từ single master đến HA control plane

## 1.1. Cụm single master ở Buổi 02

Ở Buổi 02, cụm Kubernetes được triển khai theo mô hình:

```text
+-------------------+
| master-01         |
| Control Plane     |
| etcd              |
| kube-apiserver    |
| scheduler         |
| controller-manager|
+---------+---------+
          |
          |
+---------+---------+       +-------------------+
| worker-01         |       | worker-02         |
| kubelet           |       | kubelet           |
| containerd        |       | containerd        |
+-------------------+       +-------------------+
```

Mô hình này phù hợp cho lab cơ bản.

Tuy nhiên, trong mô hình này, `master-01` là điểm lỗi rất quan trọng.

Nếu `master-01` gặp sự cố:

- Các Pod đang chạy trên worker node có thể vẫn tiếp tục chạy.
- Nhưng người quản trị không thể thao tác API Server.
- Scheduler không thể schedule Pod mới.
- Controller Manager không thể reconcile desired state.
- Không thể scale, rollout, rollback, apply manifest.
- Nếu etcd trên master bị mất dữ liệu mà không có backup, trạng thái cluster có thể mất.

Nói cách khác:

```text
Single master không có HA control plane.
```

---

## 1.2. Cụm HA control plane

Với cụm HA control plane, ta có nhiều control plane node.

Trong phần mở rộng này, mô hình sẽ là:

```text
+-------------------+       +-------------------+       +-------------------+
| master-01         |       | master-02         |       | master-03         |
| kube-apiserver    |       | kube-apiserver    |       | kube-apiserver    |
| etcd              |       | etcd              |       | etcd              |
| scheduler         |       | scheduler         |       | scheduler         |
| controller-manager|       | controller-manager|       | controller-manager|
+---------+---------+       +---------+---------+       +---------+---------+
          \                         |                         /
           \                        |                        /
            \                       |                       /
             +----------------------+----------------------+
                                    |
                         +----------+-----------+
                         | API VIP / LB         |
                         | k8s-api.k8s.local    |
                         +----------+-----------+
                                    |
                   +----------------+----------------+
                   |                                 |
          +--------+---------+              +--------+---------+
          | worker-01        |              | worker-02        |
          | kubelet          |              | kubelet          |
          | containerd       |              | containerd       |
          +------------------+              +------------------+
```

Điểm quan trọng nhất:

```text
Client, kubelet và các node không nên trỏ trực tiếp vào IP của một master cụ thể.
Tất cả nên đi qua một endpoint ổn định gọi là controlPlaneEndpoint.
```

Ví dụ:

```text
k8s-api.k8s.local:8443
```

hoặc trong production:

```text
k8s-api.k8s.local:6443
```

---

# Phần 2. Thành phần mới cần hiểu trong mô hình HA

## 2.1. controlPlaneEndpoint

`controlPlaneEndpoint` là endpoint ổn định đại diện cho Kubernetes API Server.

Trong cụm single master, thường thấy API Server là:

```text
https://master-01:6443
```

hoặc:

```text
https://10.10.10.11:6443
```

Nhưng trong cụm HA, nếu kubeconfig và kubelet đều trỏ vào `10.10.10.11`, khi `master-01` lỗi, toàn bộ cluster sẽ phụ thuộc vào node đó.

Vì vậy ta cần một endpoint trung gian:

```text
https://k8s-api.k8s.local:8443
```

Endpoint này sẽ trỏ đến VIP hoặc load balancer.

Load balancer sẽ phân phối request đến các kube-apiserver đang khỏe mạnh:

```text
k8s-api.k8s.local:8443
        |
        +--> master-01:6443
        +--> master-02:6443
        +--> master-03:6443
```

Trong kubeadm, endpoint này được khai báo bằng flag:

```bash
--control-plane-endpoint "k8s-api.k8s.local:8443"
```

Hoặc khai báo trong kubeadm config file:

```yaml
controlPlaneEndpoint: "k8s-api.k8s.local:8443"
```

---

## 2.2. Vì sao lab này dùng port 8443 cho controlPlaneEndpoint?

Trong production, nếu dùng load balancer bên ngoài cụm, ta thường dùng:

```text
k8s-api.k8s.local:6443
```

Ví dụ:

```text
External Load Balancer: 10.10.10.10:6443
Backend:
  master-01:6443
  master-02:6443
  master-03:6443
```

Tuy nhiên, trong lab này ta giả định chỉ có 5 VM:

- 3 master
- 2 worker

Không có thêm VM riêng cho load balancer.

Vì vậy ta có thể chạy HAProxy + Keepalived trực tiếp trên 3 master node.

Vấn đề là kube-apiserver trên mỗi master đã dùng port `6443`.

Nếu HAProxy cũng bind port `6443` trên chính master node, có thể bị xung đột port.

Do đó trong lab này ta dùng:

```text
VIP: 10.10.10.10
Frontend HAProxy: 10.10.10.10:8443
Backend kube-apiserver: master-01:6443, master-02:6443, master-03:6443
controlPlaneEndpoint: k8s-api.k8s.local:8443
```

Mô hình:

```text
Client / kubelet
      |
      v
k8s-api.k8s.local:8443
      |
      v
HAProxy trên node đang giữ VIP
      |
      +--> master-01:6443
      +--> master-02:6443
      +--> master-03:6443
```

Đây là mô hình phù hợp cho lab.

Trong production, nên ưu tiên load balancer riêng, ví dụ:

- HAProxy/Keepalived trên 2 VM riêng.
- F5.
- Nginx stream.
- LVS.
- Load balancer phần cứng.
- Load balancer của hạ tầng ảo hóa/private cloud.

---

## 2.3. Stacked etcd và external etcd

Kubernetes HA với kubeadm có hai kiểu topology phổ biến.

### Stacked etcd

Trong mô hình stacked etcd, mỗi control plane node chạy một instance etcd local.

Ví dụ:

```text
master-01: kube-apiserver + scheduler + controller-manager + etcd
master-02: kube-apiserver + scheduler + controller-manager + etcd
master-03: kube-apiserver + scheduler + controller-manager + etcd
```

Ưu điểm:

- Ít node hơn.
- Dễ triển khai hơn.
- Phù hợp lab, staging, cụm nhỏ và vừa.
- Được kubeadm hỗ trợ thuận tiện.

Nhược điểm:

- Control plane node và etcd cùng nằm trên một node.
- Nếu control plane node gặp lỗi, đồng thời mất luôn một etcd member.
- Cần đặc biệt chú ý backup etcd.

Phần mở rộng này sử dụng mô hình:

```text
Stacked etcd
```

---

### External etcd

Trong mô hình external etcd, etcd chạy trên các node riêng, tách khỏi control plane.

Ví dụ:

```text
Control Plane:
  master-01
  master-02
  master-03

External etcd:
  etcd-01
  etcd-02
  etcd-03
```

Ưu điểm:

- Tách biệt database của cluster khỏi control plane node.
- Có thể thiết kế backup/restore riêng cho etcd.
- Phù hợp production lớn hơn.

Nhược điểm:

- Cần nhiều node hơn.
- Cấu hình phức tạp hơn.
- Tốn tài nguyên hơn.
- Cần vận hành etcd cluster độc lập.

Trong khóa học này, external etcd nên để thành bài nâng cao riêng.

---

## 2.4. etcd quorum trong cụm 3 master

Vì phần này dùng stacked etcd, mỗi master sẽ là một etcd member.

Với 3 etcd member:

```text
master-01 etcd
master-02 etcd
master-03 etcd
```

etcd cần quorum để hoạt động.

Công thức quorum đơn giản:

```text
quorum = floor(n / 2) + 1
```

Với `n = 3`:

```text
quorum = floor(3 / 2) + 1 = 2
```

Nghĩa là cụm etcd 3 node cần ít nhất 2 node còn hoạt động.

Khả năng chịu lỗi:

```text
3 etcd members chịu được lỗi 1 member.
```

Nếu mất 1 master:

```text
Còn 2/3 etcd member -> còn quorum -> cluster vẫn hoạt động.
```

Nếu mất 2 master:

```text
Còn 1/3 etcd member -> mất quorum -> Kubernetes API có thể không hoạt động đúng.
```

Đây là lý do production phải backup etcd cẩn thận.

---

# Phần 3. Mô hình lab đề xuất

## 3.1. Danh sách VM

Mô hình lab sử dụng 5 VM:

| Node | Vai trò | Hostname | IP |
|---|---|---|---|
| 1 | Master / Control Plane | master-01 | 10.10.10.11 |
| 2 | Master / Control Plane | master-02 | 10.10.10.12 |
| 3 | Master / Control Plane | master-03 | 10.10.10.13 |
| 4 | Worker | worker-01 | 10.10.10.21 |
| 5 | Worker | worker-02 | 10.10.10.22 |

Thông tin bổ sung:

| Thành phần | Giá trị |
|---|---|
| VIP Kubernetes API | 10.10.10.10 |
| DNS API Endpoint | k8s-api.k8s.local |
| API Endpoint trong lab | k8s-api.k8s.local:8443 |
| Backend API Server | master-01/02/03:6443 |
| Pod CIDR | 192.168.0.0/16 |
| Service CIDR | 10.96.0.0/12 |
| CNI | Calico |
| Container Runtime | containerd |
| Kubernetes Version | v1.35.x |

> Lưu ý: IP, hostname, interface có thể thay đổi theo môi trường của giảng viên. Trong bài lab này giả định interface chính là `ens33`. Nếu môi trường dùng `ens160`, `ens192`, `eth0`, cần thay lại trong cấu hình Keepalived.

---

## 3.2. Sơ đồ logic

```text
                    +--------------------------------+
                    | DNS                            |
                    | k8s-api.k8s.local -> 10.10.10.10|
                    +---------------+----------------+
                                    |
                                    v
                    +--------------------------------+
                    | VIP 10.10.10.10:8443          |
                    | HAProxy + Keepalived          |
                    +---------------+----------------+
                                    |
             +----------------------+----------------------+
             |                      |                      |
             v                      v                      v
+------------------------+ +------------------------+ +------------------------+
| master-01              | | master-02              | | master-03              |
| 10.10.10.11            | | 10.10.10.12            | | 10.10.10.13            |
| kube-apiserver:6443    | | kube-apiserver:6443    | | kube-apiserver:6443    |
| etcd                   | | etcd                   | | etcd                   |
| scheduler              | | scheduler              | | scheduler              |
| controller-manager     | | controller-manager     | | controller-manager     |
+-----------+------------+ +-----------+------------+ +-----------+------------+
            \                          |                          /
             \                         |                         /
              \                        |                        /
               +-----------------------+-----------------------+
                                       |
                      +----------------+----------------+
                      |                                 |
                      v                                 v
             +------------------+              +------------------+
             | worker-01        |              | worker-02        |
             | 10.10.10.21      |              | 10.10.10.22      |
             +------------------+              +------------------+
```

---

# Phần 4. Chuẩn bị DNS và host mapping

## 4.1. Cấu hình DNS nội bộ

Trong môi trường tốt nhất, nên tạo bản ghi DNS:

```text
k8s-api.k8s.local    A    10.10.10.10
```

Các node phải resolve được:

```bash
nslookup k8s-api.k8s.local
```

Kết quả kỳ vọng:

```text
Name:   k8s-api.k8s.local
Address: 10.10.10.10
```

---

## 4.2. Nếu chưa có DNS, dùng `/etc/hosts`

Trên tất cả 5 node, thêm nội dung sau:

```bash
sudo tee -a /etc/hosts <<EOF
10.10.10.10 k8s-api.k8s.local k8s-api
10.10.10.11 master-01.k8s.local master-01
10.10.10.12 master-02.k8s.local master-02
10.10.10.13 master-03.k8s.local master-03
10.10.10.21 worker-01.k8s.local worker-01
10.10.10.22 worker-02.k8s.local worker-02
EOF
```

Kiểm tra:

```bash
getent hosts k8s-api.k8s.local
getent hosts master-01
getent hosts master-02
getent hosts master-03
```

Kết quả kỳ vọng:

```text
10.10.10.10 k8s-api.k8s.local k8s-api
10.10.10.11 master-01.k8s.local master-01
10.10.10.12 master-02.k8s.local master-02
10.10.10.13 master-03.k8s.local master-03
```

---

# Phần 5. Chuẩn bị hệ điều hành trên tất cả node

Các bước trong phần này chạy trên cả 5 node:

- `master-01`
- `master-02`
- `master-03`
- `worker-01`
- `worker-02`

---

## 5.1. Đặt hostname

Trên `master-01`:

```bash
sudo hostnamectl set-hostname master-01
```

Trên `master-02`:

```bash
sudo hostnamectl set-hostname master-02
```

Trên `master-03`:

```bash
sudo hostnamectl set-hostname master-03
```

Trên `worker-01`:

```bash
sudo hostnamectl set-hostname worker-01
```

Trên `worker-02`:

```bash
sudo hostnamectl set-hostname worker-02
```

Kiểm tra:

```bash
hostnamectl
```

---

## 5.2. Tắt swap

Kubernetes yêu cầu swap phải được tắt, trừ khi cluster được cấu hình rõ ràng để hỗ trợ swap.

Trong lab này, tắt swap hoàn toàn:

```bash
sudo swapoff -a
```

Comment dòng swap trong `/etc/fstab`:

```bash
sudo sed -i '/ swap / s/^/#/' /etc/fstab
```

Kiểm tra:

```bash
free -h
```

Kỳ vọng:

```text
Swap: 0B
```

---

## 5.3. Load kernel modules cần thiết

```bash
sudo tee /etc/modules-load.d/k8s.conf <<EOF
overlay
br_netfilter
EOF
```

Load ngay lập tức:

```bash
sudo modprobe overlay
sudo modprobe br_netfilter
```

Kiểm tra:

```bash
lsmod | grep overlay
lsmod | grep br_netfilter
```

---

## 5.4. Cấu hình sysctl cho Kubernetes networking

```bash
sudo tee /etc/sysctl.d/k8s.conf <<EOF
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF
```

Apply:

```bash
sudo sysctl --system
```

Kiểm tra:

```bash
sysctl net.bridge.bridge-nf-call-iptables
sysctl net.bridge.bridge-nf-call-ip6tables
sysctl net.ipv4.ip_forward
```

Kỳ vọng:

```text
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward = 1
```

---

# Phần 6. Cài đặt containerd trên tất cả node

## 6.1. Cài containerd

```bash
sudo apt update
sudo apt install -y containerd
```

Tạo file cấu hình mặc định:

```bash
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml > /dev/null
```

---

## 6.2. Cấu hình systemd cgroup driver

Kubernetes kubelet và container runtime nên dùng cùng cgroup driver.

Với Ubuntu 24.04 và systemd, cấu hình containerd dùng systemd cgroup:

```bash
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml
```

Kiểm tra:

```bash
grep SystemdCgroup /etc/containerd/config.toml
```

Kết quả kỳ vọng:

```text
SystemdCgroup = true
```

Restart containerd:

```bash
sudo systemctl restart containerd
sudo systemctl enable containerd
```

Kiểm tra:

```bash
sudo systemctl status containerd --no-pager
```

---

# Phần 7. Cài kubeadm, kubelet, kubectl trên tất cả node

## 7.1. Cài các gói phụ thuộc

```bash
sudo apt update
sudo apt install -y apt-transport-https ca-certificates curl gpg
```

---

## 7.2. Thêm repository Kubernetes v1.35

```bash
sudo mkdir -p /etc/apt/keyrings
```

```bash
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.35/deb/Release.key \
  | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
```

```bash
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.35/deb/ /' \
  | sudo tee /etc/apt/sources.list.d/kubernetes.list
```

Cập nhật package index:

```bash
sudo apt update
```

---

## 7.3. Cài kubelet, kubeadm, kubectl

```bash
sudo apt install -y kubelet kubeadm kubectl
```

Giữ version để tránh tự động upgrade ngoài ý muốn:

```bash
sudo apt-mark hold kubelet kubeadm kubectl
```

Kiểm tra:

```bash
kubeadm version
kubectl version --client
kubelet --version
```

---

# Phần 8. Cài HAProxy và Keepalived trên 3 master node

Phần này chỉ chạy trên:

- `master-01`
- `master-02`
- `master-03`

---

## 8.1. Cài HAProxy và Keepalived

```bash
sudo apt update
sudo apt install -y haproxy keepalived
```

---

## 8.2. Cho phép HAProxy bind địa chỉ VIP khi VIP chưa nằm trên node

Trên cả 3 master:

```bash
echo "net.ipv4.ip_nonlocal_bind = 1" | sudo tee /etc/sysctl.d/99-haproxy-nonlocal-bind.conf
sudo sysctl --system
```

Kiểm tra:

```bash
sysctl net.ipv4.ip_nonlocal_bind
```

Kết quả kỳ vọng:

```text
net.ipv4.ip_nonlocal_bind = 1
```

---

## 8.3. Cấu hình HAProxy

Trên cả 3 master, backup file cấu hình cũ:

```bash
sudo cp /etc/haproxy/haproxy.cfg /etc/haproxy/haproxy.cfg.bak
```

Tạo cấu hình mới:

```bash
sudo tee /etc/haproxy/haproxy.cfg <<'EOF'
global
    log /dev/log local0
    log /dev/log local1 notice
    daemon
    maxconn 4096

defaults
    log global
    mode tcp
    option tcplog
    option dontlognull
    timeout connect 5s
    timeout client  50s
    timeout server  50s

frontend kubernetes_api_frontend
    bind *:8443
    mode tcp
    option tcplog
    default_backend kubernetes_api_backend

backend kubernetes_api_backend
    mode tcp
    option tcp-check
    balance roundrobin
    server master-01 10.10.10.11:6443 check fall 3 rise 2
    server master-02 10.10.10.12:6443 check fall 3 rise 2
    server master-03 10.10.10.13:6443 check fall 3 rise 2
EOF
```

Restart HAProxy:

```bash
sudo systemctl restart haproxy
sudo systemctl enable haproxy
```

Kiểm tra:

```bash
sudo systemctl status haproxy --no-pager
```

Kiểm tra port:

```bash
sudo ss -lntp | grep 8443
```

Kết quả kỳ vọng:

```text
LISTEN 0 ... *:8443 ...
```

---

## 8.4. Giải thích cấu hình HAProxy

Đoạn cấu hình quan trọng:

```text
frontend kubernetes_api_frontend
    bind *:8443
    mode tcp
    default_backend kubernetes_api_backend
```

Ý nghĩa:

- HAProxy lắng nghe port `8443`.
- Dùng TCP mode vì Kubernetes API Server sử dụng HTTPS.
- HAProxy không cần terminate TLS.
- Request được chuyển sang backend `kubernetes_api_backend`.

Đoạn backend:

```text
backend kubernetes_api_backend
    mode tcp
    option tcp-check
    balance roundrobin
    server master-01 10.10.10.11:6443 check fall 3 rise 2
    server master-02 10.10.10.12:6443 check fall 3 rise 2
    server master-03 10.10.10.13:6443 check fall 3 rise 2
```

Ý nghĩa:

- Backend là các kube-apiserver chạy trên 3 master.
- Mỗi kube-apiserver vẫn chạy port mặc định `6443`.
- HAProxy kiểm tra trạng thái backend bằng TCP check.
- Nếu backend down, HAProxy không forward request đến backend đó.

---

## 8.5. Cấu hình Keepalived trên master-01

Xác định interface mạng chính:

```bash
ip -br addr
```

Giả sử interface là:

```text
ens33
```

Tạo cấu hình Keepalived trên `master-01`:

```bash
sudo tee /etc/keepalived/keepalived.conf <<'EOF'
vrrp_script chk_haproxy {
    script "pidof haproxy"
    interval 2
    weight -20
}

vrrp_instance VI_1 {
    state MASTER
    interface ens33
    virtual_router_id 51
    priority 110
    advert_int 1

    authentication {
        auth_type PASS
        auth_pass k8sHApass
    }

    virtual_ipaddress {
        10.10.10.10/24
    }

    track_script {
        chk_haproxy
    }
}
EOF
```

Restart Keepalived:

```bash
sudo systemctl restart keepalived
sudo systemctl enable keepalived
```

Kiểm tra:

```bash
sudo systemctl status keepalived --no-pager
```

Kiểm tra VIP:

```bash
ip addr show ens33 | grep 10.10.10.10
```

Trên `master-01`, kỳ vọng thấy VIP:

```text
inet 10.10.10.10/24 ...
```

---

## 8.6. Cấu hình Keepalived trên master-02

Trên `master-02`:

```bash
sudo tee /etc/keepalived/keepalived.conf <<'EOF'
vrrp_script chk_haproxy {
    script "pidof haproxy"
    interval 2
    weight -20
}

vrrp_instance VI_1 {
    state BACKUP
    interface ens33
    virtual_router_id 51
    priority 105
    advert_int 1

    authentication {
        auth_type PASS
        auth_pass k8sHApass
    }

    virtual_ipaddress {
        10.10.10.10/24
    }

    track_script {
        chk_haproxy
    }
}
EOF
```

Restart:

```bash
sudo systemctl restart keepalived
sudo systemctl enable keepalived
```

---

## 8.7. Cấu hình Keepalived trên master-03

Trên `master-03`:

```bash
sudo tee /etc/keepalived/keepalived.conf <<'EOF'
vrrp_script chk_haproxy {
    script "pidof haproxy"
    interval 2
    weight -20
}

vrrp_instance VI_1 {
    state BACKUP
    interface ens33
    virtual_router_id 51
    priority 100
    advert_int 1

    authentication {
        auth_type PASS
        auth_pass k8sHApass
    }

    virtual_ipaddress {
        10.10.10.10/24
    }

    track_script {
        chk_haproxy
    }
}
EOF
```

Restart:

```bash
sudo systemctl restart keepalived
sudo systemctl enable keepalived
```

---

## 8.8. Kiểm tra VIP từ tất cả node

Trên tất cả node:

```bash
ping -c 4 10.10.10.10
```

Kiểm tra resolve DNS:

```bash
getent hosts k8s-api.k8s.local
```

Kiểm tra port HAProxy:

```bash
nc -vz k8s-api.k8s.local 8443
```

Ở thời điểm này, trước khi `kubeadm init`, backend kube-apiserver chưa chạy, nên có thể port vẫn mở nhưng chưa có API Server phản hồi đúng.

Điều quan trọng là:

- VIP phải ping được.
- DNS phải trỏ đúng VIP.
- HAProxy service phải chạy.
- Không bị conflict port `6443`.

---

# Phần 9. Bootstrap master đầu tiên

Phần này chỉ chạy trên:

```text
master-01
```

---

## 9.1. Kiểm tra trước khi init

Kiểm tra containerd:

```bash
sudo systemctl status containerd --no-pager
```

Kiểm tra kubelet:

```bash
sudo systemctl status kubelet --no-pager
```

Lưu ý: trước khi init, kubelet có thể ở trạng thái chưa fully running. Đây là bình thường vì kubelet chưa có cấu hình cluster.

Kiểm tra swap:

```bash
free -h
```

Kiểm tra endpoint:

```bash
getent hosts k8s-api.k8s.local
```

Kiểm tra HAProxy:

```bash
sudo systemctl status haproxy --no-pager
```

Kiểm tra Keepalived:

```bash
sudo systemctl status keepalived --no-pager
```

---

## 9.2. Chạy dry-run trước khi init thật

Trước khi bootstrap thật, nên dùng `--dry-run` để kiểm tra logic cấu hình.

```bash
sudo kubeadm init \
  --kubernetes-version=v1.35.0 \
  --control-plane-endpoint "k8s-api.k8s.local:8443" \
  --apiserver-cert-extra-sans "k8s-api.k8s.local,10.10.10.10,10.10.10.11,10.10.10.12,10.10.10.13" \
  --pod-network-cidr "192.168.0.0/16" \
  --cri-socket "unix:///run/containerd/containerd.sock" \
  --upload-certs \
  --dry-run
```

Ý nghĩa:

- `--dry-run`: mô phỏng quá trình init, không thay đổi hệ thống thật.
- Dùng để phát hiện lỗi trước khi bootstrap.
- Rất hữu ích khi triển khai production.

Nếu dry-run không có lỗi nghiêm trọng, tiếp tục init thật.

---

## 9.3. Init master-01

```bash
sudo kubeadm init \
  --kubernetes-version=v1.35.0 \
  --control-plane-endpoint "k8s-api.k8s.local:8443" \
  --apiserver-cert-extra-sans "k8s-api.k8s.local,10.10.10.10,10.10.10.11,10.10.10.12,10.10.10.13" \
  --pod-network-cidr "192.168.0.0/16" \
  --cri-socket "unix:///run/containerd/containerd.sock" \
  --upload-certs
```

Sau khi chạy thành công, kubeadm sẽ in ra hai nhóm lệnh join:

- Lệnh join control plane node.
- Lệnh join worker node.

Ví dụ:

```text
You can now join any number of the control-plane node running the following command on each as root:

  kubeadm join k8s-api.k8s.local:8443 --token <token> \
    --discovery-token-ca-cert-hash sha256:<hash> \
    --control-plane --certificate-key <certificate-key>

Then you can join any number of worker nodes by running the following on each as root:

  kubeadm join k8s-api.k8s.local:8443 --token <token> \
    --discovery-token-ca-cert-hash sha256:<hash>
```

Cần lưu lại:

- Token.
- Discovery token CA cert hash.
- Certificate key.

---

## 9.4. Giải thích các flag quan trọng

### `--control-plane-endpoint`

```bash
--control-plane-endpoint "k8s-api.k8s.local:8443"
```

Ý nghĩa:

- Khai báo endpoint ổn định của Kubernetes API Server.
- Kubeconfig sẽ dùng endpoint này.
- Worker node sẽ join qua endpoint này.
- Các control plane node sau cũng join qua endpoint này.
- Đây là flag cực kỳ quan trọng trong mô hình HA.

Nếu không dùng flag này, cluster có thể bị gắn chặt vào IP/hostname của `master-01`.

---

### `--apiserver-cert-extra-sans`

```bash
--apiserver-cert-extra-sans "k8s-api.k8s.local,10.10.10.10,10.10.10.11,10.10.10.12,10.10.10.13"
```

Ý nghĩa:

- Thêm DNS/IP vào Subject Alternative Names của certificate API Server.
- Client truy cập API Server qua tên hoặc IP nào thì tên/IP đó nên có trong SAN.
- Nếu thiếu SAN, có thể gặp lỗi x509.

Ví dụ lỗi thường gặp:

```text
x509: certificate is valid for ..., not k8s-api.k8s.local
```

Trong lab này cần có:

- `k8s-api.k8s.local`
- `10.10.10.10`
- IP các master node

---

### `--pod-network-cidr`

```bash
--pod-network-cidr "192.168.0.0/16"
```

Ý nghĩa:

- Khai báo dải IP dành cho Pod.
- Phải khớp với cấu hình CNI.
- Với Calico, dải thường dùng trong lab là `192.168.0.0/16`.

Nếu Pod CIDR không khớp CNI, Pod có thể không giao tiếp đúng.

---

### `--cri-socket`

```bash
--cri-socket "unix:///run/containerd/containerd.sock"
```

Ý nghĩa:

- Chỉ định container runtime endpoint.
- Trong bài này dùng containerd.
- Giúp kubeadm biết kubelet sẽ nói chuyện với runtime nào.

---

### `--upload-certs`

```bash
--upload-certs
```

Ý nghĩa:

- Upload control-plane certificates lên cluster dưới dạng Secret tạm thời.
- Secret này được mã hóa bằng certificate key.
- Certificate key dùng khi join thêm control plane node.
- Secret này có thời hạn ngắn, thường dùng trong quá trình bootstrap ban đầu.

Nếu certificate key hết hạn hoặc mất, có thể tạo lại bằng lệnh:

```bash
sudo kubeadm init phase upload-certs --upload-certs
```

---

## 9.5. Cấu hình kubectl trên master-01

Chạy trên `master-01`:

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

Kiểm tra:

```bash
kubectl get nodes
```

Ở thời điểm này có thể thấy:

```text
NAME        STATUS     ROLES           AGE   VERSION
master-01   NotReady   control-plane   ...   v1.35.x
```

`NotReady` là bình thường vì chưa cài CNI.

---

# Phần 10. Cài Calico CNI

Phần này chạy trên `master-01`.

## 10.1. Cài Calico operator

Ví dụ dùng manifest chính thức của Calico:

```bash
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.31.0/manifests/tigera-operator.yaml
```

Tải custom resources:

```bash
curl -O https://raw.githubusercontent.com/projectcalico/calico/v3.31.0/manifests/custom-resources.yaml
```

Kiểm tra CIDR trong file:

```bash
grep -n "cidr" custom-resources.yaml
```

Nếu cần, chỉnh về:

```yaml
cidr: 192.168.0.0/16
```

Apply:

```bash
kubectl create -f custom-resources.yaml
```

---

## 10.2. Theo dõi Calico

```bash
watch kubectl get pods -n calico-system
```

Kiểm tra node:

```bash
kubectl get nodes
```

Kỳ vọng sau một thời gian:

```text
NAME        STATUS   ROLES           AGE   VERSION
master-01   Ready    control-plane   ...   v1.35.x
```

---

# Phần 11. Join master-02 và master-03 vào control plane

Phần này chạy trên:

- `master-02`
- `master-03`

---

## 11.1. Dùng lệnh join control plane từ kubeadm init

Trên `master-02`, chạy lệnh dạng:

```bash
sudo kubeadm join k8s-api.k8s.local:8443 \
  --token <token> \
  --discovery-token-ca-cert-hash sha256:<hash> \
  --control-plane \
  --certificate-key <certificate-key>
```

Trên `master-03`, chạy tương tự:

```bash
sudo kubeadm join k8s-api.k8s.local:8443 \
  --token <token> \
  --discovery-token-ca-cert-hash sha256:<hash> \
  --control-plane \
  --certificate-key <certificate-key>
```

---

## 11.2. Nếu token hết hạn

Token mặc định có thời hạn.

Nếu token hết hạn, tạo token mới trên `master-01`:

```bash
kubeadm token create --print-join-command
```

Lệnh này tạo join command cho worker.

Để join control plane, cần thêm:

```bash
--control-plane --certificate-key <certificate-key>
```

---

## 11.3. Nếu certificate key hết hạn hoặc bị mất

Trên `master-01`, tạo lại certificate key:

```bash
sudo kubeadm init phase upload-certs --upload-certs
```

Kết quả sẽ có certificate key mới:

```text
Using certificate key:
<new-certificate-key>
```

Sau đó dùng certificate key này để join control plane node.

---

## 11.4. Cấu hình kubectl trên master-02 và master-03

Sau khi join thành công, trên mỗi master mới:

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

Kiểm tra:

```bash
kubectl get nodes
```

Kỳ vọng:

```text
NAME        STATUS   ROLES           AGE   VERSION
master-01   Ready    control-plane   ...   v1.35.x
master-02   Ready    control-plane   ...   v1.35.x
master-03   Ready    control-plane   ...   v1.35.x
```

---

# Phần 12. Join worker-01 và worker-02

Phần này chạy trên:

- `worker-01`
- `worker-02`

Dùng lệnh join worker từ kubeadm init hoặc tạo lại bằng:

```bash
kubeadm token create --print-join-command
```

Ví dụ:

```bash
sudo kubeadm join k8s-api.k8s.local:8443 \
  --token <token> \
  --discovery-token-ca-cert-hash sha256:<hash>
```

Sau khi join, trên master kiểm tra:

```bash
kubectl get nodes -o wide
```

Kết quả kỳ vọng:

```text
NAME        STATUS   ROLES           AGE   VERSION   INTERNAL-IP
master-01   Ready    control-plane   ...   v1.35.x   10.10.10.11
master-02   Ready    control-plane   ...   v1.35.x   10.10.10.12
master-03   Ready    control-plane   ...   v1.35.x   10.10.10.13
worker-01   Ready    <none>          ...   v1.35.x   10.10.10.21
worker-02   Ready    <none>          ...   v1.35.x   10.10.10.22
```

---

# Phần 13. Kiểm tra trạng thái control plane

## 13.1. Kiểm tra static pod control plane

```bash
kubectl get pods -n kube-system -o wide
```

Cần thấy các Pod tương tự:

```text
etcd-master-01
etcd-master-02
etcd-master-03
kube-apiserver-master-01
kube-apiserver-master-02
kube-apiserver-master-03
kube-controller-manager-master-01
kube-controller-manager-master-02
kube-controller-manager-master-03
kube-scheduler-master-01
kube-scheduler-master-02
kube-scheduler-master-03
```

Lưu ý:

- Có 3 kube-apiserver.
- Có 3 scheduler.
- Có 3 controller-manager.
- Có 3 etcd.
- Nhưng scheduler và controller-manager hoạt động theo cơ chế leader election.
- Tại một thời điểm, chỉ có một leader chính xử lý nhiệm vụ.

---

## 13.2. Kiểm tra API endpoint

Từ một node bất kỳ:

```bash
curl -k https://k8s-api.k8s.local:8443/readyz
```

Kết quả kỳ vọng:

```text
ok
```

Kiểm tra chi tiết:

```bash
curl -k https://k8s-api.k8s.local:8443/readyz?verbose
```

Kết quả sẽ liệt kê nhiều health check của API Server.

---

## 13.3. Kiểm tra kubeconfig đang trỏ về endpoint HA

```bash
kubectl config view --minify
```

Tìm dòng:

```yaml
server: https://k8s-api.k8s.local:8443
```

Nếu kubeconfig đang trỏ về:

```text
https://10.10.10.11:6443
```

thì chưa đúng tinh thần HA endpoint.

---

## 13.4. Kiểm tra kubeadm config

```bash
kubectl -n kube-system get configmap kubeadm-config -o yaml
```

Tìm:

```yaml
controlPlaneEndpoint: k8s-api.k8s.local:8443
```

---

# Phần 14. Kiểm tra etcd cluster

## 14.1. Kiểm tra etcd Pod

```bash
kubectl get pods -n kube-system -l component=etcd -o wide
```

Kết quả kỳ vọng:

```text
NAME             READY   STATUS    NODE
etcd-master-01   1/1     Running   master-01
etcd-master-02   1/1     Running   master-02
etcd-master-03   1/1     Running   master-03
```

---

## 14.2. Kiểm tra etcd endpoint status

Chạy trên một master node, ví dụ `master-01`.

Trước hết kiểm tra container etcd:

```bash
sudo crictl ps | grep etcd
```

Dùng `etcdctl` bên trong container etcd.

Lấy container ID:

```bash
ETCD_CONTAINER=$(sudo crictl ps --name etcd -q | head -n 1)
echo $ETCD_CONTAINER
```

Chạy etcdctl:

```bash
sudo crictl exec -it $ETCD_CONTAINER etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  endpoint status --write-out=table
```

Kết quả kỳ vọng có thông tin:

- Endpoint.
- ID.
- Version.
- DB Size.
- Is Leader.
- Raft Term.
- Raft Index.

Ví dụ:

```text
+------------------------+------------------+---------+---------+-----------+------------+
|        ENDPOINT        |        ID        | VERSION | DB SIZE | IS LEADER | RAFT TERM  |
+------------------------+------------------+---------+---------+-----------+------------+
| https://127.0.0.1:2379 | ...              | 3.x     | ...     | true/false| ...        |
+------------------------+------------------+---------+---------+-----------+------------+
```

---

## 14.3. Kiểm tra danh sách etcd member

```bash
sudo crictl exec -it $ETCD_CONTAINER etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  member list --write-out=table
```

Kỳ vọng có 3 member tương ứng 3 master.

---

# Phần 15. Triển khai workload kiểm thử

## 15.1. Tạo namespace

```bash
kubectl create namespace ha-demo
```

---

## 15.2. Tạo Deployment

Tạo file:

```bash
cat <<'EOF' > ha-demo-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ha-web
  namespace: ha-demo
spec:
  replicas: 4
  selector:
    matchLabels:
      app: ha-web
  template:
    metadata:
      labels:
        app: ha-web
    spec:
      containers:
      - name: nginx
        image: nginx:1.29
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

Apply:

```bash
kubectl apply -f ha-demo-deployment.yaml
```

---

## 15.3. Giải thích manifest

Đoạn:

```yaml
kind: Deployment
```

Ý nghĩa:

- Tạo workload dạng Deployment.
- Deployment quản lý ReplicaSet và Pod.
- Kiến thức này đã học ở Buổi 03.

Đoạn:

```yaml
replicas: 4
```

Ý nghĩa:

- Yêu cầu Kubernetes duy trì 4 Pod.
- Nếu Pod bị lỗi, Deployment/ReplicaSet sẽ tạo lại Pod khác.
- Đây là self-healing đã học ở Buổi 07.

Đoạn:

```yaml
resources:
  requests:
    cpu: "50m"
    memory: "64Mi"
  limits:
    cpu: "200m"
    memory: "128Mi"
```

Ý nghĩa:

- Cung cấp thông tin tài nguyên cho scheduler.
- Scheduler dựa vào `requests` để chọn node phù hợp.
- Kiến thức này đã học ở Buổi 08.

---

## 15.4. Kiểm tra Pod phân bố trên worker

```bash
kubectl get pods -n ha-demo -o wide
```

Kỳ vọng Pod được schedule lên worker node.

Nếu Pod chạy trên master node, kiểm tra taint của control-plane node:

```bash
kubectl describe node master-01 | grep -i taint -A2
```

Mặc định control-plane node thường có taint để không chạy workload thường.

---

## 15.5. Tạo Service NodePort kiểm thử

Tạo file:

```bash
cat <<'EOF' > ha-demo-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: ha-web-nodeport
  namespace: ha-demo
spec:
  type: NodePort
  selector:
    app: ha-web
  ports:
  - name: http
    port: 80
    targetPort: 80
    nodePort: 30080
EOF
```

Apply:

```bash
kubectl apply -f ha-demo-service.yaml
```

Kiểm tra:

```bash
kubectl get svc -n ha-demo
```

Truy cập thử từ bên ngoài:

```bash
curl http://10.10.10.21:30080
curl http://10.10.10.22:30080
```

---

# Phần 16. Kiểm tra failover API endpoint

## 16.1. Xác định master đang giữ VIP

Trên từng master:

```bash
ip addr show ens33 | grep 10.10.10.10
```

Node nào có VIP là node đang giữ vai trò MASTER của Keepalived.

Giả sử VIP đang nằm trên `master-01`.

---

## 16.2. Kiểm tra API trước failover

Từ worker hoặc máy quản trị:

```bash
curl -k https://k8s-api.k8s.local:8443/readyz
```

Kỳ vọng:

```text
ok
```

Kiểm tra kubectl:

```bash
kubectl get nodes
```

---

## 16.3. Mô phỏng HAProxy lỗi trên node giữ VIP

Trên node đang giữ VIP, ví dụ `master-01`:

```bash
sudo systemctl stop haproxy
```

Keepalived track script sẽ phát hiện HAProxy không còn chạy và giảm priority.

VIP sẽ chuyển sang master khác.

Kiểm tra trên các master:

```bash
ip addr show ens33 | grep 10.10.10.10
```

---

## 16.4. Kiểm tra API sau failover

Từ máy quản trị hoặc worker:

```bash
curl -k https://k8s-api.k8s.local:8443/readyz
```

Kỳ vọng:

```text
ok
```

Kiểm tra kubectl:

```bash
kubectl get nodes
kubectl get pods -A
```

Nếu vẫn hoạt động, nghĩa là API endpoint đã failover thành công.

Khởi động lại HAProxy trên node vừa stop:

```bash
sudo systemctl start haproxy
```

---

## 16.5. Mô phỏng mất một master node

Trên một master, ví dụ `master-03`:

```bash
sudo systemctl stop kubelet
```

Kiểm tra node từ master khác:

```bash
kubectl get nodes
```

Có thể sau một thời gian `master-03` chuyển sang `NotReady`.

Kiểm tra API:

```bash
kubectl get pods -A
kubectl get nodes
```

Với 3 master stacked etcd, mất 1 master vẫn còn quorum 2/3.

Khôi phục:

```bash
sudo systemctl start kubelet
```

Theo dõi:

```bash
watch kubectl get nodes
```

---

# Phần 17. Các lệnh kiểm tra quan trọng sau triển khai

## 17.1. Kiểm tra node

```bash
kubectl get nodes -o wide
```

## 17.2. Kiểm tra Pod toàn cluster

```bash
kubectl get pods -A -o wide
```

## 17.3. Kiểm tra component control plane

```bash
kubectl get pods -n kube-system -o wide
```

## 17.4. Kiểm tra endpoint API

```bash
curl -k https://k8s-api.k8s.local:8443/readyz
```

## 17.5. Kiểm tra kubeconfig

```bash
kubectl config view --minify
```

## 17.6. Kiểm tra kubeadm config

```bash
kubectl -n kube-system get cm kubeadm-config -o yaml
```

## 17.7. Kiểm tra certificate SAN của API Server

Trên master:

```bash
sudo openssl x509 -in /etc/kubernetes/pki/apiserver.crt -text -noout | grep -A2 "Subject Alternative Name"
```

Cần thấy:

```text
DNS:k8s-api.k8s.local
IP Address:10.10.10.10
IP Address:10.10.10.11
IP Address:10.10.10.12
IP Address:10.10.10.13
```

## 17.8. Kiểm tra HAProxy

```bash
sudo systemctl status haproxy --no-pager
sudo ss -lntp | grep 8443
```

## 17.9. Kiểm tra Keepalived

```bash
sudo systemctl status keepalived --no-pager
ip addr show ens33 | grep 10.10.10.10
```

## 17.10. Kiểm tra Calico

```bash
kubectl get pods -n calico-system -o wide
kubectl get nodes
```

---

# Phần 18. Sử dụng kubeadm config file thay vì nhiều flags

Trong triển khai production hoặc bài lab nâng cao, nên dùng kubeadm config file để dễ kiểm soát hơn.

Tạo file trên `master-01`:

```bash
cat <<'EOF' > kubeadm-ha-config.yaml
apiVersion: kubeadm.k8s.io/v1beta4
kind: ClusterConfiguration
kubernetesVersion: v1.35.0
clusterName: kubernetes
controlPlaneEndpoint: "k8s-api.k8s.local:8443"
networking:
  podSubnet: "192.168.0.0/16"
  serviceSubnet: "10.96.0.0/12"
apiServer:
  certSANs:
  - "k8s-api.k8s.local"
  - "10.10.10.10"
  - "10.10.10.11"
  - "10.10.10.12"
  - "10.10.10.13"
---
apiVersion: kubelet.config.k8s.io/v1beta1
kind: KubeletConfiguration
cgroupDriver: systemd
EOF
```

Dry-run:

```bash
sudo kubeadm init --config kubeadm-ha-config.yaml --upload-certs --dry-run
```

Init thật:

```bash
sudo kubeadm init --config kubeadm-ha-config.yaml --upload-certs
```

## 18.1. Khi nào nên dùng config file?

Nên dùng config file khi:

- Cụm có nhiều tham số.
- Muốn lưu lại cấu hình triển khai.
- Muốn version control cấu hình.
- Muốn dễ review trước khi triển khai.
- Muốn giảm rủi ro gõ sai flag dài.
- Muốn dùng lại cấu hình cho các môi trường lab/staging/production.

---

# Phần 19. Troubleshooting các lỗi thường gặp

## 19.1. Lỗi VIP không ping được

Triệu chứng:

```bash
ping 10.10.10.10
```

không phản hồi.

Nguyên nhân có thể:

- Keepalived chưa chạy.
- Sai interface trong `keepalived.conf`.
- VRRP bị chặn bởi firewall.
- VIP trùng với IP khác trong mạng.
- Các master không cùng broadcast domain/VLAN.
- Network không hỗ trợ VRRP multicast đúng cách.

Kiểm tra:

```bash
sudo systemctl status keepalived --no-pager
journalctl -u keepalived -xe
ip -br addr
```

Cách xử lý:

- Sửa đúng interface.
- Kiểm tra VLAN/subnet.
- Tạm tắt firewall trong lab.
- Đổi VIP khác chưa sử dụng.
- Nếu môi trường không hỗ trợ VRRP, dùng load balancer ngoài thay vì Keepalived.

---

## 19.2. Lỗi HAProxy không start

Kiểm tra:

```bash
sudo systemctl status haproxy --no-pager
journalctl -u haproxy -xe
```

Các nguyên nhân thường gặp:

- Sai syntax trong `/etc/haproxy/haproxy.cfg`.
- Port `8443` đã bị service khác dùng.
- Bind IP cụ thể không tồn tại trên node.
- Thiếu quyền bind.

Kiểm tra syntax:

```bash
sudo haproxy -c -f /etc/haproxy/haproxy.cfg
```

Kiểm tra port:

```bash
sudo ss -lntp | grep 8443
```

---

## 19.3. Lỗi kubeadm init bị timeout khi gọi API endpoint

Triệu chứng:

```text
context deadline exceeded
```

hoặc:

```text
timed out waiting for the condition
```

Nguyên nhân có thể:

- `controlPlaneEndpoint` trỏ sai.
- DNS `k8s-api.k8s.local` không resolve đúng VIP.
- HAProxy chưa chạy.
- VIP chưa hoạt động.
- HAProxy backend chưa trỏ đúng IP master.
- Firewall chặn port.
- Dùng port 6443 cho HAProxy trên cùng master gây conflict với kube-apiserver.

Kiểm tra:

```bash
getent hosts k8s-api.k8s.local
nc -vz k8s-api.k8s.local 8443
sudo systemctl status haproxy --no-pager
sudo systemctl status keepalived --no-pager
```

---

## 19.4. Lỗi x509 certificate không hợp lệ

Triệu chứng:

```text
x509: certificate is valid for ..., not k8s-api.k8s.local
```

Nguyên nhân:

- Thiếu `k8s-api.k8s.local` trong `--apiserver-cert-extra-sans`.
- Thiếu VIP `10.10.10.10` trong certificate SAN.
- Đã init cluster sai SAN từ đầu.

Kiểm tra:

```bash
sudo openssl x509 -in /etc/kubernetes/pki/apiserver.crt -text -noout | grep -A2 "Subject Alternative Name"
```

Cách xử lý trong lab:

- Nếu mới init và chưa có dữ liệu quan trọng, reset cluster và init lại đúng SAN.
- Với production, cần quy trình rotate certificate cẩn thận.

---

## 19.5. Lỗi join control plane vì certificate key hết hạn

Triệu chứng:

```text
couldn't download the kubeadm-certs Secret
```

hoặc lỗi liên quan certificate key.

Nguyên nhân:

- Secret chứa certificate control plane chỉ tồn tại tạm thời.
- Certificate key cũ đã hết hạn.
- Join command đã quá cũ.

Tạo lại certificate key trên master hiện có:

```bash
sudo kubeadm init phase upload-certs --upload-certs
```

Tạo lại token:

```bash
kubeadm token create --print-join-command
```

Sau đó join control plane với:

```bash
--control-plane --certificate-key <new-certificate-key>
```

---

## 19.6. Node NotReady sau khi join

Kiểm tra:

```bash
kubectl describe node <node-name>
kubectl get pods -A -o wide
```

Nguyên nhân thường gặp:

- CNI chưa cài.
- Calico Pod lỗi.
- Pod CIDR không khớp.
- containerd chưa chạy.
- kubelet lỗi.
- Swap chưa tắt.

Kiểm tra trên node lỗi:

```bash
sudo systemctl status kubelet --no-pager
journalctl -u kubelet -xe
sudo systemctl status containerd --no-pager
free -h
```

---

## 19.7. Calico không chạy

Kiểm tra:

```bash
kubectl get pods -n calico-system -o wide
kubectl describe pod -n calico-system <pod-name>
```

Nguyên nhân thường gặp:

- Pod CIDR không đúng.
- Node không route được đến nhau.
- Kernel module/sysctl thiếu.
- Firewall chặn traffic.
- Interface auto-detection sai.

Trong lab on-premise nhiều card mạng, có thể cần cấu hình Calico chọn đúng interface.

---

## 19.8. Mất 2 master và etcd mất quorum

Triệu chứng:

- Kubernetes API không ổn định hoặc không hoạt động.
- `kubectl get nodes` timeout.
- etcd không còn quorum.

Nguyên nhân:

- Cụm 3 etcd member chỉ chịu được lỗi 1 member.
- Mất 2 member thì không còn quorum.

Cách xử lý:

- Khôi phục ít nhất một master/etcd member.
- Nếu mất dữ liệu nghiêm trọng, dùng quy trình restore etcd snapshot đã học ở Buổi 12.
- Không tự ý xóa/rejoin etcd member khi chưa hiểu trạng thái quorum.

---

# Phần 20. So sánh triển khai single master và 3 master

| Tiêu chí | Single Master | 3 Master HA |
|---|---|---|
| Số control plane node | 1 | 3 |
| etcd | 1 member | 3 members |
| Chịu lỗi control plane | Không | Có, chịu được 1 master lỗi |
| Cần API LB/VIP | Không bắt buộc | Bắt buộc nên có |
| kubeadm init | Đơn giản | Cần `controlPlaneEndpoint`, cert SAN, upload certs |
| Join node | Worker only | Có control-plane join và worker join |
| etcd quorum | Không có HA | Cần quorum 2/3 |
| Vận hành | Dễ | Phức tạp hơn |
| Phù hợp | Lab cơ bản | Lab nâng cao, staging, production nhỏ |

---

# Phần 21. Lab tổng hợp

## Mục tiêu lab

Học viên triển khai hoàn chỉnh cụm Kubernetes HA gồm:

- 3 master.
- 2 worker.
- containerd.
- kubeadm.
- Calico.
- HAProxy.
- Keepalived.
- VIP API endpoint.
- Workload kiểm thử.

---

## Yêu cầu đầu vào

Mỗi VM cần có:

- Ubuntu Server 24.04.
- IP static.
- Hostname đúng.
- Kết nối network giữa các node.
- Internet để tải package và manifest.
- Tối thiểu lab đề xuất:
  - Master: 2 vCPU, 4GB RAM, 40GB disk.
  - Worker: 2 vCPU, 4GB RAM, 40GB disk.

---

## Các bước thực hiện

### Bước 1. Chuẩn bị hostname và `/etc/hosts`

Thực hiện trên tất cả node.

### Bước 2. Tắt swap và cấu hình kernel/sysctl

Thực hiện trên tất cả node.

### Bước 3. Cài containerd

Thực hiện trên tất cả node.

### Bước 4. Cài kubeadm/kubelet/kubectl

Thực hiện trên tất cả node.

### Bước 5. Cài HAProxy và Keepalived

Thực hiện trên 3 master.

### Bước 6. Kiểm tra VIP

Đảm bảo:

```bash
ping 10.10.10.10
getent hosts k8s-api.k8s.local
```

### Bước 7. `kubeadm init` trên master-01

Dùng:

```bash
--control-plane-endpoint
--apiserver-cert-extra-sans
--pod-network-cidr
--upload-certs
```

### Bước 8. Cài Calico

Thực hiện từ master-01.

### Bước 9. Join master-02 và master-03

Dùng:

```bash
kubeadm join ... --control-plane --certificate-key ...
```

### Bước 10. Join worker-01 và worker-02

Dùng join command worker.

### Bước 11. Kiểm tra toàn cluster

```bash
kubectl get nodes -o wide
kubectl get pods -A -o wide
```

### Bước 12. Kiểm tra failover API endpoint

Stop HAProxy trên node giữ VIP và kiểm tra VIP chuyển node khác.

### Bước 13. Deploy workload kiểm thử

Triển khai Deployment và Service NodePort.

### Bước 14. Kiểm tra self-healing

Tắt kubelet trên một master và kiểm tra API vẫn hoạt động.

---

# Phần 22. Checklist triển khai nhanh

## Trước khi kubeadm init

- [ ] Tất cả node có hostname đúng.
- [ ] Tất cả node resolve được `k8s-api.k8s.local`.
- [ ] Swap đã tắt.
- [ ] Kernel module `overlay` và `br_netfilter` đã load.
- [ ] `net.ipv4.ip_forward=1`.
- [ ] containerd chạy ổn định.
- [ ] `SystemdCgroup = true`.
- [ ] kubeadm/kubelet/kubectl đã cài đúng version.
- [ ] HAProxy chạy trên 3 master.
- [ ] Keepalived chạy trên 3 master.
- [ ] VIP active trên một master.
- [ ] Port `8443` listen trên master.
- [ ] Không dùng IP master đơn lẻ làm endpoint HA.

## Sau khi kubeadm init

- [ ] `kubectl get nodes` chạy được.
- [ ] kubeconfig trỏ về `k8s-api.k8s.local:8443`.
- [ ] Calico đã cài.
- [ ] master-01 Ready.
- [ ] Lưu lại join command.
- [ ] Lưu lại certificate key.

## Sau khi join toàn bộ node

- [ ] Có 3 control-plane node.
- [ ] Có 2 worker node.
- [ ] Tất cả node Ready.
- [ ] Có 3 etcd Pod.
- [ ] Có 3 kube-apiserver Pod.
- [ ] Có 3 scheduler Pod.
- [ ] Có 3 controller-manager Pod.
- [ ] Calico Pod Running.
- [ ] API endpoint `/readyz` trả về `ok`.
- [ ] VIP failover hoạt động.
- [ ] Workload test chạy được trên worker.

---

# Phần 23. Câu hỏi kiểm tra cuối phần

## Câu 1

Vì sao cụm Kubernetes 3 master cần `controlPlaneEndpoint`?

Gợi ý trả lời:

`controlPlaneEndpoint` cung cấp một endpoint ổn định cho Kubernetes API Server. Nếu kubeconfig và kubelet trỏ trực tiếp vào một master cụ thể, khi master đó lỗi thì cluster mất endpoint quản trị. Trong mô hình HA, endpoint này thường trỏ đến VIP hoặc load balancer phía trước các kube-apiserver.

---

## Câu 2

Trong mô hình 3 master stacked etcd, cụm chịu được tối đa bao nhiêu master lỗi?

Gợi ý trả lời:

Với 3 etcd member, quorum là 2. Do đó cụm chịu được lỗi 1 master/etcd member. Nếu mất 2 master, etcd mất quorum và API Server có thể không hoạt động đúng.

---

## Câu 3

Sự khác nhau giữa stacked etcd và external etcd là gì?

Gợi ý trả lời:

Stacked etcd chạy etcd trên chính các control plane node. External etcd chạy etcd trên các node riêng, tách khỏi control plane. Stacked etcd dễ triển khai hơn và cần ít node hơn; external etcd phức tạp hơn nhưng phù hợp với môi trường production lớn hơn.

---

## Câu 4

`--upload-certs` dùng để làm gì?

Gợi ý trả lời:

`--upload-certs` upload các certificate cần thiết cho control plane lên cluster dưới dạng Secret tạm thời, được mã hóa bằng certificate key. Khi join thêm control plane node, node mới dùng `--certificate-key` để lấy certificate và tham gia control plane.

---

## Câu 5

Vì sao trong lab này dùng endpoint `k8s-api.k8s.local:8443` thay vì `:6443`?

Gợi ý trả lời:

Vì HAProxy và Keepalived được chạy trực tiếp trên master node. kube-apiserver trên master đã dùng port `6443`, nên HAProxy dùng frontend port `8443` để tránh conflict. HAProxy sau đó forward traffic về backend kube-apiserver trên port `6443`.

---

## Câu 6

Nếu join control plane node báo lỗi certificate key hết hạn thì xử lý thế nào?

Gợi ý trả lời:

Trên master hiện có, chạy:

```bash
sudo kubeadm init phase upload-certs --upload-certs
```

Sau đó dùng certificate key mới cùng với join command có `--control-plane`.

---

## Câu 7

Nếu `kubectl config view --minify` cho thấy server là `https://10.10.10.11:6443`, cụm HA có vấn đề gì?

Gợi ý trả lời:

Kubeconfig đang trỏ trực tiếp vào `master-01`, không trỏ vào endpoint HA. Nếu `master-01` lỗi, kubectl có thể không truy cập được API Server dù các master khác vẫn còn hoạt động. Cụm nên dùng `https://k8s-api.k8s.local:8443` hoặc endpoint HA tương ứng.

---

# Phần 24. Kết luận

Cụm Kubernetes 3 master + 2 worker là bước nâng cấp quan trọng từ mô hình lab cơ bản sang mô hình gần với thực tế vận hành hơn.

Điểm cần nhớ nhất không phải chỉ là “thêm 2 master”, mà là toàn bộ tư duy HA phía sau:

- API Server cần endpoint ổn định.
- Endpoint đó nên nằm sau load balancer hoặc VIP.
- kubeadm phải được init đúng với `controlPlaneEndpoint`.
- Certificate API Server phải có SAN phù hợp.
- Control plane node mới cần join bằng `--control-plane`.
- Certificate control plane cần được chia sẻ an toàn bằng `--upload-certs` và `--certificate-key`.
- etcd quorum là yếu tố sống còn.
- 3 etcd member chỉ chịu được lỗi 1 member.
- Backup etcd vẫn là bắt buộc.
- HA control plane giúp tăng khả năng sẵn sàng của API/control plane, nhưng không thay thế backup, monitoring và quy trình vận hành chuẩn.

Sau phần này, đã có thể hiểu và triển khai một cụm Kubernetes kubeadm HA cơ bản trong môi trường on-premise.

---

# Phụ lục. Tài liệu tham khảo nên đọc thêm

- Kubernetes Documentation - Creating Highly Available Clusters with kubeadm  
  `https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/high-availability/`

- Kubernetes Documentation - Options for Highly Available Topology  
  `https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/ha-topology/`

- Kubernetes Documentation - Creating a cluster with kubeadm  
  `https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/`

- Kubernetes Documentation - kubeadm init  
  `https://kubernetes.io/docs/reference/setup-tools/kubeadm/kubeadm-init/`

- Kubernetes Documentation - kubeadm Configuration v1beta4  
  `https://kubernetes.io/docs/reference/config-api/kubeadm-config.v1beta4/`

- Kubernetes Documentation - Set up a High Availability etcd Cluster with kubeadm  
  `https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/setup-ha-etcd-with-kubeadm/`
