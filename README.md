# Home Infrastructure

这个仓库保存个人基础设施配置。目前包含一套 Ansible 配置，用于在远程 AlmaLinux 宿主机上通过 libvirt/KVM 创建 AlmaLinux 10 虚拟机，并使用 `lablabs.rke2` role 初始化 RKE2 Kubernetes 集群。

## 目录

```text
ansible/
  config/                 集群和虚拟机统一配置
  inventory/              Ansible inventory 示例和本地 inventory
  playbooks/              部署 playbook
  templates/              cloud-init 模板
  generated/              本地拉回的 kubeconfig，不应提交
```

## 当前拓扑

默认部署 2 台 AlmaLinux 10 VM：

```text
rke2-server-1   RKE2 server   192.168.122.11
rke2-agent-1    RKE2 agent    192.168.122.12
```

VM 默认挂到远程 AlmaLinux 宿主机的专用 bridge `br-rke2`。Ansible 通过 SSH ProxyCommand 经宿主机连接 VM。

## 使用

```bash
cd ansible
ansible-galaxy install -r requirements.yml
python -m pip install -r requirements.txt
ansible-playbook playbooks/site.yml
```

如果远程用户需要 sudo 密码：

```bash
ansible-playbook playbooks/site.yml --ask-become-pass
```

完整说明见 [ansible/README.md](ansible/README.md)。
