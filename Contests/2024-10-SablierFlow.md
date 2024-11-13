## Summary

According to [EIP4906-specification](https://eips.ethereum.org/EIPS/eip-4906#specification), the smart contracts that are implementing it must have a `supportsInferface(bytes4)` function that returns true when called with `0x49064906`. But `supportsInterface(bytes4)` doesn't return true when its called with `0x49064906`

## Vulnerability Details

The contract inherits from `SablierFlowBase` that inherits from `ERC721` that inherits from`ERC165`

But, `supportsInterface()` in `ERC721` is overridden to return true with `type(IERC721).interfaceId` && `type(IERC721Metadata).interfaceId` in addition to the default implementation

## Impact

When integrating with external protocols like NFT marketplaces or an UI wallet, they check for `supportsInterface()` function with `0x49064906` interface id to make sure that our NFTs supports `metadata and batch metadata update`. But in our case, `supportsInterface()` function is not returning `true` while it **MUST**. Thus, the NFT markets or wallets will not update the FLOW state.

Unlike other NFTs, **FLOW** `NFTs` are different. They contain various attributes like `balance`, `snapshotTime`, `ratePerSecond` and `snapshotDebtScaled` of the stream. Not updating these attributes for NFTs can lead to both recipients and sender inconvenience.

**I see this as medium severity** since the whole idea of the protocol is to track the **FLOW** Stream data and there is a trust assumptions between involved parties, so a stale data is impacting a core functionality of the protocol

## Tools Used

Manual Analysis

## Recommendations

make the `supportsInterface()` function in the `ERC721` contract like the [reference implementation](https://eips.ethereum.org/EIPS/eip-4906#reference-implementation) suggested by EIP4906 specification.
