# Buổi 05 - ConfigMap, Secret, Volume, PV, PVC, StorageClass và CSI NFS

## 1. Mục tiêu buổi học

Sau các buổi trước,  đã biết cách dựng một cụm Kubernetes bằng `kubeadm`, biết cách chạy workload bằng Pod, ReplicaSet, Deployment, DaemonSet, StatefulSet, Job, CronJob và biết cách cho workload giao tiếp với nhau thông qua Service.

Buổi 05 đi vào một nhóm vấn đề rất thực tế khi triển khai ứng dụng trên Kubernetes:

> Ứng dụng cần file cấu hình, biến môi trường, password, certificate, thư mục ghi dữ liệu, dữ liệu dùng chung và cơ chế cấp phát storage. Kubernetes quản lý những phần này như thế nào?

Trong môi trường truyền thống, người quản trị thường SSH vào server, chỉnh file cấu hình, đặt biến môi trường, copy file secret, tạo thư mục data, mount NFS hoặc gắn disk thủ công. Trong Kubernetes, cách làm này cần được chuẩn hóa thành các Kubernetes object để Pod có thể được tạo lại, di chuyển giữa node, scale lên hoặc thay thế mà vẫn giữ được cấu hình và dữ liệu cần thiết.

Mục tiêu cụ thể của buổi học:

- Hiểu vì sao không nên hard-code cấu hình vào container image.
- Hiểu vai trò của ConfigMap trong việc tách cấu hình khỏi image.
- Hiểu vai trò của Secret trong việc lưu dữ liệu nhạy cảm ở mức Kubernetes object.
- Biết sự khác nhau giữa ConfigMap và Secret.
- Biết consume ConfigMap dưới dạng environment variable.
- Biết consume ConfigMap dưới dạng file thông qua volume mount.
- Biết consume Secret dưới dạng environment variable.
- Biết consume Secret dưới dạng file thông qua volume mount.
- Hiểu khái niệm Volume trong Pod.
- Hiểu vì sao dữ liệu trong container filesystem có thể mất khi container bị thay thế.
- Biết sử dụng một số volume cơ bản như `emptyDir`, `configMap`, `secret` và `persistentVolumeClaim`.
- Hiểu PersistentVolume là gì.
- Hiểu PersistentVolumeClaim là gì.
- Hiểu StorageClass là gì.
- Hiểu static provisioning và dynamic provisioning.
- Hiểu CSI là gì và vì sao Kubernetes cần CSI driver.
- Triển khai thêm một VM làm NFS Server cho lab storage.
- Cài đặt NFS CSI Driver cho Kubernetes cluster.
- Tạo StorageClass dùng NFS CSI.
- Tạo PVC được cấp phát động từ NFS CSI.
- Mount PVC vào Pod hoặc Deployment.
- Kiểm tra dữ liệu còn tồn tại khi Pod bị xóa và tạo lại.
- Biết troubleshooting các lỗi phổ biến liên quan đến ConfigMap, Secret, PVC, StorageClass và NFS CSI.

## 2. Phạm vi kiến thức của buổi này

### 2.1. Được phép sử dụng từ các buổi trước

Trong bài này, chúng ta được phép dùng các kiến thức đã học:

- Cluster
- Node
- Control Plane
- Worker Node
- kube-apiserver
- kubelet
- container runtime
- CNI
- Calico
- Pod
- Multi-container Pod
- Label
- Selector
- Namespace
- ReplicaSet
- Deployment
- DaemonSet
- StatefulSet ở mức cơ bản
- Job
- CronJob
- Service loại ClusterIP
- Service loại NodePort
- Headless Service ở mức cơ bản
- kubectl
- Manifest YAML
- `kubectl apply`
- `kubectl get`
- `kubectl describe`
- `kubectl logs`
- `kubectl exec`

### 2.2. Kiến thức mới được mở khóa trong buổi này

Buổi này mở khóa các khái niệm mới:

- ConfigMap
- Secret
- `data`
- `stringData`
- Secret type `Opaque`
- Environment variable từ ConfigMap
- Environment variable từ Secret
- Volume từ ConfigMap
- Volume từ Secret
- Volume
- `volumeMounts`
- `mountPath`
- `readOnly`
- `emptyDir`
- `hostPath` ở mức cảnh báo
- PersistentVolume
- PersistentVolumeClaim
- Access Mode
- Reclaim Policy
- StorageClass
- Dynamic Provisioning
- Static Provisioning
- CSI
- NFS CSI Driver
- NFS Server cho lab
- `nfs-common`
- `nfs-kernel-server`
- StorageClass cho NFS CSI
- PVC dùng StorageClass
- PV được tạo động

### 2.3. Các nội dung chưa đi sâu trong buổi này

Để giữ đúng tiến trình khóa học, bài này chưa đi sâu vào:

- Helm
- Kustomize
- External Secrets Operator
- Sealed Secrets
- HashiCorp Vault
- CSI Secret Store Driver
- Encryption at rest cho Kubernetes Secret trong etcd
- RBAC chi tiết cho Secret
- Pod Security Admission
- SecurityContext nâng cao
- Volume Snapshot
- Volume Clone
- Volume Expansion nâng cao theo từng CSI driver
- Ceph CSI
- Longhorn
- OpenEBS
- Local Path Provisioner
- Cloud disk CSI của AWS, Azure, GCP
- Database production trên NFS
- Backup/restore PVC
- StatefulSet production với volumeClaimTemplates

Lưu ý quan trọng: trong bài này có sử dụng `StorageClass` và `PVC` với NFS CSI để  hiểu dynamic provisioning trong môi trường on-premise. Đây là lab học tập rất tốt, nhưng không có nghĩa NFS luôn là lựa chọn tối ưu cho tất cả workload production.

## 3. Môi trường lab sử dụng trong bài

Bài lab kế thừa cụm Kubernetes từ Buổi 02 và bổ sung thêm một VM làm NFS Server.

| Thành phần | Giá trị mẫu |
|---|---|
| Kubernetes version | v1.35 |
| Bootstrap tool | kubeadm |
| OS các node Kubernetes | Ubuntu Server 24.04 |
| Container runtime | containerd |
| CNI | Calico |
| Control plane | 1 master node |
| Worker | 2 worker node |
| Storage server | 1 NFS Server VM |
| CSI driver | NFS CSI Driver |

Mô hình VM đề xuất:

| Hostname | Vai trò | IP mẫu |
|---|---|---|
| `k8s-master-01` | Control Plane | `10.10.10.11` |
| `k8s-worker-01` | Worker Node | `10.10.10.21` |
| `k8s-worker-02` | Worker Node | `10.10.10.22` |
| `nfs-server-01` | NFS Server | `10.10.10.50` |

Thông tin NFS export dùng trong bài:

| Thông số | Giá trị |
|---|---|
| NFS Server IP | `10.10.10.50` |
| NFS export path | `/srv/nfs/k8s` |
| NFS protocol khuyến nghị lab | NFSv4.1 |
| StorageClass name | `nfs-csi` |
| CSI provisioner | `nfs.csi.k8s.io` |

Sơ đồ tổng quan:

```text
                       +----------------------+
                       |   nfs-server-01      |
                       |   10.10.10.50        |
                       |   /srv/nfs/k8s       |
                       +----------+-----------+
                                  |
                                  | NFS
                                  |
+----------------------+          |          +----------------------+
| k8s-master-01        |----------+----------| k8s-worker-01        |
| Control Plane        |                     | Worker Node          |
| kube-apiserver       |                     | kubelet              |
| etcd                 |                     | containerd           |
| scheduler            |                     | NFS CSI node plugin  |
+----------------------+                     +----------------------+
                                  |
                                  |
                       +----------+-----------+
                       | k8s-worker-02        |
                       | Worker Node          |
                       | kubelet              |
                       | containerd           |
                       | NFS CSI node plugin  |
                       +----------------------+
```

## 4. Bức tranh tổng quan: cấu hình và lưu trữ trong Kubernetes

Khi triển khai ứng dụng thật, container image thường chỉ nên chứa application binary, dependency và default configuration ở mức tối thiểu. Những giá trị thay đổi theo môi trường như database host, application mode, feature flag, log level, API endpoint, username, password, token, certificate hoặc thư mục dữ liệu không nên hard-code trực tiếp vào image.

Kubernetes cung cấp nhiều object để giải quyết vấn đề này:

| Nhu cầu | Kubernetes object/cơ chế phù hợp |
|---|---|
| Cấu hình không nhạy cảm | ConfigMap |
| Password, token, key, credential | Secret |
| Mount file cấu hình vào Pod | ConfigMap volume |
| Mount file nhạy cảm vào Pod | Secret volume |
| Dữ liệu tạm trong vòng đời Pod | emptyDir volume |
| Dữ liệu bền vững | PersistentVolume + PersistentVolumeClaim |
| Tự động cấp phát storage | StorageClass + CSI driver |
| Kết nối Kubernetes với hệ thống lưu trữ ngoài | CSI |
| Shared file storage on-premise đơn giản | NFS CSI |

Có thể hình dung luồng như sau:

```text
ConfigMap / Secret
        |
        | được tham chiếu bởi Pod
        v
Pod spec
        |
        +--> env / envFrom
        |
        +--> volumes + volumeMounts

PVC
        |
        | yêu cầu storage
        v
StorageClass
        |
        | gọi CSI driver
        v
NFS CSI Driver
        |
        | tạo thư mục con trên NFS Server
        v
PersistentVolume
        |
        | bind với PVC
        v
Pod mount PVC vào container
```

Điểm quan trọng nhất của buổi này là  phải hiểu rằng Pod không nên tự biết quá nhiều về hạ tầng storage vật lý. Pod chỉ nên nói rằng: “tôi cần một volume dung lượng 1Gi, access mode như thế này, storage class như thế này”. Phần còn lại do Kubernetes và CSI driver xử lý.

## 5. ConfigMap

### 5.1. ConfigMap là gì?

ConfigMap là Kubernetes object dùng để lưu dữ liệu cấu hình không nhạy cảm theo dạng key-value.

Ví dụ dữ liệu phù hợp để đưa vào ConfigMap:

- `APP_ENV=production`
- `LOG_LEVEL=info`
- `DATABASE_HOST=mysql.default.svc.cluster.local`
- `FEATURE_X_ENABLED=true`
- Nội dung file cấu hình `nginx.conf`
- Nội dung file cấu hình `application.properties`

Không nên đưa vào ConfigMap:

- Password
- Token
- Private key
- TLS private key
- API secret
- Database credential

Lý do: ConfigMap không phải cơ chế bảo mật. Nó không được thiết kế để che giấu dữ liệu nhạy cảm.

### 5.2. Tại sao cần ConfigMap?

Giả sử ta có cùng một image ứng dụng `myapp:v1` nhưng triển khai ở nhiều môi trường:

| Môi trường | Database host | Log level |
|---|---|---|
| Dev | `mysql-dev` | `debug` |
| Staging | `mysql-staging` | `info` |
| Production | `mysql-prod` | `warn` |

Nếu hard-code cấu hình vào image, mỗi môi trường có thể cần một image khác nhau. Điều này làm image khó tái sử dụng, khó quản lý version và dễ sai lệch giữa các môi trường.

Với ConfigMap, image giữ nguyên, cấu hình được inject từ bên ngoài.

```text
same image + different ConfigMap = different runtime configuration
```

### 5.3. Manifest ConfigMap hoàn chỉnh

Tạo file `01-configmap-app.yaml`:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: webapp-config
  namespace: default
data:
  APP_ENV: "lab"
  LOG_LEVEL: "info"
  WELCOME_MESSAGE: "Hello from Kubernetes ConfigMap"
  nginx.conf: |
    server {
      listen 80;
      server_name _;

      location / {
        root /usr/share/nginx/html;
        index index.html;
      }
    }
```

Áp dụng manifest:

```bash
kubectl apply -f 01-configmap-app.yaml
```

Kiểm tra:

```bash
kubectl get configmap
kubectl describe configmap webapp-config
kubectl get configmap webapp-config -o yaml
```

### 5.4. Giải thích manifest ConfigMap

```yaml
apiVersion: v1
```

ConfigMap thuộc core API group nên dùng `apiVersion: v1`.

```yaml
kind: ConfigMap
```

Khai báo object này là ConfigMap.

```yaml
metadata:
  name: webapp-config
  namespace: default
```

ConfigMap là namespaced object. Pod muốn sử dụng ConfigMap này cũng cần nằm cùng namespace `default`, trừ khi ứng dụng tự gọi Kubernetes API để đọc ConfigMap từ namespace khác.

```yaml
data:
  APP_ENV: "lab"
  LOG_LEVEL: "info"
  WELCOME_MESSAGE: "Hello from Kubernetes ConfigMap"
```

Các key đơn giản, thường dùng để inject thành environment variable.

```yaml
  nginx.conf: |
    server {
      listen 80;
      server_name _;
```

Key `nginx.conf` có value dạng multi-line. Khi mount ConfigMap thành volume, key này có thể trở thành một file tên `nginx.conf` bên trong container.

Dấu `|` trong YAML có nghĩa là giữ nguyên nội dung multi-line bên dưới.

## 6. Sử dụng ConfigMap trong Pod

ConfigMap có thể được sử dụng phổ biến theo hai cách:

- Biến môi trường.
- File được mount vào container thông qua volume.

### 6.1. Sử dụng ConfigMap làm environment variable

Tạo file `02-pod-configmap-env.yaml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-configmap-env
  labels:
    app: configmap-demo
spec:
  containers:
    - name: demo
      image: busybox:1.36
      command: ["/bin/sh", "-c"]
      args:
        - |
          echo "APP_ENV=$APP_ENV";
          echo "LOG_LEVEL=$LOG_LEVEL";
          echo "WELCOME_MESSAGE=$WELCOME_MESSAGE";
          sleep 3600;
      env:
        - name: APP_ENV
          valueFrom:
            configMapKeyRef:
              name: webapp-config
              key: APP_ENV
        - name: LOG_LEVEL
          valueFrom:
            configMapKeyRef:
              name: webapp-config
              key: LOG_LEVEL
        - name: WELCOME_MESSAGE
          valueFrom:
            configMapKeyRef:
              name: webapp-config
              key: WELCOME_MESSAGE
```

Áp dụng:

```bash
kubectl apply -f 02-pod-configmap-env.yaml
```

Kiểm tra log:

```bash
kubectl logs pod-configmap-env
```

Kết quả mong đợi:

```text
APP_ENV=lab
LOG_LEVEL=info
WELCOME_MESSAGE=Hello from Kubernetes ConfigMap
```

### 6.2. Giải thích các dòng quan trọng

```yaml
      env:
        - name: APP_ENV
```

Khai báo environment variable tên `APP_ENV` bên trong container.

```yaml
          valueFrom:
            configMapKeyRef:
              name: webapp-config
              key: APP_ENV
```

Giá trị của biến `APP_ENV` được lấy từ key `APP_ENV` trong ConfigMap `webapp-config`.

```yaml
        - name: LOG_LEVEL
          valueFrom:
            configMapKeyRef:
              name: webapp-config
              key: LOG_LEVEL
```

Tương tự, biến `LOG_LEVEL` được lấy từ ConfigMap.

Điểm cần nhớ: nếu ConfigMap không tồn tại, hoặc key được tham chiếu không tồn tại, Pod có thể không start được và sẽ xuất hiện lỗi trong `kubectl describe pod`.

### 6.3. Sử dụng ConfigMap làm file volume

Tạo file `03-pod-configmap-volume.yaml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-configmap-volume
  labels:
    app: configmap-volume-demo
spec:
  containers:
    - name: nginx
      image: nginx:1.27-alpine
      ports:
        - containerPort: 80
      volumeMounts:
        - name: nginx-config-volume
          mountPath: /etc/nginx/conf.d/default.conf
          subPath: nginx.conf
          readOnly: true
  volumes:
    - name: nginx-config-volume
      configMap:
        name: webapp-config
        items:
          - key: nginx.conf
            path: nginx.conf
```

Áp dụng:

```bash
kubectl apply -f 03-pod-configmap-volume.yaml
```

Kiểm tra file trong container:

```bash
kubectl exec -it pod-configmap-volume -- cat /etc/nginx/conf.d/default.conf
```

Kiểm tra Pod:

```bash
kubectl get pod pod-configmap-volume -o wide
kubectl describe pod pod-configmap-volume
```

### 6.4. Giải thích ConfigMap volume

```yaml
      volumeMounts:
        - name: nginx-config-volume
```

Container mount một volume có tên `nginx-config-volume`.

```yaml
          mountPath: /etc/nginx/conf.d/default.conf
```

Đây là đường dẫn bên trong container nơi file cấu hình sẽ xuất hiện.

```yaml
          subPath: nginx.conf
```

`subPath` cho phép mount một file cụ thể từ volume vào một đường dẫn file cụ thể trong container. Nếu không dùng `subPath`, toàn bộ volume có thể được mount vào một thư mục.

```yaml
          readOnly: true
```

ConfigMap thường nên được mount read-only vì nó là cấu hình do Kubernetes quản lý.

```yaml
  volumes:
    - name: nginx-config-volume
      configMap:
        name: webapp-config
```

Khai báo volume lấy dữ liệu từ ConfigMap `webapp-config`.

```yaml
        items:
          - key: nginx.conf
            path: nginx.conf
```

Chỉ lấy key `nginx.conf` trong ConfigMap và tạo thành file `nginx.conf` trong volume.

### 6.5. Lưu ý quan trọng về cập nhật ConfigMap

Nếu ConfigMap được dùng dưới dạng environment variable, container thường cần restart để nhận giá trị mới.

Nếu ConfigMap được mount dưới dạng volume, Kubernetes có thể cập nhật file được mount theo cơ chế eventually consistent. Tuy nhiên, khi dùng `subPath`, container không tự nhận update mới của ConfigMap qua file mount.

Trong thực tế production, khi thay đổi ConfigMap, người quản trị thường rollout lại Deployment để Pod mới nhận cấu hình mới một cách rõ ràng.

Ví dụ:

```bash
kubectl rollout restart deployment <deployment-name>
```

Lệnh `rollout restart` sẽ được học kỹ hơn ở buổi Rolling Update/Rollback.

## 7. Secret

### 7.1. Secret là gì?

Secret là Kubernetes object dùng để lưu một lượng nhỏ dữ liệu nhạy cảm như password, token hoặc key.

Ví dụ dữ liệu phù hợp để đưa vào Secret:

- Database username
- Database password
- API token
- Basic authentication credential
- SSH private key
- Docker registry credential
- TLS certificate và private key

Secret giúp không phải ghi trực tiếp password hoặc token vào Pod manifest hoặc container image.

Tuy nhiên, cần nói rất rõ với :

> Kubernetes Secret không đồng nghĩa với “an toàn tuyệt đối”. Theo mặc định, Secret được lưu trong etcd dưới dạng chưa mã hóa ở tầng application nếu cluster chưa bật encryption at rest.

Vì vậy, Secret tốt hơn hard-code password vào image hoặc manifest public, nhưng production vẫn cần kết hợp RBAC, encryption at rest, audit, namespace isolation và công cụ quản lý secret chuyên dụng nếu yêu cầu bảo mật cao.

### 7.2. Secret dùng `stringData`

Tạo file `04-secret-app.yaml`:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: webapp-secret
  namespace: default
type: Opaque
stringData:
  DB_USERNAME: "appuser"
  DB_PASSWORD: "SuperSecretPassword123"
  API_TOKEN: "token-lab-123456"
```

Áp dụng:

```bash
kubectl apply -f 04-secret-app.yaml
```

Kiểm tra:

```bash
kubectl get secret
kubectl describe secret webapp-secret
kubectl get secret webapp-secret -o yaml
```

Khi xem YAML,  sẽ thấy dữ liệu được chuyển sang base64 trong trường `data`.

Ví dụ:

```bash
kubectl get secret webapp-secret -o jsonpath='{.data.DB_PASSWORD}' | base64 -d; echo
```

### 7.3. Giải thích Secret manifest

```yaml
apiVersion: v1
kind: Secret
```

Secret cũng thuộc core API group nên dùng `apiVersion: v1`.

```yaml
type: Opaque
```

`Opaque` là loại Secret generic, dùng cho các key-value tùy ý.

```yaml
stringData:
  DB_USERNAME: "appuser"
  DB_PASSWORD: "SuperSecretPassword123"
```

`stringData` cho phép ghi giá trị dạng plain text trong manifest. Khi object được tạo, Kubernetes chuyển dữ liệu này sang trường `data` ở dạng base64.

Điểm cần nhấn mạnh: base64 không phải encryption. Base64 chỉ là encoding.

Nếu dùng trường `data`, ta phải tự base64 trước:

```bash
echo -n 'SuperSecretPassword123' | base64
```

Ví dụ dạng `data`:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: example-secret-data
type: Opaque
data:
  DB_PASSWORD: U3VwZXJTZWNyZXRQYXNzd29yZDEyMw==
```

Trong lab cho , dùng `stringData` dễ đọc và dễ giảng hơn.

## 8. Sử dụng Secret trong Pod

Secret cũng thường được consume theo hai cách:

- Environment variable.
- File mount thông qua Secret volume.

### 8.1. Sử dụng Secret làm environment variable

Tạo file `05-pod-secret-env.yaml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-secret-env
  labels:
    app: secret-env-demo
spec:
  containers:
    - name: demo
      image: busybox:1.36
      command: ["/bin/sh", "-c"]
      args:
        - |
          echo "DB_USERNAME=$DB_USERNAME";
          echo "DB_PASSWORD is injected but should not be printed in real systems";
          sleep 3600;
      env:
        - name: DB_USERNAME
          valueFrom:
            secretKeyRef:
              name: webapp-secret
              key: DB_USERNAME
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: webapp-secret
              key: DB_PASSWORD
```

Áp dụng:

```bash
kubectl apply -f 05-pod-secret-env.yaml
```

Kiểm tra:

```bash
kubectl logs pod-secret-env
kubectl exec -it pod-secret-env -- printenv | grep DB_
```

### 8.2. Giải thích Secret env

```yaml
      env:
        - name: DB_USERNAME
          valueFrom:
            secretKeyRef:
              name: webapp-secret
              key: DB_USERNAME
```

Biến môi trường `DB_USERNAME` trong container lấy giá trị từ key `DB_USERNAME` của Secret `webapp-secret`.

```yaml
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: webapp-secret
              key: DB_PASSWORD
```

Biến môi trường `DB_PASSWORD` lấy từ Secret.

Lưu ý vận hành: không nên in secret ra log trong ứng dụng thật. Trong lab có thể dùng để kiểm tra, nhưng cần nói rõ đây không phải thói quen production.

### 8.3. Sử dụng Secret làm file volume

Tạo file `06-pod-secret-volume.yaml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-secret-volume
  labels:
    app: secret-volume-demo
spec:
  containers:
    - name: demo
      image: busybox:1.36
      command: ["/bin/sh", "-c"]
      args:
        - |
          echo "Secret files mounted under /etc/app-secret";
          ls -l /etc/app-secret;
          sleep 3600;
      volumeMounts:
        - name: app-secret-volume
          mountPath: /etc/app-secret
          readOnly: true
  volumes:
    - name: app-secret-volume
      secret:
        secretName: webapp-secret
```

Áp dụng:

```bash
kubectl apply -f 06-pod-secret-volume.yaml
```

Kiểm tra:

```bash
kubectl exec -it pod-secret-volume -- ls -l /etc/app-secret
kubectl exec -it pod-secret-volume -- cat /etc/app-secret/DB_USERNAME
kubectl exec -it pod-secret-volume -- cat /etc/app-secret/DB_PASSWORD
```

### 8.4. Giải thích Secret volume

```yaml
  volumes:
    - name: app-secret-volume
      secret:
        secretName: webapp-secret
```

Khai báo volume lấy dữ liệu từ Secret `webapp-secret`.

```yaml
      volumeMounts:
        - name: app-secret-volume
          mountPath: /etc/app-secret
          readOnly: true
```

Mount Secret thành thư mục `/etc/app-secret` trong container. Mỗi key trong Secret trở thành một file trong thư mục đó.

Ví dụ:

```text
/etc/app-secret/DB_USERNAME
/etc/app-secret/DB_PASSWORD
/etc/app-secret/API_TOKEN
```

Secret volume thường phù hợp hơn environment variable khi ứng dụng đọc credential từ file, ví dụ TLS key, token file hoặc config file nhạy cảm.

## 9. So sánh ConfigMap và Secret

| Tiêu chí | ConfigMap | Secret |
|---|---|---|
| Mục đích | Lưu cấu hình không nhạy cảm | Lưu dữ liệu nhạy cảm |
| Ví dụ | log level, app mode, config file | password, token, key |
| Dạng dữ liệu | `data`, `binaryData` | `data`, `stringData` |
| Có mã hóa mặc định không? | Không | Không mã hóa ở etcd nếu chưa bật encryption at rest |
| Có thể mount thành file không? | Có | Có |
| Có thể inject thành env không? | Có | Có |
| Có nên commit vào Git không? | Có thể, nếu không chứa dữ liệu nhạy cảm | Không nên commit secret thật |
| Kích thước phù hợp | Nhỏ, cấu hình | Nhỏ, credential |

Quy tắc dễ nhớ:

```text
Configuration bình thường -> ConfigMap
Credential hoặc dữ liệu nhạy cảm -> Secret
File dữ liệu lớn -> Không dùng ConfigMap/Secret, nên dùng storage khác
```

## 10. Volume trong Kubernetes

### 10.1. Vấn đề dữ liệu trong container

Container filesystem có tính tạm thời. Khi container bị xóa và tạo lại, dữ liệu ghi bên trong lớp filesystem của container có thể mất.

Ví dụ một container ghi file vào `/tmp/data.txt`. Nếu container crash và được kubelet tạo lại, file đó không nên được xem là dữ liệu bền vững.

Trong Kubernetes, muốn container sử dụng dữ liệu theo cách có kiểm soát, ta dùng Volume.

### 10.2. Volume là gì?

Volume là một thư mục được Kubernetes cung cấp cho Pod và được mount vào một hoặc nhiều container trong Pod.

Một Pod có thể có nhiều volume. Một container có thể mount một hoặc nhiều volume. Nhiều container trong cùng Pod có thể mount chung một volume để chia sẻ dữ liệu.

Mô hình đơn giản:

```text
Pod
├── volumes
│   └── shared-data
│
├── container A
│   └── mount shared-data at /data
│
└── container B
    └── mount shared-data at /data
```

### 10.3. Các loại Volume cần biết ở buổi này

| Volume type | Mục đích |
|---|---|
| `emptyDir` | Thư mục tạm, tồn tại trong vòng đời Pod |
| `configMap` | Mount ConfigMap thành file |
| `secret` | Mount Secret thành file |
| `persistentVolumeClaim` | Mount storage bền vững thông qua PVC |
| `hostPath` | Mount file/thư mục từ node vào Pod, chỉ nên dùng rất cẩn thận |

Trong bài này, `emptyDir`, `configMap`, `secret` và `persistentVolumeClaim` là trọng tâm. `hostPath` chỉ nhắc để  biết và tránh lạm dụng.

## 11. Volume `emptyDir`

### 11.1. emptyDir là gì?

`emptyDir` là volume rỗng được tạo khi Pod được gán vào một node. Volume này tồn tại trong suốt vòng đời của Pod. Nếu container trong Pod restart, dữ liệu trong `emptyDir` vẫn còn. Nhưng nếu Pod bị xóa, dữ liệu trong `emptyDir` mất theo Pod.

`emptyDir` phù hợp cho:

- Scratch space.
- Cache tạm.
- Chia sẻ file giữa nhiều container trong cùng Pod.
- Dữ liệu không cần giữ sau khi Pod mất.

Không phù hợp cho:

- Database data.
- File upload quan trọng.
- Dữ liệu cần giữ lâu dài.

### 11.2. Manifest Pod dùng emptyDir cho multi-container

Tạo file `07-pod-emptydir-multicontainer.yaml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-emptydir-demo
  labels:
    app: emptydir-demo
spec:
  containers:
    - name: writer
      image: busybox:1.36
      command: ["/bin/sh", "-c"]
      args:
        - |
          while true; do
            date >> /shared/time.txt;
            sleep 5;
          done
      volumeMounts:
        - name: shared-volume
          mountPath: /shared

    - name: reader
      image: busybox:1.36
      command: ["/bin/sh", "-c"]
      args:
        - |
          while true; do
            echo "----- content from shared volume -----";
            cat /shared/time.txt 2>/dev/null || true;
            sleep 10;
          done
      volumeMounts:
        - name: shared-volume
          mountPath: /shared

  volumes:
    - name: shared-volume
      emptyDir: {}
```

Áp dụng:

```bash
kubectl apply -f 07-pod-emptydir-multicontainer.yaml
```

Kiểm tra log container `reader`:

```bash
kubectl logs pod-emptydir-demo -c reader
```

Kiểm tra file từ container `writer`:

```bash
kubectl exec -it pod-emptydir-demo -c writer -- cat /shared/time.txt
```

Kiểm tra file từ container `reader`:

```bash
kubectl exec -it pod-emptydir-demo -c reader -- cat /shared/time.txt
```

### 11.3. Giải thích emptyDir manifest

```yaml
  volumes:
    - name: shared-volume
      emptyDir: {}
```

Tạo một volume loại `emptyDir` tên `shared-volume`.

```yaml
      volumeMounts:
        - name: shared-volume
          mountPath: /shared
```

Cả hai container `writer` và `reader` cùng mount volume này vào đường dẫn `/shared`.

```yaml
            date >> /shared/time.txt;
```

Container `writer` ghi dữ liệu vào file nằm trong volume.

```yaml
            cat /shared/time.txt
```

Container `reader` đọc dữ liệu từ cùng volume đó.

Điểm cần nhớ: `emptyDir` chia sẻ dữ liệu tốt giữa container trong cùng Pod, nhưng không phải persistent storage.

## 12. Cảnh báo về hostPath

`hostPath` cho phép Pod mount file hoặc thư mục từ filesystem của node vào container.

Ví dụ:

```yaml
volumes:
  - name: host-log
    hostPath:
      path: /var/log
      type: Directory
```

`hostPath` có thể hữu ích cho DaemonSet thu thập log hoặc agent hệ thống, nhưng không nên dùng làm storage ứng dụng thông thường vì:

- Pod bị gắn chặt với node cụ thể.
- Nếu Pod chạy sang node khác, dữ liệu không còn ở đó.
- Có rủi ro bảo mật nếu mount nhầm thư mục nhạy cảm.
- Khó backup và quản lý lifecycle.
- Không phù hợp cho dynamic provisioning.

Trong khóa học này, storage bền vững sẽ đi theo hướng chuẩn hơn: PVC, StorageClass và CSI.

## 13. PersistentVolume và PersistentVolumeClaim

### 13.1. Tại sao cần PV/PVC?

Nếu Pod trực tiếp khai báo chi tiết storage vật lý, manifest ứng dụng sẽ bị phụ thuộc vào hạ tầng.

Ví dụ không tốt về tư duy thiết kế:

```text
Ứng dụng biết rõ NFS server IP, export path, loại storage, cấu hình mount...
```

Kubernetes tách vấn đề này thành hai vai trò:

| Vai trò | Đối tượng quan tâm |
|---|---|
| Cluster admin | Tạo hoặc cấu hình storage backend |
| Application owner | Yêu cầu dung lượng và access mode |

PV/PVC giúp tách hai lớp đó.

```text
Application -> PVC -> PV -> Storage backend
```

### 13.2. PersistentVolume là gì?

PersistentVolume, viết tắt là PV, là tài nguyên storage trong cluster. PV có lifecycle độc lập với Pod.

PV có thể được tạo thủ công bởi admin hoặc được tạo tự động bởi StorageClass thông qua CSI driver.

PV mô tả thông tin như:

- Dung lượng.
- Access mode.
- Reclaim policy.
- StorageClass.
- Backend storage thực tế.
- Driver hoặc plugin được dùng.

### 13.3. PersistentVolumeClaim là gì?

PersistentVolumeClaim, viết tắt là PVC, là yêu cầu sử dụng storage từ phía workload.

PVC mô tả nhu cầu như:

- Cần bao nhiêu dung lượng.
- Cần access mode nào.
- Muốn dùng StorageClass nào.

Ví dụ:

```text
Tôi cần 1Gi storage, ReadWriteMany, từ StorageClass nfs-csi.
```

Kubernetes sẽ tìm hoặc tạo PV phù hợp để bind với PVC.

### 13.4. Quan hệ giữa Pod, PVC và PV

```text
Pod
 |
 | references
 v
PVC
 |
 | binds to
 v
PV
 |
 | backed by
 v
NFS / SAN / Ceph / Cloud Disk / Other Storage
```

Pod không mount PV trực tiếp. Pod mount PVC.

Đây là điểm  rất dễ nhầm.

```text
Pod dùng PVC
PVC bind với PV
PV đại diện cho storage thật
```

### 13.5. Access Mode

Access mode mô tả cách volume có thể được mount bởi node hoặc Pod.

| Access Mode | Ý nghĩa |
|---|---|
| `ReadWriteOnce` / `RWO` | Volume có thể được mount read-write bởi một node |
| `ReadOnlyMany` / `ROX` | Volume có thể được mount read-only bởi nhiều node |
| `ReadWriteMany` / `RWX` | Volume có thể được mount read-write bởi nhiều node |
| `ReadWriteOncePod` / `RWOP` | Volume có thể được mount read-write bởi một Pod |

NFS thường được dùng cho `ReadWriteMany`, vì nhiều Pod ở nhiều node có thể cùng mount một NFS share.

Tuy nhiên, hỗ trợ access mode thực tế phụ thuộc vào storage backend và CSI driver.

### 13.6. Reclaim Policy

Reclaim policy quyết định chuyện gì xảy ra với PV khi PVC bị xóa.

| Reclaim Policy | Ý nghĩa |
|---|---|
| `Retain` | Giữ lại PV và dữ liệu, admin xử lý thủ công |
| `Delete` | Xóa PV và yêu cầu backend xóa storage tương ứng nếu driver hỗ trợ |

Trong lab NFS CSI, nếu dùng `Delete`, khi PVC bị xóa, driver có thể xóa hoặc xử lý thư mục tương ứng trên NFS tùy tham số `onDelete` của driver.

Trong production, với dữ liệu quan trọng, cần cân nhắc kỹ giữa `Delete` và `Retain`.

## 14. Static Provisioning và Dynamic Provisioning

### 14.1. Static Provisioning

Static provisioning là cách admin tạo PV trước. Sau đó người dùng tạo PVC để bind vào PV phù hợp.

Luồng:

```text
Admin tạo PV thủ công
       |
User tạo PVC
       |
Kubernetes bind PVC với PV phù hợp
       |
Pod dùng PVC
```

Ưu điểm:

- Kiểm soát chặt chẽ storage.
- Phù hợp khi storage đã được cấp phát sẵn.
- Dễ hiểu khi học PV/PVC.

Nhược điểm:

- Thủ công.
- Không scale tốt khi có nhiều PVC.
- Admin phải tạo nhiều PV trước.

### 14.2. Dynamic Provisioning

Dynamic provisioning là cách user tạo PVC, sau đó Kubernetes dùng StorageClass và CSI driver để tự tạo PV.

Luồng:

```text
User tạo PVC có storageClassName
       |
Kubernetes gọi CSI provisioner
       |
CSI driver tạo storage backend
       |
PV được tạo tự động
       |
PVC bind với PV
       |
Pod dùng PVC
```

Ưu điểm:

- Tự động.
- Phù hợp môi trường có nhiều ứng dụng.
- Giảm thao tác thủ công cho admin.
- Gần với cách vận hành production.

Nhược điểm:

- Phụ thuộc CSI driver.
- Cần hiểu StorageClass.
- Troubleshooting phức tạp hơn static provisioning.

Trong bài này, ta sẽ giới thiệu cả static provisioning ở mức khái niệm, nhưng lab chính sẽ dùng dynamic provisioning với NFS CSI.

## 15. StorageClass

### 15.1. StorageClass là gì?

StorageClass là object mô tả một “lớp storage” trong Kubernetes.

Ví dụ trong một cluster production có thể có nhiều StorageClass:

| StorageClass | Backend | Đặc tính |
|---|---|---|
| `nfs-csi` | NFS | Shared file storage, RWX |
| `ceph-rbd` | Ceph RBD | Block storage, RWO |
| `cephfs` | CephFS | Shared filesystem, RWX |
| `fast-ssd` | SAN SSD | IOPS cao |
| `backup-retain` | Storage có reclaim Retain | Giữ dữ liệu khi xóa PVC |

Kubernetes không tự định nghĩa “nhanh” hay “chậm”. Admin là người đặt tên StorageClass và ánh xạ đến backend phù hợp.

### 15.2. Các trường quan trọng của StorageClass

| Field | Ý nghĩa |
|---|---|
| `provisioner` | Tên CSI driver hoặc provisioner xử lý StorageClass |
| `parameters` | Tham số truyền cho driver |
| `reclaimPolicy` | Chính sách xử lý PV khi PVC bị xóa |
| `volumeBindingMode` | Thời điểm bind/provision volume |
| `allowVolumeExpansion` | Có cho phép expand PVC hay không |
| `mountOptions` | Tùy chọn mount filesystem |

### 15.3. volumeBindingMode

Có hai giá trị thường gặp:

| Giá trị | Ý nghĩa |
|---|---|
| `Immediate` | Tạo/bind volume ngay khi PVC được tạo |
| `WaitForFirstConsumer` | Chờ đến khi có Pod dùng PVC rồi mới provision/bind volume |

Với NFS, `Immediate` thường đủ đơn giản cho lab vì NFS là shared storage, không phụ thuộc zone hoặc node locality phức tạp.

Với cloud disk hoặc local disk, `WaitForFirstConsumer` thường quan trọng hơn vì scheduler cần biết Pod chạy ở đâu trước khi chọn storage phù hợp.

## 16. CSI trong Kubernetes

### 16.1. CSI là gì?

CSI là viết tắt của Container Storage Interface. Đây là chuẩn giao tiếp giúp Kubernetes tích hợp với nhiều hệ thống lưu trữ khác nhau mà không cần nhúng logic của từng vendor trực tiếp vào core Kubernetes.

Trước đây, Kubernetes có nhiều in-tree volume plugin. Về lâu dài, hướng tiếp cận chuẩn là dùng CSI driver bên ngoài.

CSI driver thường chịu trách nhiệm:

- Provision volume.
- Delete volume.
- Attach/detach volume nếu backend cần.
- Mount/unmount volume vào node.
- Expand volume nếu hỗ trợ.
- Snapshot nếu driver hỗ trợ.

### 16.2. CSI Driver thường có các thành phần nào?

Một CSI driver trong Kubernetes thường có:

| Thành phần | Vai trò |
|---|---|
| Controller plugin | Xử lý provision/delete/attach/detach ở cấp control plane |
| Node plugin | Chạy trên từng node để mount/unmount volume cho Pod |
| CSI sidecars | Các container phụ như provisioner, attacher, registrar, resizer tùy driver |

Với NFS CSI,  thường thấy:

```bash
kubectl -n kube-system get pod -l app=csi-nfs-controller
kubectl -n kube-system get pod -l app=csi-nfs-node
```

Controller plugin thường chạy dạng Deployment. Node plugin thường chạy dạng DaemonSet để có mặt trên các node cần mount volume.

## 17. NFS CSI Driver trong bài lab

### 17.1. NFS CSI Driver làm gì?

NFS CSI Driver cho phép Kubernetes dynamic provision PV dựa trên một NFS server đã có sẵn.

Điểm rất quan trọng:

> NFS CSI Driver không tự dựng NFS Server. Nó cần một NFS Server đã được cấu hình trước.

Trong lab này:

- VM `nfs-server-01` cung cấp NFS export `/srv/nfs/k8s`.
- Kubernetes cluster cài NFS CSI Driver.
- StorageClass `nfs-csi` trỏ về NFS Server.
- Khi tạo PVC, NFS CSI Driver tạo thư mục con dưới `/srv/nfs/k8s`.
- PV được tạo động và bind vào PVC.
- Pod mount PVC để đọc/ghi dữ liệu.

### 17.2. Vì sao dùng NFS CSI cho lab on-premise?

NFS CSI phù hợp cho bài lab vì:

- Dễ dựng trên Ubuntu 24.04.
- Không cần cloud provider.
- Không cần SAN thật.
- Hỗ trợ shared filesystem.
- Dễ quan sát dữ liệu trực tiếp trên NFS Server.
- Giúp  hiểu rõ PVC, PV, StorageClass và CSI.

Nhưng cần nhấn mạnh giới hạn:

- NFS không phải lựa chọn tối ưu cho mọi database.
- Hiệu năng phụ thuộc network, disk, NFS server tuning.
- NFS Server là điểm rất quan trọng về HA nếu đưa vào production.
- Lab này không xử lý HA cho NFS Server.

## 18. Chuẩn bị NFS Server VM

Thực hiện trên VM `nfs-server-01`.

### 18.1. Cài đặt NFS Server

```bash
sudo apt update
sudo apt install -y nfs-kernel-server
```

Kiểm tra service:

```bash
systemctl status nfs-server --no-pager
```

### 18.2. Tạo thư mục export

```bash
sudo mkdir -p /srv/nfs/k8s
sudo chown -R nobody:nogroup /srv/nfs/k8s
sudo chmod 0777 /srv/nfs/k8s
```

Trong lab, dùng permission `0777` để giảm lỗi phân quyền khi  mới học. Trong production, không nên dùng đơn giản như vậy mà cần thiết kế UID/GID, quyền ghi và bảo mật NFS cẩn thận hơn.

### 18.3. Cấu hình `/etc/exports`

Mở file:

```bash
sudo nano /etc/exports
```

Thêm dòng sau, giả định subnet lab là `10.10.10.0/24`:

```text
/srv/nfs/k8s 10.10.10.0/24(rw,sync,no_subtree_check,no_root_squash)
```

Áp dụng cấu hình:

```bash
sudo exportfs -rav
sudo systemctl restart nfs-server
```

Kiểm tra export:

```bash
sudo exportfs -v
showmount -e 127.0.0.1
```

Nếu chưa có `showmount`, cài thêm:

```bash
sudo apt install -y nfs-common
```

### 18.4. Giải thích cấu hình NFS export

```text
/srv/nfs/k8s
```

Đường dẫn thư mục được export.

```text
10.10.10.0/24
```

Chỉ cho phép các node trong subnet lab truy cập.

```text
rw
```

Cho phép đọc và ghi.

```text
sync
```

Ghi đồng bộ, an toàn hơn so với async nhưng có thể chậm hơn.

```text
no_subtree_check
```

Giảm một số vấn đề kiểm tra subtree khi export thư mục con.

```text
no_root_squash
```

Cho phép root từ client giữ quyền root trên export. Trong lab giúp tránh lỗi permission. Trong production cần cân nhắc rất kỹ vì có rủi ro bảo mật.

## 19. Chuẩn bị các Kubernetes node để mount NFS

Thực hiện trên tất cả node Kubernetes:

- `k8s-master-01`
- `k8s-worker-01`
- `k8s-worker-02`

Cài NFS client package:

```bash
sudo apt update
sudo apt install -y nfs-common
```

Kiểm tra từ mỗi node:

```bash
showmount -e 10.10.10.50
```

Kết quả mong đợi:

```text
Export list for 10.10.10.50:
/srv/nfs/k8s 10.10.10.0/24
```

Test mount thủ công trên một worker:

```bash
sudo mkdir -p /mnt/test-nfs
sudo mount -t nfs -o nfsvers=4.1 10.10.10.50:/srv/nfs/k8s /mnt/test-nfs
mount | grep test-nfs
```

Tạo file test:

```bash
echo "hello from $(hostname)" | sudo tee /mnt/test-nfs/test-from-$(hostname).txt
ls -l /mnt/test-nfs
```

Unmount sau khi test:

```bash
sudo umount /mnt/test-nfs
```

Kiểm tra trên NFS Server:

```bash
ls -l /srv/nfs/k8s
```

Nếu bước mount thủ công không chạy, không nên cài CSI vội. Phải xử lý NFS trước.

## 20. Cài đặt NFS CSI Driver

Lab này dùng cách cài bằng script chính thức từ repository `kubernetes-csi/csi-driver-nfs`, không dùng Helm để giữ đúng tiến trình khóa học.

Thực hiện trên `k8s-master-01` nơi có `kubectl` admin config.

### 20.1. Cài driver

Ví dụ dùng version `v4.13.2`:

```bash
curl -skSL https://raw.githubusercontent.com/kubernetes-csi/csi-driver-nfs/v4.13.2/deploy/install-driver.sh | bash -s v4.13.2 --
```

### 20.2. Kiểm tra Pod của NFS CSI

```bash
kubectl -n kube-system get pod -o wide -l app=csi-nfs-controller
kubectl -n kube-system get pod -o wide -l app=csi-nfs-node
```

Kết quả mong đợi:

```text
NAME                                  READY   STATUS    RESTARTS   AGE
csi-nfs-controller-xxxxxxxxxx-xxxxx   4/4     Running   0          1m

NAME                 READY   STATUS    RESTARTS   AGE
csi-nfs-node-xxxxx   3/3     Running   0          1m
csi-nfs-node-yyyyy   3/3     Running   0          1m
csi-nfs-node-zzzzz   3/3     Running   0          1m
```

Số lượng `csi-nfs-node` thường tương ứng với số node mà DaemonSet được schedule lên.

### 20.3. Kiểm tra CSIDriver object

```bash
kubectl get csidriver
```

Kỳ vọng có driver:

```text
nfs.csi.k8s.io
```

Kiểm tra tài nguyên liên quan:

```bash
kubectl -n kube-system get deploy,ds | grep csi-nfs
```

## 21. Tạo StorageClass cho NFS CSI

Tạo file `08-storageclass-nfs-csi.yaml`:

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: nfs-csi
provisioner: nfs.csi.k8s.io
parameters:
  server: 10.10.10.50
  share: /srv/nfs/k8s
  subDir: ${pvc.metadata.namespace}-${pvc.metadata.name}-${pv.metadata.name}
  mountPermissions: "0777"
  onDelete: delete
reclaimPolicy: Delete
volumeBindingMode: Immediate
allowVolumeExpansion: true
mountOptions:
  - nfsvers=4.1
```

Áp dụng:

```bash
kubectl apply -f 08-storageclass-nfs-csi.yaml
```

Kiểm tra:

```bash
kubectl get storageclass
kubectl describe storageclass nfs-csi
```

### 21.1. Giải thích StorageClass manifest

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
```

StorageClass thuộc API group `storage.k8s.io/v1`.

```yaml
metadata:
  name: nfs-csi
```

Tên StorageClass. PVC sẽ tham chiếu đến tên này bằng `storageClassName: nfs-csi`.

```yaml
provisioner: nfs.csi.k8s.io
```

Chỉ định provisioner xử lý StorageClass này. Đây là tên driver của NFS CSI.

```yaml
parameters:
  server: 10.10.10.50
  share: /srv/nfs/k8s
```

Khai báo NFS Server và export path.

```yaml
  subDir: ${pvc.metadata.namespace}-${pvc.metadata.name}-${pv.metadata.name}
```

Khi PVC được tạo, driver tạo thư mục con dưới NFS share. Cách đặt tên này giúp dễ truy vết thư mục tương ứng với namespace, PVC và PV.

```yaml
  mountPermissions: "0777"
```

Yêu cầu driver set quyền cho thư mục được mount. Đây là cấu hình tiện cho lab. Production cần thiết kế quyền chặt hơn.

```yaml
  onDelete: delete
```

Khi volume bị xóa, driver xử lý thư mục theo chính sách delete. NFS CSI còn có các lựa chọn như retain hoặc archive tùy nhu cầu.

```yaml
reclaimPolicy: Delete
```

Khi PVC bị xóa, PV động cũng bị xóa theo.

```yaml
volumeBindingMode: Immediate
```

PVC sẽ được provision ngay khi tạo, không chờ Pod đầu tiên sử dụng.

```yaml
allowVolumeExpansion: true
```

Cho phép mở rộng PVC nếu driver/backend hỗ trợ.

```yaml
mountOptions:
  - nfsvers=4.1
```

Chỉ định mount bằng NFS version 4.1.

## 22. Tạo PVC dùng StorageClass NFS CSI

Tạo file `09-pvc-nfs-csi.yaml`:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: web-content-pvc
  namespace: default
spec:
  accessModes:
    - ReadWriteMany
  storageClassName: nfs-csi
  resources:
    requests:
      storage: 1Gi
```

Áp dụng:

```bash
kubectl apply -f 09-pvc-nfs-csi.yaml
```

Kiểm tra PVC:

```bash
kubectl get pvc
kubectl describe pvc web-content-pvc
```

Kết quả mong đợi:

```text
NAME              STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
web-content-pvc   Bound    pvc-xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx   1Gi        RWX            nfs-csi        10s
```

Kiểm tra PV được tạo động:

```bash
kubectl get pv
kubectl describe pv <pv-name>
```

Kiểm tra thư mục trên NFS Server:

```bash
ls -lah /srv/nfs/k8s
```

### 22.1. Giải thích PVC manifest

```yaml
kind: PersistentVolumeClaim
```

Object yêu cầu storage từ Kubernetes.

```yaml
metadata:
  name: web-content-pvc
```

Tên PVC mà Pod sẽ tham chiếu.

```yaml
spec:
  accessModes:
    - ReadWriteMany
```

PVC yêu cầu volume có thể được mount read-write bởi nhiều node/Pod. Đây là kiểu phù hợp với NFS.

```yaml
  storageClassName: nfs-csi
```

Yêu cầu Kubernetes dùng StorageClass `nfs-csi` để provision volume.

```yaml
  resources:
    requests:
      storage: 1Gi
```

Yêu cầu dung lượng 1Gi.

Lưu ý với NFS: quota thực tế không phải lúc nào cũng được enforce như block storage. PVC vẫn hiển thị capacity, nhưng khả năng giới hạn dung lượng thật phụ thuộc backend và driver.

## 23. Mount PVC vào Pod

Tạo file `10-pod-pvc-writer.yaml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-pvc-writer
  labels:
    app: pvc-writer
spec:
  containers:
    - name: writer
      image: busybox:1.36
      command: ["/bin/sh", "-c"]
      args:
        - |
          echo "Pod name: $HOSTNAME" > /data/index.html;
          echo "Written at: $(date)" >> /data/index.html;
          echo "This file is stored on PVC backed by NFS CSI." >> /data/index.html;
          sleep 3600;
      volumeMounts:
        - name: web-content
          mountPath: /data
  volumes:
    - name: web-content
      persistentVolumeClaim:
        claimName: web-content-pvc
```

Áp dụng:

```bash
kubectl apply -f 10-pod-pvc-writer.yaml
```

Kiểm tra:

```bash
kubectl get pod pod-pvc-writer -o wide
kubectl logs pod-pvc-writer
kubectl exec -it pod-pvc-writer -- cat /data/index.html
```

Kiểm tra trên NFS Server:

```bash
sudo find /srv/nfs/k8s -type f -name index.html -exec echo "--- {}" \; -exec cat {} \;
```

### 23.1. Giải thích Pod dùng PVC

```yaml
      volumeMounts:
        - name: web-content
          mountPath: /data
```

Mount volume tên `web-content` vào thư mục `/data` trong container.

```yaml
  volumes:
    - name: web-content
      persistentVolumeClaim:
        claimName: web-content-pvc
```

Volume `web-content` lấy storage từ PVC `web-content-pvc`.

Pod không cần biết NFS Server là IP nào. Pod chỉ biết PVC name.

Đây là abstraction rất quan trọng.

## 24. Kiểm tra tính persistent của PVC

Xóa Pod:

```bash
kubectl delete pod pod-pvc-writer
```

Tạo lại Pod bằng cùng manifest:

```bash
kubectl apply -f 10-pod-pvc-writer.yaml
```

Kiểm tra file:

```bash
kubectl exec -it pod-pvc-writer -- cat /data/index.html
```

Nếu file vẫn tồn tại hoặc được ghi lại trên cùng PVC,  sẽ thấy dữ liệu nằm ngoài vòng đời của container.

Để kiểm tra rõ hơn, sửa command trong Pod để không ghi đè file, hoặc tạo một Pod reader chỉ đọc dữ liệu.

Tạo file `11-pod-pvc-reader.yaml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-pvc-reader
  labels:
    app: pvc-reader
spec:
  containers:
    - name: reader
      image: busybox:1.36
      command: ["/bin/sh", "-c"]
      args:
        - |
          echo "Reading data from PVC";
          cat /data/index.html;
          sleep 3600;
      volumeMounts:
        - name: web-content
          mountPath: /data
  volumes:
    - name: web-content
      persistentVolumeClaim:
        claimName: web-content-pvc
```

Áp dụng:

```bash
kubectl apply -f 11-pod-pvc-reader.yaml
kubectl logs pod-pvc-reader
```

Vì PVC dùng `ReadWriteMany`, nhiều Pod có thể mount cùng PVC trong lab này.

## 25. Deployment dùng PVC, ConfigMap và Secret

Phần này tổng hợp ba nhóm kiến thức:

- Deployment từ Buổi 03.
- Service từ Buổi 04.
- ConfigMap, Secret và PVC từ Buổi 05.

Mục tiêu: chạy Nginx bằng Deployment, cấu hình Nginx từ ConfigMap, mount nội dung web từ PVC, inject một số biến môi trường từ ConfigMap và Secret.

### 25.1. Tạo Job ghi nội dung web vào PVC

Vì Nginx container chỉ phục vụ file, ta dùng Job để tạo file `index.html` trên PVC trước.

Tạo file `12-job-init-web-content.yaml`:

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: init-web-content
spec:
  template:
    metadata:
      labels:
        app: init-web-content
    spec:
      restartPolicy: OnFailure
      containers:
        - name: init
          image: busybox:1.36
          command: ["/bin/sh", "-c"]
          args:
            - |
              cat > /data/index.html <<'EOF'
              <html>
              <head><title>Kubernetes Storage Lab</title></head>
              <body>
                <h1>Hello from Kubernetes PVC</h1>
                <p>This page is stored on NFS CSI backed PersistentVolumeClaim.</p>
              </body>
              </html>
              EOF
              echo "Web content initialized."
          volumeMounts:
            - name: web-content
              mountPath: /data
      volumes:
        - name: web-content
          persistentVolumeClaim:
            claimName: web-content-pvc
```

Áp dụng:

```bash
kubectl apply -f 12-job-init-web-content.yaml
kubectl get job
kubectl logs job/init-web-content
```

### 25.2. Tạo Deployment Nginx dùng ConfigMap, Secret và PVC

Tạo file `13-deployment-nginx-pvc-config-secret.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-storage-demo
  labels:
    app: nginx-storage-demo
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nginx-storage-demo
  template:
    metadata:
      labels:
        app: nginx-storage-demo
    spec:
      containers:
        - name: nginx
          image: nginx:1.27-alpine
          ports:
            - containerPort: 80
          env:
            - name: APP_ENV
              valueFrom:
                configMapKeyRef:
                  name: webapp-config
                  key: APP_ENV
            - name: LOG_LEVEL
              valueFrom:
                configMapKeyRef:
                  name: webapp-config
                  key: LOG_LEVEL
            - name: API_TOKEN
              valueFrom:
                secretKeyRef:
                  name: webapp-secret
                  key: API_TOKEN
          volumeMounts:
            - name: web-content
              mountPath: /usr/share/nginx/html
            - name: nginx-config
              mountPath: /etc/nginx/conf.d/default.conf
              subPath: nginx.conf
              readOnly: true
            - name: app-secret
              mountPath: /etc/app-secret
              readOnly: true
      volumes:
        - name: web-content
          persistentVolumeClaim:
            claimName: web-content-pvc
        - name: nginx-config
          configMap:
            name: webapp-config
            items:
              - key: nginx.conf
                path: nginx.conf
        - name: app-secret
          secret:
            secretName: webapp-secret
```

Áp dụng:

```bash
kubectl apply -f 13-deployment-nginx-pvc-config-secret.yaml
```

Kiểm tra:

```bash
kubectl get deployment nginx-storage-demo
kubectl get pod -l app=nginx-storage-demo -o wide
kubectl describe deployment nginx-storage-demo
```

### 25.3. Tạo Service NodePort để truy cập web

Tạo file `14-service-nginx-storage-demo.yaml`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-storage-demo-svc
spec:
  type: NodePort
  selector:
    app: nginx-storage-demo
  ports:
    - name: http
      protocol: TCP
      port: 80
      targetPort: 80
      nodePort: 30085
```

Áp dụng:

```bash
kubectl apply -f 14-service-nginx-storage-demo.yaml
```

Kiểm tra:

```bash
kubectl get svc nginx-storage-demo-svc
kubectl get endpointslice -l kubernetes.io/service-name=nginx-storage-demo-svc
```

Truy cập từ máy ngoài cluster:

```bash
curl http://10.10.10.21:30085
curl http://10.10.10.22:30085
```

### 25.4. Giải thích Deployment tổng hợp

```yaml
          env:
            - name: APP_ENV
              valueFrom:
                configMapKeyRef:
                  name: webapp-config
                  key: APP_ENV
```

Container lấy biến môi trường từ ConfigMap.

```yaml
            - name: API_TOKEN
              valueFrom:
                secretKeyRef:
                  name: webapp-secret
                  key: API_TOKEN
```

Container lấy biến môi trường từ Secret.

```yaml
            - name: web-content
              mountPath: /usr/share/nginx/html
```

Mount PVC vào thư mục web root của Nginx.

```yaml
            - name: nginx-config
              mountPath: /etc/nginx/conf.d/default.conf
              subPath: nginx.conf
              readOnly: true
```

Mount file `nginx.conf` từ ConfigMap vào đúng vị trí cấu hình Nginx.

```yaml
            - name: app-secret
              mountPath: /etc/app-secret
              readOnly: true
```

Mount Secret thành file trong container.

```yaml
      volumes:
        - name: web-content
          persistentVolumeClaim:
            claimName: web-content-pvc
```

Volume lấy từ PVC.

```yaml
        - name: nginx-config
          configMap:
            name: webapp-config
```

Volume lấy từ ConfigMap.

```yaml
        - name: app-secret
          secret:
            secretName: webapp-secret
```

Volume lấy từ Secret.

## 26. Kiểm tra dữ liệu khi Pod trong Deployment bị xóa

Liệt kê Pod:

```bash
kubectl get pod -l app=nginx-storage-demo -o wide
```

Xóa một Pod bất kỳ:

```bash
kubectl delete pod <nginx-pod-name>
```

ReplicaSet sẽ tạo Pod mới.

Kiểm tra lại:

```bash
kubectl get pod -l app=nginx-storage-demo -o wide
curl http://10.10.10.21:30085
```

Nếu trang web vẫn trả về nội dung từ PVC,  sẽ thấy:

- Pod có thể bị thay thế.
- Container filesystem không phải nơi giữ dữ liệu bền vững.
- Dữ liệu web nằm trên PVC backed by NFS.
- Deployment có thể tạo Pod mới và mount lại cùng PVC.

## 27. Static Provisioning với NFS CSI ở mức minh họa

Phần này giúp  hiểu PV có thể được tạo thủ công. Không bắt buộc chạy nếu lab chính đã đủ thời gian.

### 27.1. Tạo thư mục riêng trên NFS Server

Trên `nfs-server-01`:

```bash
sudo mkdir -p /srv/nfs/k8s/static-demo
sudo chmod 0777 /srv/nfs/k8s/static-demo
```

### 27.2. Tạo PV static dùng CSI NFS

Tạo file `15-static-pv-nfs-csi.yaml`:

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: static-nfs-pv
spec:
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteMany
  persistentVolumeReclaimPolicy: Retain
  storageClassName: static-nfs
  mountOptions:
    - nfsvers=4.1
  csi:
    driver: nfs.csi.k8s.io
    volumeHandle: 10.10.10.50#/srv/nfs/k8s#static-demo
    volumeAttributes:
      server: 10.10.10.50
      share: /srv/nfs/k8s
      subDir: static-demo
```

Áp dụng:

```bash
kubectl apply -f 15-static-pv-nfs-csi.yaml
```

### 27.3. Tạo PVC bind vào PV static

Tạo file `16-static-pvc-nfs-csi.yaml`:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: static-nfs-pvc
spec:
  accessModes:
    - ReadWriteMany
  storageClassName: static-nfs
  resources:
    requests:
      storage: 1Gi
```

Áp dụng:

```bash
kubectl apply -f 16-static-pvc-nfs-csi.yaml
```

Kiểm tra:

```bash
kubectl get pv static-nfs-pv
kubectl get pvc static-nfs-pvc
```

### 27.4. Giải thích static PV

```yaml
kind: PersistentVolume
```

Ở đây admin tạo PV thủ công.

```yaml
persistentVolumeReclaimPolicy: Retain
```

Khi PVC bị xóa, PV và dữ liệu được giữ lại để admin xử lý thủ công.

```yaml
storageClassName: static-nfs
```

PVC phải yêu cầu cùng `storageClassName` để bind phù hợp.

```yaml
  csi:
    driver: nfs.csi.k8s.io
```

PV này dùng CSI driver NFS.

```yaml
    volumeHandle: 10.10.10.50#/srv/nfs/k8s#static-demo
```

Định danh duy nhất cho volume trong driver.

```yaml
    volumeAttributes:
      server: 10.10.10.50
      share: /srv/nfs/k8s
      subDir: static-demo
```

Thông tin NFS backend.

## 28. Lab tổng hợp cuối buổi

### 28.1. Mục tiêu lab

 triển khai một ứng dụng web Nginx có:

- Cấu hình Nginx lấy từ ConfigMap.
- Biến môi trường thường lấy từ ConfigMap.
- API token lấy từ Secret.
- Nội dung web lưu trên PVC.
- PVC được dynamic provision bằng NFS CSI.
- Service NodePort để truy cập ứng dụng từ bên ngoài.
- Kiểm tra dữ liệu còn tồn tại sau khi Pod bị xóa.

### 28.2. Danh sách manifest sử dụng

| File | Mục đích |
|---|---|
| `01-configmap-app.yaml` | Tạo ConfigMap |
| `04-secret-app.yaml` | Tạo Secret |
| `08-storageclass-nfs-csi.yaml` | Tạo StorageClass NFS CSI |
| `09-pvc-nfs-csi.yaml` | Tạo PVC |
| `12-job-init-web-content.yaml` | Ghi nội dung web vào PVC |
| `13-deployment-nginx-pvc-config-secret.yaml` | Chạy Nginx Deployment |
| `14-service-nginx-storage-demo.yaml` | Expose Nginx bằng NodePort |

### 28.3. Thứ tự triển khai

```bash
kubectl apply -f 01-configmap-app.yaml
kubectl apply -f 04-secret-app.yaml
kubectl apply -f 08-storageclass-nfs-csi.yaml
kubectl apply -f 09-pvc-nfs-csi.yaml
kubectl apply -f 12-job-init-web-content.yaml
kubectl apply -f 13-deployment-nginx-pvc-config-secret.yaml
kubectl apply -f 14-service-nginx-storage-demo.yaml
```

### 28.4. Kiểm tra tổng thể

```bash
kubectl get configmap webapp-config
kubectl get secret webapp-secret
kubectl get storageclass nfs-csi
kubectl get pvc web-content-pvc
kubectl get pv
kubectl get job init-web-content
kubectl get deployment nginx-storage-demo
kubectl get pod -l app=nginx-storage-demo -o wide
kubectl get svc nginx-storage-demo-svc
```

### 28.5. Kiểm tra truy cập web

```bash
curl http://10.10.10.21:30085
curl http://10.10.10.22:30085
```

### 28.6. Kiểm tra mount trong Pod

Lấy tên một Pod Nginx:

```bash
POD_NAME=$(kubectl get pod -l app=nginx-storage-demo -o jsonpath='{.items[0].metadata.name}')
echo $POD_NAME
```

Kiểm tra file web:

```bash
kubectl exec -it $POD_NAME -- ls -l /usr/share/nginx/html
kubectl exec -it $POD_NAME -- cat /usr/share/nginx/html/index.html
```

Kiểm tra Secret file:

```bash
kubectl exec -it $POD_NAME -- ls -l /etc/app-secret
```

Kiểm tra env:

```bash
kubectl exec -it $POD_NAME -- printenv | grep -E 'APP_ENV|LOG_LEVEL|API_TOKEN'
```

Lưu ý: việc in `API_TOKEN` ra màn hình chỉ dùng cho lab. Không nên làm trong production.

### 28.7. Kiểm tra self-healing kết hợp persistent data

Xóa một Pod:

```bash
kubectl delete pod $POD_NAME
```

Chờ Pod mới được tạo:

```bash
kubectl get pod -l app=nginx-storage-demo -w
```

Kiểm tra truy cập lại:

```bash
curl http://10.10.10.21:30085
```

Kết luận mong đợi:

- Deployment tạo lại Pod.
- Pod mới mount lại PVC.
- Dữ liệu web vẫn còn.
- ConfigMap và Secret vẫn được inject vào Pod mới.

## 29. Troubleshooting ConfigMap và Secret

### 29.1. Pod không start do thiếu ConfigMap

Triệu chứng:

```bash
kubectl get pod
```

Pod có thể ở trạng thái:

```text
CreateContainerConfigError
```

Kiểm tra:

```bash
kubectl describe pod <pod-name>
```

Có thể thấy lỗi kiểu:

```text
configmap "webapp-config" not found
```

Cách xử lý:

```bash
kubectl get configmap
kubectl apply -f 01-configmap-app.yaml
```

### 29.2. Pod không start do thiếu Secret

Kiểm tra:

```bash
kubectl describe pod <pod-name>
```

Lỗi thường gặp:

```text
secret "webapp-secret" not found
```

Cách xử lý:

```bash
kubectl get secret
kubectl apply -f 04-secret-app.yaml
```

### 29.3. Key trong ConfigMap hoặc Secret sai

Nếu object có tồn tại nhưng key không đúng, Pod vẫn có thể lỗi.

Kiểm tra key:

```bash
kubectl get configmap webapp-config -o yaml
kubectl get secret webapp-secret -o yaml
```

Với Secret, decode giá trị:

```bash
kubectl get secret webapp-secret -o jsonpath='{.data.API_TOKEN}' | base64 -d; echo
```

### 29.4. Sửa ConfigMap nhưng ứng dụng chưa nhận cấu hình mới

Nguyên nhân có thể:

- ConfigMap được inject qua env, Pod cần restart.
- ConfigMap mount qua volume nhưng cần thời gian đồng bộ.
- Đang dùng `subPath`, file mount không tự update.
- Ứng dụng không tự reload config.

Cách xử lý đơn giản với Deployment:

```bash
kubectl rollout restart deployment nginx-storage-demo
```

## 30. Troubleshooting PVC, PV, StorageClass và NFS CSI

### 30.1. PVC Pending

Kiểm tra:

```bash
kubectl get pvc
kubectl describe pvc web-content-pvc
```

Nguyên nhân thường gặp:

- Sai `storageClassName`.
- StorageClass chưa tồn tại.
- CSI controller chưa chạy.
- NFS CSI provisioner lỗi.
- Tham số `server` hoặc `share` sai.

Kiểm tra StorageClass:

```bash
kubectl get storageclass
kubectl describe storageclass nfs-csi
```

Kiểm tra CSI controller:

```bash
kubectl -n kube-system get pod -l app=csi-nfs-controller
kubectl -n kube-system logs -l app=csi-nfs-controller --all-containers=true --tail=100
```

### 30.2. Pod Pending do PVC chưa Bound

Kiểm tra Pod:

```bash
kubectl describe pod <pod-name>
```

Có thể thấy event liên quan PVC chưa bound.

Kiểm tra PVC:

```bash
kubectl get pvc
kubectl describe pvc <pvc-name>
```

Cần xử lý PVC trước, không xử lý Pod trước.

### 30.3. Pod ContainerCreating do mount NFS lỗi

Triệu chứng:

```bash
kubectl get pod
```

Pod có thể ở trạng thái:

```text
ContainerCreating
```

Kiểm tra:

```bash
kubectl describe pod <pod-name>
```

Lỗi có thể liên quan:

```text
MountVolume.MountDevice failed
mount.nfs: bad option
connection timed out
permission denied
```

Cách xử lý:

Kiểm tra `nfs-common` trên node:

```bash
sudo apt install -y nfs-common
```

Kiểm tra node có mount được NFS thủ công không:

```bash
showmount -e 10.10.10.50
sudo mount -t nfs -o nfsvers=4.1 10.10.10.50:/srv/nfs/k8s /mnt/test-nfs
```

Kiểm tra firewall giữa node và NFS Server.

Kiểm tra export trên NFS Server:

```bash
sudo exportfs -v
sudo systemctl status nfs-server --no-pager
```

### 30.4. Permission denied khi ghi vào PVC

Nguyên nhân thường gặp:

- Quyền thư mục NFS không phù hợp.
- UID/GID trong container không có quyền ghi.
- NFS export đang bật root squash.
- `mountPermissions` không đúng hoặc driver không chmod như mong đợi.

Trong lab, có thể xử lý đơn giản trên NFS Server:

```bash
sudo chmod -R 0777 /srv/nfs/k8s
sudo chown -R nobody:nogroup /srv/nfs/k8s
```

Production không nên xử lý thô như vậy. Cần thiết kế UID/GID và quyền thư mục chuẩn.

### 30.5. Xóa PVC nhưng dữ liệu vẫn còn trên NFS

Kiểm tra reclaim policy và tham số `onDelete`:

```bash
kubectl get pv
kubectl describe storageclass nfs-csi
```

Nếu `onDelete: retain` hoặc `archive`, thư mục có thể được giữ lại hoặc đổi tên.

Nếu `reclaimPolicy: Retain`, PV không bị xóa tự động.

Cần phân biệt:

```text
reclaimPolicy của Kubernetes PV
và
onDelete behavior của NFS CSI driver
```

## 31. Các lệnh quan sát quan trọng

### 31.1. ConfigMap và Secret

```bash
kubectl get configmap
kubectl describe configmap webapp-config
kubectl get configmap webapp-config -o yaml

kubectl get secret
kubectl describe secret webapp-secret
kubectl get secret webapp-secret -o yaml
```

### 31.2. Volume trong Pod

```bash
kubectl describe pod <pod-name>
kubectl exec -it <pod-name> -- mount
kubectl exec -it <pod-name> -- df -h
```

### 31.3. PV, PVC và StorageClass

```bash
kubectl get storageclass
kubectl describe storageclass nfs-csi

kubectl get pvc
kubectl describe pvc web-content-pvc

kubectl get pv
kubectl describe pv <pv-name>
```

### 31.4. CSI Driver

```bash
kubectl get csidriver
kubectl -n kube-system get pod -l app=csi-nfs-controller -o wide
kubectl -n kube-system get pod -l app=csi-nfs-node -o wide
kubectl -n kube-system logs -l app=csi-nfs-controller --all-containers=true --tail=100
kubectl -n kube-system logs -l app=csi-nfs-node --all-containers=true --tail=100
```

### 31.5. NFS Server

```bash
sudo exportfs -v
showmount -e 127.0.0.1
ls -lah /srv/nfs/k8s
sudo journalctl -u nfs-server -n 100 --no-pager
```

## 32. Cleanup lab

Xóa workload:

```bash
kubectl delete -f 14-service-nginx-storage-demo.yaml
kubectl delete -f 13-deployment-nginx-pvc-config-secret.yaml
kubectl delete -f 12-job-init-web-content.yaml
kubectl delete -f 11-pod-pvc-reader.yaml --ignore-not-found
kubectl delete -f 10-pod-pvc-writer.yaml --ignore-not-found
kubectl delete -f 07-pod-emptydir-multicontainer.yaml --ignore-not-found
kubectl delete -f 06-pod-secret-volume.yaml --ignore-not-found
kubectl delete -f 05-pod-secret-env.yaml --ignore-not-found
kubectl delete -f 03-pod-configmap-volume.yaml --ignore-not-found
kubectl delete -f 02-pod-configmap-env.yaml --ignore-not-found
```

Xóa PVC:

```bash
kubectl delete -f 09-pvc-nfs-csi.yaml
```

Kiểm tra PV:

```bash
kubectl get pv
```

Xóa StorageClass:

```bash
kubectl delete -f 08-storageclass-nfs-csi.yaml
```

Xóa ConfigMap và Secret:

```bash
kubectl delete -f 04-secret-app.yaml
kubectl delete -f 01-configmap-app.yaml
```

Nếu đã tạo static PV/PVC:

```bash
kubectl delete -f 16-static-pvc-nfs-csi.yaml --ignore-not-found
kubectl delete -f 15-static-pv-nfs-csi.yaml --ignore-not-found
```

Gỡ NFS CSI driver nếu muốn reset môi trường:

```bash
curl -skSL https://raw.githubusercontent.com/kubernetes-csi/csi-driver-nfs/v4.13.2/deploy/uninstall-driver.sh | bash -s v4.13.2 --
```

Xóa dữ liệu lab trên NFS Server nếu chắc chắn không cần giữ:

```bash
sudo rm -rf /srv/nfs/k8s/*
```

## 33. Checklist kiến thức cuối buổi

Sau buổi này,  cần trả lời được các câu hỏi sau:

- ConfigMap dùng để làm gì?
- Secret khác ConfigMap ở điểm nào?
- Secret có được mã hóa mặc định trong etcd không?
- Vì sao không nên lưu password trong ConfigMap?
- Có mấy cách phổ biến để Pod consume ConfigMap?
- Có mấy cách phổ biến để Pod consume Secret?
- Environment variable từ ConfigMap có tự update trong container đang chạy không?
- Mount ConfigMap bằng volume khác gì inject qua environment variable?
- `subPath` có ảnh hưởng gì đến việc update ConfigMap/Secret volume?
- Volume trong Kubernetes là gì?
- `emptyDir` tồn tại trong vòng đời nào?
- Vì sao `emptyDir` không phải persistent storage?
- Vì sao không nên lạm dụng `hostPath`?
- PV là gì?
- PVC là gì?
- Pod mount PV trực tiếp hay mount PVC?
- StorageClass là gì?
- Static provisioning khác dynamic provisioning thế nào?
- CSI là gì?
- NFS CSI Driver có tự dựng NFS Server không?
- Vì sao cần cài `nfs-common` trên Kubernetes node?
- PVC `Pending` thường do các nguyên nhân nào?
- Pod `ContainerCreating` khi dùng PVC thường cần kiểm tra gì?
- `ReadWriteMany` có ý nghĩa gì?
- `reclaimPolicy: Delete` khác `Retain` thế nào?

## 34. Câu hỏi kiểm tra cuối buổi

### Câu 1

ConfigMap phù hợp nhất để lưu loại dữ liệu nào?

A. Database password thật  
B. TLS private key  
C. Log level và application mode  
D. API token production  

Đáp án đúng: C.

Giải thích: ConfigMap dùng cho cấu hình không nhạy cảm. Password, private key và API token nên dùng Secret hoặc hệ thống quản lý secret chuyên dụng.

### Câu 2

Secret trong Kubernetes mặc định có phải là encrypted data trong etcd không?

A. Có, luôn được mã hóa mặc định  
B. Không nhất thiết, cần bật encryption at rest nếu muốn mã hóa ở etcd  
C. Có, vì Secret dùng base64  
D. Không lưu trong etcd  

Đáp án đúng: B.

Giải thích: Base64 chỉ là encoding, không phải encryption. Muốn tăng bảo mật cần cấu hình encryption at rest, RBAC và các biện pháp bảo vệ khác.

### Câu 3

Pod sử dụng PVC bằng cách nào?

A. Pod mount trực tiếp StorageClass  
B. Pod mount trực tiếp CSI Driver  
C. Pod khai báo volume loại `persistentVolumeClaim` tham chiếu đến PVC  
D. Pod SSH vào NFS Server  

Đáp án đúng: C.

Giải thích: Pod không mount StorageClass hay CSI driver trực tiếp. Pod tham chiếu PVC, PVC bind với PV, PV đại diện cho backend storage.

### Câu 4

`emptyDir` mất dữ liệu khi nào?

A. Khi container trong Pod restart  
B. Khi Pod bị xóa khỏi node  
C. Khi đọc file bằng `kubectl exec`  
D. Khi Service bị xóa  

Đáp án đúng: B.

Giải thích: `emptyDir` tồn tại trong vòng đời Pod. Container restart trong cùng Pod thường không làm mất dữ liệu `emptyDir`, nhưng Pod bị xóa thì dữ liệu mất.

### Câu 5

Trong lab NFS CSI, thành phần nào cung cấp NFS export thật sự?

A. kube-apiserver  
B. kubelet  
C. NFS CSI Driver  
D. VM `nfs-server-01`  

Đáp án đúng: D.

Giải thích: NFS CSI Driver không tự dựng NFS Server. Nó dùng NFS Server đã có sẵn để provision PV.

### Câu 6

PVC ở trạng thái `Pending`, bước kiểm tra hợp lý nhất là gì?

A. Xóa ngay kube-apiserver  
B. Kiểm tra `kubectl describe pvc`, StorageClass và CSI controller  
C. Restart tất cả VM  
D. Xóa Calico  

Đáp án đúng: B.

Giải thích: PVC Pending thường liên quan đến StorageClass, provisioner, CSI controller hoặc backend storage. Cần đọc event của PVC trước.

### Câu 7

Access mode nào thường phù hợp với NFS để nhiều Pod ở nhiều node cùng đọc/ghi?

A. ReadWriteOnce  
B. ReadOnlyMany  
C. ReadWriteMany  
D. HostPathOnly  

Đáp án đúng: C.

Giải thích: NFS là shared filesystem, thường phù hợp cho RWX nếu driver và export cho phép.

### Câu 8

`storageClassName` trong PVC dùng để làm gì?

A. Chọn namespace chạy Pod  
B. Chọn Service expose ứng dụng  
C. Chọn lớp storage dùng để provision hoặc bind volume  
D. Chọn image container  

Đáp án đúng: C.

Giải thích: `storageClassName` giúp PVC yêu cầu một loại StorageClass cụ thể.

### Câu 9

Khi ConfigMap được inject vào container bằng environment variable, sau khi sửa ConfigMap, ứng dụng đang chạy thường cần gì để nhận giá trị mới?

A. Restart hoặc recreate Pod  
B. Xóa Service  
C. Xóa StorageClass  
D. Tắt CoreDNS  

Đáp án đúng: A.

Giải thích: Env var được thiết lập khi container start. Muốn nhận giá trị mới thường cần restart Pod hoặc rollout lại Deployment.

### Câu 10

Trong StorageClass NFS CSI, `provisioner: nfs.csi.k8s.io` có ý nghĩa gì?

A. Dùng kube-proxy để tạo volume  
B. Chỉ định NFS CSI Driver xử lý dynamic provisioning cho PVC dùng StorageClass này  
C. Chỉ định Pod chạy trên master node  
D. Chỉ định Service dùng NodePort  

Đáp án đúng: B.

Giải thích: `provisioner` xác định driver/provisioner sẽ xử lý yêu cầu cấp phát volume.

## 35. Gợi ý cách giảng trên lớp

Phần này nên giảng theo nhịp sau:

### Giai đoạn 1: Tách cấu hình khỏi image

Bắt đầu bằng câu hỏi thực tế:

> Nếu cùng một image chạy ở dev, staging và production nhưng database host khác nhau, ta có nên build ba image khác nhau không?

Từ đó dẫn vào ConfigMap.

### Giai đoạn 2: Tách secret khỏi manifest và image

Sau khi  hiểu ConfigMap, hỏi tiếp:

> Nếu dữ liệu là password thì có đưa vào ConfigMap không?

Từ đó dẫn vào Secret, đồng thời nhấn mạnh Secret không phải phép màu bảo mật.

### Giai đoạn 3: Container filesystem không bền vững

Cho  chạy Pod ghi file vào container filesystem, xóa Pod và tạo lại để thấy dữ liệu mất. Sau đó giới thiệu Volume.

### Giai đoạn 4: emptyDir cho chia sẻ tạm

Dùng multi-container Pod với writer/reader để minh họa volume dùng chung trong Pod.

### Giai đoạn 5: PVC là cách ứng dụng yêu cầu storage

Giải thích Pod không nên biết backend storage. Pod chỉ cần PVC.

### Giai đoạn 6: StorageClass và CSI giúp tự động hóa

Triển khai NFS CSI để  nhìn thấy khi tạo PVC thì PV tự sinh ra.

### Giai đoạn 7: Lab tổng hợp

Chạy Nginx dùng cả ConfigMap, Secret và PVC. Xóa Pod để chứng minh workload có thể thay thế nhưng dữ liệu vẫn còn.

## 36. Tóm tắt buổi học

Trong buổi này,  đã học cách Kubernetes quản lý cấu hình, dữ liệu nhạy cảm và lưu trữ cho ứng dụng.

ConfigMap giúp tách cấu hình không nhạy cảm khỏi container image. Secret giúp tách dữ liệu nhạy cảm khỏi image và Pod manifest, nhưng cần hiểu rằng Secret không tự động đồng nghĩa với mã hóa hoàn chỉnh nếu cluster chưa được cấu hình bảo mật phù hợp.

Volume giúp container có thư mục dữ liệu được Kubernetes quản lý. `emptyDir` phù hợp cho dữ liệu tạm trong vòng đời Pod, còn dữ liệu bền vững cần dùng PVC.

PV là tài nguyên storage trong cluster, PVC là yêu cầu storage từ workload. Pod sử dụng PVC, PVC bind với PV, và PV đại diện cho backend storage thật.

StorageClass giúp admin mô tả các lớp storage khác nhau. Khi kết hợp với CSI driver, StorageClass cho phép dynamic provisioning, giúp PVC tự động được cấp phát PV.

Trong lab on-premise, NFS CSI Driver cho phép Kubernetes sử dụng một NFS Server có sẵn để cấp phát volume động. Đây là mô hình rất phù hợp để học storage trong Kubernetes trước khi đi sang các backend production phức tạp hơn như Ceph, Longhorn, SAN CSI hoặc cloud disk CSI.

## 37. Tài liệu tham khảo

- Kubernetes Documentation - ConfigMaps: https://v1-35.docs.kubernetes.io/docs/concepts/configuration/configmap/
- Kubernetes Documentation - Secrets: https://v1-35.docs.kubernetes.io/docs/concepts/configuration/secret/
- Kubernetes Documentation - Volumes: https://kubernetes.io/docs/concepts/storage/volumes/
- Kubernetes Documentation - Persistent Volumes: https://kubernetes.io/docs/concepts/storage/persistent-volumes/
- Kubernetes Documentation - Storage Classes: https://kubernetes.io/docs/concepts/storage/storage-classes/
- Kubernetes CSI Driver for NFS: https://github.com/kubernetes-csi/csi-driver-nfs
- NFS CSI Driver Parameters: https://github.com/kubernetes-csi/csi-driver-nfs/blob/master/docs/driver-parameters.md
