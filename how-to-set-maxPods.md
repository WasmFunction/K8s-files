## How to set maxPods

K8s默认节点的Pod上限是110。可以通过kubelet的配置文件更改上限值。

1. 创建一个kubelet的配置文件。如`/etc/rancher/k3s/kubelet.yaml。`

   ```		yaml
   apiVersion: kubelet.config.k8s.io/v1beta1
   kind: KubeletConfiguration
   maxPods: 250
   ```

2. 更改k3s的服务文件`/etc/systemd/system/k3s.service`启动项，追加启动配置使k3s使用kubelet配置文件。

   ```service
   ExecStart=/usr/local/bin/k3s \
     ......
     '--kubelet-arg=config=/etc/rancher/kubelet.yaml' \
   ```

   追加内容为`--kubelet-arg=config=/etc/rancher/kubelet.yaml`，将路径替换为实际路径。

3. 重新加载服务。

   ```bash
   sysetmctl daemon-reload
   # 如果是主节点
   systemctl resetart k3s.service
   # 如果是工作节点
   systemctl resetart k3s-agent.service
   ```

4. 查看是否生效。

   ```
   > kubectl describe node xxx | grep pods
     pods:               250
   ```