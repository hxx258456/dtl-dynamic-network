dtl-dynamic-network

fabric动态添加节点，组织，orderer示例



## 初始网络规划

|    节点    |   宿主机ip    |         hosts          | port |
| :--------: | :-----------: | :--------------------: | :--: |
|  orderer0  | 192.168.0.170 |  orderer0.example.com  | 7050 |
| org1-peer0 | 192.168.0.170 | peer0.org1.example.com | 8051 |
| org2-peer0 | 192.168.0.172 | peer0.org2.example.com | 9051 |

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
cryptogen generate --config=./configtx/crypto-config.yaml --output="organizations"
```

### 生成创世块

```shell
# pwd config目录下
configtxgen -profile TwoOrgsOrdererGenesis -channelID system-channel -outputBlock ../system-genesis-block/genesis.block
```

### 生成通道文件

```shell
configtxgen -profile TwoOrgsChannel -outputCreateChannelTx ../channel-artifacts/mychannel.tx -channelID mychannel
```

### org1锚节点定义

```shell
configtxgen -profile TwoOrgsChannel -outputAnchorPeersUpdate ../channel-artifacts/Org1MSPanchors.tx -channelID mychannel -asOrg Org1MSP
```

### org2锚节点定义

```shell
configtxgen -profile TwoOrgsChannel -outputAnchorPeersUpdate ../channel-artifacts/Org2MSPanchors.tx -channelID mychannel -asOrg Org2MSP
```



### 创建通道

```shell
export FABRIC_CFG_PATH=$PWD/configtx/
export CORE_PEER_LOCALMSPID="Org1MSP"
export CORE_PEER_TLS_ENABLED=true
export CORE_PEER_TLS_ROOTCERT_FILE=${PWD}/organizations/peerOrganizations/org1.example.com/peers/peer0.org1.example.com/tls/ca.crt
export CORE_PEER_MSPCONFIGPATH=${PWD}/organizations/peerOrganizations/org1.example.com/users/Admin@org1.example.com/msp
export CORE_PEER_ADDRESS=peer0.org1.example.com:8051
export ORDERER_CA=${PWD}/organizations/ordererOrganizations/example.com/orderers/orderer.example.com/msp/tlscacerts/tlsca.example.com-cert.pem
peer channel create -o orderer.example.com:7050 -c mychannel --ordererTLSHostnameOverride orderer.example.com -f ./channel-artifacts/mychannel.tx --outputBlock ./channel-artifacts/mychannel.block --tls --cafile $ORDERER_CA

```

### peer0-org1加入通道

```shell
export FABRIC_CFG_PATH=$PWD/configtx/
export CORE_PEER_LOCALMSPID="Org1MSP"
export CORE_PEER_TLS_ENABLED=true
export CORE_PEER_TLS_ROOTCERT_FILE=${PWD}/organizations/peerOrganizations/org1.example.com/peers/peer0.org1.example.com/tls/ca.crt
export CORE_PEER_MSPCONFIGPATH=${PWD}/organizations/peerOrganizations/org1.example.com/users/Admin@org1.example.com/msp
export CORE_PEER_ADDRESS=peer0.org1.example.com:8051
export ORDERER_CA=${PWD}/organizations/ordererOrganizations/example.com/orderers/orderer.example.com/msp/tlscacerts/tlsca.example.com-cert.pem
peer channel join -b ./channel-artifacts/mychannel.block

#更新通道节点
peer channel update -o localhost:7050 --ordererTLSHostnameOverride orderer.example.com -c mychannel -f ./channel-artifacts/Org1MSPanchors.tx --tls --cafile $ORDERER_CA
```

### peer-org2加入通道

```shell
export FABRIC_CFG_PATH=$PWD/configtx/
export CORE_PEER_LOCALMSPID="Org2MSP"
export CORE_PEER_TLS_ENABLED=true
export CORE_PEER_TLS_ROOTCERT_FILE=${PWD}/organizations/peerOrganizations/org2.example.com/peers/peer0.org2.example.com/tls/ca.crt
export CORE_PEER_MSPCONFIGPATH=${PWD}/organizations/peerOrganizations/org2.example.com/users/Admin@org2.example.com/msp
export CORE_PEER_ADDRESS=peer0.org2.example.com:9051
export ORDERER_CA=${PWD}/organizations/ordererOrganizations/example.com/orderers/orderer.example.com/msp/tlscacerts/tlsca.example.com-cert.pem
peer channel join -b ./channel-artifacts/mychannel.block

#更新通道节点
peer channel update -o localhost:7050 --ordererTLSHostnameOverride orderer.example.com -c mychannel -f ./channel-artifacts/Org2MSPanchors.tx --tls --cafile $ORDERER_CA
```



### 安装智能合约

```shell
git clone https://gitlab.sxtxhy.com/fabric/generalchaincode.git

peer lifecycle chaincode package general.tar.gz --path ./generalchaincode --lang golang --label general_1.0

peer lifecycle chaincode install general.tar.gz
```

### 智能合约审批

```shell
peer lifecycle chaincode approveformyorg -o 192.168.0.170:7050 --ordererTLSHostnameOverride orderer.example.com --tls --cafile $ORDERER_CA --channelID mychannel --name general --version 1.0 --package-id general_1.0:d14561a15856732cf28d0208155ab7ee9f1151d6be7f2dbfd6621f1b6866e951 --sequence 1
	
peer lifecycle chaincode checkcommitreadiness --channelID mychannel --name general --version 1.0 --sequence 1 --output json
```

### 智能合约提交

```shell
peer lifecycle chaincode commit -o localhost:7050 --ordererTLSHostnameOverride orderer.example.com --tls --cafile $ORDERER_CA --channelID mychannel --name general --version 1.0 --sequence 1 --peerAddresses peer0.org1.example.com:8051 --tlsRootCertFiles ${PWD}/organizations/peerOrganizations/org1.example.com/peers/peer0.org1.example.com/tls/ca.crt --peerAddresses peer0.org2.example.com:9051 --tlsRootCertFiles ${PWD}/organizations/peerOrganizations/org2.example.com/peers/peer0.org2.example.com/tls/ca.crt

peer lifecycle chaincode querycommitted --channelID mychannel --name general
```



## 添加Org3

### 生成锚节点文件

```shell
configtxgen -profile TwoOrgsChannel -outputAnchorPeersUpdate \
    ../channel-artifacts/Org3MSPanchors.tx -channelID mychannel -asOrg Org3MSP
```

### env

```shell
export FABRIC_CFG_PATH=$PWD/configtx/
export CORE_PEER_LOCALMSPID="Org3MSP"
export CORE_PEER_TLS_ENABLED=true
export CORE_PEER_TLS_ROOTCERT_FILE=${PWD}/organizations/peerOrganizations/org3.example.com/peers/peer0.org3.example.com/tls/ca.crt
export CORE_PEER_MSPCONFIGPATH=${PWD}/organizations/peerOrganizations/org3.example.com/users/Admin@org3.example.com/msp
export CORE_PEER_ADDRESS=peer0.org3.example.com:10051
export ORDERER_CA=${PWD}/organizations/ordererOrganizations/example.com/orderers/orderer.example.com/msp/tlscacerts/tlsca.example.com-cert.pem
```

### 注册身份

```
cryptogen generate --config=./addOrg3/org3-crypto.yaml --output="organizations"
```

### 生成网络定义

```shell
# 操作环境192.168.0.170 env peer0.org1
configtxgen -printOrg Org3MSP -configPath ./addOrg3 > org3.json

peer channel fetch config config_block.pb -o localhost:7050 --ordererTLSHostnameOverride orderer.example.com -c mychannel --tls --cafile $ORDERER_CA

configtxlator proto_decode --input config_block.pb --type common.Block > config.json
cat config.json | jq .data.data[0].payload.data.config > test.json
mv test.json config.json

jq -s '.[0] * {"channel_group":{"groups":{"Application":{"groups": {"Org3MSP":.[1]}}}}}' config.json ./addOrg2/org3.json > modified_config.json

configtxlator proto_encode --input config.json --type common.Config >original_config.pb
configtxlator proto_encode --input modified_config.json --type common.Config >modified_config.pb
configtxlator compute_update --channel_id mychannel --original original_config.pb --updated modified_config.pb >config_update.pb
configtxlator proto_decode --input config_update.pb --type common.ConfigUpdate >config_update.json

echo '{"payload":{"header":{"channel_header":{"channel_id":"mychannel", "type":2}},"data":{"config_update":'$(cat config_update.json)'}}}' | jq . > config_update_in_envelope.json

configtxlator proto_encode --input config_update_in_envelope.json --type common.Envelope > org3_update_in_envelope.pb

peer channel signconfigtx -f org3_update_in_envelope.pb

#env peer0.org2
peer channel update -f org3_update_in_envelope.pb -c mychannel -o localhost:7050 --ordererTLSHostnameOverride orderer.example.com --tls --cafile ${ORDERER_CA}
```

### 加入网络

```shell
peer channel fetch 0 mychannel.block -o 192.168.0.170:7050 --ordererTLSHostnameOverride orderer.example.com -c mychannel --tls --cafile $ORDERER_CA

peer channel join -b ./channel-artifacts/mychannel.block
```

