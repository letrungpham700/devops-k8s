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
