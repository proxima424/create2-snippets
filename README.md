# create2-snippets

This repository provides some code snippets that are useful for `create2` that was introduced in Ethereum Constantinople update. 

## Environment

The article uses the following environment for solidity development and testing:
* Solidity: 0.5.16
* Truffle: 5.1.9
* ganache-cli: 6.9.0
* Node: 11.15.0

## Basic concepts

`create2` is a new opCode that has been introduced into Ethereum in the Constantinople update and provides a new way for contracts to deploy contracts. To understand what is going under the hood, we need to know how EVM handles when there's a transaction that tries to deploy a contract.

### The three kind of bytecodes

When we compile a contract, it turns into bytecode that can be run in EVM. There are two kinds of bytecode that would be the product of the compilation. The first piece of bytecode is called the `initCode` and only exists and being executed when the contract is being deployed. The `initCode` manipulates storage, and in the end it returns the bytecode that would stay in the EVM storage space. The bytecode it returns is the `runtime bytecode` and is what we typically refer to as the deployed smart contract. If you only code through solidity, then you could make the following analogy:

* `initCode`: the constructor of the smart contract that is being deployed, along with its arguments
* `runtime bytecode`: everything except the constructor in the smart contract

Indeed, the smart contract we write in solidity will actually be broken down after we compiled it. This is also the reason why in solidity the "constructor" doesn't exist when you try to call it in other functions of the same contract. 

When you compile with solidity (without assembly), the compiled code for sure returns the `runtime bytecode` of the smart contract you are deploying. However, one could be more creative and make the `initCode` to be more complex. For example, have the `initCode` to return two different pieces of bytecode based on some condition. This would in turn make the EVM to deploy different `runtime bytecode` under different situations. We will revisit this later and see the creative ways of using `create2`.

The `initCode` can be further broken down into two pieces. One is the `creationCode`, which can be viewed as the `constructor` logic, and the arguments that are provided to the `constructor`. They are concatenated together to form the `initCode`. 

Here's a summary of the relationship between `creationCode`, `initCode`, and `runtime bytecode` in terms of pseudocode:

* `initCode` = `creationCode` + (arguments)
* `runtime bytecode` = EVM(`initCode`)

### create2

The way which `create` and `create2` deploys the contract are the same. The only difference is how the address of the deployed contract is being calculated. `create` depends on some blockchain state whereas `create2` is independent of the blockchain state. This independence enables developers to calculate the address with certainty without consluting with the blockchain, in short, we can calculate addresses off-chain.

`create2` calculates the address with: (1) hash of `initCode` (2) salt (3) msg.sender: the address that deploys the new contract.

`create2` is used by one contract to deploy another contract, we will refer to them as factory contract and target contract respectively.

## Deploying contracts with create2

Solidity provides assembly level support for `create2`: `create2(v, p, n, s)`, with `v` being the amount of ether in wei sent along in the transaction, `p` being the start of the `initCode`, `n` being the length of `initCode`, and `s` a 256-bit value. (It also provides higher level support in 0.6.x, but as the 0.6.x are not really production ready because the lack of tools, I would stick to the assembly.)

In Solidity, bytecode can be stored in `bytes memory`, a dynamic bytes array. This is what we would used to store `initCode`. Let's suppose that we have already declared a `bytes memory` variable called `initCode` and it properly stores the `initCode` that is used to deploy a contract. as `s` requires a 256-bit value, we could declare a `uint256 salt` and use it. 

With these, we can deploy a contract with `create2` as follow:

```
assembly{
  deployedContract := create2(0, add(initCode, 0x20), mload(initCode), salt)
}
```

Let's disect the assembly here and gain a deeper understanding. 

As the `initCode` is a dynamic bytes array, in solidity, the first 32 bytes stored in its location is a `uint256` that stores the length of the array. The actual data is stored after those 32 bytes. The `p` requires the start of the actual code, which is the location of our data, thus we would shift the location of `initCode` with 32 bytes: `add(initCode, 0x20)`. `n` asks for the length, which is exactly the first 32 bytes stored in the bytes array, we can use `mload` as it loads the 32 bytes starting from its argument: `mload(initCode)`.

## Now, where do we get the initCode?

As previously described, `initCode` is the piece of bytecode that deploys the smart contract and you would need this for your `create2`. Where could we get them? We can certainly obtain it by compiling the smart contract with `solc`, but that's not the most convenient way to do things. In Solidity, it is possible to obtain the `creationCode` of the target contract `DeployMe` by:

```
bytes memory creationCode = type(DeployMe).creationCode;
```

Recall that creationCode is not the `initCode` yet. If the constructor of the target contract `DeployMe` needs arguments, then you will need to append the encoded arguments after the creationCode. If not, then the `creationCode` would be the `initCode` that we needed. 

* If the contract's contstructor don't have arguments: 

```
contract DeployMe{
  constructor(){}
  // ...
}

// ...

bytes memory initCode = creationCode;
```

* If the contract's constructor have arguments: 
```
contract DeployMe{
  constructor(uint256 someNumber, address someAddress){
    // ....
  }
  // ...
}

// ...

bytes memory initCode = abi.encode(creationCode, someNumber, someAddress);
```

The above would come very handy when developing in solidity. To write tests, we would need a way to efficiently load them into javascript. If you use the truffle framework, then the compiled bytecode can be found in the json files of the `build` directory after compilation. 

In the json files, you could find two pieces of bytecode: one labeled as `bytecode` and the other labeled as `deployedBytecode`. The `bytecode` is the `creationCode` and the `deployedBytecode` is the `runtime bytecode` that we have mentioned above. They could be imported through a require statement. 

```
const { abi: DeployMeABI, bytecode: DeployMeCreationCode } = require('../build/contracts/DeployMe.json');
```

