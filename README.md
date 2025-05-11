# geth-prysm-node

## Install Dependecies
**Packages:**
```bash
sudo apt-get update && sudo apt-get upgrade -y
```
```bash
sudo apt install curl iptables build-essential git wget lz4 jq make gcc nano automake autoconf tmux htop nvme-cli libgbm1 pkg-config libssl-dev libleveldb-dev tar clang bsdmainutils ncdu unzip libleveldb-dev  -y
```
**Docker:**
```bash
sudo apt update -y && sudo apt upgrade -y
for pkg in docker.io docker-doc docker-compose podman-docker containerd runc; do sudo apt-get remove $pkg; done

sudo apt-get update
sudo apt-get install ca-certificates curl gnupg
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg

echo \
  "deb [arch="$(dpkg --print-architecture)" signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  "$(. /etc/os-release && echo "$VERSION_CODENAME")" stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt update -y && sudo apt upgrade -y

sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

# Test Docker
sudo docker run hello-world

sudo systemctl enable docker
sudo systemctl restart docker
```

---

## Create Directories
```bash
mkdir -p /root/ethereum/execution
mkdir -p /root/ethereum/consensus
```

---

## Generate the JWT secret:
```bash
openssl rand -hex 32 | tr -d "\n" > ~/ethereum/execution/jwt.hex
```
```bash
cp /root/ethereum/execution/jwt.hex /root/ethereum/consensus/jwt.hex
```
**Verify JWT secrets exist**:
```bash
cat /root/ethereum/execution/jwt.hex
cat /root/ethereum/consensus/jwt.hex
```

---

## Configure `docker-compose.yml`
```bash
cd ethereum
```
```bash
nano docker-compose.yml
```
* Replace the following code into your `docker-compose.yml` file:
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

---

## Port conflicts:
* The setup uses ports 30303, 8545, 8546, 8551, 4000, and 3500. Verify they’re free:
```
netstat -tuln | grep -E '30303|8545|8546|8551|4000|3500'
```
If a port is in use, edit the `docker-compose.yml` to map to a different host port (e.g., `8546:8545` to `8547:8545`).

## Run Geth & Prysm Nodes
```bash
docker compose up -d
```

---

## Getting the RPC Endpoints
### Execution Node (Geth)
Geth provides an HTTP RPC endpoint for interacting with the execution layer of Ethereum. Based on `docker-compose.yml` setup, Geth exposes port `8545` for HTTP RPC. The endpoints are:
* Inside the VPS: `http://localhost:8545`
* Outside the VPS: `http://<vps-ip>:8545` (replace `<vps-ip>` with your VPS’s public IP address, e.g., `http://203.0.113.5:8545`).

### Beacon Node (Prysm)
Prysm, as the beacon node, offers an HTTP gateway on port `3500`. the endpoints are:
* Inside the VPS: `http://localhost:3500`
* Outside the VPS: `http://<vps-ip>:3500` (e.g., `http://203.0.113.5:3500`).

---

## Checking If Nodes Are Synced
**Execution Node (Geth)**
```
curl -X POST -H "Content-Type: application/json" --data '{"jsonrpc":"2.0","method":"eth_syncing","params":[],"id":1}' http://localhost:8545
```
* Response if synced:
```json
{"jsonrpc":"2.0","id":1,"result":false}
```
* Response if syncing:
```json
{"jsonrpc":"2.0","id":1,"result":{"currentBlock":"0xabc","highestBlock":"0xdef",...}}
```
* `currentBlock` vs. `highestBlock` shows syncing progress.

**Beacon Node (Prysm)**
```bash
curl http://localhost:3500/eth/v1/node/syncing
```
* Response if synced:
```json
{"data":{"head_slot":"12345","sync_distance":"0","is_syncing":false}}
```
If `is_syncing` is `false` and `sync_distance` is `0`, the beacon node is fully synced.

* Response if syncing:
```json
{"data":{"head_slot":"12345","sync_distance":"500","is_syncing":true}}
```
* If `is_syncing` is `true`, the node is still syncing, and `sync_distance` indicates how many slots behind it is.

---

## VPS Firewall
* Enable Firewall:
```bash
sudo ufw allow 22
sudo ufw allow ssh
sudo ufw enable
```

* Allow incoming traffic on the Geth/Prysm ports:
```bash
sudo ufw allow 8545/tcp    # Geth HTTP RPC
sudo ufw allow 3500/tcp    # Prysm HTTP API
sudo ufw allow 30303/tcp   # Geth P2P
sudo ufw allow 30303/udp   # Geth P2P
```




