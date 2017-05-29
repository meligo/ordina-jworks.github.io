---
layout: post
authors: [ken_coenen, jeroen_de_prest, kevin_leyssens]
title: 'Blockchain hands-on'
image: 
tags: [Blockchain, Ethereum, Smart contracts, Solidity, Geth, spring boot, java]
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
8. [Ethereum and Java with web3j](#ethereum-and-java-with-web3j)
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

//TODO check commando
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
A good reference to learn Solidity can be found [here](https://learnxinyminutes.com/docs/solidity/) or the [Solidity docs](https://solidity.readthedocs.io/en/develop/). 
If you are creating a token contract please take the [ERC20 token standard](https://theethereum.wiki/w/index.php/ERC20_Token_Standard) into account.

```
    pragma solidity ^0.4.8;
        
        contract VendingMachine {
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
  
  We will also need to add `int public stock;`, `int public maxStock`, `int public minStock` and `uint priceInFinney;` and these will need to be set in the constructor. 
  I will be setting the stock and maxStock to 50, the minStock to 45(for testing purposes) and the price to 20. 
  The price is in finney, finney is a sub unit under ether. 
  1 finney is 0.001 ether. 
  Solidity expects us to use wei because that is the smallest unit for ether. 
  We don't give Solidity our data in wei because it just isn't user-friendly because 1 ether is 1000000000000000000 wei. 
  That is wei we use the price in finney and then multiply by 1 finney to get the wei value. 
  Currencies are also always a uint.
  
  Now the explanation for the function.
  
  `` payable `` this is a modifier to allow this method to be paid.
  
  ``msg.value`` Gives the amount of ether, in wei, that the user send with their transaction. 
  
   ``if(!client.send(change)) throw;`` Try to send the change back to the sender when they send too much. 
    if it fails then revert state to before the call was made.
    
   Then we just lower the stock and that is our first version of the pay method we will add the auto refill and distribution between stakeholder and contract in a later step. 

```
  function resupply(int amount) payable returns (int) {
  if(amount<=0 && stock + amount > maxStock) throw;
  
  stock += amount;
          
    return stock;
  }
 ```
 
This is a pretty straight forward function. 
First we check if the amount given is bigger then zero and then we will check if the maxStock isn't exceeded by adding the amount to the current stock. 
Then we add the amount to the stock. 
The supplier will be added later as another contract.
Now that we have the resupply function we can add the automatic resupply functionality ot the pay method.
 
```
  if(stock == minStock) this.resupply(maxStock-stock);
```

Add this below the `stock--;`. 
We check our new stock if it equals our minStock. 
 If it does we then call our resupply function to fill to maxStock again.
 
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
 
Here we use the `internal` modifier to make sure users can't call them from the chain directly.
First we take half to keep on the machine although we don't need it yet. 
Then we will also need to create an array of stakeholders `address[] private stakeholders;` 
Then we divide the profit by the amount of stakeholders (don't worry about dividing by 0).
End we will run a loop through the array and try to send the shares to the stakeholders.
Then we still need to add the function to the pay method.

```

if(stock != maxStock && stakeholders.length > 0)
    divideProfit();

```

Add this above the `stock--;`.
The `stock!=maxStock` is to make sure when we add the supplier we will have enough money to pay him because we only get the money from the client after the pay function has been executed. <-- change this if contract test worked
Here we also prevent the divide by 0 problem.

You can go even more advanced with this if different stakeholders have a bigger stake then others. 
I would then recommend to use a [struct](https://solidity.readthedocs.io/en/develop/structure-of-a-contract.html#structs-types) in combination with [mapping](http://solidity.readthedocs.io/en/develop/types.html#mappings).
Then use the struct as valuetype for the mapping and the address as the key type.

Now we will need to create the supplier contract.
This can be in the same file. 

```

contract Supplier {

    address owner;
    uint public priceInFinney;

    /* this function is executed at initialization and sets the owner of the contract */
    function Supplier() { 
        owner = msg.sender;
        priceInFinney = 10;
    }
    
    function withdrawAll(){
        if(!owner.send(this.balance)) throw;
    }
    
    function withdraw(int amountInFinney){
        if(!owner.send(uint256(amountInFinney * 1 finney))) throw;
    }

    /* Function to recover the funds on the contract */
    function kill() onlyOwner()  {
        selfdestruct(owner); 
    }
    

    
    function setPrice(uint newPriceInFinney){
        priceInFinney = newPriceInFinney;
    }
    
    
   function () payable {
        
    }
}

```

This is a pretty basic contract that has the price of the product from this supplier and a withdraw functionality to withdraw the money to the owner account.
`this.balance` is built in to get the balance of a contract.

We will need to make some changes to the first contract. 
We need to add a place to store the supplier contract `Supplier s;` and the supplier contract address.
Then we will make a method to set this supplier.

```

function setSupplier(address a) {
        supplier = a;
        s = Supplier(supplier);
}

```

In the resupply method we will now add this line under the first if.

```
if(!supplier.send((uint256(amount) * (s.priceInFinney() * 1 finney)))) throw;
```

Here we try to pay the supplier by getting the price the supplier is asking with `s.priceInFinney()` and making it a wei value by multiplying by 1 finney and multiplying that by the amount we want to order. 
If it fails we will revert state to before the call was done.

Now for the security part now we were protecting some of our functions with an if statement like `if(msg.sender == owner)` but there is a better way to do this.
We can use [modifiers](http://solidity.readthedocs.io/en/develop/common-patterns.html#restricting-access).

```
modifier onlyOwner(){
        if(owner == msg.sender){
            _;
        }
        throw;
    }
```

We just us the if in the modifier and if it passes we continue to the function the user requested with `_;`. 
To use this modifier we just add after the function name and brackets.

```

function kill() onlyOwner()  {
    selfdestruct(owner); 
}

```

We can then remove the if-statements and replace them like you can see in the example above.
With these modifiers we can also create an account system. 

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

The modifier will restrict access to the array of addresses given to it. 
Then we use an add function and a remove function to add and remove our users.
Removing is a litle bit special because when you use `delete x` you get gaps in the array. 
So I copy the last account in the array over the one I want to remove and the remove the last account.
You can also do this the with multiple user groups like I did with admins and users or use method overloading.

Then the only thing we are missing are a few getters and setters and than this is basically our PoC contract.

## Deploying and using the contract
There are three options we will discuss when it comes to deploying the contract.

### From Mist
This is what I recommend to use when you want to upload a contract to the blockchain.
You simply go to the contract section and then select Deploy new contract. 
Then you select the account you want to use to upload the contract and you paste the source code of the contract you want to use in the designated section.
You select the contract in the dropdown and press deploy.

If a contract you want to use is already on the chain then you can use the watch contract feature. 
You will need the address and the abi from the contract to be able to watch it.

### From Remix
You can directly upload the contract from the Remix IDE. 


### From commandline


## Ethereum and Java with web3j
> Web3j is a lightweight, reactive, type safe Java and Android library for integrating with nodes on Ethereum blockchains.

![image ethereum web3j](https://raw.githubusercontent.com/web3j/web3j/master/docs/source/images/web3j_network.png "json-rpc blockchain java application")


The full documentation is available at [GitHub](https://github.com/web3j/web3j) or the [docs](https://docs.web3j.io/) from the web3j website.
We will give you a brief overview about the most used and basic functions it supports.

### Converting solidity contract to java class
First off, one of the biggest advantages of using web3j is that you can make a __java class from your contract__. This makes the interaction with the contract fairly easy. How great is that!
The process goes as follows: compile the solidity source code to .abi and .bin then create a java class from these 2 files. 
To do so, you'll need to install [Solidity](https://solidity.readthedocs.io/en/latest/installing-solidity.html) locally with the command `npm install -g solc`.
Thanks to the solidity we just installed globally with npm we can generate our .abi and .bin files.
Where \<contract> is the name of our contract and \<output-dir> the directory where you want to store your 2 created files.
```
solc <contract>.sol --bin --abi --optimize -o <output-dir>/
```
Now we have 2 files, but still no java class.
Here is where the [Command Line Tools](https://docs.web3j.io/command_line.html) comes in.
Download and unzip it. 
Change your directory in the command prompt to the unzipped directory and we are ready to make our java class with following command.
```
web3j solidity generate /path/to/<smart-contract>.bin /path/to/<smart-contract>.abi -o /path/to/src/main/java -p com.your.organisation.name
```
So now the web3j command line tool has created a .java file from the previous generated .bin and .abi file.

Ok, we've got our smart contract converted to a java class, what now?



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
Start your Ethereum client if you didn't already have one running.
```
geth --rpcapi personal,db,eth,net,web3,parity --rpc --testnet
```

#### Start requests
For our purpose we'll make a Web3jService.java class in our spring boot application.
In this __service__ we will create a simple function to get our client version.
Note that every transaction on the ethereum chain requires gas to be excuted.
But this function doesn't require to do a transaction, so you don't have to pay for this function.

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

 #### Using the converted java class
Previously we talked about __converting__ a solidity __contract__ to a java class.
We added this java class to our project in the model folder.
So let's use this class. 
In our example the class is called Vending.
To load our vending contract, we need __credentials__ to sign in on the chain.
It is recomended to use the credentials of the wallet who uploaded the contract.


 ```java
@Service
public class Web3jService {
    private Web3j web3;
    Vending vendingContract;
    private Credentials credentials;

    public Web3jService() {
        this.web3  = Web3j.build(new HttpService());
        this.credentials  = WalletUtils.loadCredentials("password-of-local-wallet",   "path-to-local-wallet");
        vendingContract = Vending.load("id-of-contract",web3,credentials, ManagedTransaction.GAS_PRICE, Contract.GAS_LIMIT);
    }

   ...
}
 ```
#### Calling getters a.k.a. contant methods
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
In our contract is an array in which users are stored through their walletID. This is an extra security check on the contract level. 
So only users can __invoke__ certain methods. So the function adds a walletID (user) to the array. 
First we create an Address from the walletID. Then we wait untill there is a __TransactionReceipt__ from the method. 
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

 With the transaction above, the person (credentials) who loaded the contract at the start needs to __pay__ for this transaction. Aswell on security level, you want to invoke the transaction from the current __logged in user__. Because there are admins and normal users and they have different permissions. So let's take a look how we can invoke a function from the contract with the __credentials__ of the current user.

 So we'll create a doEthFunction to build our own custom function. We have a pay and setmax value as function.

```java
    ...

   public int doEthFunction(String currentwalletID,String passwordWallet, String func,int amountStockup) throws InterruptedException, ExecutionException, CipherException, IOException {

        Function function=null;
        BigInteger ether = Convert.toWei("0.3", Convert.Unit.ETHER).toBigInteger();;
        BigInteger am = BigInteger.valueOf(amountStockup);
        int stock = getStock();

        if(func.equalsIgnoreCase("pay")){

            function = new Function("pay", Arrays.<Type>asList(), Collections.<TypeReference<?>>emptyList());
            ether = Convert.toWei(String.valueOf(getPriceFinneyToEther()), Convert.Unit.ETHER).toBigInteger().add(Transaction.DEFAULT_GAS.multiply(gaslimit));
            BigDecimal accountBalance = Convert.fromWei(parity.ethGetBalance(currentwalletID,DefaultBlockParameterName.LATEST).send().getBalance().toString() , Convert.Unit.ETHER);
            //check if there is enough money in the wallet. Transaction would automaticly be discarded from the chain, but now we don't need to wait till transaction is verified.
            BigDecimal ethersend = new BigDecimal(ether);
            if(ethersend.compareTo(accountBalance)< 0 ){return getStock();}
        }else if (func.equalsIgnoreCase("setmax")){
            if(amountStockup<getStock()){return doReturn(func);}
            function = new Function("setMaxStock", Arrays.<Type>asList(new Int256(am)), Collections.<TypeReference<?>>emptyList());
            ether = Convert.toWei("0.0", Convert.Unit.ETHER).toBigInteger();
        }

        //unlock accounts
        PersonalUnlockAccount currentacc = parity.personalUnlockAccount(currentwalletID,passwordWallet, duration).send();
        if(currentacc==null){
            System.out.println("CurrentAccount is null!");
            return doReturn(func);
        }
        if (currentacc.accountUnlocked()) {

            EthGetTransactionCount ethGetTransactionCount = web3.ethGetTransactionCount(currentwalletID, DefaultBlockParameterName.LATEST).sendAsync().get();
            BigInteger nonce = ethGetTransactionCount.getTransactionCount();
            String encodedFunction = FunctionEncoder.encode(function);
            org.web3j.protocol.core.methods.request.Transaction transaction = org.web3j.protocol.core.methods.request.Transaction.createFunctionCallTransaction(currentwalletID, nonce, Transaction.DEFAULT_GAS, gaslimit, BlockchainLocalSettings.VENDING_CONTRACT, ether, encodedFunction);
            org.web3j.protocol.core.methods.response.EthSendTransaction transactionResponse =parity.personalSignAndSendTransaction(transaction,passwordWallet).send();
            final String transactionHash = transactionResponse.getTransactionHash();
            if (transactionHash == null) {
                System.out.println(transactionResponse.getError().getMessage());
                return doReturn(func);
            }
            EthGetTransactionReceipt transactionReceipt = null;
            //todo: indien niet toegevoegd door error moet deze niet wachten op de transactionreceipt. Dus een timeout hierop plaatsen?
            do {
                transactionReceipt = web3.ethGetTransactionReceipt(transactionHash).send();
            } while (!transactionReceipt.getTransactionReceipt().isPresent());

            return doReturn(func);
        }else{
            System.out.println("account is locked");
            return doReturn(func);
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

    public int setMaxStock(int amount, String currentwalletID, String passwordWallet) throws InterruptedException, ExecutionException, CipherException, IOException {
        return doEthFunction(currentwalletID,passwordWallet,"setmax",amount);
    }

    public int buyOne(String currentwalletID,String passwordWallet) throws IOException, CipherException, ExecutionException, InterruptedException {
        return doEthFunction(currentwalletID,passwordWallet,"pay",0);
    }
    ...
}
 ```

Wouldn't it be great if we could observe our blocks and trasactions? Of course! This is not a problem with the web3j. Following function will subscribe and print some information about the transactions.
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

# Conclusion

# Recommended reading



