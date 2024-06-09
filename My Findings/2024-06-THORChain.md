# Findings

## M-1: Calling `transferOutAndCallV5` function with fee on transfer token will result in DOS
Any time `transferOutAndCallV5` function was called with a fee on transfer token and the aggregator contract have 0 balance of the token the transaction will fail because a value that is greater than the value (-fee) received by the aggregator contract was used in calling `swapOutV5` function look at line 17 of below code.
```javascript
function _transferOutAndCallV5(TransferOutAndCallData calldata aggregationPayload) private {

  // ..

  // transfer aggregationPayload.fromAmount token to aggregator but it will receive aggregationPayload.fromAmount - fee
  (bool transferSuccess, bytes memory data) = aggregationPayload.fromAsset.call(abi.encodeWithSignature("transfer(address,uint256)"aggregationPayload.target, aggregationPayload.fromAmount));
  
  require(transferSuccess && (data.length == 0 || abi.decode(data, (bool))), "Failed to transfer token before dex agg call");

  (bool _dexAggSuccess, ) = aggregationPayload.target.call{value: 0}(abi.encodeWithSignature("swapOutV5(address,uint256,address,address,uint256,bytes,string)",
  aggregationPayload.fromAsset,
  aggregationPayload.fromAmount, // original aggregationPayload.fromAmount was used in the swapOutV5 instead of aggregationPayload.fromAmount - fee
  aggregationPayload.toAsset,
  aggregationPayload.recipient,
  aggregationPayload.amountOutMin,
  aggregationPayload.payload,
  aggregationPayload.originAddress
  ));


  // ..

}
```

And in the `swapOutV5` function aggregationPayload.fromAmount passed to the function was was used to approve swapRouter by the aggregator contract which is greater than what it received and later the fromAmount was used to call `swapExactTokensForETH` which will fail due to insufficient of fund.


```javascript
function swapOutV5(address fromAsset, uint256 fromAmount, address toAsset, address recipient, uint256 amountOutMin, bytes memory payload, string memory originAddress) public payable nonReentrant {

  // ..

  // The aggregator contract approve the swapRouter to spend fromAMount which is greater than what it received (fromAmount - fee)
  safeApprove(fromAsset, address(swapRouter), 0);
  safeApprove(fromAsset, address(swapRouter), fromAmount);

  if (payload.length == 0) {
    // no payload, process without wasting gas
    // UniV2 transfers from 'msg.sender', so we need a low-level `.call` to change the msg.sender from the target's perspective
    (bool aggSuccess, ) = address(swapRouter).call(abi.encodeWithSignature(
      "swapExactTokensForETH(uint256,uint256,address[],address,uint256)",
      fromAmount,
      amountOutMin,
      path,
      recipient,
      type(uint).max // Assuming using the maximum uint256 value as the deadline));

    require(aggSuccess, "swapExactTokensForETH failed");
  } else {

  // ..

}
```

###Proof of code


###Recommendation
Use the amount minus fee for calling `swapOutV5` or make sure the aggregator contract does not have zero value of the token.

- [M-1: Calling `transferOutAndCallV5` function with fee on transfer token will result in DOS]

```javascript
function _transferOutAndCallV5(TransferOutAndCallData calldata aggregationPayload) private {
  if (aggregationPayload.fromAsset == address(0)) {
    // call swapOutV5 with ether
    (bool swapOutSuccess, ) = aggregationPayload.target.call{value: msg.value} (abi.encodeWithSignature(
      "swapOutV5(address,uint256,address,address,uint256,bytes,string)",
      aggregationPayload.fromAsset,
      aggregationPayload.fromAmount,
      aggregationPayload.toAsset,
      aggregationPayload.recipient,
      aggregationPayload.amountOutMin,
      aggregationPayload.payload,
      aggregationPayload.originAddress));

      if (!swapOutSuccess) {
        bool sendSuccess = payable(aggregationPayload.target).send(msg.value); // If can't swap, just send the recipient the gas asset

        if (!sendSuccess) {
          payable(address(msg.sender)).transfer(msg.value); // For failure, bounce back to vault & continue.
        }
      }


      // code after ...
  }
}
```

Recommendation
```diff
function _transferOutAndCallV5(TransferOutAndCallData calldata aggregationPayload) private {
  if (aggregationPayload.fromAsset == address(0)) {
    // call swapOutV5 with ether
    (bool swapOutSuccess, ) = aggregationPayload.target.call{value: msg.value} (abi.encodeWithSignature(
      "swapOutV5(address,uint256,address,address,uint256,bytes,string)",
      aggregationPayload.fromAsset,
      aggregationPayload.fromAmount,
      aggregationPayload.toAsset,
      aggregationPayload.recipient,
      aggregationPayload.amountOutMin,
      aggregationPayload.payload,
      aggregationPayload.originAddress));

      if (!swapOutSuccess) {
-       bool sendSuccess = payable(aggregationPayload.target).send(msg.value); // If can't swap, just send the recipient the gas asset
+       bool sendSuccess = payable(aggregationPayload.recipient).send(msg.value);

        if (!sendSuccess) {
          payable(address(msg.sender)).transfer(msg.value); // For failure, bounce back to vault & continue.
        }
      }


      // code after ...
  }
}
```
