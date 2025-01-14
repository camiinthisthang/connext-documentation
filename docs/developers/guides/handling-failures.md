---
sidebar_position: 5
id: handling-failures
---

# Handling Failed xCalls

There are a few failure conditions to watch out for when using `xcall`. 

## High Slippage

When tokens are bridged through Connext, slippage can impact the `xcall` during the swap on origin or destination. If slippage is too high on the origin swap, the `xcall` will just revert. If slippage is too high on the destination swap (after it has already gone through the origin swap), then there are a couple options to consider.

- Cancel the transfer and bridge back to origin (sender will lose any funds related to the origin slippage) [not available yet].
- Wait it out until slippage conditions improve (relayers will continuously re-attempt the transfer execution).
- Increase the slippage tolerance.

### Increasing Slippage Tolerance

The [`_delegate`](../reference/contracts/calls#parameters-8) parameter of `xcall` is an address that has rights to update the original slippage tolerance by calling Connext's [forceUpdateSlippage](https://github.com/connext/nxtp/blob/27bbf7871a78b03d8613b06ece2675a57309d573/packages/deployments/contracts/contracts/core/connext/facets/BridgeFacet.sol#L395) function with the following signature:

```solidity
function forceUpdateSlippage(TransferInfo calldata _params, uint256 _slippage) external;
```

The `TransferInfo` struct that must be supplied:

```solidity
struct TransferInfo {
  uint32 originDomain;
  uint32 destinationDomain;
  uint32 canonicalDomain;
  address to;
  address delegate;
  bool receiveLocal;
  bytes callData;
  uint256 slippage;
  address originSender;
  uint256 bridgedAmt;
  uint256 normalizedIn;
  uint256 nonce;
  bytes32 canonicalId;
}
```

The parameters in `TransferInfo` must match the same parameters used in the original `xcall`. It's possible to obtain the original parameters by [querying the subgraph](./xcall-status#querying-subgraphs) (origin *or* destination) with the `transferId` associated with the `xcall`.


## Reverts on Receiver Contract

If the call on the receiver contract (also referred to as "target" contract) reverts, the Connext bridge contract may unwillingly end up with custody of funds involved in the operation. The xApp developer should be aware that in these cases, those funds may be considered forfeit. To avoid these kinds of situations, developers should build any contract implementing `IXReceive` defensively. 

Ultimately, the goal should be to handle any revert-susceptible code and ensure that the logical owner of funds *always* maintains agency over them.

### Try/Catch with External Calls

One way to guard against unexpected reverts is to use `try/catch` statements which allow contracts to handle errors on external function calls.

```solidity
contract TargetContract {
  ...
  function xReceive(
    bytes32 _transferId,
    uint256 _amount,
    address _asset,
    address _originSender,
    uint32 _origin,
    bytes memory _callData
  ) external returns (bytes memory) {
    try {
      someExternalCall();
    } catch { 
      // Make sure funds are delivered to logical owner on failing external calls
    }
  }
}
```
