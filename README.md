# Home Infrastructure

这个仓库保存个人基础设施配置。目前包含一套 Ansible 配置，用于在远程 AlmaLinux 宿主机上通过 libvirt/KVM 创建 Talos Linux 虚拟机，并初始化一个 Kubernetes 集群。

## 目录

```text
ansible/
  config/                 集群和虚拟机统一配置
  inventory/              Ansible inventory 示例和本地 inventory
  playbooks/              部署 playbook
  templates/              libvirt 与 Talos 配置模板
  generated/              本地拉回的 kubeconfig/talosconfig，不应提交
```

## 当前拓扑

默认部署 2 台 Talos VM：

```text
talos-cp-1      control-plane   192.168.122.11
talos-worker-1  worker          192.168.122.12
```

Talos VM 位于远程 AlmaLinux 宿主机的 `virbr0` NAT 网络后面。`talosctl` 会安装并运行在远程宿主机上，因此控制机不需要直接访问 Talos VM。

## 使用

```bash
cd ansible
ansible-galaxy collection install -r requirements.yml
ansible-playbook playbooks/site.yml
```

如果远程用户需要 sudo 密码：

```bash
ansible-playbook playbooks/site.yml --ask-become-pass
```

完整说明见 [ansible/README.md](ansible/README.md)。

