# Summary

| ID                                                                                                                             | Title                                         | Severity |
| ------------------------------------------------------------------------------------------------------------------------------ | --------------------------------------------- | -------- |
| [L-01](2024-08-Midas.md#m-01-discrepancies-between-the-spec-and-the-code-'redeem-mTBILL-for-USDC-pulled-from-BUIDL'-Edge-case) | Compliance issue with eip-4906 in sablierflow | low      |


# Compliance issue with eip-4906 in sablierflow


## summary

sablierflow is supposed to be compliant with [EIP-4906]([https://eips.ethereum.org/EIPS/eip-4906#specification)](https://eips.ethereum.org/EIPS/eip-4906#specification) , but it's not due to the `supportsInterface` function in erc165 been overridden in erc721 function.

## Vulnerability Details

As specified in docs specs for eip-4906

> The `supportsInterface` method MUST return `true` when called with `0x49064906`.

This hasn't been met as the has been overwritten

```
File: ERC721.sol
47:     function supportsInterface(bytes4 interfaceId) public view virtual override(ERC165, IERC165) returns (bool) {
48:         return
49:@>           interfaceId == type(IERC721).interfaceId ||
50:@>           interfaceId == type(IERC721Metadata).interfaceId ||
51:             super.supportsInterface(interfaceId);
52:     }
```

Note that this contract is inherited in sablierflowbase contract also erc721 inherit `ERC165`so the function is overridden

## impact

`supportInterface` Must always return true with `0x49064906`, but as shown above it doesn't

knowing that Different NFTs have different metadata, and calling `supportsInterface()` with `0x49064906`  checks metadata and batch update support.

When integrating with NFT marketplaces or an UI wallet the Flow state won't be updated.

## Tools Used

manual review

## Recommendations

adjust the `supprotsInterface` function

```
    function supportsInterface(bytes4 interfaceId) public view virtual override(IERC165, ERC721) returns (bool) {
        return interfaceId == bytes4(0x49064906) || super.supportsInterface(interfaceId);
    }
```