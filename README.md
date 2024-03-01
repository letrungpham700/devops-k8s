# Kubernetes

Kubernetes là một hệ thống điều phối các containers (vùng chứa) và quản lý chúng ở một quy mô lớn. Để hiểu rõ hơn về hệ thống này, tìm hiểu về cách thiết lập và tạo Kubernetes Cluster bằng Kubeadm trên Ubuntu 20.04. Cùng tìm hiểu qua bài chia sẻ dưới đây.

## Giới thiệu Kubernetes Cluster

Kubernetes là một hệ thống điều phối các containers (vùng chứa) và quản lý chúng theo quy mô. Hệ thống này được phát triển ban đầu bởi Google dựa trên kinh nghiệm vận hành các containers trong quá trình phát triển sản phẩm. Tới thời điểm hiện tại, Kubernetes có mã nguồn mở và được phát triển rất tích cực bởi cộng đồng công nghệ trên toàn thế giới.

Kubeadm tự động cài đặt và cấu hình các thành phần Kubernetes như là máy chủ API, Controller Manager và Kube DNS. Tuy nhiên, Kubeadm không tạo users hoặc xử lý việc cài đặt và cấu hình các dependencies ở cấp độ hệ điều hành.

Đối với các tác vụ sơ bộ này, bạn có thể sử dụng công cụ quản lý cấu hình như Ansible hoặc SaltStack. Việc sử dụng các công cụ trên giúp tạo mới các Cluster (cụm) bổ sung hoặc tạo lại các Cluster hiện có nhưng đơn giản và ít xảy ra lỗi hơn.

### Yêu cầu

- 🚧 Cluster diagram: 1 Control Plane & 2 Nodes.
- 🖥️ Machine operating system: [Ubuntu 20.04](https://vietnix.vn/ubuntu-20-04/)
- ⚙️ System resource: 2 GB of RAM and 2 CPUs per machine.
- 🧱 No firewall configuration. (Allow all traffic by default)
- 🌐 Full network connectivity between all machines in the cluster (private network, network interface ens18 of virtual machines).

### Các bước cần làm
- ▶️ Tạo VM với cấu hình yêu cầu
- ▶️ Cài đặt container runtime (docker) trên tất cả VM
- ▶️ Cài đặt kubeadm, kubelet và kubectl trên tất cả các máy ảo
- ▶️ Khởi động control plane và nodes
- ▶️ Dọn dẹp môi trường
### Cài đặt container runtime (docker) trên tất cả VM
***Thực hiện công việc ở bước này trên tất cả các máy ảo***

Bạn cần phải cài đặt một `container runtime` trên mỗi node trong cụm K8s (Kubernetes) để các `Pods` có thể chạy ở đó. Và phiên bản K8s 1.26 yêu cầu phải sử dụng một `container runtime` tương thích với `Container Runtime Interface (CRI)` của K8s. Đây là một số `container runtime` phổ biến với Kubernetes:

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

