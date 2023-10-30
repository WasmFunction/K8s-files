# 构建与部署

基于WebAssembly的容器技术研究项目系统的构建与部署。



## Kubernetes



### Install WasmEdge

- [Install and uninstall WasmEdge | WasmEdge Developer Guides](https://wasmedge.org/docs/start/install/)

安装官方版本0.11.2的WasmEdge。

```bash
curl -sSf https://raw.githubusercontent.com/WasmEdge/WasmEdge/master/utils/install.sh | bash -s -- -v 0.11.2
```



### Install wasmkeeper

wasmkeeper用于创建容器进程，运行WebAssembly实例。

- [WasmFunction/wasmkeeper: A simple http server that runs wasm functions. (github.com)](https://github.com/WasmFunction/wasmkeeper)

**构建条件**

- cmake
- vcpkg
- wasmedge v0.11.2

**编译与安装**

```bash
# install dependency using vcpkg
# cd path/to/vcpkg
./vcpkg install jsoncpp

# clone repository
git clone https://github.com/WasmFunction/wasmkeeper.git
cd wasmkeeper

# cd path/to/wasmkeeper
mkdir build && cd build

# change [vcpkg root] to path/to/vcpkg
cmake .. -DCMAKE_TOOLCHAIN_FILE=[vcpkg root]/scripts/buildsystems/vcpkg.cmake

# compile
cmake --build .

# install
cp wasmkeeper /usr/local/bin/
```



### Install wasm-sandboxer

用于被Containerd调用，提供沙盒管理服务和Task服务，负责调用wasmkeeper创建容器进程、管理容器。

- [WasmFunction/kuasar: A multi-sandbox container runtime that provides cloud-native, all-scenario multiple sandbox container solutions. (github.com)](https://github.com/WasmFunction/kuasar)

**构建条件**

- Rust

**编译与安装**

```bash
# clone repository
git clone https://github.com/WasmFunction/kuasar.git
cd kuasar
# *** important ***
git checkout wasmkeeper

# compile
make bin/wasm-sandboxer

# install
cp bin/wasm-sandboxer /usr/local/bin/
```

**运行**

这里推荐使用tmux在后台运行wasm-sandboxer。

```bash
tmux new-session -d -s wasm-sandboxer 'wasm-sandboxer --listen /run/wasm-sandboxer.sock --dir /run/kuasar-wasm'
```



### Install Containerd

适配wasm-sandboxer的Containerd，同时配合Fission更新Pod IP。

- [WasmFunction/containerd at fission_event (github.com)](https://github.com/WasmFunction/containerd/tree/fission_event)

**构建条件**

- Go

**编译与安装**

其中**runc**和**CNI**安装与原版Containerd相同，如果以下安装方法失效可以参考Containerd官方文档安装。

- [containerd/docs/getting-started.md at main · containerd/containerd (github.com)](https://github.com/containerd/containerd/blob/main/docs/getting-started.md)

1. 安装**runc**，以能够运行OCI容器。

   Download the `runc.<ARCH>` binary from https://github.com/opencontainers/runc/releases , verify its sha256sum, and install it as `/usr/local/sbin/runc`.

   ```bash
   install -m 755 runc.amd64 /usr/local/sbin/runc
   ```

2. 安装**CNI**插件。

   Download the `cni-plugins-<OS>-<ARCH>-<VERSION>.tgz` archive from https://github.com/containernetworking/plugins/releases , verify its sha256sum, and extract it under `/opt/cni/bin`：

   ```bash
   mkdir -p /opt/cni/bin
   
   tar Cxzvf /opt/cni/bin cni-plugins-<OS>-<ARCH>-<VERSION>.tgz
   ```

3. 编译与安装**Containerd**。

   ```bash
   # clone repository
   git clone https://github.com/WasmFunction/containerd.git
   cd containerd
   # *** important ***
   git checkout fission_event
   
   make bin/containerd
   
   cp bin/containerd  /usr/local/bin/
   ```

   将`kubernetes/K8s-files`目录下的containerd配置文件`config.toml.kuasar`，放至Containerd的配置目录。

   ```bash
   mkdir /etc/containerd/
   cp config.toml.kuasar /etc/containerd/config.toml
   ```

​		*该配置文件在Kuasar的Containerd的配置文件的基础上，将CNI插件配置更换为K3s的配置。*

**运行**

这里推荐使用tmux在后台运行Containerd。

```bash
tmux new-session -d -s containerd 'ENABLE_CRI_SANDBOXES=1 containerd'
```



### Install K3s

在原K3s的基础上优化了定时器问题。

- [WasmFunction/k3s-k8s: customized kubernetes from k3s (github.com)](https://github.com/WasmFunction/k3s-k8s)

**构建条件**

- Go
- Docker

**编译与安装**

```bash
# clone repository
git clone https://github.com/WasmFunction/k3s-k8s.git
cd k3s-k8s

# config
mkdir -p build/data && make download && make generate

# compile
SKIP_VALIDATE=true make

# install
cp ./dist/artifacts/k3s /usr/local/bin/
```

**运行**

使用`kubernetes/K8s-files`目录下的`k3s_install.sh`来初始化k3s service。

**主节点**

```bash
# init service
INSTALL_K3S_SKIP_DOWNLOAD=true sh ./k3s_install.sh --container-runtime-endpoint /run/containerd/containerd.sock

# kubeconfig set up
mkdir ~/.kube
cp /etc/rancher/k3s/k3s.yaml ~/.kube/config
```

*如果主节点不运行wasm工作负载，则无需安装上述程序。*

```bash
# init service
INSTALL_K3S_SKIP_DOWNLOAD=true sh ./k3s_install.sh

# kubeconfig set up
mkdir ~/.kube
cp /etc/rancher/k3s/k3s.yaml ~/.kube/config
```

**从节点**

主节点初始化k3s之后在**主节点**查看`token`

```bash
cat /var/lib/rancher/k3s/server/token
```

将`[master ip]`和`token`替换为主节点实际值，运行命令使从节点加入集群。

```bash
INSTALL_K3S_SKIP_DOWNLOAD=true K3S_URL=https://[master ip]:6443 K3S_TOKEN=[token] sh ./k3s_install.sh --container-runtime-endpoint /run/containerd/containerd.sock
```



### Test

使用镜像和配置文件部署应用进行测试。

```bash
# clone repository
git clone https://github.com/WasmFunction/K8s-files
cd K8s-files

# load image
ctr -n k8s.io images import --all-platforms apps/img.tar

# create RuntimeClass
kc -f apps/wasm-runtime.yaml

# create Pod
kc -f apps/wasm.yaml

# test service
curl -X POST -H "Content-Type: application/json" -d '{"args":["White", "Hank"]}' localhost:32132
```


