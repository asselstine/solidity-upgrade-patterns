# Solidity Design Patterns

## Motivation

For the past few months I've immersed myself in Ethereum development.  I began with the mechanics of Ethereum: from how the chain is built to the structure of solidity code and helpful frameworks.

With that knowledge in place I began to write some simple Dapps using Web3.  Being a professional developer I got up and running pretty quickly.  However, after deploying even the most basic contracts I began to notice some serious deficiencies in how I was structuring my code.

I come from a traditional web background with some dabbling in iOS and React Native.  The application code was only ever accessible by my team, and if there was a security hole or bug we simply deployed new code.

The immutable environment of Ethereum is a completely different beast.  When a smart contract has been deployed it's behavior is preserved on the chain forever; the only change that can occur is within the stored state.

This means two things:

1. Smart contracts need thorough auditing before they are deployed.  An exploit in an Ethereum contract can lead to financial theft, rather than more mundane (arguably) data theft in traditional web applications.
2. If there is a bug in smart contract code we *have* to be able to update it's behaviour.

Smart contract security is covered really well by articles all over the internet.  The exploits are famous (see the [DAO](http://hackingdistributed.com/2016/06/18/analysis-of-the-dao-exploit/)) and have been well publicized; leading to thorough analysis.

Contract upgradeability is also [well defined](https://blog.zeppelin.solutions/proxy-libraries-in-solidity-79fbe4b970fd), but how the pattern fits into application architecture was less clear to me.  I decided to survey the landscape of Ethereum projects to see how they use this pattern.

## Introduction

In this article I examine several mature Ethereum projects to determine how they approach contract storage and upgradeability.  My goal is to contrast and compare them to discover any patterns in their design.  I'll be examining [Augur](http://www.augur.net/), [Aragon](https://aragon.one/), [Colony](https://colony.io/), [uPort](https://www.uport.me/), and [Rocket Pool](https://www.rocketpool.net/).

### DELEGATECALL Prerequisite

Every project implements upgradeable contracts using the DELEGATECALL opcode. They're usually called either [Dispatcher](https://gist.github.com/Arachnid/4ca9da48d51e23e5cfe0f0e14dd6318f) [2](https://github.com/maraoz/solidity-proxy/blob/master/contracts/Dispatcher.sol), [Proxy](http://martin.swende.se/blog/EVM-Assembly-trick.html) or [Delegate]().  You should be familiar with DELEGATECALL before continuing.

## Augur

Augur manages all of it's contracts using a central [Controller](https://github.com/AugurProject/augur-core/blob/master/source/contracts/Controller.sol).  The controller stores a bytes32 to ContractDetails mapping.  Each ContractDetails struct contains the address of the deployed contract and a few bookkeeping fields.

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

It's not evident in the psuedocode above, but note that the assembly uses the `[returndatasize](http://solidity.readthedocs.io/en/develop/assembly.html)` opcode introduced in the [Byzantium](https://blog.ethereum.org/2017/10/12/byzantium-hf-announcement/) upgrade.

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

Augur uses a combination of a contract registry and delegate to allow for partial upgrading of their system.  It's interesting to note that they plan on locking down the registry by disabling a '[dev-mode](https://github.com/AugurProject/augur-core/blob/master/source/contracts/Controller.sol#L8)' in the Controller.  At some point they will freeze the contracts in production and lock themselves out of the Controller.  Afterwards, if they want to deploy an upgrade it will result in an entirely separate set of contracts; in a sense forking the application.

## Colony

[Colony](https://github.com/JoinColony/colonyNetwork)

- Colony, like Aragon, has a 'ColonyStorage' superclass that stores the Contract fields.  The Colony object subclasses it.

- Colony has a [Resolver](https://github.com/JoinColony/colonyNetwork/blob/develop/contracts/Resolver.sol) contract that appears to manage

Colony outlines their use of the [EtherRouter](https://github.com/ownage-ltd/ether-router) pattern in the Token Upgradeability section of their article on the [Colony Token Sale contract](https://blog.colony.io/colony-sale-contract-and-the-clny-token-3da67d833087)




## Aragon

Aragon is a bit more of a challenge to analyze because it is an app framework rather than an app.  However, it's an ambitious project and is tackling contract versioning admirably.  Aragon itself has been recently modularized into Aragon apps.  You can see their announcement [here](https://blog.aragon.one/introducing-aragonos-3-0-alpha-the-new-operating-system-for-protocols-and-dapps-348f7ac92cff).

The Aragon system revolves around a DAO, into which apps can be deployed.  The contract representing a DAO is named the Kernel.

/*

  Open questions:

  How are Aragon 'apps' registered?

*/










## [Aragon](https://github.com/aragon/aragonOS)

- Uses a '[Kernel](https://github.com/aragon/aragonOS/blob/dev/contracts/kernel/Kernel.sol)' to lookup 'apps'.  This may be a registry, have to double-check.


# Architecture Notes:

## Augur

- Controller acts as a registry for Contracts to find and create new ones.
- Controller is able to halt activity on all contracts
- Contracts can be replaced in the controller
- The controller itself can be replaced then the rest of the contracts updated.

## uPort

- Allows users to upgrade to a new IdentityManager
- Uses a Proxy contract so that users can have a single permanent identity to operate on their behalf.

## [Gnosis](https://github.com/gnosis/gnosis-contracts)

- Split between storage and implementation, but no upgradeability

## [Rocketpool](https://github.com/rocket-pool/rocketpool)

- RocketStorage acts as both the contract registry and a generic data store.
