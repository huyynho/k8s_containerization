# BUỔI 12: BACKUP, RESTORE VÀ DISASTER RECOVERY TRONG KUBERNETES v1.35

## 1. Mục tiêu buổi học

Sau buổi học này, cần hiểu và thực hành được các nội dung sau:

- Hiểu vì sao backup Kubernetes không chỉ là backup VM hoặc backup filesystem của node.
- Phân biệt được các lớp dữ liệu cần bảo vệ trong một cụm Kubernetes.
- Hiểu vai trò của `etcd` trong việc lưu trữ trạng thái cluster.
- Thực hiện backup `etcd` snapshot trên cụm Kubernetes được bootstrap bằng `kubeadm`.
- Hiểu sự khác nhau giữa backup manifest, backup `etcd`, backup application và backup dữ liệu trong PersistentVolume.
- Sử dụng Velero để backup và restore tài nguyên Kubernetes ở cấp namespace/application.
- Hiểu cách xử lý dữ liệu PersistentVolume trong môi trường on-premise dùng NFS CSI.
- Thực hành tình huống xóa namespace, mất workload, mất PVC và khôi phục lại ứng dụng.
- Xây dựng tư duy Disaster Recovery cho Kubernetes on-premise.

Buổi này nối tiếp trực tiếp từ các buổi trước:

- Buổi 01: Kiến trúc Kubernetes.
- Buổi 02: Triển khai cluster bằng `kubeadm`, `containerd`, Calico.
- Buổi 03: Workload: Pod, Deployment, StatefulSet, Job, CronJob.
- Buổi 04: Networking và Service.
- Buổi 05: ConfigMap, Secret, Volume, PV, PVC, StorageClass, CSI NFS.
- Buổi 06: Helm, MetalLB, Ingress NGINX Controller.
- Buổi 07: Application Lifecycle, Health Check, Rolling Update, Rollback, Self-healing.
- Buổi 08: Resource Management và Scheduling cơ bản.
- Buổi 09: Kubernetes Security cơ bản.
- Buổi 10: Observability và Troubleshooting.
- Buổi 11: Autoscaling.

Trong buổi 12, sẽ bắt đầu nhìn Kubernetes dưới góc độ vận hành dài hạn: nếu có sự cố thì khôi phục như thế nào.

---

## 2. Mô hình lab sử dụng trong buổi học

Mô hình cluster vẫn bám theo các buổi trước.

| Thành phần | Hostname ví dụ | Vai trò | Hệ điều hành |
|---|---|---|---|
| Master Node | `k8s-master-01` | Control Plane | Ubuntu Server 24.04 |
| Worker Node 1 | `k8s-worker-01` | Worker | Ubuntu Server 24.04 |
| Worker Node 2 | `k8s-worker-02` | Worker | Ubuntu Server 24.04 |
| NFS Server | `nfs-server-01` | NFS Storage + Backup Repository lab | Ubuntu Server 24.04 |

Cluster đã có sẵn:

- Kubernetes v1.35.
- Bootstrap bằng `kubeadm`.
- Container runtime: `containerd`.
- CNI: Calico.
- Storage: NFS CSI từ Buổi 05.
- Helm từ Buổi 06.
- MetalLB và Ingress NGINX Controller từ Buổi 06.
- Metrics Server từ Buổi 10/11.

Trong buổi này, VM `nfs-server-01` sẽ được dùng thêm cho mục đích backup lab:

- Lưu file snapshot của `etcd`.
- Lưu các file manifest export từ cluster.
- Có thể chạy S3-compatible object storage dạng lab để Velero sử dụng.

Trong production, backup repository nên nằm ngoài failure domain của cluster. Ví dụ:

- Object Storage ngoài cluster.
- S3-compatible storage độc lập.
- NAS/SAN độc lập có cơ chế snapshot/replication.
- Backup appliance riêng.
- Cloud object storage.

Không nên chỉ lưu backup bên trong chính cluster đang cần bảo vệ.

---

## 3. Kiến thức cần mở khóa trong buổi này

### 3.1. Kubernetes cần backup những gì?

Khi nói “backup Kubernetes”, nhiều người dễ hiểu nhầm rằng chỉ cần backup VM node là đủ. Thực tế, Kubernetes có nhiều lớp dữ liệu khác nhau.

Các lớp dữ liệu quan trọng gồm:

| Lớp dữ liệu | Ví dụ | Mục đích backup |
|---|---|---|
| Cluster state | Tất cả object lưu trong `etcd` | Khôi phục trạng thái toàn cụm |
| Manifest/YAML | Deployment, Service, Ingress, ConfigMap, Secret, PVC | Khôi phục theo hướng declarative |
| Application data | Dữ liệu bên trong PV/PVC | Khôi phục dữ liệu ứng dụng |
| Node configuration | kubelet, containerd, OS, network config | Rebuild node nhanh hơn |
| Certificate / kubeconfig | `/etc/kubernetes/pki`, kubeconfig admin | Duy trì khả năng quản trị cluster |
| External dependency | DNS, Load Balancer, NFS, Object Storage | Đảm bảo app hoạt động lại sau restore |

Trong cụm kubeadm 1 master + 2 worker, lớp quan trọng nhất ở cấp control plane là `etcd`.

---

### 3.2. etcd là gì trong Kubernetes?

Trong kiến trúc Kubernetes, `etcd` là distributed key-value store được control plane sử dụng để lưu toàn bộ trạng thái cluster.

Các object như sau đều được lưu trong `etcd`:

- Namespace.
- Pod.
- ReplicaSet.
- Deployment.
- DaemonSet.
- StatefulSet.
- Job.
- CronJob.
- Service.
- EndpointSlice.
- ConfigMap.
- Secret.
- PV.
- PVC.
- StorageClass.
- Ingress.
- Role.
- RoleBinding.
- ServiceAccount.
- NetworkPolicy.
- HPA.

Nói đơn giản:

```text
Nếu Kubernetes API là cửa ngõ quản lý cluster,
thì etcd là nơi lưu trí nhớ của cluster.
```

Vì vậy, mất `etcd` có thể dẫn tới mất toàn bộ trạng thái cluster.

---

### 3.3. etcd backup khác gì Velero backup?

Cần phân biệt rõ hai loại backup thường gặp:

| Tiêu chí | etcd snapshot | Velero backup |
|---|---|---|
| Phạm vi | Toàn bộ state của cluster | Theo namespace, label, resource hoặc toàn cluster |
| Cấp độ | Cluster-level | Application-level / resource-level |
| Phù hợp khi | Mất control plane, mất toàn bộ cluster state | Mất app, mất namespace, migrate app |
| Dễ restore chọn lọc | Không | Có |
| Có backup PV data không | Không trực tiếp | Có thể, nếu cấu hình volume backup/snapshot |
| Có phù hợp migrate app giữa cluster không | Không lý tưởng | Phù hợp hơn |
| Rủi ro restore | Cao hơn, ảnh hưởng toàn cluster | Có thể kiểm soát theo phạm vi |

Tư duy thực tế:

- `etcd snapshot` dùng cho disaster recovery cấp cluster.
- Velero dùng cho backup/restore ứng dụng, namespace, workload, PVC data.
- Git repository dùng để lưu manifest nguồn theo hướng GitOps/declarative.
- Backup storage backend dùng để lưu dữ liệu backup ngoài cluster.

---

## 4. Chiến lược backup Kubernetes trong môi trường on-premise

### 4.1. Không có một backup duy nhất giải quyết mọi thứ

Trong production, không nên chỉ phụ thuộc vào một cách backup.

Một chiến lược hợp lý thường gồm nhiều lớp:

```text
Layer 1: Manifest/Git backup
Layer 2: etcd snapshot
Layer 3: Application backup bằng Velero
Layer 4: PersistentVolume data backup
Layer 5: Backup external dependency
Layer 6: Restore drill định kỳ
```

Trong đó:

- Manifest/Git giúp rebuild app rõ ràng.
- etcd snapshot giúp cứu cluster state.
- Velero giúp restore app/namespace/PVC linh hoạt.
- PV backup giúp bảo vệ dữ liệu thật của ứng dụng.
- Restore drill giúp kiểm tra backup có dùng được thật hay không.

Backup không được test restore thì chưa thể xem là backup đáng tin cậy.

---

### 4.2. RPO và RTO

Hai khái niệm quan trọng trong Disaster Recovery:

| Khái niệm | Ý nghĩa | Ví dụ |
|---|---|---|
| RPO | Recovery Point Objective | Chấp nhận mất tối đa bao nhiêu dữ liệu |
| RTO | Recovery Time Objective | Cần khôi phục hệ thống trong bao lâu |

Ví dụ:

```text
RPO = 1 giờ
```

Nghĩa là doanh nghiệp chấp nhận mất tối đa dữ liệu trong vòng 1 giờ gần nhất.

```text
RTO = 4 giờ
```

Nghĩa là sau sự cố, hệ thống cần được khôi phục trong vòng 4 giờ.

Với Kubernetes:

- `etcd snapshot` mỗi 6 giờ có thể chưa đủ cho app thay đổi liên tục.
- Backup PV mỗi ngày có thể không đủ cho database transaction cao.
- Backup namespace bằng Velero mỗi ngày phù hợp với app ít thay đổi.
- App database cần backup riêng ở tầng database để đảm bảo tính nhất quán.

---

## 5. Backup manifest thủ công bằng kubectl

### 5.1. Vì sao cần backup manifest?

Manifest là cách mô tả trạng thái mong muốn của ứng dụng.

Ví dụ:

- Cần bao nhiêu Pod?
- Dùng image nào?
- Service expose port nào?
- Ingress dùng hostname nào?
- PVC cần bao nhiêu dung lượng?
- ConfigMap và Secret chứa cấu hình gì?

Nếu có manifest chuẩn, ta có thể apply lại để tái tạo ứng dụng.

Tuy nhiên, cần hiểu rõ:

```text
kubectl get -o yaml không phải lúc nào cũng tạo ra manifest sạch như file nguồn ban đầu.
```

Vì YAML export từ cluster thường chứa nhiều field runtime như:

- `metadata.creationTimestamp`
- `metadata.resourceVersion`
- `metadata.uid`
- `metadata.managedFields`
- `status`

Các field này không nên dùng lại khi viết manifest nguồn.

---

### 5.2. Tạo namespace demo

```bash
kubectl create namespace backup-demo
```

Kiểm tra:

```bash
kubectl get ns backup-demo
```

---

### 5.3. Tạo ứng dụng demo có ConfigMap, Secret, PVC, Deployment, Service

Tạo file:

```bash
mkdir -p ~/k8s-buoi-12
cd ~/k8s-buoi-12
nano app-demo.yaml
```

Nội dung:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: web-config
  namespace: backup-demo
data:
  APP_NAME: "Kubernetes Backup Demo"
  APP_ENV: "lab"
---
apiVersion: v1
kind: Secret
metadata:
  name: web-secret
  namespace: backup-demo
type: Opaque
stringData:
  DB_USERNAME: "demo_user"
  DB_PASSWORD: "DemoPassword123!"
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: web-content-pvc
  namespace: backup-demo
spec:
  accessModes:
    - ReadWriteMany
  storageClassName: nfs-csi
  resources:
    requests:
      storage: 1Gi
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-backup-demo
  namespace: backup-demo
  labels:
    app: web-backup-demo
spec:
  replicas: 2
  selector:
    matchLabels:
      app: web-backup-demo
  template:
    metadata:
      labels:
        app: web-backup-demo
    spec:
      containers:
        - name: nginx
          image: nginx:1.27-alpine
          ports:
            - containerPort: 80
          env:
            - name: APP_NAME
              valueFrom:
                configMapKeyRef:
                  name: web-config
                  key: APP_NAME
            - name: DB_USERNAME
              valueFrom:
                secretKeyRef:
                  name: web-secret
                  key: DB_USERNAME
          volumeMounts:
            - name: web-content
              mountPath: /usr/share/nginx/html
          resources:
            requests:
              cpu: 50m
              memory: 64Mi
            limits:
              cpu: 200m
              memory: 128Mi
      volumes:
        - name: web-content
          persistentVolumeClaim:
            claimName: web-content-pvc
---
apiVersion: v1
kind: Service
metadata:
  name: web-backup-demo-svc
  namespace: backup-demo
spec:
  type: ClusterIP
  selector:
    app: web-backup-demo
  ports:
    - name: http
      port: 80
      targetPort: 80
```

Apply:

```bash
kubectl apply -f app-demo.yaml
```

Kiểm tra:

```bash
kubectl get all -n backup-demo
kubectl get pvc -n backup-demo
```

---

### 5.4. Giải thích các phần quan trọng trong manifest

#### ConfigMap

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: web-config
  namespace: backup-demo
data:
  APP_NAME: "Kubernetes Backup Demo"
  APP_ENV: "lab"
```

Ý nghĩa:

- `kind: ConfigMap`: định nghĩa object lưu cấu hình dạng key-value.
- `namespace: backup-demo`: ConfigMap nằm trong namespace demo.
- `data`: chứa các cấu hình không nhạy cảm.

ConfigMap là object Kubernetes, vì vậy nó sẽ được lưu trong `etcd` và có thể được backup bằng Velero.

#### Secret

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: web-secret
  namespace: backup-demo
type: Opaque
stringData:
  DB_USERNAME: "demo_user"
  DB_PASSWORD: "DemoPassword123!"
```

Ý nghĩa:

- `kind: Secret`: định nghĩa object lưu dữ liệu nhạy cảm.
- `stringData`: cho phép nhập dữ liệu dạng plain text, Kubernetes sẽ chuyển sang `data` dạng base64 khi lưu.
- `type: Opaque`: loại Secret generic.

Lưu ý rất quan trọng:

```text
Base64 không phải encryption.
```

Trong production, Secret cần được bảo vệ bằng RBAC, encryption at rest cho etcd và quản lý quyền truy cập chặt chẽ.

#### PVC

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: web-content-pvc
  namespace: backup-demo
spec:
  accessModes:
    - ReadWriteMany
  storageClassName: nfs-csi
  resources:
    requests:
      storage: 1Gi
```

Ý nghĩa:

- PVC yêu cầu một volume có dung lượng `1Gi`.
- `storageClassName: nfs-csi` dùng StorageClass đã học ở Buổi 05.
- `ReadWriteMany` phù hợp NFS vì nhiều Pod có thể mount cùng volume.

#### Deployment

```yaml
spec:
  replicas: 2
```

Deployment tạo 2 Pod replica.

```yaml
volumeMounts:
  - name: web-content
    mountPath: /usr/share/nginx/html
```

Mount volume vào thư mục web root của Nginx.

```yaml
volumes:
  - name: web-content
    persistentVolumeClaim:
      claimName: web-content-pvc
```

Gắn PVC `web-content-pvc` vào Pod.

---

### 5.5. Ghi dữ liệu vào PVC

Chọn một Pod:

```bash
POD_NAME=$(kubectl get pod -n backup-demo -l app=web-backup-demo -o jsonpath='{.items[0].metadata.name}')
```

Ghi file vào PVC:

```bash
kubectl exec -n backup-demo $POD_NAME -- sh -c 'echo "Hello from PVC before backup" > /usr/share/nginx/html/index.html'
```

Kiểm tra từ Service:

```bash
kubectl run curl-test -n backup-demo --image=curlimages/curl:8.10.1 --rm -it --restart=Never -- \
  curl http://web-backup-demo-svc.backup-demo.svc.cluster.local
```

Kết quả kỳ vọng:

```text
Hello from PVC before backup
```

---

### 5.6. Export manifest từ namespace

Tạo thư mục backup trên master:

```bash
mkdir -p ~/k8s-backups/manual/backup-demo
```

Export một số resource chính:

```bash
kubectl get configmap -n backup-demo -o yaml > ~/k8s-backups/manual/backup-demo/configmaps.yaml
kubectl get secret -n backup-demo -o yaml > ~/k8s-backups/manual/backup-demo/secrets.yaml
kubectl get pvc -n backup-demo -o yaml > ~/k8s-backups/manual/backup-demo/pvcs.yaml
kubectl get deployment -n backup-demo -o yaml > ~/k8s-backups/manual/backup-demo/deployments.yaml
kubectl get service -n backup-demo -o yaml > ~/k8s-backups/manual/backup-demo/services.yaml
```

Hoặc export toàn bộ namespace:

```bash
kubectl get all,configmap,secret,pvc,ingress,hpa,networkpolicy -n backup-demo -o yaml \
  > ~/k8s-backups/manual/backup-demo/all-resources.yaml
```

Lưu ý:

- Cách này hữu ích để quan sát hoặc backup tạm thời.
- Không thay thế GitOps hoặc manifest chuẩn.
- Không backup được dữ liệu thật bên trong PVC.

---

## 6. Backup etcd snapshot

### 6.1. Xác định etcd static Pod

Trên master node:

```bash
kubectl get pods -n kube-system -l component=etcd -o wide
```

Kết quả ví dụ:

```text
NAME                    READY   STATUS    RESTARTS   AGE   IP
etcd-k8s-master-01      1/1     Running   0          10d   10.10.10.11
```

Với kubeadm, `etcd` thường chạy dạng static Pod trên control plane node.

Kiểm tra manifest static Pod:

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

---

### 6.2. Chuẩn bị thư mục backup trên master

```bash
sudo mkdir -p /opt/k8s-backup/etcd
sudo chown $(id -u):$(id -g) /opt/k8s-backup/etcd
```

Tạo biến thời gian:

```bash
BACKUP_DATE=$(date +%Y%m%d-%H%M%S)
SNAPSHOT_FILE=/opt/k8s-backup/etcd/etcd-snapshot-${BACKUP_DATE}.db
```

---

### 6.3. Cài etcdctl và etcdutl trên master

Nên dùng phiên bản `etcdctl`/`etcdutl` tương thích với phiên bản etcd đang chạy trong cluster.

Kiểm tra image etcd đang dùng:

```bash
kubectl -n kube-system get pod -l component=etcd \
  -o jsonpath='{.items[0].spec.containers[0].image}'
```

Ví dụ output:

```text
registry.k8s.io/etcd:3.6.x-0
```

Tải binary etcd tương ứng:

```bash
ETCD_VER=v3.6.4
ARCH=amd64

curl -L https://github.com/etcd-io/etcd/releases/download/${ETCD_VER}/etcd-${ETCD_VER}-linux-${ARCH}.tar.gz \
  -o /tmp/etcd-${ETCD_VER}-linux-${ARCH}.tar.gz

tar -xzf /tmp/etcd-${ETCD_VER}-linux-${ARCH}.tar.gz -C /tmp
sudo cp /tmp/etcd-${ETCD_VER}-linux-${ARCH}/etcdctl /usr/local/bin/
sudo cp /tmp/etcd-${ETCD_VER}-linux-${ARCH}/etcdutl /usr/local/bin/
```

Kiểm tra:

```bash
etcdctl version
etcdutl version
```

Ghi chú:

- Nếu image etcd của cluster dùng version khác, hãy thay `ETCD_VER` cho phù hợp.
- Không nên dùng bừa một version quá cũ hoặc không tương thích.

---

### 6.4. Thực hiện etcd snapshot

Trên master node, chạy:

```bash
ETCDCTL_API=3 etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  snapshot save ${SNAPSHOT_FILE}
```

Kết quả kỳ vọng:

```text
Snapshot saved at /opt/k8s-backup/etcd/etcd-snapshot-YYYYMMDD-HHMMSS.db
```

Giải thích lệnh:

```bash
ETCDCTL_API=3
```

Bắt buộc sử dụng API v3 của etcdctl.

```bash
--endpoints=https://127.0.0.1:2379
```

Kết nối tới etcd endpoint local trên master node.

```bash
--cacert=/etc/kubernetes/pki/etcd/ca.crt
```

CA certificate để xác thực etcd server.

```bash
--cert=/etc/kubernetes/pki/etcd/server.crt
--key=/etc/kubernetes/pki/etcd/server.key
```

Client certificate và private key để etcd cho phép truy cập.

```bash
snapshot save ${SNAPSHOT_FILE}
```

Tạo snapshot file.

---

### 6.5. Kiểm tra snapshot

```bash
etcdutl snapshot status ${SNAPSHOT_FILE} --write-out=table
```

Kết quả ví dụ:

```text
+----------+----------+------------+------------+
|   HASH   | REVISION | TOTAL KEYS | TOTAL SIZE |
+----------+----------+------------+------------+
| abcdef12 |    12345 |       2500 |      12 MB |
+----------+----------+------------+------------+
```

Ý nghĩa:

- `HASH`: mã hash snapshot.
- `REVISION`: revision của etcd tại thời điểm snapshot.
- `TOTAL KEYS`: số key trong etcd.
- `TOTAL SIZE`: dung lượng snapshot.

Nếu không verify snapshot, ta chưa chắc file backup có usable hay không.

---

### 6.6. Copy snapshot sang NFS Server

Trên `nfs-server-01`, tạo thư mục lưu backup:

```bash
sudo mkdir -p /srv/k8s-backup/etcd
sudo chown -R $USER:$USER /srv/k8s-backup
```

Trên master node, copy snapshot sang NFS Server:

```bash
scp ${SNAPSHOT_FILE} user@nfs-server-01:/srv/k8s-backup/etcd/
```

Hoặc nếu master đã mount NFS backup path:

```bash
sudo mkdir -p /mnt/k8s-backup
sudo mount nfs-server-01:/srv/k8s-backup /mnt/k8s-backup
sudo cp ${SNAPSHOT_FILE} /mnt/k8s-backup/etcd/
```

Kiểm tra trên NFS Server:

```bash
ls -lh /srv/k8s-backup/etcd/
```

---

### 6.7. Những gì etcd snapshot không giải quyết

`etcd` snapshot không trực tiếp backup nội dung file bên trong PV/PVC.

Ví dụ:

- PVC object được lưu trong `etcd`.
- PV object được lưu trong `etcd`.
- Nhưng dữ liệu thật trong NFS share không nằm trong `etcd`.

Nói cách khác:

```text
etcd biết rằng PVC tồn tại,
nhưng etcd không chứa toàn bộ file dữ liệu của ứng dụng trong PVC.
```

Vì vậy, backup `etcd` vẫn cần đi kèm backup dữ liệu storage.

---

## 7. Restore etcd snapshot trong mô hình kubeadm single control plane

### 7.1. Cảnh báo trước khi restore

Restore `etcd` là thao tác có rủi ro cao.

Không làm trên production nếu chưa có kế hoạch rõ ràng.

Trước khi restore cần hiểu:

- Restore `etcd` ảnh hưởng toàn bộ state của cluster.
- Các object tạo sau thời điểm snapshot có thể bị mất.
- Nếu storage external còn dữ liệu nhưng object PVC/PV bị quay về trạng thái cũ, có thể phát sinh lệch state.
- Với HA control plane hoặc external etcd, quy trình restore phức tạp hơn single master lab.

Trong bài này, ta chỉ thực hành theo mô hình:

```text
1 master kubeadm + stacked etcd + 2 worker
```

---

### 7.2. Tạo một object sau thời điểm backup để quan sát restore

Sau khi đã có snapshot, tạo namespace mới:

```bash
kubectl create namespace after-etcd-backup
kubectl get ns after-etcd-backup
```

Namespace này được tạo sau snapshot.

Nếu restore về snapshot cũ, namespace này sẽ biến mất.

---

### 7.3. Dừng kubelet trên master

Trên master node:

```bash
sudo systemctl stop kubelet
```

Kiểm tra:

```bash
sudo systemctl status kubelet
```

---

### 7.4. Backup thư mục etcd hiện tại trước khi restore

```bash
sudo mv /var/lib/etcd /var/lib/etcd.bak.$(date +%Y%m%d-%H%M%S)
```

Không xóa ngay dữ liệu cũ.

Luôn đổi tên để có đường lui khi lab lỗi.

---

### 7.5. Restore snapshot vào data dir mới

Giả sử file snapshot nằm tại:

```bash
/opt/k8s-backup/etcd/etcd-snapshot-YYYYMMDD-HHMMSS.db
```

Chạy:

```bash
sudo etcdutl snapshot restore /opt/k8s-backup/etcd/etcd-snapshot-YYYYMMDD-HHMMSS.db \
  --data-dir /var/lib/etcd
```

Kiểm tra:

```bash
sudo ls -l /var/lib/etcd
```

---

### 7.6. Khởi động lại kubelet

```bash
sudo systemctl start kubelet
```

Theo dõi static Pod control plane:

```bash
watch kubectl get pods -n kube-system
```

Có thể mất một lúc để kube-apiserver, controller-manager, scheduler và etcd ổn định lại.

---

### 7.7. Kiểm tra kết quả restore

Kiểm tra namespace được tạo sau backup:

```bash
kubectl get ns after-etcd-backup
```

Nếu restore thành công về thời điểm snapshot cũ, namespace này sẽ không còn.

Kiểm tra namespace demo ban đầu:

```bash
kubectl get ns backup-demo
kubectl get all -n backup-demo
```

Kiểm tra cluster:

```bash
kubectl get nodes
kubectl get pods -A
```

---

### 7.8. Khi nào dùng etcd restore?

Dùng khi:

- Mất control plane state.
- etcd data directory bị corrupt.
- Xóa nhầm nhiều object cluster-level và không thể restore chọn lọc.
- Cần khôi phục toàn bộ cluster state tại một thời điểm.

Không nên dùng etcd restore cho các trường hợp nhỏ như:

- Xóa nhầm một Deployment.
- Xóa nhầm một Namespace ứng dụng.
- Cần migrate app sang cluster khác.
- Cần restore một PVC đơn lẻ.

Với các tình huống đó, Velero phù hợp hơn.

---

## 8. Giới thiệu Velero

### 8.1. Velero là gì?

Velero là công cụ backup, restore và migrate tài nguyên Kubernetes.

Velero thường được dùng để:

- Backup tài nguyên Kubernetes.
- Restore namespace/application khi bị xóa.
- Migrate ứng dụng từ cluster này sang cluster khác.
- Backup dữ liệu PV nếu cấu hình thêm file system backup hoặc snapshot/data mover.
- Tạo lịch backup định kỳ.

Velero làm việc thông qua Kubernetes API.

Khi backup, Velero đọc object từ Kubernetes API server, sau đó lưu metadata backup vào object storage.

Object storage có thể là:

- AWS S3.
- Azure Blob.
- Google Cloud Storage.
- S3-compatible storage.
- MinIO.
- Ceph RGW.
- Object storage on-premise khác.

---

### 8.2. Kiến trúc cơ bản của Velero

Các thành phần chính:

| Thành phần | Vai trò |
|---|---|
| Velero CLI | Công cụ dòng lệnh để tạo backup/restore/schedule |
| Velero Server | Chạy trong cluster, xử lý backup và restore |
| BackupStorageLocation | Khai báo nơi lưu backup metadata và artifact |
| VolumeSnapshotLocation | Khai báo nơi lưu snapshot volume nếu dùng snapshot provider |
| Node Agent | DaemonSet hỗ trợ file system backup hoặc data movement |
| Object Storage | Nơi lưu backup ngoài cluster |
| Provider Plugin | Plugin kết nối tới AWS/Azure/GCP/S3-compatible storage |

Luồng đơn giản:

```text
velero CLI
   |
   v
Kubernetes API Server
   |
   v
Velero Server trong cluster
   |
   v
Object Storage / Backup Repository
```

Nếu backup volume bằng file system backup:

```text
Pod volume data
   |
   v
Velero Node Agent
   |
   v
Object Storage
```

---

### 8.3. Velero backup gì?

Velero có thể backup:

- Namespace.
- Deployment.
- ReplicaSet.
- Pod metadata.
- Service.
- ConfigMap.
- Secret.
- PVC.
- PV metadata.
- Ingress.
- NetworkPolicy.
- RBAC object.
- HPA.
- CRD và custom resource, nếu có.

Velero không tự động đảm bảo application-consistent backup cho mọi database.

Ví dụ:

- Với MySQL/PostgreSQL/MongoDB, cần backup logic hoặc snapshot có quiesce/freeze phù hợp.
- Với file tĩnh trên NFS, file system backup có thể đủ cho lab.
- Với database đang ghi liên tục, cần thiết kế backup riêng ở tầng database.

---

## 9. Chuẩn bị object storage cho Velero trong lab on-premise

### 9.1. Nguyên tắc thiết kế lab

Trong production, object storage nên nằm ngoài cluster.

Trong lab này, ta dùng `nfs-server-01` để chạy một S3-compatible endpoint phục vụ Velero.

Mô hình:

```text
Kubernetes Cluster
  - k8s-master-01
  - k8s-worker-01
  - k8s-worker-02

External Backup Repository Lab
  - nfs-server-01
  - chạy S3-compatible object storage
  - lưu backup Velero
  - lưu etcd snapshot
```

Ghi chú:

- Phần S3-compatible endpoint có thể dùng MinIO hoặc một object storage khác.
- Trong tài liệu này, các lệnh bên dưới dùng MinIO chạy bằng Podman ở mức lab.
- Nếu doanh nghiệp đã có S3-compatible storage thật, hãy thay endpoint/bucket/credential tương ứng.

---

### 9.2. Cài Podman trên NFS Server

Trên `nfs-server-01`:

```bash
sudo apt update
sudo apt install -y podman
```

Tạo thư mục lưu dữ liệu object storage:

```bash
sudo mkdir -p /srv/minio/data
sudo chown -R 1000:1000 /srv/minio/data
```

---

### 9.3. Chạy MinIO lab bằng Podman

```bash
sudo podman run -d \
  --name minio \
  --restart=always \
  -p 9000:9000 \
  -p 9001:9001 \
  -e MINIO_ROOT_USER=minioadmin \
  -e MINIO_ROOT_PASSWORD='Minio@123456' \
  -v /srv/minio/data:/data \
  docker.io/minio/minio:latest server /data --console-address ':9001'
```

Kiểm tra:

```bash
sudo podman ps
```

Truy cập console:

```text
http://<NFS_SERVER_IP>:9001
```

Ví dụ:

```text
http://10.10.10.31:9001
```

Thông tin đăng nhập lab:

```text
Username: minioadmin
Password: Minio@123456
```

Tạo bucket tên:

```text
velero
```

Có thể tạo bucket bằng web console hoặc dùng MinIO client.

---

### 9.4. Tạo bucket bằng MinIO client container

Trên `nfs-server-01`:

```bash
sudo podman run --rm --network host docker.io/minio/mc sh -c \
  "mc alias set local http://127.0.0.1:9000 minioadmin 'Minio@123456' && mc mb --ignore-existing local/velero"
```

Kiểm tra bucket:

```bash
sudo podman run --rm --network host docker.io/minio/mc sh -c \
  "mc alias set local http://127.0.0.1:9000 minioadmin 'Minio@123456' && mc ls local"
```

Kết quả kỳ vọng:

```text
velero/
```

---

## 10. Cài Velero CLI và Velero Server

### 10.1. Chọn version Velero

Với Kubernetes v1.35, nên dùng Velero version đã được kiểm thử hoặc tương thích với Kubernetes v1.35.

Trong lab này dùng ví dụ:

```text
Velero v1.18.x
AWS plugin v1.14.x
```

Lưu ý:

- Hãy kiểm tra compatibility matrix của Velero trước khi triển khai production.
- Không nên copy version từ tài liệu cũ mà không kiểm tra tương thích.
- S3-compatible storage dùng plugin AWS vì nó nói chuyện qua S3 API.

---

### 10.2. Cài Velero CLI trên master

Trên `k8s-master-01`:

```bash
cd /tmp
VELERO_VERSION=v1.18.0

curl -LO https://github.com/vmware-tanzu/velero/releases/download/${VELERO_VERSION}/velero-${VELERO_VERSION}-linux-amd64.tar.gz

tar -xzf velero-${VELERO_VERSION}-linux-amd64.tar.gz
sudo mv velero-${VELERO_VERSION}-linux-amd64/velero /usr/local/bin/
```

Kiểm tra:

```bash
velero version --client-only
```

---

### 10.3. Tạo file credential cho S3-compatible storage

Trên master node:

```bash
mkdir -p ~/velero-lab
cd ~/velero-lab
nano credentials-velero
```

Nội dung:

```ini
[default]
aws_access_key_id = minioadmin
aws_secret_access_key = Minio@123456
```

Phân quyền file:

```bash
chmod 600 credentials-velero
```

---

### 10.4. Cài Velero server vào cluster

Thay `<NFS_SERVER_IP>` bằng IP thật của `nfs-server-01`.

Ví dụ:

```bash
NFS_SERVER_IP=10.10.10.31
```

Cài Velero:

```bash
velero install \
  --provider aws \
  --plugins velero/velero-plugin-for-aws:v1.14.0 \
  --bucket velero \
  --secret-file ./credentials-velero \
  --backup-location-config region=minio,s3ForcePathStyle="true",s3Url=http://${NFS_SERVER_IP}:9000 \
  --use-volume-snapshots=false \
  --use-node-agent \
  --default-volumes-to-fs-backup \
  --wait
```

Giải thích các flag quan trọng:

```bash
--provider aws
```

Dùng provider AWS vì Velero dùng AWS plugin để giao tiếp với S3 API.

```bash
--plugins velero/velero-plugin-for-aws:v1.14.0
```

Cài plugin AWS tương thích với Velero v1.18.x.

```bash
--bucket velero
```

Bucket object storage dùng để lưu backup.

```bash
--secret-file ./credentials-velero
```

File credential để Velero đăng nhập vào object storage.

```bash
--backup-location-config region=minio,s3ForcePathStyle="true",s3Url=http://${NFS_SERVER_IP}:9000
```

Cấu hình endpoint S3-compatible.

- `region=minio`: region đặt tùy ý cho MinIO lab.
- `s3ForcePathStyle=true`: quan trọng với MinIO/S3-compatible storage vì nhiều hệ thống không dùng virtual-host style mặc định.
- `s3Url`: endpoint object storage.

```bash
--use-volume-snapshots=false
```

Không dùng native volume snapshot trong lab này vì NFS CSI không phải backend snapshot block storage kiểu cloud EBS/Azure Disk/GCE PD.

```bash
--use-node-agent
```

Cài Velero Node Agent để hỗ trợ backup volume dạng file system.

```bash
--default-volumes-to-fs-backup
```

Mặc định backup volume bằng file system backup khi có volume phù hợp.

```bash
--wait
```

Chờ các thành phần Velero sẵn sàng.

---

### 10.5. Kiểm tra Velero

```bash
kubectl get ns velero
kubectl get pods -n velero
```

Kết quả kỳ vọng:

```text
velero deployment running
node-agent daemonset running trên các node
```

Kiểm tra backup location:

```bash
velero backup-location get
```

Kết quả kỳ vọng:

```text
NAME      PROVIDER   BUCKET/PREFIX   PHASE       LAST VALIDATED
 default  aws        velero          Available   ...
```

Nếu phase không phải `Available`, kiểm tra:

```bash
kubectl logs deployment/velero -n velero
velero backup-location describe default
```

---

## 11. Backup ứng dụng bằng Velero

### 11.1. Kiểm tra app demo trước khi backup

```bash
kubectl get all -n backup-demo
kubectl get pvc -n backup-demo
```

Kiểm tra nội dung file trong PVC:

```bash
POD_NAME=$(kubectl get pod -n backup-demo -l app=web-backup-demo -o jsonpath='{.items[0].metadata.name}')

kubectl exec -n backup-demo $POD_NAME -- cat /usr/share/nginx/html/index.html
```

Kết quả:

```text
Hello from PVC before backup
```

---

### 11.2. Tạo backup cho namespace

```bash
velero backup create backup-demo-001 --include-namespaces backup-demo
```

Kiểm tra trạng thái:

```bash
velero backup get
```

Xem chi tiết:

```bash
velero backup describe backup-demo-001 --details
```

Xem log:

```bash
velero backup logs backup-demo-001
```

Kết quả kỳ vọng:

```text
Phase: Completed
Errors: 0
```

---

### 11.3. Kiểm tra object storage đã có dữ liệu backup

Trên `nfs-server-01`:

```bash
sudo podman run --rm --network host docker.io/minio/mc sh -c \
  "mc alias set local http://127.0.0.1:9000 minioadmin 'Minio@123456' && mc ls --recursive local/velero | head"
```

Nếu có object mới được tạo, nghĩa là Velero đã upload backup lên object storage.

---

## 12. Mô phỏng sự cố: xóa namespace ứng dụng

### 12.1. Xóa namespace

```bash
kubectl delete namespace backup-demo
```

Chờ namespace bị xóa hoàn toàn:

```bash
kubectl get ns backup-demo
```

Kết quả kỳ vọng:

```text
Error from server (NotFound): namespaces "backup-demo" not found
```

Kiểm tra resource:

```bash
kubectl get all -n backup-demo
```

Kết quả:

```text
No resources found hoặc namespace not found
```

Đây là tình huống rất thực tế:

```text
Admin xóa nhầm namespace production.
```

---

## 13. Restore namespace bằng Velero

### 13.1. Restore từ backup

```bash
velero restore create restore-backup-demo-001 --from-backup backup-demo-001
```

Kiểm tra restore:

```bash
velero restore get
```

Xem chi tiết:

```bash
velero restore describe restore-backup-demo-001 --details
```

Xem logs:

```bash
velero restore logs restore-backup-demo-001
```

---

### 13.2. Kiểm tra namespace sau restore

```bash
kubectl get ns backup-demo
kubectl get all -n backup-demo
kubectl get pvc -n backup-demo
```

Kiểm tra Pod:

```bash
kubectl get pods -n backup-demo -o wide
```

---

### 13.3. Kiểm tra dữ liệu trong PVC sau restore

Chọn Pod mới:

```bash
POD_NAME=$(kubectl get pod -n backup-demo -l app=web-backup-demo -o jsonpath='{.items[0].metadata.name}')
```

Đọc nội dung file:

```bash
kubectl exec -n backup-demo $POD_NAME -- cat /usr/share/nginx/html/index.html
```

Nếu file system backup hoạt động đúng, kết quả kỳ vọng:

```text
Hello from PVC before backup
```

Nếu chỉ restore được object Kubernetes nhưng không restore được dữ liệu PV, cần kiểm tra:

```bash
velero backup describe backup-demo-001 --details
velero backup logs backup-demo-001
velero restore logs restore-backup-demo-001
kubectl get pods -n velero
kubectl logs -n velero daemonset/node-agent
```

---

## 14. Backup theo label selector

### 14.1. Vì sao cần backup theo label?

Trong một namespace có thể có nhiều ứng dụng.

Nếu chỉ muốn backup một nhóm resource cụ thể, có thể dùng label selector.

Ví dụ app có label:

```yaml
labels:
  app: web-backup-demo
```

Tạo backup theo label:

```bash
velero backup create web-backup-demo-label \
  --selector app=web-backup-demo
```

Kiểm tra:

```bash
velero backup describe web-backup-demo-label --details
```

Lưu ý:

- Backup theo label phụ thuộc vào việc resource có label đầy đủ hay không.
- Nếu Service, ConfigMap, Secret, PVC không có label phù hợp, chúng có thể không được backup.
- Vì vậy trong production nên có chiến lược labeling nhất quán.

---

### 14.2. Cập nhật manifest để có label đồng nhất

Ví dụ nên thêm label vào các object chính:

```yaml
metadata:
  labels:
    app: web-backup-demo
    backup: velero
    environment: lab
```

Khi đó có thể backup bằng:

```bash
velero backup create backup-by-label \
  --selector backup=velero
```

---

## 15. Backup định kỳ bằng Velero Schedule

### 15.1. Tạo schedule backup hàng ngày

```bash
velero schedule create backup-demo-daily \
  --schedule="0 1 * * *" \
  --include-namespaces backup-demo
```

Ý nghĩa:

```text
0 1 * * *
```

Chạy lúc 01:00 mỗi ngày.

Kiểm tra:

```bash
velero schedule get
```

Xem chi tiết:

```bash
velero schedule describe backup-demo-daily
```

---

### 15.2. Tạo schedule backup mỗi 6 giờ

```bash
velero schedule create backup-demo-every-6h \
  --schedule="0 */6 * * *" \
  --include-namespaces backup-demo
```

Phù hợp với app có RPO thấp hơn.

---

### 15.3. Backup retention bằng TTL

Tạo backup có TTL 7 ngày:

```bash
velero backup create backup-demo-ttl-7d \
  --include-namespaces backup-demo \
  --ttl 168h
```

Ý nghĩa:

```text
168h = 7 ngày
```

Cần lưu ý:

- TTL giúp tránh backup tăng vô hạn.
- Retention nên bám theo policy doanh nghiệp.
- Không nên chỉ giữ một bản backup gần nhất.

---

## 16. Restore sang namespace khác

### 16.1. Vì sao cần restore sang namespace khác?

Trong thực tế, nhiều khi không muốn restore đè lên namespace cũ.

Ví dụ:

- Muốn kiểm tra backup có dùng được không.
- Muốn phục hồi dữ liệu để so sánh.
- Muốn clone môi trường production sang staging.

Velero hỗ trợ namespace mapping khi restore.

---

### 16.2. Restore `backup-demo` sang `backup-demo-restore-test`

```bash
velero restore create restore-to-new-ns \
  --from-backup backup-demo-001 \
  --namespace-mappings backup-demo:backup-demo-restore-test
```

Kiểm tra:

```bash
kubectl get ns backup-demo-restore-test
kubectl get all -n backup-demo-restore-test
kubectl get pvc -n backup-demo-restore-test
```

Đọc dữ liệu:

```bash
POD_NAME=$(kubectl get pod -n backup-demo-restore-test -l app=web-backup-demo -o jsonpath='{.items[0].metadata.name}')

kubectl exec -n backup-demo-restore-test $POD_NAME -- cat /usr/share/nginx/html/index.html
```

---

## 17. Backup và restore PVC: cần hiểu đúng

### 17.1. PVC object và dữ liệu PVC là hai chuyện khác nhau

PVC object là một object Kubernetes.

Dữ liệu PVC là dữ liệu nằm trên storage backend.

Ví dụ với NFS CSI:

```text
PVC object nằm trong etcd.
Dữ liệu thật nằm trong thư mục export trên NFS Server.
```

Khi backup bằng Velero:

- Nếu chỉ backup Kubernetes object, PVC object có thể restore lại nhưng dữ liệu chưa chắc còn.
- Nếu bật file system backup hoặc snapshot, Velero có thể backup dữ liệu volume.

---

### 17.2. NFS CSI và volume snapshot

Trong môi trường cloud, nhiều volume backend hỗ trợ snapshot native:

- AWS EBS.
- Azure Disk.
- Google Persistent Disk.
- Một số CSI storage enterprise.

Nhưng với NFS:

- Không phải backend snapshot block storage native.
- Thường không có `VolumeSnapshot` đúng nghĩa như block volume.
- File system backup phù hợp hơn cho lab NFS.

Vì vậy trong bài này dùng:

```bash
--use-volume-snapshots=false
--use-node-agent
--default-volumes-to-fs-backup
```

---

### 17.3. Khi nào file system backup không đủ?

File system backup đọc dữ liệu từ filesystem đang được mount.

Với app ghi dữ liệu liên tục, có thể xảy ra:

- File đang ghi dở.
- Database chưa flush dữ liệu.
- Backup không application-consistent.
- Restore lên app có thể cần recovery logic.

Với database production, nên dùng:

- Backup native của database.
- Pre-backup hook để quiesce/freeze app.
- Post-backup hook để unfreeze app.
- Storage snapshot có application-consistent mechanism.
- Operator backup chuyên dụng nếu có.

---

## 18. Velero hook cơ bản

### 18.1. Hook là gì?

Hook cho phép chạy lệnh trước hoặc sau khi backup/restore.

Ví dụ:

- Trước backup: flush dữ liệu ra disk.
- Trước backup: khóa ghi tạm thời.
- Sau backup: mở khóa ghi.
- Sau restore: chạy migration hoặc kiểm tra dữ liệu.

Trong Velero, hook có thể cấu hình bằng annotation.

---

### 18.2. Ví dụ pre-backup hook đơn giản

Ví dụ minh họa thêm annotation vào Pod template của Deployment:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-backup-demo
  namespace: backup-demo
spec:
  replicas: 2
  selector:
    matchLabels:
      app: web-backup-demo
  template:
    metadata:
      labels:
        app: web-backup-demo
      annotations:
        pre.hook.backup.velero.io/container: nginx
        pre.hook.backup.velero.io/command: '["/bin/sh", "-c", "sync"]'
    spec:
      containers:
        - name: nginx
          image: nginx:1.27-alpine
          volumeMounts:
            - name: web-content
              mountPath: /usr/share/nginx/html
      volumes:
        - name: web-content
          persistentVolumeClaim:
            claimName: web-content-pvc
```

Giải thích:

```yaml
pre.hook.backup.velero.io/container: nginx
```

Chạy hook trong container tên `nginx`.

```yaml
pre.hook.backup.velero.io/command: '["/bin/sh", "-c", "sync"]'
```

Chạy lệnh `sync` trước backup để flush filesystem buffer.

Ghi chú:

- Đây chỉ là ví dụ đơn giản.
- Với database, hook phải dùng command phù hợp với database engine.
- Không nên tự nghĩ hook nếu không hiểu app.

---

## 19. Disaster Recovery scenario tổng hợp

### 19.1. Mục tiêu lab tổng hợp

Trong lab tổng hợp, sẽ thực hiện:

- Tạo ứng dụng có ConfigMap, Secret, PVC, Deployment, Service.
- Ghi dữ liệu vào PVC.
- Backup ứng dụng bằng Velero.
- Xóa namespace để mô phỏng sự cố.
- Restore namespace.
- Kiểm tra workload, service và dữ liệu PVC.
- Tạo etcd snapshot.
- Tạo object mới sau snapshot.
- Restore etcd để quan sát state quay về thời điểm backup.

---

### 19.2. Bước 1: Chuẩn bị app

```bash
kubectl create namespace backup-demo --dry-run=client -o yaml | kubectl apply -f -
kubectl apply -f app-demo.yaml
```

Kiểm tra:

```bash
kubectl get all,pvc -n backup-demo
```

---

### 19.3. Bước 2: Ghi dữ liệu vào PVC

```bash
POD_NAME=$(kubectl get pod -n backup-demo -l app=web-backup-demo -o jsonpath='{.items[0].metadata.name}')

kubectl exec -n backup-demo $POD_NAME -- sh -c 'echo "Data version 1 before Velero backup" > /usr/share/nginx/html/index.html'
```

Kiểm tra:

```bash
kubectl exec -n backup-demo $POD_NAME -- cat /usr/share/nginx/html/index.html
```

---

### 19.4. Bước 3: Tạo Velero backup

```bash
velero backup create backup-demo-final \
  --include-namespaces backup-demo \
  --ttl 168h
```

Theo dõi:

```bash
velero backup get
velero backup describe backup-demo-final --details
```

---

### 19.5. Bước 4: Mô phỏng lỗi mất namespace

```bash
kubectl delete namespace backup-demo
```

Kiểm tra:

```bash
kubectl get ns backup-demo
```

---

### 19.6. Bước 5: Restore namespace

```bash
velero restore create restore-backup-demo-final \
  --from-backup backup-demo-final
```

Theo dõi:

```bash
velero restore get
velero restore describe restore-backup-demo-final --details
```

---

### 19.7. Bước 6: Kiểm tra app sau restore

```bash
kubectl get all,pvc -n backup-demo
```

Kiểm tra dữ liệu:

```bash
POD_NAME=$(kubectl get pod -n backup-demo -l app=web-backup-demo -o jsonpath='{.items[0].metadata.name}')

kubectl exec -n backup-demo $POD_NAME -- cat /usr/share/nginx/html/index.html
```

Kết quả kỳ vọng:

```text
Data version 1 before Velero backup
```

---

### 19.8. Bước 7: Tạo etcd snapshot

```bash
BACKUP_DATE=$(date +%Y%m%d-%H%M%S)
SNAPSHOT_FILE=/opt/k8s-backup/etcd/etcd-snapshot-${BACKUP_DATE}.db

ETCDCTL_API=3 etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  snapshot save ${SNAPSHOT_FILE}

etcdutl snapshot status ${SNAPSHOT_FILE} --write-out=table
```

---

### 19.9. Bước 8: Tạo object sau snapshot

```bash
kubectl create namespace created-after-snapshot
kubectl get ns created-after-snapshot
```

---

### 19.10. Bước 9: Restore etcd snapshot để quan sát state rollback

Chỉ làm trong lab.

```bash
sudo systemctl stop kubelet
sudo mv /var/lib/etcd /var/lib/etcd.bak.$(date +%Y%m%d-%H%M%S)
sudo etcdutl snapshot restore ${SNAPSHOT_FILE} --data-dir /var/lib/etcd
sudo systemctl start kubelet
```

Chờ control plane ổn định:

```bash
watch kubectl get pods -n kube-system
```

Kiểm tra namespace tạo sau snapshot:

```bash
kubectl get ns created-after-snapshot
```

Kết quả kỳ vọng:

```text
NotFound
```

Vì namespace đó được tạo sau thời điểm snapshot.

---

## 20. Troubleshooting Velero

### 20.1. Backup location không Available

Kiểm tra:

```bash
velero backup-location get
velero backup-location describe default
kubectl logs deployment/velero -n velero
```

Nguyên nhân thường gặp:

- Sai endpoint `s3Url`.
- MinIO/Object Storage không reachable từ Pod Velero.
- Sai credential.
- Bucket chưa tồn tại.
- Thiếu `s3ForcePathStyle=true` khi dùng MinIO.
- Firewall chặn port 9000.

Kiểm tra network từ Pod trong cluster:

```bash
kubectl run curl-s3-test -n velero --image=curlimages/curl:8.10.1 --rm -it --restart=Never -- \
  curl -I http://<NFS_SERVER_IP>:9000
```

---

### 20.2. Backup Completed nhưng PVC data không restore

Kiểm tra backup có volume backup hay không:

```bash
velero backup describe backup-demo-001 --details
```

Kiểm tra Node Agent:

```bash
kubectl get pods -n velero
kubectl logs -n velero daemonset/node-agent
```

Nguyên nhân thường gặp:

- Không cài `--use-node-agent`.
- Không bật `--default-volumes-to-fs-backup`.
- Volume không được đánh dấu backup.
- Pod đang không mount volume tại thời điểm backup.
- Permission filesystem gây lỗi đọc dữ liệu.

---

### 20.3. Restore có warning hoặc error

Kiểm tra:

```bash
velero restore describe <restore-name> --details
velero restore logs <restore-name>
```

Nguyên nhân thường gặp:

- Resource đã tồn tại.
- Namespace mapping sai.
- StorageClass không tồn tại ở cluster đích.
- IngressClass khác cluster nguồn.
- Secret hoặc ConfigMap bị conflict.
- PVC không bind được.

---

### 20.4. Restore PVC bị Pending

Kiểm tra:

```bash
kubectl get pvc -n <namespace>
kubectl describe pvc <pvc-name> -n <namespace>
kubectl get storageclass
kubectl get pods -n kube-system | grep nfs
```

Nguyên nhân thường gặp:

- StorageClass `nfs-csi` chưa có ở cluster đích.
- NFS CSI Driver chưa chạy.
- NFS Server không reachable.
- Access mode không phù hợp.
- Reclaim policy hoặc PV cũ gây conflict.

---

### 20.5. etcd snapshot restore xong nhưng kube-apiserver không lên

Kiểm tra kubelet:

```bash
sudo journalctl -u kubelet -f
```

Kiểm tra static pod manifest:

```bash
sudo ls -l /etc/kubernetes/manifests/
```

Kiểm tra etcd data dir:

```bash
sudo ls -l /var/lib/etcd
```

Nguyên nhân thường gặp:

- Restore sai snapshot.
- Permission thư mục `/var/lib/etcd` sai.
- etcd static Pod không mount đúng data dir.
- Certificate bị thiếu hoặc sai.
- Restore theo quy trình single control plane nhưng cluster thực tế là HA.

---

## 21. Best practices backup và DR cho Kubernetes on-premise

### 21.1. Backup repository phải nằm ngoài cluster

Không nên để backup chỉ nằm trong chính cluster.

Sai lầm thường gặp:

```text
Chạy MinIO trong cluster, lưu dữ liệu trên PVC trong chính cluster,
rồi xem đó là backup production.
```

Nếu cluster mất hoàn toàn, backup có thể mất theo.

Thiết kế tốt hơn:

- Object storage nằm ngoài cluster.
- Backup repository có replication.
- Backup có encryption.
- Backup có retention.
- Backup có kiểm soát quyền truy cập.

---

### 21.2. Luôn backup etcd định kỳ

Với kubeadm cluster:

- Backup etcd trước khi upgrade.
- Backup etcd trước khi thay đổi lớn.
- Backup etcd định kỳ theo RPO.
- Copy snapshot ra ngoài master node.
- Encrypt snapshot nếu chứa dữ liệu nhạy cảm.

Không nên chỉ lưu snapshot trong `/opt` trên master.

---

### 21.3. Không xem Velero là thay thế hoàn toàn database backup

Velero rất hữu ích cho Kubernetes resource và volume backup.

Nhưng với database production:

- PostgreSQL nên có `pg_dump`, base backup, WAL archiving hoặc operator backup.
- MySQL/MariaDB nên có logical/physical backup phù hợp.
- MongoDB nên dùng backup mechanism riêng.
- Redis cần RDB/AOF strategy rõ ràng.

Velero có thể là một lớp trong chiến lược backup, không phải toàn bộ chiến lược.

---

### 21.4. Test restore định kỳ

Backup định kỳ nhưng không restore test là chưa đủ.

Nên có lịch:

- Hàng tuần: restore namespace test.
- Hàng tháng: restore app có PVC.
- Hàng quý: DR drill cluster-level.
- Trước major upgrade: backup và test restore critical workload.

Checklist restore test:

- Object restore được.
- Pod chạy lại.
- Service có endpoint.
- Ingress truy cập được.
- PVC bind được.
- Dữ liệu đọc được.
- Secret/ConfigMap đúng.
- App pass health check.

---

### 21.5. Lưu manifest trong Git

Nên quản lý manifest bằng Git:

```text
Git repository = source of truth
Cluster = runtime state
Backup = recovery mechanism
```

Không nên chỉ quản lý ứng dụng bằng lệnh `kubectl apply` thủ công trên master.

Nếu không có Git:

- Không biết ai đổi gì.
- Khó rollback cấu hình.
- Khó rebuild cluster.
- Khó audit.

---

### 21.6. Bảo vệ Secret trong backup

Backup Kubernetes có thể chứa Secret.

Vì vậy cần:

- Hạn chế người có quyền đọc backup.
- Encrypt backup repository.
- Không lưu credential trong plain text public repo.
- Cân nhắc External Secrets / Vault ở production.
- Bật encryption at rest cho Secret trong etcd nếu có yêu cầu bảo mật cao.

---

## 22. Câu hỏi kiểm tra cuối buổi

### Câu 1

Vì sao backup VM của master node chưa chắc đã đủ để backup Kubernetes cluster?

Gợi ý trả lời:

- Kubernetes có nhiều lớp dữ liệu.
- State chính nằm trong `etcd`.
- Dữ liệu ứng dụng nằm trong storage backend.
- Runtime state và desired state cần được hiểu riêng.

---

### Câu 2

`etcd snapshot` phù hợp với loại sự cố nào?

Gợi ý trả lời:

- Mất hoặc corrupt etcd.
- Mất control plane state.
- Cần restore toàn bộ cluster state về một thời điểm.

---

### Câu 3

Vì sao `etcd snapshot` không đủ để backup dữ liệu trong PVC?

Gợi ý trả lời:

- etcd lưu object PVC/PV.
- Dữ liệu thật nằm trên storage backend.
- Với NFS CSI, dữ liệu nằm trên NFS Server, không nằm trong etcd.

---

### Câu 4

Velero khác gì so với `kubectl get -o yaml`?

Gợi ý trả lời:

- Velero có backup/restore workflow.
- Velero lưu vào object storage.
- Velero hỗ trợ schedule, restore, namespace mapping, volume backup.
- `kubectl get -o yaml` chỉ export object ở thời điểm hiện tại.

---

### Câu 5

Khi dùng MinIO/S3-compatible storage với Velero, vì sao cần `s3ForcePathStyle=true`?

Gợi ý trả lời:

- Nhiều S3-compatible storage như MinIO mặc định dùng path-style access.
- Nếu Velero/AWS SDK dùng virtual-host style, có thể upload được nhưng download/restore lỗi.

---

### Câu 6

Vì sao cần test restore định kỳ?

Gợi ý trả lời:

- Để xác nhận backup dùng được thật.
- Để phát hiện lỗi credential, storage, version, PVC, permission.
- Để đo RTO thực tế.
- Để tránh phát hiện backup lỗi khi đã xảy ra thảm họa.

---

## 23. Tổng kết buổi học

Trong buổi này, đã học được:

- Kubernetes backup không chỉ là backup node VM.
- `etcd` lưu toàn bộ cluster state và cần được backup định kỳ.
- `etcd snapshot` phù hợp cho disaster recovery cấp cluster.
- Manifest export hữu ích nhưng không thay thế GitOps/source manifest chuẩn.
- Velero phù hợp cho backup/restore application, namespace và migration.
- Object storage là thành phần quan trọng trong kiến trúc backup Kubernetes.
- Với NFS CSI, file system backup phù hợp hơn snapshot native.
- Backup PVC object và backup dữ liệu PVC là hai vấn đề khác nhau.
- Restore test định kỳ là bắt buộc nếu muốn backup có giá trị thực tế.

Thông điệp quan trọng nhất:

```text
Backup không phải là có file backup.
Backup là khả năng khôi phục hệ thống về trạng thái mong muốn trong RPO/RTO đã cam kết.
```

---

## 24. Chuẩn bị cho buổi tiếp theo

Sau buổi này, đã biết cách bảo vệ workload và dữ liệu ở mức cơ bản.

Buổi tiếp theo có thể đi vào một trong các hướng sau:

- Helm nâng cao và tự viết Helm Chart.
- GitOps với Argo CD hoặc Flux.
- Kubernetes upgrade và maintenance.
- Production operation checklist.
- Tổng hợp triển khai ứng dụng production-like.

Nếu đi theo roadmap ban đầu, buổi tiếp theo nên là:

```text
Buổi 13: Helm nâng cao và tự viết Helm Chart cơ bản
```

Lý do:

- đã viết nhiều manifest thủ công.
- Đã dùng Helm để cài MetalLB, Ingress, có thể Velero.
- Đã hiểu cấu trúc ứng dụng gồm Deployment, Service, Ingress, ConfigMap, Secret, PVC.
- Đây là thời điểm phù hợp để gom các manifest đó thành chart tái sử dụng được.

---

## 25. Tài liệu tham khảo

- Kubernetes Documentation - Operating etcd clusters for Kubernetes: https://kubernetes.io/docs/tasks/administer-cluster/configure-upgrade-etcd/
- Velero Documentation - Basic Install: https://velero.io/docs/v1.17/basic-install/
- Velero Documentation - Quick start evaluation install with Minio: https://velero.io/docs/v1.17/contributions/minio/
- Velero Documentation - Backup Reference: https://velero.io/docs/v1.14/backup-reference/
- Velero Documentation - Restore Reference: https://velero.io/docs/v1.11/restore-reference/
- Velero Documentation - File System Backup: https://velero.io/docs/main/file-system-backup/
- Velero Documentation - CSI Snapshot Data Movement: https://velero.io/docs/main/csi-snapshot-data-movement/
- Velero AWS Plugin Compatibility: https://github.com/velero-io/velero-plugin-for-aws
