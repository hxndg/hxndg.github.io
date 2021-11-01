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



编译失败，原因不明，retry即可

```
ERROR:/builds/root/qcraft/onboard/nets/custom_ops/BUILD:1203:13: error executing shell command: '/bin/bash -c nvcc -arch=sm_70 -gencode=arch=compute_70,code=sm_70 -gencode=arch=compute_75,code=sm_75 -gencode=arch=compute_80,code=sm_80 -gencode=arch=compute_86,code=sm_86 -std=c++14 --shared --c...' failed (Exit 1): bash failed:error executing command /bin/bash -c ... (remaining 35 argument(s) skipped)
Use --sandbox_debug to see verbose messages from the sandbox bash failed:error executing command /bin/bash -c ...  (remaining 35 argument(s) skipped)
Use --sandbox_debug to see verbose messages from the sandbox
Catastrophic error: cannot open source file "/tmp/tmpxft_00000002_00000000-239_arg_min.compute_75.cpp1.ii"
1 catastrophic error detected in the compilation of "onboard/nets/custom_ops/arg_min.cu"
Compilation terminated
```







push到阿里云服务器时候失败，看网络上博客有两种解决方法：1重新打tag 2 在push之前先logout再logi。但是在脚本当中没看到谁这么干的

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



提示找不到bazel，需要在gitlab ci/cd的job里面添加上image，指定使用哪个镜像。

```
Entering 'onboard/params/run_params/vehicles'
Executing "step_script" stage of the job script
00:00
$ LATEST_COMMIT_HASH="`git rev-parse HEAD`";
$ echo LATEST_COMMIT_HASH:$LATEST_COMMIT_HASH
LATEST_COMMIT_HASH:5dc12e4b4e785cf0dbc539a982ec21c74d42adea
$ echo "[Testing] Just a simple test for sim_server_grey_release"
[Testing] Just a simple test for sim_server_grey_release
$ echo $KUBE_NAMESPACE
staging
$ echo $LATEST_COMMIT_HASH
5dc12e4b4e785cf0dbc539a982ec21c74d42adea
$ echo "Build sim_server_image"
Build sim_server_image
$ scripts/build_sim_server_debug.sh
Using dev-format-20211013_1431
Using REMOTE Repo
RELEASE_IMG: qcraft-docker.qcraft.ai/qcraft/offboard/dashboard/sim_server:5dc12e4b4e785cf0dbc539a982ec21c74d42adea
RELEASE_ALIYUN_IMG:registry.qcraftai.com/qcraft/sim_server:5dc12e4b4e785cf0dbc539a982ec21c74d42adea
scripts/build_sim_server_debug.sh: line 40: bazel: command not found
```





镜像pull失败，重新pull即可

```
ERROR: An error occurred during the fetch of repository 'prod_docker_base_china':
   Traceback (most recent call last):
	File "/home/qcrafter/.cache/bazel/_bazel_qcrafter/c418f5acab2a0d971f0829ca9f48d1c3/external/io_bazel_rules_docker/container/pull.bzl", line 189, column 13, in _impl
		fail("Pull command failed: %s (%s)" % (result.stderr, " ".join([str(a) for a in args])))
Error in fail: Pull command failed: 2021/10/18 08:49:56 Running the Image Puller to pull images from a Docker Registry...
2021/10/18 08:49:57 Image pull was unsuccessful: reading image "548416963446.dkr.ecr.cn-northwest-1.amazonaws.com.cn/qcraft/prod-docker@sha256:cdf98085664529be19ca6fce0d645c97f0d4347392468e36f4ea0458e0ebe2c6": GET https://548416963446.dkr.ecr.cn-northwest-1.amazonaws.com.cn/v2/qcraft/prod-docker/manifests/sha256:cdf98085664529be19ca6fce0d645c97f0d4347392468e36f4ea0458e0ebe2c6: unsupported status code 401; body: Not Authorized
 (/home/qcrafter/.cache/bazel/_bazel_qcrafter/c418f5acab2a0d971f0829ca9f48d1c3/external/go_puller_linux_amd64/file/downloaded -directory /home/qcrafter/.cache/bazel/_bazel_qcrafter/c418f5acab2a0d971f0829ca9f48d1c3/external/prod_docker_base_china/image -os linux -os-version  -os-features  -architecture amd64 -variant  -features  -name 548416963446.dkr.ecr.cn-northwest-1.amazonaws.com.cn/qcraft/prod-docker@sha256:cdf98085664529be19ca6fce0d645c97f0d4347392468e36f4ea0458e0ebe2c6 -timeout 99999)
INFO: Repository pcl instantiated at:
  /builds/root/qcraft/WORKSPACE:155:17: in <toplevel>
  /home/qcrafter/.cache/bazel/_bazel_qcrafter/c418f5acab2a0d971f0829ca9f48d1c3/external/rules_pcl/bzl/repositories.bzl:51:16: in pcl_repositories
  /home/qcrafter/.cache/bazel/_bazel_qcrafter/c418f5acab2a0d971f0829ca9f48d1c3/external/rules_pcl/bzl/repositories.bzl:74:18: in _maybe_repo
Repository rule http_archive defined at:
  /home/qcrafter/.cache/bazel/_bazel_qcrafter/c418f5acab2a0d971f0829ca9f48d1c3/external/bazel_tools/tools/build_defs/repo/http.bzl:336:31: in <toplevel>
ERROR: /builds/root/qcraft/release/common/BUILD:13:25: //release/common:prod_docker_wrapper_china depends on @prod_docker_base_china//image:image in repository @prod_docker_base_china which failed to fetch. no such package '@prod_docker_base_china//image': Pull command failed: 2021/10/18 08:49:56 Running the Image Puller to pull images from a Docker Registry...
2021/10/18 08:49:57 Image pull was unsuccessful: reading image "548416963446.dkr.ecr.cn-northwest-1.amazonaws.com.cn/qcraft/prod-docker@sha256:cdf98085664529be19ca6fce0d645c97f0d4347392468e36f4ea0458e0ebe2c6": GET https://548416963446.dkr.ecr.cn-northwest-1.amazonaws.com.cn/v2/qcraft/prod-docker/manifests/sha256:cdf98085664529be19ca6fce0d645c97f0d4347392468e36f4ea0458e0ebe2c6: unsupported status code 401; body: Not Authorized
 (/home/qcrafter/.cache/bazel/_bazel_qcrafter/c418f5acab2a0d971f0829ca9f48d1c3/external/go_puller_linux_amd64/file/downloaded -directory /home/qcrafter/.cache/bazel/_bazel_qcrafter/c418f5acab2a0d971f0829ca9f48d1c3/external/prod_docker_base_china/image -os linux -os-version  -os-features  -architecture amd64 -variant  -features  -name 548416963446.dkr.ecr.cn-northwest-1.amazonaws.com.cn/qcraft/prod-docker@sha256:cdf98085664529be19ca6fce0d645c97f0d4347392468e36f4ea0458e0ebe2c6 -timeout 99999)
ERROR: Analysis of target '//release/offboard/dashboard:sim_server_image_cn' failed; build aborted: Analysis failed
INFO: Elapsed time: 212.063s
INFO: 0 processes.
FAILED: Build did NOT complete successfully (471 packages loaded, 31719 targets configured)
ERROR: Build failed. Not running target
FAILED: Build did NOT complete successfully (471 packages loaded, 31719 targets configured)
Cleaning up file based variables
00:00
ERROR: Job failed: command terminated with exit code 1
```





没有这个context，在gitlab deploy的过程当中实际上context在job的environment已经订好了，有一个隐含的转换关系，所以不需要再`kubectl config use-context eks-prod-hk`命令了

```
Pushing sim_server to registry. Done
All done
$ sed -i "s#$ECR_US_REGISTRY_DOCKER_REPO$ECR_US_DOCKER_IMG_PATH:live#$ECR_US_REGISTRY_DOCKER_REPO$ECR_US_DOCKER_IMG_PATH:${LATEST_COMMIT_HASH}#g" "$SIM_SERVER_BASE_YAML"
$ kubectl config use-context eks-prod-01
error: no context exists with the name: "eks-prod-01"

#另一个相似的错误
$ sed -i "s#$ECR_US_REGISTRY_DOCKER_REPO$ECR_US_DOCKER_IMG_PATH:live#$ECR_US_REGISTRY_DOCKER_REPO$ECR_US_DOCKER_IMG_PATH:${LATEST_COMMIT_HASH}#g" "$SIM_SERVER_BASE_YAML"
$ kubectl config use-context edge-cn
error: no context exists with the name: "edge-cn"
```





这个是因为写的build_sim_server_v2.sh脚本的权限不正确，没有更新为755

```shell
25842320 drwxrwxrwx 19 root root 12288 Oct 19 11:24 scripts
25842684 drwxrwxrwx 80 root root  4096 Oct 19 11:24 third_party
25843681 drwxrwxrwx  7 root root  4096 Oct 19 11:25 testlogs
/bin/bash: line 196: scripts/build_sim_server_v2.sh: Permission denied
$ if [ -z "$RUNNER_BAZEL_CACHE_ADDR" ]; then REMOTE_CACHE_ADDR="`scripts/find_remote_cache.sh`"; if [ "$REMOTE_CACHE_ADDR" != "UNKNOWN" ]; then echo "Found Remote Cache Addr:" $REMOTE_CACHE_ADDR; echo "common --remote_cache=$REMOTE_CACHE_ADDR" >> .bazelrc; fi; else echo "Found Remote Cache Addr:" $RUNNER_BAZEL_CACHE_ADDR; echo "common --remote_cache=$RUNNER_BAZEL_CACHE_ADDR" >> .bazelrc; fi; echo "common --repository_cache=/home/qcrafter/.repository_cache" >> .bazelrc; echo "common --distdir=/home/qcrafter/.distdir" >> .bazelrc;
Found Remote Cache Addr: http://bazel-cache-cn-aliyun.qcraftai.com:19099
$ scripts/build_sim_server_v2.sh
Running after_script
00:00
Running after script...
$ echo "================= CI_PIPELINE_ID $CI_PIPELINE_ID"; echo "================= CI_JOB_ID ID $CI_JOB_ID"; echo "================= CI Runner `cat /etc/host_hostname`"; echo "================= Docker Instance `hostname`"; echo "================= Current Path `pwd`"; echo "================= PULL MAP $PULL_MAP"; echo "================= CI_MERGE_REQUEST_LABELS $CI_MERGE_REQUEST_LABELS";
================= CI_PIPELINE_ID 83032
================= CI_JOB_ID ID 1812237
```





另一个问题，进行push的时候，在gitlab ci/cd job里面指定的部署目录为staging，而xxx/yyy/zzz/on_server里面的deployment.yaml文件指定kubernetes的namepsace是production。最后部署的时候namespcae会以xxx/yyy/zzz/one_server里面的部署为准。解决方法是在kubectl apply 里面加上

```shell
  scripts:
  #会部署失败，需要改成在-k前面加上-n $KUBE_NAMESPACE，
      - kubectl apply -k xxx/yyy/zzz/one_server
  environment:
    name: xxx
    url: yyy 
    kubernetes:
      namespace: staging
```



gilab 提示部署失败因为依赖其它job，但是job实际上并没有依赖什么东西，并没有依赖什么job，目前尝试两种解决方法：

+ redploy job，无法成功，依然会失败
+ 对job添加denpendencies:[]表示没有依赖

```shell

This job depends on other jobs with expired/erased artifacts: k8s-gpu-bazel-test-presubmit, g-bazel-test-presubmit-edge
Please refer to https://docs.gitlab.com/ee/ci/yaml/README.html#dependencies
```



莫名其妙的错误，两个实验方法：

+ 一种是重启
+ 另一种是deploy测试

```
^onboard/can/kvaser_usb_can/linuxcan/include/poppack.h:116:9: note: previous '#pragma pack' directive that modifies alignment is here
#pragma pack()
        ^
2 warnings generated.
[5,932 / 6,032] GUNZIP external/prod_docker_base_china/image/007.tar.gz.nogz; 28s remote-cache ... (16 actions, 13 running)
[6,093 / 6,099] JoinLayers external/prod_docker_base_china/image/image.tar; 41s remote-cache
[6,093 / 6,099] JoinLayers external/prod_docker_base_china/image/image.tar; 101s remote-cache
Running after_script
00:00
Cleaning up file based variables
00:00
ERROR: Job failed: command terminated with exit code 137
```





这个错误出现失败是因为yaml文件有错误，里面的image出现了一个错误的变量，替换就会失败

```shell
error: accumulating resources: accumulation err='accumulating resources from '../base': '/builds/root/qcraft/production/k8s/offboard/dashboard/sim_server_v2/base' must resolve to a file': recursed accumulation of path '/builds/root/qcraft/production/k8s/offboard/dashboard/sim_server_v2/base': accumulating resources: accumulation err='accumulating resources from 'deployment.yaml': yaml: line 25: mapping values are not allowed in this context': got file 'deployment.yaml', but '/builds/root/qcraft/production/k8s/offboard/dashboard/sim_server_v2/base/deployment.yaml' must be a directory to be a root
```



pod处于CrashLoopBackOff状态，错误提示为。这个错误出现的主要表现为，先编译sim_server`，再rsync拷贝进tmp，然后再依托base_image打包生成镜像，然后部署到rancher就会失败。而使用直接让bazel打包image:sim_server_cn，则没有这个问题。初步怀疑是base_imgage的问题，还需要更进一步的

```
[NVBLAS] No Gpu available
[NVBLAS] NVBLAS_CONFIG_FILE environment variable is NOT set : relying on default config filename 'nvblas.conf'
[NVBLAS] Cannot open default config file 'nvblas.conf'
[NVBLAS] Config parsed
[NVBLAS] CPU Blas library need to be provided
WARNING: Logging before InitGoogleLogging() is written to STDERR
I1022 04:57:07.596469     1 cass_util.cc:13] [CASS_ERROR]: Unable to connect to any contact points, query=session connect
F1022 04:57:07.596509     1 sim_server_main.cc:111] Check failed: qcraft::cassandra::ConnectCassSession(cluster, session) == CASS_OK 
*** Check failure stack trace: ***
*** SIGABRT received at time=1634878627 on cpu 23 ***
PC: @     0x7f5fba26bfb7  (unknown)  raise
    @     0x559002e1addf        256  absl::lts_20210324::WriteFailureInfo()
    @     0x559002e1ab28         64  absl::lts_20210324::AbslFailureSignalHandler()
    @     0x7f60238b3980  1312354464  (unknown)
    @     0x5590031d63ca        176  google::LogMessage::SendToLog()
    @     0x5590031d6658         48  google::LogMessage::Flush()
    @     0x5590031d822f         32  google::LogMessageFatal::~LogMessageFatal()
    @     0x5590024c445d        112  GetCassandraConnection()
    @     0x5590024c47f2         64  GetCassandraOptions()
    @     0x5590024c4ca9       1488  main
    @     0x7f5fba24ebf7  (unknown)  __libc_start_main
    @ 0x3ad6258d4c544155  (unknown)  (unknown)
[failure_signal_handler.cc : 334] RAW: Signal 11 raised at PC=0x7f5fba26da10 while already in AbslFailureSignalHandler()
*** SIGSEGV received at time=1634878627 on cpu 23 ***
PC: @     0x7f5fba26da10  (unknown)  abort
    @     0x559002e1addf        256  absl::lts_20210324::WriteFailureInfo()
    @     0x559002e1ab28         64  absl::lts_20210324::AbslFailureSignalHandler()
    @     0x7f60238b3980  1312354464  (unknown)
    @     0x5590031d63ca        176  google::LogMessage::SendToLog()
    @     0x5590031d6658         48  google::LogMessage::Flush()
    @     0x5590031d822f         32  google::LogMessageFatal::~LogMessageFatal()
    @     0x5590024c445d        112  GetCassandraConnection()
    @     0x5590024c47f2         64  GetCassandraOptions()
    @     0x5590024c4ca9       1488  main
    @     0x7f5fba24ebf7  (unknown)  __libc_start_main
    @ 0x3ad6258d4c544155  (unknown)  (unknown)

```





runner的docker没起来，手动rerun即可

```
WARNING: Failed to pull image with policy "": image pull failed: Back-off pulling image "registry.qcraftai.com/qcraft/qcraft-ci:dev-lightgbm-20211021_1800"
ERROR: Job failed (system failure): prepare environment: pulling image "registry.qcraftai.com/qcraft/qcraft-ci:dev-lightgbm-20211021_1800": image pull failed: Back-off pulling image "registry.qcraftai.com/qcraft/qcraft-ci:dev-lightgbm-20211021_1800". Check https://docs.gitlab.com/runner/shells/index.html#shell-profile-loading for more information
```



kubectl部署镜像失败，

```
Building sim_server image. Done
Pushing sim_server image to registry
Error parsing reference: "registry.us-west-1.aliyuncs.com/qcraft/sim_server::a4fe32e97a5747aa81bd3d0fa9983c0fc492b848" is not a valid repository/tag: invalid reference format
invalid reference format
Pushing sim_server to registry. Done
All done
$ sed -i "s#qcraft-docker.qcraft.ai/qcraft/offboard/dashboard/sim_server:live#${RELEASE_IMG}#g" "production/k8s/offboard/dashboard/sim_server_v2/base/deployment.yaml"
$ kubectl apply -n $KUBE_NAMESPACE -k $KUSTOMIZE_FOLDER
error: You must be logged in to the server (the server has asked for the client to provide credentials)
Running after_script
00:00
Running after script...
$ echo "================= CI_PIPELINE_ID $CI_PIPELINE_ID"; echo "================= CI_JOB_ID ID $CI_JOB_ID"; echo "================= CI Runner `cat /etc/host_hostname`"; echo "================= Docker Instance `hostname`"; echo "================= Current Path `pwd`"; echo "================= PULL MAP $PULL_MAP"; echo "================= CI_MERGE_REQUEST_LABELS $CI_MERGE_REQUEST_LABELS";
================= CI_PIPELINE_ID 83861
================= CI_JOB_ID ID 1826425
================= CI Runner us-edge-05
================= Docker Instance runner-qtzubhgz-project-4-concurrent-15zs9p
================= Current Path /builds/root/qcraft
================= PULL MAP true
================= CI_MERGE_REQUEST_LABELS 
$ scripts/ci_cleanup.sh
Cleaning up file based variables
```



bazel cc_image当中向编译多个binary文件，使用`data=[]`文件不会被打包，使用`deps=[]`会报错，

```
        fail("kwarg does nothing when binary is specified", "deps")
```



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







job一直在pending，任务很忙，一直拿不到？

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







gitconfig权限错误，导致runner环境不能正确的找到具体运行的环境

```
$ echo ==================; # collapsed multi-line command
==================
aliyun-edge-2080ti-03
runner-hsijyn14-project-4-concurrent-6s6kt6
==================
found refs/pipelines/85227
Fetching changes with git depth set to 50...
Initialized empty Git repository in /builds/root/qcraft/.git/
Created fresh repository.
Checking out 8b841d3d as refs/merge-requests/12046/head...
Updating/initializing submodules recursively with git depth set to 50...
Submodule 'dev-env' (https://gitlab-ci-token:[MASKED]@gitlab-cn.qcraftai.com/qcraft-eng/dev-env.git) registered for path 'dev-env'
Submodule 'onboard/control/audio/data' (https://gitlab-ci-token:[MASKED]@gitlab-cn.qcraftai.com/qcraft-all/qcraft-audios.git) registered for path 'onboard/hmi/audio/data'
Submodule 'onboard/params/run_params/vehicles' (https://gitlab-ci-token:[MASKED]@gitlab-cn.qcraftai.com/qcraft-all/vehicles.git) registered for path 'onboard/params/run_params/vehicles'
Cloning into '/builds/root/qcraft/dev-env'...
Cloning into '/builds/root/qcraft/onboard/hmi/audio/data'...
remote: The project you were looking for could not be found or you don't have permission to view it.
fatal: repository 'https://gitlab-cn.qcraftai.com/qcraft-all/qcraft-audios.git/' not found
fatal: clone of 'https://gitlab-ci-token:[MASKED]@gitlab-cn.qcraftai.com/qcraft-all/qcraft-audios.git' into submodule path '/builds/root/qcraft/onboard/hmi/audio/data' failed
Failed to clone 'onboard/hmi/audio/data'. Retry scheduled
Cloning into '/builds/root/qcraft/onboard/params/run_params/vehicles'...
remote: The project you were looking for could not be found or you don't have permission to view it.
fatal: repository 'https://gitlab-cn.qcraftai.com/qcraft-all/vehicles.git/' not found
fatal: clone of 'https://gitlab-ci-token:[MASKED]@gitlab-cn.qcraftai.com/qcraft-all/vehicles.git' into submodule path '/builds/root/qcraft/onboard/params/run_params/vehicles' failed
Failed to clone 'onboard/params/run_params/vehicles'. Retry scheduled
Cloning into '/builds/root/qcraft/onboard/hmi/audio/data'...
remote: The project you were looking for could not be found or you don't have permission to view it.
fatal: repository 'https://gitlab-cn.qcraftai.com/qcraft-all/qcraft-audios.git/' not found
fatal: clone of 'https://gitlab-ci-token:[MASKED]@gitlab-cn.qcraftai.com/qcraft-all/qcraft-audios.git' into submodule path '/builds/root/qcraft/onboard/hmi/audio/data' failed
Failed to clone 'onboard/hmi/audio/data' a second time, aborting
Uploading artifacts for failed job
01:29
Uploading artifacts...
WARNING: ./testlogs/**/test.xml: no matching files 
ERROR: No files to upload                          
Uploading artifacts...
WARNING: ./testlogs/**/test.xml: no matching files 
ERROR: No files to upload                          
Cleaning up file based variables
00:04
ERROR: Job failed: command terminated with exit code 1
```







```
CHINA MODE true
Cloning into '/builds/root/qcraft/onboard/hmi/audio/data'...
remote: The project you were looking for could not be found or you don't have permission to view it.
fatal: repository 'https://gitlab-ci-token:[MASKED]@gitlab-cn.qcraftai.com/qcraft-all/qcraft-audios.git/' not found
fatal: clone of 'https://gitlab-ci-token:[MASKED]@gitlab-cn.qcraftai.com/qcraft-all/qcraft-audios.git' into submodule path '/builds/root/qcraft/onboard/hmi/audio/data' failed
Failed to clone 'onboard/hmi/audio/data'. Retry scheduled
Cloning into '/builds/root/qcraft/onboard/params/run_params/vehicles'...
remote: The project you were looking for could not be found or you don't have permission to view it.
fatal: repository 'https://gitlab-ci-token:[MASKED]@gitlab-cn.qcraftai.com/qcraft-all/vehicles.git/' not found
fatal: clone of 'https://gitlab-ci-token:[MASKED]@gitlab-cn.qcraftai.com/qcraft-all/vehicles.git' into submodule path '/builds/root/qcraft/onboard/params/run_params/vehicles' failed
Failed to clone 'onboard/params/run_params/vehicles'. Retry scheduled
Cloning into '/builds/root/qcraft/onboard/hmi/audio/data'...
remote: The project you were looking for could not be found or you don't have permission to view it.
fatal: repository 'https://gitlab-ci-token:[MASKED]@gitlab-cn.qcraftai.com/qcraft-all/qcraft-audios.git/' not found
fatal: clone of 'https://gitlab-ci-token:[MASKED]@gitlab-cn.qcraftai.com/qcraft-all/qcraft-audios.git' into submodule path '/builds/root/qcraft/onboard/hmi/audio/data' failed
Failed to clone 'onboard/hmi/audio/data' a second time, aborting
Running after_script
00:00
Running after script...
$ rsync -a --prune-empty-dirs --include '*/' --include 'test.xml' --exclude '*' bazel-out/k8-opt/testlogs .
rsync: change_dir "/builds/root/qcraft//bazel-out/k8-opt" failed: No such file or directory (2)
rsync error: some files/attrs were not transferred (see previous errors) (code 23) at main.c(1196) [sender=3.1.2]
Cleaning up file based variables
```



美国挂载的某块盘没有挂载上，所以文件找不到。里面的文件路径不对，多一个/是无所谓的

```
$ if [ -z "$RUNNER_BAZEL_CACHE_ADDR" ]; then REMOTE_CACHE_ADDR="`scripts/find_remote_cache.sh`"; if [ "$REMOTE_CACHE_ADDR" != "UNKNOWN" ]; then echo "Found Remote Cache Addr:" $REMOTE_CACHE_ADDR; echo "common --remote_cache=$REMOTE_CACHE_ADDR" >> .bazelrc; fi; else echo "Found Remote Cache Addr:" $RUNNER_BAZEL_CACHE_ADDR; echo "common --remote_cache=$RUNNER_BAZEL_CACHE_ADDR" >> .bazelrc; fi; echo "common --repository_cache=/home/qcrafter/.repository_cache" >> .bazelrc; echo "common --distdir=/home/qcrafter/.distdir" >> .bazelrc;
Found Remote Cache Addr: http://bazel-cache-us-local.qcraftai.com:19099
$ CHINA_MODE="`scripts/find_china_mode.sh`"; echo "CHINA_MODE:$CHINA_MODE"; if [ "$CHINA_MODE" != "true" ] && [ "$PULL_MAP" == true ]; then echo "initing us map"; scripts/ci_init_us_map.sh; fi;
CHINA_MODE:false
initing us map
Init US Map Data..
tar (child): /media/users/qcraft-maps-global-lfsed.tar.gz: Cannot open: No such file or directory
tar (child): Error is not recoverable: exiting now
tar: Child returned status 2
tar: Error is not recoverable: exiting now
Running after_script
00:01
Running after script...
$ rsync -a --prune-empty-dirs --include '*/' --include 'test.xml' --exclude '*' bazel-out/k8-opt/testlogs .
rsync: change_dir "/builds/root/qcraft//bazel-out/k8-opt" failed: No such file or directory (2)
rsync error: some files/attrs were not transferred (see previous errors) (code 23) at main.c(1196) [sender=3.1.2]
Uploading artifacts for failed job
00:02
WARNING: ./testlogs/**/test.xml: no matching files 
ERROR: No files to upload                          
Uploading artifacts...
Uploading artifacts...
WARNING: ./testlogs/**/test.xml: no matching files 
ERROR: No files to upload                          
Cleaning up file based variables
00:01
ERROR: Job failed: command terminated with exit code 1
```



q9999机器新拿到的镜像不应该连接gpu库，但是却链接了gpu库，所以报错了。但是为什么出现这个错误的原因目前还不确定

```shell
chuan@zhangchuan-pc:~/Workspace/qcraft4$ argo logs -f b2a26648-b2b4-4095-8f97-0734a8c6d994
b2a26648-b2b4-4095-8f97-0734a8c6d994-2621466868: offboard/simulation/scenario_test_parallel: error while loading shared libraries: /usr/lib/x86_64-linux-gnu/libcuda.so.1: file too short
b2a26648-b2b4-4095-8f97-0734a8c6d994-473490351: offboard/simulation/scenario_test_parallel: error while loading shared libraries: /usr/lib/x86_64-linux-gnu/libcuda.so.1: file too short
```



```
8d8e61aa0ac3: Layer already exists
6babb56be259: Layer already exists
81e12139262c: Pushed
66d28a25-bdd6-4455-b719-591b6f18eaea: digest: sha256:28f5405ecb0b3f9f04a6d9d79f31796d2758e045bffb3028c618bff7fc949fb0 size: 10440
push image took 0m:38s
I1029 08:01:57.504375   783 argo_launcher.cc:318] build and push image takes 190 seconds
I1029 08:01:57.504688   204 argo_launcher.cc:392] Submitting workflow 373128bc-a3c3-41bd-a0e8-ea05677bebfa
I1029 08:01:59.021728   204 launcher.cc:21] Job status will be uploaded to http://sim-dash.qcraftai.com/results/1714940098294186112
I1029 08:02:01.201225   204 argo_launcher.cc:425] workflow status RUNNING
I1029 08:02:04.172566   204 argo_launcher.cc:425] workflow status RUNNING
I1029 08:02:09.136693   204 argo_launcher.cc:425] workflow status RUNNING
I1029 08:02:18.161056   204 argo_launcher.cc:425] workflow status RUNNING
I1029 08:02:35.220659   204 argo_launcher.cc:425] workflow status RUNNING
I1029 08:03:08.316159   204 argo_launcher.cc:425] workflow status RUNNING
I1029 08:04:08.317135   204 argo_launcher.cc:425] workflow status RUNNING
I1029 08:05:09.683758   204 argo_launcher.cc:425] workflow status TERMINATED
I1029 08:05:09.683840   204 argo_launcher.cc:364] Workflow finished, cleaning up resources
W1029 08:06:10.181689   204 retry.h:21] 14 Socket closed retrying
I1029 08:06:11.220700   204 job.cc:107] Job failed, result uploaded to http://sim-dash.qcraftai.com/results/1714940098294186112
I1029 08:06:11.220736   204 job.cc:143] SMTP client not set, skip sending notification
Running after_script
00:00
Running after script...
$ rsync -a --prune-empty-dirs --include '*/' --include 'test.xml' --exclude '*' bazel-out/k8-opt/testlogs .
$ echo "================= CI_PIPELINE_ID $CI_PIPELINE_ID"; echo "================= CI_JOB_ID ID $CI_JOB_ID"; echo "================= CI Runner `cat /etc/host_hostname`"; echo "================= Docker Instance `hostname`"; echo "================= Current Path `pwd`"; echo "================= PULL MAP $PULL_MAP"; echo "================= CI_MERGE_REQUEST_LABELS $CI_MERGE_REQUEST_LABELS";
================= CI_PIPELINE_ID 85507
================= CI_JOB_ID ID 1855037
================= CI Runner qcraft-ci-03
================= Docker Instance runner-bcwge4ow-project-4-concurrent-5nwvdr
================= Current Path /builds/root/qcraft
================= PULL MAP false
================= CI_MERGE_REQUEST_LABELS 
$ scripts/ci_cleanup.sh
Cleaning up file based variables
00:00
ERROR: Job failed: command terminated with exit code 1
```









```
	Unschedulable: "0/6 nodes are available: 1 Insufficient pods, 1 node(s) were unschedulable, 4 node(s) had taints that the pod didn't tolerate."
Waiting for pod gitlab-runner/runner-bcwge4ow-project-4-concurrent-0gglgq to be running, status is Pending
	Unschedulable: "0/6 nodes are available: 1 Insufficient pods, 1 node(s) were unschedulable, 4 node(s) had taints that the pod didn't tolerate."
Waiting for pod gitlab-runner/runner-bcwge4ow-project-4-concurrent-0gglgq to be running, status is Pending
	Unschedulable: "0/6 nodes are available: 1 Insufficient pods, 1 node(s) were unschedulable, 4 node(s) had taints that the pod didn't tolerate."
Waiting for pod gitlab-runner/runner-bcwge4ow-project-4-concurrent-0gglgq to be running, status is Pending
	Unschedulable: "0/6 nodes are available: 1 Insufficient pods, 1 node(s) were unschedulable, 4 node(s) had taints that the pod didn't tolerate."
Waiting for pod gitlab-runner/runner-bcwge4ow-project-4-concurrent-0gglgq to be running, status is Pending
	Unschedulable: "0/6 nodes are available: 1 Insufficient pods, 1 node(s) were unschedulable, 4 node(s) had taints that the pod didn't tolerate."
Waiting for pod gitlab-runner/runner-bcwge4ow-project-4-concurrent-0gglgq to be running, status is Pending
	Unschedulable: "0/6 nodes are available: 1 Insufficient pods, 1 node(s) were unschedulable, 4 node(s) had taints that the pod didn't tolerate."
Waiting for pod gitlab-runner/runner-bcwge4ow-project-4-concurrent-0gglgq to be running, status is Pending
	Unschedulable: "0/6 nodes are available: 1 Insufficient pods, 1 node(s) were unschedulable, 4 node(s) had taints that the pod didn't tolerate."
Waiting for pod gitlab-runner/runner-bcwge4ow-project-4-concurrent-0gglgq to be running, status is Pending
	Unschedulable: "0/6 nodes are available: 1 Insufficient pods, 1 node(s) were unschedulable, 4 node(s) had taints that the pod didn't tolerate."
ERROR: Job failed (system failure): prepare environment: timed out waiting for pod to start. Check https://docs.gitlab.com/runner/shells/index.html#shell-profile-loading for more information
```










## 结尾
唉，尴尬

![狗头的赞赏码.jpg](https://i.loli.net/2020/08/27/BFHNyfpx3EsIDUG.jpg)