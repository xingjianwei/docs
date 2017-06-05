# TiDB和CrDB编译安装

## CockroachDB macosX编译

beta-20170423后，编译变化较大。

clang和gcc共存时，使用gcc导致google protobuf 和 rocksdb编译出现错误。ld: symbol(s) not found for architecture x86_64。
所以必须使用clang，设置`$PATH=/usr/bin:$PATH`。

## CockroachDB docker编译

[编译方法介绍](https://github.com/cockroachdb/cockroach/tree/master/build)
### 下载镜像
https://github.com/cockroachdb/cockroach/blob/master/build/deploy/Dockerfile 所需镜像

运行部署基础镜像：
debian:8.7

开发编译基础镜像：
https://github.com/cockroachdb/cockroach/blob/master/build/Dockerfile文件中所需镜像

编译环境基础镜像：
ubuntu:xenial-20170214

```
cd build/
./builder.sh init
```

### 开发环境

```
docker tag cockroachdb/builder  cockroachdb/builder:$verson
(docker tag cockroachdb/builder  cockroachdb/builder:20170422-212842)
取自pkg/acceptance/cluster/localcluster.go中的builderTag       = "20170422-212842"
sh builder.sh

```
可以进入开发环境。

```
make TYPE=release build
docker build --tag=cockroachdb/cockroach deploy
docker run  cockroachdb/cockroach  version
```

### 编译成可执行镜像

```
./build-docker-deploy.sh
docker run  cockroachdb/cockroach  version
docker run  -it  --entrypoint=""  cockroachdb/cockroach    bash
```

## TiDB编译
### 下载golang docker镜像

```docker pull debian:jessie
docker pull buildpack-deps:jessie-curl
docker pull buildpack-deps:jessie-scm
docker pull golang:1.8.0
```
### 构建rust镜像
Dockerfile如下：

```
FROM debian:jessie
USER root
ENV USER root
ENV DEBIAN_FRONTEND=noninteractive

RUN echo "deb http://mirrors.aliyun.com/debian/ jessie main non-free contrib\ndeb http://mirrors.aliyun.com/debian/ jessie-proposed-updates main non-free contrib\ndeb-src http://mirrors.aliyun.com/debian/ jessie main non-free contrib\ndeb-src http://mirrors.aliyun.com/debian/ jessie-proposed-updates main non-free contrib" > /etc/apt/sources.list \
&& apt-get update  \
&& apt-get install ca-certificates \
 curl \ 
 gcc-4.8 g++-4.8 \
 make \
 zlib1g-dev libbz2-dev libsnappy-dev libgflags-dev liblz4-dev \
 libc6-dev \
 -qqy \
--no-install-recommends \
&& update-alternatives --install /usr/bin/gcc gcc /usr/bin/gcc-4.8 20 \
&&  update-alternatives --install /usr/bin/g++ g++ /usr/bin/g++-4.8 20 \
&& update-alternatives --config gcc \
&&  update-alternatives --config g++ \
&& ln -fs /usr/bin/gcc /usr/bin/cc \
&& ln -fs /usr/bin/g++ /usr/bin/c++ \
&& rm -rf /var/lib/apt/lists/*


ENV RUST_ARCHIVE=rust-nightly-x86_64-unknown-linux-gnu.tar.gz
#ENV RUST_DOWNLOAD_URL=https://static.rust-lang.org/dist/$RUST_ARCHIVE

RUN mkdir -p /rust
WORKDIR /rust
COPY $RUST_ARCHIVE .

RUN tar -C /rust -xzf $RUST_ARCHIVE --strip-components=1 \
    && rm $RUST_ARCHIVE \
    && ./install.sh
COPY cargo-config /root/.cargo/config
```
运行命令：
`docker build -t beagle/rust-gcc4.8-20170324 -f Dockerfile  . `

### 编译rocksdb：


```FROM  beagle/rust-gcc4.8-20170324
ADD . /rocksdb

RUN cd /rocksdb && \
    make shared_lib && \
    make install-shared && \
    ldconfig && \
    make clean
```
运行命令：
`docker build -t beagle/rust-gcc-4.8-rocksdb-20170324 -f Dockerfile  . `

### 编译TiKV：

```
FROM beagle/rust-gcc4.8-20170324

ADD . /tikv

RUN cd /tikv && \
    cargo build --release && \
    cp -f target/release/tikv-server /tikv-server && \
    cargo clean

EXPOSE 20160

ENTRYPOINT ["/tikv-server"]

```
运行命令：
`docker build -t beagle/tikv-20170324 -f Dockerfile  . `

编译出错：

```Compiling nix v0.5.0-pre (https://github.com/carllerche/nix-rust?rev=c4257f8a76#c4257f8a)
error[E0591]: `extern "C" fn(*mut std::boxed::Box<std::ops::FnMut() -> isize>) -> i32 {sched::clone::callback}` is zero-sized and can't be transmuted to `*const extern "C" fn(*const std::boxed::Box<std::ops::FnMut() -> isize>) -> i32`
   --> /root/.cargo/git/checkouts/nix-rust-d606302ceff0dca5/c4257f8/src/sched.rs:190:20
    |
190 |         ffi::clone(mem::transmute(callback), ptr as *mut c_void, flags, &mut cb)
    |                    ^^^^^^^^^^^^^^
    |
note: cast with `as` to a pointer instead
   --> /root/.cargo/git/checkouts/nix-rust-d606302ceff0dca5/c4257f8/src/sched.rs:190:20
    |
190 |         ffi::clone(mem::transmute(callback), ptr as *mut c_void, flags, &mut cb)
    |                    ^^^^^^^^^^^^^^
```


