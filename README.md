# Contract Storage and Upgrading in Production Dapps

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

The Controller is not upgradeable.  Deploying a new Controller contract would be considered a major upgrade to the system and would be handled using a different process.

## Deployment

Augur uses a set of custom deployment scripts written in TypeScript to deploy their contracts.  The contracts are deployed as such:

1. The Controller is deployed.
2. The Augur contract is deployed and the Controller state variable is set.
3. The remaining contracts are each deployed, the Controller state variable set, then registered with the Controller.

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

You may have noticed that the ExampleValueObject inherits from DelegationTarget somewhat needlessly; this is so that the ExampleValueObject storage will be correctly offset by the amount of storage required by the Delegator, thereby preventing any memory from being trampled.  The memory constraints are well defined in this [gist](https://gist.github.com/Arachnid/4ca9da48d51e23e5cfe0f0e14dd6318f) by Nick Johnson.

Not all contracts that require storage are created by a factory.  Some contracts are singletons and are instead registered twice in the controller: the first being an instance of the contract whose name is suffixed with 'Target' and the second being a Delegator registered under the original name and pointing to the suffixed name.

### Upgrades

It's interesting to note that they plan on locking down the registry by disabling a '[dev-mode](https://github.com/AugurProject/augur-core/blob/7f3c79a5dd471a98df8f66a640902e063f15f796/source/contracts/Controller.sol#L6)' in the Controller.  At some point they will freeze the contracts in production and lock themselves out of the Controller.  Afterwards, if they want to upgrade, they will need to deploy an entirely new version of the application.

## Colony

[Colony](https://github.com/JoinColony/colonyNetwork)

- Colony, like Aragon, has a 'ColonyStorage' superclass that stores the Contract state variables.  The Colony object subclasses it.

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

To create a new Colony the `ColonyNetwork#createColony(...)` function is called.  The `ColonyNetworkStorage` contains the state variable `mapping (bytes32 => address) _colonies` which maps the EtherRouter instances for each Colony.  On creation, the `EtherRouter` instance is bound to the latest version of the Colony `Resolver` (which has the latest versions of the Colony, ColonyTask, ColonyFunding and ColonyTransactionReviewer contracts registered).

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
