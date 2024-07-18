
# Allora

This tutorial is for Allora Testnet V2 only.

I wrote this guide mostly based on 0xMoei's guide here: https://github.com/0xmoei/allora-testnet


## Delete old data

You can skip this step if you have never installed Allora node before

- Stop and delete all Allora's previous images 

BE CAREFUL IF YOU ARE RUNNING OTHER NODES, THOSE COMMANDS BELOW WILL DELETE ALL IMAGES AND CONTAINERS.



```bash
  docker rm -vf $(docker ps -aq)
  docker rmi -f $(docker images -aq)
```
- Delete ```allora-chain``` & ```basic-coin-prediction-node``` folder
```bash
cd $HOME
rm -rf allora-chain
rm -rf basic-coin-prediction-node
```
## Install Dependencies
```bash
sudo apt update & sudo apt upgrade -y

sudo apt install ca-certificates zlib1g-dev libncurses5-dev libgdbm-dev libnss3-dev curl git wget make jq build-essential pkg-config lsb-release libssl-dev libreadline-dev libffi-dev gcc screen unzip lz4 -y
```
Install Python3
```bash
sudo apt install python3
sudo apt install python3-pip
```
Install Docker
```bash
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg

echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt-get update
sudo apt-get install docker-ce docker-ce-cli containerd.io
docker version
```
Install Docker Compose
```bash
VER=$(curl -s https://api.github.com/repos/docker/compose/releases/latest | grep tag_name | cut -d '"' -f 4)

curl -L "https://github.com/docker/compose/releases/download/"$VER"/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose

chmod +x /usr/local/bin/docker-compose
docker-compose --version
```
Docker Permission
```bash
sudo groupadd docker
sudo usermod -aG docker $USER
```
Go
```bash
sudo rm -rf /usr/local/go
curl -L https://go.dev/dl/go1.22.4.linux-amd64.tar.gz | sudo tar -xzf - -C /usr/local
echo 'export PATH=$PATH:/usr/local/go/bin:$HOME/go/bin' >> $HOME/.bash_profile
echo 'export PATH=$PATH:$(go env GOPATH)/bin' >> $HOME/.bash_profile
source .bash_profile
go version
```
## Install Allorad
```bash
git clone https://github.com/allora-network/allora-chain.git
cd allora-chain && make all
allorad version
```
Create or recover wallet 
```bash
# Recover your wallet with seed-phrase
allorad keys add testkey --recover

#OR

# Create a new wallet
allorad keys add testkey
```
## Get Testnet faucet
Get allora-testnet-1 faucet here: https://faucet.testnet-1.testnet.allora.network/

## Install workers
In this section, we run 2 Price Prediction Workers with topic 1 & topic 2

Topic ID can be found here: https://docs.allora.network/devs/get-started/existing-topics

```bash
# Install
cd $HOME && git clone https://github.com/allora-network/basic-coin-prediction-node
cd basic-coin-prediction-node

mkdir workers
mkdir workers/worker-1 workers/worker-2 head-data

# Give certain permissions
sudo chmod -R 777 workers/worker-1
sudo chmod -R 777 workers/worker-2
sudo chmod -R 777 head-data

# Create head keys
sudo docker run -it --entrypoint=bash -v ./head-data:/data alloranetwork/allora-inference-base:latest -c "mkdir -p /data/keys && (cd /data/keys && allora-keys)"

# Create worker keys
sudo docker run -it --entrypoint=bash -v ./workers/worker-1:/data alloranetwork/allora-inference-base:latest -c "mkdir -p /data/keys && (cd /data/keys && allora-keys)"
sudo docker run -it --entrypoint=bash -v ./workers/worker-2:/data alloranetwork/allora-inference-base:latest -c "mkdir -p /data/keys && (cd /data/keys && allora-keys)"
```
- Copy the head-id
```bash 
cat head-data/keys/identity
```
## Connect to Allora chain
- Delete and create new ```docker-compose.yml``` file
```bash
rm -rf docker-compose.yml && nano docker-compose.yml
```
- Copy & Paste the following code in it
- Replace ```head-id``` & ```SEED_PHRASE``` in worker-1 and worker-2 containers
```bash
version: '3'

services:
  inference:
    container_name: inference
    build:
      context: .
    command: python -u /app/app.py
    ports:
      - "8000:8000"
    networks:
      eth-model-local:
        aliases:
          - inference
        ipv4_address: 172.22.0.4
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8000/inference/ETH"]
      interval: 10s
      timeout: 10s
      retries: 12
    volumes:
      - ./inference-data:/app/data
  
  updater:
    container_name: updater
    build: .
    environment:
      - INFERENCE_API_ADDRESS=http://inference:8000
    command: >
      sh -c "
      while true; do
        python -u /app/update_app.py;
        sleep 24h;
      done
      "
    depends_on:
      inference:
        condition: service_healthy
    networks:
      eth-model-local:
        aliases:
          - updater
        ipv4_address: 172.22.0.5
  
  head:
    container_name: head
    image: alloranetwork/allora-inference-base-head:latest
    environment:
      - HOME=/data
    entrypoint:
      - "/bin/bash"
      - "-c"
      - |
        if [ ! -f /data/keys/priv.bin ]; then
          echo "Generating new private keys..."
          mkdir -p /data/keys
          cd /data/keys
          allora-keys
        fi
        allora-node --role=head --peer-db=/data/peerdb --function-db=/data/function-db  \
          --runtime-path=/app/runtime --runtime-cli=bls-runtime --workspace=/data/workspace \
          --private-key=/data/keys/priv.bin --log-level=debug --port=9010 --rest-api=:6000 \
          --boot-nodes=/dns4/head-0-p2p.v2.testnet.allora.network/tcp/32130/p2p/12D3KooWGKY4z2iNkDMERh5ZD8NBoAX6oWzkDnQboBRGFTpoKNDF
    ports:
      - "6000:6000"
    volumes:
      - ./head-data:/data
    working_dir: /data
    networks:
      eth-model-local:
        aliases:
          - head
        ipv4_address: 172.22.0.100

  worker-1:
    container_name: worker-1
    environment:
      - INFERENCE_API_ADDRESS=http://inference:8000
      - HOME=/data
    build:
      context: .
      dockerfile: Dockerfile_b7s
    entrypoint:
      - "/bin/bash"
      - "-c"
      - |
        if [ ! -f /data/keys/priv.bin ]; then
          echo "Generating new private keys..."
          mkdir -p /data/keys
          cd /data/keys
          allora-keys
        fi
        # Change boot-nodes below to the key advertised by your head
        allora-node --role=worker --peer-db=/data/peerdb --function-db=/data/function-db \
          --runtime-path=/app/runtime --runtime-cli=bls-runtime --workspace=/data/workspace \
          --private-key=/data/keys/priv.bin --log-level=debug --port=9011 \
          --boot-nodes=/dns4/heads.testnet.allora.network/tcp/9527/p2p/head-id \
          --topic=allora-topic-1-worker --allora-chain-worker-mode=worker \
          --allora-chain-restore-mnemonic='SEED_PHRASE' \
          --allora-node-rpc-address=https://allora-rpc.testnet-1.testnet.allora.network/ \
          --allora-chain-key-name=worker-1 \
          --allora-chain-topic-id=1
    volumes:
      - ./workers/worker-1:/data
    working_dir: /data
    depends_on:
      - inference
      - head
    networks:
      eth-model-local:
        aliases:
          - worker1
        ipv4_address: 172.22.0.12

  worker-2:
    container_name: worker-2
    environment:
      - INFERENCE_API_ADDRESS=http://inference:8000
      - HOME=/data
    build:
      context: .
      dockerfile: Dockerfile_b7s
    entrypoint:
      - "/bin/bash"
      - "-c"
      - |
        if [ ! -f /data/keys/priv.bin ]; then
          echo "Generating new private keys..."
          mkdir -p /data/keys
          cd /data/keys
          allora-keys
        fi
        # Change boot-nodes below to the key advertised by your head
        allora-node --role=worker --peer-db=/data/peerdb --function-db=/data/function-db \
          --runtime-path=/app/runtime --runtime-cli=bls-runtime --workspace=/data/workspace \
          --private-key=/data/keys/priv.bin --log-level=debug --port=9013 \
          --boot-nodes=/dns4/heads.testnet.allora.network/tcp/9527/p2p/head-id \
          --topic=allora-topic-2-worker --allora-chain-worker-mode=worker \
          --allora-chain-restore-mnemonic='SEED_PHRASE' \
          --allora-node-rpc-address=https://allora-rpc.testnet-1.testnet.allora.network/ \
          --allora-chain-key-name=worker-2 \
          --allora-chain-topic-id=2
    volumes:
      - ./workers/worker-2:/data
    working_dir: /data
    depends_on:
      - inference
      - head
    networks:
      eth-model-local:
        aliases:
          - worker1
        ipv4_address: 172.22.0.13
  
networks:
  eth-model-local:
    driver: bridge
    ipam:
      config:
        - subnet: 172.22.0.0/24

volumes:
  inference-data:
  workers:
  head-data:
```
## Run worker
```bash
docker compose build
docker compose up -d
```
## Check your node status
```bash
# Check worker 1 logs
docker compose logs -f worker-1

# Check worker 2 logs
docker compose logs -f worker-2
```
## How to check if your worker node is working well

```bash
network_height=$(curl -s -X 'GET' 'https://allora-rpc.testnet-1.testnet.allora.network/abci_info?' -H 'accept: application/json' | jq -r .result.response.last_block_height) && \
curl --location 'http://localhost:6000/api/v1/functions/execute' --header 'Content-Type: application/json' --data '{
    "function_id": "bafybeigpiwl3o73zvvl6dxdqu7zqcub5mhg65jiky2xqb4rdhfmikswzqm",
    "method": "allora-inference-function.wasm",
    "parameters": null,
    "topic": "1",
    "config": {
        "env_vars": [
            {
                "name": "BLS_REQUEST_PATH",
                "value": "/api"
            },
            {
                "name": "ALLORA_ARG_PARAMS",
                "value": "ETH"
            },
            {
                "name": "ALLORA_BLOCK_HEIGHT_CURRENT",
                "value": "'"${network_height}"'"
            }
        ],
        "number_of_nodes": -1,
        "timeout": 10
    }
}' | jq
```
Response

```bash
{
  "code": "200",
  "request_id": "7a3c05ca-309f-4e77-a448-f27316f0ef15",
  "results": [
    {
      "result": {
        "stdout": "{\"worker\":\"allo18xs6l509akdm65gpktsnlufu6wcl07tjm2nqsx\",\"inference_forecasts_bundle\":{\"inference\":{\"topic_id\":1,\"block_height\":81890,\"inferer\":\"allo18xs6l509akdm65gpktsnlufu6wcl07tjm2nqsx\",\"value\":\"2932.7177149541167\"}},\"inferences_forecasts_bundle_signature\":\"maLl4i9MxwgxfDPxBCjq88qnx97/LwnxJCB6VUX3fD9tu1J4Yfm0El6xJ3nl9IWzOZzEMZGkRUsICTiGkK6QNw==\",\"pubkey\":\"020f096b6fdc103bb9d05bd20331719aab3fee28632211bf095b5b9e6be836284d\",\"blockHeight\":81890,\"topicId\":1}",
        "stderr": "",
        "exit_code": 0
      },
      "peers": [
        "12D3KooWBHkScWQT1kHboyY8veeSEb7b9JqtT6VmKMCNjN8umjzU"
      ],
      "frequency": 50
    },
    {
      "result": {
        "stdout": "{\"worker\":\"allo16tjdq3m33u20ru9nlq338rxweyfn8j8x9gryvu\",\"inference_forecasts_bundle\":{\"inference\":{\"topic_id\":1,\"block_height\":81890,\"inferer\":\"allo16tjdq3m33u20ru9nlq338rxweyfn8j8x9gryvu\",\"value\":\"3473.133811362153\"}},\"inferences_forecasts_bundle_signature\":\"r1JHzQ18dIatCB6Gi8N5/0FWcviOr2cOJrCcQqp4PTMKFH3M4lmcVnNGuIvsvuew0bzyqhKEFfF4jgvlruesPw==\",\"pubkey\":\"031840cfbd26956ed0b60e92db11ab65da82e0b5c30aa802ed806aac681b3d04ef\",\"blockHeight\":81890,\"topicId\":1}",
        "stderr": "",
        "exit_code": 0
      },
      "peers": [
        "12D3KooWJmcDzNFRfWJgZMxQfBNeqCPiHbRU4bQJDW2mfEs6yD94"
      ],
      "frequency": 50
    }
  ],
  "cluster": {
    "peers": [
      "12D3KooWBHkScWQT1kHboyY8veeSEb7b9JqtT6VmKMCNjN8umjzU",
      "12D3KooWJmcDzNFRfWJgZMxQfBNeqCPiHbRU4bQJDW2mfEs6yD94"
    ]
  }
}
```
If you get code 200, your worker node might working but currently I am not sure that I have done it completely correct because the point system is not working yet.

I will update here when the Dashboard updates scores next week.



## References

 - [0xMoei's Allora testnet guide](https://awesomeopensource.com/project/elangosundar/awesome-README-templates)
 - [Allora's official doc](https://docs.allora.network/home/explore)


