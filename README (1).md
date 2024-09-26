# Hygon平台cri-rm用户手册

 
## 1.介绍
Cri-rm（cri-resource-manager）是intel开源的k8s cni代理。负责截获k8s下发给CNI的资源请求，通过对节点硬件拓扑感知输出最优的容器资源分配方案。   
由于hygon平台与intel平台的差异，我们做了更精细的适配优化，使之较原始项目在hygon平台有更优异的性能。  
本文旨在介绍如何在hygon平台使用优化后的cri-rm。


## 2.使用cri-rm

### 2.1准备工作
在安装cri-rm之前，保证系统上k8s已经正常安装并可用。本文不涉及k8s的安装部署。

### 2.2安装配置
Cri-rm截获的是kubelet传给cri的信息，因此只有安装了cri-rm的k8s节点才能有numa优化的能力。  
一般只在k8s worker节点上安装cri-rm。

### 2.2.1安装
下载0.8.4 rpm或deb包，url为https://github.com/intel/cri-resource-manager/releases   
使用rpm或dpkg安装cri-rm。Cri-rm会安装binary到/usr/bin，用hygon提供的cri-resmgr静态binary替换掉/usr/bin/cri-resmgr。

### 2.2.2配置
1.Cri-rm的配置文件位于/etc/cri-resmgr/，进入该目录，复制fallback.cfg.sample为fallback.cfg  
2.替换kubelet cni为cri-resmgr  
默认情况下，kubelet的cni为containerd，替换成cri-rm。对于高版本的kubelet，对应配置文件为：/var/lib/kubelet/kubeadm-flags.env，下面是一个参考:

```
KUBELET_KUBEADM_ARGS="--container-runtime-endpoint=unix:///var/run/cri-resmgr/cri-resmgr.sock --pod-infra-container-image=registry.aliyuncs.com/google_containers/pause:3.9"
```

### 2.2.3启动
启动cri-rm:
```
systemctl start cri-resource-manager
systemctl start kubelet
```
查看cri-rm是否active:
```
systemctl status cri-resource-manager
```
查看kubelet是否active:
```
systemctl status kubelet
```
查看该节点是否ready，在master节点查看pod:
```
kubectl get pods
```


# 3.测试
1.安装crictl用于查看资源分配情况 ，该工具：https://github.com/kubernetes-sigs/cri-tools  
2.下面是一个安装的例子：

```
VERSION="v1.31.0"
wget https://github.com/kubernetes-sigs/cri-tools/releases/download/$VERSION/crictl-$VERSION-linux-amd64.tar.gz
sudo tar zxvf crictl-$VERSION-linux-amd64.tar.gz -C /usr/local/bin
rm -f crictl-$VERSION-linux-amd64.tar.gz
```

在master节点创建一个yaml用于创建pod。设置cpu数量，假设为1，创建pod后，查看pod运行的node，在对应node上使用crictl查看容器cpuset。方法为：
1. 首先使用crictl ps查看启动的容器id  
2. Crictl inspect $id | grep –i cpuset

接着可试验其它cpu数量，观察容器被分配的cpuset。正常情况下,用伪代码表示为：

```
If cpuNum <= cap(ccx); then
	ret=ccx;
elif cpuNum <= cap(numa); then
	ret=numa;
elif cpuNum <= cap(numa) + cap(ccx); then
	ret=numa+ccx;
elif cpuNum <= 2 * cap(numa); then
	ret = numa + numa
else
	ret = socket or whole system
```
 
# 4.其他
该优化后的patch只能在hygon/amd平台工作，不适用intel/arm平台。
对于虚拟机场景，需要将host上的cpu拓扑正确passthrough到VM中。
例如将某个numa节点绑定到VM中，在libvirt的xml中：

```
  <vcpu placement='static'>32</vcpu>
  <cputune>
    <vcpupin vcpu='0' cpuset='0'/>
    <vcpupin vcpu='1' cpuset='64'/>
    <vcpupin vcpu='2' cpuset='1'/>
    <vcpupin vcpu='3' cpuset='65'/>
    <vcpupin vcpu='4' cpuset='2'/>
    <vcpupin vcpu='5' cpuset='66'/>
  </cputune>
```

Vcpu0对应cpu0，vcpu1必须对应与cpu0同在一个cpu核中的另一个cpu线程，以此类推。因为在qemu-kvm虚拟机默认将两个在同一个cpu核中的超线程连续命名。

```
<cpu mode='host-passthrough' check='none' migratable='on'>
<topology sockets='1' dies='1' cores='16' threads='2'/>
<cache mode='passthrough'/>
<feature policy='require' name='topoext'/>
```

Cpu mode必须设置为‘host-passthrough’，cache mode设为’passthrough’，feature policy必须添加topoext。单个numa中cores，threads数量必须与host保持一致。

# 异常处理
在安装过程中如遇到启动不成功的情况，可以按照以下步骤解决：

```
$ sudo rm -f /var/lib/cri-resmgr/
$ sudo systemctl restart cri-resource-manager
```
