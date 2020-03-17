# Contract Management

This document describes the design of the freezing/unfreezing/destroying operations(referred to as contract status management operation on below) and their operation permissions in contract life cycle management.

```eval_rst
.. important::
   The contract status management operation on the contract supports storagestate storage mode, but not mptstate storage mode.The contracts mentioned here are only Solidity contracts and do not include pre-compiled contracts or CRUD contracts currently.
```

## noun explanation

Contract management related operations include `freezeContract`, `unfreezeContract`, `destroyContract`, `grantContractStatusManager`, `getContractStatus`, `listContractStatusManager`.

- [freezeContract](../../manual/console.html#freezecontract) : Reversible operation, the interfaces of a frozen contract can not be called
- [unfreezeContract](../../manual/console.html#unfreezecontract) : Undo the `freezeContract` operation, the interfaces of an unfrozen contract can be called
- [destroyContract](../../manual/console.html#destroycontract) : Irreversible operation，the interfaces of a frozen contract whose codes were deleted can not be called and cannot be restored
- [getContractStatus](../../manual/console.html#getcontractstatus) : Query the status of a contract to return the status of available/frozen/destroyed
- [grantContractStatusManager](../../manual/console.html#grantcontractstatusmanager) : Grant the account's permission of contract status managememt
- [listContractStatusManager](../../manual/console.html#listcontractstatusmanager) : Query a list of authorized accounts that can manage a specified contract

The state transition moments are shown below:

|          | available | frozen  | destroyed |
| -------- | --------- | ------- | --------- |
| freeze   | Success   | Fail    | Fail      |
| unfreeze | Fail      | Success | Fail      |
| destroy  | Success   | Success | Fail      |

## Implementation

### Record of Contract status

- Reuse the existing `alive` field in the contract table to record whether the contract has been destroyed. This field defaults to true, and the value is false when killed.
- A new field `frozen` is used to record whether the contract has been frozen. The default of this field is false, indicating that it is available. When frozen, the value is true.
- A new field `authority` is used to record accounts that can manage contract status. Each account with permission corresponds to one line of `authority` records.

**Note:**

1. False will be returned when querying the field for the contract table with no field `frozen`;
2. When a contract is deployed, the tx.origin will be written to field `authority` by default.
3. When an interface of contract A was called to create contract B, the tx.origin and the authorization of contract A is written to field `authority` of contract B by default.

### Judgment of contract status

In the Executive module, the values of alive and frozen fields are obtained according to the address of a contract, and the transaction will be executed smoothly, or an exception is thrown to indicate that the contract has been frozen or destroyed after judgment.

### Judgment of authority

- The authority to update contract status needs to be determined. Only the accounts in authority list can set the contract status;
- The authority to grant authorization needs to be determined. Only the account in the authority list can grant other accounts the authorization to manage the contract;
- Any account can query contract status and authorization list.

### Interfaces of contract management

A contract management precompiled named ContractStatusPrecompiled is added with 0x1007 address, which is used to set and query contract status.

```text
contract ContractStatusPrecompiled {
    function destroy(address addr) public returns(int);
    function freeze(address addr) public returns(int);
    function unfreeze(address addr) public returns(int);
    function grantManager(address contractAddr, address userAddr) public returns(int);
    function getStatus(address addr) public constant returns(uint,string);
    function listManager(address addr) public constant returns(uint,address[]);
}
```

### Description of return code

| code   | message                                                    |
| ------ | ---------------------------------------------------------- |
| 0      | success                                                    |
| -50000 | permission denied                                          |
| -51900 | the contract has been destroyed                            |
| -51901 | the contract has been frozen                               |
| -51902 | the contract is available                                  |
| -51903 | the contract has been granted authorization with same user |

```eval_rst
.. important::
   Contract management related operations can only be performed on 2.3 and above.
```