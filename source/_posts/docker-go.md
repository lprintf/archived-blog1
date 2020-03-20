---
title: vscode-go语言开发容器构建
categories:
  - 开发
tags:
  - docker
  - go
  - vscode
type: projects
date: 2020-03-20 06:34:35
---

(本文供有vscode容器开发基础的人食用.)

这几天陆续搭建了很多开发容器,其中遇到的坑最多的便是go语言容器了,今天在这里总结一下.

国内使用vscode官网go语言容器配置会遇到不少问题,具体修复操作如下.

## 开始

- 国内镜像加速配置
```s
sed -i s@/deb.debian.org/@/mirrors.aliyun.com/@g /etc/apt/sources.list \ 
apt-get -o Acquire::Check-Valid-Until=false update \
```
第一句常使用sed命令修改apt配置文件
第二句解决容器时区不匹配造成的证书过期问题

- 服务器无法访问

报错如下:

`Get "https://proxy.golang.org/golang.org/x/tools/gopls/@v/list": dial tcp 172.217.27.145:443: connect: connection refused`

容器默认不会使用主机代理,通过容器内声明代理服务器可解决.

`export http_proxy=http://host.docker.internal:1080`

- git TLS解码错误

`error: RPC failed; curl 56 GnuTLS recv error (-9): Error decoding the received TLS packet.`

这里猜测git若先于openssl安装会使用TLS做https的安全协议,调换git安装顺序后侥幸解决.

然而排错过程中还是偶尔报该错误,故猜测是数据传输时出现错误,遇到便重试.

- golang安装脚本bug

日志如下,报错为空,实际上为缺少BINARY=golang-ci-lint变量的缘故.其中RUN命令可供参考.

```log
[2020-03-20T06:20:54.768Z] [PID 13888] [167755 ms] cd .
git clone -- https://github.com/stamblerre/gocode /tmp/gotools/src/github.com/stamblerre/gocode
[2020-03-20T06:23:56.527Z] [PID 13888] [349515 ms] cd /tmp/gotools/src/github.com/stamblerre/gocode
git submodule update --init --recursive
cd /tmp/gotools/src/github.com/stamblerre/gocode
[2020-03-20T06:23:56.562Z] [PID 13888] [349550 ms] 
git show-ref
cd /tmp/gotools/src/github.com/stamblerre/gocode
git submodule update --init --recursive
[2020-03-20T06:24:00.884Z] [PID 13888] [353872 ms] golangci/golangci-lint info checking GitHub for latest tag
[2020-03-20T06:24:00.928Z] [PID 13888] [353916 ms] 
[2020-03-20T06:24:03.145Z] [PID 13888] [356133 ms] golangci/golangci-lint info found version: 1.24.0 for v1.24.0/linux/amd64
[2020-03-20T06:36:46.049Z] [PID 13888] [1119037 ms] The command '/bin/sh -c sed -i s@/deb.debian.org/@/mirrors.aliyun.com/@g /etc/apt/sources.list     && apt-get -o Acquire::Check-Valid-Until=false update     && apt-get -y install --no-install-recommends apt-utils dialog 2>&1     && apt-get -y install openssh-client less iproute2 procps lsb-release git     && mkdir -p /tmp/gotools     && cd /tmp/gotools     && export http_proxy=http://host.docker.internal:1080     && GOPATH=/tmp/gotools GO111MODULE=on go get -v golang.org/x/tools/gopls@latest 2>&1     && GOPATH=/tmp/gotools GO111MODULE=on go get -v         honnef.co/go/tools/...@latest         golang.org/x/tools/cmd/gorename@latest         golang.org/x/tools/cmd/goimports@latest         golang.org/x/tools/cmd/guru@latest         golang.org/x/lint/golint@latest         github.com/mdempsky/gocode@latest         github.com/cweill/gotests/...@latest         github.com/haya14busa/goplay/cmd/goplay@latest         github.com/sqs/goreturns@latest         github.com/josharian/impl@latest
  github.com/davidrjenni/reftools/cmd/fillstruct@latest         github.com/uudashr/gopkgs/v2/cmd/gopkgs@latest          github.com/ramya-rao-a/go-outline@latest          github.com/acroca/go-symbols@latest          github.com/godoctor/godoctor@latest          github.com/rogpeppe/godef@latest          github.com/zmb3/gogetdoc@latest         github.com/fatih/gomodifytags@latest          github.com/mgechev/revive@latest          github.com/go-delve/delve/cmd/dlv@latest 2>&1     && GOPATH=/tmp/gotools go get -v github.com/alecthomas/gometalinter 2>&1     && GOPATH=/tmp/gotools go get -x -d github.com/stamblerre/gocode 2>&1     && GOPATH=/tmp/gotools go build -o gocode-gomod github.com/stamblerre/gocode     && mv /tmp/gotools/bin/* /usr/local/bin/     && mv gocode-gomod /usr/local/bin/     && curl -sSfL https://raw.githubusercontent.com/golangci/golangci-lint/master/install.sh | sh -s -- -b /usr/local/bin 2>&1     && groupadd --gid $USER_GID $USERNAME     && useradd -s /bin/bash --uid $USE
R_UID --gid $USER_GID -m $USERNAME     && apt-get install -y sudo     && echo $USERNAME ALL=\(root\) NOPASSWD:ALL > /etc/sudoers.d/$USERNAME     && chmod 0440 /etc/sudoers.d/$USERNAME     && apt-get autoremove -y     && apt-get clean -y     && rm -rf /var/lib/apt/lists/* /tmp/gotools' returned a non-zero code: 18
[2020-03-20T06:36:46.050Z] [PID 13888] [1119038 ms] Failed: Building an image from the Dockerfile (this can take a while).
[2020-03-20T06:36:46.052Z] [PID 13888] [1119040 ms] Command failed: C:\Program Files\Docker\Docker\resources\bin\docker.exe build -f d:\code\go\.devcontainer\Dockerfile -t vsc-go-92c6f27f59ab08d675ee8547f2079099 d:\code\go\.devcontainer

```

golang安装命令管道符后紧跟BINARY=golang-ci-lint可解决:

`curl -sSfL https://raw.githubusercontent.com/golangci/golangci-lint/master/install.sh | BINARY=golang-ci-lint sh -s -- -b /usr/local/bin 2>&1`

## 总结

这怕是目前我遇到的最棘手的开发容器了.要不是手头有几本书消遣,这么漫长的重试等待时间怕是很影响我心态.也不正是这些困难,才有此刻拨云见日的欣喜么.此外,有些疑问尚未解决,譬如上面所说的`git clone`命令间歇故障.希望能在评论区得到大佬的指教.
