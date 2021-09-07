# Ki Foundation testnet IBC relayer Task

## Precondition

In order to use the Inter-Blockchain Communication Protocol (IBC) we must first have another blockchain that also supports IBC. 
In our example we will use testnet Rizon.

So we need to
* access to the RPC Kichain (you can run local node or use testnet public RPC https://rpc-challenge.blockchain.ki/)
* access to RPC Rizon (you can run local node)
* some tokens in each network, which we will have to forward to the addresses created for the relayer in each network
* The relayer program itself.In our example we will use the official relayer from the Cosmos project https://github.com/cosmos/relayer.git.


## 1. installing Relayer and Initializing
```
git clone https://github.com/cosmos/relayer.git
cd relayer
make install
rly version
```
output:
> version: 1.0.0-rc1–152-g112205b

## 2. Create chain configurations

Create JSON files with network setting

Config for Kichain:
```bash
cat > $HOME/relayer_chain_kichain-t-4.json <<EOF
{
  "chain-id": "kichain-t-4",
  "rpc-addr": "http://kichain-node-ip:26657",
  "account-prefix": "tki",
  "gas-adjustment": 1.5,
  "gas-prices": "0.0025utki",
  "trusting-period": "48h"
}
EOF
```
Config for RIZON:
```bash
cat > $HOME/relayer_chain_groot-011.json <<EOF
{
  "chain-id": "groot-011",
  "rpc-addr": "http://rizon-node-ip:26657",
  "account-prefix": "rizon",
  "gas-adjustment": 1.5,
  "gas-prices": "0.025uatolo",
  "trusting-period": "48h"
}
EOF
```

Pay attention to the parameter rpc-addr - here you must specify the corresponding addresses
and ports of the RPC work nodes. Normally, if you run a node locally, the rpc-addr will be `http://localhost:26657`.

Note that if you are running several tendermint nodes on one computer you will need to change
the settings so that they run on different ports and do not conflict with each other.

## 3. Configure realyer chains

**First of all we have to initialize relayer:**
```bash
rly config init
```

Then import chain's configs
```bash
rly chains add -f $HOME/relayer_chain_kichain-t-4.json
rly chains add -f $HOME/relayer_chain_groot-011.json
```

Create chain env variables for convenience
```bash
cat >> $HOME/.profile <<EOF
export chain_ki=kichain-t-4
export chain_rizon=groot-011
EOF
source $HOME/.profile
```

## 4. Create new wallets or restore the ones you already have:

Create new wallets:
```
rly keys add $chain_ki IBC-ki
rly keys add $chain_rizon IBC-rizon
```

or restore:
```
rly keys restore $chain_ki IBC-ki "mnemonic of saved wallet"
rly keys restore $chain_rizon IBC-rizon "mnemonic of saved wallet"
```

## 5. Add created keys to the chains config
```
rly chains edit $chain_ki key IBC-ki
rly chains edit $chain_rizon key IBC-rizon
```

## 6. Change the relayer block confirmation timeout to 30s
```bash
sed -i 's/timeout: .*$/timeout: 30s/' $HOME/.relayer/config/config.yaml
```

## 7. Fund wallets on both networks

Now we should fund our relayer addresses.
You can find address of your relayer on Rizon and Ki chains:
```bash
rly chains address $chain_rizon
rly chains address $chain_ki
```

For Rizon testnet network you can use faucet: http://faucet.rizon.world/
For Kichain testnetwork you can ask faucet in discord.

check balances:
```bash
rly q balance $chain_ki
rly q balance $chain_rizon
```

## 8. Generate a channel between the networks:

First of all we have to init light clients:
```bash
rly light init $chain_rizon -f
rly light init $chain_ki -f
```

After that you can generate you paths
```bash
rly paths generate $chain_rizon $chain_ki transfer  -- port=transfer
```
you should see the output: 

> Generated path(transfer), run 'rly paths show transfer  -- yaml' to see details

Now you can link chains by path
```bash
rly tx link transfer -d
```

You should see the output
```
I[2021-09-07|03:08:08.174] ★ Clients created: client(07-tendermint-9) on chain[kichain-t-4] and client(07-tendermint-24) on chain[groot-011]
I[2021-09-07|03:08:08.546] - [kichain-t-4]@{{4 225821}}conn(connection-14)-{STATE_OPEN} : [groot-011]@{{0 423442}}conn(connection-21)-{STATE_OPEN}
I[2021-09-07|03:08:08.546] ★ Connection created: [kichain-t-4]client{07-tendermint-9}conn{connection-14} -> [groot-011]client{07-tendermint-24}conn{connection-21}
I[2021-09-07|03:08:08.769] - [kichain-t-4]@{{4 225821}}chan(channel-56)-{STATE_OPEN} : [groot-011]@{{0 423442}}chan(channel-18)-{STATE_OPEN}
I[2021-09-07|03:08:08.769] ★ Channel created: [kichain-t-4]chan{channel-56}port{transfer} -> [groot-011]chan{channel-18}port{transfer}
```

Let's check if all is good
```
rly paths show transfer
```

If you see checkmarks – everything is good: 
```
Path "transfer" strategy(naive):
  SRC(kichain-t-4)
    ClientID:     07-tendermint-9
    ConnectionID: connection-14
    ChannelID:    channel-56
    PortID:       transfer
  DST(groot-011)
    ClientID:     07-tendermint-24
    ConnectionID: connection-21
    ChannelID:    channel-18
    PortID:       transfer
  STATUS:
    Chains:       ✔
    Clients:      ✔
    Connection:   ✔
    Channel:      ✔
```

## 9. Transfer funds between our test wallets: 

Finally, we can proceed with transfers between networks

### 9.1 From Rizon to KiChain
```
rly tx transfer $chain_rizon $chain_ki 1000uatolo $(rly chains address $chain_ki)
```

Output
> I[2021-09-07|03:12:25.443] ✔ [groot-011]@{423479} - msg(0:transfer) hash(83C0282A9254D54A4662E4DC6F05F4ADAE0F18D2442367E6F1AA08DADF026D36)

Lets check balance of destination address
```bash
 rly q balance $chain_ki
```

My output:
> 1000transfer/channel-56/uatolo,92078utki

### 9.1 From Rizon to KiChain
```
rly tx transfer $chain_ki $chain_rizon 1000utki $(rly chains address $chain_rizon)
```

Lets check balance of destination address
```bash
 rly q balance $chain_rizon
```
 
My output:
> 1000ransfer/channel-18/utki,85159uatolo

### 9.3 Transfer IBC tokens back

First we should find IBC hash for transfered token
```
rly q balance $chain_rizon -i
```
Output:
> 1000ibc/D89C13A84011EDCE75521C5F9DD79C292B96C44B72839D9CCC6FDD96F83EB7F8,85159uatolo

Now transfer it back to KiChain
```bash
rly tx transfer $chain_rizon $chain_ki 1000ibc/D89C13A84011EDCE75521C5F9DD79C292B96C44B72839D9CCC6FDD96F83EB7F8 $(rly chains address $chain_ki)
```

Check balances:
```bash
rly q balance $chain_rizon
rly q balance $chain_ki
```


## 10. Transfer between networks by ralayer channel

When you have working relayer path configured, you can start relayer as service to transfer tokens send to IBC cnahhel

Start relayer path in foreground
```bash
rly start transfer --time-threshold 1h
```

After that you can tranfer tokens between networks from any wallet on one chain to another wallet on other, for example using `kid` command
```bash
kid tx ibc-transfer transfer <path_name> <channel-N> rizon1NetworkAddress 100utki --from <Key> --chain-id kichain-t-4
```

For out configuration this command will look like:
```bash
kid tx ibc-transfer transfer  transfer channel-56 rizon1c78wk39xqckygwdvrct7yj5usqs9092w5xwjgh 9utki --from Mercury --chain-id kichain-t-4
```
