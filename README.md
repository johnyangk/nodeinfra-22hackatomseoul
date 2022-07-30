# HackAtom 

## team nodeinfra

### run provider node

~~~ bash

interchain-security-pd init provider-node-moniker --chain-id provider --home prov-node-dir

jq ".app_state.gov.voting_params.voting_period = \"180s\"" \
    prov-node-dir/config/genesis.json > prov-node-dir/edited_genesis.json

mv prov-node-dir/edited_genesis.json prov-node-dir/config/genesis.json

interchain-security-pd keys add provider-keyname --home prov-node-dir \
    --keyring-backend test --output json > prov-node-dir/provider_keyname_keypair.json 2>&1

PROV_ACCOUNT_ADDR=$(jq -r .address prov-node-dir/provider_keyname_keypair.json)

interchain-security-pd add-genesis-account $PROV_ACCOUNT_ADDR 1000000000stake \
    --keyring-backend test --home prov-node-dir

interchain-security-pd gentx provider-keyname 100000000stake \
    --keyring-backend test \
    --moniker provider-node-moniker \
    --chain-id provider \
    --home prov-node-dir

interchain-security-pd collect-gentxs --home prov-node-dir \
    --gentx-dir prov-node-dir/config/gentx/

sed -i -r "/node =/ s/= .*/= \"tcp:\/\/localhost:26658\"/" \
    prov-node-dir/config/client.toml
    
interchain-security-pd start 
    --home prov-node-dir \
    --rpc.laddr tcp://localhost:26658 \
    --grpc.address localhost:9091 \
    --address tcp://localhost:26655 \
    --p2p.laddr tcp://localhost:26656 \
    --grpc-web.enable=false
              
interchain-security-pd q staking validators --home prov-node-dir
     
tee consumer-proposal.json<<EOF
{
    "title": "Create consumer chain",
    "description": "Gonna be a great chain",
    "chain_id": "consumer", 
    "initial_height": {
        "revision_height": 1
    },
    "genesis_hash": "Z2VuX2hhc2g=",
    "binary_hash": "YmluX2hhc2g=",
    "spawn_time": "2022-03-11T09:02:14.718477-08:00",
    "deposit": "10000001stake"
}
EOF

interchain-security-pd tx gov submit-proposal \
       create-consumer-chain consumer-proposal.json \
       --keyring-backend test \
       --chain-id provider \
       --from provider-keyname \
       --home prov-node-dir \
       -b block

interchain-security-pd tx gov vote 1 yes --from provider-keyname   --keyring-backend test --chain-id provider --home prov-node-dir -b block

# check vote success

interchain-security-pd q gov proposal 1 --home prov-node-dir
~~~

### run consumer node
~~~ bash

interchain-security-cd init consumer-node-moniker --chain-id consumer --home cons-node-dir


interchain-security-cd keys add consumer-keyname --home cons-node-dir \
    --keyring-backend test --output json > cons-node-dir/consumer_keyname_keypair.json 2>&1


CONS_ACCOUNT_ADDR=$(jq -r .address cons-node-dir/consumer_keyname_keypair.json)

interchain-security-cd add-genesis-account $CONS_ACCOUNT_ADDR 1000000000stake \
    --keyring-backend test --home cons-node-dir


interchain-security-pd query provider consumer-genesis consumer \
    --home prov-node-dir -o json > ccvconsumer_genesis.json


jq -s '.[0].app_state.ccvconsumer = .[1] | .[0]' cons-node-dir/config/genesis.json ccvconsumer_genesis.json > \
      cons-node-dir/edited_genesis.json 

mv cons-node-dir/edited_genesis.json cons-node-dir/config/genesis.json

echo '{"height": "0","round": 0,"step": 0}' > cons-node-dir/data/priv_validator_state.json  
cp prov-node-dir/config/priv_validator_key.json cons-node-dir/config/priv_validator_key.json  
cp prov-node-dir/config/node_key.json cons-node-dir/config/node_key.json


sed -i -r "/node =/ s/= .*/= \"tcp:\/\/localhost:26648\"/" cons-node-dir/config/client.toml

# consumer local node use the following command
interchain-security-cd start --home cons-node-dir \
        --rpc.laddr tcp://localhost:26648 \
        --grpc.address localhost:9081 \
        --address tcp://localhost:26645 \
        --p2p.laddr tcp://localhost:26646 \
        --grpc-web.enable=false


~~~


### run hermes
~~~ bash
mkdir ~/.hermes

tee ~/.hermes/config.toml<<EOF
[global]
 log_level = "info"

[[chains]]
account_prefix = "cosmos"
clock_drift = "5s"
gas_multiplier = 1.1
grpc_addr = "tcp://localhost:9081"
id = "consumer"
key_name = "relayer"
max_gas = 2000000
rpc_addr = "http://localhost:26648"
rpc_timeout = "10s"
store_prefix = "ibc"
trusting_period = "14days"
websocket_addr = "ws://localhost:26648/websocket"

[chains.gas_price]
       denom = "stake"
       price = 0.00

[chains.trust_threshold]
       denominator = "3"
       numerator = "1"

[[chains]]
account_prefix = "cosmos"
clock_drift = "5s"
gas_multiplier = 1.1
grpc_addr = "tcp://localhost:9091"
id = "provider"
key_name = "relayer"
max_gas = 2000000
rpc_addr = "http://localhost:26658"
rpc_timeout = "10s"
store_prefix = "ibc"
trusting_period = "14days"
websocket_addr = "ws://localhost:26658/websocket"

[chains.gas_price]
       denom = "stake"
       price = 0.00

[chains.trust_threshold]
       denominator = "3"
       numerator = "1"
EOF

hermes keys add --key-file cons-node-dir/consumer_keyname_keypair.json --chain consumer
hermes keys add --key-file prov-node-dir/provider_keyname_keypair.json --chain provider
     
hermes create connection \
    --a-chain consumer \
    --a-client 07-tendermint-0 \
    --b-client 07-tendermint-0

hermes create channel \
    --a-chain consumer \
    --a-port consumer \
    --b-port provider \
    --order ordered \
    --channel-version 1 \
    --a-connection connection-0

hermes --json start      

DELEGATIONS=$(interchain-security-pd q staking delegations \
  $(jq -r .address prov-node-dir/provider_keyname_keypair.json) --home prov-node-dir -o json)

OPERATOR_ADDR=$(echo $DELEGATIONS | jq -r '.delegation_responses[0].delegation.validator_address')

interchain-security-pd tx staking delegate $OPERATOR_ADDR 1000000stake \
                --from provider-keyname \
                --keyring-backend test \
                --home prov-node-dir \
                --chain-id provider \
                -y -b block

interchain-security-pd q tendermint-validator-set --home prov-node-dir

interchain-security-pd q tendermint-validator-set --home cons-node-dir

~~~
