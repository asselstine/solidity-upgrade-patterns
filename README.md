# Solidity Design Patterns

## Motivation

For the past few months I've immersed myself in Ethereum development.  I began with the mechanics of Ethereum: from how the chain is built to the structure of solidity code and helpful frameworks.

With that knowledge in place I wrote some simple Dapps using Web3.  Being a professional developer I got up and running pretty quickly.  However, after deploying even the most basic contracts I began to notice some serious deficiencies in how I was structuring my code.

I come from a traditional web background with some dabbling in iOS and React Native.  The application code was only ever accessible by my team, and if there was a bug or security hole we simply deployed new code.

The immutable environment of Ethereum is a completely different beast.  When a smart contract has been deployed it's behavior is preserved on the chain forever; the only change that can occur is within the stored state.

This means two things:

1. Smart contracts need thorough auditing before they are deployed.  An exploit in an Ethereum contract can lead to financial theft rather than more mundane (arguably) data theft in traditional web applications.
2. If there is a bug in smart contract code we *have* to be able to update it's behaviour.

Smart contract security is covered really well by articles all over the internet.  The exploits are famous (see the [DAO](http://hackingdistributed.com/2016/06/18/analysis-of-the-dao-exploit/)) and have been well publicized and analyzed.

Contract upgradeability is also [well defined](https://blog.indorse.io/ethereum-upgradeable-smart-contract-strategies-456350d0557c), but how the pattern fits into application architecture was less clear to me.  I decided to survey the landscape of Ethereum projects to see how they integrate the proxy abstraction.

## Introduction

In this article I examine several mature Ethereum projects to see how they approach contract storage and upgrades.  I'll be examining [Augur](http://www.augur.net/), [Colony](https://colony.io/), [Aragon](https://aragon.one/) and [Rocket Pool](https://www.rocketpool.net/).

Before continuing you should have a solid understanding of the [`DELEGATECALL`](http://solidity.readthedocs.io/en/develop/assembly.html) opcode.  Most of the projects use this opcode to implement upgradeable contracts.  For an example see Manuel Aráoz's [Dispatcher](https://github.com/maraoz/solidity-proxy/blob/master/contracts/Dispatcher.sol).  More examples can be found in the references section below.

## Augur

[Augur](http://www.augur.net/) is a prediction market that allows people to gamble on the outcomes of future events.  It's been under development since 2014 and has a strong team of developers behind it.  Augur is an extremely transparent organization and publishes it's smart contracts to Github under [augur-core](https://github.com/AugurProject/augur-core).

### Structure

Augur consists of 70+ smart contracts.  All of the contracts (with the exception of some edge cases) inherit 


All of these contracts are registered with a singleton [Controller](https://github.com/AugurProject/augur-core/blob/master/source/contracts/Controller.sol).  The controller stores a bytes32 to ContractDetails mapping.  Each ContractDetails struct contains the address of the deployed contract and a few bookkeeping fields.

```solidity
contract Controller is IController {
    struct ContractDetails {
        bytes32 name;
        address contractAddress;
        bytes20 commitHash;
        bytes32 bytecodeHash;
    }
    mapping(bytes32 => ContractDetails) public registry;

    function lookup(bytes32 _key) public view returns (address) {
        return registry[_key].contractAddress;
    }

    function registerContract(bytes32 _key, address _address, bytes20 _commitHash, bytes32 _bytecodeHash) public onlyOwnerCaller returns (bool) {
        registry[_key] = ContractDetails(_key, _address, _commitHash, _bytecodeHash);
        getAugur().logContractAddedToRegistry(_key, _address, _commitHash, _bytecodeHash);
        return true;
    }

    // yadda yadda yadda
}
```

This allows contracts to be looked up via a hashed name and replaced with new versions.

There is only ever one version of the controller; it is not upgradeable once it is deployed.  Deploying a new Controller contract would be considered a major upgrade to the system and handled using a different process.

Every contract in the system inherits from [Controlled](https://github.com/AugurProject/augur-core/blob/master/source/contracts/Controlled.sol).  Controlled contracts store a reference to the Controller, so that they are able to use `Controller#lookup` to retrieve other contracts in the system.

```solidity
contract Controlled is IControlled {
    IController internal controller;
}
```

Nearly every contract that stores data in the system is created using a Factory.  The Factories use a variant of the [Delegator](https://github.com/AugurProject/augur-core/blob/master/source/contracts/libraries/Delegator.sol) pattern to effectively "instantiate" new instances of contracts.  A new Delegator is constructed using the address of the controller and the name of the desired contract.

```solidity
contract DelegationTarget is Controlled {
  bytes32 public controllerLookupName;
}

contract Delegator is DelegationTarget {
    function Delegator(IController _controller, bytes32 _controllerLookupName) public {
        controller = _controller;
        controllerLookupName = _controllerLookupName;
    }

    function() external payable {
        address _target = controller.lookup(controllerLookupName);
        // yadda yadda yadda assembly { DELEGATECALL }
    }
}
```

The Delegator will effectively become an instance of the target contract because it implicitly shares the same storage shape and behaviour through the fallback function and the `DELEGATECALL` opcode.  Notice that the storage is separated into a `DelegationTarget` superclass; this is important because our target contracts need to inherit the Delegator fields as well.  By inheriting these fields the target contract's storage will be offset by the correct amount so that it doesn't trample the Delegator field storage.  Savvy?

It's not evident in the psuedocode above, but note that the assembly uses the [`returndatasize`](http://solidity.readthedocs.io/en/develop/assembly.html) opcode introduced in the [Byzantium](https://blog.ethereum.org/2017/10/12/byzantium-hf-announcement/) upgrade.

Here is an example contract:

```solidity
contract IExampleValueObject {
  function get() external view returns (uint);
}

contract ExampleValueObject is DelegationTarget {
  uint private value;

  function initialize() external {
    value = uint(42);
  }

  function get() external view returns (uint) {
    return value;
  }
}

// Assume that ExampleValueObject has been registered with the Controller

contract ExampleFactory is Controlled {
  function createExample() external returns (IExampleValueObject) {
    Delegator d = new Delegator(controller, 'ExampleValueObject');
    IExampleValueObject exampleValueObject = IExampleValueObject(d);
    exampleValueObject.initialize();
    return exampleValueObject;
  }
}
```

You'll notice above that the new contract instance is of the type Delegator.  When the Delegator delegates the `initialize()` call to ExampleValueObject and the code writes to the `value` field we want it to write into the first storage location after the `controller` and `controllerLookupName` fields.  That's why ExampleValueObject inherits from DelegationTarget: to shift the storage read/write positions.

If the ExampleValueObject contract is updated with new fields, they *must* be defined *after* the existing fields so that fields aren't trampled in instances of Delegator that have been bound to that type.  The constraints are very well defined in this [gist](https://gist.github.com/Arachnid/4ca9da48d51e23e5cfe0f0e14dd6318f) by Nick Johnson.

### Summary

Augur uses a combination of a contract registry and delegate to allow for partial upgrading of their system.  It's interesting to note that they plan on locking down the registry by disabling a '[dev-mode](https://github.com/AugurProject/augur-core/blob/master/source/contracts/Controller.sol#L6)' in the Controller.  At some point they will freeze the contracts in production and lock themselves out of the Controller.  Afterwards, if they want to deploy an upgrade it will result in an entirely separate set of contracts; in a sense forking the application.

## Colony

[Colony](https://github.com/JoinColony/colonyNetwork)

- Colony, like Aragon, has a 'ColonyStorage' superclass that stores the Contract fields.  The Colony object subclasses it.

- Colony has a [Resolver](https://github.com/JoinColony/colonyNetwork/blob/develop/contracts/Resolver.sol) contract that appears to manage

Colony outlines their use of the [EtherRouter](https://github.com/ownage-ltd/ether-router) pattern in the Token Upgradeability section of their article on the [Colony Token Sale contract](https://blog.colony.io/colony-sale-contract-and-the-clny-token-3da67d833087)

##### Deployment

1. Core contracts are deployed:
  - ColonyNetwork
  - ColonyNetworkStaking
  - EtherRouter
  - Resolver
2. The ColonyNetwork and ColonyNetworkStaking contracts are registered in the Resolver
3. The EtherRouter is set to point to the Resolver
4. The Colony contracts are deployed to the network
  - Colony
  - ColonyTask
  - ColonyFunding
  - ColonyTransactionReviewer
5. The contracts from step 4 are bound to a Resolver which is then added as the first Colony version

##### Storage

The EtherRouter serves as the `DELEGATECALL` mechanism.  It's fallback function will delegate to addresses registered in it's associated `Resolver` instance.  The `Resolver` is a function registry in that you register a function signature and point it to a contract address.  This means that all contracts registered in a Resolver will share the same storage address and shape, and must not have any conflicting function signatures.

Note that Colony, ColonyTask, ColonyFunding and ColonyTransactionReviewer all inherit from ColonyStorage.  ColonyNetwork and ColonyNetworkStaking both inherit from ColonyNetworkStorage.  

To create a new Colony the `ColonyNetwork#createColony(...)` function is called.  The `ColonyNetworkStorage` contains the field `mapping (bytes32 => address) _colonies` which maps the EtherRouter instances for each Colony.  On creation, the `EtherRouter` instance is bound to the latest version of the Colony `Resolver` (which has the latest versions of the Colony, ColonyTask, ColonyFunding and ColonyTransactionReviewer contracts registered).

##### Upgrades

To upgrade the `ColonyNetwork` or `ColonyNetworkStaking` contracts the user would need to register the new contracts with the global Resolver.  The global EtherRouter would immediately refer to the new code and the behaviour of the system would change.

To upgrade the Colony, ColonyTask, ColonyFunding or ColonyTransactionReviewer contracts the user would need to: deploy them, register them with a new Resolver, and then add the Resolver as the newest version to the ColonyNetwork.

Once the newest Resolver is deployed, each Colony would need to be upgraded using the  `ColonyNetwork#upgradeColony(...)` function which sets the EtherRouter instance's resolver to point to the latest `Resolver` version.  In this way Colonies can be upgraded separately.

#### Summary

Colony uses the pre-Byzantium style [EtherRouter](https://github.com/ownage-ltd/ether-router) pattern to upgrade contracts.  A global registry is maintained for ColonyNetwork and ColonyNetworkStaking contracts and a set of versioned registries are maintained within the ColonyNetwork so that Colonies can be independently versioned.

## Rocketpool

[Rocketpool](https://github.com/rocket-pool/rocketpool) is an application that provides staking pools for the upcoming Casper proof-of-stake protocol for Ethereum.  It's not an old codebase, but it showcases a different style of storage that I think is worth highlighting.







## Aragon

Aragon is a very interesting project and an exception on this list as it's an app framework.  Aragon itself has been recently modularized into Aragon OS apps.  You can see their announcement [here](https://blog.aragon.one/introducing-aragonos-3-0-alpha-the-new-operating-system-for-protocols-and-dapps-348f7ac92cff).



## References

### DELEGATECALL Patterns

The Byzantium fork introduced the `returndatasize` opcode that allowed for the Delegate pattern to be generic.

- Byzantium-style [Dispatcher](https://github.com/maraoz/solidity-proxy/blob/master/contracts/Dispatcher.sol)
- Pre-Byzantium [Dispatcher](https://gist.github.com/Arachnid/4ca9da48d51e23e5cfe0f0e14dd6318f)
 [Proxy](http://martin.swende.se/blog/EVM-Assembly-trick.html)
- Pre-Byzantium [EtherRouter](https://github.com/ownage-ltd/ether-router)
