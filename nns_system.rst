****************************
NNS System Design Overview
****************************

NNS System Functions
======================

NNS system has two functions: first, to resolve a human-readable name like beautiful.neo 
into machine-readable identifiers like NEO’s address; second, to provide descriptive data for domain names, 
such as whois, contract interface description and so forth. 

NNS has similar goals to DNS. But based on blockchain architecture design, 
NNS is decentralized and serves blockchain network. NNS operates on a system of dot-separated hierarchical names called domains, 
with the owner of a domain having full control over the distribution of subdomains.

Top-level domains, like ‘.neo’ and ‘.gas’, are owned by smart contracts called registrars. 
One registrar manages one root domain name and specifies rules governing the allocation of their subdomains. 
Anyone may, by following the rules imposed by these registrar contracts, obtain ownership of a second-level domain for their own use.

NNS Architecture
=================

NNS has four components: 
1. Top-level domain name contract.( domain name root is the script that manages root domain name. )

2. The owner (the owner could be a personal account’s address or a smart contract)

3. Registrar (A registrar is simply a smart contract that issues subdomains of that domain to users. The root domain name specifies a root domain name’s registry.)

4. Resolver (responsible for resolving a domain name or its subdomain names. )

Top-level Domain Name Contract
-------------------------------

Domain name root is the manager of all the information of a root domain name such as .test. 
No matter it is a second-level domain name like aa.test or third-level domain name like bbb.aa.test, 
both of their owners are kept in the domain name root which keeps the following data in the form of a dictionary. 

1. The owner of the domain name

2. The registrar of the domain name

3. The resolver of the domain name

4. The TTL(time to live) of the domain name


Owner
-------

The owner of the domain name could be either an account address or a smart contract. 
(ENS’s design is a smart contract that owns a domain name is called registrar. Actually, registrar is the owner’s exception.
We separate the owner of the domain name from the registrar, making the system clearer.)
Owners of domain names may: 

1. Transfer the ownership of the domain name to another address. 

2. Change registrar. The most common domain registrar is "administrators allocate subdomains manually."

3. Change the resolver. 

A smart contract is allowed to be the owner, which provides a variety of ownership models.

1. The domain name co-owned by two persons. The transfer of domain names or changing registrars is only possible with two people’s signatures.

2. The domain co-owned by multi-person. The transfer of domain names or changing registrars is only possible with more than 50% of people’s signatures.

If the owner of the domain name is an account address, the user could invoke registrar’s interface to manage second-level domain names.

Registrar
----------

(ENS’s design is a smart contract which owns top-level domains is called registrar. Actually, the registrar is the owner’s special case. 
Our separation of the owner of the domain name from the registrar makes the system clearer. Most users don’t sell their second-level domain names, 
so most users just need to configure a resolver rather than a registrar.)

A registrar is responsible for specifying subdomain names of a domain name to other owners. 
The registrar operates by invoking the domain root’s script. The domain name root checks whether the registrar has the authority to operate this domain name. 
The registrar has two functions:

1. To re-specify subdomain names of a domain name to other owners. 

2. To check whether the owner of a subdomain name is legal or not because second-level domain names could be transferred to others after third-level domain names are sold.

In the complete resolution, the registrar will be asked whether its subdomain names are allocated to specified owners. If not, the resolution is invalid. 
A registrar is a smart contract. There could be many types of registrars. 

1. "First come, first served" registrar. Users are free to grab domain names. Our testnet’s .test domain names will be registered in a "first come, first served" way. 

2. Administers allocate registrars manually. An administrator sets up how the ownership of subdomain names is processed. Individuals with second-level domains allocate subdomain names manually. 

3. Registrar auction. Testnet’s .neo domains and mainnet’s .neo domain names will be registered in the way of auction with collaterals.  

Resolver
-------------

NNS’s main function is to finish the mapping from the domain name to the resolver. The resolver is a smart contract which interprets names into addresses. 

Any smart contracts which follow NNS resolver rules could be configured to resolvers. NNS will provide general-purpose resolvers. 
If new protocol types need to be added, the direct configuration can be done without changing NNS system if current NNS rules are not disrupted.  

Resolution Rules
------------------

Domain Name Storage
~~~~~~~~~~~~~~~~~~~~~

The domain name NNS stores are 32-byte hashes, rather than the plain text of the domain name. Here are reasons for this design.

1. A unified process allows any length of the domain name.

2. It preserves the privacy of the domain name to some extent. 

3. The algorithm for converting a domain name to a hash is called NameHash, and we'll explain it in other documentation. 

The definition of NameHash is recursive.

1. for example aaa.neo corresponding

::

    hashA  =  hash256(hash256(“.neo”) + “aaa”)

2. then bbb.aaa.neo corresponding

::
    
    hashB  =  hash256(hashA+”bbb”)	

3. then  ccc.bbb.aaa.neo corresponding 

::
    
    HashC  =  hash256(hashB+”ccc”)

This definition allows us to store all levels of domain names, level 1, level 2, to countless levels, in a data structure: Map <hash256, parser> in a flat way. 
This is exactly how the registrar saves the resolution of domain names. 

This recursive calculation of NameHash can be expressed as a function: 

::

    Hash = NameHash ("xxx.xxx.xxx ..."); 
    
for the realization of NameHash, please refer to :ref:`namehash`.

Resolution Process
~~~~~~~~~~~~~~~~~~~

The user invokes the resolution function of the root domain name for resolution, and the root domain name provides both complete and quick resolution. 
You can invoke it as need. You can also query the resolver and invoke it by yourself.

**Quick resolution**

Quick resolution root domain name directly searches the resolver of a complete domain name. if not, search the parent domain name’s resolver and then invokes the resolver for resolution. 
There are fewer operations for a quick resolution, but there's a flaw: the third-level domain name is sold to someone else and the resolver exists, but the second-level domain name has been transferred. At this point, the domain name can still be resolved.

**Complete resolution**

In a complete manner, the root of the domain name will start with the root domain name and queries ownership and TTL layer by layer. It will fail if they don’t comply with.

More operations are needed in the complete resolution and operations has a linear growth with the layer number of domain names.

NNS Economic Model
======================

Two kinds of tokens were introduced in the economic model of NNS. One is NNC, which is a UTXO asset and has a total supply of 1 billion. The other is SGS, which is a NEP5 token. It’s bound with NEO’s GAS at the ratio of 1:1 and they can be converted with each other.

The NNS root domain name is initiated by the NNC holder's voting. There are two modes of voting. 

One is that the administrator initiates the root domain name voting, and the domain name is activated when less than 30% of the votes are against it in 3 days. 

The other one is that any NNC holder starts the root domain name voting. If more than 50% of the votes are in favor of it within 3 days, the root domain name will be activated. Either way, voters or non-voters will be participants in the game. 
The GAS income from domain name registration will be redistributed to NNC holders. 


NNC: an equity proof token
-----------------------------

NNC is an equity proof token introduced into the NNS system. In order to sustain the system, NNS has introduced a fee redistribution system. The fees charged for all domain name auctions will be completely redistributed to NNC's holders.

The initial issuance of the NNC is in the form of airdrop(s). NNC will be only airdropped to NEO holders. Specific airdrop rules will be announced in the future.

SGAS-a kind of gas token
----------------------------


In order to facilitate the use of GAS in application contracts, NNS has issued a NEP5-based token with a total supply of 100 million, which is  bound with NEO's GAS at the ratio of 1:1 and they can be converted with each other freely..

The GAS used for the redemption of SGAS will be stored in the account of the SGAS contract. NNS will not transfer or use this GAS. Therefore, it is guaranteed that as long as the user holds the SGAS, it can be converted to the same amount of GAS.

In the NNS system, SGAS mainly has the following functions:


It can be converted with GAS and vice-versa
- Recharge/withdraw from the registrar.
- Participate in domain auctions. 
- Auction fee payment


In addition to being used within the NNS system, since the SGAS itself is a NEP5 token system that is deployed on the Mainnet, all contract applications can also use this SGAS contract to perform convenient intra-contract GAS operations.

Bonus pools
---------------

When a user bids for a domain name, NNS will generate income from SGAS. There are two main sources:

1. The bid winner. If the user wins the bid and obtains the domain name ownership, then the bid winner will be charged all the bid fund as the fee.

2. The bid loser. For users who participate in the auction, but lose the bid, 5% of the bid is charged as a fee.

All fee income will be transferred to bonus pools. In the bonus pools, all NNC holders can receive SGAS in proportion to their NNC holdings.

Domain Name Browser
=====================

NNS domain name browser is the entrance which provides NNS domain name query, auction, transfer and other functions.

Reverse Resolution
==================
NNS will support reverse resolution which will become an effective way to verify addresses and smart contracts. 

Roadmap
==========

**First quarter, 2018**

- January 2018, officially released NNS technical white paper
- January 2018, completed the technical principle test and verification
- January 31st, 2018, release the NNS Phase 1 testing service, including registrar and resolver, on the test net, anyone can register unregistered and rules-compliant domain names.
- February 2018,  launch testnet-based Domain Name Browser V1

**Second quarter, 2018**

- March 2018, issue NNC on testnet. 
- March 2018, release NNS Stage 2 testing service including bidding service on testnet when anyone can apply to NEL for NDS bidding test domain name
- April 2018, launch testnet-based domain name browser V2.
- May 2018, issue NNC on mainnet. 
- June 2018, release NNS service on mainnet. Here comes Neo domain name era. 
- June 2018, release mainnet-based domain name browser. 