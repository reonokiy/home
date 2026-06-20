# Ansible Talos on AlmaLinux

这个项目用 Ansible 在一台远程 AlmaLinux 宿主机上准备 KVM/libvirt，创建 1 个 Talos control-plane VM 和 1 个 Talos worker VM，并把它们初始化为 Kubernetes 集群。

## 前置条件

- 控制机已安装 `ansible`、`kubectl`。
- 控制机能通过 SSH 登录远程 AlmaLinux 宿主机。
- 远程 AlmaLinux 宿主机支持硬件虚拟化，且你愿意让 Ansible 安装并启用 libvirt。
- VM 使用静态 IP；这些 IP 必须能从 AlmaLinux 宿主机访问到 Talos API。

网络模型：默认 `libvirt_network_bridge: virbr0` 使用远程宿主机的 libvirt NAT 网络。Ansible 会把 `talosctl` 安装到 AlmaLinux 宿主机，并在宿主机上初始化 Talos，因为宿主机能访问 `192.168.122.x` 的 VM IP。控制机不需要直接访问 Talos VM。

首次启动 Talos ISO 时，Talos maintenance 模式会先通过 DHCP 获取地址。使用 `virbr0` 时，playbook 会在 libvirt default NAT 网络里为每个 Talos VM 的 MAC 添加 DHCP 固定租约，确保它们拿到 inventory 中配置的 `talos_ip`。

## 初始化

```bash
ansible-galaxy collection install -r requirements.yml
cp inventory/hosts.example.yml inventory/hosts.yml
```

编辑：

- `inventory/hosts.yml`：填写 AlmaLinux 宿主机的 `ansible_host`、`ansible_user`，以及每个 Talos VM 的 IP/MAC。
- `config/cluster.yml`：调整 Talos/Kubernetes 版本、网关、DNS、libvirt bridge、VM CPU/内存/磁盘。

远程 AlmaLinux 宿主机需要 root/sudo 权限。推荐用普通用户加 sudo：

```yaml
hypervisors:
  hosts:
    alma-hypervisor:
      ansible_host: 203.0.113.10
      ansible_user: your_user
      ansible_become: true
```

如果 sudo 需要密码，执行时加：

```bash
ansible-playbook playbooks/site.yml --ask-become-pass
```

如果你要直接用 root SSH，也可以写成：

```yaml
hypervisors:
  hosts:
    alma-hypervisor:
      ansible_host: 203.0.113.10
      ansible_user: root
```

默认拓扑：

```text
talos-cp-1      control-plane   192.168.122.11
talos-worker-1  worker          192.168.122.12
```

## 执行

分步执行更容易排错：

```bash
ansible-playbook playbooks/prepare_hypervisor.yml
ansible-playbook playbooks/create_vms.yml
ansible-playbook playbooks/bootstrap_talos.yml
```

或者一次执行：

```bash
ansible-playbook playbooks/site.yml
```

这些 playbook 设计为可以重复运行：

- 已存在的 VM 磁盘不会被覆盖。
- 已启动的 libvirt domain 会保持运行。
- Talos secrets 会保存在远程宿主机的 `/var/lib/libvirt/images/talos/generated/secrets.yaml`，不会每次重建。
- 基础 Talos 配置缺失时才生成；如果要强制重写，设置 `talos_regenerate_configs: true`。
- 如果 Talos API 已可用，`bootstrap_talos.yml` 会跳过 insecure 初始安装步骤，只刷新 kubeconfig。

如果你明确要重新走首次 `talosctl apply-config --insecure` 流程，把 `config/cluster.yml` 里的 `talos_force_initial_apply` 临时设为 `true`。注意这只适合节点仍处于 Talos maintenance 模式时使用。

完成后 kubeconfig 会写到：

```bash
generated/kubeconfig
```

Talos 初始化状态保存在远程 AlmaLinux 宿主机：

```text
/var/lib/libvirt/images/talos/generated/
```

注意：默认 kubeconfig 里的 API server 仍然是 `https://192.168.122.11:6443`。如果你的控制机不能直接访问这个地址，需要通过 SSH 转发：

```bash
ssh -L 6443:192.168.122.11:6443 iots.i.nokiy.net
```

然后另开一个终端，把 kubeconfig 里的 server 临时指到本地转发端口：

```bash
cp generated/kubeconfig generated/kubeconfig.local
kubectl --kubeconfig generated/kubeconfig.local config set-cluster talos-k8s --server=https://127.0.0.1:6443
KUBECONFIG=generated/kubeconfig.local kubectl get nodes
```

## 重要变量

- `libvirt_pool_path`：远程 AlmaLinux 宿主机上的统一存储目录。默认 `/var/lib/libvirt/images/talos`，Talos ISO 和 VM qcow2 磁盘都放在这里。
- `libvirt_network_bridge`：默认 `virbr0`。Talos 初始化命令会在 AlmaLinux 宿主机上执行，所以 VM IP 只需要从宿主机可达。
- `libvirt_nat_network_name`：默认 `default`，用于给 `virbr0` NAT 网络添加 Talos DHCP 固定租约。
- `talos_gateway` / `talos_network_prefix` / `talos_nameservers`：必须匹配 VM 所在网络。
- `talos_controlplane_node`：默认 `talos-cp-1`，用于决定 Kubernetes API endpoint。
- `talos_endpoint_ip`：默认取 `talos_controlplane_node` 的 IP。生产环境可改成负载均衡或 VIP。
- `talos_install_disk`：默认 `/dev/vda`，对应 libvirt XML 里的 virtio 磁盘。
- `talos_allow_scheduling_on_controlplanes`：默认 `false`，业务负载只调度到 worker。

## 注意

远程 `/var/lib/libvirt/images/talos/generated/` 内会包含 Talos secrets、talosconfig 和 kubeconfig；本地 `generated/` 会保存拉回来的 kubeconfig/talosconfig。不要提交或公开这些文件。
