---
layout: post
authors: [ken_coenen, jeroen_de_prest, kevin_leyssens]
title: 'Blockchain hands-on'
image: /img/blockchain/blockchainHeaderImageHandsOnPNG.png
tags: [Blockchain, Ethereum, Smart contracts, Solidity, Geth, spring boot, java]
category: Blockchain
comments: true
---


> Last time we gave an introduction about blockchain.
Now we will dive deeper into blockchain. 
We will set up our first blockchain, code a smart contract, deploy this contract and implement it with Java. 
For this example we won't be deploying it on the Ethereum network, but locally in our development environment.

# Topics

Following topics will be discussed in our second blockchain article:

1. [Intro](#intro)
2. [What do I need?](#what-do-I-need?)
3. [Setting up Geth](#setting-up-geth)
4. [A look at Mist](#a-look-at-mist)
5. [Working with the Remix IDE](#working-with-the-remix-ide)
6. [Getting started with Solidity](#getting-started-with-solidity)
7. [Deploying and using the contract](#deploying-and-using-the-contract)
8. [Ethereum and Java with web3j](#ethereum-and-java-with-web3j)
9. [Conclusion](#conclusion)


# Intro
In this post we will teach you how to __setup__ your first __private__ ethereum chain, code your first __smart contract__ and integrate it with spring boot.
We will be using [Ethereum](https://www.ethereum.org) to create our first blockchain.
We made this choice because it seemed to be the most user-friendliest and it's pretty well documented for a beginner.
If you would like to use Hyperledger to create your blockchain, then it is recommended to use the Go programming language to write the [chaincode](https://github.com/IBM-Blockchain/learn-chaincode) (IBMs version of Smart contracts).

# What do I need?
We will need a few things to successfully setup a network and create smart contracts.
 
* [Ethereum client](#setting-up-your-ethereum-client)
* [Mist](#a-look-at-mist)
* [Remix IDE](#working-with-the-remix-ide)
* [Solidity](#getting-started-with-solidity)
* [Web3j](#deploying-and-using-the-contract) 

## Setting up your Ethereum client
At the time of writing this post, there are eight Ethereum clients that can be used to setup your network. 
The two most popular ones are __[Geth](https://github.com/ethereum/go-ethereum)__ and __[Parity](https://github.com/paritytech/parity)__. 
Geth is written in Go and Parity is written in Rust.
Information about the other ones can be found [here](http://ethdocs.org/en/latest/ethereum-clients/choosing-a-client.html).
We use Geth because it is the easiest to setup and to get started with.
Geth is also the __default__ for Mist.
Geth had some problems with spam attacks in the late 2016, that's why some people go for Parity.
Parity didn't have these problems because it is faster in reverting states. 
This is a [heap vs stack problem](https://www.reddit.com/r/ethereum/comments/55v4nk/geth_vs_parity/) that might be solved by the time you are reading this article.
We will be using a __private test network__, so it won't be a problem.

### Setting up Geth
Let's start with downloading and installing Geth. 
[Here](https://geth.ethereum.org/downloads/) we can find the latest release installer (Windows) and archive. 
When connecting to the public test network, we just can use following command:
```
geth --testnet --fast --cache=512 console
``` 
A quick explanation of the flags used in the command above: 

`--testnet` is used for testing purpose. We don't want to upload our contract to the public network before testing the code.

`--fast` because we first need to sync with the entire network and that can take quite a while. 
The fast flag avoids processing the entire history of the Ethereum network, which is very CPU intensive. By syncing through state downloads instead of downloading the full block data.

`--cache=512` is specifically useful for HDD users to increase sync speed. 
The recommended range is between 512MB and 2GB. 
The default is 16MB.

Another one that can be very useful is the `--jitvm` flag. 
This is the just-in-time virtual machine that can do further optimizations. 
You can read more about that [here](https://blog.ethereum.org/2016/06/02/go-ethereums-jit-evm/)

### Starting a private network
First of all, we need to create our __genesis__ block.
This is the __first block__ in the chain. 
If nodes want to connect to a network they will need the __same__ genesis block.
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
 
It is recommended to alter `"chainId": 4689` to another random number, because this is also a parameter for other random unknown nodes to find your network.
It's also better to __randomize__ `"nonce" : "0x2285182ebf7fe31c"` to make sure the network isn't found by random nodes that accidentally have the same genesis block. 
`"difficulty" : "0x400"` can be altered to make it easier or harder to mine a transaction.
The file should be saved as a __json__ file. 
For example `CustomGenesis.json`.
__Every node__ that connects to your private network should be initialized with your custom genesis file!
To start the __initialization__ of your node use following command: 

```
geth init /path/to/genesis/file
```

You can add following optional flag to define a specific init location: `--datadir path/to/custom/data/folder`.

An example of the command with the flag from above goes as follows: 
```
geth --datadir chaindata init CustomGenesis.json
```

Now we need to __start__ our node. 
This command has a lot of optional flags. 
We'll discuss a few in this article and the other ones can be found [here](https://github.com/ethereum/go-ethereum/wiki/Command-Line-Options) or by using the `geth help` command. 

When using the shortest command, we can't interact with the chain through the console in the same window.
```
geth --datadir path/to/custom/data/folder
```

When adding the `console` flag at the end of the command, we'll be able to interact with the chain in the console.
```
geth --datadir path/to/custom/data/folder console
```

We will now discuss a few more important flags that can be added to the command: 

1. `--nodiscover`: We use this flag to turn off the auto discoverability of the nodes. 
This is to make sure that we are running a private chain. 
Adding nodes will have to happen __manually__.

2. `--maxpeers 5`: A __security__ measure to limit the nodes that can connect to your node at the same time. 
The default is 25.

3. `--rpcapi "db,eth,net,web3,personal,parity" `: This starts the __HTTP interface__ with db, eth, net, web3, personal and parity enabled.

4. `--identity "name"`: Can be added to give your node an __identity__ in the network.

5. `--fakepow`: This command speeds up the mining process by __skipping__ it.
This can be useful for testing contracts. But in our genesis file we used 0x400 as a difficulty level, which is already very a low difficulty.


```
geth --datadir chaindata --nodiscover --maxpeers 2 --rpcapi "db,eth,net,web3,personal,parity" console
```



Now our node is running. 
But we need an __etherbase__ account. 
On this account the miner will store its ether he gets from __mining__.
The account will also be the first one created on our chain. 
We do this by using the following command:

```
personal.newAccount("password")
```

*Account creation can also be done with mist. 
Just skip ahead to [here](#a-look-at-mist) if you don't want to use the cmd.*

We can now start the miner. 
The miner is needed to add transactions to the chain. 
The number four is the amount of threads. 
It can take a while for the miner to start so __be patient__. 
We can still use other commands while the miner is starting.

```
miner.start(4)
``` 

Now we know how to setup our __first node__. 
Fairly easy right? 
The same process is used to setup other nodes. 
If you are setting up multiple nodes on the same network make sure they have a unique `--datadir`, `--port`, `--rpcport` and `--ipcpath` (or disabled `--ipcdisable`). 

To connect these new nodes with the `--nodiscover` flag up on the chain, we need their __identification__. 
```
admin.nodeInfo.enode
```

The response should be something similar like this:  
```
"enode://de9e751faf75e9e4dc570c35379a4c22281fecec4bbedb1e1f69b230da1946f7f80b286ceab2928fb92fb37ce1eb5ce919bc97005680445c8be300c40349a31c@[::]:30303?discport=0"
``` 

When running every node locally on the same machine, we need to alter the port from `[::]:30303` to whatever port we are connecting to (the `[::]` can stay there). 
If we are running the nodes in a local network then we need to alter  `[::]` to the __local ip__ of the node we're connecting to or the public ip if we're connecting to a node outside of your network. 
The command to add the nodes is `admin.addPeer("enode of the peer you want to connect to")` 
```
    admin.addPeer("enode://de9e751faf75e9e4dc570c35379a4c22281fecec4bbedb1e1f69b230da1946f7f80b286ceab2928fb92fb37ce1eb5ce919bc97005680445c8be300c40349a31c@192.168.0.2:30303?discport=0")`
```

To check the connected nodes use the command `admin.peers`.

We have now successfully setup a __private__ test network with __multiple nodes__ and miners (a miner per node). 
Note that we are mining test ether and not real ether. 
In this setup a miner for each node is required.

## A look at Mist
Mist is a __tool__ to browse and use Dapps (Decentralized Apps). 
We will be using this more as a UI for our contract testing on the blockchain. 
You can download it [here](https://github.com/ethereum/mist/releases).

If you already have your account in the previous section you don't have to do it again here but it might be useful to see how the interface works.

<div class="row" style="margin: 0 auto 2.5rem auto; width: 100%;">
    <div class="col-md-offset-3 col-md-6" style="padding: 0;">
	    {% include image.html img="/img/blockchain/mist-ui.png" alt="mist ui explanation" title="Explaining the Mist-ui" caption="The Mist UI when a node has just started up" %}
    </div>
</div>

1. All our __local accounts__ will be displayed and their individual balance.
2. The __total balance__ of all accounts on this node.
3. The __transactions__ that have been executed by the accounts from this node. 
4. General information about the network
      * The top left icon will show the amount of __blocks__ mined. 
      * The top right icon shows the amount of __peers__ connected. 
      * The middle icon shows the time since the __last mined block__. 
    It says 47 years because there aren't any blocks mined yet and in our genesis block the timestamp is 0 unix time so 01/01/1970. 
      * The last icon reminds us in red that we are on a __private__ network.

<div class="row" style="margin: 0 auto 2.5rem auto; width: 100%;">
    <div class="col-md-offset-3 col-md-6" style="padding: 0;">
	    {% include image.html img="/img/blockchain/mist-add-accounts.png" alt="mist add an account" title="How to add an account" caption="The steps to add an account" %}
    </div>
</div>

When completing the steps shown in the picture above, an account called Main account and in brackets __Etherbase__ will be created. 
The etherbase account is the account where the miner's ether is stored.
If you have created accounts through __geth__ in the previous part of the article, there already will be an etherbase account. 
Now start your miner in Geth `miner.start(4)`.
After the miner is started, the balance of the account increases. 

## Working with the Remix IDE
Remix is the IDE that is __included__ within Mist. 
You can also use online IDEs, but they are exactly the same. 
They all don't really do that much.
[Here](https://chriseth.github.io/browser-solidity/#version=soljson-latest.js) is an example of an online IDE.

<div class="row" style="margin: 0 auto 2.5rem auto; width: 100%;">
    <div class="col-md-offset-3 col-md-6" style="padding: 0;">
	    {% include image.html img="/img/blockchain/mist-remix.png" alt="mist open remix ide" title="Finding Remix IDE" caption="Opening the remix IDE" %}
    </div>
</div>

<div class="row" style="margin: 0 auto 2.5rem auto; width: 100%;">
    <div class="col-md-offset-3 col-md-6" style="padding: 0;">
	    {% include image.html img="/img/blockchain/remix-ide.png" alt="remix ide" title="The remix ide" caption="Remix IDE first opening" %}
    </div>
</div>

## Getting started with Solidity
When Ethereum created __blockchain 2.0__ (addition of smart contracts) they created __Solidity__ as well.
Solidity is now required to write __Smart contracts__ for the ethereum chain.
A good reference to learn Solidity can be found [here](https://learnxinyminutes.com/docs/solidity/) or the [Solidity docs](https://solidity.readthedocs.io/en/develop/). 
If you are creating a token contract please take the [ERC20 token standard](https://theethereum.wiki/w/index.php/ERC20_Token_Standard) into account.

```
    pragma solidity ^0.4.8;
        
        contract VendingMachine {
            address private owner;
            
            //Executed when contract is uploaded on the chain, so it's a constructor
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

This is a valid contract, but it won't do anything. 
Now a short explanation about the code above. 

`address private owner;` is a private variable. 
Only this contract needs to know who the owner is. 
The owner is used to restrict some functions later on.
 
 ```
 function FirstContract(){
    owner = msg.sender;
 }
 
 ```
 
 This is the __constructor__ of the contract. 
 When the contract is added to the chain, it will run this function first before the others. 
 For now we are setting our owner to the address (user) that uploaded the contract (executed the transaction) to the chain (`msg.sender`).

 ```
 function kill()  {
    if(msg.sender == owner)
        selfdestruct(owner); 
 }
 
 ```
 
 This function will __kill__ the contract and withdraw the ether that is left on the contract to the owners wallet. 
 The if-statement here is used to be sure only the owner can call this function.
 Note that in a blockchain nothing can be removed.
 The selfdestruct command just makes it __inaccessible__.
 Every ether send to a destroyed contracted, will be lost.

  ```
  function (){
    throw;
  }
  
  ```
  
  If someone calls a __wrong function__, this function will be initiated. We throw, which means the contract will revert its state to the state it had before the call.
  
  We will now expand the contract following the contract we used in our PoC.
  But first, let's explain our PoC.
  We will create a vending machine contract where a user can buy something.
  The machine holds half of the ether payed and distributes the other half to his stakeholder.
  When the stock is below a certain point, the contract (machine) will ask the supplier for a refill and pay the supplier directly from the contract. 
  We will also be implementing some security directly in the contract.  
  
  The first functionality we will add is paying for a "product". 
  
  ```
  function pay() payable {
          
          address client = msg.sender;
          if(stock>0){
          
              if(msg.value >( finneyPrice * 1 finney)){
                  
                  uint change = msg.value - (finneyPrice * 1 finney);
                  
                  if(!client.send(change)) throw;
                  
              }else if (msg.value < (finneyPrice * 1 finney)){
                  if(!client.send(msg.value)) throw;
                  throw;
              }
              
              stock--;
              
              return true;
              
          }else{
              throw;
          } 
          
      }
  ```
  
We will also need to add `int public stock;`, `int public maxStock`, `int public minStock` and `uint priceInFinney;`.
These variables will need to be initialized in the constructor. 
We'll be setting the stock and maxStock to 50, the minStock to 45 (for testing purposes) and the price to 20. 
The price is in __finney__. 
Finney is a __sub unit__ under __ether__. 
1 finney is 0.001 ether. 
Solidity expects us to use __wei__. 
Wei is the __smallest unit__ for ether. 
We won't be giving Solidity our data in wei, because it just isn't user-friendly.
1 ether is 1000000000000000000 wei.
Or in other words, 1 ether is 10^18wei. 
That is why we use the price in finney. 
By multiply by 1 finney we get the wei value. 
__Currencies__ are also always in __uint__ in Solidity.

Now some explanation of the function:
 * `` payable `` This is a modifier which allows you to send ether with the function. 
 Modifiers will be explained a little further in the article.
 * ``msg.value`` Gives the amount of ether, in wei, that the user send with their transaction. 
 * ``if(!client.send(change)) throw;`` Send the change back to the sender. 
If it fails then revert state to before the call was made.
* `stock--` Lowers the stock.

This is our first version of the pay method.
We will expand this method later.

```
  function resupply(int amount) payable returns (int) {
    if(amount<=0 && stock + amount > maxStock) throw;
  
    stock += amount;
          
    return stock;
  }
 ```
 
This is a pretty straight forward function. 
First we check if the amount given is bigger then zero and then we will check if the maxStock isn't exceeded by adding the amount to the current stock. 
Afterwards we add the amount to the stock. 
Later on we will add a supplier to this function.
Now that we have the resupply function we can add the automatic resupply functionality in the pay method.
 
```
  ...
  if(stock == minStock) this.resupply(maxStock-stock);
  ...
```

Add the code above, below the `stock--;` in the pay() method. 
It is a simple check to see if we reached the minimum stock.
If it is true, we call our resupply function to fill to maxStock again.
 
The next step is adding our stakeholders. 

```
  function divideProfit() internal{
          uint profit = (finneyPrice * 1 finney)/2;
          uint share = profit / stakeholders.length;
          
          for(uint x = 0; x < stakeholders.length; x++) {
              if(!stakeholders[x].send(share)) throw;
          }
  }
 ```
 
Here we use the `internal` modifier. 
Functions and state variables using this modifier can only be accessed __internally__ (i.e. from within the current contract or contracts deriving from it).
We decided to keep half of the amount in the contracts wallet to buy from the supplier when the stock is at its minimum.
The other half is the profit.
We'll be dividing it over the stakeholders. 
For this to work, we will need to create an __array__ of stakeholders `address[] private stakeholders;` with stakeholders in it.
Then we divide the profit by the amount of stakeholders (don't worry about dividing by 0).
This amount will be send to every stakeholders through a __loop__.
This function is complete, so now we need to add this function to the pay method.

```
...
if(stakeholders.length > 0)
    divideProfit();
...
```

Add the code above, above the `stock--;` in the pay method.
Here we also prevent the divide by 0 problem.

You can go more advanced with this.
If different stakeholders have a bigger stake then others. 
We would recommend to use a __[struct](https://solidity.readthedocs.io/en/develop/structure-of-a-contract.html#structs-types)__ in combination with __[mapping](http://solidity.readthedocs.io/en/develop/types.html#mappings)__.
Then use the struct as valuetype for the mapping and the address as the key type.

Now we will need to create the supplier contract.
This contract can be in the same file as the vending contract. 

```

contract Supplier {

    address owner;
    uint public priceInFinney;
    int public stock;
    
    modifier onlyOwner(){
        if(owner == msg.sender){
            _;
        }
    }
    
    /* this function is executed at initialization and sets the owner of the contract */
    function Supplier(uint price,int defaultStock) { 
        owner = msg.sender;
        priceInFinney = price;
        stock = defaultStock;
    }
    
    function withdrawAll() onlyOwner(){
        if(!owner.send(this.balance)){ throw;}
    }
    
    function withdraw(int amountInFinney) onlyOwner(){
        if(!owner.send(uint256(amountInFinney * 1 finney))){ throw;}
    }

    /* Function to recover the funds on the contract */
    function kill() onlyOwner()  {
        selfdestruct(owner); 
    }
    
    function buyStock(int amount) payable returns (bool){
        if(stock - amount >= 0 && ((uint(amount) * priceInFinney)*1 finney) - 1 finney <= msg.value ){
            stock -= amount;
            return true;
        }else{
            if(!msg.sender.send(msg.value)){
                throw;
            }
            return false;
        }
    }
    
    function setPrice(uint newPriceInFinney){
        priceInFinney = newPriceInFinney;
    }
    
    
   function () payable {
        throw;
    }
}

```


This is a pretty basic contract.
It has the price of the product and a function to withdraw the money to the owner's account.
`this.balance` is built in to get the balance of a contract. 

We will need to make some changes to the first Vending contract. 
Add a place to store the supplier contract and the supplier contract address `Supplier s;`.
Then we will make a method to set this supplier.

```

function setSupplier(address a) {
        supplier = a;
        s = Supplier(supplier);
}

```

Add the code below in the resupply method.
Just under the first if-method.

```
...
uint weiToSend = (uint256(amount) * (s.priceInFinney() * 1 finney)) + msg.value-s.priceInFinney();
if(!s.buyStock.value(weiToSend)(amount)) throw;
...
```

The method pays the supplier with the price that is defined in the supplier's contract.
We got the price by following command `s.priceInFinney()`.
Converting the price into wei is done by multiplying by 1 finney.
Then we multiply that by the amount we want to order. 
We add the current message value minus the supplier price of a product, because the payment for the current transaction has not yet been added to the balance of the contract.
The next line is something new.
If we append `.value(weiToSend)` to the function call, we can pass wei to the next function.
This function must be payable. 
If it fails, we will revert the state to before the call was done.

We did some __security__ checks in the contracts.
These checks were done by simple if-statements. `if(msg.sender == owner)` 
But there is a better way to do this.
By using __[modifiers](http://solidity.readthedocs.io/en/develop/common-patterns.html#restricting-access)__.

```
modifier onlyOwner(){
        if(owner == msg.sender){
            _;
        }
        throw;
    }
```

We just use the if-statement in the modifier.
When the statement is true we continue to the function the user requested with `_;`. 
To use this modifier we just add it after the function's name.

```

function kill() onlyOwner()  {
    selfdestruct(owner); 
}

```

Now we can remove the if-statements and add the modifiers instead like you see in the example above.
With these modifiers we can also create an __account__ system. 

```

modifier restrictAccessTo(address[] _collection){
        for(uint i = 0; i < _collection.length; i++) {
            if (_collection[i] == msg.sender) {
                _;
                return;
            }
        }
        
        
        if(msg.value > 0){
            if(!msg.sender.send(msg.value)) throw;
        }
    
    }
    
    function addUser(address user){
            for(uint x = 0; x < accounts.length; x++) {
                if (accounts[x] == user) {
                    throw;
                }
            }
            accounts.push(user);
            users++;
        }
        
        function removeUser(address user) restrictAccessTo(admins){
            for(uint x = 0; x < accounts.length; x++) {
                if (accounts[x] == user) {
                    //To fill the gap in the array
                    accounts[x] == accounts[accounts.length-1];
                    delete accounts[accounts.length-1];
                    users--;
                    break;
                }
            }
        }

```

Only users that are in the given array, that is passed through a parameter, can access those functions. 
We'll make an add and remove function to add and remove our users.
When __removing__ from an array, you will get gaps.
In order to bypass this problem,  we switch the last account in the array with the one we want to remove and then remove the last account.
These functions can be used with __multiple__ user groups.
For example with admins and normal users.
Method overloading can be used for this.

Another thing we can do, is create a modifier for the following code block: 

```
...
if(msg.value >( finneyPrice * 1 finney)){
    uint change = msg.value - (finneyPrice * 1 finney);
                  
    if(!client.send(change)) throw;
                  
    }else if (msg.value < (finneyPrice * 1 finney)){
        if(!client.send(msg.value)) throw;
        throw;
    }
...
``` 

We can replace it with the following modifier:

```
modifier costs(uint _amount) {
    require(msg.value >= _amount * 1 finney);
     _;
    if (msg.value > _amount)
        if(!msg.sender.send(msg.value - _amount * 1 finney))throw;
    }
```

You can implement some getters and setters and you'll have a decent contract.

[Events](http://solidity.readthedocs.io/en/develop/contracts.html#events) are used by many UIs from Dapps to comunicate with the blockchain.
We didn't go over this, because it is pretty basic.
This can also be very useful for debugging and/or error catching.
Something else you can look at is the [Security Considerations](http://solidity.readthedocs.io/en/develop/security-considerations.html) and the [Common Patterns](http://solidity.readthedocs.io/en/develop/common-patterns.html) from the Solidity docs.

### The complete code
```
pragma solidity ^0.4.8;

contract Supplier {

    address owner;
    uint public priceInFinney;
    int public stock;
    
    modifier onlyOwner(){
        if(owner == msg.sender){
            _;
        }
    }
    
    event error(string message);
    event success(string message);
    event success(string message,uint value);
    
    /* this function is executed at initialization and sets the owner of the contract */
    function Supplier(uint price,int defaultStock) { 
        owner = msg.sender;
        priceInFinney = price;
        stock = defaultStock;
    }
    
    function withdrawAll() onlyOwner(){
        if(!owner.send(this.balance)){error("send back money Failed"); throw;}
    }
    
    function withdraw(int amountInFinney) onlyOwner(){
        if(!owner.send(uint256(amountInFinney * 1 finney))){error("send back money Failed"); throw;}
    }

    /* Function to recover the funds on the contract */
    function kill() onlyOwner()  {
        selfdestruct(owner); 
    }
    
    function buyStock(int amount) payable returns (bool){
        
        if(stock - amount >= 0 && ((uint(amount) * priceInFinney)*1 finney) - 1 finney <= msg.value ){
            stock -= amount;
            return true;
        }else{
            if(!msg.sender.send(msg.value)){
                error("send back money Failed");
                throw;
            }
            error("stock is insufficient and/or not enough ether has been send");
            return false;
        }
    }
    
    function setPrice(uint newPriceInFinney){
        priceInFinney = newPriceInFinney;
        success("Price has been set",newPriceInFinney);
    }
    
    
   function () payable {
        error("Something went wrong");
        throw;
    }
}

contract VendingMachine {

    address owner;
    uint public finneyPrice;
    address supplier;
    address stakeholder;
    int public maxStock;
    int public minStock;
    int public stock;
    int public users;
    int public adminUsers;
    address[] internal accounts;
    address[] internal admins;
    address[] internal stakeholders;
    Supplier s ;
    
    event error(string message);
    event success(string message);
    event success(string message, uint value);
    
    modifier restrictAccessTo(address[] _collection){
        if(msg.sender == address(this)){_;return;}

        for(uint i = 0; i < _collection.length; i++) {
            if (_collection[i] == msg.sender) {
                _;
                return;
            }
        }        
        
        if(msg.value > 0){
            if(!msg.sender.send(msg.value)) throw;
        }
    
    }
    
    modifier onlyOwner(){
        if(owner == msg.sender){
            _;
        }
        
    }

    modifier costs(uint _amount) {
        require(msg.value >= _amount * 1 finney);
        _;
        if (msg.value > _amount)
            if(!msg.sender.send(msg.value - _amount * 1 finney))throw;
    }
    

    /* this function is executed at initialization and sets the owner of the contract */
    function VendingMachine(int max, int min, uint price) { 
        owner = msg.sender;
        maxStock = max;
        minStock = min;
        stock = maxStock;
        finneyPrice = price;
        addUser(msg.sender);
        adminUsers++;
        admins.push(msg.sender);
    }

    /* Function to recover the funds on the contract */
    function kill() onlyOwner()  {
        selfdestruct(owner); 
    }
    
    function pay() payable restrictAccessTo(accounts) costs(finneyPrice ){
        if(stock>0){
            
            if(stakeholders.length > 0){
                divideProfit();
            }else{
                error("No stakeholders available");
            }
            
            stock--;

            if(stock == minStock){ this.resupply(maxStock-stock);}
        }else{
            error("Not enough stock to buy a product. Please wait for a resupply.");
            throw;
        } 
        
        
    }
    
    function addStakeholder(address stakeholder) onlyOwner(){
        stakeholders.push(stakeholder);
        success("Stakeholder has been added");
    }
    
    function divideProfit() internal{
        uint profit = ((finneyPrice-s.priceInFinney()) * 1 finney);
        uint share = profit / stakeholders.length;
        
        for(uint x = 0; x < stakeholders.length; x++) {
            if(!stakeholders[x].send(share)) throw;
        }
        
        success("Share has been divided",share);
    }
    
    function resupply(int amount) restrictAccessTo(admins) payable returns (int) {
        if(amount<=0 && stock + amount > maxStock) throw;
        //msg.value minus the supplier price has been added here because the value from the current call isn't added yet to to contract balance if it is an automatic refill.
        uint weiToSend = (uint256(amount) * (s.priceInFinney() * 1 finney)) + msg.value-s.priceInFinney();
        if(!s.buyStock.value(weiToSend)(amount)) throw;
        stock += amount;
        success("Stock has been resupplied",uint(amount));
        return stock;
    }
    
    function setSupplier(address a) onlyOwner() {
        supplier = a;
        s = Supplier(supplier);
        success("supplier has been set");
    }
    
    function setPrice(uint newPrice) onlyOwner() {
        finneyPrice = newPrice;
        success("Price has been set",newPrice);
    }
    
    function setMaxStock(int newStock) restrictAccessTo(admins) {
        if(stock>newStock || minStock>newStock) throw;
        maxStock= newStock;
        success("New max stock",uint(newStock));
    }
    
    function setMinStock(int newStock) restrictAccessTo(admins) {
        if(maxStock<newStock || stock< newStock) throw;
        minStock = newStock;
        success("New max stock",uint(minStock));
    }
    
    function addUser(address user){
        for(uint x = 0; x < accounts.length; x++) {
            if (accounts[x] == user) {
                throw;
            }
        }
        accounts.push(user);
        users++;
        success("User has been added");
    }
    
    function removeUser(address user) restrictAccessTo(admins){
        for(uint x = 0; x < accounts.length; x++) {
            if (accounts[x] == user) {
                //To fill the gap in the array
                accounts[x] == accounts[accounts.length-1];
                delete accounts[accounts.length-1];
                users--;
                success("Account has been removed from users");
                break;
            }
        }
    }
    
    function removeAdmin(address admin) restrictAccessTo(admins){
        for(uint x = 0; x < admins.length; x++) {
            if (admins[x] == admin) {
                //To fill the gap in the array
                admins[x] == admins[admins.length-1];
                delete admins[admins.length-1];
                adminUsers--;
                success("Account has been removed from admins");
                break;
            }
        }
    }
    
    function deleteAccount() restrictAccessTo(accounts){
        removeUser(msg.sender);
        removeAdmin(msg.sender);
    }
    
    function addAdmin(address admin) restrictAccessTo(admins){
        //add admin to the accounts list
        addUser(admin);
        for(uint x = 0; x < admins.length; x++) {
            if (admins[x] == admin) {
                throw;
            }
        }
        admins.push(admin);
        adminUsers++;
        
        success("Admin has been added");
    }
    
    
   function () {
       error("Something went wrong");
        throw; // throw reverts state to before call
    }
}

```

## Deploying and using the contract
There are multiple ways to deploy a contract.
We will be discussing 3 of them.

### Deploy

<div class="row" style="margin: 0 auto 2.5rem auto; width: 100%;">
    <div class="col-md-offset-3 col-md-6" style="padding: 0;">
	    {% include image.html img="/img/blockchain/deploy-steps.png" alt="Mist contract deployment steps" title="These are the steps necessary to deploy a contract" %}
    </div>
</div>

<div class="row" style="margin: 0 auto 2.5rem auto; width: 100%;">
    <div class="col-md-offset-3 col-md-6" style="padding: 0;">
	    {% include image.html img="/img/blockchain/transaction-box.png" alt="Transaction Box" title="These are the steps necessary confirm a transaction" %}
    </div>
</div>

#### From Mist
This is the __easiest__ way to upload a contract to the blockchain.
Simply go to the contract section in Mist.
Select 'deploy new contract'. 
Then select the account you want to use to upload the contract.
In our case this will be the owner, in and from the contract.
Then paste the source code of the contract in the designated section.
Select the contract in the dropdown and press deploy.
A confirmation will be asked from Mist.
Just type the password and the contract is ready to get mined! 
So don't forget to start the miner of the node.
When uploading huge contracts, it is sometimes necessary to increase the amount of gas added to the contract.

<div class="row" style="margin: 0 auto 2.5rem auto; width: 100%;">
    <div class="col-md-offset-3 col-md-6" style="padding: 0;">
	    {% include image.html img="/img/blockchain/command-line-upload-requirements.png" alt="The requirements for watching" title="Where to find the necessary data to watch a contract" %}
    </div>
</div>

There is a __watch function__ in Mist. 
Which allows us to watch a __contract__ that is already __in use__ on the chain.
So we can use this contract without uploading it again and creating a new version of this contract. 
For this to work we'll need the address and the abi from the in-use contract to be able to watch it.
The abi can be found in remix in the contract section on the right side under the contract.
There we can select "Contract details (bytecode, interface etc.)".
And under the interface section we can get the abi.

<div class="row" style="margin: 0 auto 2.5rem auto; width: 100%;">
    <div class="col-md-offset-3 col-md-6" style="padding: 0;">
	    {% include image.html img="/img/blockchain/remix-deploy-contract.png" alt="Deploying from remix" title="The steps to deploy from Remix" %}
    </div>
</div>

#### From Remix
Uploading a contract directly from the Remix IDE is not a problem.
Go to the contract section on the right side of the screen. 
Expand the contract and press the (red) __create__ button.
The mist transaction box will popup where a password is needed.
Adjust the amount of gas when needed (only with large contracts).
When the contract has deployed successfully, the address of the contract will be returned.

#### From commandline
It's recommended to use Remix to compile the contract before using the commandline, but the commandline can also do the compiling (as seen [here](https://ethereum.gitbooks.io/frontier-guide/content/compiling_contract.html)).

Now for __deploying__ it onto the chain through the command line. 
Let's go again to the contract section on the right hand side of the Remix IDE and expand the contract that is ready to deploy. 
Click on "Contract details (bytecode, interface etc.)" and go to the web3 deploy section.
__2 commands__ will be displayed.
Enter the 2 commands __separately__ in the javascript (geth) console.
Once the contract is mined, the contract address will be shown.

<div class="row" style="margin: 0 auto 2.5rem auto; width: 100%;">
    <div class="col-md-offset-3 col-md-6" style="padding: 0;">
	    {% include image.html img="/img/blockchain/using-a-contract.png" alt="Using the contract" title="Steps to use the pay method" %}
    </div>
</div>

### Using a contract.
Using the contract and its functions is recommended through Mist for beginners.
To use the contract with __mist__, go again to the contract section and let's select the contract we want to use.
The public variables of the contract and the methods that are defined in the contract are shown.
When selecting a function, mist will automatically ask for the required __parameters__. 
Then press the execute button and type in the password.
Mist does a __call__ to the method before the actual transaction.
This means it can __predict__ if the transaction will succeed and it will clearly warn us if it won't succeed.
When failing, always check if the transaction is payable.
As well check if there was enough ether send.
Most people forget to send ether with the transaction.

## Ethereum and Java with web3j
> Web3j is a lightweight, reactive, type safe Java and Android library for integrating with nodes on Ethereum blockchains.

<div class="row" style="margin: 0 auto 2.5rem auto; width: 100%;">
    <div class="col-md-offset-3 col-md-6" style="padding: 0;">
	    {% include image.html img="/img/blockchain/web3j_network.png" alt="web3j image" title="Explaining the web3j" %}
    </div>
</div>


The full documentation is available at [GitHub](https://github.com/web3j/web3j) or the [docs](https://docs.web3j.io/) from the web3j website.
We will give a brief overview about the most used and basic functions it supports.

### Converting solidity contract to java class
First off, one of the biggest advantages of using web3j is that we can make a __java class from our contract__. 
This makes the interaction with the contract fairly easy. 
How great is that!
The process goes as follows: compile the solidity source code to .abi and .bin then create a java class from these 2 files. 
To do so, we'll need to install __[Solidity](https://solidity.readthedocs.io/en/latest/installing-solidity.html)__ locally with the command `npm install -g solc`.
Thanks to the solidity, we just installed globally with npm, we can generate our .abi and .bin files.
Where \<contract> is the name of our contract and \<output-dir> the directory where we want to store the 2 created files.
```
solc <contract>.sol --bin --abi --optimize -o <output-dir>/
```
Now we have 2 files, but still no java class.
Here is where the __[Command Line Tools](https://docs.web3j.io/command_line.html)__ comes in.
Download and unzip it. 
Change your directory in the command prompt to the unzipped directory and we are ready to make our java class with following command.
```
web3j solidity generate /path/to/<smart-contract>.bin /path/to/<smart-contract>.abi -o /path/to/src/main/java -p com.your.organisation.name
```




### Implementation
We need to add our __dependency__.
#### Maven
```
Java 8:

    <dependency>
        <groupId>org.web3j</groupId>
        <artifactId>core</artifactId>
        <version>2.2.1</version>
    </dependency>


Android: 

    <dependency>
        <groupId>org.web3j</groupId>
        <artifactId>core-android</artifactId>
        <version>2.2.1</version>
    </dependency>
```

#### Gradle
```
Java 8:

    compile ('org.web3j:core:2.2.1')


Android:

    compile ('org.web3j:core-android:2.2.1')
```

#### Start client
[Start](#setting-up-your-ethereum-client) your Ethereum client if you didn't already have one running.

#### Start requests
For our purpose we'll make a Web3jService.java class in our spring boot application.
In this __service__ we will create a simple function to get our client version.

 ```java
@Service
public class Web3jService {
    private Web3j web3;

    public Web3jService() {
        this.web3  = Web3j.build(new HttpService());
    }

    public String getClientVersion() throws IOException, ExecutionException, InterruptedException {
        Web3ClientVersion web3ClientVersion = web3.web3ClientVersion().send();
        String clientVersion = web3ClientVersion.getWeb3ClientVersion();
        return clientVersion;
    }
}
 ```

#### Using the converted smart contract
Previously we talked about __converting__ a solidity __contract__ to a java class.
We added this java class to our project in the model folder.
So let's use this class. 
In our example the class is called Vending.
To load our vending contract, we need __credentials__ to sign in on the chain.
It is recommended to use the credentials of the wallet that uploaded the contract.


 ```java
@Service
public class Web3jService {
    private Web3j web3;
    Vending vendingContract;
    private Credentials credentials;

    public Web3jService() {
        this.web3  = Web3j.build(new HttpService());//default http://localhost:8545/
        this.credentials  = WalletUtils.loadCredentials("password-of-local-wallet",   "path-to-local-wallet");
        vendingContract = Vending.load("id-of-contract",web3,credentials, ManagedTransaction.GAS_PRICE, Contract.GAS_LIMIT);
    }

   ...
}
 ```
#### Calling constant methods a.k.a. getters
So now we have the contract loaded in our vendingContract object. 
Let's get some values of the contract. We will create following functions: getStock, getMaxStock and getMinStock. 
Because we are __not altering the state__ of the contract, but just using __'getters'__ we use the Type property.
Note that these variables must be __public__ on the solidity contract level and that these are not functions but variables!

 ```java
    ...

    public Integer getStock() throws IOException, ExecutionException, InterruptedException, CipherException {
        Type result = vendingContract.stock().get();
        return Integer.parseInt(result.getValue().toString());
    }

    public int getMaxStock() throws ExecutionException, InterruptedException {
        Type result = vendingContract.maxStock().get();
        return Integer.parseInt(result.getValue().toString());
    }

    public int getMinStock() throws ExecutionException, InterruptedException {
        Type result = vendingContract.minStock().get();
        return Integer.parseInt(result.getValue().toString());
    }
    ...
}
 ```
#### Transactions
So now let's use the famous __transactions__. 
Our contract contains an array in which users are stored through their address (wallet ID). This is an extra security check on the contract level. 
So only users can __invoke__ certain methods. The function addNewUser() adds an address (wallet ID) to the array. 
First we create an Address from the walletID. Then we wait until there is a __TransactionReceipt__ from the method. 
A transactionreceipt will be given when the transaction is __mined__ / added to the chain.

 ```java
    ...

    public boolean addNewUser(String walletID) throws ExecutionException, InterruptedException {
        Address newAddress = new Address(walletID);
        //will wait till block is mined, otherwise no transaction receipt
        TransactionReceipt transactionReceipt= vendingContract.add(newAddress).get();
        return true;
    }

    ...
}
 ```

 With the transaction above, the person (credentials) who loaded the contract at the start needs to __pay__ for this transaction. 
 As well on security level, you want to invoke the transaction from the current __logged in user__. 
 Because there are admins and normal users and they have different permissions. 
 Let's take a look how we can invoke a function from the contract with the __credentials__ of the current user.

 So we'll create a doEthFunction() to build our own custom function. 
 The currentwalletID and passwordWallet are from the current logged in user. 
 We have a buyOne() and setMaxStock() function who uses the doEthFunction. 
 Depending on which function invokes the doEthFunction we create a new function.
 
 The pay method has no incoming parameters, but the setMaxStock has an integer which needs to be the new minimum stock. 



 ```java
 ...
  if(func.equalsIgnoreCase("pay")){
            function = new Function("pay", Arrays.<Type>asList(), Collections.<TypeReference<?>>emptyList());
        }else if (func.equalsIgnoreCase("setmax")){
            function = new Function("setMaxStock", Arrays.<Type>asList(new Int256("amount to set max")), Collections.<TypeReference<?>>emptyList());
        }
...
 ```
 Before we can invoke these functions with the current logged in user __account__, this account needs to be unlocked. When you create an account, it is by default locked. 
 To __unlock__ accounts we use __parity__. 
 To use parity, we add `Parity parity` as new local variable and add following code to our constructor from the service class: `this.parity = Parity.build(new HttpService());`. 
 Then we unlock the given walletID and password with parity. Which is as well default set on http://localhost:8545/.
 ```java
 ...
 PersonalUnlockAccount currentacc = parity.personalUnlockAccount(currentwalletID,passwordWallet, duration).send();
        if(currentacc==null){ 
            throw new Exception("CurrentAccount doens't exist!");
        }
        if (currentacc.accountUnlocked()) {
            //invoke the function
        }
...
 ```
When the account is unlocked, we can start making and invoking our transaction. 
First we need the next available nonce. A quick refresh about the nonce. 
>The __nonce__ is an increasing numeric value which is used to __uniquely__ identify transactions. 
A nonce can only be used once and until a transaction is mined, it is possible to send multiple versions of a transaction with the same nonce, however, once mined, any subsequent submissions will be rejected.

When we got the nonce, we'll encode the function and start building the transaction. A transaction needs following parameters:
1. Address (wallet ID) from the sender
2. Valid nonce
3. The gas price
4. The gas limit
5. The receiver's address (wallet ID).
6. The amount of WEI you wish to send with the transaction (1WEI = 10^18 Ether)
7. The encoded function

After making a valid transaction we send it to the chain to be mined. 
The easiest way to do this is to use parity. 
We just __sign and send__ the transaction. 
In the response, we'll check for the __transactionhash__. 
If this is null we know there was something wrong with the transaction. 
Some causes may be: not enough ether send, gas limit is below the asked gas, wrong credentials, bad nonce. 
But if everything was successful and valid, the transaction is pending and waiting to be mined. 
We'll keep asking for the transaction receipt. 
When it is not null anymore, the transaction is added to the blockchain.



```java
...
            //get the next available nonce
            EthGetTransactionCount ethGetTransactionCount = web3.ethGetTransactionCount("wallet ID current logged in user", DefaultBlockParameterName.LATEST).sendAsync().get();
            BigInteger nonce = ethGetTransactionCount.getTransactionCount();
            //encode the function
            String encodedFunction = FunctionEncoder.encode(function);
            //start building transaction
            org.web3j.protocol.core.methods.request.Transaction transaction = org.web3j.protocol.core.methods.request.Transaction.createFunctionCallTransaction("wallet ID current logged in user", nonce, Transaction.DEFAULT_GAS, gaslimit, "ID vending contract","wei amount", encodedFunction);
            org.web3j.protocol.core.methods.response.EthSendTransaction 
            //sign and send transaction through parity (account is unlocked)
            transactionResponse =parity.personalSignAndSendTransaction(transaction,passwordWallet).send();
            final String transactionHash = transactionResponse.getTransactionHash();
            //if the hash is null it means the transaction was not succesfull made and send.
            if (transactionHash == null) {
                throw new Exception(transactionResponse.getError().getMessage());
            }
            EthGetTransactionReceipt transactionReceipt = null;
            //keep asking for transaction receipt untill it is not null. So transaction is mined.
            do {
                transactionReceipt = web3.ethGetTransactionReceipt(transactionHash).send();
            } while (!transactionReceipt.getTransactionReceipt().isPresent());

            return doReturn(func);
...
```

When we put all the small pieces of code from above in the service class you'll get following new code:
```java
    ...
   public int setMaxStock(int amount, String currentwalletID, String passwordWallet) throws InterruptedException, ExecutionException, CipherException, IOException {
        return doEthFunction(currentwalletID,passwordWallet,"setmax",amount);
    }

    public int buyOne(String currentwalletID,String passwordWallet) throws IOException, CipherException, ExecutionException, InterruptedException {
        return doEthFunction(currentwalletID,passwordWallet,"pay",0);
    }


   public int doEthFunction(String currentwalletID,String passwordWallet, String func,int amountStockup) throws InterruptedException, ExecutionException, CipherException, IOException {

        Function function=null;
        BigInteger ether;
        BigInteger am = BigInteger.valueOf(amountStockup);
        int stock = getStock();

        if(func.equalsIgnoreCase("pay")){
            function = new Function("pay", Arrays.<Type>asList(), Collections.<TypeReference<?>>emptyList());
            ether = Convert.toWei(String.valueOf("value of a soda"), Convert.Unit.ETHER).toBigInteger().add(Transaction.DEFAULT_GAS.multiply(gaslimit));
        }else if (func.equalsIgnoreCase("setmax")){
            function = new Function("setMaxStock", Arrays.<Type>asList(new Int256(am)), Collections.<TypeReference<?>>emptyList());
            ether = Convert.toWei("0.0", Convert.Unit.ETHER).toBigInteger();
        }

        //unlock account
        PersonalUnlockAccount currentacc = parity.personalUnlockAccount(currentwalletID,passwordWallet, duration).send();
        if(currentacc==null){
            throw new Exception("CurrentAccount doens't exist, is null!");
        }
        if (currentacc.accountUnlocked()) {

            EthGetTransactionCount ethGetTransactionCount = web3.ethGetTransactionCount("wallet ID current logged in user", DefaultBlockParameterName.LATEST).sendAsync().get();
            BigInteger nonce = ethGetTransactionCount.getTransactionCount();
            //encode the function
            String encodedFunction = FunctionEncoder.encode(function);
            //start building transaction
            org.web3j.protocol.core.methods.request.Transaction transaction = org.web3j.protocol.core.methods.request.Transaction.createFunctionCallTransaction("wallet ID current logged in user", nonce, Transaction.DEFAULT_GAS, gaslimit, "ID vending contract","wei amount", encodedFunction);
            org.web3j.protocol.core.methods.response.EthSendTransaction 
            //sign and send transaction through parity (account is unlocked)
            transactionResponse =parity.personalSignAndSendTransaction(transaction,passwordWallet).send();
            final String transactionHash = transactionResponse.getTransactionHash();
            //if the hash is null it means the transaction was not succesfull made and send.
            if (transactionHash == null) {
               throw new Exception(transactionResponse.getError().getMessage());
            }
            EthGetTransactionReceipt transactionReceipt = null;
            //keep asking for transaction receipt untill it is not null. So transaction is mined.
            do {
                transactionReceipt = web3.ethGetTransactionReceipt(transactionHash).send();
            } while (!transactionReceipt.getTransactionReceipt().isPresent());
            return doReturn(func);
        }else{
            throw new Exception("Account is locked");
        }


    }

    public int doReturn(String function) throws InterruptedException, ExecutionException, CipherException, IOException {
        if(function.equalsIgnoreCase("pay")){
            return getStock();
        }else if(function.equalsIgnoreCase("setmax")){
            return getMaxStock();
        } else {
            return 0;
        }
    }
    ...
}
 ```

#### New wallet
Making a new wallet can be done with __parity__ in 1 command. 
Just give a password and it will create the new wallet. 
In our example we return the address (wallet ID) when creating the new wallet. 
If you want to send it async, just replace the `.send()` by `.sendAsync().get();`.

```java
    public String makeNewWallet(String password) throws ExecutionException, InterruptedException, IOException {
        String res = parity.personalNewAccount(password).send().getAccountId();
        return res;
    }
```

#### Observers
But wouldn't it be great if we could __observe__ our blocks and transactions? 
Of course! 
This is not a problem with web3j. 
The following function will subscribe to the transactions and print the __hashes__ from the transactions.
```java
...
public void subscribeToTransactionsandBlocks(){
        System.out.println("started subscription");
        //pending transactions that are not yet added to the chain
        subscription = web3.pendingTransactionObservable().subscribe(tx -> {
            System.out.println("Following transaction is pending: " + tx.getHash());
        });

        //Transactions who are added to the blockchain
        subscription1 = web3.transactionObservable().subscribe(tx -> {
            System.out.println("Transaction is added to the blockchain: " + tx.getHash());
        });

        subscription2 = web3.blockObservable(false).subscribe(block -> {
        //Every transaction in the block
            for (EthBlock.TransactionResult transactionResult:
                    block.getBlock().getTransactions() ) {
                System.out.println("transaction in block equals: " + transactionResult.get().hashCode());
            }
        });

    }
...
 ```

### Events
The smart contract fires an event when there occurs an error or when the transaction succeeded.
To improve our backend web3 service, we could listen to these events instead of waiting for the transaction receipt.
These events have custom error strings.
Due to the end of our intern we couldn't implement the event listener in the backend.

# Conclusion
We learned a lot from researching the technologies we used for this project, but there is still a lot more to learn out there.
The technologies are evolving at an enormous rate.
As we see it right now, there are two big options to create your project in blockchain. 
You have Ethereum (Solidity) and Hyperledger (Go) in combination with BigchainDB or the InterPlanetary File System as data storage.
The InterPlanetary File System (ipfs) is protocol designed to store files in a decentralized manner.
For more information about Ethereum, Hyperledger and BigChainDB you can view our [previous post]("/2017-05-10-Blockchain-Introduction#existing-platforms").
The technologies mentioned above are the "blockchain stacks" we see the most in the current ongoing projects.



