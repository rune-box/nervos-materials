### Intro to Script
A Script in Nervos CKB is a binary executable that can be executed on-chain. It is Turing-complete and can perform arbitrary logic to guard and protect your on-chain assets. You can think of it as smart contract.

We already mentioned that a script is a small program, but it is important to understand exactly what this means. In Nervos, a "script" is a very general term that indicates the code being executed to validate a transaction. This could refer to the data structure within a cell that indicates what script code should be executed or the script code binary that is executed. Often times "script" is used interchangeably with "smart contract" when speaking in a general sense.

When we talk about a script as data structure, we are talking about a structure that indicates what script code should execute, and what arguments should be passed to it. This script structure is used in every transaction. You already used it many times in the previous Lumos examples without realizing it. The `addressToScript()` function used to specify the owner of the cell generates this for us. In the next lesson, we will cover the details about this structure.

### How a Script Work
When executing a Script, CKB takes the executables and runs them in a virtual machine environment called CKB-VM. After the execution, if the program returns a code of 0, we consider the Script successful; any non-zero return codes will be considered Script failures.

When you submit a transaction to CKB, it executes all the Scripts from the transaction to ensure that each Script succeeds. If any Script fails, the transaction will not be included on-chain.

In this way, we can allow the Cell to carry different Scripts to perform various validations for the current transaction, similar to how smart contracts work in other blockchains.

### Script Types
A Script can be one of two types:

* Lock Script - Used to control ownership and access to a Cell.
* Type Script - Used to control how a Cell is used in a transaction.

In most cases, Lock Script works the same with Type Script. The difference is that, only the Lock Script from the input Cells will be exeuted in the transaction, while the Type Script from both the input Cells and output Cells will be executed in the transaction. A lock script is concerned with ownership, and this is the reason that a lock script must execute on inputs. Once the lock script has validated the input cell, the value which was locked in that cell is now unlocked for usage. In most cases, the lock script does not need to be concerned with how the unlocked value is used. Therefore, there is no need for lock scripts to execute on output cells.

This difference has lead to the different usecases of Lock Script and Type Script as we have mentioned above. Lock Script is often used to control owner ship of a Cell while Type Script defines what kinds of changes of a Cell is valid for the transaction. A type script is concerned with state transition, and this is the reason that a type script must execute on both inputs and outputs. A very common use for type scripts is the creation of tokens. The type script will contain all the logic of the token, including the monetary policy. One of the most simple monetary policy requirements is that a user cannot create more tokens out of thin air. This is enforced with a single simple rule: `input_tokens >= output_tokens`. In effect, this means you cannot send more tokens than you already have.

### Script Structure
Script has the following structure:

```
pub struct Script {
    pub code_hash: H256,
    pub hash_type: ScriptHashType,
    pub args: JsonBytes,
}
```

The `code_hash` serves to identify a Script code, allowing the CKB-VM to load the binary code of the Script correctly.

A Script also includes the `args` part, which differentiates one Script from another using the same Script code. The `args` can provide additional arguments for a CKB Script; for example, while multiple users might utilize the same default Lock Script code, each user can have their own public key hash stored in `args`. This setup allows each user to have a unique Lock Script while sharing the same Lock Script code.

`hash_type` indicates the method CKB-VM uses to locate the Script code for a Script. Possible values include `type`, `data`, `data1`, and `data2`. Each specifies a different way of referencing the required Script code.

### Using Scripts

In the previous chapter, we used the `addressToScript()` function to specify the owner of a cell. As the name indicates, this function converts an address into a script. Specifically, this function generates a lock script. This is possible because an address is actually just a lock script encoded in a way to make it human-readable. Let's take a deeper look at what is being generated when we run the following code.

```
let lockScript = addressToScript("ckt1qzda0cr08m85hc8jlnfp3zer7xulejywt49kt2rr0vthywaa50xwsqvc32wruaxqnk4hdj8yr4yp5u056dkhwtc94sy8q");
```

When the above code is executed, `lockScript` will be set to the following.

```
{
    codeHash: '0x9bd7e06f3ecf4be0f2fcd2188b23f1b9fcc88e5d4b65a8637b17723bbda3cce8',
    hashType: 'type',
    args: '0x988a9c3e74c09dab76c8e41d481a71f4d36d772f'
}
```

The `codeHash` and `hashType` fields indicate what code should be executed. The `codeHash` value is a Blake2b hash that indicates what script code we need to execute, and `hashType` indicates how we need to treat code hash in order to match it up properly. The combination of the two together specifies what code should execute. If this doesn't make sense yet, don't worry. Later in this lesson, we will use it in an example that will make it perfectly clear.

The `args` value specifies the data that will be passed to the script when it executes. This is just like passing a few arguments to a command-line program. The value of the `args` field can be set to any value, and what is placed there is determined by the requirements of the script that is executing.

In the example above, the `codeHash` and `hashType` values specify the script code for the default lock. The `args` value is Blake2b hash of the owner's public key. When the transaction is submitted to the network, the default lock's script code will be executed and passed the `args` value. Using the `args` value in combination with the other values in the transaction, the default lock can make the determination if proper credentials were provided for this cell. If proper credentials were provided, a value of `0` is returned, indicating that execution was successful. If improper credentials were provided, then an error code will be returned, indicating that execution was not successful and that the transaction is invalid.

### Hash Type

Both a lock script and a type script use `codeHash` and `hashType` to determine what code should execute. The `codeHash` specifies a value that is used to match the code to execute. The `hashType` value controls how the `codeHash` value should be interpreted.

Below are the possible values for `hashType`. 

| Hash Type  | Matching  | CKB-VM Version    |
| ---        | ---       | ---               |
| data       | Match code by data hash.    | 0                  |
| type       | Match code by type hash.    | 1 (always newest)  |
| data1      | Match code by data hash.    | 1                  |

Using a `hashType` of `data` or `data1` indicates that the `codeHash` value must match the Blake2b hash of the binary executable located in a cell. Using `data` means to always use CKB-VM version 0, and using `data1` means to always use CKB-VM version 1. In this way Using `data` or `data1` mean that the binary must be matched byte for byte which implies that they cannot be upgraded. This is the most decentralized and trustless approach since the script owner cannot change the functionality of the binary at a later time, and the same CKB-VM version will always be used.

Using a `hashType` of `type` indicates that the `codeHash` value must match the Blake2b hash of the type script on a cell. This method means that it will execute any binary code in a cell, as long as that cell has the proper type script attached to it. This allows for upgradeable scripts, which will be covered in a later section. Using `type` always executes in the newest available version of CKB-VM because it implies upgradeability, and that the maintainer is responsible for making sure their binary is compatible with the newest version of CKB-VM.

### Cell Deps

Our lock script uses the `codeHash` and `hashType` to determine what code should execute, but it does not specify where that code exists in the blockchain. This is where cell deps come into play.

We already learned about input cells and output cells in a transaction. Cell deps are the third type. Short for cell dependencies, cell deps are similar to input cells, but they are not consumed.

Since a cell dep is not consumed, it can be used repeatedly by many scripts as a read-only component of the transaction. This enables any resource specified as a cell dep to be reused repeatedly.

Some of the common uses of cells deps are:
* Script Code - Any code that executes on-chain, such as the always success lock, is referenced in a transaction using a cell dep.
* Script Libraries - Just like a library for a normal desktop application, a script library contains commonly used code for different scripts.
* State Data - A cell can contain any data, including state data for a smart contract. Data from an oracle is a good example. The data published by the oracle is read-only and can be utilized by many smart contracts that rely on it.

With the addition of cell deps, our transaction now knows what code is needed, and where the code exists, making execution possible.

When the transaction is executed, every cell in the inputs will execute its lock script. The `codeHash` identifies what code needs to execute. The code that needs to be executed will be matched against the cell dep with a matching `dataHash`. The data field of the cell from the matching cell dep contains the script code that will be executed.

This method of providing resources enables code reuse in a way that is not possible on most other blockchains. Millions of cells can exist on-chain and all rely on a single cell dep that provides the code they need. This provides massive on-chain space savings and allows for complete code reuse between smart contracts.

