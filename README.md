# geth-prysm-node

mkdir -p /root/ethereum/execution
mkdir -p /root/ethereum/consensus

openssl rand -hex 32 | tr -d "\n" > ~/ethereum/execution/jwt.hex

cp /root/ethereum/execution/jwt.hex /root/ethereum/consensus/jwt.hex

cat /root/ethereum/execution/jwt.hex
cat /root/ethereum/consensus/jwt.hex

cd ethereum

nano docker-compose.yml

```yaml
version: "3.8"

services:
  geth:
    image: ethereum/client-go:stable
    container_name: geth
    restart: unless-stopped
    ports:
      - 30303:30303
      - 30303:30303/udp
      - 8545:8545
      - 8546:8546
      - 8551:8551
    volumes:
      - /root/ethereum/execution:/root
    command:
      - --http
      - --http.api=eth,net,web3
      - --http.addr=0.0.0.0
      - --authrpc.addr=0.0.0.0
      - --authrpc.vhosts=*
      - --authrpc.jwtsecret=/root/jwt.hex
      - --authrpc.port=8551

  prysm:
    image: gcr.io/prysmaticlabs/prysm/beacon-chain
    container_name: prysm
    restart: unless-stopped
    volumes:
      - /root/ethereum/consensus:/data
    depends_on:
      - geth
    ports:
      - 4000:4000
      - 3500:3500
    command:
      - --accept-terms-of-use
      - --datadir=/data
      - --disable-monitoring
      - --rpc-host=0.0.0.0
      - --execution-endpoint=http://geth:8551
      - --jwt-secret=/data/jwt.hex
      - --rpc-host=0.0.0.0
      - --rpc-port=4000
      - --grpc-gateway-corsdomain=*
      - --grpc-gateway-host=0.0.0.0
      - --grpc-gateway-port=3500
      - --min-sync-peers=7
      - --checkpoint-sync-url=https://mainnet.checkpoint.sigp.io
      - --genesis-beacon-api-url=https://mainnet.checkpoint.sigp.io
```

docker compose up -d
