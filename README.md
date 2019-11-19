# fabric_new_org_remove

■ 참조 사이트 : https://medium.com/@kctheservant/remove-an-organization-from-a-running-fabric-network-55f52cd0a012
				https://github.com/kctam/3org1ch_143#3org1ch_143

< Remove an Organization from a Running Fabric Network >

1. 하나의 채널에 3개의 조직구성의 네트워크 환경 셋업 및 검증작업

1.1 Instruction (3org1ch_143)

Step 1 : clone this repo in fabric-samples directory
$ cd fabric-samples
$ git clone https://github.com/kctam/3org1ch_143.git
$ cd 3org1ch_143

Step 2 : generate the required crypto material for organizations
$ ../bin/cryptogen generate --config=./crypto-config.yaml

Step 3 : generate the channel artifacts
$ mkdir channel-artifacts && export FABRIC_CFG_PATH=$PWD
$ ../bin/configtxgen -profile ThreeOrgsOrdererGenesis -outputBlock ./channel-artifacts/genesis.block
$ ../bin/configtxgen -profile ThreeOrgsChannel -outputCreateChannelTx ./channel-artifacts/channel.tx -channelID mychannel

Step 4 : Deploy 3org1ch network with docker compose.
$ docker-compose -f docker-compose-cli.yaml up -d
$ docker ps -a

Step 5 : Set up three terminals.

5.1 For Org1
$ docker exec -it cli bash

5.2 For Org2
$ docker exec -e "CORE_PEER_LOCALMSPID=Org2MSP" -e "CORE_PEER_MSPCONFIGPATH=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org2.example.com/users/Admin@org2.example.com/msp" -e "CORE_PEER_ADDRESS=peer0.org2.example.com:7051" -it cli bash

5.3 For Org3
$ docker exec -e "CORE_PEER_LOCALMSPID=Org3MSP" -e "CORE_PEER_MSPCONFIGPATH=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org3.example.com/users/Admin@org3.example.com/msp" -e "CORE_PEER_ADDRESS=peer0.org3.example.com:7051" -it cli bash 

Step 6 : Create channel mychannel
- 세 조직 (피어)이 채널에 가입하면서, 최신 블록에 대한 정보가 Block #0 알수 있음 (세조직)

//# org1
$# peer channel create -o orderer.example.com:7050 -c mychannel -f /opt/gopath/src/github.com/hyperledger/fabric/peer/channel-artifacts/channel.tx
- Result : ~~Received block: 0

//# org1, org2, org3
$# peer channel join -b mychannel.block
- Result : ~~Successfully submitted proposal to join channel

//# org1, org2, org3 check block
$# peer channel getinfo -c mychannel
- Result : Blockchain info: {"height":1 ~~~, (Block# 0 의미함)

Step 7 : Install and Instantiate SACC in mychannel.
- 데모상에서는 보증 정책을 하나의 승인으로 설정했음

//# org1, org2, org3
$# peer chaincode install -n sacc -p github.com/chaincode/sacc -v 1
- Result : Installed remotely response:<status:200 payload:"OK" >

//#org1
$# peer chaincode instantiate -o orderer.example.com:7050 -C mychannel -c '{"Args":["a", "100"]}' -n sacc -v 1 -P "OR('Org1MSP.peer', 'Org2MSP.peer', 'Org3MSP.peer')"
- Result : Using default vscc
 
Step 8 : Query Result from All Orgs

//# org1, org2, org3
$# peer chaincode query -C mychannel -n sacc -c '{"Args":["get","a"]}'
- Result : the value 100

Step 9 : Invoke from Org1, and check result again from all orgs

//# org1
$# peer chaincode invoke -o orderer.example.com:7050 -C mychannel -n sacc -c '{"Args":["set","a","10"]}'
- Result : Chaincode invoke successful. result: status:200 payload:"10"

//# org1, org2, org3
$# peer chaincode query -C mychannel -n sacc -c '{"Args":["get","a"]}'
- Result : the value 10

2. 채널 mychannel에서 Org3 제거 작업

2.1 Work on Configuration Update Transaction (그림참조)

Step 1 : 최신 구성 블록을 가져옴. (Block #0)
//# org1
$# peer channel fetch config config_block.pb -o orderer.example.com:7050 -c mychannel

$# configtxlator proto_decode --input config_block.pb --type common.Block | jq .data.data[0].payload.data.config > config.json

Step 2 : config.json 에서 Org3MSP를 제거 하고 그 결과를 modified_config.json에 유지함 ( config.json vs modified_config.json 파일내용을 비교해봄)
//# org1
$# jq 'del(.channel_group.groups.Application.groups.Org3MSP)' config.json > modified_config.json

Step 3 : config.json 및 modified_config.json 파일을 모두 PB 형식으로 인코딩 후 update.pb 저장
//# org1
$# configtxlator proto_encode --input config.json --type common.Config --output config.pb
$# configtxlator proto_encode --input modified_config.json --type common.Config --output modified_config.pb
$# configtxlator compute_update --channel_id mychannel --original config.pb --updated modified_config.pb --output update.pb

Step 4 : Add back envelope for update.pb, which first decode it into update.json
         Add the envelope
		 update_in_envelope.pb 파일로 다시 encoding함

//# org1
$# configtxlator proto_decode --input update.pb --type common.ConfigUpdate | jq . > update.json

$# echo '{"payload":{"header":{"channel_header":{"channel_id":"mychannel", "type":2}},"data":{"config_update":'$(cat update.json)'}}}' | jq . > update_in_envelope.json

$# configtxlator proto_encode --input update_in_envelope.json --type common.Envelope --output update_in_envelope.pb

- update.json 과 update_in_envelope.json 내용을 살펴봄.

Step 5 : Org1이 update_in_envelope.pb 에 서명, Org2는이 파일을 업데이트로 보냅니다 (Org2의 서명도 포함).
- 아래 명령 실행 후 Orderer는 Org1 및 Org2로만 새 블록을 보냅니다.

//# org1
$# peer channel signconfigtx -f update_in_envelope.pb
- Result : Endorser and orderer connections initialized

//# org2
$# peer channel update -f update_in_envelope.pb -c mychannel -o orderer.example.com:7050
- Result : Endorser and orderer connections initialized
		   Successfully submitted channel update

Step 6 : Check channel block information. (org1, org2, org3) (그림참조)
- Org1과 Org2가 동일한 블록 체인 높이와 동일한 블록을 가지고있는 반면, Org3는 전체 프로세스 전에 수신 한 최신 블록을 유지합니다. 
- Block #2 (Org3 참조)의 " currentBlockHash " 는 Block #3 (Org1 및 Org2 참조) 의 "previousBlockHash"( bJtj… KZM = )입니다.

//# org1
$# peer channel getinfo -c mychannel
- Result : Blockchain info: {"height":4,"currentBlockHash":"yRb+WUXxxjlfx3JaJjbpzkWeyC76BV1vxaJm9T5JWtI=","previousBlockHash":"VmXUr7CzID+0f1ZDf9W7gr/xsMPEp5PmehygAODLqOM="}

//# org2
$# peer channel getinfo -c mychannel
- Result : Blockchain info: {"height":4,"currentBlockHash":"yRb+WUXxxjlfx3JaJjbpzkWeyC76BV1vxaJm9T5JWtI=","previousBlockHash":"VmXUr7CzID+0f1ZDf9W7gr/xsMPEp5PmehygAODLqOM="}

//# org3
$# peer channel getinfo -c mychannel
- Result : Blockchain info: {"height":3,"currentBlockHash":"VmXUr7CzID+0f1ZDf9W7gr/xsMPEp5PmehygAODLqOM=","previousBlockHash":"ury1KzsMtbtRnJrk4uBcsaLdfC4yvnboirSKJ3Kq4DY="}

2.2 Upgrade the Chaincode

Step 7 : Org1 및 Org2에 새 버전의 SACC를 설치하고 체인 코드를이 새 버전으로 업그레이드
- 원장의 상태 (a, 10)를 유지하기 위해 또 다른 initial key/value (b, 1)을 사용합니다.

//# org1, org2
$# peer chaincode install -n sacc -p github.com/chaincode/sacc -v 2

//# org1
$# peer chaincode upgrade -o orderer.example.com:7050 -C mychannel -c '{"Args":["b", "1"]}'  -n sacc -v 2 -P "OR('Org1MSP.peer', 'Org2MSP.peer')"

Step 8: Query from all peers.
-  see all 10
- Org3이 이미 채널에서 제거되었지만 모든 조직의 원장이 이전 상태를 유지한다는 것을 의미합니다.

//# org1, org2, org3
$# peer chaincode query -C mychannel -n sacc -c '{"Args":["get","a"]}'
- Result : the value 10

Step 9 : Invoke from Org1, and check result again from all orgs.
- Org3이 채널에서 제거 된 후에 더 이상 Orderer로부터 블록 업데이트를 수신하지 않음을 의미합니다.

//# org1
$# peer chaincode invoke -o orderer.example.com:7050 -C mychannel -n sacc -c '{"Args":["set","a","1"]}'

//# org1
$# peer chaincode query -C mychannel -n sacc -c '{"Args":["get","a"]}'
- Result : the value 1

//# org2
$# peer chaincode query -C mychannel -n sacc -c '{"Args":["get","a"]}'
- Result : the value 1

//# org3
$# peer chaincode query -C mychannel -n sacc -c '{"Args":["get","a"]}'
- Result : the value 10

Step 10 : Check the latest block in the ledger of all organizations.
- 블록높이 및 해쉬 값을 비교해봄.
 
//# org1
$# peer channel getinfo -c mychannel
- Result : Blockchain info: {"height":6,"currentBlockHash":"UFrB+E56yTsgFmg/2alUBsiTLAvzBatCAc8Jv+Ouh5U=","previousBlockHash":"Pao5Rv0DhJxVppNIT3j1fbxBNySIGnR5U2N+d5wXEwE="}

//# org2
$# peer channel getinfo -c mychannel
- Result : Blockchain info: {"height":6,"currentBlockHash":"UFrB+E56yTsgFmg/2alUBsiTLAvzBatCAc8Jv+Ouh5U=","previousBlockHash":"Pao5Rv0DhJxVppNIT3j1fbxBNySIGnR5U2N+d5wXEwE="}

//# org3
$# peer channel getinfo -c mychannel
- Result : Blockchain info: {"height":3,"currentBlockHash":"VmXUr7CzID+0f1ZDf9W7gr/xsMPEp5PmehygAODLqOM=","previousBlockHash":"ury1KzsMtbtRnJrk4uBcsaLdfC4yvnboirSKJ3Kq4DY="}

3. 요약
- 제거는 채널 레벨에서 수행되므로 제거 된 노드가 더 이상 채널의 일부가 아님을 의미합니다. 
- 컨테이너 레벨, 데이터베이스 레벨 또는 하드웨어 레벨에서 완전히 제거되지 않는 한 제거 전에 보존된 내용은 피어에 남아 있음을 기억하세요.

4. Clean Up

$ cd first-network

$ ./byfn.sh down

$ docker rm $(docker ps -aq)

$ docker rmi $(docker images dev-* -q)

$ docker network prune
