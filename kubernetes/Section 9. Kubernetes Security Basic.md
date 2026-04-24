# Buổi 09: Kubernetes Security Cơ Bản - ServiceAccount, RBAC, Secret Security, SecurityContext, Pod Security Standards và NetworkPolicy

## 1. Mục tiêu bài học

Sau buổi học này, cần hiểu được Kubernetes Security không phải là một tính năng đơn lẻ, mà là nhiều lớp kiểm soát kết hợp với nhau.

Ở mức cơ bản, cần nắm được:

- Cách Kubernetes kiểm soát truy cập vào Kubernetes API.
- Sự khác nhau giữa User và ServiceAccount.
- Cách dùng RBAC để giới hạn quyền theo namespace.
- Cách kiểm tra quyền bằng `kubectl auth can-i`.
- Cách sử dụng Secret an toàn hơn trong ứng dụng.
- Vì sao Secret không đồng nghĩa với mã hóa tuyệt đối.
- Cách hardening Pod bằng `securityContext`.
- Cách dùng Pod Security Admission thông qua label trên namespace.
- Cách dùng NetworkPolicy để giới hạn traffic giữa các Pod.
- Tư duy least privilege khi vận hành Kubernetes.

Buổi này không biến thành chuyên gia security ngay lập tức, nhưng giúp có nền tảng đúng để không triển khai Kubernetes theo kiểu “mọi thứ đều full quyền”.

## 2. Phạm vi kiến thức trong buổi 09

### 2.1. Kiến thức đã được mở khóa từ các buổi trước

Buổi này được xây dựng dựa trên các kiến thức đã học:

- Cluster, Node, Control Plane, Worker Node.
- Pod, Deployment, ReplicaSet, DaemonSet, StatefulSet, Job, CronJob.
- Namespace.
- ConfigMap và Secret.
- Service, ClusterIP, NodePort, Headless Service, ExternalName.
- Calico CNI.
- Ingress NGINX Controller.
- Resource requests/limits.
- Probes, rolling update, rollback, self-healing.
- Basic scheduling, nodeSelector, taint/toleration, affinity cơ bản.

### 2.2. Kiến thức mới được mở khóa trong buổi này

Trong buổi 09, chúng ta mở khóa thêm các khái niệm:

- Authentication.
- Authorization.
- Admission Control.
- ServiceAccount.
- RBAC.
- Role.
- ClusterRole.
- RoleBinding.
- ClusterRoleBinding.
- Secret security.
- ServiceAccount token.
- `automountServiceAccountToken`.
- Pod Security Standards.
- Pod Security Admission.
- SecurityContext.
- NetworkPolicy.

### 2.3. Những nội dung chưa đi sâu trong buổi này

Các nội dung sau chưa đi sâu trong buổi này để tránh vượt quá tiến trình giáo trình:

- OIDC / LDAP / SSO tích hợp Kubernetes API.
- Certificate-based user management chi tiết.
- Audit logging nâng cao.
- Admission Webhook custom.
- Policy engine như Kyverno hoặc OPA Gatekeeper.
- Image signing, SBOM, vulnerability scanning.
- Runtime security bằng Falco, Tetragon, eBPF.
- Service Mesh security.
- Zero Trust architecture đầy đủ.

Các chủ đề này nên để dành cho các buổi nâng cao hoặc module Kubernetes Security chuyên sâu.

## 3. Mô hình lab sử dụng trong buổi học

Cụm Kubernetes vẫn dùng mô hình đã triển khai từ Buổi 02.

```text
+----------------------+        +----------------------+
| master-01            |        | worker-01            |
| Ubuntu 24.04         |        | Ubuntu 24.04         |
| Control Plane        |        | Worker Node          |
| kube-apiserver       |        | kubelet              |
| etcd                 |        | containerd           |
| kube-scheduler       |        | Calico CNI           |
| controller-manager   |        +----------------------+
+----------------------+
          |
          | Kubernetes Cluster Network
          |
+----------------------+        +----------------------+
| worker-02            |        | nfs-01               |
| Ubuntu 24.04         |        | Ubuntu 24.04         |
| Worker Node          |        | NFS Server           |
| kubelet              |        | Đã dùng từ Buổi 05   |
| containerd           |        +----------------------+
| Calico CNI           |
+----------------------+
```

Trong buổi này, NFS Server không phải trọng tâm. Nếu cụm lab đã có NFS CSI từ Buổi 05 thì vẫn giữ nguyên, nhưng bài security này không phụ thuộc vào NFS.

## 4. Bức tranh tổng quan về Kubernetes Security

Khi một hành động được gửi vào Kubernetes API, ví dụ:

```bash
kubectl get pods -n default
kubectl create deployment web --image=nginx
kubectl delete secret db-password -n production
```

request đó sẽ đi qua nhiều lớp kiểm soát.

```text
User / ServiceAccount
        |
        v
+-----------------------+
| Authentication        |
| Bạn là ai?            |
+-----------------------+
        |
        v
+-----------------------+
| Authorization         |
| Bạn được làm gì?      |
+-----------------------+
        |
        v
+-----------------------+
| Admission Control     |
| Object có được nhận?  |
+-----------------------+
        |
        v
+-----------------------+
| etcd                  |
| Lưu trạng thái object |
+-----------------------+
```

Trong đó:

- Authentication xác định danh tính của chủ thể gửi request.
- Authorization kiểm tra chủ thể đó có quyền thực hiện hành động hay không.
- Admission Control kiểm tra hoặc thay đổi object trước khi lưu vào etcd.
- etcd lưu trạng thái mong muốn và trạng thái hiện tại của cluster.

Ví dụ dễ hiểu:

```text
Một Pod muốn list danh sách Pod trong namespace app.

1. Pod sử dụng ServiceAccount token để gọi kube-apiserver.
2. kube-apiserver xác định Pod đó đang đại diện cho ServiceAccount nào.
3. RBAC kiểm tra ServiceAccount đó có quyền list pods trong namespace app không.
4. Nếu có quyền, kube-apiserver trả kết quả.
5. Nếu không có quyền, kube-apiserver trả lỗi Forbidden.
```

## 5. Authentication trong Kubernetes

Authentication trả lời câu hỏi:

```text
Ai đang gửi request vào Kubernetes API?
```

Trong Kubernetes, chủ thể có thể là:

- Human user.
- ServiceAccount.
- Component nội bộ như kubelet, controller-manager, scheduler.
- External system gọi Kubernetes API.

### 5.1. User trong Kubernetes

Kubernetes có khái niệm User, nhưng không có object `User` giống như có object `Pod`, `Service` hay `Deployment`.

Nói cách khác, ta không tạo User bằng lệnh kiểu như:

```bash
kubectl create user alice
```

Lệnh trên không tồn tại.

User thường đến từ cơ chế xác thực bên ngoài, ví dụ:

- Client certificate.
- OIDC.
- Token.
- Authentication proxy.
- Cloud IAM nếu chạy trên cloud managed Kubernetes.

Trong lab kubeadm on-premise, khi chúng ta dùng file:

```bash
~/.kube/config
```

thì `kubectl` sử dụng thông tin trong kubeconfig để xác thực với kube-apiserver.

Kiểm tra kubeconfig hiện tại:

```bash
kubectl config view --minify
```

Xem user hiện tại trong context:

```bash
kubectl config current-context
```

```bash
kubectl config view --minify -o jsonpath='{.contexts[0].context.user}'
echo
```

Thông thường, với cluster kubeadm mới cài, người quản trị dùng context có quyền cluster-admin.

Điều này rất mạnh, nhưng cũng rất nguy hiểm nếu dùng bừa trong môi trường production.

### 5.2. ServiceAccount trong Kubernetes

ServiceAccount là identity dành cho workload chạy bên trong cluster.

Ví dụ:

- Một Pod cần gọi Kubernetes API để đọc ConfigMap.
- Một controller cần watch Deployment.
- Một application cần liệt kê Pod trong namespace của nó.
- Một CI/CD agent chạy trong cluster cần tạo Job.

Khi đó, không nên dùng kubeconfig admin để nhét vào Pod. Thay vào đó, ta tạo ServiceAccount với quyền tối thiểu.

Kiểm tra ServiceAccount mặc định trong namespace `default`:

```bash
kubectl get serviceaccount -n default
```

Output tham khảo:

```text
NAME      SECRETS   AGE
default   0         10d
```

Từ các phiên bản Kubernetes mới, ServiceAccount token không còn luôn luôn được tạo thành Secret dài hạn như trước. Khi cần token, Kubernetes sử dụng cơ chế token ngắn hạn thông qua TokenRequest API.

Tạo token tạm thời cho ServiceAccount:

```bash
kubectl create token default -n default --duration=10m
```

Lưu ý:

- Token này đại diện cho ServiceAccount.
- Không nên lưu token lâu dài trong file text.
- Không nên commit token vào Git.
- Không nên dùng token của namespace này cho namespace khác nếu không cần thiết.

## 6. Authorization và RBAC

Authorization trả lời câu hỏi:

```text
Chủ thể này được phép làm hành động gì trên tài nguyên nào?
```

Kubernetes hỗ trợ nhiều authorization mode, nhưng phổ biến nhất trong thực tế là RBAC.

RBAC là viết tắt của:

```text
Role-Based Access Control
```

Nghĩa là kiểm soát quyền dựa trên vai trò.

### 6.1. Các object chính trong RBAC

RBAC có 4 object quan trọng:

```text
Role
ClusterRole
RoleBinding
ClusterRoleBinding
```

### 6.2. Role

Role định nghĩa quyền trong phạm vi một namespace.

Ví dụ Role cho phép đọc Pod trong namespace `secure-demo`:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-reader
  namespace: secure-demo
rules:
  - apiGroups: [""]
    resources: ["pods"]
    verbs: ["get", "list", "watch"]
```

Giải thích các dòng quan trọng:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
```

Sử dụng API group của RBAC.

```yaml
kind: Role
```

Khai báo đây là một Role. Role chỉ có hiệu lực trong namespace được chỉ định.

```yaml
metadata:
  name: pod-reader
  namespace: secure-demo
```

Role này tên là `pod-reader` và chỉ thuộc namespace `secure-demo`.

```yaml
apiGroups: [""]
```

Chuỗi rỗng đại diện cho core API group. Các resource như Pod, Service, ConfigMap, Secret nằm trong core API group.

```yaml
resources: ["pods"]
```

Role này áp dụng lên resource Pod.

```yaml
verbs: ["get", "list", "watch"]
```

Chỉ cho phép đọc, liệt kê và theo dõi thay đổi Pod. Không cho phép tạo, sửa hoặc xóa Pod.

### 6.3. ClusterRole

ClusterRole định nghĩa quyền ở phạm vi toàn cluster hoặc dùng cho resource không nằm trong namespace.

Ví dụ:

- Node là cluster-scoped resource.
- PersistentVolume là cluster-scoped resource.
- Namespace là cluster-scoped resource.
- StorageClass là cluster-scoped resource.

Ví dụ ClusterRole cho phép đọc Node:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: node-reader
rules:
  - apiGroups: [""]
    resources: ["nodes"]
    verbs: ["get", "list", "watch"]
```

Điểm khác biệt quan trọng:

```yaml
metadata:
  name: node-reader
```

ClusterRole không khai báo namespace, vì nó là cluster-scoped object.

### 6.4. RoleBinding

Role chỉ định nghĩa quyền. RoleBinding mới là object gán quyền đó cho subject.

Subject có thể là:

- User.
- Group.
- ServiceAccount.

Ví dụ gán Role `pod-reader` cho ServiceAccount `app-reader`:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: app-reader-pod-reader
  namespace: secure-demo
subjects:
  - kind: ServiceAccount
    name: app-reader
    namespace: secure-demo
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

Giải thích:

```yaml
subjects:
  - kind: ServiceAccount
    name: app-reader
    namespace: secure-demo
```

Quyền sẽ được gán cho ServiceAccount `app-reader` trong namespace `secure-demo`.

```yaml
roleRef:
  kind: Role
  name: pod-reader
```

RoleBinding tham chiếu đến Role `pod-reader`.

### 6.5. ClusterRoleBinding

ClusterRoleBinding gán ClusterRole cho subject ở phạm vi toàn cluster.

Đây là object rất mạnh, cần dùng cẩn thận.

Ví dụ nguy hiểm:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: dangerous-cluster-admin
subjects:
  - kind: ServiceAccount
    name: app
    namespace: default
roleRef:
  kind: ClusterRole
  name: cluster-admin
  apiGroup: rbac.authorization.k8s.io
```

Manifest trên gán quyền `cluster-admin` cho ServiceAccount `app` trong namespace `default`.

Trong production, đây là kiểu cấu hình cần tránh trừ khi có lý do cực kỳ rõ ràng.

## 7. Lab 01: Tạo namespace, ServiceAccount và RBAC tối thiểu

### 7.1. Tạo thư mục lab

Thực hiện trên node có kubeconfig admin, thường là `master-01`.

```bash
mkdir -p ~/k8s-buoi-09-security
cd ~/k8s-buoi-09-security
```

### 7.2. Tạo namespace

Tạo file `00-namespace.yaml`:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: secure-demo
  labels:
    course: kubernetes
    session: security-basic
```

Apply:

```bash
kubectl apply -f 00-namespace.yaml
```

Kiểm tra:

```bash
kubectl get ns secure-demo --show-labels
```

### 7.3. Tạo ServiceAccount

Tạo file `01-serviceaccount.yaml`:

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: app-reader
  namespace: secure-demo
automountServiceAccountToken: false
```

Apply:

```bash
kubectl apply -f 01-serviceaccount.yaml
```

Kiểm tra:

```bash
kubectl get sa -n secure-demo
```

Giải thích manifest:

```yaml
kind: ServiceAccount
```

Tạo identity cho workload hoặc process chạy trong cluster.

```yaml
metadata:
  name: app-reader
  namespace: secure-demo
```

ServiceAccount này chỉ nằm trong namespace `secure-demo`.

```yaml
automountServiceAccountToken: false
```

Không tự động mount token vào Pod nếu Pod sử dụng ServiceAccount này.

Đây là best practice quan trọng. Không phải Pod nào cũng cần gọi Kubernetes API. Nếu Pod không cần gọi API, không nên tự động mount token vào container.

### 7.4. Tạo Role chỉ cho phép đọc Pod, Service, ConfigMap

Tạo file `02-role-readonly.yaml`:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: app-readonly
  namespace: secure-demo
rules:
  - apiGroups: [""]
    resources:
      - pods
      - services
      - configmaps
    verbs:
      - get
      - list
      - watch
```

Apply:

```bash
kubectl apply -f 02-role-readonly.yaml
```

Giải thích các dòng quan trọng:

```yaml
resources:
  - pods
  - services
  - configmaps
```

Role này chỉ tác động lên Pod, Service và ConfigMap.

```yaml
verbs:
  - get
  - list
  - watch
```

Role này chỉ cho phép đọc. Không có quyền `create`, `update`, `patch`, `delete`.

Lưu ý quan trọng:

```yaml
resources:
  - secrets
```

Không xuất hiện trong Role này. Nghĩa là ServiceAccount này không được quyền đọc Secret.

### 7.5. Gán Role cho ServiceAccount bằng RoleBinding

Tạo file `03-rolebinding-readonly.yaml`:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: app-reader-readonly-binding
  namespace: secure-demo
subjects:
  - kind: ServiceAccount
    name: app-reader
    namespace: secure-demo
roleRef:
  kind: Role
  name: app-readonly
  apiGroup: rbac.authorization.k8s.io
```

Apply:

```bash
kubectl apply -f 03-rolebinding-readonly.yaml
```

Kiểm tra object RBAC:

```bash
kubectl get role,rolebinding -n secure-demo
```

### 7.6. Kiểm tra quyền bằng kubectl auth can-i

Kiểm tra ServiceAccount `app-reader` có được list Pod không:

```bash
kubectl auth can-i list pods \
  --as=system:serviceaccount:secure-demo:app-reader \
  -n secure-demo
```

Kết quả mong đợi:

```text
yes
```

Kiểm tra có được đọc Secret không:

```bash
kubectl auth can-i get secrets \
  --as=system:serviceaccount:secure-demo:app-reader \
  -n secure-demo
```

Kết quả mong đợi:

```text
no
```

Kiểm tra có được xóa Pod không:

```bash
kubectl auth can-i delete pods \
  --as=system:serviceaccount:secure-demo:app-reader \
  -n secure-demo
```

Kết quả mong đợi:

```text
no
```

Đây là tinh thần least privilege.

ServiceAccount chỉ có quyền vừa đủ để làm nhiệm vụ của nó.

## 8. Secret Security

Secret đã được giới thiệu ở Buổi 05. Trong Buổi 09, ta nhìn Secret dưới góc độ security.

### 8.1. Secret không phải là mã hóa tuyệt đối

Một hiểu nhầm rất phổ biến:

```text
Dữ liệu trong Secret được base64 nên đã an toàn.
```

Điều này sai.

Base64 chỉ là encoding, không phải encryption.

Ví dụ:

```bash
echo -n 'P@ssw0rd123' | base64
```

Output:

```text
UEBzc3cwcmQxMjM=
```

Decode lại:

```bash
echo 'UEBzc3cwcmQxMjM=' | base64 -d
```

Output:

```text
P@ssw0rd123
```

Vì vậy, Secret chỉ giúp tách dữ liệu nhạy cảm khỏi manifest ứng dụng thông thường. Nó không tự động làm dữ liệu trở nên tuyệt đối an toàn.

### 8.2. Các rủi ro chính của Secret

Secret có thể bị lộ qua nhiều đường:

- Người dùng có quyền `get secrets` trong namespace.
- Người dùng có quyền tạo Pod trong namespace có thể mount Secret vào Pod.
- Secret được commit nhầm vào Git.
- Secret xuất hiện trong log ứng dụng.
- Secret được inject qua environment variable và bị dump ra process environment.
- etcd không bật encryption at rest.
- Backup etcd không được bảo vệ.

### 8.3. Best practices cơ bản với Secret

Trong môi trường production, nên áp dụng:

- Không commit Secret thật vào Git.
- Không cấp quyền đọc Secret rộng rãi.
- Không dùng `cluster-admin` cho application.
- Ưu tiên mount Secret dưới dạng file nếu phù hợp.
- Hạn chế truyền Secret qua environment variable nếu ứng dụng có thể đọc file.
- Bật encryption at rest cho Secret ở mức kube-apiserver.
- Bảo vệ backup etcd.
- Xoay vòng Secret định kỳ.
- Cân nhắc external secret manager ở môi trường production.

## 9. Lab 02: Tạo Secret và kiểm tra RBAC không được đọc Secret

### 9.1. Tạo Secret

Tạo file `04-secret.yaml`:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: database-credential
  namespace: secure-demo
type: Opaque
stringData:
  username: appuser
  password: P@ssw0rd123
```

Apply:

```bash
kubectl apply -f 04-secret.yaml
```

Kiểm tra Secret:

```bash
kubectl get secret -n secure-demo
```

Xem Secret bằng quyền admin:

```bash
kubectl get secret database-credential -n secure-demo -o yaml
```

Giải thích:

```yaml
stringData:
  username: appuser
  password: P@ssw0rd123
```

`stringData` cho phép nhập plain text trong manifest. Khi lưu vào Kubernetes, dữ liệu sẽ được đưa vào trường `data` ở dạng base64.

Lưu ý:

Trong lab dùng plain text để dễ học. Trong production, không nên lưu file này với password thật trong Git repository.

### 9.2. Kiểm tra ServiceAccount app-reader không được đọc Secret

```bash
kubectl auth can-i get secret database-credential \
  --as=system:serviceaccount:secure-demo:app-reader \
  -n secure-demo
```

Kết quả mong đợi:

```text
no
```

Kiểm tra có được list secret không:

```bash
kubectl auth can-i list secrets \
  --as=system:serviceaccount:secure-demo:app-reader \
  -n secure-demo
```

Kết quả mong đợi:

```text
no
```

Đây là ví dụ rõ nhất cho RBAC least privilege.

## 10. ServiceAccount token trong Pod

### 10.1. Vì sao cần quan tâm ServiceAccount token?

Mặc định, Pod có thể dùng ServiceAccount để gọi Kubernetes API.

Nếu một container bị compromise và trong container đó có ServiceAccount token, attacker có thể dùng token đó để gọi kube-apiserver.

Vì vậy, câu hỏi quan trọng là:

```text
Pod này có thật sự cần gọi Kubernetes API không?
```

Nếu không cần, hãy tắt tự động mount token.

Có thể tắt ở cấp ServiceAccount:

```yaml
automountServiceAccountToken: false
```

Hoặc tắt ở cấp Pod:

```yaml
spec:
  automountServiceAccountToken: false
```

Nếu Pod có cấu hình ở cả hai nơi, cấu hình trong Pod sẽ được ưu tiên hơn.

### 10.2. Manifest Pod không mount ServiceAccount token

Tạo file `05-pod-no-token.yaml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-no-token
  namespace: secure-demo
  labels:
    app: pod-no-token
spec:
  serviceAccountName: app-reader
  automountServiceAccountToken: false
  containers:
    - name: app
      image: busybox:1.36
      command:
        - sh
        - -c
        - sleep 3600
```

Apply:

```bash
kubectl apply -f 05-pod-no-token.yaml
```

Kiểm tra Pod:

```bash
kubectl get pod pod-no-token -n secure-demo
```

Exec vào Pod:

```bash
kubectl exec -it pod-no-token -n secure-demo -- sh
```

Trong container, kiểm tra thư mục token mặc định:

```sh
ls -l /var/run/secrets/kubernetes.io/serviceaccount
```

Kết quả mong đợi:

```text
ls: /var/run/secrets/kubernetes.io/serviceaccount: No such file or directory
```

Thoát container:

```sh
exit
```

Giải thích:

```yaml
serviceAccountName: app-reader
```

Pod sử dụng ServiceAccount `app-reader`.

```yaml
automountServiceAccountToken: false
```

Token của ServiceAccount không được tự động mount vào container.

Đây là cấu hình nên dùng cho các workload không cần gọi Kubernetes API.

## 11. SecurityContext

SecurityContext cho phép cấu hình các thiết lập bảo mật ở cấp Pod hoặc cấp container.

Nó trả lời các câu hỏi như:

- Container chạy bằng user nào?
- Có được chạy bằng root không?
- Có được privilege escalation không?
- Root filesystem có read-only không?
- Linux capabilities nào được giữ lại hoặc loại bỏ?
- Seccomp profile nào được sử dụng?
- Volume được gán group ownership như thế nào?

### 11.1. SecurityContext cấp Pod và cấp container

Có 2 vị trí thường dùng:

```yaml
spec:
  securityContext: {}
```

Đây là securityContext cấp Pod.

```yaml
containers:
  - name: app
    securityContext: {}
```

Đây là securityContext cấp container.

Một số field chỉ có ở Pod-level, một số field có ở container-level, một số field có thể dùng ở cả hai.

### 11.2. Các field quan trọng

#### runAsNonRoot

```yaml
runAsNonRoot: true
```

Yêu cầu container không được chạy bằng root.

Nếu image bắt buộc chạy bằng root, Pod có thể fail.

#### runAsUser

```yaml
runAsUser: 1000
```

Chạy process chính trong container bằng UID 1000.

#### runAsGroup

```yaml
runAsGroup: 1000
```

Chạy process chính với GID 1000.

#### fsGroup

```yaml
fsGroup: 1000
```

Khi mount volume, Kubernetes có thể áp dụng group ownership để process trong container có quyền truy cập volume.

#### allowPrivilegeEscalation

```yaml
allowPrivilegeEscalation: false
```

Không cho process trong container tăng quyền thông qua cơ chế như setuid binary.

#### readOnlyRootFilesystem

```yaml
readOnlyRootFilesystem: true
```

Root filesystem của container ở chế độ read-only.

Nếu ứng dụng cần ghi file, cần mount volume riêng vào đường dẫn được phép ghi như `/tmp`, `/cache`, `/var/run`.

#### capabilities

```yaml
capabilities:
  drop:
    - ALL
```

Loại bỏ toàn bộ Linux capabilities mặc định.

Đây là hardening rất quan trọng.

#### seccompProfile

```yaml
seccompProfile:
  type: RuntimeDefault
```

Dùng seccomp profile mặc định của container runtime.

Seccomp giúp giới hạn system calls mà process có thể gọi.

## 12. Lab 03: Deployment chạy non-root, read-only root filesystem và drop capabilities

### 12.1. Tạo Deployment an toàn hơn

Tạo file `06-secure-web-deployment.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: secure-web
  namespace: secure-demo
  labels:
    app: secure-web
spec:
  replicas: 2
  selector:
    matchLabels:
      app: secure-web
  template:
    metadata:
      labels:
        app: secure-web
    spec:
      serviceAccountName: app-reader
      automountServiceAccountToken: false
      securityContext:
        runAsNonRoot: true
        runAsUser: 1000
        runAsGroup: 1000
        fsGroup: 1000
        seccompProfile:
          type: RuntimeDefault
      containers:
        - name: web
          image: python:3.12-alpine
          imagePullPolicy: IfNotPresent
          command:
            - sh
            - -c
          args:
            - |
              echo "Hello from secure Kubernetes workload" > /tmp/index.html
              cd /tmp
              python -m http.server 8080
          ports:
            - name: http
              containerPort: 8080
          resources:
            requests:
              cpu: 50m
              memory: 64Mi
            limits:
              cpu: 200m
              memory: 128Mi
          securityContext:
            allowPrivilegeEscalation: false
            readOnlyRootFilesystem: true
            capabilities:
              drop:
                - ALL
          volumeMounts:
            - name: tmp
              mountPath: /tmp
      volumes:
        - name: tmp
          emptyDir: {}
---
apiVersion: v1
kind: Service
metadata:
  name: secure-web
  namespace: secure-demo
spec:
  type: ClusterIP
  selector:
    app: secure-web
  ports:
    - name: http
      port: 80
      targetPort: 8080
```

Apply:

```bash
kubectl apply -f 06-secure-web-deployment.yaml
```

Kiểm tra:

```bash
kubectl get deploy,pod,svc -n secure-demo -l app=secure-web
```

Test nội bộ bằng Pod curl tạm thời:

```bash
kubectl run curl-test -n secure-demo --rm -it \
  --image=curlimages/curl:8.10.1 \
  --restart=Never -- \
  curl -s http://secure-web.secure-demo.svc.cluster.local
```

Kết quả mong đợi:

```text
Hello from secure Kubernetes workload
```

### 12.2. Giải thích manifest

```yaml
serviceAccountName: app-reader
```

Pod dùng ServiceAccount `app-reader`.

```yaml
automountServiceAccountToken: false
```

Dù Pod có ServiceAccount, token không được mount vào container.

```yaml
runAsNonRoot: true
runAsUser: 1000
runAsGroup: 1000
```

Container không chạy bằng root mà chạy bằng UID/GID 1000.

```yaml
fsGroup: 1000
```

Giúp volume được mount có group phù hợp để process UID/GID 1000 có thể ghi.

```yaml
seccompProfile:
  type: RuntimeDefault
```

Dùng seccomp profile mặc định của runtime.

```yaml
allowPrivilegeEscalation: false
```

Không cho phép process tăng quyền.

```yaml
readOnlyRootFilesystem: true
```

Root filesystem của container chỉ đọc.

```yaml
capabilities:
  drop:
    - ALL
```

Loại bỏ toàn bộ Linux capabilities.

```yaml
volumeMounts:
  - name: tmp
    mountPath: /tmp
```

Vì root filesystem read-only, ứng dụng cần chỗ ghi file tạm. Ở đây ta mount `emptyDir` vào `/tmp`.

```yaml
volumes:
  - name: tmp
    emptyDir: {}
```

Tạo volume tạm trong vòng đời Pod.

## 13. Pod Security Standards

Pod Security Standards định nghĩa 3 mức policy cho Pod.

```text
Privileged  -> ít hạn chế nhất
Baseline    -> chặn các cấu hình nguy hiểm phổ biến
Restricted  -> hạn chế mạnh hơn, hướng đến hardening tốt hơn
```

### 13.1. Privileged

Mức `privileged` gần như không hạn chế.

Phù hợp với một số workload hệ thống đặc biệt như CNI, CSI, node agent.

Không nên dùng cho ứng dụng thông thường.

### 13.2. Baseline

Mức `baseline` chặn các cấu hình nguy hiểm phổ biến.

Ví dụ:

- Không cho Pod privileged.
- Hạn chế host namespaces.
- Hạn chế hostPath ở mức nhất định.
- Hạn chế capabilities nguy hiểm.

Phù hợp với nhiều workload ứng dụng thông thường cần mức bảo vệ cơ bản.

### 13.3. Restricted

Mức `restricted` mạnh hơn.

Thường yêu cầu:

- Không chạy privileged.
- Không privilege escalation.
- Drop capabilities.
- Chạy non-root.
- Dùng seccomp.
- Không dùng host namespaces.

Đây là mức nên hướng đến cho application production nếu image và ứng dụng hỗ trợ.

## 14. Pod Security Admission

Pod Security Admission là admission controller built-in của Kubernetes để enforce Pod Security Standards ở cấp namespace.

Nó hoạt động bằng label trên namespace.

Có 3 mode chính:

```text
enforce  -> chặn Pod vi phạm policy
warn     -> cho chạy nhưng cảnh báo người dùng
audit    -> ghi nhận audit annotation khi vi phạm
```

Ví dụ label namespace:

```bash
kubectl label namespace secure-demo \
  pod-security.kubernetes.io/enforce=restricted \
  pod-security.kubernetes.io/enforce-version=v1.35 \
  pod-security.kubernetes.io/warn=restricted \
  pod-security.kubernetes.io/warn-version=v1.35 \
  pod-security.kubernetes.io/audit=restricted \
  pod-security.kubernetes.io/audit-version=v1.35 \
  --overwrite
```

Giải thích:

```text
pod-security.kubernetes.io/enforce=restricted
```

Chặn các Pod không đạt mức Restricted.

```text
pod-security.kubernetes.io/warn=restricted
```

Hiển thị warning nếu manifest vi phạm Restricted.

```text
pod-security.kubernetes.io/audit=restricted
```

Ghi nhận audit metadata nếu request vi phạm policy.

```text
pod-security.kubernetes.io/enforce-version=v1.35
```

Khóa version policy theo Kubernetes v1.35 để behavior ổn định theo version bài lab.

## 15. Lab 04: Enforce Restricted Pod Security Standard

### 15.1. Label namespace secure-demo

Chạy lệnh:

```bash
kubectl label namespace secure-demo \
  pod-security.kubernetes.io/enforce=restricted \
  pod-security.kubernetes.io/enforce-version=v1.35 \
  pod-security.kubernetes.io/warn=restricted \
  pod-security.kubernetes.io/warn-version=v1.35 \
  pod-security.kubernetes.io/audit=restricted \
  pod-security.kubernetes.io/audit-version=v1.35 \
  --overwrite
```

Kiểm tra label:

```bash
kubectl get ns secure-demo --show-labels
```

### 15.2. Tạo Pod vi phạm Restricted policy

Tạo file `07-bad-privileged-pod.yaml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: bad-privileged-pod
  namespace: secure-demo
spec:
  containers:
    - name: bad
      image: busybox:1.36
      command:
        - sh
        - -c
        - sleep 3600
      securityContext:
        privileged: true
```

Apply:

```bash
kubectl apply -f 07-bad-privileged-pod.yaml
```

Kết quả mong đợi:

```text
Error from server (Forbidden): error when creating "07-bad-privileged-pod.yaml": pods "bad-privileged-pod" is forbidden: violates PodSecurity "restricted:..."
```

Ý nghĩa:

- Pod yêu cầu `privileged: true`.
- Namespace đang enforce `restricted`.
- Admission controller chặn Pod trước khi object được tạo.

### 15.3. Tạo Pod đạt Restricted policy

Tạo file `08-good-restricted-pod.yaml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: good-restricted-pod
  namespace: secure-demo
  labels:
    app: good-restricted-pod
spec:
  automountServiceAccountToken: false
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
    runAsGroup: 1000
    seccompProfile:
      type: RuntimeDefault
  containers:
    - name: app
      image: busybox:1.36
      command:
        - sh
        - -c
        - sleep 3600
      securityContext:
        allowPrivilegeEscalation: false
        capabilities:
          drop:
            - ALL
```

Apply:

```bash
kubectl apply -f 08-good-restricted-pod.yaml
```

Kiểm tra:

```bash
kubectl get pod good-restricted-pod -n secure-demo
```

Pod này nên được tạo thành công.

Giải thích các dòng quan trọng:

```yaml
automountServiceAccountToken: false
```

Không mount token API nếu Pod không cần.

```yaml
runAsNonRoot: true
runAsUser: 1000
runAsGroup: 1000
```

Không chạy bằng root.

```yaml
seccompProfile:
  type: RuntimeDefault
```

Sử dụng seccomp profile mặc định.

```yaml
allowPrivilegeEscalation: false
```

Không cho privilege escalation.

```yaml
capabilities:
  drop:
    - ALL
```

Drop toàn bộ capabilities.

## 16. NetworkPolicy

NetworkPolicy dùng để kiểm soát traffic vào hoặc ra khỏi Pod.

Trong cluster của chúng ta, Calico đã được cài từ Buổi 02. Calico hỗ trợ Kubernetes NetworkPolicy, nên ta có thể dùng NetworkPolicy để kiểm soát traffic giữa các Pod.

### 16.1. Mặc định Pod có thể nói chuyện với nhau

Nếu không có NetworkPolicy, mặc định các Pod thường có thể giao tiếp với nhau trong cluster network.

Ví dụ:

```text
frontend Pod  ---> backend Pod
attacker Pod  ---> backend Pod
random Pod    ---> backend Pod
```

Tất cả đều có thể truy cập nếu không có chính sách chặn.

### 16.2. NetworkPolicy hoạt động theo label selector

NetworkPolicy không chọn Pod bằng tên Pod trực tiếp. Nó chọn Pod bằng label.

Ví dụ:

```yaml
podSelector:
  matchLabels:
    app: backend
```

Policy này áp dụng cho các Pod có label:

```text
app=backend
```

### 16.3. Ingress và Egress trong NetworkPolicy

Trong NetworkPolicy:

```text
Ingress = traffic đi vào Pod được chọn
Egress  = traffic đi ra từ Pod được chọn
```

Ví dụ:

```text
frontend ---> backend
```

Đối với backend:

```text
Đây là ingress traffic
```

Đối với frontend:

```text
Đây là egress traffic
```

### 16.4. Default deny

NetworkPolicy phổ biến nhất là default deny.

Ví dụ default deny ingress:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-ingress
  namespace: secure-demo
spec:
  podSelector: {}
  policyTypes:
    - Ingress
```

Giải thích:

```yaml
podSelector: {}
```

Chọn tất cả Pod trong namespace.

```yaml
policyTypes:
  - Ingress
```

Áp dụng cho traffic đi vào Pod.

```yaml
ingress:
```

Không có rule ingress nào được khai báo, nên mặc định chặn toàn bộ ingress vào các Pod được chọn.

## 17. Lab 05: Giới hạn traffic bằng NetworkPolicy

Trong lab này, ta triển khai:

```text
frontend Pod  ---> được phép truy cập backend
attacker Pod  ---> bị chặn truy cập backend
```

### 17.1. Tạo backend Deployment và Service

Tạo file `09-backend.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend
  namespace: secure-demo
  labels:
    app: backend
spec:
  replicas: 2
  selector:
    matchLabels:
      app: backend
  template:
    metadata:
      labels:
        app: backend
    spec:
      automountServiceAccountToken: false
      securityContext:
        runAsNonRoot: true
        runAsUser: 1000
        runAsGroup: 1000
        seccompProfile:
          type: RuntimeDefault
      containers:
        - name: backend
          image: python:3.12-alpine
          command:
            - sh
            - -c
          args:
            - |
              echo "Backend service is reachable" > /tmp/index.html
              cd /tmp
              python -m http.server 8080
          ports:
            - name: http
              containerPort: 8080
          securityContext:
            allowPrivilegeEscalation: false
            readOnlyRootFilesystem: true
            capabilities:
              drop:
                - ALL
          volumeMounts:
            - name: tmp
              mountPath: /tmp
      volumes:
        - name: tmp
          emptyDir: {}
---
apiVersion: v1
kind: Service
metadata:
  name: backend
  namespace: secure-demo
spec:
  type: ClusterIP
  selector:
    app: backend
  ports:
    - name: http
      port: 80
      targetPort: 8080
```

Apply:

```bash
kubectl apply -f 09-backend.yaml
```

Kiểm tra:

```bash
kubectl get deploy,pod,svc -n secure-demo -l app=backend
```

### 17.2. Tạo frontend Pod và attacker Pod

Tạo file `10-client-pods.yaml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: frontend
  namespace: secure-demo
  labels:
    app: frontend
spec:
  automountServiceAccountToken: false
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
    runAsGroup: 1000
    seccompProfile:
      type: RuntimeDefault
  containers:
    - name: curl
      image: curlimages/curl:8.10.1
      command:
        - sh
        - -c
        - sleep 3600
      securityContext:
        allowPrivilegeEscalation: false
        capabilities:
          drop:
            - ALL
---
apiVersion: v1
kind: Pod
metadata:
  name: attacker
  namespace: secure-demo
  labels:
    app: attacker
spec:
  automountServiceAccountToken: false
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
    runAsGroup: 1000
    seccompProfile:
      type: RuntimeDefault
  containers:
    - name: curl
      image: curlimages/curl:8.10.1
      command:
        - sh
        - -c
        - sleep 3600
      securityContext:
        allowPrivilegeEscalation: false
        capabilities:
          drop:
            - ALL
```

Apply:

```bash
kubectl apply -f 10-client-pods.yaml
```

Kiểm tra:

```bash
kubectl get pod -n secure-demo -l app
```

### 17.3. Test trước khi có NetworkPolicy

Từ frontend truy cập backend:

```bash
kubectl exec -n secure-demo frontend -- \
  curl -s --connect-timeout 3 http://backend.secure-demo.svc.cluster.local
```

Kết quả mong đợi:

```text
Backend service is reachable
```

Từ attacker truy cập backend:

```bash
kubectl exec -n secure-demo attacker -- \
  curl -s --connect-timeout 3 http://backend.secure-demo.svc.cluster.local
```

Kết quả mong đợi:

```text
Backend service is reachable
```

Lúc này chưa có NetworkPolicy, nên cả frontend và attacker đều truy cập được backend.

### 17.4. Tạo default deny ingress cho backend

Tạo file `11-networkpolicy-backend-deny.yaml`:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: backend-default-deny-ingress
  namespace: secure-demo
spec:
  podSelector:
    matchLabels:
      app: backend
  policyTypes:
    - Ingress
```

Apply:

```bash
kubectl apply -f 11-networkpolicy-backend-deny.yaml
```

Test lại từ frontend:

```bash
kubectl exec -n secure-demo frontend -- \
  curl -s --connect-timeout 3 http://backend.secure-demo.svc.cluster.local
```

Kết quả mong đợi:

```text
command terminated with exit code 28
```

Hoặc timeout.

Test từ attacker:

```bash
kubectl exec -n secure-demo attacker -- \
  curl -s --connect-timeout 3 http://backend.secure-demo.svc.cluster.local
```

Kết quả cũng bị timeout.

Giải thích:

- Policy chọn Pod `app=backend`.
- Policy chỉ định `Ingress`.
- Không có rule allow nào.
- Vì vậy toàn bộ traffic vào backend bị chặn.

### 17.5. Cho phép chỉ frontend truy cập backend

Tạo file `12-networkpolicy-allow-frontend-to-backend.yaml`:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend-to-backend
  namespace: secure-demo
spec:
  podSelector:
    matchLabels:
      app: backend
  policyTypes:
    - Ingress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app: frontend
      ports:
        - protocol: TCP
          port: 8080
```

Apply:

```bash
kubectl apply -f 12-networkpolicy-allow-frontend-to-backend.yaml
```

Test từ frontend:

```bash
kubectl exec -n secure-demo frontend -- \
  curl -s --connect-timeout 3 http://backend.secure-demo.svc.cluster.local
```

Kết quả mong đợi:

```text
Backend service is reachable
```

Test từ attacker:

```bash
kubectl exec -n secure-demo attacker -- \
  curl -s --connect-timeout 3 http://backend.secure-demo.svc.cluster.local
```

Kết quả mong đợi:

```text
command terminated with exit code 28
```

Hoặc timeout.

### 17.6. Giải thích manifest allow

```yaml
podSelector:
  matchLabels:
    app: backend
```

Policy áp dụng lên Pod backend.

```yaml
policyTypes:
  - Ingress
```

Policy kiểm soát traffic đi vào backend.

```yaml
from:
  - podSelector:
      matchLabels:
        app: frontend
```

Chỉ cho phép source Pod có label `app=frontend`.

```yaml
ports:
  - protocol: TCP
    port: 8080
```

Chỉ cho phép traffic TCP vào port 8080 trên backend Pod.

Lưu ý quan trọng:

Trong NetworkPolicy, `port: 8080` là port trên Pod backend, không phải Service port 80.

Service `backend` nhận port 80 và forward đến `targetPort: 8080`. NetworkPolicy kiểm soát traffic đến Pod, nên cần allow port 8080.

## 18. ClusterRole và ClusterRoleBinding dùng khi nào?

Trong lab trên, ta dùng Role và RoleBinding vì chỉ thao tác trong namespace `secure-demo`.

ClusterRole và ClusterRoleBinding dùng khi:

- Cần quyền trên resource toàn cluster như Node, PV, Namespace, StorageClass.
- Cần quyền đọc resource cùng loại ở nhiều namespace.
- Cần tạo controller hoặc operator chạy ở cấp cluster.

Ví dụ chỉ đọc Node:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: node-readonly
rules:
  - apiGroups: [""]
    resources:
      - nodes
    verbs:
      - get
      - list
      - watch
```

Gán cho ServiceAccount:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: app-reader-node-readonly
subjects:
  - kind: ServiceAccount
    name: app-reader
    namespace: secure-demo
roleRef:
  kind: ClusterRole
  name: node-readonly
  apiGroup: rbac.authorization.k8s.io
```

Cảnh báo:

Không nên dùng ClusterRoleBinding nếu RoleBinding trong namespace đã đủ đáp ứng nhu cầu.

## 19. Các lỗi thường gặp trong Kubernetes Security

### 19.1. Lỗi Forbidden khi gọi Kubernetes API

Ví dụ:

```text
Error from server (Forbidden): pods is forbidden: User "system:serviceaccount:secure-demo:app-reader" cannot list resource "pods"
```

Nguyên nhân:

- ServiceAccount chưa được bind Role.
- Role thiếu verb cần thiết.
- Role thiếu resource cần thiết.
- RoleBinding trỏ sai ServiceAccount.
- Thao tác ở namespace khác.

Cách kiểm tra:

```bash
kubectl auth can-i list pods \
  --as=system:serviceaccount:secure-demo:app-reader \
  -n secure-demo
```

Kiểm tra RoleBinding:

```bash
kubectl describe rolebinding app-reader-readonly-binding -n secure-demo
```

### 19.2. Pod bị chặn bởi Pod Security Admission

Ví dụ:

```text
Error from server (Forbidden): violates PodSecurity "restricted"
```

Nguyên nhân thường gặp:

- Container chạy privileged.
- Không set `runAsNonRoot`.
- Không set `allowPrivilegeEscalation: false`.
- Không drop capabilities.
- Không set `seccompProfile`.
- Chạy image bắt buộc root.

Cách kiểm tra label namespace:

```bash
kubectl get ns secure-demo --show-labels
```

Cách xem manifest Pod:

```bash
kubectl get pod <pod-name> -n secure-demo -o yaml
```

### 19.3. Pod chạy non-root bị lỗi permission denied

Nguyên nhân:

- Ứng dụng cần ghi vào thư mục chỉ root mới có quyền.
- Root filesystem đang read-only.
- Volume mount chưa có ownership phù hợp.
- Thiếu `fsGroup`.

Cách xử lý:

- Chạy ứng dụng trên port lớn hơn 1024.
- Mount `emptyDir` vào thư mục cần ghi.
- Dùng `fsGroup` cho volume.
- Sử dụng image hỗ trợ non-root.
- Không ép read-only root filesystem nếu ứng dụng chưa sẵn sàng.

### 19.4. NetworkPolicy không có tác dụng

Nguyên nhân:

- CNI không hỗ trợ NetworkPolicy.
- Policy chọn sai label.
- Policy áp dụng sai namespace.
- Đang test nhầm port Service thay vì Pod port.
- Dùng DNS name sai.

Với cụm lab này, Calico hỗ trợ NetworkPolicy nên policy phải có tác dụng nếu selector đúng.

Kiểm tra Pod label:

```bash
kubectl get pod -n secure-demo --show-labels
```

Kiểm tra policy:

```bash
kubectl describe networkpolicy -n secure-demo
```

### 19.5. Secret bị lộ qua quyền tạo Pod

Một điểm rất quan trọng:

Nếu một user có quyền tạo Pod trong namespace, người đó có thể tạo Pod để mount Secret trong namespace đó, dù user không trực tiếp có quyền `get secrets`.

Vì vậy, quyền `create pods` cũng cần được kiểm soát cẩn thận.

Không nên nghĩ rằng:

```text
User không có quyền get secret nghĩa là không thể đọc secret.
```

Nếu user có quyền tạo Pod, họ có thể gián tiếp đọc Secret thông qua Pod.

## 20. Lab tổng hợp cuối buổi

### 20.1. Yêu cầu lab

Tạo một môi trường `secure-demo` đáp ứng các yêu cầu:

- Namespace có Pod Security Admission mức Restricted.
- Có ServiceAccount riêng cho ứng dụng.
- ServiceAccount không tự động mount token.
- Role chỉ cho phép đọc Pod, Service, ConfigMap.
- ServiceAccount không được đọc Secret.
- Ứng dụng chạy non-root.
- Root filesystem read-only.
- Drop toàn bộ capabilities.
- Dùng seccomp RuntimeDefault.
- Backend chỉ cho frontend truy cập qua NetworkPolicy.
- Attacker Pod bị chặn.

### 20.2. Thứ tự thực hiện

```bash
cd ~/k8s-buoi-09-security

kubectl apply -f 00-namespace.yaml
kubectl apply -f 01-serviceaccount.yaml
kubectl apply -f 02-role-readonly.yaml
kubectl apply -f 03-rolebinding-readonly.yaml
kubectl apply -f 04-secret.yaml
kubectl apply -f 06-secure-web-deployment.yaml

kubectl label namespace secure-demo \
  pod-security.kubernetes.io/enforce=restricted \
  pod-security.kubernetes.io/enforce-version=v1.35 \
  pod-security.kubernetes.io/warn=restricted \
  pod-security.kubernetes.io/warn-version=v1.35 \
  pod-security.kubernetes.io/audit=restricted \
  pod-security.kubernetes.io/audit-version=v1.35 \
  --overwrite

kubectl apply -f 09-backend.yaml
kubectl apply -f 10-client-pods.yaml
kubectl apply -f 11-networkpolicy-backend-deny.yaml
kubectl apply -f 12-networkpolicy-allow-frontend-to-backend.yaml
```

### 20.3. Kiểm tra RBAC

```bash
kubectl auth can-i list pods \
  --as=system:serviceaccount:secure-demo:app-reader \
  -n secure-demo
```

Mong đợi:

```text
yes
```

```bash
kubectl auth can-i get secrets \
  --as=system:serviceaccount:secure-demo:app-reader \
  -n secure-demo
```

Mong đợi:

```text
no
```

### 20.4. Kiểm tra Pod Security Admission

```bash
kubectl apply -f 07-bad-privileged-pod.yaml
```

Mong đợi:

```text
Forbidden
```

```bash
kubectl apply -f 08-good-restricted-pod.yaml
```

Mong đợi:

```text
pod/good-restricted-pod created
```

### 20.5. Kiểm tra ứng dụng secure-web

```bash
kubectl get pod -n secure-demo -l app=secure-web
```

Test service:

```bash
kubectl run curl-secure-web -n secure-demo --rm -it \
  --image=curlimages/curl:8.10.1 \
  --restart=Never -- \
  curl -s http://secure-web.secure-demo.svc.cluster.local
```

Mong đợi:

```text
Hello from secure Kubernetes workload
```

### 20.6. Kiểm tra NetworkPolicy

Từ frontend:

```bash
kubectl exec -n secure-demo frontend -- \
  curl -s --connect-timeout 3 http://backend.secure-demo.svc.cluster.local
```

Mong đợi:

```text
Backend service is reachable
```

Từ attacker:

```bash
kubectl exec -n secure-demo attacker -- \
  curl -s --connect-timeout 3 http://backend.secure-demo.svc.cluster.local
```

Mong đợi:

```text
command terminated with exit code 28
```

Hoặc timeout.

## 21. Optional: Encryption at rest cho Secret trong kubeadm cluster

Phần này chỉ giới thiệu ở mức định hướng. Không bắt buộc làm trong lab chính vì thao tác này thay đổi cấu hình kube-apiserver.

Trong Kubernetes, Secret có thể được mã hóa khi lưu trong etcd thông qua `EncryptionConfiguration`.

Ví dụ file cấu hình:

```yaml
apiVersion: apiserver.config.k8s.io/v1
kind: EncryptionConfiguration
resources:
  - resources:
      - secrets
    providers:
      - aescbc:
          keys:
            - name: key1
              secret: <BASE64_ENCODED_32_BYTE_KEY>
      - identity: {}
```

Ý nghĩa:

```yaml
resources:
  - secrets
```

Chỉ định mã hóa Secret.

```yaml
aescbc:
```

Sử dụng provider AES-CBC.

```yaml
identity: {}
```

Provider fallback. Nếu đặt `identity` đầu tiên thì dữ liệu mới sẽ không được mã hóa. Vì vậy provider mã hóa phải đứng trước `identity`.

Với cluster kubeadm, kube-apiserver chạy dạng static Pod tại:

```bash
/etc/kubernetes/manifests/kube-apiserver.yaml
```

Muốn bật encryption at rest cần:

- Backup etcd trước.
- Tạo encryption config trên control plane node.
- Mount file này vào kube-apiserver static Pod.
- Thêm flag `--encryption-provider-config`.
- Đảm bảo kube-apiserver restart thành công.
- Re-write các Secret cũ nếu muốn mã hóa lại dữ liệu đã tồn tại.

Không nên thao tác phần này trên production nếu chưa có kế hoạch backup/rollback rõ ràng.

## 22. Checklist security cơ bản cho application namespace

Khi triển khai application namespace trong Kubernetes, nên kiểm tra các điểm sau:

### Namespace

- Namespace riêng cho từng môi trường hoặc từng ứng dụng lớn.
- Có label Pod Security Admission phù hợp.
- Không để mọi workload chạy chung trong `default`.

### ServiceAccount

- Mỗi application quan trọng có ServiceAccount riêng.
- Không dùng ServiceAccount `default` cho workload production.
- Tắt `automountServiceAccountToken` nếu Pod không cần gọi Kubernetes API.

### RBAC

- Dùng RoleBinding trong namespace nếu đủ.
- Tránh ClusterRoleBinding nếu không cần.
- Không gán `cluster-admin` cho application.
- Không dùng wildcard `*` bừa bãi.
- Kiểm tra quyền bằng `kubectl auth can-i`.

### Secret

- Không commit Secret thật vào Git.
- Không cấp quyền đọc Secret rộng rãi.
- Cân nhắc encryption at rest.
- Cân nhắc external secret manager.
- Xoay vòng Secret định kỳ.

### Pod Security

- Chạy non-root nếu có thể.
- Không chạy privileged trừ workload hệ thống đặc biệt.
- Tắt privilege escalation.
- Drop capabilities không cần thiết.
- Dùng seccomp RuntimeDefault.
- Dùng read-only root filesystem nếu ứng dụng hỗ trợ.

### Network

- Áp dụng NetworkPolicy cho namespace quan trọng.
- Bắt đầu bằng default deny rồi mở dần theo nhu cầu.
- Dùng label rõ ràng cho frontend/backend/database.
- Kiểm tra cả ingress và egress nếu cần kiểm soát chặt.

## 23. Câu hỏi kiểm tra cuối buổi

### Câu 1

ServiceAccount dùng để làm gì trong Kubernetes?

Gợi ý trả lời:

ServiceAccount là identity dành cho process chạy trong Pod hoặc workload nội bộ cluster khi cần tương tác với Kubernetes API.

### Câu 2

Sự khác nhau giữa Role và ClusterRole là gì?

Gợi ý trả lời:

Role có phạm vi trong một namespace. ClusterRole có phạm vi toàn cluster hoặc dùng cho resource không thuộc namespace.

### Câu 3

RoleBinding và ClusterRoleBinding khác nhau thế nào?

Gợi ý trả lời:

RoleBinding gán Role hoặc ClusterRole trong phạm vi namespace. ClusterRoleBinding gán ClusterRole ở phạm vi toàn cluster.

### Câu 4

Vì sao không nên gán `cluster-admin` cho application ServiceAccount?

Gợi ý trả lời:

Vì application sẽ có toàn quyền trên cluster. Nếu application bị compromise, attacker có thể kiểm soát toàn bộ cluster.

### Câu 5

Base64 trong Secret có phải encryption không?

Gợi ý trả lời:

Không. Base64 chỉ là encoding, có thể decode dễ dàng. Muốn bảo vệ Secret tốt hơn cần RBAC, encryption at rest, quản lý Secret an toàn và external secret manager nếu cần.

### Câu 6

`automountServiceAccountToken: false` có tác dụng gì?

Gợi ý trả lời:

Ngăn Kubernetes tự động mount ServiceAccount token vào Pod, giảm rủi ro token bị lộ nếu container bị compromise.

### Câu 7

`allowPrivilegeEscalation: false` dùng để làm gì?

Gợi ý trả lời:

Ngăn process trong container tăng quyền thông qua các cơ chế như setuid binary.

### Câu 8

Pod Security Standards có những mức nào?

Gợi ý trả lời:

Privileged, Baseline và Restricted.

### Câu 9

NetworkPolicy chọn Pod bằng gì?

Gợi ý trả lời:

NetworkPolicy chọn Pod bằng label selector, không chọn trực tiếp bằng tên Pod.

### Câu 10

Nếu NetworkPolicy allow port 8080 nhưng Service port là 80, vì sao vẫn đúng?

Gợi ý trả lời:

Vì NetworkPolicy kiểm soát traffic đến Pod, nên port cần allow là container port hoặc targetPort trên Pod backend, không phải Service port.

## 24. Bài tập về nhà

### Bài 1

Tạo namespace `team-a` với Pod Security Admission ở mức `baseline`.

Yêu cầu:

- Tạo namespace.
- Label namespace theo PSA baseline.
- Tạo Pod bình thường chạy được.
- Tạo Pod privileged và quan sát kết quả.

### Bài 2

Tạo ServiceAccount `viewer` trong namespace `team-a`.

Yêu cầu:

- Chỉ được `get`, `list`, `watch` Pod.
- Không được đọc Secret.
- Không được xóa Pod.
- Dùng `kubectl auth can-i` để chứng minh.

### Bài 3

Tạo Deployment `safe-app`.

Yêu cầu:

- Chạy non-root.
- Tắt automount ServiceAccount token.
- Drop ALL capabilities.
- Không cho privilege escalation.
- Dùng seccomp RuntimeDefault.

### Bài 4

Tạo NetworkPolicy cho namespace `team-a`.

Yêu cầu:

- Backend chỉ nhận traffic từ frontend.
- Pod khác không truy cập được backend.
- Chứng minh bằng `curl` từ hai Pod khác nhau.

## 25. Dọn dẹp lab

Nếu muốn xóa toàn bộ tài nguyên buổi 09:

```bash
kubectl delete ns secure-demo
```

Kiểm tra:

```bash
kubectl get ns
```

Nếu đã tạo namespace khác cho bài tập về nhà:

```bash
kubectl delete ns team-a
```

## 26. Tóm tắt cuối buổi

Trong buổi này, đã học được các lớp bảo mật cơ bản trong Kubernetes.

RBAC giúp kiểm soát ai được làm gì với tài nguyên nào.

ServiceAccount là identity dành cho workload chạy trong cluster.

Secret cần được bảo vệ bằng RBAC, encryption at rest và quy trình quản lý an toàn; base64 không phải encryption.

SecurityContext giúp hardening Pod/container bằng cách chạy non-root, drop capabilities, tắt privilege escalation và dùng seccomp.

Pod Security Admission giúp enforce Pod Security Standards ở cấp namespace.

NetworkPolicy giúp kiểm soát traffic giữa các Pod, đặc biệt quan trọng khi cần phân tách frontend/backend/database.

Tư duy quan trọng nhất của buổi này là:

```text
Least privilege by default.
```

Không cấp quyền vì tiện.

Không chạy root vì quen.

Không mở network vì dễ.

Không dùng cluster-admin cho application.

Không xem Secret là an toàn chỉ vì nó nằm trong Kubernetes.

## 27. Tài liệu tham khảo chính thức

- Kubernetes Documentation v1.35 - Using RBAC Authorization: https://v1-35.docs.kubernetes.io/docs/reference/access-authn-authz/rbac/
- Kubernetes Documentation v1.35 - Service Accounts: https://v1-35.docs.kubernetes.io/docs/concepts/security/service-accounts/
- Kubernetes Documentation v1.35 - Secrets: https://v1-35.docs.kubernetes.io/docs/concepts/configuration/secret/
- Kubernetes Documentation v1.35 - Configure a Security Context for a Pod or Container: https://v1-35.docs.kubernetes.io/docs/tasks/configure-pod-container/security-context/
- Kubernetes Documentation v1.35 - Pod Security Standards: https://v1-35.docs.kubernetes.io/docs/concepts/security/pod-security-standards/
- Kubernetes Documentation v1.35 - Pod Security Admission: https://v1-35.docs.kubernetes.io/docs/concepts/security/pod-security-admission/
- Kubernetes Documentation v1.35 - Network Policies: https://v1-35.docs.kubernetes.io/docs/concepts/services-networking/network-policies/
- Kubernetes Documentation v1.35 - Encrypting Confidential Data at Rest: https://v1-35.docs.kubernetes.io/docs/tasks/administer-cluster/encrypt-data/
- Kubernetes Documentation v1.35 - RBAC Good Practices: https://v1-35.docs.kubernetes.io/docs/concepts/security/rbac-good-practices/
