## 编译准备

### 1. 编译安装clang 15

github下载llvm-project

git clone https://github.com/llvm/llvm-project.git

进入源码目录，执行以下命令

mkdir build

cd build

cmake -G "Unix Makefiles" -DCMAKE_BUILD_TYPE=Release  -DLLVM_ENABLE_PROJECTS="clang;clang-tools-extra"  -DCMAKE_INSTALL_PREFIX=/usr/local/clang  -DLLVM_ENABLE_ASSERTIONS=On ../llvm

make -j8

make install

### 2. 准备高版本gcc

yum install epel-release -y

yum clean all && yum makecache

yum install devtoolset-8-gcc -y

搜索原有环境上的gcc版本并删除

yum search gcc

yum remove gcc

软链接高版本gcc到/usr/bin中

ln -s /opt/rh/devtoolset-8/root/usr/bin/gcc /usr/bin/gcc

ln -s /opt/rh/devtoolset-8/root/usr/bin/g++ /usr/bin/g++

### 3. 安装python3和对应的scons

sudo yum install python36-scons.noarch

### 4. 相关依赖

以上依赖仅限于当前环境遇到的欠缺依赖，并不代表完全一样，编译时如果欠缺什么库环境，需要手动安装

````bash
install \
    boost-python36-devel \
    bzip2 \
    clang-analyzer \
    cmake3 \
    CUnit-devel \
    e2fsprogs \
    file \
    flex \
    fuse3 \
    fuse3-devel \
    gcc \
    gcc-c++ \
    git \
    golang \
    graphviz \
    hwloc-devel \
    ipmctl \
    java-1.8.0-openjdk \
    json-c-devel \
    lcov \
    libaio-devel \
    libcmocka-devel \
    libevent-devel \
    libipmctl-devel \
    libiscsi-devel \
    libtool \
    libtool-ltdl-devel \
    libunwind-devel \
    libuuid-devel \
    libyaml-devel \
    Lmod \
    lz4-devel \
    make \
    maven \
    meson \
    ndctl \
    ninja-build \
    numactl \
    numactl-devel \
    openmpi3-devel \
    openssl-devel \
    patch \
    patchelf \
    pciutils \
    python3-pip \
    python36-defusedxml \
    python36-devel \
    python36-distro \
    python36-junit_xml \
    python36-tabulate \
    python36-pyxattr \
    python36-PyYAML \
    python36-scons \
    sg3_utils \
    sudo \
    valgrind-devel \
    yasm
````







## 常见问题总结

git clone容易失败，一般是由于网络的原因，多下载几次就行了，实在不行手动下载到对应的位置。

删除

