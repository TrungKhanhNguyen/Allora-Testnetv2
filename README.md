
# Allora HuggingFaceModel with Random params

Based on Rejump's guide

## Install Dependencies
```bash
sudo apt-get update
sudo apt-get install -y make gcc jq
```
Install Python
```bash
sudo apt install python3
sudo apt install python3-pip
```
Install Docker
```bash
sudo apt update
sudo apt install apt-transport-https ca-certificates curl software-properties-common -y
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"
apt-cache policy docker-ce
sudo apt install docker-ce -y
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

## Install workers
In this section, we run 6 Price Prediction Workers with topic 1, 3, 5, 7, 8, 9

Topic ID can be found here: https://docs.allora.network/devs/get-started/existing-topics

```bash
git clone https://github.com/allora-network/allora-huggingface-walkthrough
cd allora-huggingface-walkthrough
mkdir -p worker-data && chmod -R 777 worker-data
rm -rf config.json && nano config.json
```
- Paste this to config file, replace ```PHRASE``` with your Mnemonic
```bash
{
    "wallet": {
        "addressKeyName": "test",
        "addressRestoreMnemonic": "PHRASE",
        "alloraHomeDir": "/root/.allorad",
        "gas": "1000000",
        "gasAdjustment": 1.0,
        "nodeRpc": "https://allora-rpc.testnet-1.testnet.allora.network/",
        "maxRetries": 1,
        "delay": 1,
        "submitTx": false
    },
    "worker": [
        {
            "topicId": 1,
            "inferenceEntrypointName": "api-worker-reputer",
            "loopSeconds": 5,
            "parameters": {
                "InferenceEndpoint": "http://inference:8000/inference/{Token}",
                "Token": "ETH"
            }
        },
        {
            "topicId": 3,
            "inferenceEntrypointName": "api-worker-reputer",
            "loopSeconds": 5,
            "parameters": {
                "InferenceEndpoint": "http://inference:8000/inference/{Token}",
                "Token": "BTC"
            }
        },
        {
            "topicId": 5,
            "inferenceEntrypointName": "api-worker-reputer",
            "loopSeconds": 5,
            "parameters": {
                "InferenceEndpoint": "http://inference:8000/inference/{Token}",
                "Token": "SOL"
            }
        },
        {
            "topicId": 7,
            "inferenceEntrypointName": "api-worker-reputer",
            "loopSeconds": 5,
            "parameters": {
                "InferenceEndpoint": "http://inference:8000/inference/{Token}",
                "Token": "ETH"
            }
        },
        {
            "topicId": 8,
            "inferenceEntrypointName": "api-worker-reputer",
            "loopSeconds": 5,
            "parameters": {
                "InferenceEndpoint": "http://inference:8000/inference/{Token}",
                "Token": "BNB"
            }
        },
        {
            "topicId": 9,
            "inferenceEntrypointName": "api-worker-reputer",
            "loopSeconds": 5,
            "parameters": {
                "InferenceEndpoint": "http://inference:8000/inference/{Token}",
                "Token": "ARB"
            }
        }
        
    ]
}
```
- Run config
```bash
chmod +x init.config
./init.config
```
- Edit app.py file, with ```API_KEY``` is your API Key in Coingecko
```bash
rm -rf app.py && nano app.py
```
```bash
from flask import Flask, Response
import requests
import json
import pandas as pd
import torch
import random

# create our Flask app
app = Flask(__name__)
        
def get_simple_price(token):
    base_url = "https://api.coingecko.com/api/v3/simple/price?ids="
    token_map = {
        'ETH': 'ethereum',
        'SOL': 'solana',
        'BTC': 'bitcoin',
        'BNB': 'binancecoin',
        'ARB': 'arbitrum'
    }
    token = token.upper()
    if token in token_map:
        url = f"{base_url}{token_map[token]}&vs_currencies=usd"
        return url
    else:
        raise ValueError("Unsupported token") 
               
# define our endpoint
@app.route("/inference/<string:token>")
def get_inference(token):

    try:
      
        url = get_simple_price(token)
        headers = {
          "accept": "application/json",
          "x-cg-demo-api-key": "API_KEY" # replace with your API key
        }
    
        response = requests.get(url, headers=headers)
        if response.status_code == 200:
          data = response.json()
          if token == 'BTC':
            price1 = data["bitcoin"]["usd"] + data["bitcoin"]["usd"]*1/200
            price2 = data["bitcoin"]["usd"] - data["bitcoin"]["usd"]*1/200
          if token == 'ETH':
            price1 = data["ethereum"]["usd"] + data["ethereum"]["usd"]*1/200
            price2 = data["ethereum"]["usd"] - data["ethereum"]["usd"]*1/200      
          if token == 'SOL':
            price1 = data["solana"]["usd"] + data["solana"]["usd"]*1/200
            price2 = data["solana"]["usd"] - data["solana"]["usd"]*1/200  
          if token == 'BNB':
            price1 = data["binancecoin"]["usd"] + data["binancecoin"]["usd"]*1/200
            price2 = data["binancecoin"]["usd"] - data["binancecoin"]["usd"]*1/200   
          if token == 'ARB':
            price1 = data["arbitrum"]["usd"] + data["arbitrum"]["usd"]*1/200
            price2 = data["arbitrum"]["usd"] - data["arbitrum"]["usd"]*1/200            
          random_float = str(round(random.uniform(price1, price2), 2))
        return random_float
    except Exception as e:
        url = get_simple_price(token)
        headers = {
          "accept": "application/json",
          "x-cg-demo-api-key": "API_KEY" # replace with your API key
        }
    
        response = requests.get(url, headers=headers)
        if response.status_code == 200:
          data = response.json()
          if token == 'BTC':
            price1 = data["bitcoin"]["usd"] + data["bitcoin"]["usd"]*1/200
            price2 = data["bitcoin"]["usd"] - data["bitcoin"]["usd"]*1/200
          if token == 'ETH':
            price1 = data["ethereum"]["usd"] + data["ethereum"]["usd"]*1/200
            price2 = data["ethereum"]["usd"] - data["ethereum"]["usd"]*1/200      
          if token == 'SOL':
            price1 = data["solana"]["usd"] + data["solana"]["usd"]*1/200
            price2 = data["solana"]["usd"] - data["solana"]["usd"]*1/200  
          if token == 'BNB':
            price1 = data["binancecoin"]["usd"] + data["binancecoin"]["usd"]*1/200
            price2 = data["binancecoin"]["usd"] - data["binancecoin"]["usd"]*1/200   
          if token == 'ARB':
            price1 = data["arbitrum"]["usd"] + data["arbitrum"]["usd"]*1/200
            price2 = data["arbitrum"]["usd"] - data["arbitrum"]["usd"]*1/200            
          random_float = str(round(random.uniform(price1, price2)),2)
        return random_float

    
# run our Flask app
if __name__ == '__main__':
    app.run(host="0.0.0.0", port=8000, debug=True)

```
- Edit ```requirements.txt```
```bash
flask[async]
gunicorn[gthread]
transformers[torch]
pandas
python-dotenv
```

***REMEMBER change port on ```app.py```, ```docker-compose.yml```, ```config.json```, ```Dockerfile``` before run worker***
## Run worker
```bash
docker compose up --build -d
```
- Check logs
```bash
docker compose logs -f
```




