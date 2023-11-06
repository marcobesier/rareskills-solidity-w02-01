# ERC721A

## How does ERC721A save gas?

### Optimization 1 - Removing duplicate storage from OpenZeppelin's ERC721Enumerable

OZ's ERC721Enumerable implementation includes redundant storage of each token's metadata. This approach optimizes for read functions at a significant cost to write functions, which is often not ideal since users are, in most cases, much less likely to pay for read functions. ERC721A removes this duplicate storage, saving considerable gas when users use write functions.

Furthermore, the fact that all tokens are serially numbered starting from 0 allows the ERC721A implementation to remove some additional redundant storage from OZ's implementation.

### Optimization 2 - Updating the owner's balance once per batch mint request, instead of per minted NFT

In the case of batch mints, it is much more gas-efficient to update a user's balance from the current state directly to the final state, e.g., from 2 to 7 (in case of a batch size of 5), rather than updating the balance value once per additional token (e.g., from 2 to 3, 3 to 4, etc.).

Updating storage is one of the most gas-intensive operations one can perform on Ethereum. Still, most projects have not adopted this approach yet because the default OZ implementation does not include a batch mint API. 

### Optimization 3 - Updating the owner data once per batch mint request, instead of per minted NFT

This is similar to the previous optimization. Suppose Alice wanted to buy 3 tokens - token #42, #43, and #44. Instead of saving Alice as the owner 3 times, one can instead save the owner value just once in a way that semantically implies that Alice owns all 3 of those tokens.

As an example, suppose Alice mints tokens #42, #43, and #44, and Bob mints tokens #45, and #46. In that case, the internal ERC721A tracker would look something like this:

- #42
  - Owner: Alice
- #43
  - Owner: _not set_
- #44
  - Owner: _not set_
- #45
  - Owner: Bob
- #46
  - Owner: _not set_

The key here is that if one wanted to see who owned #44, one doesn't need to have Alice set explicitly as the explicit owner of #44 to do so. One could just change the ownerOf function to do the following:

```solidity
function ownerOf(uint256 tokenId) public view virtual override returns (address) {
    require(_exists(tokenId), "ERC721A: owner query for nonexistent token");

    uint256 lowestTokenToCheck;
    if (tokenId >= maxBatchSize) {
        lowestTokenToCheck = tokenId - maxBatchSize + 1;
    }

    for (uint256 curr = tokenId; curr >= lowestTokenToCheck; curr--) {
        address owner = _owners[curr];
        if (owner != address(0)) {
            return owner;
        }
    }

    revert("ERC721A: unable to determin the owner of token");
}
```

This approach results in significant gas savings at mint, thus decreasing the severity of concentrated gas spikes for the entire ecosystem at mint time. Notice that this optimization involves implementing some additional logic, especially when it comes to transfers.

## Where does ERC721A add cost?

### Security
ERC721A is not as standard and battle-tested as OZ's implementation. Therefore, implementing this approach is riskier from a security perspective.

### Updated ownerOf adds to long-term gas costs
While the ownerOf implementation described above results in significant gas savings at mint, calling the function itself becomes more costly later down the line because the owner often needs to be searched by the contract rather than being directly available for the requested token ID.
