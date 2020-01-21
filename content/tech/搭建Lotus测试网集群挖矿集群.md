+++
title = "搭建Lotus测试网集群挖矿集群"
date = "2020-01-21T14:03:47+08:00"
tags = ["ipfs", "decentralized", "Filecoin", "lotus", "矿工"]
slug = "big-miner-filecoin-cluster"
description = "Filecoin挖矿集群"
gitinfo = true
displayCopyright = false
Categories =  ["ipfs", "decentralized", "Filecoin", "lotus", "矿工"]
+++


# 硬件要求[^1] 

## 配置示例

 以下设置是在Lotus上密封32个GiB扇区的最小示例：
 2 TB硬盘空间。
 8核CPU
 128 GiB的RAM

 请注意，1GB扇区并不需要那么高的规格，但是随着我们提高32GB扇区密封性能，有可能将其删除。

## 基准GPU

 GPU是获得块状奖励的必备条件。这些已被证实可以足够快地生成SNARK以便成功地在Lotus Testnet上挖掘块的方法如下。

 GeForce RTX 2080 Ti
 GeForce RTX 2080超级
 GeForce RTX 2080
 GeForce GTX 1080 Ti
 GeForce GTX 1080
 GeForce GTX 1060

# 编译安装[^2] 

## 准备
    ```
    sudo apt update
    sudo apt install mesa-opencl-icd ocl-icd-opencl-dev
    ```

## 安装支持软件包 
    ```
    sudo add-apt-repository ppa:longsleep/golang-backports
    sudo apt update
    sudo apt install golang-go gcc git bzr jq pkg-config mesa-opencl-icd ocl-icd-opencl-dev
    ```

## 克隆
    ```
    git clone https://github.com/filecoin-project/lotus.git
    cd lotus/
```

## 编译和安装
    ```
    make clean && make all
    sudo make install
    ```

# 配置

    Lotus 的测试网的源代码编译只有三个文件 lotus， lotus-storage-miner， lotus-seal-worker。 其中 lotus 和 lotus-storage-miner 分别是区块链节点和矿工节点的启动程序， lotus-seal-worker 是数据密封节点的启动程序。在没有高配机器的情况下，我们可以使用 一台 Miner 节点 + N 台 Worker 节点来实现自己 上排行榜 的需求。

    下面我们就看看具体怎么搭建这个集群。

## 配置方案

    配置方案：Master Miner 节点 x 1 + seal-worker 节点 x N
    Master Miner 节点：使用一台带 GPU 的大容量机器作为 miner 主节点，这机器主要做的工作是 add pieces 和提交 PoSt proving

    Seal Worker 节点： 多核 CPU 配上高速企业硬盘，以便可以增加 sealing 速度， 如果 CPU 的能力不强，可以使用 GPU 代替，效果更好。

## 配置步骤
    一般来说linux根分区不会很大，这里会用到40G左右的临时文件，如果默认TMPDIR空间不够，需要修改$TMPDIR路径
    在lotus节点起来和完成同步后需要获取代币和创建矿工
    具体参考：https://docs.lotu.sh/en+join-testnet
    中国区节点需要这是代理路径：
    ```
    export IPFS_GATEWAY="https://proof-parameters.s3.cn-south-1.jdcloud-oss.com/ipfs/"
    ```

    在 master 矿工初始化之后，也就是在下面的脚本运行之后
    lotus-storage-miner init --actor=t01849 --owner=t3v3gwdujm35f6vwrv6x7utimbhu7s64hvto6m327xscsoyraj5dyzxom2bthormfd7e3imqnpabp7hp55tlfq

1. 修改矿工配置文档 vim .lotusstorage/config.toml

    [API]
    ListenAddress = "/ip4/{IP}/tcp/2345/http"
    Timeout = "30s"
    这里的 {IP} 是当前 master 节点的 ip 地址，默认是 127.0.0.1 这里你需要把它改成局域网或者公网 IP(如果你有静态 ip 的话)， 如果你搞不清你是否需要外网访问，你就配置成 0.0.0.0

    如果所有 worker 节点都在一个网络的话，建议使用内网 IP，如果需要大量外网机器同时挖，则需要填写公网 IP

2. 启动 master 矿工

    BELLMAN_CUSTOM_GPU="GeForce RTX 2070 SUPER:2560" lotus-storage-miner run
    前面的变量是你 GPU 的配置信息，具体配置详情, 请参考： https://docs.lotu.sh/en+hardware-mining

3. 复制必须文件到其它节点
   
   拷贝 master 节点的 /var/tmp/filecoin-proof-parameters, .lotusstorage/api 和 .lotusstorage/token 到 worker 节点， 同时拷贝 lotus-seal-worker 到 worker 节点

4. 运行 worker 节点

    nohup ./lotus-seal-worker run > miner.log &
    如果你的 worker 有 GPU 的话，则你可以启用 GPU 做复制证明

    BELLMAN_CUSTOM_GPU="GeForce RTX 2070 SUPER:2560" nohup ./lotus-seal-worker run > miner.log &
    启动成功会看到类似的信息

    INFO main lotus-seal-worker/sub.go:69 Waiting for new task
    这时你在 master 节点运行 lotus-storage-miner info 应该可以看到 remote worker

    ![remote-worker](/images/image-aca38f34a0e1c92094a2332b4bb8c38a.png)

5. 接下来按照步骤(4)的方法启动 N 个 Worker 节点

    到此，整个集群就开始运行了，等你的 Miner 节点准备好密封数据(PreCommit)，Worker 就会开始拉取数据计算了，计算完之后会 Push 回 Miner 节点。

    ![push-data](/images/image-f0941695a85fdd0c2e95a556bc57e5f6.png)

    这里需要注意的是，Worker 计算完之后会 Push 11GB 数据回去，也就是说复制证明会计算 10 次，Master Miner 节点需要准备的磁盘空间与实际存储数据的 比值是 1:11，所以你得准备足够多的磁盘空间，以免磁盘空间不足导致矿工程序退出。不过这些 cache 数据再你完成 Proving 之后会删除大部分，只保留 2.1GB。


[^1]: 硬件配置参考：https://docs.lotu.sh/en+hardware-mining
[^2]: 官方Ubuntu安装教程：https://docs.lotu.sh/en+install-lotus-ubuntu
