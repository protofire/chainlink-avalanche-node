# Chainlink Avalanche Development Node	

> Configuration files originally setup by the Avalanche team. Verified working by the ProtoFire team.

This folder contains all files required to run a Chainlink Node for Avalanche. You need to have `docker` and `docker-compose` 

It is recommended you get a Avalanche provider, either by creating your own [Node](https://docs.avax.network/build/tutorials/nodes-and-staking) or by using an online provider such as [DataHub](https://figment.io/datahub/avalanche/). 

## Contract Deployment

All Chainlink v6 contracts required (including [LinkToken](https://cchain.explorer.avax-test.network/address/0xD3896BDD73E61a4275e27F660dDF095522F0a1D3/transactions) ,  [Oracle](https://cchain.explorer.avax-test.network/address/0xa5f33f7d680E596d2f13D1e12434C4de8204C678/transactions) . [AccessControlledAggregator](https://cchain.explorer.avax-test.network/address/0xDcaD20CA76D7F0df84e50CdD80AeA43cEEA980fc/transactions) and [EACAggregatorProxy](https://cchain.explorer.avax-test.network/address/0x8581D1D42b5Cb8d6fA2A901ebbdD0BD4f19aB9f6/transactions)) have been tested, verified and working on the Fuji testnet. For the test we did an AVAX/USD price aggregator with only one Oracle node. You can view the already deployed ones here

```
LinkToken - 0xD3896BDD73E61a4275e27F660dDF095522F0a1D3
Oracle - 0xa5f33f7d680E596d2f13D1e12434C4de8204C678
AccessControlAggregator - 0xDcaD20CA76D7F0df84e50CdD80AeA43cEEA980fc
EACAggregatorProxy - 0x8581D1D42b5Cb8d6fA2A901ebbdD0BD4f19aB9f6
```

Deploying these contracts required no code modifications, and can be deployed using remix and following the [Deploy a Smart Contract on Avalanche Using Remix and MetaMask Guide](https://docs.avax.network/build/tutorials/smart-contracts/deploy-a-smart-contract-on-avalanche-using-remix-and-metamask)

## Configure

There are four files to configure in this folder, `.env.chainlink`, `.env.postgres`, `data/password` and `data/api`. All files should already be present in this folder, just configure them to your liking. 

* `.env.chainlink` - All environment variables for the Chainlink Node. The ones present are the ones used for internal testing and validated as working
* `.env.postgres` - All environment variables for the Postgres Database server. 
* `data` - This folder is mapped via docker to `/chainlink` in the Chainlink Node container
* `data/password` - The password used by the Chainlink Node to lock the wallet
* `data/api` - The password used by the Chainlink Node used for login and API calls

For the postgres database password, please make sure they match in both `.env.chainlink` and `.env.postgres`.

### RPC URLs

For performance reasons, it is **not recommended** to use the default RPC endpoints offered by the Avalanche project. The best alternatives are to sign up for a free endpoint at [DataHub](https://figment.io/datahub/avalanche/) or to [run your own Avalanche node](https://docs.avax.network/build/tutorials/nodes-and-staking/set-up-node-with-installer).

## External Adapters

For price aggregation, a script is included to automatically clone the [External Adapters JS Repo](https://github.com/smartcontractkit/external-adapters-js.git) and build the following external adapters

* Cryptocompare
* metalsapi
* alphavantage

You may use the script `build_adapters.sh` inside this folder to clone and build these, or you may choose to build external adapters yourself. Be sure to include the External Adapter container inside the `docker-compose.yml` file.

### Adding External Adapters

To add an external adapter to the `docker-compose.yml` file, add a section for your external adapter at the bottom, like so

```
  chainlink-external-cryptocompare:
    image: cryptocompare-adapter:latest
    ports:
      - "8080:8080"
    environment:
      API_KEY: <API_KEY_HERE>
```

then, add the container as a requirement to the `chainlink` container, like so

```
services:
  chainlink:
    image: smartcontract/chainlink:latest
    depends_on:
      - chainlink-db
      - chainlink-external-cryptocompare
      ...more external adapters containers here...
```

### Configuring External Adapter in Node

To configure the external adapter in the chainlink node, simply point to the name of the container and the port it's exposing, like so

![example bridge](https://i.ibb.co/0Mj82pZ/msedge-LHpxwubo72.png) 

For this example node setup, the job spec to be configured in the node is the following (assuming the on-chain aggregator has been set to use 8 decimals):

```json
{
    "initiators": [
        {
            "type": "fluxmonitor",
            "params": {
                "address": "<AGGREGATOR ADDRESS>",
                "requestData": {
                    "data": {
                      "from": "ETH",
                      "to": "USD"
                    }
                },
                "feeds": [
                    {"bridge": "cryptocompare"}
                ],
                "threshold": 0.5,
                "absoluteThreshold": 0.01,
                "precision": 8,
                "pollTimer": {
                    "period": "1m0s"
                },
                "idleTimer": {
                    "duration": "1h0m0s"
                }
            }
        }
    ],
    "tasks": [
        {
            "type": "multiply",
            "confirmations": null,
            "params": {
                "times": 100000000
            }
        },
        {
            "type": "ethint256",
            "confirmations": null,
            "params": {}
        },
        {
            "type": "ethtx",
            "confirmations": null,
            "params": {}
        }
    ],
    "startAt": null,
    "endAt": null
}

```

## Running

To run, simply run `docker-compose up` inside this folder after configuring. This will launch all required containers

To stop, simply run `docker-compose down` or use `CTRL + C` 