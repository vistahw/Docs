# How to use PodDB in contract?

PodDB is an on-chain database, and easy use of PodDB in contract is a basic design goal of PodDB. For ease of understanding, let's use an example of user reputation sharing to illustrate how to use PodDB in a contract.

Suppose you are a user reputation provider, you can evaluate the user's reputation value in a certain aspect according to the user's address in various chain protocol operation records, such as Defi interaction records, DAO governance records, etc., so that your customer can accurately identify his target user according to the user's reputation value, so as to better participate in the following business. For example, the reputation value of the target user can be used to determine how many airdrops he can get.

After researching and demonstrating, you decided to use PodDB to store and manage user reputation data because storing it in PodDB would be a better way to share and collaborate with your target customers than storing it in your own contract.

## Create TagClass

In PodDB, all data belongs to a certain category, which is used to distinguish different data under the same object.

The category of  data in PodDB is TagClass. Therefore, to store user reputation data, we first create a TagClass, which we call ReputationTagClass. ( TagClass is generally created through our dApp tool or SDK. In order to introduce how to use PodDB in the contract, I chose to create this ReputationTagClass directly in the contract).

To do this, we create a contract, then create a ReputationTagClass in the constructor of this contract, and record the TagClassId of the ReputationTagClass. ( TagClassId is very important, it is used to uniquely identify a TagClass).

```solidity
// SPDX-License-Identifier: GPL-3.0
pragma solidity ^0.8.4;

import "./librarys/OpenZeppelin/Ownable.sol";
import "./librarys/PodDB/IPodDB.sol";
import "./librarys/PodDB/Helper.sol";
import "./librarys/PodDB/WriteBuffer.sol";

contract Reputation is Ownable {
    using Helper for *;
    using WriteBuffer for *;
    using ReadBuffer for *;

    IPodDB private _PodDB;
    bytes20 ReputationTagClassId;

    constructor(address podBDAddress) Ownable() {
        _PodDB = IPodDB(podBDAddress);

        //create reputation tagClass;
        Helper.TagClassFieldBuilder memory builder;
        builder.init().put("Score", IPodDB.TagFieldType.Uint16, false);
        string memory tagClassName = "reputation";
        string memory tagClassDesc = "Reputation for user";
        uint8 tagClassFlags = 0;
        IPodDB.TagAgent memory agent;
        ReputationTagClassId = _PodDB.newTagClass(
            tagClassName,
            builder.getFieldNames(),
            builder.getFieldTypes(),
            tagClassDesc,
            tagClassFlags,
            agent
        );
    }
}
```

The IPodDB.sol, Helper.sol, and WriteB uffer.sol used in the contract can be found in the PodDB code base.

It should be noted that PodDB may be upgraded at a later time. In order to use the data generated by the new version of PodDB, the ability to update the PodDB contract address is very important in the contract. ( PodDB provides post-compatibility, that is, the new version of PodDB can parse the old version of data, but the old version of PodDB cannot parse the new version of data).

Therefore, it is best to add the following code to the contract:

```solidity
function setPodDB(address podDBAddress) public onlyOwner {
    _PodDB = IPodDB(podDBAddress);
}
```

### Define TagClass fields

The most important TagClass definition is the TagClass field definition, which defines what data can be stored under this TagClass, and also tells others how to parse the data you define.

The TagClass definition consists of two parts: the field type definition and the field name definition.

The field type must be the type defined in the PodDB type enumeration. The field type definition directly uses the hexadecimal encoding of the type emuneration value. There is no separator between the field type values. If a field is an array, the type value of the array must be added to the front of the type definition.

The field name definition uses a comma "," delimited list of field names directly, and the number must be consistent with the number of field names.

To facilitate field definition, PodDB's Helper tool library provides a TagClass FieldBuilder struct to help PodDB users define TagClass fields. In ReputationTagClass, in order to simplify the problem, we only define a field of type Uint16 with the name Score to represent the user's specific reputation value.

### Set agent

In order to better manage TagClass and the data generated under this TagClass, modifying TagClass and creating Tag require this TagClass write permission.

The write-permission of the TagClass is usually the owner of the TagClass (the Owner  is the definer of the TagClass usually). Of course, the owner of this TagClass can also change the ownership to another address.

If only the owner has write-permission, it is difficult to meet the requirements of complex scenarios. To this end, PodDB has designed a proxy mechanism to allow owner to proxy TagClass's write-permissions by an agent to meet more complex permission designs.

Please refer to the PodDB design documentation for more information. Typically, the TagClass owner is an external account (EOA account) and the Agent is a contract account (CA account).

For simplicity, ReputationTagClass does not have an Agent set up, and can be set up again if necessary.

## Write data

The basic data unit in TagClass is Tag, and the operation of writing data is called SetTag. If the same TagClass is repeatedly written to the same user, the following will overwrite the previous data.

The calculation of Reputation scoer is the core of our business, which is usually done off the chain. After the calculation, we can save the user's Reputation score in PodDB through the setReputation contract function. The code for setReputation is as follows:

```solidity
function setReputation(uint16 reputation) public returns (bytes20) {
    IPodDB.TagObject memory object = IPodDB.TagObject(msg.sender, 0);
    WriteBuffer.buffer memory wBuf;
    bytes memory data = wBuf.init(2).writeUint16(reputation).getBytes();
    uint32 expiredTime = 0; //never expired;
    uint8 tagFlag = 0;
    bytes20 tagId = _PodDB.setTag(
        ReputationTagClassId,
        object,
        data,
        expiredTime,
        tagFlag
    );
    return tagId;
}
```

The Object of the data is called a TagObject. TagObject consists of address and TokenId, which can be used to describe an externally owned address (EOA address), a contract address (CA address), or even another TagClass object. When describing an NFT, you need to specify both the contract address and the TokenId.

In this example, TagOject is a user address, so it can be constructed using IPodDB. TagObject (msg.sender, 0).

The most important thing in SetTag is to construct the written data. The data needs to be serialized for transfer and storage between contracts.

The WriteBuffer in PodDB can complete the serialization operation. WriteBuffer is a bytes buffer that can be dynamically expanded. In order to avoid unnecessary expansion operations in use, it is best to init sufficient storage capacity at the beginning.

### The data ExpiredTime

When you set Tag, you can set the expired time of the Tag. The expired time is a Uint32 number in seconds. The actual expiration time needs to add the block time when the transaction was sent.

After expired, the Tag cannot be queried.

Because the calculation of the ReputationTag score does not currently need to consider the  expiration time, the expiration time is set to 0. That means it never expired.

## Read data

As a Reputation data service provider, we usually don't read our own data in PodDB, but in order to explain how to read data from PodDB, we also put this part of the code into the Reputation contract.

There are usually three contract functions for reading data from PodDB: hasTag, getTagData, and getTagByObejct.

### hasTag

HasTag is used to determine whether a certain TagObject has a Tag under a specific TagClass, regardless of what the data in the Tag is.

If the TagObject has no Tag, or Tag expired, hasTag return false.

HasTag is commonly used in permission systems to check whether a user has some kind of permission.

### getTagData

getTagData is used to obtain the Tag data of a certain TagObject under a specific TagClass. If the user does not have this Tag, or this Tag has expired, an empty byte array will be returned.

This is the choice in most cases, as is the Reputation Contract. The specific code is as follows:

```solidity
function getReputation(address user)
    public
    view
    returns (uint16 score)
{
    IPodDB.TagObject memory object = IPodDB.TagObject(user, 0);
    bytes memory data = _PodDB.getTagData(ReputationTagClassId, object);
    if(data.length == 0){
        return 0;
    }
    ReadBuffer.buffer memory rBuf = ReadBuffer.fromBytes(data);
    score = rBuf.readUint16();
    return score;
}
```

After getting the tag data, you should first check whether the data is empty, maybe this user does not have this Tag at all.

Then you need to parse the Data in the Tag.

Parsing Data requires knowing the fields type definition of this TagClass (this definition can be viewed through the SDK or PodDB dApp tool).

According to this definition, each field can be read one by one through the ReadBuffer library in PodDB. In  ReputationTagClass, the field is a Uint16, so you only need readUint16 to get the score.

If one of these fields is array type, you need to read the length of the array with readLength first, and then read the array elements according this number.

### getTagByObject

Compared with getTagData, getTagByObject can obtain more metadata about Tags, such as expiration time. But this is not necessary in most scenes.

## Delete data

For some tags that you don't want to keep, you can delete them through PodDB's deleteTag contract function. The code is as follows:

```solidity
function deleteReputation(bytes20 tagId) public returns(bool){
    IPodDB.TagObject memory object = IPodDB.TagObject(msg.sender, 0);
    return _PodDB.deleteTag(tagId, ReputationTagClassId, object);
}
```

## Update TagClass

After TagClass is created, everything can be modified except the field type and field name. The TagClass Name and Desc can be modified through PodDB's UpdateTagClassInfo contract function.

The Owner, Agent, and TagClassFlags can be modified through PodDB's UpdateTagClass contract method.

### Deprecated

TagClass cannot be deleted after creation, but can be set to deprecated.  The deprecated TagClass can be viewed, and the Tags of the TagClass can be viewed and deleted, but can not create a new Tag and modify the old Tag.

Set TagClass deprecated can be achieved through UdpateTagClass's Flags. The code is as follows:

```solidity
function deprecatedReputationTagClass() public onlyOwner {
    IPodDB.TagClass memory reputationTagClass = _PodDB.getTagClass(
        ReputationTagClassId
    );
    reputationTagClass.Flags |= 128;
    _PodDB.updateTagClass(
        reputationTagClass.ClassId,
        reputationTagClass.Owner,
        reputationTagClass.Agent,
        reputationTagClass.Flags
    );
 }
```

Of course,  The deprecated TagClass can be set to normal state through UpdateTagClass.

The complete code can be found in [poddb-evm-example](https://github.com/PodDBio/poddb-evm-example.git) repo.