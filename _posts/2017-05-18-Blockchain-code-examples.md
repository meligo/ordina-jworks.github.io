---
layout: post
authors: [ken_coenen, jeroen_de_prest, kevin_leyssens]
title: 'Blockchain hands-on'
image: 
tags: [Blockchain, Ethereum, Smart contracts, Solidity, Geth]
category: Blockchain
comments: true
---


> Last time we gave an introduction to blockchain.
Now we will help you get started with setting up your first blockchain, coding a smart contract and deploying this contract. 
We will still stay in the development environment though we will not deploy on the actual public Ethereum network.

# Topics

In this second article about the innovative blockchain technology, we'll cover the following topics:

1. [Intro](#intro)
2. [What do I need?](#what-do-I-need?)
3. [Setting up Geth](#setting-up-geth)
4. [A look at Mist](#a-look-at-mist)
5. [Working with the Remix IDE](#working-with-the-remix-ide)
6. [Getting started with Solidity](#getting-started-with-solidity)
7. [Deploying and using the contract](#deploying-and-using-the-contract)
8. [Integration with Spring boot](#integration-with-spring-boot)
9. [Conclusion](#conclusion)
10. [Recommended reading](#recommended-reading)


# Intro
In this post we will teach you how to setup your first private ethereum chain, code your first smart contract and integrate it with spring boot.
We chose to use [Ethereum](https://www.ethereum.org) to create our first blockchain.
We made this choice because it seemed to be the most user-friendly and it's pretty well documented for a beginner.
If you would want to use Hyperledger to create your blockchain then I would recommend you use Go to code your chaincode (IBMs version of Smart contracts).

# What do I need?
We will need a few things to successfully setup a network and create smart contracts.
 
* [Ethereum client](#setting-up-your-ethereum-client)
* [Mist](#a-look-at-mist)
* [Remix IDE](#working-with-the-remix-ide)
* [Solidity](#getting-started-with-solidity)
* [Web3j](#deploying-and-using-the-contract) 

## Setting up your Ethereum client
At the time of writing this post there are eight Ethereum clients that can be used to setup your network. 
The two most popular ones are [Geth](https://github.com/ethereum/go-ethereum) and [Parity](https://github.com/paritytech/parity). 
Geth is written in Go and Parity is written in Rust.
Information about the other ones can be found [here](http://ethdocs.org/en/latest/ethereum-clients/choosing-a-client.html).
We use Geth because it is the easiest to setup and get started with.
Geth is also the default for Mist.
The reason you would go for Parity might be because Geth had some problems with spam attacks in late 2016. 
Parity didn't have these problems because it faster in reverting states. 
This is a [heap vs stack problem](https://www.reddit.com/r/ethereum/comments/55v4nk/geth_vs_parity/) that might be solved by now but if it isn't then you at least know about it. 
For us this doesn't really matter because we will be using a private test network.

### Setting up Geth
Let's start with downloading and installing Geth. 
[Here](https://geth.ethereum.org/downloads/) you can find the latest release installer(Windows) and archive. 
If you just want to test your contract on the public test network you can use `geth --testnet --fast --cache=512 console` command to start your node.
A quick explanation of the flags used here. 

`--testnet` is used because you don't want to connect to the public network before testing your code.

`--fast` because you first need to sync with the entire network and that can take quite a while. 
The fast flag avoids processing the entire history of the Ethereum network, which is very CPU intensive, but focuses on downloading more data.

`--cache=512` this flag is specifically useful for HDD users to increase sync speed. 
The recommended range is between 512MB and 2GB. 
The default is 16MB.

Not shown in the example but another one that is still useful is the `--jitvm` flag. 
This is the just-in-time virtual machine that can do further optimizations. 
You can read more about that [here](https://blog.ethereum.org/2016/06/02/go-ethereums-jit-evm/)

### Starting a private network
We first need to create our genesis block. 
This is the first block in the chain. 
If nodes want to connect to a network they will need the same genesis block.
Here is an example of a genesis block.

```
    {
        "config": {
              "chainId": 4689,
              "homesteadBlock": 0,
              "eip155Block": 0,
              "eip158Block": 0
          },
        "alloc"      : {},
        "coinbase"   : "0x0000000000000000000000000000000000000000",
        "difficulty" : "0x400",
        "extraData"  : "",
        "gasLimit"   : "0x2fefd8",
        "nonce"      : "0x2285182ebf7fe31c",
        "mixhash"    : "0x0000000000000000000000000000000000000000000000000000000000000000",
        "parentHash" : "0x0000000000000000000000000000000000000000000000000000000000000000",
        "timestamp"  : "0x00"
    }
  ```
 
I would recommend to alter `"chainId": 4689` to another random number because this is also a parameter for other random unknown nodes to find your network.
Then you should also randomize `"nonce" : "0x2285182ebf7fe31c"` value to make sure you network isn't found by random nodes that accidentally have the same genesis block. 
`"difficulty" : "0x400"` can be altered to make it easier or harder to mine a transaction.
You should save this file as a json file. 
For example `CustomGenesis.json`.
**Every** node that should connect to your private network should be initialized with your custom genesis file!

//TODO write about bootnodes

To start the initialization of your node you use: 

`geth init /path/to/genesis/file`

`--datadir path/to/custom/data/folder` is an optional flag to define a specific init location.

For me this is `geth --datadir chaindata init CustomGenesis.json`.

Now we still need to start our node. 
This command can use a lot of different flags. 
I will discuss a few here the others can be found [here](https://github.com/ethereum/go-ethereum/wiki/Command-Line-Options) or by using the `geth help` command. 

`geth --datadir path/to/custom/data/folder` is the shortest command that you can use to start your node but you can't interact with it in the cmd.

To be able to interact with the node in the cmd after it started start the node with `console` at the end.

Now a few flags that can be added: 

`--nodiscover` We use this flag to turn of the auto discoverability of the nodes. 
This is to make sure that we definitly running private and that only nodes are active on the network that we want to be active. 
Adding of nodes will then also have to happen manually

`--maxpeers 5` This is also a security measure to limit the nodes that can connect to your node. 
The default is 25.

`--rpcapi "db,eth,net,web3,personal,parity" ` We added parity here because web3j uses this to comunicate with the blockchain. 
So if you are not using web3j then you can leave parity out.

`--identity "name"` can be added to give your node an identity in the network.

`--fakepow` this command speeds up the mining process by skipping it, this can be useful for testing contracts but 0x400 is already a low difficulty.

This then creates this command for me. 

`geth --datadir chaindata --nodiscover --maxpeers 2 --rpcapi "db,eth,net,web3,personal,parity" console`



Now our node is running we will first create a etherbase. 
This is where the miner will store its ether. 
We do this by using the following command. 

`personal.newAccount("password")`

Because it is the first account this will be the etherbase.
*Account creation can also be done with [mist](#a-look-at-mist)*
We can now start the miner. 

`miner.start(4)` 

The number four is the amount of threads. It can take some time for the miner to start so **be patient**. 
You can still other commands while the miner is starting.

This how you setup your first node. 
The same process is used to setup other nodes. 
If you are setting up multiple nodes on the same network make sure they have a unique `--datadir`, `--port`, `--rpcport` and `--ipcpath`(or diabled `--ipcdisable`). 

To connect these nodes with `--nodiscover` we need their identification. 
We get this by using `admin.nodeInfo.enode` I get this returned to me `"enode://de9e751faf75e9e4dc570c35379a4c22281fecec4bbedb1e1f69b230da1946f7f80b286ceab2928fb92fb37ce1eb5ce919bc97005680445c8be300c40349a31c@[::]:30303?discport=0"`. 
If you are running every node locally you need to alter the port from `[::]:30303` to whatever port you are connecting to(the `[::]` can stay there). 
If you are running your nodes in a local network then alter  `[::]` to the local ip of the node you are connecting to or the public ip if you are connecting to a node outside of your network. 
Now to add the actual node you use `admin.addPeer("enode of the peer you want to connect to")` For example this 
```
    admin.addPeer("enode://de9e751faf75e9e4dc570c35379a4c22281fecec4bbedb1e1f69b230da1946f7f80b286ceab2928fb92fb37ce1eb5ce919bc97005680445c8be300c40349a31c@192.168.0.2:30303?discport=0")`.
```

You can now check that the nodes are connected to each other by using `admin.peers`.

We have now setup a private test network with multiple nodes and they are mining (test)ether.

## A look at Mist
Mist is a tool to browse and use Dapps(Decentralized Apps). 
We will be using this more as a UI for our blockchain during our contract testing. 
You can download it [here](https://github.com/ethereum/mist/releases) at the bottom of the page.

If you have skipped over the account setup on Geth in the previous section, I will go over that here again but then through Mist.

<div class="row" style="margin: 0 auto 2.5rem auto; width: 100%;">
    <div class="col-md-offset-3 col-md-6" style="padding: 0;">
	    {% include image.html img="/img/blockchain/mist-ui.png" alt="mist ui explanation" title="Explaining the Mist-ui" caption="The Mist UI when a node has just started up" %}
    </div>
</div>

1. Here our accounts will be displayed when we have them and also their individual balance.
1. The total balance of all accounts on this node.
1. The transactions that have been executed by the accounts from this node. 
I have a few grayed out transaction because they are from other networks I have on my pc.
1. General information about the network
      * The top left icon will show the blocks mined. 
      * The top right shows the amount of peers. 
      * The middle icon shows the time since the last mined block. 
    It shows 47 years because our genesis block says it was "mined" at 0 unix time so 01/01/1970. 
      * The last icon reminds us in red that we are on a private network.

<div class="row" style="margin: 0 auto 2.5rem auto; width: 100%;">
    <div class="col-md-offset-3 col-md-6" style="padding: 0;">
	    {% include image.html img="/img/blockchain/mist-add-accounts.png" alt="mist add an account" title="How to add an account" caption="The steps to add an account" %}
    </div>
</div>

When you complete the steps shown in the picture then you will see that it created a account called Main account and in brackets Etherbase. 
This means you can now start your miner in Geth. 
After a while you will see the balance of your account increase. 

## Working with the Remix IDE
Remix is the IDE that is included with Mist. 
You can also use online IDEs but they are exactly the same. 
They all don't really do that much. 
[Here](https://chriseth.github.io/browser-solidity/#version=soljson-latest.js) is an online example.

<div class="row" style="margin: 0 auto 2.5rem auto; width: 100%;">
    <div class="col-md-offset-3 col-md-6" style="padding: 0;">
	    {% include image.html img="/img/blockchain/mist-remix.png" alt="mist open remix ide" title="Finding Remix IDE" caption="Open the remix IDE" %}
    </div>
</div>

<div class="row" style="margin: 0 auto 2.5rem auto; width: 100%;">
    <div class="col-md-offset-3 col-md-6" style="padding: 0;">
	    {% include image.html img="/img/blockchain/remix-ide.png" alt="remix ide" title="The remix ide" caption="Remix IDE firt open" %}
    </div>
</div>

We can then start adding code to the contract. 

## Getting started with Solidity
When Ethereum created blockchain 2.0 (addition of smart contracts) they also created Solidity.
Solidity is now required to write Smart contracts for the ethereum chain.
A good reference to learn Solidity can be found [here](https://learnxinyminutes.com/docs/solidity/).

```
    pragma solidity ^0.4.8;
        
        contract FirstContract {
            address private owner;
            
            //Executed when contract is uploaded on the chain, so it's a
            function FirstContract(){
                owner = msg.sender;
            }
            
            // Function to recover the funds on the contract 
            function kill()  {
                if(msg.sender == owner)
                    selfdestruct(owner); 
            }
            
            function (){
                //Safety function reverts state if something goes wrong
                throw;
            }
        
        }

```

This is a valid contract but it won't do anything. 
For this guide we will be writing a contract similar to our PoC but smaller.

Now a short explanation about the code that we have here. 

`address private owner;` is a private variable because only this contract needs to know who the owner is.
 
 ```
 function FirstContract(){
    owner = msg.sender;
 }
 
 ```
 
 This is the constructor of the contract. 
 When the contract is added to the chain it will run this function first before the others. 
 For now we are setting our owner to the user that executed the transaction (`msg.sender`).

 ```
 function kill()  {
    if(msg.sender == owner)
        selfdestruct(owner); 
 }
 
 ```
 
 This function will kill the contract an withdraw the ether that is left on the contract to the owners wallet. 
 The if-statement here is used to be sure only the owner does this.
 
  ```
  function (){
    throw;
  }
  
  ```
  
  If someone calls a wrong function they will end in this function and the throw means it will revert state to before the call.
  
  We will now expand the contract following the contract we used in our PoC. 
  We will create a vending machine contract where a user can buy something and the machine holds half of the ether payed and gives the other half to his stakeholder.
  When the stock is below a certain point it will ask the supplier for a refill and pay the supplier directly from the contract. 
  We will then also be implementing some security directly in the contract. 
  
  The first functionality we will add is paying for a "product". 
  
  ```
  function pay() payable {
          
          address client = msg.sender;
          if(stock>0){
          
              if(msg.value >( finneyPrice * 1 finney)){
                  
                  uint change = msg.value - (finneyPrice * 1 finney);
                  
                  if(!client.send(change)) throw;
                  
              }else if (msg.value < (finneyPrice * 1 finney)){
                  if(client.send(msg.value)) throw;
                  throw;
              }
              
              stock--;
              
              return true;
              
          }else{
              throw;
          } 
          
      }
  ```
  
  We will also need to add `int public stock;` and `uint priceInFinney;` and these will need to be set in the constructor. 
  I will be setting the stock to 50 and the price to 20. 
  Finney is a sub unit under ether. 
  1 finney is 0.001 ether. 
  Solidity expects us to use wei because that is the smallest unit for ether. 
  We don't give Solidity our data in wei because it just isn't user-friendly because 1 ether is 1000000000000000000 wei. 
  That is wei we use the price in finney and then multiply by 1 finney to get the wei value. 
  Currencies are also always a uint.
  
  Now the explanation for the function.
  
  ``msg.value`` Gives the amount of ether, in wei, that the user send with their transaction. 
  
   ``if(!client.send(change)) throw;`` Try to send the change back to the sender when they send too much. 
    if it fails then revert state to before the call was made.
    
   Then we just lower the stock and that is our first version of the pay method we will add the auto refill and distribution between stakeholder and contract in a later step. 
## Deploying and using the contract
Show the process.
Web3j explanation.


# Conclusion

# Recommended reading



