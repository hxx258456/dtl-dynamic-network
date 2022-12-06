# dtl-dynamic-network

fabric动态添加节点，组织，orderer示例

## 初始网络规划

|    节点    |   宿主机ip    |         hosts          | port |
| :--------: | :-----------: | :--------------------: | :--: |
|  orderer0  | 192.168.0.170 |  orderer0.example.com  | 7500 |
| org1-peer0 | 192.168.0.172 | peer0.org1.example.com | 7051 |
| ca-orderer | 192.168.0.170 | ca_orderer.example.com | 9054 |
|  ca-org1   | 192.168.0.170 |  ca_org1.example.com   | 7054 |

### 拉取镜像和二进制文件

```shell
git clone https://gitlab.sxtxhy.com/fabric/fabric-gm-bin.git
# 将fabric-gm-bin文件夹中bin目录下的可执行文件添加到path

docker pull 39.100.227.166:5000/gcbaas-gm/fabric-tools:latest
docker pull 39.100.227.166:5000/gcbaas-gm/fabric-peer:latest
docker pull 39.100.227.166:5000/gcbaas-gm/fabric-orderer:latest
docker pull 39.100.227.166:5000/gcbaas-gm/fabric-ccenv:latest
docker pull 39.100.227.166:5000/gcbaas-gm/fabric-baseos:latest
docker pull 39.100.227.166:5000/gcbaas-gm/fabric-ca:latest
```

### 生成组织身份信息

```shell
sh -x scripts/registerEnrollUser.sh
```

### 生成创世块

```shell
# pwd config目录下
configtxgen -profile TwoOrgsOrdererGenesis -channelID system-channel -outputBlock ../system-genesis-block/genesis.block
```

