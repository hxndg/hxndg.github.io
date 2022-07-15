---
layout:     post   				    # 使用的布局（不需要改）
title:      ci_cd相关学习
subtitle:   看起来有好多要学要做 #副标题
date:       2021-10-11 				# 时间
author:     大狗 						# 作者
header-img: img/post-bg-map.jpg
catalog: true 						# 是否归档
tags:								
    - 学习
---

# gitlab常见问题总结

记录一些遇到的问题和解决方法

### 1.1 push镜像失败

push到阿里云服务器时候失败，有两种可能的原因：

1. 没登录，需要登陆下 
2. image的名字过于长了，简单说就是本来image的名字应该是repo_name/namespace/image_name:tag，如果image_name特别长比方说是/xxx/yyy/zzz/ddd/aaa/eee/ccc这种，name就会出现push失败的情况。提示requested access to the resource is denied。这种情况属于阿里云本身的问题

```shell

The push refers to repository [registry.qcraftai.com/qcraft/sim_server]
...
21639b09744f: Waiting
78220a8ee18f: Waiting
c65cd6950943: Waiting
29c579ad1c5a: Waiting
767a7c7801b5: Waiting
a24b8da85c42: Waiting
denied: requested access to the resource is denied
```



### 1.2 本地format.sh格式化过了，ci还是不过

错误原因是dev docker过旧，新版本的format.sh有变化，所以更新dev docker即可

```shell
[ OK ] Congrats, commit author check pass
[ OK ] Done buildifier /builds/root/qcraft/offboard/dashboard/BUILD
[INFO] Done formatting /builds/root/qcraft/offboard/dashboard/health.proto
[ OK ] Done buildifier /builds/root/qcraft/offboard/dashboard/services/health/BUILD
[INFO] Done formatting /builds/root/qcraft/offboard/dashboard/services/health/health.cc
[INFO] Done formatting /builds/root/qcraft/offboard/dashboard/services/health/health.h
[INFO] Done formatting /builds/root/qcraft/offboard/dashboard/services/health/health_client.cc
[INFO] Done formatting /builds/root/qcraft/offboard/dashboard/sim_server_main.cc
[ OK ] Done formatting /builds/root/qcraft/production/k8s/offboard/dashboard/sim_server_v2/deploy.sh
[ERROR] Format issue found, please run "scripts/format.sh --git" before commit
 offboard/dashboard/services/health/health_client.cc | 4 ++--
 1 file changed, 2 insertions(+), 2 deletions(-)
```





### 1.3 Job一直pending

job一直在pending，任务很忙，一直拿不到？目前看起来有什么可能的原因呢？有多种可能：

+ master发布升级命令，然后pod开始升级，包括拉取镜像，挂载设备等等。如果镜像仓库有问题，一直拉不下来，那么就会一直pending，比方说2021/11/11号早上，阿里云仓库出了问题，镜像一直拉不下来。这里多说一句，CI的镜像是在master里面通过对CN_IMG/USA_IMG指定的

```
Waiting for pod gitlab-runner/runner-u15fg-a7-project-4-concurrent-0tw4c2 to be running, status is Pending
	ContainersNotReady: "containers with unready status: [build helper]"
	ContainersNotReady: "containers with unready status: [build helper]"
Waiting for pod gitlab-runner/runner-u15fg-a7-project-4-concurrent-0tw4c2 to be running, status is Pending
	ContainersNotReady: "containers with unready status: [build helper]"
	ContainersNotReady: "containers with unready status: [build helper]"
Waiting for pod gitlab-runner/runner-u15fg-a7-project-4-concurrent-0tw4c2 to be running, status is Pending
	ContainersNotReady: "containers with unready status: [build helper]"
	ContainersNotReady: "containers with unready status: [build helper]"
Waiting for pod gitlab-runner/runner-u15fg-a7-project-4-concurrent-0tw4c2 to be running, status is Pending
	ContainersNotReady: "containers with unready status: [build helper]"
	ContainersNotReady: "containers with unready status: [build helper]"
```



### 1.4 代码保护报错

代码保护导致的错误，代码保护软件似乎有个问题，即白名单加多了以后就不生效了，然后gitlab-runner拉代码就会出先下面的错误。今天虽然加了代码但是没生效的原因是因为gitlab-runner上的客户端死了，所以没同步配置。代码就没拉下来。

```
[580s] can not find refs/pipelines/90402, retry later ...
fatal: unable to access 'https://gitlab-cn.qcraftai.com/root/qcraft.git/': error:1408F10B:SSL routines:ssl3_get_record:wrong version number
[585s] can not find refs/pipelines/90402, retry later ...
fatal: unable to access 'https://gitlab-cn.qcraftai.com/root/qcraft.git/': error:1408F10B:SSL routines:ssl3_get_record:wrong version number
[590s] can not find refs/pipelines/90402, retry later ...
fatal: unable to access 'https://gitlab-cn.qcraftai.com/root/qcraft.git/': error:1408F10B:SSL routines:ssl3_get_record:wrong version number
[595s] can not find refs/pipelines/90402, retry later ...
fatal: unable to access 'https://gitlab-cn.qcraftai.com/root/qcraft.git/': error:1408F10B:SSL routines:ssl3_get_record:wrong version number
[600s] can not find refs/pipelines/90402, retry later ...
can not find refs/pipelines/90402, exit!
Cleaning up file based variables
00:01
ERROR: Job failed: command terminated with exit code 1
```



### 1.5 sim-server的奇怪错误

今天gitlab部署sim-server出错了，然后报错的地方和原因八杆子打不着，原来是配置文件写错哦了

```yaml
           - name: GITLAB_ACCESS_TOKEN
              valueFrom:  //这里多写了一行
              valueFrom:
                secretKeyRef:
                  name: gitlab-access-token
```

具体报错信息在这里

```
...
Events:
  Type     Reason     Age                 From               Message
  ----     ------     ----                ----               -------
  Normal   Scheduled  3m7s                default-scheduler  Successfully assigned staging/sim-server-grey-6bc7997545-mx4t8 to cn-zhangjiakou.172.20.2.197
  Warning  Unhealthy  90s                 kubelet            Liveness probe failed: OCI runtime exec failed: exec failed: container_linux.go:346: starting container process caused "process_linux.go:101: executing setns process caused \"exit status 1\"": unknown
  Warning  Unhealthy  84s (x2 over 94s)   kubelet            Readiness probe failed:
  Normal   Started    67s (x3 over 99s)   kubelet            Started container sim-server
  Warning  Unhealthy  60s                 kubelet            Liveness probe failed:
  Warning  Unhealthy  59s                 kubelet            Readiness probe errored: rpc error: code = Unknown desc = container not running (b47a56e00cddda91ee7e446a77ece77d8c6955320dd11a031a3d8c71372ab3d4)
  Warning  BackOff    44s (x5 over 82s)   kubelet            Back-off restarting failed container
  Normal   Pulling    28s (x4 over 3m5s)  kubelet            Pulling image "registry.qcraftai.com/global/sim_server:c8ad37ec94ebfda0a08106f77210cb92ed67387d"
  Normal   Pulled     28s (x4 over 101s)  kubelet            Successfully pulled image "registry.qcraftai.com/global/sim_server:c8ad37ec94ebfda0a08106f77210cb92ed67387d"
  Normal   Created    27s (x4 over 100s)  kubelet            Created container sim-server

```



看起来是你不小心编辑了/home/qcraft/.aws/credentials 这个文件导致的，要么想办法还原，要么重新setup一次，删了重新跑aws configure

```
有人碰到过这个问题吗？初始化tools报错
[INFO] Start goofys mounting from /qcraftroaddata to /media/s3/run_data_2
2021/12/20 11:02:22.424581 main.FATAL Unable to mount file system, see syslog for details
查看syslog信息是
Dec 20 11:18:28 yanguodong /usr/local/bin/goofys[7687]: main.ERROR Unable to setup backend: SharedConfigLoadError: failed to load config file, /home/qcraft/.aws/credentials#012caused by: INIParseError: invalid state with ASTKind {completed_stmt {0 NONE 0 []} false [{section_stmt {1 STRING 0 [78 111 110 101]} true []}]} and TokenType {4 NONE 0 [58]}
Dec 20 11:18:28 yanguodong /usr/local/bin/goofys[7687]: main.FATAL Mounting file system: Mount: initialization failed
```



### 1.6 安装升级cuda和nvidia驱动

我们的cuda的runtime library已经是11.3了，但是新建的centos7没有装驱动，需要手动装驱动。参考的链接是https://blog.csdn.net/JimmyOrigin/article/details/112972883

```
Events:
  Type     Reason     Age    From               Message
  ----     ------     ----   ----               -------
  Normal   Scheduled  2m36s  default-scheduler  Successfully assigned gitlab-runner/runner-fcakjyg6-project-4-concurrent-0dgs2w to cn-gpu03016ack
  Normal   Pulled     2m29s  kubelet            Container image "registry.gitlab.com/gitlab-org/gitlab-runner/gitlab-runner-helper:x86_64-58ba2b95" already present on machine
  Normal   Created    2m29s  kubelet            Created container init-permissions
  Normal   Started    2m29s  kubelet            Started container init-permissions
  Normal   Pulling    2m28s  kubelet            Pulling image "registry.qcraftai.com/global/qcraft-ci:dev-libgit-20211214_0012"
  Normal   Pulled     91s    kubelet            Successfully pulled image "registry.qcraftai.com/global/qcraft-ci:dev-libgit-20211214_0012" in 56.658518838s
  Normal   Created    81s    kubelet            Created container build
  Warning  Failed     81s    kubelet            Error: failed to start container "build": Error response from daemon: OCI runtime create failed: container_linux.go:380: starting container process caused: process_linux.go:545: container init caused: Running hook #0:: error running hook: exit status 1, stdout: , stderr: nvidia-container-cli: requirement error: unsatisfied condition: cuda>=11.3, please update your driver to a newer version, or use an earlier cuda container: unknown
  Normal   Pulled     81s    kubelet            Container image "registry.gitlab.com/gitlab-org/gitlab-runner/gitlab-runner-helper:x86_64-58ba2b95" already present on machine
  Normal   Created    81s    kubelet            Created container helper
  Normal   Started    81s    kubelet            Started container helper
```



centos7系统安装gpu机器驱动和cuda：

```
# 以root用户登录进centos7.9系统，删除旧的nvidia驱动
yum remove nvidia*
# 需要重启才能卸载当前正在运行的nvidia mod
shutdown -r now
# 大部分的驱动安装都说要禁用nouveau，但是我使用默认的centos7检查的时候lsmod nouveau直接就没有，所以我们省略了这一步
# 下载nvidia驱动文件

# 更新依赖组件和包
yum update
yum groupinstall "Development Tools"
yum install kernel-devel epel-release
# 确定提示出来的内核的版本一致，不一致使用yum -y upgrade kernel kernel-devel
uname -r
rpm -q kernel-devel
# 运行驱动，执行安装.安装32bit nvidia驱动时选择no,update your x configuration选择yes。剩下的都ok即可
chmod +x ./NVIDIA-Linux-x86_64-470.94.run 
./NVIDIA-Linux-x86_64-470.94.run

# 查看匹配的cuda版本
nvidia smi
#执行结果见下
[root@cn-gpu03016ack ~]# nvidia-smi
Tue Dec 21 13:06:23 2021       
+-----------------------------------------------------------------------------+
| NVIDIA-SMI 470.94       Driver Version: 470.94       CUDA Version: 11.4     |
|-------------------------------+----------------------+----------------------+
| GPU  Name        Persistence-M| Bus-Id        Disp.A | Volatile Uncorr. ECC |
| Fan  Temp  Perf  Pwr:Usage/Cap|         Memory-Usage | GPU-Util  Compute M. |
|                               |                      |               MIG M. |
|===============================+======================+======================|
|   0  NVIDIA GeForce ...  Off  | 00000000:00:08.0 Off |                  N/A |
| 15%   41C    P0    64W / 250W |      0MiB / 11019MiB |      0%      Default |
|                               |                      |                  N/A |
+-------------------------------+----------------------+----------------------+
                                                                               
+-----------------------------------------------------------------------------+
| Processes:                                                                  |
|  GPU   GI   CI        PID   Type   Process name                  GPU Memory |
|        ID   ID                                                   Usage      |
|=============================================================================|
|  No running processes found                                                 |
+-----------------------------------------------------------------------------+
# 下载对应的CUDA版本
wget https://developer.download.nvidia.com/compute/cuda/11.4.0/local_installers/cuda_11.4.0_470.42.01_linux.run

# 进行安装,这里有几点要注意: 出现一个文档，这里要输入accept。然后CUDA installer里面组件选择去掉驱动的选中，我们已经装了驱动了，然后再Install
sh cuda_11.4.0_470.42.01_linux.run

# 添加CUDA到环境变量
export PATH=/usr/local/cuda-11.4/bin:$PATH
export LD_LIBRARY_PATH=$LDLIBRARY_PATH:/usr/local/cuda-11.4/lib64
source ~/.bashrc

# 测试cuda指令，可以看到cuda已经安装成功，版本为11.4
nvcc -V

# 第一步的时候把nvidia-docker啥的都删除了，所以需要重新添加repo源
distribution=$(. /etc/os-release;echo $ID$VERSION_ID)    && curl -s -L https://nvidia.github.io/nvidia-docker/$distribution/nvidia-docker.repo | sudo tee /etc/yum.repos.d/nvidia-docker.repo
# 更新 yum cache
yum clean expire-cache
# 安装nvidia-docker
yum install -y nvidia-docker2
# 重启docker
systemctl restart docker
```



最后整理出来的总共的初始化脚本如下：

```shell
#!/bin/bash

set -e

function add_current_script_to_rc_local() {
  script_path=$(realpath $0)
  echo "Adding boot shell $script_path"
  echo "$script_path" >> /etc/rc.d/rc.local
}

function remove_current_script_from_rc_local() {
  script_path=$(realpath $0)
  echo "Removing boot shell $script_path"
  sed -i "s#$script_path##g" /etc/rc.d/rc.local
}

function remove_nvidia_driver() {
  echo "Remove existing driver"
  echo "Remove existing nvidia driver container cli"
  yum remove -y nvidia*

  # Use default nvidia-uninstall bin to perform uninstall
  if [[ -x "/usr/bin/nvidia-uninstall" ]]; then
    echo "Remove existing nvidia driver using nvidia-uninstall"
    /usr/bin/nvidia-uninstall --silent
  fi
}

function create_nvidia_cuda_folder() {
  echo "Create folder /root/nvidia_cuda"
  if [[ ! -d "/root/nvidia_cuda" ]]; then
    mkdir "/root/nvidia_cuda"
  fi
}

function get_nvidia_driver_version() {
  echo $(nvidia-smi --query-gpu=driver_version --format=csv,noheader)
}

function download_nvidia_11_4_0_470_42_01() {
  wget -O cuda_11.4.0_470.42.01_linux.run "https://qcraft-images.oss-cn-zhangjiakou-internal.aliyuncs.com/cuda_470_42_01/cuda_11.4.0_470.42.01_linux.run"
  cuda_md5=$(md5sum ./cuda_11.4.0_470.42.01_linux.run | awk '{print $1}')
  if [[ $cuda_md5 == "cbcc1bca492d449c53ab51c782ffb0a2" ]]; then
    echo "Download cuda successfull" >> /root/nvidia_cuda/install.log
  else
    echo "Download cuda failed" >> /root/nvidia_cuda/install.log
    exit 1
  fi
}

function install_driver_needed_files() {
  echo "Installing needed header files" >> /root/nvidia_cuda/install.log
  yum update -y
  yum groupinstall -y "Development Tools"
  yum install -y kernel-devel epel-release
}

function install_nvidia_cuda_driver() {
  driver_installed=$(get_nvidia_driver_version)
  if [[ $driver_installed == "470.42.01" ]]; then
    echo "Already install nvidia driver 470.42.01, Abort install" >> /root/nvidia_cuda/install.log
    exit 0
  else
    echo "Planning to install 470.42.01 cuda & driver" >> /root/nvidia_cuda/install.log
    install_driver_needed_files
    download_nvidia_11_4_0_470_42_01
    echo "Start to install 470.42.01 cuda & driver" >> /root/nvidia_cuda/install.log
    chmod +x ./cuda_11.4.0_470.42.01_linux.run
    ./cuda_11.4.0_470.42.01_linux.run --silent
    echo "Success installed 470.42.01 cuda & driver" >> /root/nvidia_cuda/install.log
  fi
}

function install_nvidia_docker_2() {
  echo "Installing nvidia-docker 2" >> /root/nvidia_cuda/install.log
  distribution=$(
    . /etc/os-release
    echo $ID$VERSION_ID
  ) && curl -s -L https://nvidia.github.io/nvidia-docker/$distribution/nvidia-docker.repo | sudo tee /etc/yum.repos.d/nvidia-docker.repo
  yum clean expire-cache
  yum install -y nvidia-docker2
  echo "Installed nvidia-docker 2" >> /root/nvidia_cuda/install.log
}

function overwrite_daemon_json() {
  wget -O daemon.json "https://qcraft-images.oss-cn-zhangjiakou-internal.aliyuncs.com/cuda_470_42_01/daemon.json"
  json_md5=$(md5sum ./daemon.json | awk '{print $1}')
  if [[ $json_md5 == "8f8be065977394c8c75c0b4c23a2258d" ]]; then
    echo "Download daemon.json successfull" >> /root/nvidia_cuda/install.log
  else
    echo "Download daemon.json failed" >> /root/nvidia_cuda/install.log
    exit 1
  fi
  mv -f daemon.json /etc/docker/daemon.json
}

function modify_docker_sock_permission() {
  if [[ -f "/var/run/docker.sock" ]]; then
    echo "Modify docker.sock permission to 666"
    chmod 666 /var/run/docker.sock
  fi
}

if [ ! -d "/root/nvidia_cuda" ]; then
  remove_nvidia_driver
  create_nvidia_cuda_folder
  add_current_script_to_rc_local
  echo "Finish fisrt stage: remove current nvidia driver & add current stage to boot" >> /root/nvidia_cuda/install.log
  shutdown -r now
else
  remove_current_script_from_rc_local
  cd /root/nvidia_cuda
  install_nvidia_cuda_driver
  install_nvidia_docker_2
  overwrite_daemon_json
  echo "Finish second stage: install nvidia driver & cuda & nvidia docker 2 & daemon.json" >> /root/nvidia_cuda/install.log
fi

modify_docker_sock_permission

echo "Install nvidia driver & cuda success" >> /root/nvidia_cuda/install.log

```







这里的报错虽然是driver name nasplugin.csi.alibabacloud.com not found，但是实际上是运行在每个node上的daemonset没有启动，因此需要启动相对应的守护进程集。之所以出现这个问题是因为在上一步我更新驱动文件的时候将所有的nvidia相关的组件/驱动全删除了，也就罢nvidia-docker2也给删除了，因此日志里面报错：

```
Warning  FailedCreatePodSandBox  4m2s (x4 over 4m5s)    kubelet            (combined from similar events): Failed to create pod sandbox: rpc error: code = Unknown desc = failed to start sandbox container for pod "csi-plugin-n8wgn": Error response from daemon: OCI runtime create failed: unable to retrieve OCI runtime error (open /run/containerd/io.containerd.runtime.v1.linux/moby/238bd4e5cf01dd61498969daacae01454eb59624e9e11732d4bd8aa356fcbaec/log.json: no such file or directory): fork/exec /usr/bin/nvidia-container-runtime: no such file or directory: unknown
```

表现出来的报错信息为

```
Events:
  Type     Reason       Age                   From               Message
  ----     ------       ----                  ----               -------
  Normal   Scheduled    3m9s                  default-scheduler  Successfully assigned gitlab-runner/runner-cm89m7vp-project-4-concurrent-0drj7g to cn-gpu03016ack
  Warning  FailedMount  2m6s (x8 over 3m10s)  kubelet            MountVolume.MountDevice failed for volume "gitlab-runner-pvc-bazel-distdir" : kubernetes.io/csi: attacher.MountDevice failed to create newCsiDriverClient: driver name nasplugin.csi.alibabacloud.com not found in the list of registered CSI drivers
  Warning  FailedMount  2m6s (x8 over 3m10s)  kubelet            MountVolume.MountDevice failed for volume "gitlab-runner-pvc-bazel-repo-cache" : kubernetes.io/csi: attacher.MountDevice failed to create newCsiDriverClient: driver name nasplugin.csi.alibabacloud.com not found in the list of registered CSI drivers
  Warning  FailedMount  2m6s (x8 over 3m10s)  kubelet            MountVolume.MountDevice failed for volume "gitlab-runner-pvc-qcraft-maps-china" : kubernetes.io/csi: attacher.MountDevice failed to create newCsiDriverClient: driver name nasplugin.csi.alibabacloud.com not found in the list of registered CSI drivers
  Warning  FailedMount  67s                   kubelet            Unable to attach or mount volumes: unmounted volumes=[bazel-distdir qcraft-maps-china bazel-repo-cache], unattached volumes=[bazel-distdir qcraft-maps-china default-token-g2v9p docksock logs hosthostname aws repo bazel-repo-cache scripts]: timed out waiting for the condition

```



即使安装了完了诸如nvidia & cuda & nvidia-docker，依然会发现Job没有使用gpu运行，这个时候需要修改/etc/docker/daemon.json文件下面的文件.

还有一个要注意的地方，即使添加了gpu也不一定代表着test会一定rerun，

```
{
    "default-runtime": "nvidia",
    "runtimes": {
        "nvidia": {
            "path": "nvidia-container-runtime",
            "runtimeArgs": []
        }
    },
    "registry-mirror": [
             "https://registry.docker-cn.com"
    ],
        "exec-opts": ["native.cgroupdriver=systemd"],
        "live-restore": true,
        "log-driver": "json-file",
        "log-opts": {
                "max-size": "50m",
                "max-file": "5"
        },
        "bip": "169.254.123.1/24",
        "registry-mirrors": ["https://pqbap4ya.mirror.aliyuncs.com"]
}
```



报错信息为

```
  PASSED  ] 0 tests.
[  FAILED  ] 2 tests, listed below:
[  FAILED  ] list_add_test.ListAdd
[  FAILED  ] list_add_test.ListAddHalf
 2 FAILED TESTS
================================================================================
FAIL: //onboard/nets/custom_ops:gen_coordinates_test (see /home/qcrafter/.cache/bazel/_bazel_qcrafter/b7b2ac012bd759fe3fc931a5a52099ce/execroot/com_qcraft/bazel-out/k8-opt/testlogs/onboard/nets/custom_ops/gen_coordinates_test/test.log)
INFO: From Testing //onboard/nets/custom_ops:gen_coordinates_test:
==================== Test output for //onboard/nets/custom_ops:gen_coordinates_test:
[NVBLAS] No Gpu available
[NVBLAS] NVBLAS_CONFIG_FILE environment variable is NOT set : relying on default config filename 'nvblas.conf'
[NVBLAS] Cannot open default config file 'nvblas.conf'
[NVBLAS] Config parsed
[NVBLAS] CPU Blas library need to be provided
```



这里还有一个奇怪的问题，就是job报错找不到GPU，然后运行失败，但是在docker里面执行却没有问题执行成功。具体参考这个job 。为什么出这个错误？为此我把bazel run -c opt //onboard/nets/custom_ops:multiply_value_test这个扔到那个job里面。问题出现了bazel run是成功的，而bazel test是失败的。。。。有个类似的链接https://github.com/bazelbuild/rules_nodejs/issues/2325  为什么出现这个问题？因为指定了具体用不用cpu_only_flag，为了能够先跳过这个job，先

bazel run -c opt //onboard/nets/custom_ops:multiply_value_test

bazel test --cache_test_results=no -c opt --config=nolint $CPU_ONLY_PARAM --test_tag_filters=hxn  -- //...

是这两个commit  

```
# Docker 内部运行的结果
qcrafter@runner-yeb5xrzz-project-4-concurrent-0st2sc:/qcraft$ bazel run -c opt //onboard/nets/custom_ops:multiply_value_test
INFO: Invocation ID: 99fdcd7d-047b-4086-a572-722da639841f
INFO: Analyzed target //onboard/nets/custom_ops:multiply_value_test (9 packages loaded, 234 targets configured).
INFO: Found 1 target...
Target //onboard/nets/custom_ops:multiply_value_test up-to-date:
  bazel-bin/onboard/nets/custom_ops/multiply_value_test
INFO: Elapsed time: 6.002s, Critical Path: 0.36s
INFO: 196 processes: 65 remote cache hit, 130 internal, 1 processwrapper-sandbox.
INFO: Build completed successfully, 196 total actions
INFO: Build completed successfully, 196 total actions
exec ${PAGER:-/usr/bin/less} "$0" || exit 1
Executing tests from //onboard/nets/custom_ops:multiply_value_test
-----------------------------------------------------------------------------
[NVBLAS] NVBLAS_CONFIG_FILE environment variable is NOT set : relying on default config filename 'nvblas.conf'
[NVBLAS] Cannot open default config file 'nvblas.conf'
[NVBLAS] Config parsed
[NVBLAS] CPU Blas library need to be provided
Running main() from gmock_main.cc
[==========] Running 2 tests from 1 test suite.
[----------] Global test environment set-up.
[----------] 2 tests from multiply_value_test
[ RUN      ] multiply_value_test.MultiplyValue
[       OK ] multiply_value_test.MultiplyValue (117 ms)
[ RUN      ] multiply_value_test.MultiplyValueHalf
[       OK ] multiply_value_test.MultiplyValueHalf (0 ms)
[----------] 2 tests from multiply_value_test (117 ms total)

[----------] Global test environment tear-down
[==========] 2 tests from 1 test suite ran. (117 ms total)
[  PASSED  ] 2 tests.

# 在CI节点里面执行job的日志
FAIL: //onboard/nets/custom_ops:multiply_value_test (see /home/qcrafter/.cache/bazel/_bazel_qcrafter/b7b2ac012bd759fe3fc931a5a52099ce/execroot/com_qcraft/bazel-out/k8-opt/testlogs/onboard/nets/custom_ops/multiply_value_test/test.log)
INFO: From Testing //onboard/nets/custom_ops:multiply_value_test:
==================== Test output for //onboard/nets/custom_ops:multiply_value_test:
[NVBLAS] No Gpu available                这个失败的原因实际上可能是编译build导致的
[NVBLAS] NVBLAS_CONFIG_FILE environment variable is NOT set : relying on default config filename 'nvblas.conf'
[NVBLAS] Cannot open default config file 'nvblas.conf'
[NVBLAS] Config parsed
[NVBLAS] CPU Blas library need to be provided
Running main() from gmock_main.cc
[==========] Running 2 tests from 1 test suite.
[----------] Global test environment set-up.
[----------] 2 tests from multiply_value_test
[ RUN      ] multiply_value_test.MultiplyValue
onboard/nets/custom_ops/multiply_value_test.cc:55: Failure
The difference between result[i] and kOutputResult[i] is 7350900951613439, which exceeds kMaxDiffFloat, where
result[i] evaluates to 7350900951613440,
kOutputResult[i] evaluates to 1, and
kMaxDiffFloat evaluates to 9.9999999747524271e-07.
onboard/nets/custom_ops/multiply_value_test.cc:55: Failure
The difference between result[i] and kOutputResult[i] is 4, which exceeds kMaxDiffFloat, where
result[i] evaluates to 3.0677225980998895e-41,
kOutputResult[i] evaluates to 4, and
kMaxDiffFloat evaluates to 9.9999999747524271e-07.
onboard/nets/custom_ops/multiply_value_test.cc:55: Failure
The difference between result[i] and kOutputResult[i] is 5.9999999997455671, which exceeds kMaxDiffFloat, where
result[i] evaluates to 2.544326138664843e-10,
kOutputResult[i] evaluates to 6, and
kMaxDiffFloat evaluates to 9.9999999747524271e-07.
onboard/nets/custom_ops/multiply_value_test.cc:55: Failure
The difference between result[i] and kOutputResult[i] is 4, which exceeds kMaxDiffFloat, where
...
kOutputResultHalf[i] evaluates to 18, and
kMaxDiffHalf evaluates to 0.00039999998989515007.
[  FAILED  ] multiply_value_test.MultiplyValueHalf (0 ms)
[----------] 2 tests from multiply_value_test (1 ms total)
[----------] Global test environment tear-down
[==========] 2 tests from 1 test suite ran. (1 ms total)
[  PASSED  ] 0 tests.
[  FAILED  ] 2 tests, listed below:
[  FAILED  ] multiply_value_test.MultiplyValue
[  FAILED  ] multiply_value_test.MultiplyValueHalf
 2 FAILED TESTS
```



### 1.7 aliyun机器怎么格式化数据盘为xfs

数据盘为/dev/vdb，默认是ext4的格式

```
#给机器打掉label，避免相关的任务/pod/daemonset被调度这个机器上

#一路umount,去掉所有正在使用vdb的挂载，执行完了还是不能做挂载或者格式化，因为有进程再用，需要重启
umount -l /dev/vdb
umount -l /var/lib/container
umount -l /var/lib/kubelet/
umount -l /var/lib/docker
#注释掉原先挂载/dev/vdb的内容，里面有很多挂载的配置，避免重启后还是会挂载
vim /etc/fstab 
#重启
shutdown -r now
#格式化创建xfs的文件系统
mkfs.xfs -f /dev/vdb
#，把下面命令写入到/etc/fstab里面，保证正确的挂载
/dev/vdb /var/lib/container xfs defaults 0 0
/var/lib/container/kubelet /var/lib/kubelet none defaults,bind 0 0
/var/lib/container/docker /var/lib/docker none defaults,bind 0 0
#重启

# 这个时候vdb已经是xfs的了，可以注册到阿里云的机器里面了
# 首先进入节点节点，选择机器后批量移除，然后选择手动添加节点，需要每一台每一台操作，在具体的机器上执行拷贝的命令


#给机器打上对应的label
```











## 结尾
唉，尴尬

![狗头的赞赏码.jpg](https://i.loli.net/2020/08/27/BFHNyfpx3EsIDUG.jpg)