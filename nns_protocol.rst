****************
Design Details
****************

NNS Protocol Specifications 
============================

The URL we usually use on the Internet is as follows,

::

    http://aaa.bbb.test/xxx 

1. HTTP is a protocol, the domain name and protocol will be passed on separately when NNS service is requested. 

2. aaa.bbb.test is the domain name, NNS service request using the hash of the domain name

3. Xxx is the path, the path is not processed at the DNS level, the same goes with nns, if there is a path, it will be processed in other ways. 

Definitions of using strings in NNS protocol
The following are tentative definitions.  

**HTTP**

 `HTTP protocol points to a string, which means it’s an Internet address.`

**addr** 

 `addr protocol points to a string, which means it’s an NEO address. Like: AdzQq1DmnHq86yyDUkU3jKdHwLUe2MLAVv`

**script** 
 
 `script protocol points to a byte[], which means a NEO ScriptHash. Like: 0xf3b1c99910babe5c23d0b4fd0104ee84ffeec2a5`

One and the same domain name is processed differently by different protocols. 

- http://abc.test  may point to http://www.163.com
- addr://abc.test  may point to AdzQq1DmnHq86yyDUkU3jKdHwLUe2MLAVv
- script://abc.test  may point to 0xf3b1c99910babe5c23d0b4fd0104ee84ffeec2a5

.. _namehash:

Detailed Explanation NameHash Algorithm 
========================================

Namehash
---------

The domain name NNS stores are 32byte hashes, rather than the plain text of the original domain name. Here are reasons for this design. 

1. A unified process allows any length of the domain name.

2. It preserves the privacy of the domain name to some extent. 

3. The algorithm for converting a domain name to a hash is called NameHash, 

Domain, Domainarray, and Protocol
----------------------------------

The URL we usually use on the Internet is as follows,

::

    http://aaa.bbb.test/xxx 

1. HTTP is a protocol, the domain name and protocol will be passed on separately when NNS service is requested. 

2. aaa.bbb.test is the domain name, NNS service request using the hash of the domain name

3. Xxx is the path, the path is not processed at the DNS level, the same goes with nns, if there is a path, it will be processed in other ways. 

NNS uses domain name’s array rather domain name, which is a more direct process. 
Domain name aaa.bb.test is converted into byte array as ["test","bb","aa"]
You could invoke the resolution this way

::

    NNS.ResolveFull("http",["test","bb","aa"]);

And let the contract to calculate namehash.

NameHash Algorithm 
--------------------

NameHash algorithm is a way to calculate hash step by step after converting domain name into DomainArray. Its code is as follows:

::

    // algorithm for converting domain names into a hash
    static byte[] nameHash(string domain)
    {
        return SmartContract.Sha256(domain.AsByteArray());
    }
    static byte[] nameHashSub(byte[] roothash, string subdomain)
    {
        var domain = SmartContract.Sha256(subdomain.AsByteArray()).Concat(roothash);
        return SmartContract.Sha256(domain);
    }
    static byte[] nameHashArray(string[] domainarray)
    {
        byte[] hash = nameHash(domainarray[0]);
        for (var i = 1; i < domainarray.Length; i++)
        {
            hash = nameHashSub(hash, domainarray[i]);
        }
        return hash;
    }

Quick Resolution
-----------------

Complete resolution introduces the whole DomainArray and let smart contracts to check every layer’s resolution one by one. Calculating NameHash could also be done on Client, and then is passed into smart contracts. It’s invoked this way:
::

    // query http://aaa.bbb.test
    var hash = nameHashArray(["test","bbb"]);// can be calculated by Client
    NNS.Resolve("http",hash,"aaa");// invoke smart contracts

or

::

    //query http://bbb.test
    var hash = nameHashArray(["test","bbb"]);// can be calculated by Client
    NNS.Resolve("http",hash,"");// invoke smart contracts

You may be thinking why querying aaa.bbb.test is not like this.

::

    // query http://aaa.bbb.test
    var hash = nameHashArray(["test","bbb","aaa"]);// can be calculated by Client
    NNS.Resolve("http",hash,"");// invoke smart contract

We have to consider whether aaa.bb.test has a separate resolver. If aaa.bb.test is sold to someone else, 
it specifies an independent resolver so that it can be queried. If aaa.bb.test does not have a separate resolver, it is resolved by bb.test’s resolver.
So this cannot be queried.

The first query, regardless of whether aaa.bb.test has an independent resolver, can be found. 

Detailed Explanation of Top-level Domain Name
================================================

Function Signature of Top-level Domain Name Contracts
------------------------------------------------------

The function signature is as follows:

::

    public static object Main(string method, object[] args)

Deploying adopts a configuration of parameter 0710, return value 05

Interface of Top-level Domain Name Contract
--------------------------------------------

Top-level domain name’s interface is composed of three parts
Universal interface. It does not require permission verification and can be invoked by everyone.
Owner interface. It is valid only when it’s invoked by the owner signature or the owner script.
Registrar interface. It’s valid only when it’s invoked by the registrar script. 

Universal Interface
--------------------

The universal interface doesn’t need permission verification. Its code is as follows.

::

    if (method == "rootName")
        return rootName();
    if (method == "rootNameHash")
        return rootNameHash();
    if (method == "getInfo")
        return getInfo((byte[])args[0]);
    if (method == "nameHash")
        return nameHash((string)args[0]);
    if (method == "nameHashSub")
        return nameHashSub((byte[])args[0], (string)args[1]);
    if (method == "nameHashArray")
        return nameHashArray((string[])args[0]);
    if (method == "resolve")
        return resolve((string)args[0], (byte[])args[1], (string)args[2]);
    if (method == "resolveFull")
        return resolveFull((string)args[0], (string[])args[1]);

rootName()
~~~~~~~~~~~~

Return the root domain name that the current top-level domain name corresponds to, its return value is a string. 

rootNameHash()
~~~~~~~~~~~~~~

Return NameHash the current top-level domain name corresponds to, its return values are byte[]

getInfo(byte[] namehash)
~~~~~~~~~~~~~~~~~~~~~~~~~~

Return a domain name’s information, its return value is an array as follows

::

    [
        byte[] owner//owner
        byte[] register//registrar
        byte[] resolver//resolver
        BigInteger ttl//TTL
    ]

nameHash(string domain)
~~~~~~~~~~~~~~~~~~~~~~~~

Convert a section of the domain name into NameHash. For example:

::

    nameHash("test") 
    nameHash("abc")

Its return value is byte[]

nameHashSub(byte[] domainhash,string subdomain)	
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Calculate subdomain name’s NameHash. For example:

::

    var hash = nameHash("test");
    var hashsub = nameHashSub(hash,"abc")// calculate abc.test’s namehash

it's return value is byte[]

nameHashArray(string[] nameArray)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Calculate NameArray’s NameHash，aa.bb.cc.test corresponding nameArray is ["test","cc","bb","aa"]

::

    var hash = nameHashArray(["test","cc","bb","aa"]);

resolve(string protocol,byte[] hash,string or int(0) subdomain)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

resolve a domain name 

The first parameter is a protocol 

For example, HTTP maps a domain name to an Internet address. 

For example, addr maps a domain name to an NEO address( which is probably the most common mapping)

The second parameter is the hash of the domain name that is to be resolved. 

The third parameter is the subdomain name that is to be resolved. 

The following code is applied.

::

    var hash = nameHashArray(["test","cc","bb","aa"]);//calculate by Client
    resolve("http",hash,0)//contract resolve http://aa.bb.cc.test

or

::

    var hash = nameHashArray(["test","cc","bb");// calculate by Client
    resolve("http",hash,“aa")//smart resolve http://aa.bb.cc.test

The return type is byte[], how to interpret byte[] is defined by different protocols. 
byte[] saves strings. We will write another document to explore protocols. 

Second-level domain name has to be resolved in the way of 

::

    resolve("http",hash,0). 
    
Other domain names are recommended to be resolved in the way of 

::

    resolve("http",hash,“aa"). 

resolveFull(string protocol,string[] nameArray)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Complete model of domain name resolution

The first parameter is protocol

The second parameter is NameArray

The only difference in this resolution is it verifies step by step whether the ownership is consistent with the registration.

Its return type is the same with resolve.

Owner Interface
-----------------

All of the owner interfaces are in the form of 

::

    owner_SetXXX(byte[] srcowner,byte[] nnshash,byte[] xxx). 
    
Xxx is  scripthash. 

The return value is one-byte array；[0] means succeed; [1] means fail 

The owner interface accepts both direct signature of account address calls and smart contract owner calls.
If the owner is a smart contract, the owner should determine their own authority. 
If it does not meet the conditions, please do not initiate appcall on the top-level domain contract. 

owner_SetOwner(byte[] srcowner,byte[] nnshash,byte[] newowner)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Ownership transfer of domain names. The owner of a domain name could be either an account address or a smart contract. 

srcowner is only used to verify signature when the owner is an account address. It is the address’s scripthash. 

nnshash is the namehash of the domain name that is to be operated. 

newowner is the scripthash of new owners’ address. 

owner_SetRegister(byte[] srcowner,byte[] nnshash,byte[] newregister)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Set up Domain Registrar Contract (Domain Registrar is a smart contract) Domain Registrar parameter form must also be 0710, return 05 
the following interface must be achieved. 

::

    public static object Main(string method, object[] args)
    {
        if (method == "getSubOwner")
            return getSubOwner((byte[])args[0], (string)args[1]);
        ...

        getSubOwner(byte[] nnshash,string subdomain)

Anyone can call the registrar's interface to check the owner of the subdomain.

There is no regulation for other interface forms of the domain name registrar. The official registrar will be explained in the future documentation.

The domain name registrar achieved by the user only need to achieve getSubOwner interface.

owner_SetResolve(byte[] srcowner,byte[] nnshash,byte[] newresolver)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Set up a domain name resolver contract (the domain name resolver is a smart contract) 

The domain name resolver’s parameter form also has to be 0710 and return 05 

the following interface has to be achieved. 

::

    public static byte[] Main(string method, object[] args)
    {
        if (method == "resolve")
            return resolve((string)args[0], (byte[])args[1]);
        ...
    
    resolve(string protocol,byte[] nnshash)

Anyone can call the resolver interface for resolution. 

There are no regulations for other interface forms of domain name resolves. The official resolver will be explained in the future documentation.

The domain name registrar achieved by the user only need to achieve resolve interface.

Registrar Interface
--------------------

There is only one registrar interface that’s called by registrar smart contract. 

register_SetSubdomainOwner(byte[] nnshash,string subdomain,byte[] newowner,BigInteger ttl)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

register a subdomain name
 
nnshash is the namehash of the domain names that are to be operated. 
 
subdomain is the subdomain name that is to be operated. 
 
newowner is the scripthash of the new owner’s address. 
 
TTL is the time to live of the domain name( block height)
 
If succeed, return [1], if fail, return [0]

Detailed Explanation of Owner Contract
========================================

Workings of the Owner Contract
-------------------------------
The owner contract calls the owner_SetXXX interface of top-level domain name contract in the form of Appcall. 

::

    [Appcall("dffbdd534a41dd4c56ba5ccba9dfaaf4f84e1362")]
    static extern object rootCall(string method, object[] arr);
 
The top-level domain name contract will check the call stack, comparing contract it’s called by and the owner that manages the top-level domain name contract.
So only the owner contract of a domain name can manage this domain name. 

Significance of the Owner Contract 
------------------------------------

Users could achieve complex contract ownership through the owner contract. 

For example:

Owned by two persons, dual signature

Owned by more than two persons, operate by voting

Detailed Explanation of Registrar
====================================

Workings of Registrar Contract
------------------------------

The registrar contract calls register_SetSubdomainOwner interface of the top-level domain name in the form of Appcall. 

::

    [Appcall("dffbdd534a41dd4c56ba5ccba9dfaaf4f84e1362")]
    static extern object rootCall(string method, object[] arr);

Top-level domain name contracts will check the call stack, comparing the contract it’s called by and the registrar the top-level domain name contract manages.

So only the specified registrar contract can manage it. 
the registrar interface 
 
The registrar’s parameter form also has to be 0710 and return 05 

::

    public static object Main(string method, object[] args)
    {
        if (method == "getSubOwner")
            return getSubOwner((byte[])args[0], (string)args[1]);
        if (method == "requestSubDomain")
            return requestSubDomain((byte[])args[0], (byte[])args[1], (string)args[2]);
        ...

getSubOwner(byte[] nnshash,string subdomain)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

This interface is the norms and requirements of registrars. 
It has to be achieved because this interface will be invoked to verify rights when a complete resolution is conducted on the domain name. 

nnshash is the hash of the domain name

subdomain is the subdomain name

Return byte[] owner’s address, or blank

requestSubDomain(byte[] who,byte[] nnshash,string subdomain)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

This interface will be used by first come first served registrar. Users call the interface of the registrar to register the domain name. 

Who means who applies

nnshash means which domain name is applied

subdomain means subdomain name applied

Detailed Explanation of the Resolver  
=======================================

The workings of the resolver contract

1. The resolver saves resolution information by itself.

2. The top-level domain name contract calls the resolution interface of the resolver to get resolution information in the way of nep4. 

3. When the resolver sets resolution data, it calls the getInfo interface of the top-level domain name contract to verify the ownership of the domain name in the way of Appcall. 

::

    [Appcall("dffbdd534a41dd4c56ba5ccba9dfaaf4f84e1362")]
    static extern object rootCall(string method, object[] arr);

Any contract could call the getInfo interface of the top-level domain name contract to verify the ownership of the domain name in the way of Appcall. 

Resolver Interface
-------------------

The resolver’s parameter form has be 0710, it returns 05. 

::

    public static byte[] Main(string method, object[] args)
    {
        if (method == "resolve")
            return resolve((string)args[0], (byte[])args[1]);
        if (method == "setResolveData")
            return setResolveData((byte[])args[0], (byte[])args[1], (string)args[2], (string)args[3], (byte[])args[4]);
        ...

resolve(string protocol,byte[] nnshash)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

This interface is the norms and requirements of resolvers. It has to be achieved because this interface will be called for final resolution when a complete resolution is conducted on a domain name. 

Protocol  is the string of the protocol

Nnshash  nnshash is the domains name that’s to be resolved. 

return byte[] is to resolve the data

setResolveData(byte[] owner,byte[] nnshash,string or int[0] subdomain,string protocol,byte[] data)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

This interface is owned by the standard resolver for demo. The owner(currently it only supports the owner of an account address) could call this interface to configure the resolution data. 

owner means the owner of a domain name.

nnshash means set up which domain name

subdomain the set-up subdomain name ( could pass 0; if the set-up is domain name resolution, non-subdomain name passes 0)

protocol means the string of protocols

data means resolves data

Return [1] means succeed, or [0] means fail. 

NNS Domain Name Registration Mechanism
===========================================

The domain name is a scarce and unique resource. 
The core issue that needs to be considered when designing such a system is how to maximize the value of the domain name. 
This does not mean that the higher the price of the domain name is speculated, the better it is. 

But there should be a reasonable process of value discovery and use. 
The first problem faced by the economic model of the domain name system is how to reasonably realize the initial registration and distribution of domain names?

Current Domain Name Registration
----------------------------------

In the registration mechanism of blockchain domain names, there are currently two typical registration methods. 
One is first-come, first-served, such as a Bitshares; and the other is ENS’s auction with sealed bidding registration.
Let’s first look at the specific steps as well as advantages and disadvantages of these two methods.

First come, first served is the main method of DNS domain registration. 
Service providers usually set different prices for different domain names. 
Due to the decentralized nature of blockchain domain names, there will be no complicated pricing in the 
first come, first served model. Either free or charge a unified registration fee. 
First come, first served method is relatively simple to implement, and the user operation is relatively simple. 
However, there is no market pricing process at the time of initial distribution, and there is no multi-participation. 
The domain name value needs to be discovered completely in the secondary market.

When there are few use cases for blockchain domain names, its value is not high and it is not used a lot, 
first come, first served method can be used as a convenient implementation, but when the demand for blockchain domain names increases, 
the first-come-first-served does not reflect the market demand well and does not maximize the value of domain names.

Ethereum’s domain name service ENS adopts a sealed auction registration method. 
Its registration process is composed of bid opening, placing, revealing, and bid winning?. 
The whole process lasts about 7 days, and the main part is placing a bid and revealing a bid process. 
The blockchain is an open and transparent ledger system. In order to achieve sealed bidding, the user’s bid consists of two parts, 
one is the real bid, and the other is the confusion money. 
Others can see your total bid in the system rather than your real bid to avoid the information advantage of later bidding. 
In the bid revealing stage, the user needs to send his own locally saved ciphertext to the smart contract to reveal the true bid. 
After the revealing period is over, the final bid winner will be determined.

The disadvantage of auctions with sealed bidding is that they cannot achieve a complete seal, 
and there will still be information leaks because the actual bid cannot exceed the total price. 
Second, the user experience is poor. The user needs to save the ciphertext and reveal the bid in the bid revealing period on time. 
Users need to do a lot of work, or it will lose the qualification and bid fund.

We believe that sealed bids are not completely sealed, and the auction experience is poor, so NNS hopes to find a better auction mechanism.

Transparent Bidding Mechanism
------------------------------

NNS will still use the bidding method to achieve the initial registration and distribution of domain names. 
However, unlike ENS, we adopt a transparent bidding mechanism. 
The advantage of this is that the user does not need to remember the ciphertext and there will be no bid revealing period. 
As long as the bidding is over, the final result can be known at once. 
However, there will be a problem with the transparent bidding. 
If the bidding period is certain, no one is willing to bid for the domain name at the beginning, 
because others can get a little more money to outbid you at the end of the auction. 
In order to solve the problem of the early bidding disadvantage, we introduce ``randomness`` for the end time of the auction. 
The auction is composed of two phases. 

**Fixed Period**

The first phase is a fixed period, for example, 3 days. 
All bids during this period are valid. 

**Random Period**

If someone bids on the last day of the fixed period, 
then here comes an additional two-day random period, otherwise the auction ends on the third day of the fixed period. 
During the random period, the end time of the auction is uncertain.

It is necessary to wait until the hash value of the futures blocks of two additional days is determined. 
According to the size of the interval, the latter someone bids, 
the more likely he or she is to fall out of the end time of the auction and thus the bidding will be invalid, 
so it’s better to bid as early as possible. 

**Ends of auction**

After a random period is finished, 
the bid ending block will be determined according to the hash of future blocks. 
The bid winner can be determined after all the bids from the bid opening and the ending block are collected.

..  image:: image/bidding.jpeg
   :height: 700px
   :width: 800px
   :scale: 100%
   :alt: NNS logo
   :align: center

The end result of this auction method is that if you think that nobody bids against you for a domain name, 
then you only need to bid within two days after opening the bid, 
then you can win a domain name after the fixed period is finished on the third day. 

If there is competition for a domain name, then the competition mainly occurs 
at both ends of the alternating period of the fixed period and the random period. 
By introducing a random end time of the auction, the late bidding is the less likely to fall within the valid period, 
avoiding the problem of the late bidding advantage caused by increasing transparency in the auction.

Rent mechanism
---------------

The registration mechanism only achieves the value discovery at the time of initial distribution. 
If the domain name is acquired by someone but never used, it is a waste of value. 
Later, we will analyze how to use the rent mechanism to promote the circulation in the secondary domain name market.

Technical Realization of Lock-free Cyclical Redistributed Token NNC
====================================================================

The NNS’s economic system needs an asset, so we designed an asset. 

The NNS's economic system requires that the total assets remain unchanged, and the auction proceeds and rental costs are considered as destroyed, 
so the assets we design can be consumed and the consumed assets will be redistributed, since destruction and redistribution will be cyclical, 
so we call it cyclically redistributed token. Lock-free refers to the redistribution process will not lock the users’ assets. 
The details of this will be explained below. 

Initial Distribution of Tokens
-----------------------------------

NNC will be initially distributed through an ICO mechanism.

Redistribution Mechanism
-------------------------

We use the destruction interface to destroy tokens. Tokens to be destroyed are:

1. Rent cost will be destroyed by the system

2. Revenue from second-level domain name auction will be destroyed by the system. 

3. If anyone wants to destroy part of his or her tokens, they will be destroyed by the system.

Once token is destroyed, they are counted into destruction pool. Assets in the destruction pool will go into the bonus pool, from which users could collect assets. 

Lock-free Bonus Collection 
--------------------------

Like an auction, this kind of system is usually composed of four stages: opening a bid, placing a bid, revealing a bid and winning a bid. 
Users’ tokens have to be paid into the system during the bidding, which means users’ assets are locked, 
consumed after winning the bid or unlocked if the bid is missed. 

However, NNC token is not composed of stages including participating bonus collection, waiting for the bonus and collecting the bonus,
which means users’ assets are not locked in the whole process. 
 
NNC uses the mechanism of the bonus pool queue, as shown in the above picture, only a fixed number of bonus pools (for example, five) are kept. 
The oldest bonus pool(the head pool)will be destroyed when more than five bonus pools are generated.

Besides the bonus pool, users’ assets are composed of two types: fixed assets and change. The holding time of fixed assets can only increase, and users whose holding time is earlier than collection time of a bonus pool are qualified to collect the bonus.

The holding time of fixed assets increases after collecting the bonus. It’s like coin hours is consumed, thus preventing the repeated collection of the same bonus. 

Details of The Bonus Pool
----------------------------

The token will maintain several bonus pools. When each bonus pool is generated, the assets in the destruction pool will all be transferred into this bonus pool. 
If the maximum bonus pool number is exceeded, the oldest bonus pool will also be destroyed and the remaining assets in the destroyed bonus pool will also be counted in the latest bonus pool. 

The number of bonus pools is fixed, for example, a bonus pool is generated for every 4096 blocks.
A maximum of five bonus pools are maintained. When the sixth bonus pool is generated, the first bonus pool will be destroyed,
and all of its assets are placed in the latest bonus pool. The above number of bonus pools and how often one bonus pool is produced are both tentative).
Each bonus pool will correspond to a block, this block is the bonus collection time, only those whose holding time is earlier than 
the bonus collection time can collect the bonus. 

Details of Fixed Assets and Exchange
-----------------------------------------------------

Fixed assets and change, of which fixed assets record a holding time.

Fixed assets and change only affect the amount of the award, the rest of the functions are not affected.

Fixed assets + change = user's balance of assets

Fixed assets do not have a fractional part, the decimal part is counted in the change. When “ considered as fixed assets” is mentioned below, it means the integer part is considered as fixed assets, and decimal part as change. 

Change will be firstly used in the transfer of tokens and fixed assets will be used only when the change is not enough. 

Transferrer: fixed assets can only be reduced. 

Transferee: fixed assets remain unchanged, the transferred value is counted in the change. 

Fixed assets can only be increased in two ways:

1. **Create an account**. 

 (a transfer to an address which has no NNC is regarded as creating an account)
 The transferred assets are regarded as fixed assets and the holding time is the new block ID. 

2. **Collect the bonus**. 

 After collecting the bonus, personal assets and the collected bonus will be considered as fixed assets, holding time is the bonus block.

		
When collecting the bonus, users can only collect bonus when their holding time is earlier than bonus pool time. 
Bonus collection ratio is calculated as the total amount in the current bonus pools/(the total issuance volume-the total amount in the current bonus pools)
		
Let’s take numbers to exemplify it. For example, there are 3 bonus pools: they were produced by block 4096，8000，10000. One user’s fixed assets is 100. His holding time is 7000, then he cannot collect the bonus in the first pool, but can collect the bonus from the second and third pools. The current block is 10500. Once the user collects the bonus, his assets holding time becomes 10500, so he cannot collect the bonus from any pools. 
		
For example, there is 50300000 NNC in a bonus pool. Then the user’s bonus collection ratio is 

::

    50300000 /（100000000-50300000）=1.23587223
    
This user’s fixed asset is 100, then he can collect 123.587223 NNC from the pool. 
If there is 500, 000 NNC in a bonus pool, then his collection ratio is 

::

    500000 /（100000000-500000）=0.00502512
    
as the user has 100 NNC of fixed asset, then he can collect 0.502512 NNC from the pool.  

NNC Interface(only additional interfaces compared with NEP 5 will be described)
--------------------------------------------------------------------------------

NNC first meets the NEP5 standard, and the NEP standard interface will not be described any more.

balanceOfDetail(byte[] address)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Returning the details of the user's assets, such as how much fixed assets, how much change, the total amount. 
Fixed assets holding block does not need a signature. Anybody can check it. 

Return structure:

::

    {
        Cash amount
        The amount of fixed assets 
        Fixed assets generation time( new block ID)
        Balance (fixed assets + cash)
    }

use(byte[] address,BigInteger value)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

The consumption of assets in an account requires the account signature.

Consumed assets go into bonus pools.

getBonus(byte[] address)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Account signature is required when designated accounts collect the bonus.

After the collection of the bonus, the total assets in this account will be considered as fixed assets and fixed assets holding block 
of this account will be changed. 

checkBonus()
~~~~~~~~~~~~~~

Checking current bonus pool doesn’t need a signature.

*Return* `Array<BonusInfo>`

::

    BonusInfo
    {
        StartBlock;//bonus collecting block
        BonusCount;//total amount of this bonus pool
        BonusValue;// remaining amount of this bonus pool. 
        LastIndex;// the id of last bonus
    }

newBonus ()
~~~~~~~~~~~~~~

Generating a new bonus pool can be called by anyone. But the bonus pool generation has to meet the bonus pool interval, so repeated calls is useless. 
This interface can be seen as a push to generate a new bonus pool.