# Contract Storage and Upgrades in Production Ethereum Dapps

## Motivation

For the past few months I've immersed myself in Ethereum development.  I began with the mechanics of Ethereum: from how the chain is built to the structure of solidity code and helpful frameworks.

With that knowledge in place I wrote some simple Dapps using Web3.  Being a professional developer I got up and running pretty quickly.  However, after deploying even the most basic contracts I began to notice some serious deficiencies in how I was structuring my code.

I come from a traditional web background with some dabbling in iOS and React Native.  The application code was only ever accessible by my team, and if there was a bug or security hole we simply deployed new code.

The immutable environment of Ethereum is a completely different beast.  When a smart contract has been deployed it's behavior is preserved on the chain forever; the only change that can occur is within the stored state.

This means two things:

1. Smart contracts need thorough auditing before they are deployed.  An exploit in an Ethereum contract can lead to financial theft rather than more mundane (arguably) data theft in traditional web applications.
2. If there is a bug in smart contract code we *have* to be able to update it's behaviour.

Smart contract security is covered really well by articles all over the internet.  The exploits are famous (see the [DAO](http://hackingdistributed.com/2016/06/18/analysis-of-the-dao-exploit/)) and have been well publicized and analyzed.

Contract upgradeability is also [well defined](https://blog.indorse.io/ethereum-upgradeable-smart-contract-strategies-456350d0557c), but how the pattern fits into application architecture was less clear to me.  I decided to survey the landscape of Ethereum projects.

## Introduction

In this article I examine several mature Ethereum projects to see how they approach contract storage and upgrades.  I've tried to stick strictly to analysis and have avoided giving opinions or making assumptions.  I'll be examining [Augur](http://www.augur.net/), [Colony](https://colony.io/), [Aragon](https://aragon.one/) and [Rocket Pool](https://www.rocketpool.net/).  I picked these projects because they are relatively complete, have public codebases, and showcase something unique.

Before continuing you should have a solid understanding of the [`DELEGATECALL`](http://solidity.readthedocs.io/en/develop/assembly.html) opcode.  Most of the projects use this opcode to implement upgradeable contracts.  For a post-Byzantium example of opcode usage see Manuel Aráoz's [Dispatcher](https://github.com/maraoz/solidity-proxy/blob/master/contracts/Dispatcher.sol). For a pre-Byzantium example see [EtherRouter](https://github.com/ownage-ltd/ether-router). More examples can be found in the references section below.

## Augur

[Augur](http://www.augur.net/) is a prediction market that allows people to gamble on the outcomes of future events.  It's been under development since 2014 and has a strong team of developers behind it.  Augur publishes it's smart contracts to Github under [augur-core](https://github.com/AugurProject/augur-core).

Augur consists of 70+ smart contracts.  Nearly all of the contracts inherit from [Controlled](https://github.com/AugurProject/augur-core/blob/7f3c79a5dd471a98df8f66a640902e063f15f796/source/contracts/Controlled.sol).  Controlled contracts store a reference to a [Controller](https://github.com/AugurProject/augur-core/blob/7f3c79a5dd471a98df8f66a640902e063f15f796/source/contracts/Controller.sol).

```solidity
contract Controlled is IControlled {
    IController internal controller;
}
```

The Controller acts as a contract registry.  It provides a [`lookup(bytes32)`](https://github.com/AugurProject/augur-core/blob/7f3c79a5dd471a98df8f66a640902e063f15f796/source/contracts/Controller.sol#L97) method so that contracts can be retrieved by name.

```solidity
// Note that the code has been simplified

contract Controller {
    struct ContractDetails {
        bytes32 name;
        address contractAddress;
    }
    mapping(bytes32 => ContractDetails) public registry;

    function lookup(bytes32 _key) public view returns (address) {
        return registry[_key].contractAddress;
    }

    function registerContract(bytes32 _key, address _address) {
        registry[_key] = ContractDetails(_key, _address);
    }
}
```

This allows any contract to look up another using a hashed name.  It also means that contracts can be replaced at runtime by re-registering them.

## Deployment

Augur uses a set of custom deployment scripts written in TypeScript to deploy their contracts.  When the application is first deployed:

1. The Controller is deployed.
2. The Augur contract is deployed and updated with the Controller address.
3. The remaining contracts are each deployed, updated with the Controller address, and then registered with the Controller.

In this way we can see that the system is partially upgradeable.  The Controller and Augur contracts are static, but the rest of the contracts can be upgraded by re-registering them with the controller.

## Storage

Augur contracts contain both state variables and behaviour.  They have not been separated. However, in a running application, the state is actually stored in Delegator instances.  The Delegators will delegate to a registered contract and therefore adopt the same storage shape and behaviour.

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

### Factories

To create these Delegator instances nearly every contract type has a corresponding factory.

For example:

```solidity
contract IExampleValueObject {
  function initialize() external;
  function get() external view returns (uint);
}

contract ExampleValueObject is DelegationTarget, IExampleValueObject {
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

Contracts will use the Controller to lookup the contract factory, then call the factory's create method to create a new instance of a contract.

You may have noticed that the ExampleValueObject inherits from DelegationTarget somewhat needlessly; this is so that the ExampleValueObject storage will be correctly offset by the amount of storage required by the Delegator, thereby preventing any memory from being trampled.  The memory constraints are nicely detailed in this [gist](https://gist.github.com/Arachnid/4ca9da48d51e23e5cfe0f0e14dd6318f) by Nick Johnson.

### Singletons

Some contracts exist globally as singletons and don't need a factory. These singletons are instead registered twice in the controller: the first being an instance of the contract whose name is suffixed with 'Target' and the second being a Delegator registered under the original name and pointing to the suffixed name.  In this way the original contract with the suffixed name can be swapped out while the Delegator remains the same.  Savvy?

### Upgrades

Augur is partially upgradeable.  The majority of contracts can be swapped out at runtime by re-registering them in the Controller, but some core contracts of the system such as Controller and Augur cannot be swapped out: changing these contracts would necessitate an entirely new app deployment.  In fact there is code that halts the entire system and allows users to withdraw their funds.

It's interesting to note that they plan on locking down the registry by disabling a '[dev-mode](https://github.com/AugurProject/augur-core/blob/7f3c79a5dd471a98df8f66a640902e063f15f796/source/contracts/Controller.sol#L6)' in the Controller.  At some point they will freeze the contracts in production and lock themselves out of the Controller.  Afterwards, if they want to upgrade, they will need to deploy an entirely new version of the application.

## Colony

[Colony](https://colony.io/) is a platform for creating decentralized organizations.  The code has recently been made public on Github under the [colonyNetwork](https://github.com/JoinColony/colonyNetwork) project.   The project history began in early 2016.

Colony consists of about 14 smart contracts.  Conceptually, these contracts can be divided into those that concern the Colony Network as a whole, and those that concern an individual Colony.  Concretely, the two groups are delineated by their inheritance hierarchy; all state variables are stored in either the [ColonyNetworkStorage](https://github.com/JoinColony/colonyNetwork/blob/82764b58a52c19326957316e46328662e3e80de7/contracts/ColonyNetworkStorage.sol) or the [ColonyStorage](https://github.com/JoinColony/colonyNetwork/blob/82764b58a52c19326957316e46328662e3e80de7/contracts/ColonyStorage.sol) contracts.  All contracts inherit from one of these.

An individual Colony is defined by the contracts [Colony](https://github.com/JoinColony/colonyNetwork/blob/82764b58a52c19326957316e46328662e3e80de7/contracts/Colony.sol), [ColonyTask](https://github.com/JoinColony/colonyNetwork/blob/82764b58a52c19326957316e46328662e3e80de7/contracts/ColonyTask.sol), [ColonyFunding](https://github.com/JoinColony/colonyNetwork/blob/82764b58a52c19326957316e46328662e3e80de7/contracts/ColonyFunding.sol), and [ColonyTransactionReviewer](https://github.com/JoinColony/colonyNetwork/blob/82764b58a52c19326957316e46328662e3e80de7/contracts/ColonyTransactionReviewer.sol).  These contracts all inherit from ColonyStorage.

The Colony Network is defined by two contracts: [ColonyNetwork](https://github.com/JoinColony/colonyNetwork/blob/82764b58a52c19326957316e46328662e3e80de7/contracts/ColonyNetwork.sol) and [ColonyNetworkStaking](https://github.com/JoinColony/colonyNetwork/blob/82764b58a52c19326957316e46328662e3e80de7/contracts/ColonyNetworkStaking.sol).  These contracts inherit from ColonyNetworkStorage.

The contracts within each group are registered with their respective [Resolver](https://github.com/JoinColony/colonyNetwork/blob/develop/contracts/Resolver.sol) instances. The Resolver acts as a function registry; contract addresses are looked up by their function signatures.

```solidity
// Simplified for brevity

contract Resolver is DSAuth {
  struct Pointer { address destination; uint outsize; }
  mapping (bytes4 => Pointer) public pointers;

  function register(string signature, address destination, uint outsize) public auth {
    pointers[stringToSig(signature)] = Pointer(destination, outsize);
  }

  function lookup(bytes4 sig) public view returns(address, uint) {
    return (destination(sig), outsize(sig));
  }
}
```

This means that function signatures [must be unique](https://github.com/JoinColony/colonyNetwork/blob/82764b58a52c19326957316e46328662e3e80de7/helpers/upgradable-contracts.js#L12) across any contracts that are registered with the Resolver.  This also means that the registered contracts must share the same storage shape, because [EtherRouter](https://github.com/JoinColony/colonyNetwork/blob/82764b58a52c19326957316e46328662e3e80de7/contracts/EtherRouter.sol) instances that point to this Resolver will take on the behaviour of all the registered contract functions.

A Resolver that contains a set of Colony contracts is registered and versioned with the ColonyNetwork contract using the [`addColonyVersion()`](https://github.com/JoinColony/colonyNetwork/blob/82764b58a52c19326957316e46328662e3e80de7/contracts/ColonyNetwork.sol#L153) method.

```solidity
contract ColonyNetwork is ColonyNetworkStorage {
  function addColonyVersion(uint _version, address _resolver) public auth {
    colonyVersionResolver[_version] = _resolver;
    if (_version > latestColonyVersion) {
      latestColonyVersion = _version;
    }
  }
}
```

New Colonies are created through the ColonyNetwork contract using the [createColony()](https://github.com/JoinColony/colonyNetwork/blob/82764b58a52c19326957316e46328662e3e80de7/contracts/ColonyNetwork.sol#L111) method.  This method will create a new EtherRouter instance and register it under the desired name.  The EtherRouter is then bound to the most recent Colony Resolver version.

```solidity
// Simplified for brevity

contract ColonyNetwork is ColonyNetworkStorage {
  function createColony(bytes32 _name) {
    var etherRouter = new EtherRouter();
    etherRouter.setResolver(colonyVersionResolver[latestColonyVersion]);
    _colonies[_name] = colony;
  }
}
```

### Deployment

Colony uses Truffle for contract deployment.  The steps are as follows:

1. The Colony Network contracts are deployed:
  - ColonyNetwork
  - ColonyNetworkStaking
  - EtherRouter
  - Resolver
2. The ColonyNetwork and ColonyNetworkStaking contracts are registered with the Resolver
3. The EtherRouter is set to point to the Resolver
4. The Colony contracts are deployed to the network
  - Colony
  - ColonyTask
  - ColonyFunding
  - ColonyTransactionReviewer
5. A new Resolver is created and each of the contracts from step 4 are registered.
6. The new Resolver is added to the Colony Network as the first Colony version.

### Storage

The first EtherRouter that is deployed becomes the main point of entry for the platform.  It acts as the storage and will take on the storage shape and behaviour of the contracts in the Colony Network Resolver (it's shape will be the same as ColonyNetworkStorage).

As mentioned previously, when the Colony Network creates a Colony it will create a new instance of an EtherRouter that points to the latest version of the Colony Resolver.  In this way the Colony EtherRouter will take on the storage shape of the latest Resolver version (it's shape will be the same as ColonyStorage).

##### Upgrades

Colony is almost entirely upgradeable; the only exceptions are the EtherRouter and the Resolver.

To upgrade the Colony Network contracts the user would need to deploy and register the new contracts with the global Resolver.  The global EtherRouter would immediately refer to the new code and the behaviour of the Colony Network would change.

To upgrade the Colony, ColonyTask, ColonyFunding or ColonyTransactionReviewer contracts the user would need to deploy and register the contract with a new Resolver, then add that resolver as a new version in the Colony Network.  They can then use the [`ColonyNetwork#upgradeColony(...)`](https://github.com/JoinColony/colonyNetwork/blob/82764b58a52c19326957316e46328662e3e80de7/contracts/ColonyNetwork.sol#L171) to upgrade their Colony to the new version.

#### Summary

Colony uses the pre-Byzantium style [EtherRouter](https://github.com/ownage-ltd/ether-router) pattern to upgrade contracts.  A global Resolver is maintained for ColonyNetwork and ColonyNetworkStaking contracts and a versioned registry of Resolvers is maintained within the ColonyNetwork so that Colonies can be independently versioned and upgraded.

## Rocketpool

[Rocketpool](https://www.rocketpool.net/) is an application that provides staking pools for the upcoming Casper proof-of-stake protocol for Ethereum.  It's not an old codebase, but it showcases a very different approach that I think is worth discussing.  The project began around the end of 2016.  The smart contract code is published on Github under [rocketpool](https://github.com/rocket-pool/rocketpool).

Rocketpool consists of about 16 contracts.  This is the only project I reviewed that does not use the DELEGATECALL opcode; instead opting for a design that [separates storage and behaviour](https://medium.com/rocket-pool/upgradable-solidity-contract-design-54789205276d).

All of the contracts, with the exception of [RocketStorage](https://github.com/rocket-pool/rocketpool/blob/3e66165880346f0c4e95c7eea345b5cecdf3defc/contracts/RocketStorage.sol) inherit from the [RocketBase](https://github.com/rocket-pool/rocketpool/blob/3e66165880346f0c4e95c7eea345b5cecdf3defc/contracts/RocketBase.sol).  The RocketBase contract stores the address of the RocketStorage contract:

```solidity
contract RocketBase {
    RocketStorageInterface rocketStorage = RocketStorageInterface(0);

    function RocketBase(address _rocketStorageAddress) public {
        rocketStorage = RocketStorageInterface(_rocketStorageAddress);
    }
}
```

The RocketStorage contract serves as a generic data store holding registries of primitives such as addresses, uint256, bool etc.

```solidity
contract RocketStorage {
    mapping(bytes32 => uint256)    private uIntStorage;
    mapping(bytes32 => string)     private stringStorage;
    mapping(bytes32 => address)    private addressStorage;
    mapping(bytes32 => bytes)      private bytesStorage;
    mapping(bytes32 => bool)       private boolStorage;
    mapping(bytes32 => int256)     private intStorage;
}
```





## Aragon

Aragon is a very interesting project and an exception on this list as it's an app framework.  Aragon itself has been recently modularized into Aragon OS apps.  You can see their announcement [here](https://blog.aragon.one/introducing-aragonos-3-0-alpha-the-new-operating-system-for-protocols-and-dapps-348f7ac92cff).



## References

### DELEGATECALL Patterns

The Byzantium fork introduced the `returndatasize` opcode that allowed for the Delegate pattern to be generic.

- Byzantium-style [Dispatcher](https://github.com/maraoz/solidity-proxy/blob/master/contracts/Dispatcher.sol)
- Pre-Byzantium [Dispatcher](https://gist.github.com/Arachnid/4ca9da48d51e23e5cfe0f0e14dd6318f)
 [Proxy](http://martin.swende.se/blog/EVM-Assembly-trick.html)
- Pre-Byzantium [EtherRouter](https://github.com/ownage-ltd/ether-router)
