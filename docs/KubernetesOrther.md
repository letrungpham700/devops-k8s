## Kubernetes installation
### Cài đặt container runtime (docker) trên tất cả VM
***Thực hiện công việc ở bước này trên tất cả các máy ảo***

Bạn cần phải cài đặt một container runtime trên mỗi node trong cụm K8s (Kubernetes) để các Pods có thể chạy ở đó. Và phiên bản K8s 1.26 yêu cầu phải sử dụng một container runtime tương thích với Container Runtime Interface (CRI) của K8s. Đây là một số container runtime phổ biến với Kubernetes:

- [containerd.io](https://containerd.io/)
- [Docker Engine](https://docs.docker.com/engine/install/ubuntu/)

### Cài đặt và thiết lập các yêu cầu cần chuẩn bị trước

#### Tải các module của nhân Linux
``` bash
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter

```

- `modules-load.d` trong Linux là thư mục hệ thống sử dụng để thiết lập các module của kernel được tải lên tiến trình. Nó bao gồm các tệp đuôi `.conf` chỉ định các module được tải khi hệ thống khởi động.

- Module `overlay` được sử dụng để cung cấp overlay filesystem, là một kiểu filesystem cho phép nhiều filesystem xếp chồng lên nhau. Nó cực kỳ hữu dụng trong công nghệ containerization, nơi các container cần những filesystem cô lập của riêng nó.

- Module `br_netfilter` được sử dụng để bật tính năng lọc và thao tác gói tin ở kernel-level, hữu dụng đối với việc kiểm soát lưu lượng mạng và bảo mật. Nó thường được dùng cùng với các mạng của namespace và các thiết bị mạng ảo để cung cấp việc cô lập cũng như định tuyến cho ứng dụng containerized.

Để kiểm tra module overlay và br_netfilter đã được load, chạy lệnh dưới đây:

```bash
lsmod | grep overlay
lsmod | grep br_netfilter
```

#### Chuyển tiếp IPv4 và cho phép iptables nhận diện brigded traffic

```bash
#  Thiết lập các tham số sysctl, luôn tồn tại dù khởi động lại
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

# Áp dụng các tham số sysctl mà không cần khởi động lại
sudo sysctl --system
```

- `sysctl.d` là thư mục hệ thống trong Linux, sử dụng để thiết lập các tham số cho kernel trong runtime. Nó chứa các tệp có đuôi `.conf` quy định các giá trị của biến sysctl, đó là các tham số của kernel có thể được sử dụng để tinh chỉnh hành vi của Linux kernel.

- Trong môi trường containerized, thông thường cần phải bật (đặt giá trị 1) `net.bridge.bridge-nf-call-iptables` để traffic giữa các container có thể lọc bởi iptables firewall của máy chủ. Điều này rất quan trọng vì lý do bảo mật, nó cho phép máy chủ cung cấp các lớp bảo mật mạng cho ứng dụng containerized.

Để kiểm tra `net.bridge.bridge-nf-call-iptables`, `net.bridge.bridge-nf-call-ip6tables`, `net.ipv4.ip_forward` đã được bật trong thiết lập sysctl hay chưa, chạy lệnh:

```bash
sysctl net.bridge.bridge-nf-call-iptables net.bridge.bridge-nf-call-ip6tables net.ipv4.ip_forward
```

### Cài đặt kubeadm, kubelet và kubectl trên VM

***Thực hiện công việc ở bước này trên tất cả các máy ảo***


`Kubeadm` là một command-line tool dùng để khởi tạo một `Kubernetes cluster`. Nó là một bản phân phối chính thức của `Kubernetes` và được thiết kế để đơn giản hóa quá trình thiết lập `Kubernetes cluster`. `Kubeadm` tự động hóa nhiều tác vụ liên quan đến thiết lập cluster chẳng hạn như cấu hình `control plane components`, tạo `TLS certificates`, và thiết lập `Kubernetes networking`.

#### Tắt swap space

Bạn phải tắt tính năng `swap` để `kubelet` hoạt động bình thường. Xem thêm thảo luận về điều này trong [issue](https://github.com/kubernetes/kubernetes/issues/53533):

`Kubelet`, là `node agent` chính chạy trên `worker node`, giả sử mỗi `node` có một lượng bộ nhớ khả dụng cố định. Nếu `node` bắt đầu tiến hành swap, `kubelet` có thể bị delay hoặc các vấn đề khác ảnh hưởng đến tính `stability` và `reliability` của `Kubernetes cluster`. Chính vì vậy, `Kubernetes` khuyên `swap` nên được **disabled** trên mỗi `node` trong `cluster`.

Để tắt `swap` trên Linux, sử dụng:

```bash
# Đầu tiên là tắt swap
sudo swapoff -a

# Sau đó tắt swap mỗi khi khởi động trong /etc/fstab
sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab
```

#### Cài đặt kubeadm, kubelet và kubectl:

- `kubelet`: component chạy trên tất cả các máy trong `cluster` và thực hiện những việc như khởi động các `pod` và `container`.

- `kubectl`: command line tool dùng để nói chuyện với `cluster`.

- `kubeadm`: công cụ cài đặt các component còn lại của `kubernetes cluster`.

`kubeadm` sẽ không cài đặt `kubelet` hay `kubectl` cho bạn, vì vậy hãy đảm bảo chúng sử dụng các phiên bản phù hợp với các `component` khác trong `Kubernetes control plane` mà `kubeadm` cài cho bạn.

> Cảnh báo: Hướng dẫn này sẽ loại bỏ các `Kubernetes packages` ra khỏi mọi tiến trình system upgrade. Do `kubeadm` và `Kubernetes` cần được đặc biệt chú ý mỗi khi upgrade.


#### Cập nhật `apt package index` và cài các `package` cần thiết để sử dụng trong `Kubernetes apt repository`:

```bash
sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl
```

#### Tải `Google Cloud public signing key`:

```bash
sudo mkdir -m 0755 -p /etc/apt/keyrings
sudo curl -fsSLo /etc/apt/keyrings/kubernetes-archive-keyring.gpg https://packages.cloud.google.com/apt/doc/apt-key.gpg
```

#### Thêm `Kubernetes apt repository`:

```bash
echo "deb [signed-by=/etc/apt/keyrings/kubernetes-archive-keyring.gpg] https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list
```

#### Cập nhật lại `apt package index`, cài đặt phiên bản mới nhất của `kubelet`, `kubeadm` và `kubectl`, ghim phiên bản hiện tại tránh việc tự động cập nhật:

```bash
sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```

`kubelet` sẽ tự động khởi động lại mỗi giây vì ở trạng thái crashloop, đợi `kubeadm` đưa ra yêu cầu cần thực hiện.


### Khởi động control plane và nodes
#### Khởi tạo control plane

`control plane` là nơi chạy các `component` bao gồm etcd (cơ sở dữ liệu của `cluster`) và `API Server` (nơi các câu lệnh `kubectl` giao tiếp).

Để tiến hành khởi tạo, chạy câu lệnh sau ở máy ảo mà chúng ta đặt tên là `kubemaster`:

```bash
sudo kubeadm init --apiserver-advertise-address=192.168.56.2 --pod-network-cidr=10.244.0.0/16
```
- `--apiserver-advertise-address=192.168.56.2`: Địa chỉ IP mà máy chủ API sẽ lắng nghe các câu lệnh. Trong hướng dẫn này sẽ là địa chỉa IP của máy ảo `kubemaster`.
- `--pod-network-cidr=10.244.0.0/16`: control plane sẽ tự động phân bổ địa chỉ IP trong `CIDR` chỉ định cho các `pod` trên mọi `node` trong cụm `cluster`. **Bạn sẽ cần phải chọn CIDR sao cho không trùng với bất kỳ dải mạng hiện có để tránh xung đột địa chỉ IP**.

`kubeadm init` đầu tiên sẽ chạy một loại các bước kiểm tra để đảm bảo máy đã sẵn sàng chạy `Kubernetes`. Những bước kiểm tra này sẽ đưa ra các cảnh báo và thoát lệnh khi có lỗi. Kế tiếp `kubeadm init` tải xuống và cài đặt các thành phần của `control plane`. Việc này có thể sẽ mất vài phút, sau khi kết thúc bạn sẽ thấy thông báo:

```bash
Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

You should now deploy a Pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
/docs/concepts/cluster-administration/addons/

You can now join any number of machines by running the following on each node as root:

kubeadm join <control-plane-host>:<control-plane-port> --token <token> --discovery-token-ca-cert-hash sha256:<hash>
```

#### Lưu lại câu lệnh `kubeadm join`... khi init thành công để thêm các node vào `cluster`.

Để `kubectl` có thể dùng với `non-root user`, chạy những lệnh sau, chúng cũng được nhắc trong output khi `kubeadm init` thành công:

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

Mặt khác, nếu bạn là `root user`, có thể dùng lệnh sau:

```bash
export KUBECONFIG=/etc/kubernetes/admin.conf
```

> Cảnh báo: `Kubeadm` cấp certificate trong `admin.conf` để có `Subject: O = system:masters`, `CN = kubernetes-admin`. `system:masters` là một nhóm người dùng siêu cấp, bỏ qua lớp ủy quyền (như [RBAC](https://docs.oracle.com/cd/E19253-01/816-4557/rbac-1/)). Tuyệt đối không chia sẻ tệp `admin.conf` với bất kỳ ai, thay vào đó hãy cấp cho người dùng các quyền tùy chỉnh bằng cách tạo cho họ một tệp `kubeconfig` với lệnh `kubeadm kubeconfig`. Để biết thêm chi tiết hãy đọc [Generating kubeconfig files for additional users](https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/kubeadm-certs/#kubeconfig-additional-users).

### Thêm các node vào cluster

Chạy câu lệnh trong phần output của `kubeadm init` trên tất cả các `worker node` - máy ảo:`kubenode01`, `kubenode02` với sudo permission:

```bash
sudo kubeadm join --token <token> <control-plane-host>:<control-plane-port> --discovery-token-ca-cert-hash sha256:<hash>
```

#### Nếu bạn không lưu lại lệnh `kubeadm join`, quay lại máy control-plane: `kubemaster`.

#### Lấy `<token>` bằng lệnh

```bash
kubeadm token list
```

Mặc định, `<tokens>` sẽ hết hạn sau 24 giờ. Nếu bạn thêm `worker node` khi `<token>` đã hết hạn, bạn có thể tạo `<token>` mới bằng cách chạy lệnh sau trên `control-plane node`:

```bash
kubeadm token create
```
