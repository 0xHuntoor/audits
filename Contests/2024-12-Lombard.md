# Unused Function in CLAdapter Contract

## Brief/Intro

The `initWithdrawalNoSignatures` function in CLAdapter.sol has the `onlyTokenPool` modifier but is never called from the `TokenPool` contract or any other location in the codebase. This creates dead code that increases contract size and gas costs unnecessarily while potentially creating confusion about the security model around signature-less withdrawals.

## Vulnerability Details

The function is defined in CLAdapter.sol:

```solidity
function initWithdrawalNoSignatures(
    uint64 remoteSelector,
    bytes calldata onChainData
) external onlyTokenPool returns (uint64) {
    _receive(getChain[remoteSelector], onChainData);
    return bridge.withdraw(onChainData);
}
```


Key observations:

1. The function has `onlyTokenPool` modifier meaning only the TokenPool contract can call it
2. Examining TokenPool.sol shows no calls to this function
3. The function allows withdrawals without signature verification unlike `initiateWithdrawal`

## Impact Details

While this is not an exploitable vulnerability since the function is protected by `onlyTokenPool`, it has several negative impacts:

1. Increased deployment costs due to unnecessary bytecode
2. Potential future security risks if the function is accidentally used instead of the signature-verified version

## References

- CLAdapter.sol: Line 192-198
- TokenPool.sol implementation
