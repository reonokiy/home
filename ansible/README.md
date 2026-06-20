# Ansible RKE2 on AlmaLinux VMs

这个项目用 Ansible 在远程 AlmaLinux 宿主机上准备 KVM/libvirt，创建 AlmaLinux 10 cloud-init 虚拟机，并使用现成的 [`lablabs.rke2`](https://github.com/lablabs/ansible-role-rke2) role 安装 RKE2 Kubernetes 集群。

默认拓扑：

```text
rke2-server-1   RKE2 server   192.168.122.11
rke2-agent-1    RKE2 agent    192.168.122.12
```

## 设计

- `prepare_hypervisor.yml`：准备远程 AlmaLinux 宿主机上的 libvirt/KVM、storage pool、NAT DHCP reservation。
- `create_vms.yml`：用 `community.libvirt.virt_cloud_instance` 从 AlmaLinux 10 GenericCloud 镜像创建 VM。
- `install_rke2.yml`：等待 VM SSH 可达，然后调用 `lablabs.rke2` role 安装 RKE2。

VM 内部事务全部通过 Ansible SSH 管理。控制机通过 SSH ProxyCommand 连接到 `iots` 后面的 `192.168.122.x` VM。

## 初始化

```bash
ansible-galaxy install -r requirements.yml
python -m pip install -r requirements.txt
```

编辑：

- `inventory/hosts.yml`：宿主机 SSH 信息、RKE2 VM IP/MAC、ProxyCommand。
- `config/cluster.yml`：AlmaLinux 镜像、VM 规格、RKE2 版本、token、kubeconfig 输出路径。

`inventory/hosts.yml` 里的 known_hosts 路径通过 `{{ inventory_dir }}/../known_hosts` 计算，不依赖固定的本机仓库目录。`libvirt_pool_path` 是远程宿主机上的 VM 磁盘目录，默认是 `/var/lib/libvirt/images/rke2`，需要改磁盘位置时只改 `config/cluster.yml`。

## 执行

分步执行：

```bash
ansible-playbook playbooks/prepare_hypervisor.yml --ask-become-pass
ansible-playbook playbooks/create_vms.yml --ask-become-pass
ansible-playbook playbooks/install_rke2.yml
```

或者一次执行：

```bash
ansible-playbook playbooks/site.yml --ask-become-pass
```

## Kubeconfig

`lablabs.rke2` 会把 kubeconfig 下载到：

```text
generated/rke2.yaml
```

因为 VM 在 `iots` 的 `virbr0` NAT 后面，本机使用 `kubectl` 时需要转发 Kubernetes API：

```bash
ssh -L 6443:192.168.122.11:6443 iots@iots.i.nokiy.net
```

另开终端：

```bash
cp generated/rke2.yaml generated/rke2.local.yaml
kubectl --kubeconfig generated/rke2.local.yaml config set-cluster default --server=https://127.0.0.1:6443
KUBECONFIG=generated/rke2.local.yaml kubectl get nodes
```

## 重要变量

- `almalinux_cloud_image_url`：AlmaLinux 10 GenericCloud qcow2 镜像 URL。
- `libvirt_pool_path`：远程宿主机上的 VM 磁盘和镜像缓存目录。
- `rke2_vm_*`：VM 用户、CPU、内存、磁盘和 cloud-init SSH 公钥。
- `rke2_version`：RKE2 版本。
- `rke2_token`：RKE2 server/agent join token，正式使用前应修改。

## 调试

VM 层：

```bash
sudo virsh --connect qemu:///system list --all
sudo virsh --connect qemu:///system domifaddr rke2-server-1
sudo virsh --connect qemu:///system console rke2-server-1
```

RKE2 层：

```bash
ssh -J iots@iots.i.nokiy.net ansible@192.168.122.11
sudo systemctl status rke2-server
sudo journalctl -u rke2-server -f
```
