# Ibc-transaction-for-kichain

   In this article I will tell you how to set up a repeater for cross translation between two networks using the example of Rizon and Kichain
For this we need a repeater and two working nodes. Or a relay and one working node and a global rpc from the second network. Or a repeater and a global rpc from two networks. But everything is in order.
I will install a repeater for the Rizon node, but you can choose any other method that suits you. The main criterion is that the selected network must have IBC-transfer enabled.
  This parameter can be checked by entering in the terminal where the node is installed. the following command:
 
 > rizond q ibc-transfer params  # (replace rizond with your node's command)
 
 # the response should contain the following output:

> receive_enabled: true
> send_enabled: true
          
  
  So we found out that the Rizon Network is great for cross transactions. Let's start installing and adjusting the Repeater.
Download and install the repeater from the official source Relayer v.1.0.0


> git clone https://github.com/cosmos/relayer.git

> cd relayer

> make install

> cd

> rly version

output:

version: 1.0.0-rc1–152-g112205b

 
 ##                                                  Initializing the repeater

> rly config init

create a folder with network configuration and go to it:

> mkdir rly_config

> cd rly_config

##                               create a settings file for both networks:

> nano kichain-t-4.json

#### copy the content and place it in a file
#we use the global rpc as we put it on vps where the node is not installed locally kichain.

{
 "chain-id": "kichain-t-4",
 
 "rpc-addr": "https://rpc-challenge.blockchain.ki:443", 
 
 "account-prefix": "tki",
 
 "gas-adjustment": 1.5,
 
 "gas-prices": "0.025utki",
 
 "trusting-period": "48h"
}
 
#### similarly, we create a file for the second network, instead of the global rpc, we enter the local one, since the rizon node is located on one server. In theory, you can establish relaying without having a single node, but having a global rpc node of both networks.

> nano groot-011.json

{
 "chain-id": "groot-011",
 "rpc-addr": "http://localhost:26657", 
 "account-prefix": "rizon",
 "gas-adjustment": 1.5,
 "gas-prices": "0.025uatolo",
 "trusting-period": "48h"
}

#### We parse these settings into the config of the relay:

> rly chains add -f groot-011.json
> 
> rly chains add -f kichain-t-4.json
> 
> cd

#### we create new wallets or restore the ones you already have:
> rly keys add groot-011 name of your wallet 
>
>rly keys add kichain-t-4 name of your wallet
 
#### or we restore it with the command:

> rly keys restore groot-011 name of your wallet "mnemonic farce from wallet"
>
> rly keys restore kichain-t-4 name of your wallet "mnemonic farce from wallet"

 #You can use the same mnemonic for both wallets, which is very convenient for me :-) the relay will create different wallets
based on network settings with different rizon and tki prefixes:

#### Add the newly created keys to the config of the relay:
> rly chains edit groot-011 key name of your wallet
>
> rly chains edit kichain-t-4 key name of your wallet

#### Let's change the timeout of waiting for confirmation to 30s by changing the standard 10s:
> nano ~/.relayer/config/config.yaml

> timeout: 30s
 
#### Wallets must be funded on both networks. check the availability of coins with the command:
> rly q balance groot-011
>
> rly q balance kichain-t-4

#### If there are coins on the balance and the relay shows them, then we continue, We initialize the light client in both networks with the command:

> rly light init groot-011 -f
>
> rly light init kichain-t-4 -f

#### We try to generate a channel between the networks with the command:

> rly paths generate groot-011 kichain-t-4 transfer  -- port=transfer

#If the command does not work, try several times or add the - debug parameter to see the step-by-step actions of the system
the output should be as follows:

> Generated path(transfer), run 'rly paths show transfer  -- yaml' to see details
# You can open and view the generated settings with the above command.

 #### Trying to open a channel for relaying:

> rly tx link transfer  --debug

#### if it so happens that the channel does not open, then go to the config with the command:

> nano /root/.relayer/config/config.yaml

#### erase the lines in the path section on both networks:

client-id: 07-tendermint-16
 connection-id: connection-14
 channel-id: channel-11

#### we re-initialize the light client with the commands:

> rly light init groot-011 -f
>
> rly light init kichain-t-4 -f

#### run the command to open the channel again
 
> rly tx link transfer  -- debug

#### and so on until you see the output of the command:
> I[2021–09–06|09:22:54.913] ★ Channel created: [groot-011]chan{channel-11}port{transfer} -> [kichain-t-4]chan{channel-41}port{transfer}

#### now when calling the command

> rly paths list -d

> 0: transfer -> chns(✔) clnts(✔) conn(✔) chan(✔) (groot-011:transfer<>kichain-t-3:transfer)

# you will see ready to transfer and relay packets.

#### Well, let's try to perform an cross-network transaction?

> rly transact transfer [src-chain-id] [dst-chain-id] [amount] [dst-addr] [flags] 

# command template but not as devilish as it is drawn .. in our case the command will look like this:

> rly tx transfer groot-011 kichain-t-4 1000000uatolo tki1eakzw0qhxcclerm0uwxae88ugcmet2radufqqf  -- path transfer

# and the answer will be a conclusion about a successful hash transaction, which can be checked in both rizon and kichain networks in explorer
  
> I[2021–09–06|09:35:00.471] ✔ [groot-011]@{412926} - msg(0:transfer) hash(D4E8A8C5CA7E6ED0B3FD247C456938E1A160B3E850FB10E27440D25A08C69DAE)

> https://testnet.mintscan.io/rizon/txs/D4E8A8C5CA7E6ED0B3FD247C456938E1A160B3E850FB10E27440D25A08C69DAE

>https://testnet.mintscan.io/rizon/txs/D23F51989A2E65749318A2B13DC6DEA9FA0F267E43529081C16E341CF80A1015

#### now we will translate in the other direction, the command looks similar

> rly tx transfer kichain-t-4 groot-011 1000000utki rizon1eakzw0qhxcclerm0uwxae88ugcmet2raquwh9n  -- path transfer

#### if you have forgotten wallet addresses. this happens sometimes :-) then you can see the command

> rly keys list groot-011
>
> rly keys list kichaint-t-4

# Unather way to use Realayer by REMOTE from any kichain node to any wallet in second chain.
first we need to start Relayer permanent we can use tmux or screen but more sensce use service file. let make one

> sudo tee /etc/systemd/system/rlyd.service > /dev/null <<EOF
[Unit]
Description=relayer client
After=network-online.target, rizond.service
[Service]
User=$USER
ExecStart=$(which rly) start transfer
Restart=always
RestartSec=3
LimitNOFILE=65535
[Install]
WantedBy=multi-user.target
EOF

now make start our service and lets use Relayer remote
> sudo systemctl daemon-reload

> sudo systemctl enable rlyd

> sudo systemctl start rlyd                                                             
# Use thise comand to transfer 
> kid tx ibc-transfer transfer transfer channel-N rizon_WALLET_address 1000utki --from name_OF_wallet --fees=5000utki --gas=auto --chain-id kichain-t-4 --home $HOME/kichain/kid

#number of chanel you can see in config.yaml on relayer or use command 
   
   > rly paths show transfer  -- yaml
   
Send back to kichain from rizon same comand
   
> rizond tx ibc-transfer transfer transfer channel-N tki_wallet_adress 1000uatolo --from your_wallet_name --fees=5000uatolo --gas=auto --chain-id groot-011
    
# that's probably all.
