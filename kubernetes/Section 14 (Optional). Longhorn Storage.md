# Module mở rộng: Triển khai Storage Longhorn cho Kubernetes On-premise

## 1. Mục tiêu của module

Ở các buổi trước, đã học:

- Kubernetes architecture.
- Bootstrap cluster bằng `kubeadm`.
- Pod, ReplicaSet, Deployment, DaemonSet, StatefulSet, Job, CronJob.
- Service, DNS, Ingress.
- ConfigMap, Secret, Volume, PV, PVC, StorageClass và CSI NFS.
- Helm, MetalLB, Ingress NGINX Controller.
- Resource management, scheduling, security, observability, autoscaling.
- Backup/restore và disaster recovery.
- Module mở rộng cụm Kubernetes HA 3 master + 2 worker.

Module này mở rộng phần Storage bằng cách triển khai **Longhorn** làm distributed block storage cho Kubernetes on-premise.

Sau module này, cần hiểu và làm được:

- Longhorn là gì và giải quyết bài toán gì trong Kubernetes on-premise.
- Khác biệt giữa NFS CSI và Longhorn CSI.
- Kiến trúc chính của Longhorn.
- Điều kiện hạ tầng cần có trước khi triển khai Longhorn.
- Cài đặt Longhorn bằng Helm.
- Truy cập Longhorn UI qua Ingress.
- Tạo StorageClass Longhorn.
- Tạo PVC dùng Longhorn.
- Mount PVC vào Deployment.
- Kiểm tra volume, replica, attachment, node, disk.
- Mở rộng dung lượng PVC.
- Tạo snapshot, backup, recurring job.
- Hiểu các tình huống lỗi phổ biến và cách troubleshooting.
- Biết giới hạn khi triển khai Longhorn trên cụm chỉ có 2 worker node.

---

## 2. Vị trí của module trong giáo trình

Module này nên được đặt sau:

- Buổi 05: ConfigMap, Secret, Volume, PV, PVC, StorageClass, CSI NFS.
- Buổi 06: Helm, MetalLB, Ingress NGINX Controller.
- Buổi 08: Resource Management và Scheduling.
- Buổi 10: Observability và Troubleshooting.
- Buổi 12: Backup, Restore và Disaster Recovery.
- Module mở rộng: Kubernetes HA 3 master + 2 worker.

Lý do:

Longhorn không nên được học quá sớm.

Nếu chưa hiểu PV, PVC, StorageClass, CSI, Node, Pod, DaemonSet, Deployment, Ingress và troubleshooting, thì Longhorn sẽ bị hiểu như một công cụ “cài cho có giao diện”, thay vì hiểu đúng nó là một lớp storage phân tán chạy bên trong Kubernetes.

---

## 3. Bối cảnh lab

Module này tiếp tục dựa trên cụm Kubernetes on-premise sử dụng:

- Ubuntu Server 24.04.
- Kubernetes v1.35.
- Bootstrap bằng `kubeadm`.
- Container runtime: `containerd`.
- CNI: Calico.
- Helm đã được cài từ Buổi 06.
- Ingress NGINX Controller đã được cài từ Buổi 06.
- MetalLB đã được cài từ Buổi 06.

Có hai mô hình có thể dùng cho module này.

---

## 4. Mô hình lab khuyến nghị

### 4.1. Mô hình khuyến nghị cho bài học Longhorn

Để Longhorn thể hiện đúng khả năng HA với 3 replicas, nên có ít nhất **3 node có thể dùng làm storage node**.

Mô hình khuyến nghị:

| Node | Role Kubernetes | IP ví dụ | Vai trò trong Longhorn |
|---|---|---:|---|
| `master01` | control-plane | `10.10.10.11` | Không chạy storage workload |
| `master02` | control-plane | `10.10.10.12` | Không chạy storage workload |
| `master03` | control-plane | `10.10.10.13` | Không chạy storage workload |
| `worker01` | worker | `10.10.10.21` | Storage node |
| `worker02` | worker | `10.10.10.22` | Storage node |
| `worker03` | worker | `10.10.10.23` | Storage node |

Mô hình này cần thêm một worker thứ ba so với mô hình HA 3 master + 2 worker.

Đây là mô hình phù hợp nhất để dạy Longhorn, vì mặc định Longhorn thường dùng `numberOfReplicas: "3"`. Khi đó một volume sẽ có ba bản sao dữ liệu nằm trên ba node khác nhau.

---

### 4.2. Mô hình lab tối thiểu nếu chỉ có 3 master + 2 worker

Nếu lớp học chỉ có đúng 5 VM:

| Node | Role Kubernetes | IP ví dụ |
|---|---|---:|
| `master01` | control-plane | `10.10.10.11` |
| `master02` | control-plane | `10.10.10.12` |
| `master03` | control-plane | `10.10.10.13` |
| `worker01` | worker | `10.10.10.21` |
| `worker02` | worker | `10.10.10.22` |

Thì có ba hướng triển khai:

### Phương án A: dùng `replicaCount = 2`

Đây là phương án phù hợp nhất cho lab tối thiểu.

Ưu điểm:

- Không cần cho control-plane node tham gia storage.
- Dễ triển khai.
- Đúng với mô hình chỉ có 2 worker.

Nhược điểm:

- Volume chỉ có 2 replicas.
- Chịu lỗi kém hơn mô hình 3 replicas.
- Không thể minh họa đầy đủ kiến trúc replica 3 node.

### Phương án B: cho một control-plane node tham gia storage trong lab

Ví dụ cho `master03` tham gia storage.

Ưu điểm:

- Có thể chạy Longhorn với 3 replicas.
- Minh họa tốt hơn về distributed replica.

Nhược điểm:

- Không khuyến nghị cho production.
- Control-plane node phải gánh thêm storage workload.
- Cần xử lý taint/toleration hoặc cấu hình Longhorn để chạy trên node đó.

### Phương án C: thêm worker/storage node thứ ba

Đây là phương án tốt nhất nếu có thể bổ sung VM.

Ưu điểm:

- Đúng production mindset hơn.
- Không trộn control-plane với storage workload.
- Dạy được đầy đủ HA storage.

Nhược điểm:

- Cần thêm tài nguyên hạ tầng.

Trong module này, phần lab chính sẽ dùng **Phương án A: 3 master + 2 worker với `replicaCount = 2`**, vì nó phù hợp với mô hình hiện tại của lớp học. Đồng thời, bài sẽ có ghi chú riêng cho mô hình production hoặc lab mở rộng với 3 worker/storage node.

---

## 5. Longhorn là gì?

Longhorn là một giải pháp **cloud-native distributed block storage** cho Kubernetes.

Nói đơn giản:

- Mỗi node có một hoặc nhiều disk local.
- Longhorn gom các disk local đó thành một lớp storage phân tán.
- Khi ứng dụng tạo PVC, Longhorn tự động tạo volume.
- Volume có thể được replicate sang nhiều node.
- Nếu một node lỗi, dữ liệu vẫn còn replica ở node khác.
- Longhorn cung cấp snapshot, backup, restore, volume expansion, UI quản trị và CSI driver.

Trong môi trường cloud như AWS, Azure, GCP, Kubernetes thường có thể dùng storage backend của cloud provider như EBS, Azure Disk, Persistent Disk.

Trong môi trường on-premise, đặc biệt là cụm kubeadm chạy trên VM hoặc server vật lý, không có sẵn cloud disk service. Longhorn giúp giải quyết bài toán đó bằng cách cung cấp distributed block storage trực tiếp bên trong Kubernetes.

---

## 6. Longhorn khác gì NFS CSI?

Ở Buổi 05, đã triển khai NFS CSI.

NFS CSI và Longhorn đều có thể dùng để cấp PVC động, nhưng bản chất rất khác nhau.

| Tiêu chí | NFS CSI | Longhorn |
|---|---|---|
| Bản chất | File storage qua NFS | Distributed block storage |
| Backend chính | Một NFS Server có sẵn | Disk local trên nhiều Kubernetes node |
| HA storage | Phụ thuộc HA của NFS Server | Longhorn replicate volume giữa nhiều node |
| Access mode phổ biến | RWX | RWO, có hỗ trợ RWX qua share-manager/NFS |
| Performance | Phụ thuộc NFS Server/network | Phụ thuộc disk local, replica, network giữa node |
| Quản trị volume | Đơn giản | Có UI, snapshot, backup, replica, rebuild |
| Phù hợp lab | Rất phù hợp | Phù hợp lab nâng cao/production-like |
| Phù hợp stateful app | Có thể dùng | Phù hợp hơn cho nhiều workload stateful RWO |
| Khi node lỗi | NFS vẫn sống nếu NFS Server sống | Volume có thể attach lại nếu còn replica khỏe |

Điểm cần nhớ:

NFS CSI giúp Kubernetes cấp thư mục trên NFS Server thành PVC.

Longhorn tạo block volume phân tán, replicate dữ liệu giữa các node, rồi expose volume đó cho Pod thông qua CSI.

---

## 7. Kiến trúc tổng quan của Longhorn

Sơ đồ khái niệm:

```text
+---------------------------------------------------------------+
|                       Kubernetes Cluster                      |
|                                                               |
|  +-------------------+    +-------------------+               |
|  |     worker01      |    |     worker02      |               |
|  |                   |    |                   |               |
|  |  /var/lib/longhorn|    |  /var/lib/longhorn|               |
|  |       disk        |    |       disk        |               |
|  |                   |    |                   |               |
|  |  Longhorn Replica |    |  Longhorn Replica |               |
|  +---------+---------+    +---------+---------+               |
|            \                    /                             |
|             \                  /                              |
|              \                /                               |
|               +--------------+                                |
|               | Longhorn Vol |                                |
|               +------+-------+                                |
|                      |                                        |
|                      | CSI                                    |
|                      v                                        |
|                +-----------+                                  |
|                |    PVC    |                                  |
|                +-----+-----+                                  |
|                      |                                        |
|                      v                                        |
|                +-----------+                                  |
|                |    Pod    |                                  |
|                +-----------+                                  |
|                                                               |
+---------------------------------------------------------------+
```

Các thành phần chính:

| Thành phần | Vai trò |
|---|---|
| Longhorn Manager | Quản lý node, disk, volume, replica, engine |
| Longhorn CSI Driver | Tích hợp Longhorn với Kubernetes PV/PVC/StorageClass |
| Longhorn Engine | Thành phần điều phối I/O của volume |
| Longhorn Replica | Bản sao dữ liệu volume nằm trên disk của node |
| Instance Manager | Quản lý engine/replica process trên node |
| Longhorn UI | Giao diện quản trị |
| Share Manager | Cung cấp RWX volume thông qua NFSv4.1 |
| Backup Target | Nơi lưu backup ngoài cluster, ví dụ S3-compatible hoặc NFS |

---

## 8. Luồng tạo PVC với Longhorn

Khi người dùng tạo PVC:

```text
User tạo PVC
    |
    v
Kubernetes đọc storageClassName
    |
    v
StorageClass trỏ tới provisioner driver.longhorn.io
    |
    v
Longhorn CSI nhận yêu cầu provision volume
    |
    v
Longhorn tạo volume
    |
    v
Longhorn tạo replicas trên các node/disk phù hợp
    |
    v
Kubernetes tạo PV tương ứng
    |
    v
PVC chuyển sang trạng thái Bound
    |
    v
Pod mount PVC
    |
    v
Volume attach vào node đang chạy Pod
```

Điểm quan trọng:

PVC không trực tiếp biết disk vật lý nằm ở đâu.

PVC chỉ biết nó cần bao nhiêu dung lượng, access mode gì và dùng StorageClass nào.

Longhorn chịu trách nhiệm tạo volume thật, replicate dữ liệu và attach volume vào node phù hợp.

---

## 9. Điều kiện triển khai Longhorn

Trên mỗi node Longhorn chạy, cần chuẩn bị:

- Kubernetes cluster đang hoạt động.
- Container runtime tương thích Kubernetes.
- `open-iscsi` được cài và service `iscsid` đang chạy.
- `iscsi_tcp` kernel module được load.
- NFSv4 client nếu dùng RWX volume hoặc backup qua NFS.
- Filesystem hỗ trợ file extents như `ext4` hoặc `xfs`.
- Các công cụ hệ thống như `bash`, `curl`, `findmnt`, `grep`, `awk`, `blkid`, `lsblk`.
- Mount propagation được hỗ trợ.
- Longhorn components cần chạy với quyền root/privileged để thao tác block device, mount, iSCSI và host namespace.

Trong lab Ubuntu 24.04, ta sẽ cài các gói cần thiết bằng `apt`.

---

## 10. Chuẩn bị disk cho Longhorn

### 10.1. Nguyên tắc

Có hai cách dùng disk cho Longhorn:

### Cách 1: dùng thư mục mặc định `/var/lib/longhorn`

Đây là cách đơn giản nhất.

Longhorn sẽ lưu dữ liệu volume dưới:

```text
/var/lib/longhorn
```

Ưu điểm:

- Dễ dùng trong lab.
- Không cần thêm disk.

Nhược điểm:

- Dữ liệu Longhorn nằm chung với OS disk.
- Không khuyến nghị production.

### Cách 2: mount disk riêng vào `/var/lib/longhorn`

Đây là cách nên dùng hơn.

Ví dụ mỗi worker có thêm disk `/dev/sdb`.

Ta format disk và mount vào:

```text
/var/lib/longhorn
```

Ưu điểm:

- Tách OS disk và data disk.
- Gần production hơn.
- Dễ quản lý dung lượng.

Nhược điểm:

- Cần thêm disk cho từng node.

Trong module này, lab sẽ dùng cách 2 nếu VM có thêm disk `/dev/sdb`. Nếu không có, vẫn có thể dùng cách 1 để học flow.

---

## 11. Chuẩn bị node cho Longhorn

Thực hiện trên các node sẽ tham gia Longhorn storage, ví dụ:

- `worker01`
- `worker02`

Nếu có `worker03`, thực hiện thêm trên `worker03`.

### 11.1. Cài package cần thiết

```bash
sudo apt update

sudo apt install -y \
  open-iscsi \
  nfs-common \
  cryptsetup \
  dmsetup \
  curl \
  jq \
  util-linux
```

Giải thích:

| Package | Ý nghĩa |
|---|---|
| `open-iscsi` | Cần cho Longhorn attach block volume qua iSCSI |
| `nfs-common` | Cần cho RWX volume và backup target NFS |
| `cryptsetup` | Cần nếu dùng encrypted volume |
| `dmsetup` | Cần cho device mapper |
| `curl` | Tải công cụ / kiểm thử HTTP |
| `jq` | Parse JSON khi troubleshooting |
| `util-linux` | Cung cấp nhiều công cụ hệ thống như `lsblk`, `findmnt` |

---

### 11.2. Enable iscsid

```bash
sudo systemctl enable --now iscsid
sudo systemctl status iscsid --no-pager
```

Kết quả kỳ vọng:

```text
Active: active (running)
```

---

### 11.3. Load module `iscsi_tcp`

```bash
sudo modprobe iscsi_tcp
lsmod | grep iscsi_tcp
```

Để module tự load sau reboot:

```bash
echo iscsi_tcp | sudo tee /etc/modules-load.d/iscsi_tcp.conf
```

---

### 11.4. Kiểm tra NFSv4 support

```bash
grep CONFIG_NFS_V4 /boot/config-$(uname -r) || true
grep CONFIG_NFS_V4_1 /boot/config-$(uname -r) || true
```

Nếu thấy các dòng tương tự sau là ổn:

```text
CONFIG_NFS_V4=y
CONFIG_NFS_V4_1=y
```

---

## 12. Chuẩn bị dedicated disk cho Longhorn

> Chỉ thực hiện phần này nếu VM có disk riêng, ví dụ `/dev/sdb`.

### 12.1. Kiểm tra disk

```bash
lsblk
```

Ví dụ:

```text
NAME   MAJ:MIN RM  SIZE RO TYPE MOUNTPOINTS
sda      8:0    0   80G  0 disk
├─sda1   8:1    0    1G  0 part /boot/efi
└─sda2   8:2    0   79G  0 part /
sdb      8:16   0  100G  0 disk
```

Ở đây `/dev/sdb` là disk dành cho Longhorn.

---

### 12.2. Format disk

Cẩn thận: lệnh sau sẽ xóa dữ liệu trên `/dev/sdb`.

```bash
sudo parted /dev/sdb --script mklabel gpt
sudo mkfs.ext4 -F /dev/sdb
```

---

### 12.3. Mount disk vào `/var/lib/longhorn`

```bash
sudo mkdir -p /var/lib/longhorn

sudo blkid /dev/sdb
```

Ví dụ output:

```text
/dev/sdb: UUID="1111-2222-3333-4444" BLOCK_SIZE="4096" TYPE="ext4"
```

Thêm vào `/etc/fstab`:

```bash
echo "UUID=$(sudo blkid -s UUID -o value /dev/sdb) /var/lib/longhorn ext4 defaults 0 2" | sudo tee -a /etc/fstab
```

Mount:

```bash
sudo mount -a
df -h /var/lib/longhorn
```

Kết quả kỳ vọng:

```text
Filesystem      Size  Used Avail Use% Mounted on
/dev/sdb         98G   24K   93G   1% /var/lib/longhorn
```

---

## 13. Kiểm tra cluster trước khi cài Longhorn

Thực hiện trên máy có `kubectl`, thường là `master01`.

```bash
kubectl get nodes -o wide
kubectl get pods -A
kubectl get sc
```

Kết quả mong muốn:

- Tất cả node `Ready`.
- CoreDNS chạy.
- Calico chạy.
- Ingress NGINX Controller chạy nếu muốn expose UI qua Ingress.
- MetalLB chạy nếu Ingress Controller dùng Service type `LoadBalancer`.

---

## 14. Kiểm tra taint trên control-plane node

```bash
kubectl describe node master01 | grep -i taint -A2
kubectl describe node master02 | grep -i taint -A2
kubectl describe node master03 | grep -i taint -A2
```

Thường control-plane node có taint:

```text
node-role.kubernetes.io/control-plane:NoSchedule
```

Điều này có nghĩa là Pod workload thông thường không được schedule lên control-plane node.

Trong module này, mặc định Longhorn storage sẽ chạy trên worker node.

Nếu chỉ có 2 worker, ta sẽ cấu hình StorageClass replica count là 2.

---

## 15. Gắn label cho node storage

Để dễ quản lý, ta label các worker node sẽ dùng làm storage node.

```bash
kubectl label node worker01 node.longhorn.io/storage=true
kubectl label node worker02 node.longhorn.io/storage=true
```

Nếu có worker thứ ba:

```bash
kubectl label node worker03 node.longhorn.io/storage=true
```

Kiểm tra:

```bash
kubectl get nodes --show-labels | grep node.longhorn.io/storage
```

Lưu ý:

Label này dùng cho mục đích quản trị/lab. Bản thân Longhorn không tự động chỉ dùng node có label này nếu chưa cấu hình node selector hoặc cấu hình node/disk trong Longhorn.

---

## 16. Cài Longhorn bằng Helm

### 16.1. Thêm Helm repository

```bash
helm repo add longhorn https://charts.longhorn.io
helm repo update
```

---

### 16.2. Tạo file values cho lab 2 worker

Tạo file:

```bash
cat > longhorn-values-lab-2worker.yaml <<'EOF'
persistence:
  defaultClass: true
  defaultClassReplicaCount: 2
  defaultFsType: ext4
  reclaimPolicy: Delete

defaultSettings:
  defaultDataPath: /var/lib/longhorn
  defaultReplicaCount: 2
  storageOverProvisioningPercentage: 200
  storageMinimalAvailablePercentage: 25
  createDefaultDiskLabeledNodes: false
  replicaSoftAntiAffinity: disabled
  replicaAutoBalance: best-effort

longhornUI:
  replicas: 1
EOF
```

Giải thích các cấu hình quan trọng:

| Cấu hình | Ý nghĩa |
|---|---|
| `persistence.defaultClass: true` | Cho phép Longhorn tạo StorageClass mặc định |
| `persistence.defaultClassReplicaCount: 2` | StorageClass mặc định tạo volume có 2 replicas |
| `defaultSettings.defaultReplicaCount: 2` | Replica mặc định trong Longhorn là 2 |
| `defaultSettings.defaultDataPath` | Đường dẫn lưu data trên node |
| `storageOverProvisioningPercentage: 200` | Cho phép over-provision dung lượng ở mức 200% |
| `storageMinimalAvailablePercentage: 25` | Giữ tối thiểu 25% dung lượng trống |
| `replicaSoftAntiAffinity: disabled` | Không đặt nhiều replica của cùng volume trên cùng node nếu không đủ điều kiện |
| `replicaAutoBalance: best-effort` | Cố gắng cân bằng replica giữa các node |
| `longhornUI.replicas: 1` | Lab dùng 1 replica UI cho nhẹ |

Vì lab chỉ có 2 worker, ta đặt replica count là 2.

Nếu có 3 worker/storage node, hãy dùng `3` thay cho `2`.

---

### 16.3. Cài Longhorn

```bash
helm install longhorn longhorn/longhorn \
  --namespace longhorn-system \
  --create-namespace \
  --version 1.11.1 \
  -f longhorn-values-lab-2worker.yaml
```

---

### 16.4. Theo dõi Pod Longhorn

```bash
kubectl -n longhorn-system get pods -w
```

Kết quả kỳ vọng:

```text
longhorn-manager-xxxxx               1/1     Running
longhorn-driver-deployer-xxxxx       1/1     Running
longhorn-ui-xxxxx                    1/1     Running
longhorn-csi-plugin-xxxxx            3/3     Running
csi-attacher-xxxxx                   1/1     Running
csi-provisioner-xxxxx                1/1     Running
csi-resizer-xxxxx                    1/1     Running
csi-snapshotter-xxxxx                1/1     Running
instance-manager-xxxxx               1/1     Running
engine-image-xxxxx                   1/1     Running
```

Lưu ý:

Tên Pod thực tế có thể khác tùy version.

---

## 17. Kiểm tra Longhorn StorageClass

```bash
kubectl get sc
```

Kết quả ví dụ:

```text
NAME                 PROVISIONER          RECLAIMPOLICY   VOLUMEBINDINGMODE
longhorn (default)   driver.longhorn.io   Delete          Immediate
```

Xem chi tiết:

```bash
kubectl describe sc longhorn
```

Các điểm cần chú ý:

```text
Provisioner: driver.longhorn.io
AllowVolumeExpansion: True
ReclaimPolicy: Delete
VolumeBindingMode: Immediate
```

Giải thích:

| Thuộc tính | Ý nghĩa |
|---|---|
| `driver.longhorn.io` | CSI provisioner của Longhorn |
| `AllowVolumeExpansion: True` | Cho phép mở rộng PVC |
| `Delete` | Khi PVC bị xóa, PV/volume cũng bị xóa |
| `Immediate` | Volume được provision ngay khi PVC được tạo |

---

## 18. Expose Longhorn UI qua Ingress

Ở Buổi 06, đã cài Ingress NGINX Controller.

Ta có thể expose Longhorn UI bằng Ingress.

### 18.1. Tạo Secret basic auth

Cài `apache2-utils` nếu chưa có:

```bash
sudo apt install -y apache2-utils
```

Tạo file auth:

```bash
htpasswd -bc auth admin 'Longhorn@123'
```

Tạo Secret:

```bash
kubectl -n longhorn-system create secret generic longhorn-basic-auth \
  --from-file=auth
```

Giải thích:

| Thành phần | Ý nghĩa |
|---|---|
| `admin` | Username đăng nhập |
| `Longhorn@123` | Password lab |
| `longhorn-basic-auth` | Secret chứa file `auth` dùng cho NGINX Ingress |

Trong production, không nên để password trong shell history như ví dụ lab.

---

### 18.2. Tạo Ingress HTTP cho Longhorn UI

Ví dụ domain lab:

```text
longhorn.k8s.local
```

Manifest:

```bash
cat > longhorn-ui-ingress.yaml <<'EOF'
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: longhorn-ui
  namespace: longhorn-system
  annotations:
    nginx.ingress.kubernetes.io/auth-type: basic
    nginx.ingress.kubernetes.io/auth-secret: longhorn-basic-auth
    nginx.ingress.kubernetes.io/auth-realm: "Longhorn UI Authentication"
spec:
  ingressClassName: nginx
  rules:
  - host: longhorn.k8s.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: longhorn-frontend
            port:
              number: 80
EOF

kubectl apply -f longhorn-ui-ingress.yaml
```

Giải thích manifest:

```yaml
apiVersion: networking.k8s.io/v1
```

Dùng API version của Ingress.

```yaml
kind: Ingress
```

Khai báo object Ingress.

```yaml
metadata:
  name: longhorn-ui
  namespace: longhorn-system
```

Ingress nằm trong namespace `longhorn-system` vì Service `longhorn-frontend` cũng nằm ở namespace này.

```yaml
annotations:
  nginx.ingress.kubernetes.io/auth-type: basic
  nginx.ingress.kubernetes.io/auth-secret: longhorn-basic-auth
```

Bật basic authentication cho Longhorn UI.

Longhorn UI mặc định không nên expose trực tiếp ra ngoài mà không có authentication.

```yaml
spec:
  ingressClassName: nginx
```

Dùng Ingress NGINX Controller đã học ở Buổi 06.

```yaml
rules:
- host: longhorn.k8s.local
```

Chỉ route request có Host header là `longhorn.k8s.local`.

```yaml
backend:
  service:
    name: longhorn-frontend
    port:
      number: 80
```

Forward traffic vào Service UI của Longhorn.

---

### 18.3. Kiểm tra Ingress

```bash
kubectl -n longhorn-system get ingress
kubectl -n ingress-nginx get svc
```

Nếu đang dùng domain lab, thêm record vào máy client:

```text
<INGRESS_EXTERNAL_IP> longhorn.k8s.local
```

Ví dụ trên máy Linux/macOS:

```bash
sudo sh -c 'echo "10.10.10.240 longhorn.k8s.local" >> /etc/hosts'
```

Truy cập:

```text
http://longhorn.k8s.local
```

---

## 19. Tạo namespace lab

```bash
kubectl create namespace longhorn-lab
```

---

## 20. Lab 1: Tạo PVC dùng Longhorn

Manifest:

```bash
cat > pvc-longhorn-rwo.yaml <<'EOF'
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: data-rwo
  namespace: longhorn-lab
spec:
  accessModes:
  - ReadWriteOnce
  storageClassName: longhorn
  resources:
    requests:
      storage: 2Gi
EOF

kubectl apply -f pvc-longhorn-rwo.yaml
```

Giải thích manifest:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
```

Khai báo PVC.

```yaml
metadata:
  name: data-rwo
  namespace: longhorn-lab
```

PVC tên `data-rwo`, nằm trong namespace `longhorn-lab`.

```yaml
accessModes:
- ReadWriteOnce
```

PVC được mount dạng RWO.

RWO nghĩa là volume có thể được mount read-write bởi một node tại một thời điểm.

```yaml
storageClassName: longhorn
```

Yêu cầu Kubernetes dùng StorageClass `longhorn`.

```yaml
resources:
  requests:
    storage: 2Gi
```

Yêu cầu volume 2Gi.

---

### 20.1. Kiểm tra PVC và PV

```bash
kubectl -n longhorn-lab get pvc
kubectl get pv
```

Kết quả kỳ vọng:

```text
NAME       STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS
data-rwo   Bound    pvc-xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx   2Gi        RWO            longhorn
```

Xem chi tiết:

```bash
kubectl -n longhorn-lab describe pvc data-rwo
```

---

## 21. Lab 2: Mount PVC vào Pod

Manifest:

```bash
cat > pod-longhorn-rwo.yaml <<'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: pod-longhorn-rwo
  namespace: longhorn-lab
spec:
  containers:
  - name: app
    image: busybox:1.36
    command:
    - sh
    - -c
    - |
      while true; do
        date >> /data/app.log
        echo "Longhorn storage is working" >> /data/app.log
        sleep 5
      done
    volumeMounts:
    - name: app-data
      mountPath: /data
  volumes:
  - name: app-data
    persistentVolumeClaim:
      claimName: data-rwo
EOF

kubectl apply -f pod-longhorn-rwo.yaml
```

Giải thích manifest:

```yaml
image: busybox:1.36
```

Dùng image nhỏ để tạo file test.

```yaml
command:
- sh
- -c
- |
  while true; do
    date >> /data/app.log
    echo "Longhorn storage is working" >> /data/app.log
    sleep 5
  done
```

Container ghi log liên tục vào `/data/app.log`.

```yaml
volumeMounts:
- name: app-data
  mountPath: /data
```

Mount volume vào `/data`.

```yaml
volumes:
- name: app-data
  persistentVolumeClaim:
    claimName: data-rwo
```

Dùng PVC `data-rwo`.

---

### 21.1. Kiểm tra Pod

```bash
kubectl -n longhorn-lab get pod -o wide
kubectl -n longhorn-lab logs pod-longhorn-rwo
```

Kiểm tra dữ liệu:

```bash
kubectl -n longhorn-lab exec -it pod-longhorn-rwo -- tail -n 10 /data/app.log
```

---

### 21.2. Quan sát volume trong Longhorn UI

Vào Longhorn UI:

```text
http://longhorn.k8s.local
```

Quan sát:

- Volume name thường là tên PV, ví dụ `pvc-xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx`.
- State: `Attached`.
- Health: `Healthy`.
- Size: `2 Gi`.
- Replicas: `2` trong lab 2 worker.
- Node đang attach volume.

---

## 22. Lab 3: Kiểm tra self-healing của volume khi Pod bị xóa

Xóa Pod:

```bash
kubectl -n longhorn-lab delete pod pod-longhorn-rwo
```

Tạo lại Pod:

```bash
kubectl apply -f pod-longhorn-rwo.yaml
```

Kiểm tra dữ liệu:

```bash
kubectl -n longhorn-lab exec -it pod-longhorn-rwo -- tail -n 20 /data/app.log
```

Kết quả mong muốn:

Dữ liệu cũ vẫn còn.

Giải thích:

- Pod là ephemeral.
- PVC/PV/Longhorn volume vẫn tồn tại.
- Khi Pod mới mount lại PVC, dữ liệu vẫn còn trên volume Longhorn.

---

## 23. Lab 4: Dùng Deployment với Longhorn PVC

Với RWO volume, một PVC thường chỉ nên được mount read-write bởi một Pod chạy trên một node tại một thời điểm.

Manifest:

```bash
cat > deploy-longhorn-rwo.yaml <<'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-longhorn-rwo
  namespace: longhorn-lab
spec:
  replicas: 1
  selector:
    matchLabels:
      app: web-longhorn-rwo
  template:
    metadata:
      labels:
        app: web-longhorn-rwo
    spec:
      containers:
      - name: web
        image: nginx:1.27
        ports:
        - containerPort: 80
        volumeMounts:
        - name: web-data
          mountPath: /usr/share/nginx/html
      initContainers:
      - name: init-index
        image: busybox:1.36
        command:
        - sh
        - -c
        - |
          echo "<h1>Hello from Longhorn RWO volume</h1>" > /data/index.html
          echo "<p>Generated at $(date)</p>" >> /data/index.html
        volumeMounts:
        - name: web-data
          mountPath: /data
      volumes:
      - name: web-data
        persistentVolumeClaim:
          claimName: data-rwo
---
apiVersion: v1
kind: Service
metadata:
  name: web-longhorn-rwo
  namespace: longhorn-lab
spec:
  type: ClusterIP
  selector:
    app: web-longhorn-rwo
  ports:
  - port: 80
    targetPort: 80
EOF

kubectl apply -f deploy-longhorn-rwo.yaml
```

Giải thích phần liên quan đến Longhorn:

```yaml
replicas: 1
```

Vì PVC là RWO, ta chạy 1 replica để tránh nhiều Pod cùng tranh một volume.

```yaml
initContainers:
- name: init-index
```

Init container tạo file HTML trước khi container NGINX chạy.

```yaml
volumeMounts:
- name: web-data
  mountPath: /usr/share/nginx/html
```

NGINX đọc file web từ Longhorn volume.

```yaml
persistentVolumeClaim:
  claimName: data-rwo
```

Deployment dùng lại PVC `data-rwo`.

---

### 23.1. Kiểm tra Service

```bash
kubectl -n longhorn-lab get pod,svc
kubectl -n longhorn-lab run curl --image=curlimages/curl:8.10.1 -it --rm --restart=Never -- \
  curl http://web-longhorn-rwo
```

Kết quả:

```html
<h1>Hello from Longhorn RWO volume</h1>
```

---

## 24. Lab 5: Mở rộng dung lượng PVC

Longhorn StorageClass mặc định hỗ trợ volume expansion.

Kiểm tra PVC ban đầu:

```bash
kubectl -n longhorn-lab get pvc data-rwo
```

Patch PVC từ `2Gi` lên `4Gi`:

```bash
kubectl -n longhorn-lab patch pvc data-rwo \
  -p '{"spec":{"resources":{"requests":{"storage":"4Gi"}}}}'
```

Kiểm tra:

```bash
kubectl -n longhorn-lab get pvc data-rwo
kubectl -n longhorn-lab describe pvc data-rwo
```

Kiểm tra trong Pod:

```bash
kubectl -n longhorn-lab exec -it deploy/web-longhorn-rwo -- df -h /usr/share/nginx/html
```

Kết quả kỳ vọng:

Dung lượng mount point tăng lên khoảng 4Gi.

Giải thích:

- Kubernetes cập nhật request storage trên PVC.
- CSI resizer xử lý resize.
- Longhorn mở rộng block device.
- Filesystem bên trong volume được mở rộng.
- Ứng dụng tiếp tục chạy nếu online expansion hoạt động bình thường.

---

## 25. Lab 6: Tạo StorageClass riêng cho Longhorn

Trong production, không nên chỉ dùng một StorageClass mặc định cho mọi workload.

Ta có thể tạo nhiều StorageClass khác nhau.

Ví dụ:

- `longhorn-2replica-delete`
- `longhorn-2replica-retain`
- `longhorn-3replica-production`
- `longhorn-local-fast`

Trong lab 2 worker, tạo StorageClass 2 replicas:

```bash
cat > sc-longhorn-2replica-retain.yaml <<'EOF'
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: longhorn-2replica-retain
provisioner: driver.longhorn.io
allowVolumeExpansion: true
reclaimPolicy: Retain
volumeBindingMode: Immediate
parameters:
  numberOfReplicas: "2"
  staleReplicaTimeout: "30"
  fsType: "ext4"
  dataLocality: "disabled"
  replicaAutoBalance: "best-effort"
EOF

kubectl apply -f sc-longhorn-2replica-retain.yaml
```

Giải thích:

```yaml
provisioner: driver.longhorn.io
```

StorageClass này dùng Longhorn CSI.

```yaml
allowVolumeExpansion: true
```

Cho phép mở rộng PVC.

```yaml
reclaimPolicy: Retain
```

Khi PVC bị xóa, PV không bị xóa ngay. Dữ liệu có cơ hội được xử lý thủ công.

```yaml
parameters:
  numberOfReplicas: "2"
```

Volume tạo từ StorageClass này có 2 replicas.

```yaml
dataLocality: "disabled"
```

Không ép Longhorn giữ replica cùng node với workload.

```yaml
replicaAutoBalance: "best-effort"
```

Longhorn cố gắng cân bằng replica nếu có điều kiện.

---

### 25.1. Tạo PVC dùng StorageClass mới

```bash
cat > pvc-retain.yaml <<'EOF'
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: data-retain
  namespace: longhorn-lab
spec:
  accessModes:
  - ReadWriteOnce
  storageClassName: longhorn-2replica-retain
  resources:
    requests:
      storage: 1Gi
EOF

kubectl apply -f pvc-retain.yaml
```

Kiểm tra:

```bash
kubectl -n longhorn-lab get pvc data-retain
kubectl get pv | grep data-retain || true
```

---

## 26. Lab 7: Xóa PVC và quan sát reclaim policy

Xóa PVC:

```bash
kubectl -n longhorn-lab delete pvc data-retain
```

Kiểm tra PV:

```bash
kubectl get pv
```

Nếu `reclaimPolicy: Retain`, PV có thể chuyển sang trạng thái `Released`.

Ý nghĩa:

- Kubernetes không tự xóa backend storage ngay.
- Admin cần xử lý thủ công.
- Phù hợp dữ liệu quan trọng cần tránh bị xóa nhầm.

So sánh:

| Reclaim Policy | Khi PVC bị xóa |
|---|---|
| `Delete` | PV/backend volume bị xóa theo |
| `Retain` | PV giữ lại, admin xử lý thủ công |

---

## 27. Lab 8: RWX volume với Longhorn

Longhorn chủ yếu được dùng như block storage cho RWO volume, nhưng cũng hỗ trợ RWX bằng cách expose Longhorn volume qua NFSv4.1 thông qua share-manager Pod.

Tạo PVC RWX:

```bash
cat > pvc-longhorn-rwx.yaml <<'EOF'
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: data-rwx
  namespace: longhorn-lab
spec:
  accessModes:
  - ReadWriteMany
  storageClassName: longhorn
  resources:
    requests:
      storage: 2Gi
EOF

kubectl apply -f pvc-longhorn-rwx.yaml
```

Kiểm tra:

```bash
kubectl -n longhorn-lab get pvc data-rwx
kubectl -n longhorn-system get pods | grep share-manager || true
```

Nếu RWX volume được attach, Longhorn sẽ tạo share-manager Pod tương ứng.

---

### 27.1. Tạo Deployment 2 replicas dùng RWX PVC

```bash
cat > deploy-longhorn-rwx.yaml <<'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-longhorn-rwx
  namespace: longhorn-lab
spec:
  replicas: 2
  selector:
    matchLabels:
      app: web-longhorn-rwx
  template:
    metadata:
      labels:
        app: web-longhorn-rwx
    spec:
      containers:
      - name: web
        image: nginx:1.27
        ports:
        - containerPort: 80
        volumeMounts:
        - name: shared-data
          mountPath: /usr/share/nginx/html
      initContainers:
      - name: init-index
        image: busybox:1.36
        command:
        - sh
        - -c
        - |
          if [ ! -f /data/index.html ]; then
            echo "<h1>Hello from Longhorn RWX volume</h1>" > /data/index.html
            echo "<p>This content is shared by multiple Pods.</p>" >> /data/index.html
          fi
        volumeMounts:
        - name: shared-data
          mountPath: /data
      volumes:
      - name: shared-data
        persistentVolumeClaim:
          claimName: data-rwx
---
apiVersion: v1
kind: Service
metadata:
  name: web-longhorn-rwx
  namespace: longhorn-lab
spec:
  type: ClusterIP
  selector:
    app: web-longhorn-rwx
  ports:
  - port: 80
    targetPort: 80
EOF

kubectl apply -f deploy-longhorn-rwx.yaml
```

Kiểm tra:

```bash
kubectl -n longhorn-lab get pod -o wide
kubectl -n longhorn-system get pods | grep share-manager
```

Test:

```bash
kubectl -n longhorn-lab run curl --image=curlimages/curl:8.10.1 -it --rm --restart=Never -- \
  curl http://web-longhorn-rwx
```

Kết quả:

```html
<h1>Hello from Longhorn RWX volume</h1>
```

Giải thích:

- `ReadWriteMany` cho phép nhiều Pod mount cùng volume.
- Longhorn dùng share-manager để cung cấp NFS endpoint.
- Các Pod mount vào NFS endpoint đó.

Lưu ý production:

RWX qua Longhorn vẫn đi qua NFS share-manager. Không nên hiểu RWX là nhiều Pod cùng ghi trực tiếp block device. Với workload cần shared filesystem lớn, cần đánh giá kỹ performance và failure domain.

---

## 28. Lab 9: Publish ứng dụng dùng Longhorn qua Ingress

Tạo Ingress cho app RWX:

```bash
cat > ingress-web-longhorn-rwx.yaml <<'EOF'
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: web-longhorn-rwx
  namespace: longhorn-lab
spec:
  ingressClassName: nginx
  rules:
  - host: longhorn-web.k8s.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: web-longhorn-rwx
            port:
              number: 80
EOF

kubectl apply -f ingress-web-longhorn-rwx.yaml
```

Thêm hosts trên máy client:

```text
<INGRESS_EXTERNAL_IP> longhorn-web.k8s.local
```

Test:

```bash
curl http://longhorn-web.k8s.local
```

---

## 29. Lab 10: Tạo snapshot thủ công

Longhorn snapshot có thể thực hiện qua UI hoặc thông qua Kubernetes VolumeSnapshot nếu đã cài snapshot CRD/controller phù hợp.

Trong bài này, ta thao tác snapshot qua UI để dễ quan sát.

Các bước:

- Vào Longhorn UI.
- Chọn volume tương ứng PVC `data-rwo` hoặc `data-rwx`.
- Chọn tab Snapshot.
- Chọn Take Snapshot.
- Đặt tên snapshot, ví dụ:

```text
before-change
```

Sau đó thay đổi dữ liệu trong Pod:

```bash
kubectl -n longhorn-lab exec -it deploy/web-longhorn-rwo -- sh -c \
  'echo "<h1>Changed content after snapshot</h1>" > /usr/share/nginx/html/index.html'
```

Kiểm tra:

```bash
kubectl -n longhorn-lab run curl --image=curlimages/curl:8.10.1 -it --rm --restart=Never -- \
  curl http://web-longhorn-rwo
```

Trong UI, có thể thấy snapshot chain.

Lưu ý:

Snapshot nằm cùng hệ thống Longhorn. Snapshot không thay thế backup ngoài cluster.

---

## 30. Snapshot khác Backup như thế nào?

| Tiêu chí | Snapshot | Backup |
|---|---|---|
| Vị trí | Trong Longhorn cluster/local storage | Ngoài cluster, ví dụ S3/NFS |
| Mục tiêu | Khôi phục nhanh trong cùng cluster | DR, phục hồi khi mất cluster |
| Phụ thuộc cluster | Có | Ít hơn |
| Chống mất toàn cluster | Không đủ | Có ý nghĩa hơn |
| Tốc độ | Nhanh | Phụ thuộc network/backup target |
| Dùng khi | Trước update app, rollback nhanh dữ liệu | DR, migration, long-term retention |

Kết luận:

Snapshot dùng để rollback nhanh.

Backup dùng để sống sót khi mất cluster/storage node nghiêm trọng.

---

## 31. Lab 11: Cấu hình backup target NFS

Trong Buổi 05, lớp học đã có NFS Server. Có thể tận dụng VM đó làm nơi lưu backup Longhorn.

Ví dụ:

| Thành phần | Giá trị |
|---|---|
| NFS Server | `10.10.10.50` |
| Export path | `/srv/longhorn-backup` |
| Backup target URL | `nfs://10.10.10.50:/srv/longhorn-backup` |

### 31.1. Cấu hình trên NFS Server

Trên VM NFS Server:

```bash
sudo mkdir -p /srv/longhorn-backup
sudo chown nobody:nogroup /srv/longhorn-backup
sudo chmod 0777 /srv/longhorn-backup
```

Cấu hình export:

```bash
echo "/srv/longhorn-backup 10.10.10.0/24(rw,sync,no_subtree_check,no_root_squash)" | sudo tee /etc/exports.d/longhorn-backup.exports
```

Apply:

```bash
sudo exportfs -rav
sudo exportfs -v
```

---

### 31.2. Kiểm tra mount NFS từ worker

Trên `worker01`:

```bash
sudo mkdir -p /mnt/test-longhorn-backup
sudo mount -t nfs4 10.10.10.50:/srv/longhorn-backup /mnt/test-longhorn-backup
df -h /mnt/test-longhorn-backup
sudo umount /mnt/test-longhorn-backup
```

Nếu mount được, Longhorn có thể dùng backup target này.

---

### 31.3. Cấu hình backup target trong Longhorn UI

Vào Longhorn UI:

```text
Setting > Backup Target
```

Nhập:

```text
nfs://10.10.10.50:/srv/longhorn-backup
```

Save.

Sau đó vào tab Backup để kiểm tra trạng thái.

---

## 32. Lab 12: Tạo backup thủ công

Trong Longhorn UI:

- Vào Volume.
- Chọn volume đang dùng.
- Chọn Create Backup.
- Đợi backup hoàn tất.
- Vào tab Backup kiểm tra.

Kiểm tra trên NFS Server:

```bash
sudo find /srv/longhorn-backup -maxdepth 3 -type f | head
```

Nếu thấy dữ liệu Longhorn backupstore, backup đã được ghi ra NFS.

---

## 33. Lab 13: Tạo recurring snapshot và backup job

Longhorn hỗ trợ recurring job để tạo snapshot/backup định kỳ.

Tạo RecurringJob snapshot mỗi 5 phút:

```bash
cat > recurring-snapshot.yaml <<'EOF'
apiVersion: longhorn.io/v1beta2
kind: RecurringJob
metadata:
  name: snapshot-every-5min
  namespace: longhorn-system
spec:
  cron: "*/5 * * * *"
  task: "snapshot"
  groups:
  - lab
  retain: 3
  concurrency: 1
EOF

kubectl apply -f recurring-snapshot.yaml
```

Tạo RecurringJob backup mỗi 10 phút:

```bash
cat > recurring-backup.yaml <<'EOF'
apiVersion: longhorn.io/v1beta2
kind: RecurringJob
metadata:
  name: backup-every-10min
  namespace: longhorn-system
spec:
  cron: "*/10 * * * *"
  task: "backup"
  groups:
  - lab
  retain: 3
  concurrency: 1
EOF

kubectl apply -f recurring-backup.yaml
```

Giải thích:

```yaml
kind: RecurringJob
```

Đây là CRD của Longhorn.

```yaml
cron: "*/5 * * * *"
```

Chạy mỗi 5 phút.

```yaml
task: "snapshot"
```

Job này tạo snapshot.

```yaml
groups:
- lab
```

Job thuộc group `lab`.

```yaml
retain: 3
```

Giữ lại 3 bản gần nhất.

```yaml
concurrency: 1
```

Chỉ chạy 1 job đồng thời.

---

### 33.1. Gán recurring job vào volume qua PVC label

Bật PVC làm source cho recurring job label:

```bash
kubectl -n longhorn-lab label pvc data-rwo recurring-job.longhorn.io/source=enabled
```

Gán group `lab`:

```bash
kubectl -n longhorn-lab label pvc data-rwo recurring-job-group.longhorn.io/lab=enabled
```

Kiểm tra PVC label:

```bash
kubectl -n longhorn-lab get pvc data-rwo --show-labels
```

Sau một thời gian, kiểm tra Longhorn UI để thấy snapshot/backup được tạo định kỳ.

---

## 34. Lab 14: Khôi phục volume từ backup

Có nhiều cách khôi phục từ backup. Trong lab, dùng UI là dễ quan sát nhất.

Các bước:

- Vào Longhorn UI.
- Vào tab Backup.
- Chọn backup của volume cần restore.
- Chọn Restore.
- Đặt tên volume mới, ví dụ:

```text
restore-data-rwo
```

Sau khi restore xong, có thể tạo PV/PVC trỏ tới volume đã restore hoặc dùng Longhorn UI để tạo workload tùy tình huống.

Trong lớp học cơ bản, mục tiêu là hiểu:

- Backup nằm ngoài cluster.
- Restore tạo lại volume.
- Sau khi có volume, có thể gắn lại vào Kubernetes workload.

---

## 35. Lab 15: Simulate lỗi node storage

> Lab này chỉ làm trong môi trường lab. Không thực hiện trên production.

Giả sử Pod đang dùng volume chạy trên `worker01`.

Kiểm tra:

```bash
kubectl -n longhorn-lab get pod -o wide
```

Cordon node:

```bash
kubectl cordon worker01
```

Drain node:

```bash
kubectl drain worker01 --ignore-daemonsets --delete-emptydir-data
```

Quan sát:

```bash
kubectl -n longhorn-lab get pod -o wide -w
kubectl -n longhorn-system get pods -o wide
```

Trong Longhorn UI, quan sát:

- Volume detach/attach.
- Replica health.
- Rebuild nếu có.
- Workload chuyển node.

Sau lab, đưa node trở lại:

```bash
kubectl uncordon worker01
```

Lưu ý:

Nếu chỉ có 2 worker và replica count là 2, khi một node down, volume có thể ở trạng thái degraded nhưng vẫn có thể hoạt động nếu còn replica khỏe. Tuy nhiên khả năng chịu lỗi tiếp theo sẽ giảm mạnh.

---

## 36. Lab 16: Quan sát replica rebuild

Để quan sát rebuild rõ hơn, cần ít nhất 3 storage node. Với 2 worker, rebuild có thể không có đủ node để tạo replica thay thế.

Trên cluster 3 worker/storage node:

- Tạo PVC với `numberOfReplicas: "3"`.
- Chạy app ghi dữ liệu.
- Tắt một worker hoặc cordon/drain.
- Bật lại node.
- Quan sát Longhorn rebuild replica.

Lệnh quan sát:

```bash
kubectl -n longhorn-system get volumes.longhorn.io
kubectl -n longhorn-system get replicas.longhorn.io
kubectl -n longhorn-system get engines.longhorn.io
```

Xem chi tiết volume:

```bash
kubectl -n longhorn-system describe volumes.longhorn.io <LONGHORN_VOLUME_NAME>
```

---

## 37. Các CRD quan trọng của Longhorn

Longhorn tạo nhiều CRD trong cluster.

Kiểm tra:

```bash
kubectl get crd | grep longhorn
```

Một số CRD thường gặp:

| CRD | Ý nghĩa |
|---|---|
| `volumes.longhorn.io` | Longhorn volume |
| `replicas.longhorn.io` | Replica của volume |
| `engines.longhorn.io` | Engine xử lý I/O |
| `nodes.longhorn.io` | Node trong Longhorn |
| `settings.longhorn.io` | Cấu hình hệ thống |
| `recurringjobs.longhorn.io` | Job snapshot/backup định kỳ |
| `backups.longhorn.io` | Backup object |
| `backuptargets.longhorn.io` | Backup target |
| `sharemanagers.longhorn.io` | Share manager cho RWX volume |

Ví dụ:

```bash
kubectl -n longhorn-system get volumes.longhorn.io
kubectl -n longhorn-system get nodes.longhorn.io
kubectl -n longhorn-system get recurringjobs.longhorn.io
```

---

## 38. Monitoring Longhorn ở mức cơ bản

Longhorn có endpoint metrics Prometheus.

Kiểm tra Service:

```bash
kubectl -n longhorn-system get svc
```

Thông thường có Service liên quan Longhorn backend/manager.

Nếu đã học Prometheus/Grafana ở module sau, có thể scrape metrics của Longhorn.

Trong module này, chỉ cần biết:

- Longhorn có metrics.
- Longhorn UI có trạng thái volume/node/disk.
- Kubernetes events/logs vẫn là nguồn troubleshooting quan trọng.
- Support bundle có thể dùng khi cần phân tích sâu.

Các lệnh cơ bản:

```bash
kubectl -n longhorn-system get pods
kubectl -n longhorn-system logs daemonset/longhorn-manager
kubectl -n longhorn-system get events --sort-by=.lastTimestamp
```

---

## 39. Troubleshooting Longhorn

### 39.1. PVC bị Pending

Triệu chứng:

```bash
kubectl -n longhorn-lab get pvc
```

PVC ở trạng thái:

```text
Pending
```

Kiểm tra:

```bash
kubectl -n longhorn-lab describe pvc <PVC_NAME>
kubectl get sc
kubectl -n longhorn-system get pods
kubectl -n longhorn-system logs deploy/csi-provisioner
```

Nguyên nhân thường gặp:

- StorageClass không tồn tại.
- Longhorn CSI chưa chạy.
- Không đủ node/disk để tạo replica.
- Longhorn webhook lỗi.
- Node không Ready.
- Disk bị disable scheduling.

---

### 39.2. Pod bị kẹt ContainerCreating

Kiểm tra:

```bash
kubectl -n longhorn-lab describe pod <POD_NAME>
```

Nếu thấy lỗi mount/attach volume, kiểm tra tiếp:

```bash
kubectl -n longhorn-system get pods
kubectl -n longhorn-system logs daemonset/longhorn-manager
kubectl -n longhorn-system get volumes.longhorn.io
```

Nguyên nhân thường gặp:

- `open-iscsi` chưa cài.
- `iscsid` chưa chạy.
- `iscsi_tcp` chưa load.
- Volume chưa attach được.
- Node network giữa các Longhorn component có vấn đề.

---

### 39.3. Longhorn Pod không chạy trên node

Kiểm tra:

```bash
kubectl -n longhorn-system get pods -o wide
kubectl describe node <NODE_NAME>
```

Nguyên nhân:

- Node có taint.
- Node không đủ CPU/RAM.
- Pod Security Admission chặn privileged Pod.
- Node selector/toleration cấu hình sai.

Longhorn cần một số component chạy privileged để thao tác storage trên host. Nếu namespace bị enforce Pod Security `restricted`, Longhorn có thể không chạy.

---

### 39.4. Volume Degraded

Trong Longhorn UI, volume có thể báo:

```text
Degraded
```

Ý nghĩa:

- Volume vẫn hoạt động nhưng thiếu replica khỏe.
- Một replica bị lỗi, đang rebuild hoặc không thể rebuild.
- Mức HA đang bị giảm.

Kiểm tra:

```bash
kubectl -n longhorn-system get volumes.longhorn.io
kubectl -n longhorn-system describe volumes.longhorn.io <VOLUME_NAME>
kubectl -n longhorn-system get replicas.longhorn.io
```

Nguyên nhân:

- Một node down.
- Một disk full.
- Replica process lỗi.
- Network giữa node không ổn định.
- Không đủ node để rebuild replica mới.

---

### 39.5. Disk full

Kiểm tra trên node:

```bash
df -h /var/lib/longhorn
sudo du -sh /var/lib/longhorn/*
```

Kiểm tra trong UI:

- Node.
- Disk usage.
- Volume actual size.
- Snapshot usage.

Nguyên nhân thường gặp:

- Tạo quá nhiều snapshot.
- Backup chưa cleanup.
- Over-provision quá cao.
- Workload ghi dữ liệu nhiều hơn dự kiến.
- Replica count cao làm nhân dung lượng thực tế.

Cách xử lý:

- Xóa snapshot không cần thiết.
- Purge snapshot.
- Tăng disk.
- Thêm node/disk.
- Điều chỉnh `storageMinimalAvailablePercentage`.
- Thiết lập recurring job cleanup hợp lý.

---

### 39.6. Không truy cập được Longhorn UI

Kiểm tra Ingress:

```bash
kubectl -n longhorn-system get ingress
kubectl -n longhorn-system describe ingress longhorn-ui
kubectl -n longhorn-system get svc longhorn-frontend
kubectl -n ingress-nginx logs deploy/ingress-nginx-controller
```

Nguyên nhân:

- Sai DNS/hosts.
- Sai Ingress external IP.
- Sai `ingressClassName`.
- Basic auth Secret sai.
- Service backend không đúng.
- Ingress Controller chưa chạy.

---

## 40. Best practices khi dùng Longhorn

### 40.1. Không dùng OS disk cho production nếu có thể

Production nên có disk riêng cho Longhorn.

Không nên để dữ liệu Longhorn chung với root filesystem của OS.

---

### 40.2. Tối thiểu 3 storage node nếu muốn HA đúng nghĩa

Với replica count 3, cần ít nhất 3 node/disk failure domain khác nhau.

Nếu chỉ có 2 worker, dùng replica count 2 là hợp lý cho lab nhưng không lý tưởng cho production.

---

### 40.3. Tách control-plane và storage workload

Trong production:

- Control-plane node nên tập trung chạy control plane.
- Worker/storage node nên chạy workload và Longhorn storage.
- Không nên cho control-plane node gánh storage workload trừ khi có thiết kế rõ ràng.

---

### 40.4. Luôn có backup target ngoài cluster

Snapshot không thay thế backup.

Backup target nên nằm ngoài cluster, ví dụ:

- NFS server ngoài cluster.
- MinIO/S3-compatible storage ngoài cluster.
- Object storage của cloud.

---

### 40.5. Theo dõi dung lượng thực tế

Longhorn có thể over-provision.

Over-provision giúp linh hoạt nhưng cũng có nguy cơ disk full.

Cần theo dõi:

- Disk usage.
- Snapshot usage.
- Replica count.
- Actual size.
- Backup retention.

---

### 40.6. Cẩn thận với workload ghi nhiều

Các workload như database, logging, monitoring, registry, object storage cần đánh giá kỹ:

- IOPS.
- Latency.
- Network bandwidth giữa node.
- Replica count.
- Disk type.
- Backup window.
- Snapshot chain.

Longhorn phù hợp nhiều workload stateful, nhưng không có nghĩa là mọi workload ghi nặng đều nên đặt lên Longhorn mà không benchmark.

---

## 41. So sánh nhanh Longhorn với các lựa chọn storage khác

| Giải pháp | Loại storage | Phù hợp |
|---|---|---|
| NFS CSI | File storage | Lab, shared file đơn giản |
| Longhorn | Distributed block storage | On-prem Kubernetes, stateful app vừa và nhỏ, production-like |
| Ceph RBD | Distributed block storage | Production lớn, cần scale mạnh, vận hành phức tạp hơn |
| Local PV | Local disk | Performance tốt nhưng kém linh hoạt |
| Cloud Disk CSI | Cloud block storage | AWS/Azure/GCP |
| vSphere CSI | Enterprise block/file tùy backend | Kubernetes chạy trên VMware |

Thông điệp chính:

- NFS CSI dễ học và dễ dùng.
- Longhorn phù hợp để dạy distributed storage cho Kubernetes on-prem.
- Ceph mạnh hơn ở quy mô lớn nhưng phức tạp hơn.
- Cloud CSI phụ thuộc cloud provider.
- vSphere CSI phù hợp khi hạ tầng bên dưới là VMware.

---

## 42. Bài lab tổng hợp

### 42.1. Yêu cầu

Triển khai một ứng dụng web có dữ liệu lưu trên Longhorn, publish qua Ingress, có snapshot và backup.

Ứng dụng cần có:

- Namespace riêng.
- PVC Longhorn RWO.
- Deployment NGINX 1 replica.
- Init container tạo file HTML.
- Service ClusterIP.
- Ingress HTTP.
- Snapshot thủ công.
- Backup thủ công.
- Mở rộng PVC từ 2Gi lên 4Gi.
- Kiểm tra dữ liệu vẫn tồn tại sau khi xóa Pod.
- Kiểm tra volume trong Longhorn UI.

---

### 42.2. Tạo namespace

```bash
kubectl create namespace final-longhorn-lab
```

---

### 42.3. Tạo PVC

```bash
cat > final-pvc.yaml <<'EOF'
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: web-data
  namespace: final-longhorn-lab
spec:
  accessModes:
  - ReadWriteOnce
  storageClassName: longhorn
  resources:
    requests:
      storage: 2Gi
EOF

kubectl apply -f final-pvc.yaml
```

---

### 42.4. Tạo Deployment và Service

```bash
cat > final-web.yaml <<'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: final-web
  namespace: final-longhorn-lab
spec:
  replicas: 1
  selector:
    matchLabels:
      app: final-web
  template:
    metadata:
      labels:
        app: final-web
    spec:
      initContainers:
      - name: init-content
        image: busybox:1.36
        command:
        - sh
        - -c
        - |
          if [ ! -f /data/index.html ]; then
            echo "<h1>Final Longhorn Lab</h1>" > /data/index.html
            echo "<p>This page is stored on a Longhorn persistent volume.</p>" >> /data/index.html
            echo "<p>Created at $(date)</p>" >> /data/index.html
          fi
        volumeMounts:
        - name: web-data
          mountPath: /data
      containers:
      - name: nginx
        image: nginx:1.27
        ports:
        - containerPort: 80
        volumeMounts:
        - name: web-data
          mountPath: /usr/share/nginx/html
        resources:
          requests:
            cpu: "50m"
            memory: "64Mi"
          limits:
            cpu: "200m"
            memory: "128Mi"
      volumes:
      - name: web-data
        persistentVolumeClaim:
          claimName: web-data
---
apiVersion: v1
kind: Service
metadata:
  name: final-web
  namespace: final-longhorn-lab
spec:
  type: ClusterIP
  selector:
    app: final-web
  ports:
  - port: 80
    targetPort: 80
EOF

kubectl apply -f final-web.yaml
```

---

### 42.5. Tạo Ingress

```bash
cat > final-ingress.yaml <<'EOF'
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: final-web
  namespace: final-longhorn-lab
spec:
  ingressClassName: nginx
  rules:
  - host: final-longhorn.k8s.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: final-web
            port:
              number: 80
EOF

kubectl apply -f final-ingress.yaml
```

---

### 42.6. Kiểm tra ứng dụng

```bash
kubectl -n final-longhorn-lab get pod,pvc,svc,ingress
```

Từ máy client:

```bash
curl http://final-longhorn.k8s.local
```

Hoặc từ trong cluster:

```bash
kubectl -n final-longhorn-lab run curl --image=curlimages/curl:8.10.1 -it --rm --restart=Never -- \
  curl http://final-web
```

---

### 42.7. Kiểm tra volume trong Longhorn

Vào Longhorn UI:

- Volume.
- Tìm volume tương ứng PVC `web-data`.
- Kiểm tra:
  - State.
  - Health.
  - Size.
  - Replicas.
  - Attached node.
  - Snapshot.
  - Backup.

---

### 42.8. Xóa Pod và kiểm tra dữ liệu

```bash
kubectl -n final-longhorn-lab delete pod -l app=final-web
```

Đợi Pod mới chạy lại:

```bash
kubectl -n final-longhorn-lab get pod -w
```

Test lại:

```bash
curl http://final-longhorn.k8s.local
```

Dữ liệu vẫn còn.

---

### 42.9. Mở rộng PVC

```bash
kubectl -n final-longhorn-lab patch pvc web-data \
  -p '{"spec":{"resources":{"requests":{"storage":"4Gi"}}}}'
```

Kiểm tra:

```bash
kubectl -n final-longhorn-lab get pvc web-data
kubectl -n final-longhorn-lab exec -it deploy/final-web -- df -h /usr/share/nginx/html
```

---

### 42.10. Snapshot và backup

Trong Longhorn UI:

- Tạo snapshot tên `final-before-change`.
- Tạo backup từ snapshot.
- Xem backup ở tab Backup.

Sau đó thay đổi nội dung:

```bash
kubectl -n final-longhorn-lab exec -it deploy/final-web -- sh -c \
  'echo "<h1>Content changed after backup</h1>" > /usr/share/nginx/html/index.html'
```

Test:

```bash
curl http://final-longhorn.k8s.local
```

Mục tiêu:

Học viên thấy rõ dữ liệu trong Longhorn volume có thể được snapshot/backup, không phụ thuộc vòng đời Pod.

---

## 43. Câu hỏi kiểm tra cuối module

### Câu 1

Longhorn giải quyết bài toán gì trong Kubernetes on-premise?

Gợi ý trả lời:

Longhorn cung cấp distributed block storage cho Kubernetes, giúp tạo PV/PVC động từ disk local của các node, hỗ trợ replica, snapshot, backup, restore và volume expansion mà không cần cloud provider storage.

---

### Câu 2

Vì sao cluster chỉ có 2 worker không nên dùng `numberOfReplicas: "3"`?

Gợi ý trả lời:

Vì Longhorn cần đặt các replica trên các node/disk khác nhau. Nếu chỉ có 2 storage node, việc yêu cầu 3 replicas có thể khiến volume không đạt trạng thái healthy hoặc phải dùng soft anti-affinity không phù hợp. Với 2 worker, nên dùng replica count 2 cho lab hoặc bổ sung storage node thứ ba.

---

### Câu 3

Snapshot Longhorn có thay thế backup không?

Gợi ý trả lời:

Không. Snapshot nằm trong Longhorn cluster và phù hợp rollback nhanh. Backup được lưu ra backup target ngoài cluster như NFS/S3, phù hợp disaster recovery.

---

### Câu 4

Vì sao Longhorn cần `open-iscsi` trên node?

Gợi ý trả lời:

Longhorn dùng iSCSI trên host để attach persistent volume vào node Kubernetes. Nếu thiếu `open-iscsi` hoặc `iscsid` không chạy, Pod có thể bị lỗi attach/mount volume.

---

### Câu 5

RWX volume trong Longhorn hoạt động như thế nào?

Gợi ý trả lời:

Longhorn expose volume thông qua NFSv4.1 server chạy trong share-manager Pod. Các Pod mount RWX volume thông qua NFS endpoint đó.

---

### Câu 6

Khi PVC dùng `reclaimPolicy: Delete` bị xóa, điều gì xảy ra?

Gợi ý trả lời:

Kubernetes sẽ xóa PV tương ứng và Longhorn volume backend cũng bị xóa theo.

---

### Câu 7

Khi PVC dùng `reclaimPolicy: Retain` bị xóa, điều gì xảy ra?

Gợi ý trả lời:

PV/backend data không bị xóa tự động. PV có thể chuyển sang trạng thái Released và admin cần xử lý thủ công.

---

### Câu 8

Vì sao Longhorn không nên dùng chung root disk trong production?

Gợi ý trả lời:

Vì dữ liệu Longhorn có thể làm đầy root filesystem, ảnh hưởng OS và kubelet. Dedicated disk giúp tách dữ liệu, dễ quản lý dung lượng và gần production hơn.

---

### Câu 9

Các thành phần nào của Longhorn thường cần kiểm tra khi PVC bị Pending?

Gợi ý trả lời:

StorageClass, Longhorn CSI provisioner, Longhorn Manager, node/disk availability, events của PVC, logs của CSI provisioner và trạng thái Longhorn UI.

---

### Câu 10

Vì sao backup target nên nằm ngoài cluster?

Gợi ý trả lời:

Nếu backup target nằm trong cùng cluster/storage domain, khi cluster hoặc storage hỏng nghiêm trọng thì backup cũng có thể mất. Backup target ngoài cluster giúp phục hồi trong kịch bản disaster recovery thật sự.

---

## 44. Checklist triển khai nhanh

### Trên các storage node

```bash
sudo apt update
sudo apt install -y open-iscsi nfs-common cryptsetup dmsetup curl jq util-linux
sudo systemctl enable --now iscsid
sudo modprobe iscsi_tcp
echo iscsi_tcp | sudo tee /etc/modules-load.d/iscsi_tcp.conf
```

Nếu có disk riêng:

```bash
sudo parted /dev/sdb --script mklabel gpt
sudo mkfs.ext4 -F /dev/sdb
sudo mkdir -p /var/lib/longhorn
echo "UUID=$(sudo blkid -s UUID -o value /dev/sdb) /var/lib/longhorn ext4 defaults 0 2" | sudo tee -a /etc/fstab
sudo mount -a
df -h /var/lib/longhorn
```

### Cài Longhorn

```bash
helm repo add longhorn https://charts.longhorn.io
helm repo update

helm install longhorn longhorn/longhorn \
  --namespace longhorn-system \
  --create-namespace \
  --version 1.11.1 \
  -f longhorn-values-lab-2worker.yaml
```

### Kiểm tra

```bash
kubectl -n longhorn-system get pods
kubectl get sc
kubectl -n longhorn-system get volumes.longhorn.io
```

### Tạo PVC

```bash
kubectl create namespace longhorn-lab

kubectl -n longhorn-lab apply -f pvc-longhorn-rwo.yaml
kubectl -n longhorn-lab get pvc
```

---

## 45. Tổng kết module

Trong module này, đã học cách triển khai Longhorn như một lớp distributed block storage cho Kubernetes on-premise.

Các điểm cần nhớ:

- Longhorn giúp Kubernetes on-premise có dynamic block storage mà không cần cloud provider.
- Longhorn tích hợp với Kubernetes thông qua CSI driver.
- PVC dùng StorageClass `longhorn` sẽ tạo Longhorn volume tự động.
- Longhorn volume có thể có nhiều replicas trên nhiều node.
- Với lab chỉ có 2 worker, nên dùng replica count 2.
- Với production, nên có tối thiểu 3 storage node nếu muốn replica count 3.
- Snapshot không thay thế backup.
- Backup target nên nằm ngoài cluster.
- Longhorn UI giúp quan sát volume, replica, node, disk, snapshot, backup.
- Troubleshooting Longhorn cần kết hợp cả Kubernetes events/logs và Longhorn UI/CRD.
- Longhorn rất phù hợp để dạy storage production-like trong Kubernetes on-premise.

---

## 46. Tài liệu tham khảo chính thức

- Longhorn Documentation: https://longhorn.io/docs/latest/
- Longhorn Installation Requirements: https://longhorn.io/docs/latest/deploy/install/
- Install Longhorn with Helm: https://longhorn.io/docs/latest/deploy/install/install-with-helm/
- Longhorn StorageClass Parameters: https://longhorn.io/docs/latest/references/storage-class-parameters/
- Longhorn RWX Volume: https://longhorn.io/docs/latest/nodes-and-volumes/volumes/rwx-volumes/
- Longhorn Volume Expansion: https://longhorn.io/docs/latest/nodes-and-volumes/volumes/expansion/
- Longhorn Recurring Snapshots and Backups: https://longhorn.io/docs/latest/snapshots-and-backups/scheduling-backups-and-snapshots/
- Longhorn Backup and Restore: https://longhorn.io/docs/latest/snapshots-and-backups/backup-and-restore/
