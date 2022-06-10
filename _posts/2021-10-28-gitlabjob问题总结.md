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







push到阿里云服务器时候失败，看网络上博客有两种解决方法：1重新打tag 2 在push之前先logout再logi。但是在脚本当中没看到谁这么干的，问题在于是我已经登录了啊，为什么还会出错呢？这两种解决方法的意思实际上是两种，一个是说：1没登录 2 image的名字过于长了，简单说就是本来image的名字应该是repo_name/namespace/image_name:tag，如果image_name特别长比方说是/xxx/yyy/zzz/ddd/aaa/eee/ccc这种，name就会出现push失败的情况。提示requested access to the resource is denied

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







没找到相对应的node，需要添加label？

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











```
8d8e61aa0ac3: Layer already exists
6babb56be259: Layer already exists
89e6b375-223c-47b3-ab12-2201c128911e: digest: sha256:efd6dcf9e110bc691591b95b519eadc84d68100feec36d1ac03ae0ab5877bdb2 size: 10651
push image took 0m:1s
I1102 10:53:46.537779   701 argo_launcher.cc:318] build and push image takes 26 seconds
I1102 10:53:46.537994   208 argo_launcher.cc:392] Submitting workflow a5b82252-4c03-4a2b-ba4e-f50bb4af1f4a
I1102 10:53:47.969311   208 launcher.cc:21] Job status will be uploaded to http://sim-dash.qcraftai.com/results/1715313468243280512
I1102 10:53:50.110766   208 argo_launcher.cc:425] workflow status RUNNING
I1102 10:53:53.089411   208 argo_launcher.cc:425] workflow status RUNNING
I1102 10:53:58.077638   208 argo_launcher.cc:425] workflow status RUNNING
I1102 10:54:07.104588   208 argo_launcher.cc:425] workflow status RUNNING
I1102 10:54:24.083097   208 argo_launcher.cc:425] workflow status RUNNING
I1102 10:54:57.069236   208 argo_launcher.cc:425] workflow status TERMINATED
I1102 10:54:57.069286   208 argo_launcher.cc:364] Workflow finished, cleaning up resources
I1102 10:54:58.244271   208 job.cc:107] Job failed, result uploaded to http://sim-dash.qcraftai.com/results/1715313468243280512
I1102 10:54:58.244294   208 job.cc:143] SMTP client not set, skip sending notification
```





镜像拉不下来，镜像确实存在。那么是什么问题呢？有一个问题是这里面的imagepullsecret为name: aliyun。但是实际上没有这个secret，没有这个secret不会报错。而且需要添加的secret应该为docker register，这样子才能拉下来镜像。拉不下来镜像才会报下面这个错误。我添加镜像的方法错了。

```
Events:
  Type     Reason     Age                     From               Message
  ----     ------     ----                    ----               -------
  Normal   Scheduled  9m26s                   default-scheduler  Successfully assigned staging/sim-dash-6fd5b6bc99-2456v to cn-zhangjiakou.172.20.1.234
  Normal   Pulling    7m54s (x4 over 9m25s)   kubelet            Pulling image "registry.qcraftai.com/qcraft/offboard/dashboard/sim_dash:cn-edge-f0cf6de5726d84b11939da2a70a3d3c878ac4c85"
  Warning  Failed     7m54s (x4 over 9m25s)   kubelet            Failed to pull image "registry.qcraftai.com/qcraft/offboard/dashboard/sim_dash:cn-edge-f0cf6de5726d84b11939da2a70a3d3c878ac4c85": rpc error: code = Unknown desc = Error response from daemon: pull access denied for registry.qcraftai.com/qcraft/offboard/dashboard/sim_dash, repository does not exist or may require 'docker login': denied: requested access to the resource is denied
  Warning  Failed     7m54s (x4 over 9m25s)   kubelet            Error: ErrImagePull
  Normal   BackOff    7m40s (x6 over 9m25s)   kubelet            Back-off pulling image "registry.qcraftai.com/qcraft/offboard/dashboard/sim_dash:cn-edge-f0cf6de5726d84b11939da2a70a3d3c878ac4c85"
  Warning  Failed     4m16s (x21 over 9m25s)  kubelet            Error: ImagePullBackOff

```



Build image的时候 base镜像被evict出去的原因，等换了CI机器估计也就好了,

```
[NVBLAS] No Gpu available
[NVBLAS] NVBLAS_CONFIG_FILE environment variable is NOT set : relying on default config filename 'nvblas.conf'
[NVBLAS] Cannot open default config file 'nvblas.conf'
[NVBLAS] Config parsed
[NVBLAS] CPU Blas library need to be provided
WARNING: Logging before InitGoogleLogging() is written to STDERR
I1105 03:10:58.178120   423 argo_launcher.cc:259] binary_path: offboard/simulation/scenario_test_parallel
I1105 03:10:58.178198   423 task.cc:17] Job server eks-prod-sim-server.qcraft.ai:3401
I1105 03:10:59.739009   423 job.cc:49] Job result will be uploaded to results/1715556170522852672
Using base image: 548416963446.dkr.ecr.cn-northwest-1.amazonaws.com.cn/qcraft-dev-prebuild-env:dev-venv-20211101_2206-with-res
building image
.
dev-venv-20211101_2206-with-res: Pulling from qcraft-dev-prebuild-env
Digest: sha256:bbc84d5fe9e21d6efa6cae522f669b4d743136add46f3d92d24f5792e153aa71
Status: Image is up to date for 548416963446.dkr.ecr.cn-northwest-1.amazonaws.com.cn/qcraft-dev-prebuild-env:dev-venv-20211101_2206-with-res
548416963446.dkr.ecr.cn-northwest-1.amazonaws.com.cn/qcraft-dev-prebuild-env:dev-venv-20211101_2206-with-res
pull image took 0m:0s
Step 1/11 : ARG IMG
Step 2/11 : FROM ${IMG} as diff_files
 ---> db5e29603812
Step 3/11 : SHELL ["/bin/bash", "-c"]
 ---> Using cache
 ---> 95de7e76256c
Step 4/11 : COPY tmp  /qcraft-bins-goal
failed to export image: failed to set parent sha256:95de7e76256c12ee3332d726b03082afd76133f2d2cb2ab8b29b4c124cb6abe6: unknown parent image ID sha256:95de7e76256c12ee3332d726b03082afd76133f2d2cb2ab8b29b4c124cb6abe6
I1105 03:13:17.911182   916 argo_launcher.cc:318] build and push image takes 138 seconds
I1105 03:13:17.911468   423 argo_launcher.cc:364] Workflow finished, cleaning up resources
I1105 03:13:18.910075   423 launcher.cc:21] Job status will be uploaded to http://sim-dash.qcraftai.com/results/1715556170522852672
Running after_script
```





同样的配置文件，为什么edge-test集群里面多了一堆managedFields？而staging没有，初次之外的配置都差不多?因为edge-test是老版本的kubernetes，因此里面多了一堆managedFields字段

```
```



诸如affinity，nodeAffinity这些在production和staging当中并不一样。producition里面有而我们默认的配置里面没有，用哪个？上面的代码是edge-cn，下面的是edge-us的区别。这些是正常的现象，我们文件当中的配置没变

```
onboard/mock/mock_self_destruct_module.cc:160:10: warning: 'SHM_UNUSED' is deprecated [-Wdeprecated-declarations]
    case SHM_UNUSED:
         ^
bazel-out/k8-opt/bin/onboard/lite/proto/shm_message.pb.h:81:14: note: 'SHM_UNUSED' has been explicitly marked deprecated here
  SHM_UNUSED PROTOBUF_DEPRECATED_ENUM = 1
             ^
external/com_google_protobuf/src/google/protobuf/port_def.inc:320:50: note: expanded from macro 'PROTOBUF_DEPRECATED_ENUM'
# define PROTOBUF_DEPRECATED_ENUM __attribute__((deprecated))
                                                 ^
1 warning generated.
[5,878 / 5,885] JoinLayers external/prod_docker_base_china/image/image.tar; 29s remote-cache ... (2 actions, 1 running)
[5,880 / 5,885] Action release/common/prod_docker_wrapper_china_commit.tar; 41s remote-cache
INFO: From Action release/common/prod_docker_wrapper_china_commit.tar:
Loaded image: bazel/image:image
sha256:b710a0b263a8155c796728af606319e2ad79fcada7f3ed191a0ebf2e6801ab10
079c2726d234936c9d0c758d8dd27f3f1368b90510607f331ba2a0219ba5c3f1
Target //release/offboard/dashboard:sim_server_image_cn up-to-date:
  bazel-bin/release/offboard/dashboard/sim_server_image_cn-layer.tar
INFO: Elapsed time: 644.597s, Critical Path: 262.42s
INFO: 5885 processes: 5128 remote cache hit, 757 internal.
INFO: Build completed successfully, 5885 total actions
INFO: Running command line: bazel-bin/release/offboard/dashboard/sim_server_image_cn.executable --norun
INFO: Build completed successfully, 5885 total actions
Loading legacy tarball base com_qcraft/release/common/prod_docker_wrapper_china_commit.tar... //就在这里出现了错误 应该loaded image,但是直接出错了
Running after_script
00:00
Cleaning up file based variables
00:00
ERROR: Job failed: pod "runner-p-kgzmph-project-4-concurrent-2lzwhb" status is "Failed"
```

```
2 warnings generated.
[5,879 / 5,885] JoinLayers external/prod_docker_base_china/image/image.tar; 72s remote-cache
[5,880 / 5,885] Action release/common/prod_docker_wrapper_china_commit.tar; 41s remote-cache
INFO: From Action release/common/prod_docker_wrapper_china_commit.tar:
Loaded image: bazel/image:image
sha256:b710a0b263a8155c796728af606319e2ad79fcada7f3ed191a0ebf2e6801ab10
079c2726d234936c9d0c758d8dd27f3f1368b90510607f331ba2a0219ba5c3f1
Target //release/offboard/dashboard:sim_server_image_cn up-to-date:
  bazel-bin/release/offboard/dashboard/sim_server_image_cn-layer.tar
INFO: Elapsed time: 745.764s, Critical Path: 307.29s
INFO: 5885 processes: 5128 remote cache hit, 757 internal.
INFO: Build completed successfully, 5885 total actions
INFO: Running command line: bazel-bin/release/offboard/dashboard/sim_server_image_cn.executable --norun
INFO: Build completed successfully, 5885 total actions
Loading legacy tarball base com_qcraft/release/common/prod_docker_wrapper_china_commit.tar...
Loaded image: bazel/release/common:prod_docker_wrapper_china
Loaded image ID: sha256:4f740c2a7e010cc4292efc26c903e32c473ef35d78edec591060d302b5a40f9d
Tagging 4f740c2a7e010cc4292efc26c903e32c473ef35d78edec591060d302b5a40f9d as bazel/release/offboard/dashboard:sim_server_image_cn
Building sim_server image. Done
Pushing sim_server image to registry
The push refers to repository [qcraft-docker.qcraft.ai/qcraft/sim_server]
```



今天遇到一个这个错误，问题来了，为什么出现这个问题？libcuinj64这个库是dev-docker编译的

```
/app/offboard/dashboard/sim_server: error while loading shared libraries: libcuinj64.so.11.3: cannot open shared object file: No such file or directory
```





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



job不断的尝试重连，目前遇到的情况很多，一种是我将sidecar tracing的镜像改成了registry了，导致拉不下来，然后就一直重试。

```
I1124 04:14:12.669909   418 launcher.cc:42] Job status RUNNING
W1124 04:15:12.670430   418 retry.h:21] 14 Socket closed retrying
I1124 04:15:13.658504   418 launcher.cc:42] Job status RUNNING
W1124 04:16:13.659106   418 retry.h:21] 14 Socket closed retrying
I1124 04:16:14.658265   418 launcher.cc:42] Job status RUNNING
W1124 04:17:14.658864   418 retry.h:21] 14 Socket closed retrying
I1124 04:17:15.652274   418 launcher.cc:42] Job status RUNNING
W1124 04:18:15.652964   418 retry.h:21] 14 Socket closed retrying
I1124 04:18:16.692870   418 launcher.cc:42] Job status RUNNING
W1124 04:19:16.693482   418 retry.h:21] 14 Socket closed retrying
I1124 04:19:17.744310   418 launcher.cc:42] Job status RUNNING
W1124 04:20:17.744796   418 retry.h:21] 14 Socket closed retrying
I1124 04:20:18.714323   418 launcher.cc:42] Job status RUNNING
W1124 04:21:18.714792   418 retry.h:21] 14 Socket closed retrying
I1124 04:21:19.767874   418 launcher.cc:42] Job status RUNNING
W1124 04:22:19.768371   418 retry.h:21] 14 Socket closed retrying
I1124 04:22:20.755031   418 launcher.cc:42] Job status RUNNING
I1124 04:23:21.537782   418 launcher.cc:42] Job status RUNNING
W1124 04:24:21.538275   418 retry.h:21] 14 Socket closed retrying
I1124 04:24:22.519601   418 launcher.cc:42] Job status RUNNING
W1124 04:25:22.520140   418 retry.h:21] 14 Socket closed retrying
I1124 04:25:23.501000   418 launcher.cc:42] Job status RUNNING
W1124 04:26:23.501471   418 retry.h:21] 14 Socket closed retrying
I1124 04:26:24.581113   418 launcher.cc:42] Job status RUNNING
W1124 04:27:24.581622   418 retry.h:21] 14 Socket closed retrying
I1124 04:27:25.560441   418 launcher.cc:42] Job status RUNNING
```

安州牧：

信息素：战术  

渤海小吏：刘秀 & 王莽 不得不说的故事





+ scenarios test的内存溢出用valgand分析原因
+ jiannan的mr把几个场景测试合并进了bazel test









+ qsim-frontend, 语言为node，
+ yaml在哪里？线上在edge-test，production，qsim-dash



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





aws, kube, map-secret

安装了一个gitlabrunner以后，运行提示pending，没有runner，后来改了下两个NodeSelector的label解决了问题。

```shell
This job is pending because no runner with tag: "k8s-aws-mount-test"
```





sleep超时问题，超时了一秒

```
bazel-out/k8-opt/bin/external/io_opentelemetry_cpp/api/_virtual_includes/api/opentelemetry/context/runtime_context.h:25:3: warning: explicitly defaulted default constructor is implicitly deleted [-Wdefaulted-function-deleted]
  Token() noexcept = default;
  ^
bazel-out/k8-opt/bin/external/io_opentelemetry_cpp/api/_virtual_includes/api/opentelemetry/context/runtime_context.h:31:17: note: default constructor of 'Token' is implicitly deleted because field 'context_' of const-qualified type 'const opentelemetry::context::Context' would not be initialized
  const Context context_;
                ^
1 warning generated.
INFO: From Compiling onboard/qlfs/tests/filesystem/file_transfer/local_file_transfer_test.cc:
onboard/qlfs/tests/filesystem/file_transfer/local_file_transfer_test.cc:22:5: warning: ignoring return value of function declared with 'warn_unused_result' attribute [-Wunused-result]
    system(cmd.c_str());
    ^~~~~~ ~~~~~~~~~~~
1 warning generated.
FAIL: //onboard/global:logging_test (see /home/qcrafter/.cache/bazel/_bazel_qcrafter/b0b45119ef6e2a7c8a9693a61600d065/execroot/com_qcraft/bazel-out/k8-opt/testlogs/onboard/global/logging_test/test.log)
INFO: From Testing //onboard/global:logging_test:
==================== Test output for //onboard/global:logging_test:
Running main() from gmock_main.cc
[==========] Running 1 test from 1 test suite.
[----------] Global test environment set-up.
[----------] 1 test from LoggingTest
[ RUN      ] LoggingTest.TestErrorWithOnboard
onboard/global/logging_test.cc:29: Failure
Expected equality of these values:
  a
    Which is: 4
  kCounter + 1
    Which is: 3
[  FAILED  ] LoggingTest.TestErrorWithOnboard (4812 ms)
[----------] 1 test from LoggingTest (4812 ms total)
[----------] Global test environment tear-down
[==========] 1 test from 1 test suite ran. (4812 ms total)
[  PASSED  ] 0 tests.
[  FAILED  ] 1 test, listed below:
[  FAILED  ] LoggingTest.TestErrorWithOnboard
 1 FAILED TEST
================================================================================
INFO: From Compiling onboard/global/ftrace_test.cc:
onboard/global/ftrace_test.cc:71:33: warning: nested designators are a C99 extension [-Wc99-designator]
                                .args[0].key = "key_1",
                                ^~~~~~~~~~~~
onboard/global/ftrace_test.cc:71:38: warning: array designators are a C99 extension [-Wc99-designator]
                                .args[0].key = "key_1",
```



直接就出错然后挂掉了。这个啥情况？看报错信息说是用的过多存储空间，但理论上应该没用，最后直接abort了。

```
In file included from bazel-out/k8-opt/bin/external/io_opentelemetry_cpp/sdk/_virtual_includes/headers/opentelemetry/sdk/trace/tracer_provider.h:16:
In file included from bazel-out/k8-opt/bin/external/io_opentelemetry_cpp/sdk/_virtual_includes/headers/opentelemetry/sdk/trace/tracer.h:12:
In file included from bazel-out/k8-opt/bin/external/io_opentelemetry_cpp/api/_virtual_includes/api/opentelemetry/trace/noop.h:10:
bazel-out/k8-opt/bin/external/io_opentelemetry_cpp/api/_virtual_includes/api/opentelemetry/context/runtime_context.h:25:3: warning: explicitly defaulted default constructor is implicitly deleted [-Wdefaulted-function-deleted]
  Token() noexcept = default;
  ^
bazel-out/k8-opt/bin/external/io_opentelemetry_cpp/api/_virtual_includes/api/opentelemetry/context/runtime_context.h:31:17: note: default constructor of 'Token' is implicitly deleted because field 'context_' of const-qualified type 'const opentelemetry::context::Context' would not be initialized
  const Context context_;
                ^
1 warning generated.
[6,042 / 6,053] Compiling offboard/simulation/simulator_main.cc; 43s remote-cache, linux-sandbox ... (4 actions, 3 running)
[6,047 / 6,053] JoinLayers external/prod_docker_base_china/image/image.tar; 74s linux-sandbox
Running after_script
00:00
Cleaning up file based variables
00:00
ERROR: Job failed: pod "runner-wa4gnvzx-project-4-concurrent-18qqgm" status is "Failed"
```





IO出了问题，重启就好了

```
INFO: Repository rules_folly instantiated at:
  /builds/uxKdbFbb/0/root/qcraft/WORKSPACE:5:20: in <toplevel>
  /builds/uxKdbFbb/0/root/qcraft/bazel/workspace.bzl:178:33: in qcraft_repositories
  /builds/uxKdbFbb/0/root/qcraft/bazel/workspace.bzl:162:16: in initialize_third_party_repos
  /builds/uxKdbFbb/0/root/qcraft/third_party/rules_folly/workspace.bzl:9:17: in repo
Repository rule http_archive defined at:
  /home/qcrafter/.cache/bazel/_bazel_qcrafter/308b7d803ead1e4500ecbe2758e610a4/external/bazel_tools/tools/build_defs/repo/http.bzl:336:31: in <toplevel>
ERROR: An error occurred during the fetch of repository 'rules_folly':
   Traceback (most recent call last):
	File "/home/qcrafter/.cache/bazel/_bazel_qcrafter/308b7d803ead1e4500ecbe2758e610a4/external/bazel_tools/tools/build_defs/repo/http.bzl", line 111, column 45, in _http_archive_impl
		download_info = ctx.download_and_extract(
Error in download_and_extract: java.io.IOException: /home/qcrafter/.repository_cache/content_addressable/sha256/317abac1c970ad0af43c88b6eac706c9b4c5a06ee8b673d0e352143b0d9fd481/tmp-a1565851-2790-4802-97a2-5793118e6d84 (Input/output error)
ERROR: no such package '@rules_folly//bazel': java.io.IOException: /home/qcrafter/.repository_cache/content_addressable/sha256/317abac1c970ad0af43c88b6eac706c9b4c5a06ee8b673d0e352143b0d9fd481/tmp-a1565851-2790-4802-97a2-5793118e6d84 (Input/output error)
INFO: Elapsed time: 3.578s
```





ARGO SERVER出现一个>>双箭头的符号，代表没找到scenario？但是参数并不能确定到底是什么，这个得看generateScenarios(0)的日志，里面我加了日志，如果scenario为空，那么会记录一条日志





找不到qcraft1233333.com，一个已经被删除的域名还在使用。理论上不该出现，具体为啥还不是很清楚，直接retry

```
aliyun-edge-2080ti-03
runner-zlvbamvk-project-4-concurrent-1r85qs
==================
found refs/pipelines/97858
Fetching changes with git depth set to 50...
Initialized empty Git repository in /builds/root/qcraft/.git/
Created fresh repository.
Checking out d814070b as refs/merge-requests/14304/head...
LFS: Get https://qcraft123333.com/root/qcraft.git/gitlab-lfs/objects/806278546806acbd0b2073d84e2670ec64f6564a3cda4c6c822ee585c40857e1: dial tcp: lookup qcraft123333.com: no such host
error: failed to fetch some objects from 'https://gitlab-ci-token:[MASKED]@gitlab-cn.qcraftai.com/root/qcraft.git/info/lfs'
Uploading artifacts for failed job
00:00
Uploading artifacts...
WARNING: ./testlogs/**/test.xml: no matching files 
ERROR: No files to upload                          
Uploading artifacts...
WARNING: ./testlogs/**/test.xml: no matching files 
ERROR: No files to upload                          
Cleaning up file based variables
00:01
ERROR: Job failed: command terminated with exit code 1
```

```
LFS: Get https://qcraft123333.com/root/qcraft.git/gitlab-lfs/objects/63046290aac160196244f324f94c862d2a3fa2182771819cfc8ce9f8b6deaf3b: dial tcp: lookup qcraft123333.com: no such host
LFS: Get https://qcraft123333.com/root/qcraft.git/gitlab-lfs/objects/0ea0f2011258f36e4cb779737833530af99d5a6ca8c5c1a8e2a2aa047852ba32: dial tcp: lookup qcraft123333.com: no such host
LFS: Get https://qcraft123333.com/root/qcraft.git/gitlab-lfs/objects/05333001cd83c6f699130030b5d3260d0dd02f8559ff01a775fff79c97c28ba0: dial tcp: lookup qcraft123333.com: no such host
LFS: Get https://qcraft123333.com/root/qcraft.git/gitlab-lfs/objects/fae5f08742430c21ba9484cb5daefdac533322477fc5c7b6b4352ca133515883: dial tcp: lookup qcraft123333.com: no such host
LFS: Get https://qcraft123333.com/root/qcraft.git/gitlab-lfs/objects/290eaf80ebd29316a483bb0780dedbd7c664fccd2194bbde59fd6c7b078b9511: dial tcp: lookup qcraft123333.com: no such host
LFS: Get https://qcraft123333.com/root/qcraft.git/gitlab-lfs/objects/7ddffad5ce294bce527c1d7352fb0710373480bb526474b770be67d800492fb5: dial tcp: lookup qcraft123333.com: no such host
LFS: Get https://qcraft123333.com/root/qcraft.git/gitlab-lfs/objects/0b9cc89a112790e7a89e33e53902e23c444536cf029217079ec340478b98af9e: dial tcp: lookup qcraft123333.com: no such host
LFS: Get https://qcraft123333.com/root/qcraft.git/gitlab-lfs/objects/a8e1cf31848c708ebb256361b073640a9035c811154515a71aaa5406c1b7bd1a: dial tcp: lookup qcraft123333.com: no such host
LFS: Get https://qcraft123333.com/root/qcraft.git/gitlab-lfs/objects/7c69b92985c250d6f96049c22565d9f7e7f45f7e1a9bf0c16eccd7ab8e5a2b74: dial tcp: lookup qcraft123333.com: no such host
LFS: Get https://qcraft123333.com/root/qcraft.git/gitlab-lfs/objects/cbe93dd741a5fbb855d41d2f0890305f693682aacd7bf1960514e48b3aeb10c8: dial tcp: lookup qcraft123333.com: no such host
LFS: Get https://qcraft123333.com/root/qcraft.git/gitlab-lfs/objects/1d72320d1a88b34df2c87530ecf6b26073649454c03143687a202405793206e7: dial tcp: lookup qcraft123333.com: no such host
LFS: Get https://qcraft123333.com/root/qcraft.git/gitlab-lfs/objects/d625b2b66076e8df103d17cfc6320a15f567790bc512f0e5ab2beb0ff18da205: dial tcp: lookup qcraft123333.com: no such host
LFS: Get https://qcraft123333.com/root/qcraft.git/gitlab-lfs/objects/e8e3dd801debe525125724ca12099c22983055f1bf039479138f2cec71841174: dial tcp: lookup qcraft123333.com: no such host
LFS: Get https://qcraft123333.com/root/qcraft.git/gitlab-lfs/objects/fd694915977379c5d01ba2484ce790914ba73ded32d0aecf5334ac78f70f112b: dial tcp: lookup qcraft123333.com: no such host
LFS: Get https://qcraft123333.com/root/qcraft.git/gitlab-lfs/objects/fbae1e3f10c84a488f98445683645b86350555815d4fffc8b91b1dcc31606c1a: dial tcp: lookup qcraft123333.com: no such host
error: failed to fetch some objects from 'https://gitlab-ci-token:[MASKED]@gitlab-us.qcraftai.com/root/qcraft.git/info/lfs'
```



VSCode乱码问题

```

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



要了一台ack-cn的gpu机器，然后发现虽然有驱动，但是驱动版本和cuda版本太老了。所以更新下安装流程和命令：

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



跑的job被kill了，单纯的资源不足导致的

```
                                            ^
6 warnings generated.
INFO: From Compiling offboard/tools/run_processor/run_job_composer.cc:
offboard/tools/run_processor/run_job_composer.cc:342:9: warning: ignoring return value of function declared with 'warn_unused_result' attribute [-Wunused-result]
        kafka_producer_client->Produce(
        ^~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
1 warning generated.
INFO: From Compiling offboard/tools/run_processor/run_job_monitor.cc:
offboard/tools/run_processor/run_job_monitor.cc:330:3: warning: ignoring return value of function declared with 'warn_unused_result' attribute [-Wunused-result]
  qcraft::RunDBHandler::Instance()->GetRunServersTable()->insert(
  ^~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
1 warning generated.
ERROR: /builds/7EQKjzDs/6/root/qcraft/offboard/mapping/pose_graph_mapping/BUILD:954:10: Compiling offboard/mapping/pose_graph_mapping/combine_match_results_main.cc failed: (Killed): clang failed: error executing command /opt/llvm/bin/clang '--target=x86_64-unknown-linux-gnu' -U_FORTIFY_SOURCE -fstack-protector -fno-omit-frame-pointer -fcolor-diagnostics -Wall -Wthread-safety -Wself-assign -Wdeprecated-declarations ... (remaining 300 argument(s) skipped)
Use --sandbox_debug to see verbose messages from the sandbox
INFO: Elapsed time: 551.039s, Critical Path: 113.88s
INFO: 20120 processes: 10831 remote cache hit, 9198 internal, 91 linux-sandbox.
FAILED: Build did NOT complete successfully
FAILED: Build did NOT complete successfully
Running after_script
00:01
Running after script...
$ rsync -a --prune-empty-dirs --include '*/' --include 'test.xml' --exclude '*' bazel-out/k8-opt/testlogs .
$ echo "================= CI_PIPELINE_ID $CI_PIPELINE_ID"; echo "================= CI_JOB_ID ID $CI_JOB_ID"; echo "================= CI Runner `cat /etc/host_hostname`"; echo "================= Docker Instance `hostname`"; echo "================= Current Path `pwd`"; echo "================= PULL MAP $PULL_MAP"; echo "================= CI_MERGE_REQUEST_LABELS $CI_MERGE_REQUEST_LABELS";
================= CI_PIPELINE_ID 101977
================= CI_JOB_ID ID 2212690
================= CI Runner us-edge-07
================= Docker Instance runner-7eqkjzds-project-4-concurrent-6q4298
================= Current Path /builds/7EQKjzDs/6/root/qcraft
================= PULL MAP true
================= CI_MERGE_REQUEST_LABELS 
$ scripts/ci_cleanup.sh
Cleaning up file based variables
00:01
ERROR: Job failed: command terminated with exit code 1

```

```
[qcraft@qcraft-dev-qcraft:/qcraft(master) ] $ kubectl get event -n gitlab-runner | grep runner-7eqkjzds-project-4-concurrent-6q4298
25m         Normal    Scheduled            pod/runner-7eqkjzds-project-4-concurrent-6q4298                      Successfully assigned gitlab-runner/runner-7eqkjzds-project-4-concurrent-6q4298 to us-edge-07
25m         Normal    Pulled               pod/runner-7eqkjzds-project-4-concurrent-6q4298                      Container image "registry.gitlab.com/gitlab-org/gitlab-runner/gitlab-runner-helper:x86_64-58ba2b95" already present on machine
25m         Normal    Created              pod/runner-7eqkjzds-project-4-concurrent-6q4298                      Created container init-permissions
25m         Normal    Started              pod/runner-7eqkjzds-project-4-concurrent-6q4298                      Started container init-permissions
25m         Normal    Pulled               pod/runner-7eqkjzds-project-4-concurrent-6q4298                      Container image "registry.qcraftai.com/global/qcraft-ci:dev-libgit-20211214_0012" already present on machine
25m         Normal    Created              pod/runner-7eqkjzds-project-4-concurrent-6q4298                      Created container build
25m         Normal    Started              pod/runner-7eqkjzds-project-4-concurrent-6q4298                      Started container build
25m         Normal    Pulled               pod/runner-7eqkjzds-project-4-concurrent-6q4298                      Container image "registry.gitlab.com/gitlab-org/gitlab-runner/gitlab-runner-helper:x86_64-58ba2b95" already present on machine
25m         Normal    Created              pod/runner-7eqkjzds-project-4-concurrent-6q4298                      Created container helper
25m         Normal    Started              pod/runner-7eqkjzds-project-4-concurrent-6q4298                      Started container helper
14m         Normal    Killing              pod/runner-7eqkjzds-project-4-concurrent-6q4298                      Stopping container build
14m         Normal    Killing              pod/runner-7eqkjzds-project-4-concurrent-6q4298                      Stopping container helper
```



build farm挂了，重试不行

```
1 warning generated.
INFO: From Compiling offboard/tools/s3_client.cc:
offboard/tools/s3_client.cc:419:5: warning: ignoring return value of function declared with 'warn_unused_result' attribute [-Wunused-result]
    SetUploadId(s3_file_key, upload_id);
    ^~~~~~~~~~~ ~~~~~~~~~~~~~~~~~~~~~~
1 warning generated.
ERROR: /builds/uxKdbFbb/0/root/qcraft/onboard/autonomy/BUILD:26:11: Compiling onboard/autonomy/autonomy_module.cc failed: (Exit 34): UNAVAILABLE: io exception
java.io.IOException: io.grpc.StatusRuntimeException: UNAVAILABLE: io exception
	at com.google.devtools.build.lib.remote.GrpcRemoteExecutor.executeRemotely(GrpcRemoteExecutor.java:226)
	at com.google.devtools.build.lib.remote.RemoteExecutionService.execute(RemoteExecutionService.java:471)
	at com.google.devtools.build.lib.remote.RemoteSpawnRunner.lambda$exec$2(RemoteSpawnRunner.java:251)
	at com.google.devtools.build.lib.remote.Retrier.execute(Retrier.java:244)
	at com.google.devtools.build.lib.remote.RemoteRetrier.execute(RemoteRetrier.java:125)
	at com.google.devtools.build.lib.remote.RemoteRetrier.execute(RemoteRetrier.java:114)
	at com.google.devtools.build.lib.remote.RemoteSpawnRunner.exec(RemoteSpawnRunner.java:230)
	at com.google.devtools.build.lib.exec.SpawnRunner.execAsync(SpawnRunner.java:238)
	at com.google.devtools.build.lib.exec.AbstractSpawnStrategy.exec(AbstractSpawnStrategy.java:144)
	at com.google.devtools.build.lib.exec.AbstractSpawnStrategy.exec(AbstractSpawnStrategy.java:106)
	at com.google.devtools.build.lib.actions.SpawnStrategy.beginExecution(SpawnStrategy.java:47)
	at com.google.devtools.build.lib.exec.SpawnStrategyResolver.beginExecution(SpawnStrategyResolver.java:65)
	at com.google.devtools.build.lib.rules.cpp.CppCompileAction.beginExecution(CppCompileAction.java:1451)
	at com.google.devtools.build.lib.actions.Action.execute(Action.java:127)
	at com.google.devtools.build.lib.skyframe.SkyframeActionExecutor$5.execute(SkyframeActionExecutor.java:855)
	at com.google.devtools.build.lib.skyframe.SkyframeActionExecutor$ActionRunner.continueAction(SkyframeActionExecutor.java:1016)
	at com.google.devtools.build.lib.skyframe.SkyframeActionExecutor$ActionRunner.run(SkyframeActionExecutor.java:975)
	at com.google.devtools.build.lib.skyframe.ActionExecutionState.runStateMachine(ActionExecutionState.java:129)
	at com.google.devtools.build.lib.skyframe.ActionExecutionState.getResultOrDependOnFuture(ActionExecutionState.java:81)
	at com.google.devtools.build.lib.skyframe.SkyframeActionExecutor.executeAction(SkyframeActionExecutor.java:472)
	at com.google.devtools.build.lib.skyframe.ActionExecutionFunction.checkCacheAndExecuteIfNeeded(ActionExecutionFunction.java:834)
	at com.google.devtools.build.lib.skyframe.ActionExecutionFunction.compute(ActionExecutionFunction.java:307)
	at com.google.devtools.build.skyframe.AbstractParallelEvaluator$Evaluate.run(AbstractParallelEvaluator.java:477)
	at com.google.devtools.build.lib.concurrent.AbstractQueueVisitor$WrappedRunnable.run(AbstractQueueVisitor.java:398)
	at java.base/java.util.concurrent.ThreadPoolExecutor.runWorker(Unknown Source)
	at java.base/java.util.concurrent.ThreadPoolExecutor$Worker.run(Unknown Source)
	at java.base/java.lang.Thread.run(Unknown Source)
Caused by: io.grpc.StatusRuntimeException: UNAVAILABLE: io exception
	at io.grpc.Status.asRuntimeException(Status.java:533)
	at io.grpc.stub.ClientCalls$BlockingResponseStream.hasNext(ClientCalls.java:648)
	at com.google.devtools.build.lib.remote.GrpcRemoteExecutor.lambda$executeRemotely$0(GrpcRemoteExecutor.java:160)
	at com.google.devtools.build.lib.remote.Retrier.execute(Retrier.java:244)
	at com.google.devtools.build.lib.remote.RemoteRetrier.execute(RemoteRetrier.java:125)
	at com.google.devtools.build.lib.remote.RemoteRetrier.execute(RemoteRetrier.java:114)
	at com.google.devtools.build.lib.remote.GrpcRemoteExecutor.lambda$executeRemotely$1(GrpcRemoteExecutor.java:139)
	at com.google.devtools.build.lib.remote.util.Utils.refreshIfUnauthenticated(Utils.java:494)
	at com.google.devtools.build.lib.remote.GrpcRemoteExecutor.executeRemotely(GrpcRemoteExecutor.java:137)
	... 26 more
Caused by: io.netty.channel.AbstractChannel$AnnotatedConnectException: finishConnect(..) failed: Connection refused: buildfarm.qcraftai.com/172.20.2.232:80
Caused by: java.net.ConnectException: finishConnect(..) failed: Connection refused
	at io.netty.channel.unix.Errors.throwConnectException(Errors.java:124)
	at io.netty.channel.unix.Socket.finishConnect(Socket.java:243)
	at io.netty.channel.epoll.AbstractEpollChannel$AbstractEpollUnsafe.doFinishConnect(AbstractEpollChannel.java:672)
	at io.netty.channel.epoll.AbstractEpollChannel$AbstractEpollUnsafe.finishConnect(AbstractEpollChannel.java:649)
	at io.netty.channel.epoll.AbstractEpollChannel$AbstractEpollUnsafe.epollOutReady(AbstractEpollChannel.java:529)
	at io.netty.channel.epoll.EpollEventLoop.processReady(EpollEventLoop.java:465)
	at io.netty.channel.epoll.EpollEventLoop.run(EpollEventLoop.java:378)
	at io.netty.util.concurrent.SingleThreadEventExecutor$4.run(SingleThreadEventExecutor.java:989)
	at io.netty.util.internal.ThreadExecutorMap$2.run(ThreadExecutorMap.java:74)
	at io.netty.util.concurrent.FastThreadLocalRunnable.run(FastThreadLocalRunnable.java:30)
	at java.base/java.lang.Thread.run(Unknown Source)
INFO: Elapsed time: 1191.390s, Critical Path: 244.06s
INFO: 11169 processes: 6485 remote cache hit, 4204 internal, 480 remote.
FAILED: Build did NOT complete successfully
FAILED: Build did NOT complete successfully
Running after_script
00:01
Running after script...
$ echo "================= CI_PIPELINE_ID $CI_PIPELINE_ID"; echo "================= CI_JOB_ID ID $CI_JOB_ID"; echo "================= CI Runner `cat /etc/host_hostname`"; echo "================= Docker Instance `hostname`"; echo "================= Current Path `pwd`"; echo "================= PULL MAP $PULL_MAP"; echo "================= CI_MERGE_REQUEST_LABELS $CI_MERGE_REQUEST_LABELS";
================= CI_PIPELINE_ID 102061
================= CI_JOB_ID ID 2214734
================= CI Runner cn-cpu01018ack
================= Docker Instance runner-uxkdbfbb-project-4-concurrent-0lthhh
================= Current Path /builds/uxKdbFbb/0/root/qcraft
================= PULL MAP true
================= CI_MERGE_REQUEST_LABELS 
```



workflow的址址为下面的,查了一下workflow的日志里面的18.144.103.119，是canssdra的ip地址，因此是cassdra连接不上导致的错误。那么如何解决呢?最后等了一段时间，然后自己恢复了。

```
--workflows_server=a72b3aa94114d48bd99b72a1a968d294-1716175034.us-west-2.elb.amazonaws.com:3405
```

```
I1225 11:17:43.844477   979 argo_launcher.cc:102] build and push image takes 81 seconds
I1225 11:17:43.844708   421 argo_launcher.cc:155] Workflow finished, cleaning up resources
W1225 11:17:45.246464   421 retry.h:21] 5 Failed to read ID from workflow_db.workflows. retrying
E1225 11:17:48.246567   421 argo_launcher.cc:166] Error Draining Workflow 5, Failed to read ID from workflow_db.workflows.
I1225 11:17:48.246603   421 launcher.cc:22] Job status will be uploaded to http://qsim.qcraftai.com/job/1720116551797817536
```

```
#看eks-prod-01的workflow server的状态是running，但是有一些warning日志：
1640433163.614 [WARN] (connection_pool.cpp:278:void datastax::internal::core::ConnectionPool::on_reconnect(datastax::internal::core::DelayedConnector*)): Connection pool was unable to reconnect to host 18.144.103.119 because of the following error: Connect error 'connection refused'
```



很多情况下是canssdra数据库挂了，重启或者升级（还是重启）。目前canssdra的重启没有报警，需要手动到sim-server或者workflow-server上去看日志，几个ip都在sim-server deployment文件里面有写

```
I1227 11:41:08.879084  1036 launcher.cc:43] Job status PENDING
W1227 11:42:08.879546  1036 retry.h:21] 14 Socket closed retrying
I1227 11:42:17.040661  1036 launcher.cc:43] Job status PENDING
I1227 11:43:20.174173  1036 launcher.cc:43] Job status PENDING
W1227 11:44:45.257000  1036 retry.h:21] 13 Failed to read ID from sim_db.jobs. retrying
E1227 11:44:48.257108  1036 job.cc:137] Error Getting Job 13, Failed to read ID from sim_db.jobs.
```







cas空间不足导致的问题

```
15 warnings generated.
INFO: From Executing genrule //offboard/vis/vantage/topics:tree_item_moc:
offboard/vis/vantage/topics/tree_item.h:0: Note: No relevant classes found. No output generated.
ERROR: /builds/nNsD5xAU/9/root/qcraft/offboard/simulation/prt/util/BUILD:25:10: Linking offboard/simulation/prt/util/prt_history_main failed: (Exit 34): Remote Execution Failure:
Failed Precondition: Action 5dc5e44338b97515cc06a7ddbba8873ffbb8156ec213565d807228aee084315a/144 is invalid: A requested input (or the `Action` or its `Command`) was not found in the CAS..
  Precondition Failure:
    (MISSING) blobs/a1967c1f28203bae0e65366970c9de8dc4853cc066d757c32650e447fac53283/181288: A requested input (or the `Action` or its `Command`) was not found in the CAS.
java.io.IOException: com.google.devtools.build.lib.remote.ExecutionStatusException: FAILED_PRECONDITION: Action 5dc5e44338b97515cc06a7ddbba8873ffbb8156ec213565d807228aee084315a/144 is invalid: A requested input (or the `Action` or its `Command`) was not found in the CAS..
	at com.google.devtools.build.lib.remote.GrpcRemoteExecutor.executeRemotely(GrpcRemoteExecutor.java:226)
	at com.google.devtools.build.lib.remote.RemoteExecutionService.execute(RemoteExecutionService.java:471)
	at com.google.devtools.build.lib.remote.RemoteSpawnRunner.lambda$exec$2(RemoteSpawnRunner.java:251)
	at com.google.devtools.build.lib.remote.Retrier.execute(Retrier.java:244)
	at com.google.devtools.build.lib.remote.RemoteRetrier.execute(RemoteRetrier.java:125)
	at com.google.devtools.build.lib.remote.RemoteRetrier.execute(RemoteRetrier.java:114)
	at com.google.devtools.build.lib.remote.RemoteSpawnRunner.exec(RemoteSpawnRunner.java:230)
	at com.google.devtools.build.lib.exec.SpawnRunner.execAsync(SpawnRunner.java:238)
	at com.google.devtools.build.lib.exec.AbstractSpawnStrategy.exec(AbstractSpawnStrategy.java:144)
	at com.google.devtools.build.lib.exec.AbstractSpawnStrategy.exec(AbstractSpawnStrategy.java:106)
	at com.google.devtools.build.lib.actions.SpawnStrategy.beginExecution(SpawnStrategy.java:47)
	at com.google.devtools.build.lib.exec.SpawnStrategyResolver.beginExecution(SpawnStrategyResolver.java:65)
	at com.google.devtools.build.lib.rules.cpp.CppLinkAction.beginExecution(CppLinkAction.java:306)
	at com.google.devtools.build.lib.actions.Action.execute(Action.java:127)
	at com.google.devtools.build.lib.skyframe.SkyframeActionExecutor$5.execute(SkyframeActionExecutor.java:855)
	at com.google.devtools.build.lib.skyframe.SkyframeActionExecutor$ActionRunner.continueAction(SkyframeActionExecutor.java:1016)
	at com.google.devtools.build.lib.skyframe.SkyframeActionExecutor$ActionRunner.run(SkyframeActionExecutor.java:975)
	at com.google.devtools.build.lib.skyframe.ActionExecutionState.runStateMachine(ActionExecutionState.java:129)
	at com.google.devtools.build.lib.skyframe.ActionExecutionState.getResultOrDependOnFuture(ActionExecutionState.java:81)
	at com.google.devtools.build.lib.skyframe.SkyframeActionExecutor.executeAction(SkyframeActionExecutor.java:472)
	at com.google.devtools.build.lib.skyframe.ActionExecutionFunction.checkCacheAndExecuteIfNeeded(ActionExecutionFunction.java:834)
	at com.google.devtools.build.lib.skyframe.ActionExecutionFunction.compute(ActionExecutionFunction.java:307)
	at com.google.devtools.build.skyframe.AbstractParallelEvaluator$Evaluate.run(AbstractParallelEvaluator.java:477)
	at com.google.devtools.build.lib.concurrent.AbstractQueueVisitor$WrappedRunnable.run(AbstractQueueVisitor.java:398)
	at java.base/java.util.concurrent.ThreadPoolExecutor.runWorker(Unknown Source)
	at java.base/java.util.concurrent.ThreadPoolExecutor$Worker.run(Unknown Source)
	at java.base/java.lang.Thread.run(Unknown Source)
Caused by: com.google.devtools.build.lib.remote.ExecutionStatusException: FAILED_PRECONDITION: Action 5dc5e44338b97515cc06a7ddbba8873ffbb8156ec213565d807228aee084315a/144 is invalid: A requested input (or the `Action` or its `Command`) was not found in the CAS..
	at com.google.devtools.build.lib.remote.GrpcRemoteExecutor.handleStatus(GrpcRemoteExecutor.java:70)
	at com.google.devtools.build.lib.remote.GrpcRemoteExecutor.getOperationResponse(GrpcRemoteExecutor.java:82)
	at com.google.devtools.build.lib.remote.GrpcRemoteExecutor.lambda$executeRemotely$0(GrpcRemoteExecutor.java:185)
	at com.google.devtools.build.lib.remote.Retrier.execute(Retrier.java:244)
	at com.google.devtools.build.lib.remote.RemoteRetrier.execute(RemoteRetrier.java:125)
	at com.google.devtools.build.lib.remote.RemoteRetrier.execute(RemoteRetrier.java:114)
	at com.google.devtools.build.lib.remote.GrpcRemoteExecutor.lambda$executeRemotely$1(GrpcRemoteExecutor.java:139)
	at com.google.devtools.build.lib.remote.util.Utils.refreshIfUnauthenticated(Utils.java:494)
	at com.google.devtools.build.lib.remote.GrpcRemoteExecutor.executeRemotely(GrpcRemoteExecutor.java:137)
	... 26 more
INFO: Elapsed time: 349.293s, Critical Path: 66.50s
INFO: 20458 processes: 10631 remote cache hit, 9430 internal, 397 remote.
FAILED: Build did NOT complete successfully
FAILED: Build did NOT complete successfully
Running after_script
00:01
Running after script...
$ echo "================= CI_PIPELINE_ID $CI_PIPELINE_ID"; echo "================= CI_JOB_ID ID $CI_JOB_ID"; echo "================= CI Runner `cat /etc/host_hostname`"; echo "================= Docker Instance `hostname`"; echo "================= Current Path `pwd`"; echo "================= PULL MAP $PULL_MAP"; echo "================= CI_MERGE_REQUEST_LABELS $CI_MERGE_REQUEST_LABELS";
================= CI_PIPELINE_ID 103297
================= CI_JOB_ID ID 2241495
================= CI Runner cn-cpu01018ack
================= Docker Instance runner-nnsd5xau-project-4-concurrent-95mk8c
================= Current Path /builds/nNsD5xAU/9/root/qcraft
================= PULL MAP true
================= CI_MERGE_REQUEST_LABELS 
$ scripts/ci_cleanup.sh
Cleaning up file based variables
00:00
ERROR: Job failed: command terminated with exit code 1
```



内存不足导致的问题，解决方法?rerun?

```
bazel-out/k8-opt/bin/external/io_opentelemetry_cpp/api/_virtual_includes/api/opentelemetry/context/runtime_context.h:25:3: warning: explicitly defaulted default constructor is implicitly deleted [-Wdefaulted-function-deleted]
  Token() noexcept = default;
  ^
bazel-out/k8-opt/bin/external/io_opentelemetry_cpp/api/_virtual_includes/api/opentelemetry/context/runtime_context.h:31:17: note: default constructor of 'Token' is implicitly deleted because field 'context_' of const-qualified type 'const opentelemetry::context::Context' would not be initialized
  const Context context_;
                ^
1 warning generated.
[7,063 / 7,760] 31 / 2730 tests; Compiling onboard/perception/traffic_light/traffic_light_classifier.cc; 82s remote-cache, processwrapper-sandbox ... (20 actions running)
[7,063 / 7,760] 31 / 2730 tests; Compiling onboard/perception/traffic_light/traffic_light_classifier.cc; 156s remote-cache, processwrapper-sandbox ... (20 actions running)
[7,063 / 7,760] 31 / 2730 tests; Compiling onboard/perception/traffic_light/traffic_light_classifier.cc; 241s remote-cache, processwrapper-sandbox ... (20 actions running)
[7,063 / 7,760] 31 / 2730 tests; Compiling onboard/perception/traffic_light/traffic_light_classifier.cc; 360s remote-cache, processwrapper-sandbox ... (20 actions running)
[7,063 / 7,760] 31 / 2730 tests; Compiling onboard/perception/traffic_light/traffic_light_classifier.cc; 484s remote-cache, processwrapper-sandbox ... (20 actions running)
Server terminated abruptly (error code: 14, error message: 'Socket closed', log file: '/home/qcrafter/.cache/bazel/_bazel_qcrafter/3f9d30763bdd1976729c71b2e94e5c8a/server/jvm.out')
Running after_script
00:05
Running after script...
$ rsync -a --prune-empty-dirs --include '*/' --include 'test.xml' --exclude '*' bazel-out/k8-opt/testlogs .
$ echo "================= CI_PIPELINE_ID $CI_PIPELINE_ID"; echo "================= CI_JOB_ID ID $CI_JOB_ID"; echo "================= CI Runner `cat /etc/host_hostname`"; echo "================= Docker Instance `hostname`"; echo "================= Current Path `pwd`"; echo "================= PULL MAP $PULL_MAP"; echo "================= CI_MERGE_REQUEST_LABELS $CI_MERGE_REQUEST_LABELS";
================= CI_PIPELINE_ID 103405
================= CI_JOB_ID ID 2244084
================= CI Runner cn-cpu01016ack
================= Docker Instance runner-nnsd5xau-project-4-concurrent-4vd2g8
```



新加的机器，权限问题没解决，解决了就好了。解决的方法是`chmod 666 /var/run/docker.sock`，理论上加了就好。不过有个有趣的问题，为什么gpu机器就不需要加这个东西？直接就能用？

```
settings.gradle
third_party
/builds/nNsD5xAU/3/root/qcraft
$ echo "SANITIZER_CONFIG=$SANITIZER_CONFIG";
SANITIZER_CONFIG=
$ scripts/ci_link_map_k8s.sh
Create symbolic links ..
$ export GLOG_logtostderr=1
$ export USER=qcrafter
$ export BAZEL_FLAGS=$SANITIZER_CONFIG
$ scripts/run_scenario_test.sh --china_mode=$CHINA_MODE --test_sets=$SCENARIO_TEST_SETS --type=$TEST_TYPE --user=$GITLAB_USER_NAME --jobs_server=$JOBS_SERVER --labels=$COMMON_TEST_LABELS,$TEST_LABELS --disable_vantage_forwarding --description=$DESCRIPTION $CUSTOME_TAGS;
Using default TAG at dev-devquery-20211228_1109
Using ECR_CN Image
Got permission denied while trying to connect to the Docker daemon socket at unix:///var/run/docker.sock: Post "http://%2Fvar%2Frun%2Fdocker.sock/v1.24/auth": dial unix /var/run/docker.sock: connect: permission denied
Running after_script
00:00
Running after script...
$ rsync -a --prune-empty-dirs --include '*/' --include 'test.xml' --exclude '*' bazel-out/k8-opt/testlogs .
rsync: change_dir "/builds/nNsD5xAU/3/root/qcraft//bazel-out/k8-opt" failed: No such file or directory (2)
rsync error: some files/attrs were not transferred (see previous errors) (code 23) at main.c(1196) [sender=3.1.2]
Cleaning up file based variables
00:01
ERROR: Job failed: command terminated with exit code 1
```



pod关闭，然后exec在pod里被迫退出的错误码是137

```
command terminated with exit code 137
```



所有会影响到est planner的改动都有可能影响到selector_test。。。

```
FAIL: //onboard/planner/selector:selector_test (see /home/qcrafter/.cache/bazel/_bazel_qcrafter/913a3d32ff0528d9e00a91687d23280b/execroot/com_qcraft/bazel-out/k8-opt/testlogs/onboard/planner/selector/selector_test/test.log)
INFO: From Testing //onboard/planner/selector:selector_test:
==================== Test output for //onboard/planner/selector:selector_test:
Warning: please export TSAN_OPTIONS='ignore_noninstrumented_modules=1' to avoid false positive reports from the OpenMP runtime!
WARNING: Logging before InitGoogleLogging() is written to STDERR
W1230 02:46:03.330783    12 map_selector.cc:172] /qcraft-maps/ not exist!
W1230 02:46:03.330811    12 map_selector.cc:172] /qcraft-maps-china/ not exist!
[==========] Running 3 tests from 1 test suite.
[----------] Global test environment set-up.
[----------] 3 tests from Selector
[ RUN      ] Selector.OvertakeTest
I1230 02:46:03.331035    12 param_util.cc:581] Use v2 params
I1230 02:46:03.331912    12 param_manager.cc:159] Loading the main param file from onboard/params/param_files/mkz_testing.pb.txt
I1230 02:46:03.344257    12 semantic_map_io.cc:328] Try to delegate the map loading...
I1230 02:46:03.344280    12 map_loader_delegate.cc:23] Map: dojo, version:
I1230 02:46:03.344434    12 map_loader_delegate.cc:44]  set country code by office: 1
I1230 02:46:03.344444    12 semantic_map_manager.cc:42] Delegated map loading fails. Load the local map now...
I1230 02:46:03.344452    12 semantic_map_io.cc:702] [LoadSemanticMap] map_location_dir=/qcraft/onboard/maps/data/dojo/semantic_map
I1230 02:46:03.469314    12 semantic_map_manager.cc:51] Load semantic map done, Load meta:OK
W1230 02:46:03.636983    12 speed_optimizer.cc:56] The negative speed -0.017 calculated by osqp is too large,the threshold: -0.010 has been exceeded.
W1230 02:46:03.637006    12 speed_optimizer.cc:42] The cumulative distance of speed point is nonmonotonic, the threshold: -0.010 has been exceeded, prev_s: 26.867 now_s: 26.856.
[       OK ] Selector.OvertakeTest (330 ms)
[ RUN      ] Selector.LaneChangeTest
I1230 02:46:03.806254    12 param_util.cc:581] Use v2 params
I1230 02:46:03.806838    12 param_manager.cc:159] Loading the main param file from onboard/params/param_files/mkz_testing.pb.txt
I1230 02:46:03.808241    12 semantic_map_io.cc:328] Try to delegate the map loading...
I1230 02:46:03.808470    12 semantic_map_manager.cc:42] Delegated map loading fails. Load the local map now...
I1230 02:46:03.808488    12 semantic_map_io.cc:702] [LoadSemanticMap] map_location_dir=/qcraft/onboard/maps/data/dojo/semantic_map
I1230 02:46:04.013269    12 semantic_map_manager.cc:51] Load semantic map done, Load meta:OK
[       OK ] Selector.LaneChangeTest (364 ms)
[ RUN      ] Selector.EarlyLaneChangeTest
I1230 02:46:04.170562    12 param_util.cc:581] Use v2 params
I1230 02:46:04.171169    12 param_manager.cc:159] Loading the main param file from onboard/params/param_files/mkz_testing.pb.txt
I1230 02:46:04.172641    12 semantic_map_io.cc:328] Try to delegate the map loading...
I1230 02:46:04.172659    12 semantic_map_manager.cc:42] Delegated map loading fails. Load the local map now...
I1230 02:46:04.172669    12 semantic_map_io.cc:702] [LoadSemanticMap] map_location_dir=/qcraft/onboard/maps/data/dojo/semantic_map
I1230 02:46:04.302968    12 semantic_map_manager.cc:51] Load semantic map done, Load meta:OK
W1230 02:46:04.395381    12 trajectory_validation.cc:604] EstPlanner plan failed validation.
W1230 02:46:04.395907    12 trajectory_validation.cc:605] Validation details: validation_errors_ex {
  error_code: OUT_OF_DRIVE_PASSAGE_QUERY_AREA
  error_message: "QueryFrenetLatOffset fail at traj point 82 (275.584575, -220.331998), return info: OUT_OF_RANGE: 5.572352 is out of left lateral range 5.479109."
}
onboard/planner/selector/selector_test.cc:412: Failure
Value of: results[i].ok()
  Actual: false
Expected: true
[  FAILED  ] Selector.EarlyLaneChangeTest (248 ms)
[----------] 3 tests from Selector (1088 ms total)
[----------] Global test environment tear-down
[==========] 3 tests from 1 test suite ran. (1088 ms total)
[  PASSED  ] 2 tests.
[  FAILED  ] 1 test, listed below:
[  FAILED  ] Selector.EarlyLaneChangeTest
 1 FAILED TEST
```



SIM-SERVER挂了

```
26894cd74717: Pushed
36fea60e-76d4-44d1-82ad-9aa5f0e0e7e3: digest: sha256:60de6c7b2f6055c747b74d1ce3845c4e4dcbc3ceed9bbb4677fd08bb60e54183 size: 9594
push image took 0m:33s
I1130 06:29:28.351953   983 argo_launcher.cc:102] build and push image takes 105 seconds
I1130 06:29:28.352202   207 argo_launcher.cc:183] Submitting workflow bd234a59-6a21-48b0-aaa4-bc2d84143de0
I1130 06:29:28.352221   207 argo_launcher.cc:184] Workflow URL https://argo-cn.qcraftai.com/workflows/default/bd234a59-6a21-48b0-aaa4-bc2d84143de0
I1130 06:29:29.742676   207 launcher.cc:21] Job status will be uploaded to http://sim-dash.qcraftai.com/results/1717833444491428746
W1130 06:29:29.872267   207 retry.h:21] 14 failed to connect to all addresses retrying
W1130 06:29:29.972374   207 retry.h:21] 14 failed to connect to all addresses retrying
W1130 06:29:30.172513   207 retry.h:21] 14 failed to connect to all addresses retrying
W1130 06:29:30.572655   207 retry.h:21] 14 failed to connect to all addresses retrying
W1130 06:29:31.460412   207 retry.h:21] 14 failed to connect to all addresses retrying
W1130 06:29:33.291597   207 retry.h:21] 14 failed to connect to all addresses retrying
W1130 06:29:36.577880   207 retry.h:21] 14 failed to connect to all addresses retrying
W1130 06:29:41.666393   207 retry.h:21] 14 failed to connect to all addresses retrying
W1130 06:29:49.178841   207 retry.h:21] 14 failed to connect to all addresses retrying
W1130 06:29:54.178995   207 retry.h:21] 14 failed to connect to all addresses retrying
W1130 06:30:14.667342   207 retry.h:21] 14 failed to connect to all addresses retrying
E1130 06:30:19.667439   207 job.cc:67] Error Getting Job 14, failed to connect to all addresses
W1130 06:31:26.419387   207 retry.h:21] 14 failed to connect to all addresses retrying
W1130 06:32:46.578815   207 retry.h:21] 14 failed to connect to all addresses retrying
W1130 06:32:46.778951   207 retry.h:21] 14 failed to connect to all addresses retrying
W1130 06:32:47.179091   207 retry.h:21] 14 failed to connect to all addresses retrying
W1130 06:32:47.979251   207 retry.h:21] 14 failed to connect to all addresses retrying
W1130 06:32:49.579412   207 retry.h:21] 14 failed to connect to all addresses retrying
W1130 06:32:52.779572   207 retry.h:21] 14 failed to connect to all addresses retrying
W1130 06:32:57.779747   207 retry.h:21] 14 failed to connect to all addresses retrying
W1130 06:33:02.779912   207 retry.h:21] 14 failed to connect to all addresses retrying
W1130 06:33:07.780084   207 retry.h:21] 14 failed to connect to all addresses retrying
W1130 06:33:12.780243   207 retry.h:21] 14 failed to connect to all addresses retrying
E1130 06:33:17.780347   207 job.cc:137] Error Getting Job 14, failed to connect to all addresses
I1130 06:33:17.780385   207 launcher.cc:42] Job status FINISHED
```



gpu内存不足导致的问题，这个是申请内存不足导致的，rerun一般能通过，我目前改了一些sync_test让单独跑应该能一定程度缓解问题。

```
[0x56452ec76fd0]:4 :Quantization: weight scales in internalAllocate: at runtime/common/weightsPtr.cpp: 100 idx: 46 time: 1.09e-07
[0x56452ece45e0]:4 :Quantization: weight scales in internalAllocate: at runtime/common/weightsPtr.cpp: 100 idx: 52 time: 1.61e-07
[0x56452ec50060]:4 :Quantization: weight scales in internalAllocate: at runtime/common/weightsPtr.cpp: 100 idx: 44 time: 1.77e-07
[0x56452eceb4d0]:4 :Quantization: weight scales in internalAllocate: at runtime/common/weightsPtr.cpp: 100 idx: 54 time: 1.28e-07
[0x56452ed2be00]:4 :Quantization: weight scales in internalAllocate: at runtime/common/weightsPtr.cpp: 100 idx: 56 time: 1.84e-07
-------------- The current device memory allocations dump as below --------------
[0]:2109440000 :GPU context memory in ExecutionContext: at runtime/api/executionContext.cpp: 208 idx: 2 time: 0.0227677
[0x7f24d8000000]:287578112 :GPU per-runner memory in ExecutionContext: at runtime/api/executionContext.cpp: 159 idx: 1 time: 0.00108015
[0x7f250a000000]:287578112 :GpuGlob deserialization in load: at runtime/deserialization/safeDeserialize.cpp: 349 idx: 0 time: 0.00288393
E0106 05:15:11.824108 12990 tensor_net.h:496] [TRT]    Requested amount of GPU memory (2109440000 bytes) could not be allocated. There may not be enough free memory for allocation to succeed.
I0106 05:15:11.844661 12990 tensor_net.h:492] [TRT]    [MemUsageChange] Init cuBLAS/cuBLASLt: CPU +0, GPU +0, now: CPU 1509, GPU 8844 (MiB)
E0106 05:15:11.844966 12990 tensor_net.h:496] [TRT]    2: [executionContext.cpp::ExecutionContext::208] Error Code 2: OutOfMemory (no further information)
E0106 05:15:11.844991 12990 tensor_net.cc:892] [TRT]    device GPU, failed to create execution context
E0106 05:15:11.845000 12990 tensor_net.cc:872] [TRT]    device GPU, failed to create resources for CUDA engine
E0106 05:15:11.845007 12990 tensor_net.cc:813] [TRT]    failed to create TensorRT engine for onboard/params/nets/depth_net/depth_net_1122/depth_net.onnx, device GPU
E0106 05:15:11.845016 12990 tensor_net.cc:339] [TRT]    failed to load onboard/params/nets/depth_net/depth_net_1122/depth_net.onnx
================================================================================
INFO: Elapsed time: 628.470s, Critical Path: 542.68s
INFO: 25246 processes: 13893 remote cache hit, 10952 internal, 1 local, 400 processwrapper-sandbox.
INFO: Build completed, 2 tests FAILED, 25246 total actions
Test cases: finished with 4258 passing and 2 failing out of 4260 test cases
Executed 257 out of 3091 tests: 3089 tests pass and 2 fail locally.
INFO: Build completed, 2 tests FAILED, 25246 total actions
Running after_script
00:02
Running after script...
$ rsync -a --prune-empty-dirs --include '*/' --include 'test.xml' --exclude '*' bazel-out/k8-opt/testlogs .
$ echo "================= CI_PIPELINE_ID $CI_PIPELINE_ID"; echo "================= CI_JOB_ID ID $CI_JOB_ID"; echo "================= CI Runner `cat /etc/host_hostname`"; echo "================= Docker Instance `hostname`"; echo "================= Current Path `pwd`"; echo "================= PULL MAP $PULL_MAP"; echo "================= CI_MERGE_REQUEST_LABELS $CI_MERGE_REQUEST_LABELS";
================= CI_PIPELINE_ID 105448
================= CI_JOB_ID ID 2292015
================= CI Runner aliyun-edge-2080ti-05
================= Docker Instance runner-evgkcu2b-project-4-concurrent-1nncxl
================= Current Path /builds/root/qcraft
================= PULL MAP true
================= CI_MERGE_REQUEST_LABELS 
$ scripts/ci_cleanup.sh

```



buildfarm错误，目前

```shell
[15,887 / 15,964] Compiling offboard/mapping/pose_graph_mapping/imagery_run_segment.cc; 354s remote ... (30 actions, 25 running)
ERROR: /builds/sJYWyPQz/2/root/qcraft/offboard/ml/data_processing/rain_filter/BUILD:24:10: Compiling offboard/ml/data_processing/rain_filter/simulator_with_map_version.cc failed: (Exit 34): 3 errors during bulk transfer
com.google.devtools.build.lib.remote.BulkTransferException: 3 errors during bulk transfer
	at com.google.devtools.build.lib.remote.RemoteCache.waitForBulkTransfer(RemoteCache.java:291)
	at com.google.devtools.build.lib.remote.RemoteExecutionCache.ensureInputsPresent(RemoteExecutionCache.java:72)
	at com.google.devtools.build.lib.remote.RemoteExecutionService.uploadInputsIfNotPresent(RemoteExecutionService.java:439)
	at com.google.devtools.build.lib.remote.RemoteSpawnRunner.lambda$exec$2(RemoteSpawnRunner.java:236)
	at com.google.devtools.build.lib.remote.Retrier.execute(Retrier.java:244)
	at com.google.devtools.build.lib.remote.RemoteRetrier.execute(RemoteRetrier.java:125)
	at com.google.devtools.build.lib.remote.RemoteRetrier.execute(RemoteRetrier.java:114)
	at com.google.devtools.build.lib.remote.RemoteSpawnRunner.exec(RemoteSpawnRunner.java:230)
	at com.google.devtools.build.lib.exec.SpawnRunner.execAsync(SpawnRunner.java:238)
	at com.google.devtools.build.lib.exec.AbstractSpawnStrategy.exec(AbstractSpawnStrategy.java:144)
	at com.google.devtools.build.lib.exec.AbstractSpawnStrategy.exec(AbstractSpawnStrategy.java:106)
	at com.google.devtools.build.lib.actions.SpawnStrategy.beginExecution(SpawnStrategy.java:47)
	at com.google.devtools.build.lib.exec.SpawnStrategyResolver.beginExecution(SpawnStrategyResolver.java:65)
	at com.google.devtools.build.lib.rules.cpp.CppCompileAction.beginExecution(CppCompileAction.java:1451)
	at com.google.devtools.build.lib.actions.Action.execute(Action.java:127)
	at com.google.devtools.build.lib.skyframe.SkyframeActionExecutor$5.execute(SkyframeActionExecutor.java:855)
	at com.google.devtools.build.lib.skyframe.SkyframeActionExecutor$ActionRunner.continueAction(SkyframeActionExecutor.java:1016)
	at com.google.devtools.build.lib.skyframe.SkyframeActionExecutor$ActionRunner.run(SkyframeActionExecutor.java:975)
	at com.google.devtools.build.lib.skyframe.ActionExecutionState.runStateMachine(ActionExecutionState.java:129)
	at com.google.devtools.build.lib.skyframe.ActionExecutionState.getResultOrDependOnFuture(ActionExecutionState.java:81)
	at com.google.devtools.build.lib.skyframe.SkyframeActionExecutor.executeAction(SkyframeActionExecutor.java:472)
	at com.google.devtools.build.lib.skyframe.ActionExecutionFunction.checkCacheAndExecuteIfNeeded(ActionExecutionFunction.java:834)
	at com.google.devtools.build.lib.skyframe.ActionExecutionFunction.compute(ActionExecutionFunction.java:307)
	at com.google.devtools.build.skyframe.AbstractParallelEvaluator$Evaluate.run(AbstractParallelEvaluator.java:477)
	at com.google.devtools.build.lib.concurrent.AbstractQueueVisitor$WrappedRunnable.run(AbstractQueueVisitor.java:398)
	at java.base/java.util.concurrent.ThreadPoolExecutor.runWorker(Unknown Source)
	at java.base/java.util.concurrent.ThreadPoolExecutor$Worker.run(Unknown Source)
	at java.base/java.lang.Thread.run(Unknown Source)
	Suppressed: java.io.IOException: Error while uploading artifact with digest 'a6fab537e4fde1e80058855399bca551b7cdb9ef98a40221c7ac03d999abbf2b/249'
		at com.google.devtools.build.lib.remote.ByteStreamUploader.lambda$uploadBlobAsync$1(ByteStreamUploader.java:265)
		at com.google.common.util.concurrent.AbstractCatchingFuture$AsyncCatchingFuture.doFallback(AbstractCatchingFuture.java:192)
		at com.google.common.util.concurrent.AbstractCatchingFuture$AsyncCatchingFuture.doFallback(AbstractCatchingFuture.java:179)
		at com.google.common.util.concurrent.AbstractCatchingFuture.run(AbstractCatchingFuture.java:124)
		at com.google.common.util.concurrent.DirectExecutor.execute(DirectExecutor.java:30)
		at com.google.common.util.concurrent.AbstractFuture.executeListener(AbstractFuture.java:1174)
		at com.google.common.util.concurrent.AbstractFuture.complete(AbstractFuture.java:969)
		at com.google.common.util.concurrent.AbstractFuture.setException(AbstractFuture.java:760)
		at com.google.common.util.concurrent.AbstractTransformFuture.run(AbstractTransformFuture.java:100)
		at com.google.common.util.concurrent.DirectExecutor.execute(DirectExecutor.java:30)
		at com.google.common.util.concurrent.AbstractFuture.executeListener(AbstractFuture.java:1174)
		at com.google.common.util.concurrent.AbstractFuture.complete(AbstractFuture.java:969)
		at com.google.common.util.concurrent.AbstractFuture.setException(AbstractFuture.java:760)
		at com.google.common.util.concurrent.AbstractTransformFuture.run(AbstractTransformFuture.java:100)
		at com.google.common.util.concurrent.DirectExecutor.execute(DirectExecutor.java:30)
		at com.google.common.util.concurrent.AbstractFuture.executeListener(AbstractFuture.java:1174)
		at com.google.common.util.concurrent.AbstractFuture.complete(AbstractFuture.java:969)
		at com.google.common.util.concurrent.AbstractFuture.setFuture(AbstractFuture.java:800)
		at com.google.common.util.concurrent.AbstractCatchingFuture$AsyncCatchingFuture.setResult(AbstractCatchingFuture.java:203)
		at com.google.common.util.concurrent.AbstractCatchingFuture$AsyncCatchingFuture.setResult(AbstractCatchingFuture.java:179)
		at com.google.common.util.concurrent.AbstractCatchingFuture.run(AbstractCatchingFuture.java:133)
		at com.google.common.util.concurrent.DirectExecutor.execute(DirectExecutor.java:30)
		at com.google.common.util.concurrent.AbstractFuture.executeListener(AbstractFuture.java:1174)
		at com.google.common.util.concurrent.AbstractFuture.complete(AbstractFuture.java:969)
		at com.google.common.util.concurrent.AbstractFuture.setFuture(AbstractFuture.java:800)
		at com.google.common.util.concurrent.AbstractCatchingFuture$AsyncCatchingFuture.setResult(AbstractCatchingFuture.java:203)
		at com.google.common.util.concurrent.AbstractCatchingFuture$AsyncCatchingFuture.setResult(AbstractCatchingFuture.java:179)
		at com.google.common.util.concurrent.AbstractCatchingFuture.run(AbstractCatchingFuture.java:133)
		at com.google.common.util.concurrent.DirectExecutor.execute(DirectExecutor.java:30)
		at com.google.common.util.concurrent.AbstractFuture.executeListener(AbstractFuture.java:1174)
		at com.google.common.util.concurrent.AbstractFuture.complete(AbstractFuture.java:969)
		at com.google.common.util.concurrent.AbstractFuture.setException(AbstractFuture.java:760)
		at com.google.common.util.concurrent.AbstractTransformFuture.run(AbstractTransformFuture.java:100)
		at com.google.common.util.concurrent.DirectExecutor.execute(DirectExecutor.java:30)
		at com.google.common.util.concurrent.AbstractFuture.executeListener(AbstractFuture.java:1174)
		at com.google.common.util.concurrent.AbstractFuture.complete(AbstractFuture.java:969)
		at com.google.common.util.concurrent.AbstractFuture.setFuture(AbstractFuture.java:800)
		at com.google.common.util.concurrent.AbstractCatchingFuture$AsyncCatchingFuture.setResult(AbstractCatchingFuture.java:203)
		at com.google.common.util.concurrent.AbstractCatchingFuture$AsyncCatchingFuture.setResult(AbstractCatchingFuture.java:179)
		at com.google.common.util.concurrent.AbstractCatchingFuture.run(AbstractCatchingFuture.java:133)
		at com.google.common.util.concurrent.DirectExecutor.execute(DirectExecutor.java:30)
		at com.google.common.util.concurrent.AbstractFuture.executeListener(AbstractFuture.java:1174)
		at com.google.common.util.concurrent.AbstractFuture.complete(AbstractFuture.java:969)
		at com.google.common.util.concurrent.AbstractFuture.setException(AbstractFuture.java:760)
		at com.google.common.util.concurrent.AbstractTransformFuture.run(AbstractTransformFuture.java:100)
		at com.google.common.util.concurrent.DirectExecutor.execute(DirectExecutor.java:30)
		at com.google.common.util.concurrent.AbstractFuture.executeListener(AbstractFuture.java:1174)
		at com.google.common.util.concurrent.AbstractFuture.complete(AbstractFuture.java:969)
		at com.google.common.util.concurrent.AbstractFuture.setFuture(AbstractFuture.java:800)
		at com.google.common.util.concurrent.AbstractCatchingFuture$AsyncCatchingFuture.setResult(AbstractCatchingFuture.java:203)
		at com.google.common.util.concurrent.AbstractCatchingFuture$AsyncCatchingFuture.setResult(AbstractCatchingFuture.java:179)
		at com.google.common.util.concurrent.AbstractCatchingFuture.run(AbstractCatchingFuture.java:133)
		at com.google.common.util.concurrent.DirectExecutor.execute(DirectExecutor.java:30)
		at com.google.common.util.concurrent.AbstractFuture.executeListener(AbstractFuture.java:1174)
		at com.google.common.util.concurrent.AbstractFuture.complete(AbstractFuture.java:969)
		at com.google.common.util.concurrent.AbstractFuture.setException(AbstractFuture.java:760)
		at com.google.common.util.concurrent.AbstractTransformFuture.run(AbstractTransformFuture.java:100)
		at com.google.common.util.concurrent.DirectExecutor.execute(DirectExecutor.java:30)
		at com.google.common.util.concurrent.AbstractFuture.executeListener(AbstractFuture.java:1174)
		at com.google.common.util.concurrent.AbstractFuture.complete(AbstractFuture.java:969)
		at com.google.common.util.concurrent.AbstractFuture.setException(AbstractFuture.java:760)
		at io.grpc.stub.ClientCalls$GrpcFuture.setException(ClientCalls.java:563)
		at io.grpc.stub.ClientCalls$UnaryStreamToFuture.onClose(ClientCalls.java:533)
		at io.grpc.PartialForwardingClientCallListener.onClose(PartialForwardingClientCallListener.java:39)
		at io.grpc.ForwardingClientCallListener.onClose(ForwardingClientCallListener.java:23)
		at io.grpc.ForwardingClientCallListener$SimpleForwardingClientCallListener.onClose(ForwardingClientCallListener.java:40)
		at com.google.devtools.build.lib.remote.ReferenceCountedChannel$ConnectionCleanupCall$1.onClose(ReferenceCountedChannel.java:94)
		at io.grpc.internal.ClientCallImpl.closeObserver(ClientCallImpl.java:617)
		at io.grpc.internal.ClientCallImpl.access$300(ClientCallImpl.java:70)
		at io.grpc.internal.ClientCallImpl$ClientStreamListenerImpl$1StreamClosed.runInternal(ClientCallImpl.java:803)
		at io.grpc.internal.ClientCallImpl$ClientStreamListenerImpl$1StreamClosed.runInContext(ClientCallImpl.java:782)
		at io.grpc.internal.ContextRunnable.run(ContextRunnable.java:37)
		at io.grpc.internal.SerializingExecutor.run(SerializingExecutor.java:123)
		... 3 more
	Caused by: io.grpc.StatusRuntimeException: UNAVAILABLE: no available workers
		at io.grpc.Status.asRuntimeException(Status.java:524)
		at com.google.devtools.build.lib.remote.ByteStreamUploader$AsyncUpload$1.onClose(ByteStreamUploader.java:551)
		... 13 more
		Suppressed: io.grpc.StatusRuntimeException: UNAVAILABLE: no available workers
			at io.grpc.Status.asRuntimeException(Status.java:533)
			at io.grpc.stub.ClientCalls$UnaryStreamToFuture.onClose(ClientCalls.java:533)
			... 13 more
	Suppressed: java.io.IOException: Error while uploading artifact with digest '740fddd47606ef3cc0ad43a7e65cbce155266f9c1c1c30cc59783ed7a94c9c4f/1424'
		at com.google.devtools.build.lib.remote.ByteStreamUploader.lambda$uploadBlobAsync$1(ByteStreamUploader.java:265)
		at com.google.common.util.concurrent.AbstractCatchingFuture$AsyncCatchingFuture.doFallback(AbstractCatchingFuture.java:192)
		at com.google.common.util.concurrent.AbstractCatchingFuture$AsyncCatchingFuture.doFallback(AbstractCatchingFuture.java:179)
		at com.google.common.util.concurrent.AbstractCatchingFuture.run(AbstractCatchingFuture.java:124)
		at com.google.common.util.concurrent.DirectExecutor.execute(DirectExecutor.java:30)
		at com.google.common.util.concurrent.AbstractFuture.executeListener(AbstractFuture.java:1174)
		at com.google.common.util.concurrent.AbstractFuture.complete(AbstractFuture.java:969)
		at com.google.common.util.concurrent.AbstractFuture.setException(AbstractFuture.java:760)
		at com.google.common.util.concurrent.AbstractTransformFuture.run(AbstractTransformFuture.java:100)
		at com.google.common.util.concurrent.DirectExecutor.execute(DirectExecutor.java:30)
		at com.google.common.util.concurrent.AbstractFuture.executeListener(AbstractFuture.java:1174)
		at com.google.common.util.concurrent.AbstractFuture.complete(AbstractFuture.java:969)
		at com.google.common.util.concurrent.AbstractFuture.setException(AbstractFuture.java:760)
		at com.google.common.util.concurrent.AbstractTransformFuture.run(AbstractTransformFuture.java:100)
		at com.google.common.util.concurrent.DirectExecutor.execute(DirectExecutor.java:30)
		at com.google.common.util.concurrent.AbstractFuture.executeListener(AbstractFuture.java:1174)
		at com.google.common.util.concurrent.AbstractFuture.complete(AbstractFuture.java:969)
		at com.google.common.util.concurrent.AbstractFuture.setFuture(AbstractFuture.java:800)
		at com.google.common.util.concurrent.AbstractCatchingFuture$AsyncCatchingFuture.setResult(AbstractCatchingFuture.java:203)
		at com.google.common.util.concurrent.AbstractCatchingFuture$AsyncCatchingFuture.setResult(AbstractCatchingFuture.java:179)
		at com.google.common.util.concurrent.AbstractCatchingFuture.run(AbstractCatchingFuture.java:133)
		at com.google.common.util.concurrent.DirectExecutor.execute(DirectExecutor.java:30)
		at com.google.common.util.concurrent.AbstractFuture.executeListener(AbstractFuture.java:1174)
		at com.google.common.util.concurrent.AbstractFuture.complete(AbstractFuture.java:969)
		at com.google.common.util.concurrent.AbstractFuture.setFuture(AbstractFuture.java:800)
		at com.google.common.util.concurrent.AbstractCatchingFuture$AsyncCatchingFuture.setResult(AbstractCatchingFuture.java:203)
		at com.google.common.util.concurrent.AbstractCatchingFuture$AsyncCatchingFuture.setResult(AbstractCatchingFuture.java:179)
		at com.google.common.util.concurrent.AbstractCatchingFuture.run(AbstractCatchingFuture.java:133)
		at com.google.common.util.concurrent.DirectExecutor.execute(DirectExecutor.java:30)
		at com.google.common.util.concurrent.AbstractFuture.executeListener(AbstractFuture.java:1174)
		at com.google.common.util.concurrent.AbstractFuture.complete(AbstractFuture.java:969)
		at com.google.common.util.concurrent.AbstractFuture.setException(AbstractFuture.java:760)
		at com.google.common.util.concurrent.AbstractTransformFuture.run(AbstractTransformFuture.java:100)
		at com.google.common.util.concurrent.DirectExecutor.execute(DirectExecutor.java:30)
		at com.google.common.util.concurrent.AbstractFuture.executeListener(AbstractFuture.java:1174)
		at com.google.common.util.concurrent.AbstractFuture.complete(AbstractFuture.java:969)
		at com.google.common.util.concurrent.AbstractFuture.setFuture(AbstractFuture.java:800)
		at com.google.common.util.concurrent.AbstractCatchingFuture$AsyncCatchingFuture.setResult(AbstractCatchingFuture.java:203)
		at com.google.common.util.concurrent.AbstractCatchingFuture$AsyncCatchingFuture.setResult(AbstractCatchingFuture.java:179)
		at com.google.common.util.concurrent.AbstractCatchingFuture.run(AbstractCatchingFuture.java:133)
		at com.google.common.util.concurrent.DirectExecutor.execute(DirectExecutor.java:30)
		at com.google.common.util.concurrent.AbstractFuture.executeListener(AbstractFuture.java:1174)
		at com.google.common.util.concurrent.AbstractFuture.complete(AbstractFuture.java:969)
		at com.google.common.util.concurrent.AbstractFuture.setException(AbstractFuture.java:760)
		at com.google.common.util.concurrent.AbstractTransformFuture.run(AbstractTransformFuture.java:100)
		at com.google.common.util.concurrent.DirectExecutor.execute(DirectExecutor.java:30)
		at com.google.common.util.concurrent.AbstractFuture.executeListener(AbstractFuture.java:1174)
		at com.google.common.util.concurrent.AbstractFuture.complete(AbstractFuture.java:969)
		at com.google.common.util.concurrent.AbstractFuture.setFuture(AbstractFuture.java:800)
		at com.google.common.util.concurrent.AbstractCatchingFuture$AsyncCatchingFuture.setResult(AbstractCatchingFuture.java:203)
		at com.google.common.util.concurrent.AbstractCatchingFuture$AsyncCatchingFuture.setResult(AbstractCatchingFuture.java:179)
		at com.google.common.util.concurrent.AbstractCatchingFuture.run(AbstractCatchingFuture.java:133)
		at com.google.common.util.concurrent.DirectExecutor.execute(DirectExecutor.java:30)
		at com.google.common.util.concurrent.AbstractFuture.executeListener(AbstractFuture.java:1174)
		at com.google.common.util.concurrent.AbstractFuture.complete(AbstractFuture.java:969)
		at com.google.common.util.concurrent.AbstractFuture.setException(AbstractFuture.java:760)
		at com.google.common.util.concurrent.AbstractTransformFuture.run(AbstractTransformFuture.java:100)
		at com.google.common.util.concurrent.DirectExecutor.execute(DirectExecutor.java:30)
		at com.google.common.util.concurrent.AbstractFuture.executeListener(AbstractFuture.java:1174)
		at com.google.common.util.concurrent.AbstractFuture.complete(AbstractFuture.java:969)
		at com.google.common.util.concurrent.AbstractFuture.setException(AbstractFuture.java:760)
		at io.grpc.stub.ClientCalls$GrpcFuture.setException(ClientCalls.java:563)
		at io.grpc.stub.ClientCalls$UnaryStreamToFuture.onClose(ClientCalls.java:533)
		at io.grpc.PartialForwardingClientCallListener.onClose(PartialForwardingClientCallListener.java:39)
		at io.grpc.ForwardingClientCallListener.onClose(ForwardingClientCallListener.java:23)
		at io.grpc.ForwardingClientCallListener$SimpleForwardingClientCallListener.onClose(ForwardingClientCallListener.java:40)
		at com.google.devtools.build.lib.remote.ReferenceCountedChannel$ConnectionCleanupCall$1.onClose(ReferenceCountedChannel.java:94)
		at io.grpc.internal.ClientCallImpl.closeObserver(ClientCallImpl.java:617)
		at io.grpc.internal.ClientCallImpl.access$300(ClientCallImpl.java:70)
		at io.grpc.internal.ClientCallImpl$ClientStreamListenerImpl$1StreamClosed.runInternal(ClientCallImpl.java:803)
		at io.grpc.internal.ClientCallImpl$ClientStreamListenerImpl$1StreamClosed.runInContext(ClientCallImpl.java:782)
		at io.grpc.internal.ContextRunnable.run(ContextRunnable.java:37)
		at io.grpc.internal.SerializingExecutor.run(SerializingExecutor.java:123)
		... 3 more
	Caused by: io.grpc.StatusRuntimeException: UNAVAILABLE: no available workers
		at io.grpc.Status.asRuntimeException(Status.java:524)
		at com.google.devtools.build.lib.remote.ByteStreamUploader$AsyncUpload$1.onClose(ByteStreamUploader.java:551)
		... 13 more
		Suppressed: io.grpc.StatusRuntimeException: UNAVAILABLE: no available workers
			at io.grpc.Status.asRuntimeException(Status.java:533)
			at io.grpc.stub.ClientCalls$UnaryStreamToFuture.onClose(ClientCalls.java:533)
			... 13 more
	Suppressed: java.io.IOException: Error while uploading artifact with digest '8c7ae2c436cace2f45c804f7d5a2630b1e2b39010ee503b4a94232e6cc40dac1/811'
		at com.google.devtools.build.lib.remote.ByteStreamUploader.lambda$uploadBlobAsync$1(ByteStreamUploader.java:265)
		at com.google.common.util.concurrent.AbstractCatchingFuture$AsyncCatchingFuture.doFallback(AbstractCatchingFuture.java:192)
		at com.google.common.util.concurrent.AbstractCatchingFuture$AsyncCatchingFuture.doFallback(AbstractCatchingFuture.java:179)
		at com.google.common.util.concurrent.AbstractCatchingFuture.run(AbstractCatchingFuture.java:124)
		at com.google.common.util.concurrent.DirectExecutor.execute(DirectExecutor.java:30)
		at com.google.common.util.concurrent.AbstractFuture.executeListener(AbstractFuture.java:1174)
		at com.google.common.util.concurrent.AbstractFuture.complete(AbstractFuture.java:969)
		at com.google.common.util.concurrent.AbstractFuture.setException(AbstractFuture.java:760)
		at com.google.common.util.concurrent.AbstractTransformFuture.run(AbstractTransformFuture.java:100)
		at com.google.common.util.concurrent.DirectExecutor.execute(DirectExecutor.java:30)
		at com.google.common.util.concurrent.AbstractFuture.executeListener(AbstractFuture.java:1174)
		at com.google.common.util.concurrent.AbstractFuture.complete(AbstractFuture.java:969)
		at com.google.common.util.concurrent.AbstractFuture.setException(AbstractFuture.java:760)
		at com.google.common.util.concurrent.AbstractTransformFuture.run(AbstractTransformFuture.java:100)
		at com.google.common.util.concurrent.DirectExecutor.execute(DirectExecutor.java:30)
		at com.google.common.util.concurrent.AbstractFuture.executeListener(AbstractFuture.java:1174)
		at com.google.common.util.concurrent.AbstractFuture.complete(AbstractFuture.java:969)
		at com.google.common.util.concurrent.AbstractFuture.setFuture(AbstractFuture.java:800)
		at com.google.common.util.concurrent.AbstractCatchingFuture$AsyncCatchingFuture.setResult(AbstractCatchingFuture.java:203)
		at com.google.common.util.concurrent.AbstractCatchingFuture$AsyncCatchingFuture.setResult(AbstractCatchingFuture.java:179)
		at com.google.common.util.concurrent.AbstractCatchingFuture.run(AbstractCatchingFuture.java:133)
		at com.google.common.util.concurrent.DirectExecutor.execute(DirectExecutor.java:30)
		at com.google.common.util.concurrent.AbstractFuture.executeListener(AbstractFuture.java:1174)
		at com.google.common.util.concurrent.AbstractFuture.complete(AbstractFuture.java:969)
		at com.google.common.util.concurrent.AbstractFuture.setFuture(AbstractFuture.java:800)
		at com.google.common.util.concurrent.AbstractCatchingFuture$AsyncCatchingFuture.setResult(AbstractCatchingFuture.java:203)
		at com.google.common.util.concurrent.AbstractCatchingFuture$AsyncCatchingFuture.setResult(AbstractCatchingFuture.java:179)
		at com.google.common.util.concurrent.AbstractCatchingFuture.run(AbstractCatchingFuture.java:133)
		at com.google.common.util.concurrent.DirectExecutor.execute(DirectExecutor.java:30)
		at com.google.common.util.concurrent.AbstractFuture.executeListener(AbstractFuture.java:1174)
		at com.google.common.util.concurrent.AbstractFuture.complete(AbstractFuture.java:969)
		at com.google.common.util.concurrent.AbstractFuture.setException(AbstractFuture.java:760)
		at com.google.common.util.concurrent.AbstractTransformFuture.run(AbstractTransformFuture.java:100)
		at com.google.common.util.concurrent.DirectExecutor.execute(DirectExecutor.java:30)
		at com.google.common.util.concurrent.AbstractFuture.executeListener(AbstractFuture.java:1174)
		at com.google.common.util.concurrent.AbstractFuture.complete(AbstractFuture.java:969)
		at com.google.common.util.concurrent.AbstractFuture.setFuture(AbstractFuture.java:800)
		at com.google.common.util.concurrent.AbstractCatchingFuture$AsyncCatchingFuture.setResult(AbstractCatchingFuture.java:203)
		at com.google.common.util.concurrent.AbstractCatchingFuture$AsyncCatchingFuture.setResult(AbstractCatchingFuture.java:179)
		at com.google.common.util.concurrent.AbstractCatchingFuture.run(AbstractCatchingFuture.java:133)
		at com.google.common.util.concurrent.DirectExecutor.execute(DirectExecutor.java:30)
		at com.google.common.util.concurrent.AbstractFuture.executeListener(AbstractFuture.java:1174)
		at com.google.common.util.concurrent.AbstractFuture.complete(AbstractFuture.java:969)
		at com.google.common.util.concurrent.AbstractFuture.setException(AbstractFuture.java:760)
		at com.google.common.util.concurrent.AbstractTransformFuture.run(AbstractTransformFuture.java:100)
		at com.google.common.util.concurrent.DirectExecutor.execute(DirectExecutor.java:30)
		at com.google.common.util.concurrent.AbstractFuture.executeListener(AbstractFuture.java:1174)
		at com.google.common.util.concurrent.AbstractFuture.complete(AbstractFuture.java:969)
		at com.google.common.util.concurrent.AbstractFuture.setFuture(AbstractFuture.java:800)
		at com.google.common.util.concurrent.AbstractCatchingFuture$AsyncCatchingFuture.setResult(AbstractCatchingFuture.java:203)
		at com.google.common.util.concurrent.AbstractCatchingFuture$AsyncCatchingFuture.setResult(AbstractCatchingFuture.java:179)
		at com.google.common.util.concurrent.AbstractCatchingFuture.run(AbstractCatchingFuture.java:133)
		at com.google.common.util.concurrent.DirectExecutor.execute(DirectExecutor.java:30)
		at com.google.common.util.concurrent.AbstractFuture.executeListener(AbstractFuture.java:1174)
		at com.google.common.util.concurrent.AbstractFuture.complete(AbstractFuture.java:969)
		at com.google.common.util.concurrent.AbstractFuture.setException(AbstractFuture.java:760)
		at com.google.common.util.concurrent.AbstractTransformFuture.run(AbstractTransformFuture.java:100)
		at com.google.common.util.concurrent.DirectExecutor.execute(DirectExecutor.java:30)
		at com.google.common.util.concurrent.AbstractFuture.executeListener(AbstractFuture.java:1174)
		at com.google.common.util.concurrent.AbstractFuture.complete(AbstractFuture.java:969)
		at com.google.common.util.concurrent.AbstractFuture.setException(AbstractFuture.java:760)
		at io.grpc.stub.ClientCalls$GrpcFuture.setException(ClientCalls.java:563)
		at io.grpc.stub.ClientCalls$UnaryStreamToFuture.onClose(ClientCalls.java:533)
		at io.grpc.PartialForwardingClientCallListener.onClose(PartialForwardingClientCallListener.java:39)
		at io.grpc.ForwardingClientCallListener.onClose(ForwardingClientCallListener.java:23)
		at io.grpc.ForwardingClientCallListener$SimpleForwardingClientCallListener.onClose(ForwardingClientCallListener.java:40)
		at com.google.devtools.build.lib.remote.ReferenceCountedChannel$ConnectionCleanupCall$1.onClose(ReferenceCountedChannel.java:94)
		at io.grpc.internal.ClientCallImpl.closeObserver(ClientCallImpl.java:617)
		at io.grpc.internal.ClientCallImpl.access$300(ClientCallImpl.java:70)
		at io.grpc.internal.ClientCallImpl$ClientStreamListenerImpl$1StreamClosed.runInternal(ClientCallImpl.java:803)
		at io.grpc.internal.ClientCallImpl$ClientStreamListenerImpl$1StreamClosed.runInContext(ClientCallImpl.java:782)
		at io.grpc.internal.ContextRunnable.run(ContextRunnable.java:37)
		at io.grpc.internal.SerializingExecutor.run(SerializingExecutor.java:123)
		... 3 more
	Caused by: io.grpc.StatusRuntimeException: UNAVAILABLE: no available workers
		at io.grpc.Status.asRuntimeException(Status.java:524)
		at com.google.devtools.build.lib.remote.ByteStreamUploader$AsyncUpload$1.onClose(ByteStreamUploader.java:551)
		... 13 more
		Suppressed: io.grpc.StatusRuntimeException: UNAVAILABLE: no available workers
			at io.grpc.Status.asRuntimeException(Status.java:533)
			at io.grpc.stub.ClientCalls$UnaryStreamToFuture.onClose(ClientCalls.java:533)
			... 13 more
INFO: Elapsed time: 1190.402s, Critical Path: 414.85s
INFO: 15917 processes: 8589 remote cache hit, 6033 internal, 1295 remote.
FAILED: Build did NOT complete successfully
FAILED: Build did NOT complete successfully
```







RFC模板https://qcraft.feishu.cn/docs/doccn5Vgkr1N7vZrvS2sLCFxd1c#FxyDgh

```
.k8s-bazel-test-template
	k8s-bazel-test-postsubmit
	.k8s-bazel-test-schedule-template:
		k8s-bazel-test-schedule-tsan
		k8s-bazel-test-schedule-asan
		k8s-bazel-test-schedule-ubsan

```



问题，为什么https://gitlab.qcraft.ai/root/qcraft/-/merge_requests/15613 的pipeline会出现多个pipeline passed？


```

```



bazel内部错误，retry即可

```
Analyzing: 7510 targets (1034 packages loaded, 36333 targets configured)
Analyzing: 7510 targets (1047 packages loaded, 38275 targets configured)
Analyzing: 7510 targets (1056 packages loaded, 39778 targets configured)
Analyzing: 7510 targets (1072 packages loaded, 44313 targets configured)
Analyzing: 7510 targets (1085 packages loaded, 62517 targets configured)
Analyzing: 7510 targets (1174 packages loaded, 63306 targets configured)
INFO: Analyzed 7510 targets (1278 packages loaded, 64206 targets configured).
INFO: Found 7510 targets...
[0 / 562] [Prepa] BazelWorkspaceStatusAction stable-status.txt
FATAL: bazel crashed due to an internal error. Printing stack trace:
java.lang.RuntimeException: Unrecoverable error while evaluating node 'UnshareableActionLookupData{actionLookupKey=com.google.devtools.build.lib.skyframe.WorkspaceStatusValue$BuildInfoKey@1ade3d14, actionIndex=0}' (requested by nodes )
	at com.google.devtools.build.skyframe.AbstractParallelEvaluator$Evaluate.run(AbstractParallelEvaluator.java:563)
	at com.google.devtools.build.lib.concurrent.AbstractQueueVisitor$WrappedRunnable.run(AbstractQueueVisitor.java:398)
	at java.base/java.util.concurrent.ThreadPoolExecutor.runWorker(Unknown Source)
	at java.base/java.util.concurrent.ThreadPoolExecutor$Worker.run(Unknown Source)
	at java.base/java.lang.Thread.run(Unknown Source)
Caused by: java.lang.ArrayIndexOutOfBoundsException: Index 424 out of bounds for length 424
	at com.google.devtools.build.lib.profiler.TimeSeries.addRange(TimeSeries.java:80)
	at com.google.devtools.build.lib.profiler.TimeSeries.addRange(TimeSeries.java:35)
	at com.google.devtools.build.lib.profiler.Profiler.completeTask(Profiler.java:728)
	at com.google.devtools.build.lib.profiler.Profiler.lambda$profileAction$1(Profiler.java:677)
	at com.google.devtools.build.lib.skyframe.SkyframeActionExecutor$ActionRunner.run(SkyframeActionExecutor.java:976)
	at com.google.devtools.build.lib.skyframe.ActionExecutionState.runStateMachine(ActionExecutionState.java:129)
	at com.google.devtools.build.lib.skyframe.ActionExecutionState.getResultOrDependOnFuture(ActionExecutionState.java:81)
	at com.google.devtools.build.lib.skyframe.SkyframeActionExecutor.executeAction(SkyframeActionExecutor.java:472)
	at com.google.devtools.build.lib.skyframe.ActionExecutionFunction.checkCacheAndExecuteIfNeeded(ActionExecutionFunction.java:834)
	at com.google.devtools.build.lib.skyframe.ActionExecutionFunction.compute(ActionExecutionFunction.java:307)
	at com.google.devtools.build.skyframe.AbstractParallelEvaluator$Evaluate.run(AbstractParallelEvaluator.java:477)
	... 4 more
Running after_script
00:01
Running after script...
$ rsync -a --prune-empty-dirs --include '*/' --include 'test.xml' --exclude '*' bazel-out/k8-opt/testlogs .
$ echo "================= CI_PIPELINE_ID $CI_PIPELINE_ID"; echo "================= CI_JOB_ID ID $CI_JOB_ID"; echo "================= CI Runner `cat /etc/host_hostname`"; echo "================= Docker Instance `hostname`"; echo "================= Current Path `pwd`"; echo "================= PULL MAP $PULL_MAP"; echo "================= CI_MERGE_REQUEST_LABELS $CI_MERGE_REQUEST_LABELS";
================= CI_PIPELINE_ID 108207
================= CI_JOB_ID ID 2358222
================= CI Runner us-edge-09
================= Docker Instance runner-9csnzycb-project-4-concurrent-7j2vr6
================= Current Path /builds/9CsNzyCB/7/root/qcraft
================= PULL MAP true
================= CI_MERGE_REQUEST_LABELS 
$ scripts/ci_cleanup.sh
Cleaning up file based variables
00:00
ERROR: Job failed: command terminated with exit code 1
```







```
INFO: Build options --copt and --define have changed, discarding analysis cache.
Analyzing: target //offboard/simulation/tools/regression:regression_helper_main (0 packages loaded, 0 targets configured)
INFO: Analyzed target //offboard/simulation/tools/regression:regression_helper_main (0 packages loaded, 3653 targets configured).
INFO: Found 1 target...
[0 / 3] [Prepa] BazelWorkspaceStatusAction stable-status.txt
Target //offboard/simulation/tools/regression:regression_helper_main up-to-date:
  bazel-bin/offboard/simulation/tools/regression/regression_helper_main
INFO: Elapsed time: 8.853s, Critical Path: 0.09s
INFO: 890 processes: 889 remote cache hit, 1 internal.
INFO: Build completed successfully, 890 total actions
INFO: Build completed successfully, 890 total actions
Traceback (most recent call last):
  File "/builds/9CsNzyCB/7/root/qcraft/./bazel-bin/offboard/simulation/tools/regression/regression_helper_main.runfiles/com_qcraft/offboard/simulation/tools/regression/regression_helper_main.py", line 130, in <module>
    main()
  File "/builds/9CsNzyCB/7/root/qcraft/bazel-bin/offboard/simulation/tools/regression/regression_helper_main.runfiles/qcraft_py3_deps_pypi__click/click/core.py", line 1128, in __call__
    return self.main(*args, **kwargs)
  File "/builds/9CsNzyCB/7/root/qcraft/bazel-bin/offboard/simulation/tools/regression/regression_helper_main.runfiles/qcraft_py3_deps_pypi__click/click/core.py", line 1053, in main
    rv = self.invoke(ctx)
  File "/builds/9CsNzyCB/7/root/qcraft/bazel-bin/offboard/simulation/tools/regression/regression_helper_main.runfiles/qcraft_py3_deps_pypi__click/click/core.py", line 1659, in invoke
    return _process_result(sub_ctx.command.invoke(sub_ctx))
  File "/builds/9CsNzyCB/7/root/qcraft/bazel-bin/offboard/simulation/tools/regression/regression_helper_main.runfiles/qcraft_py3_deps_pypi__click/click/core.py", line 1395, in invoke
    return ctx.invoke(self.callback, **ctx.params)
  File "/builds/9CsNzyCB/7/root/qcraft/bazel-bin/offboard/simulation/tools/regression/regression_helper_main.runfiles/qcraft_py3_deps_pypi__click/click/core.py", line 754, in invoke
    return __callback(*args, **kwargs)
  File "/builds/9CsNzyCB/7/root/qcraft/./bazel-bin/offboard/simulation/tools/regression/regression_helper_main.runfiles/com_qcraft/offboard/simulation/tools/regression/regression_helper_main.py", line 98, in calculate_regression
    include_metric_detail,
  File "/builds/9CsNzyCB/7/root/qcraft/bazel-bin/offboard/simulation/tools/regression/regression_helper_main.runfiles/com_qcraft/offboard/simulation/tools/regression/implements/qsim_result_impl.py", line 194, in calculate_regression
    include_metric_diff)
  File "/builds/9CsNzyCB/7/root/qcraft/bazel-bin/offboard/simulation/tools/regression/regression_helper_main.runfiles/com_qcraft/offboard/simulation/tools/regression/implements/qsim_result_impl.py", line 185, in compare_job_result
    include_metric_diff)
  File "/builds/9CsNzyCB/7/root/qcraft/bazel-bin/offboard/simulation/tools/regression/regression_helper_main.runfiles/com_qcraft/offboard/simulation/tools/regression/rpc_service/qsim_result_rpc.py", line 27, in compare_job_result
    metric = stub.GetJobCompareMetrics(request)
  File "/builds/9CsNzyCB/7/root/qcraft/bazel-bin/offboard/simulation/tools/regression/regression_helper_main.runfiles/com_github_grpc_grpc/src/python/grpcio/grpc/_channel.py", line 946, in __call__
    return _end_unary_response_blocking(state, call, False, None)
  File "/builds/9CsNzyCB/7/root/qcraft/bazel-bin/offboard/simulation/tools/regression/regression_helper_main.runfiles/com_github_grpc_grpc/src/python/grpcio/grpc/_channel.py", line 849, in _end_unary_response_blocking
    raise _InactiveRpcError(state)
grpc._channel._InactiveRpcError: <_InactiveRpcError of RPC that terminated with:
	status = StatusCode.UNKNOWN
	details = "Unexpected error in RPC handling"
	debug_error_string = "{"created":"@1642559035.377663377","description":"Error received from peer ipv4:52.34.189.4:3401","file":"external/com_github_grpc_grpc/src/core/lib/surface/call.cc","file_line":1070,"grpc_message":"Unexpected error in RPC handling","grpc_status":2}"
>
Running after_script
00:01
Running after script...
$ rsync -a --prune-empty-dirs --include '*/' --include 'test.xml' --exclude '*' bazel-out/k8-opt/testlogs .
$ echo "================= CI_PIPELINE_ID $CI_PIPELINE_ID"; echo "================= CI_JOB_ID ID $CI_JOB_ID"; echo "================= CI Runner `cat /etc/host_hostname`"; echo "================= Docker Instance `hostname`"; echo "================= Current Path `pwd`"; echo "================= PULL MAP $PULL_MAP"; echo "================= CI_MERGE_REQUEST_LABELS $CI_MERGE_REQUEST_LABELS";
================= CI_PIPELINE_ID 108300
================= CI_JOB_ID ID 2360385
================= CI Runner us-edge-05
================= Docker Instance runner-9csnzycb-project-4-concurrent-745gdd
================= Current Path /builds/9CsNzyCB/7/root/qcraft
================= PULL MAP false
================= CI_MERGE_REQUEST_LABELS 
```



job忽然失败，根本原因是ack机器重启

```
bazel-out/k8-opt/bin/onboard/proto/perception.pb.h:6693:3: note: 'obstacle_centers_deprecated' has been explicitly marked deprecated here
  PROTOBUF_DEPRECATED const ::PROTOBUF_NAMESPACE_ID::RepeatedPtrField< ::qcraft::Vec2dProto >&
  ^
external/com_google_protobuf/src/google/protobuf/port_def.inc:305:45: note: expanded from macro 'PROTOBUF_DEPRECATED'
# define PROTOBUF_DEPRECATED __attribute__((deprecated))
                                            ^
3 warnings generated.
INFO: From Action onboard/camera/codec/libgpujpeg_huffman_encoder_cuda.so:
ptxas warning : Value of threads per SM for entry _Z44gpujpeg_huffman_encoder_serialization_kernelP15gpujpeg_segmentiPKhPh is out of range. .minnctapersm will be ignored
INFO: From Action onboard/camera/codec/libgpujpeg_huffman_decoder_cuda.so:
ptxas warning : Value of threads per SM for entry _Z37gpujpeg_huffman_decoder_decode_kernelILb0ELi192EEv27gpujpeg_huffman_gpu_decoderP17gpujpeg_componentP15gpujpeg_segmentiiPhPKmPs is out of range. .minnctapersm will be ignored
ptxas warning : Value of threads per SM for entry _Z37gpujpeg_huffman_decoder_decode_kernelILb1ELi192EEv27gpujpeg_huffman_gpu_decoderP17gpujpeg_componentP15gpujpeg_segmentiiPhPKmPs is out of range. .minnctapersm will be ignored
[5,702 / 5,713] Compiling onboard/autonomy_state/autonomy_state_manager.cc; 46s remote-cache, processwrapper-sandbox ... (8 actions running)
[5,703 / 5,713] Compiling onboard/autonomy/thread_module_manager.cc; 78s remote-cache, processwrapper-sandbox ... (6 actions running)
[5,704 / 5,713] Compiling onboard/lite/launch_autonomy_main.cc; 134s remote-cache, processwrapper-sandbox ... (4 actions running)
[5,705 / 5,713] Action onboard/nets/custom_ops/libcustom_op.so; 196s remote-cache, processwrapper-sandbox
[5,705 / 5,713] Action onboard/nets/custom_ops/libcustom_op.so; 274s remote-cache, processwrapper-sandbox
Running after_script
00:00
Cleaning up file based variables
00:00
ERROR: Job failed: pod "runner-sjywypqz-project-4-concurrent-3j89fq" status is "Failed"
```





```
10:53:54 PM  CPU    %usr   %nice    %sys %iowait    %irq   %soft  %steal  %guest  %gnice   %idle
10:53:55 PM  all   22.35    0.08   11.56    3.10    0.00    1.02    0.00    0.00    0.00   61.90
10:53:55 PM    0   56.57    0.00   12.12    6.06    0.00    0.00    0.00    0.00    0.00   25.25
10:53:55 PM    1   12.24    0.00    9.18    1.02    0.00    1.02    0.00    0.00    0.00   76.53
10:53:55 PM    2   26.00    0.00   11.00    5.00    0.00    0.00    0.00    0.00    0.00   58.00
10:53:55 PM    3   38.78    0.00   10.20    1.02    0.00    0.00    0.00    0.00    0.00   50.00
10:53:55 PM    4    8.25    0.00   11.34    6.19    0.00    0.00    0.00    0.00    0.00   74.23
10:53:55 PM    5   19.19    0.00    9.09    6.06    0.00    0.00    0.00    0.00    0.00   65.66
10:53:55 PM    6   61.62    0.00    5.05    0.00    0.00    0.00    0.00    0.00    0.00   33.33
10:53:55 PM    7   33.33    0.00   16.16    0.00    0.00    1.01    0.00    0.00    0.00   49.49
10:53:55 PM    8   13.40    0.00   11.34    2.06    0.00    0.00    0.00    0.00    0.00   73.20
10:53:55 PM    9   41.41    0.00    8.08    0.00    0.00    1.01    0.00    0.00    0.00   49.49
10:53:55 PM   10   27.55    0.00   11.22    0.00    0.00    0.00    0.00    0.00    0.00   61.22
10:53:55 PM   11    6.12    0.00   12.24    0.00    0.00    0.00    0.00    0.00    0.00   81.63
10:53:55 PM   12   32.32    0.00   12.12    4.04    0.00    0.00    0.00    0.00    0.00   51.52
10:53:55 PM   13   16.33    0.00    9.18    0.00    0.00    0.00    0.00    0.00    0.00   74.49
10:53:55 PM   14    6.19    0.00   14.43    2.06    0.00    0.00    0.00    0.00    0.00   77.32
10:53:55 PM   15    7.07    0.00   11.11    1.01    0.00    1.01    0.00    0.00    0.00   79.80
10:53:55 PM   16   20.41    0.00   11.22    0.00    0.00    0.00    0.00    0.00    0.00   68.37
10:53:55 PM   17   15.00    0.00    9.00    0.00    0.00    0.00    0.00    0.00    0.00   76.00
10:53:55 PM   18   52.53    0.00    8.08    0.00    0.00    0.00    0.00    0.00    0.00   39.39
10:53:55 PM   19    6.06    0.00   11.11    1.01    0.00    0.00    0.00    0.00    0.00   81.82
10:53:55 PM   20   15.31    0.00    8.16    1.02    0.00    0.00    0.00    0.00    0.00   75.51
10:53:55 PM   21   23.23    0.00    6.06    0.00    0.00    1.01    0.00    0.00    0.00   69.70
10:53:55 PM   22    5.21    0.00   16.67    2.08    0.00    0.00    0.00    0.00    0.00   76.04
10:53:55 PM   23    5.10    0.00   12.24    0.00    0.00    0.00    0.00    0.00    0.00   82.65
10:53:55 PM   24   16.00    0.00   15.00    1.00    0.00    0.00    0.00    0.00    0.00   68.00
10:53:55 PM   25   35.35    0.00    5.05    1.01    0.00    0.00    0.00    0.00    0.00   58.59
10:53:55 PM   26   13.27    0.00   12.24    0.00    0.00    0.00    0.00    0.00    0.00   74.49
10:53:55 PM   27   17.53    0.00   11.34    0.00    0.00    0.00    0.00    0.00    0.00   71.13
10:53:55 PM   28   11.00    0.00    9.00    2.00    0.00    2.00    0.00    0.00    0.00   76.00
10:53:55 PM   29   61.62    0.00    9.09    9.09    0.00    0.00    0.00    0.00    0.00   20.20
10:53:55 PM   30   40.82    1.02    8.16    3.06    0.00    0.00    0.00    0.00    0.00   46.94
10:53:55 PM   31    5.21    1.04   20.83   27.08    0.00    0.00    0.00    0.00    0.00   45.83
10:53:55 PM   32   24.49    0.00   13.27    3.06    0.00    0.00    0.00    0.00    0.00   59.18
10:53:55 PM   33    5.05    0.00   16.16    5.05    0.00    0.00    0.00    0.00    0.00   73.74
10:53:55 PM   34   11.22    0.00   16.33   10.20    0.00    0.00    0.00    0.00    0.00   62.24
10:53:55 PM   35   16.16    0.00   11.11    1.01    0.00    0.00    0.00    0.00    0.00   71.72
10:53:55 PM   36   12.37    0.00   16.49    4.12    0.00    0.00    0.00    0.00    0.00   67.01
10:53:55 PM   37    7.07    0.00   15.15    6.06    0.00    2.02    0.00    0.00    0.00   69.70
10:53:55 PM   38   20.20    0.00   13.13    9.09    0.00    0.00    0.00    0.00    0.00   57.58
10:53:55 PM   39   14.14    0.00   12.12    4.04    0.00    0.00    0.00    0.00    0.00   69.70
10:53:55 PM   40    9.18    0.00   12.24    1.02    0.00    0.00    0.00    0.00    0.00   77.55
10:53:55 PM   41   32.32    0.00    9.09    1.01    0.00    0.00    0.00    0.00    0.00   57.58
10:53:55 PM   42   15.46    1.03   13.40    7.22    0.00    0.00    0.00    0.00    0.00   62.89
10:53:55 PM   43   12.00    0.00    9.00    4.00    0.00    0.00    0.00    0.00    0.00   75.00
10:53:55 PM   44   21.43    0.00   11.22    0.00    0.00    0.00    0.00    0.00    0.00   67.35
10:53:55 PM   45    4.00    0.00   13.00    4.00    0.00    0.00    0.00    0.00    0.00   79.00
10:53:55 PM   46    8.08    0.00   13.13    1.01    0.00    0.00    0.00    0.00    0.00   77.78
10:53:55 PM   47   29.00    0.00   17.00    1.00    0.00    0.00    0.00    0.00    0.00   53.00
10:53:55 PM   48   97.00    0.00    3.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00
10:53:55 PM   49    1.03    0.00   15.46   11.34    0.00    0.00    0.00    0.00    0.00   72.16
10:53:55 PM   50   38.00    0.00   12.00    2.00    0.00    0.00    0.00    0.00    0.00   48.00
10:53:55 PM   51    6.06    0.00   13.13    3.03    0.00    0.00    0.00    0.00    0.00   77.78
10:53:55 PM   52   24.75    0.00    6.93    0.99    0.00    0.99    0.00    0.00    0.00   66.34
10:53:55 PM   53   45.00    0.00    6.00    2.00    0.00    0.00    0.00    0.00    0.00   47.00
10:53:55 PM   54   19.57    0.00   20.65    4.35    0.00   26.09    0.00    0.00    0.00   29.35
10:53:55 PM   55   25.77    0.00   13.40    2.06    0.00    0.00    0.00    0.00    0.00   58.76
10:53:55 PM   56   16.33    0.00   12.24    9.18    0.00    0.00    0.00    0.00    0.00   62.24
10:53:55 PM   57    8.16    0.00    8.16    2.04    0.00    0.00    0.00    0.00    0.00   81.63
10:53:55 PM   58   21.35    0.00   16.85    1.12    0.00   29.21    0.00    0.00    0.00   31.46
10:53:55 PM   59   20.20    0.00    7.07    2.02    0.00    0.00    0.00    0.00    0.00   70.71
10:53:55 PM   60   28.28    0.00   12.12    5.05    0.00    0.00    0.00    0.00    0.00   54.55
10:53:55 PM   61   26.26    0.00    9.09    4.04    0.00    1.01    0.00    0.00    0.00   59.60
10:53:55 PM   62   25.77    0.00   13.40    5.15    0.00    0.00    0.00    0.00    0.00   55.67
10:53:55 PM   63   28.57    0.00   13.27    2.04    0.00    0.00    0.00    0.00    0.00   56.12

31 & 29


sim-pipe248058spot 11
sim-pipe248252spot

```





## 结尾
唉，尴尬

![狗头的赞赏码.jpg](https://i.loli.net/2020/08/27/BFHNyfpx3EsIDUG.jpg)