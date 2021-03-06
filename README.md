# Stoner Cats Audit

Ethereum contract ERC-721 "Stoner Cats" was taken for audit.  
DAppRadar link https://dappradar.com/ethereum/collectibles/stoner-cats  
https://etherscan.io/address/0xd4d871419714b778ebec2e22c7c53572b573706e#code  
The review is taken based on the contract code deployed by address: 0xd4d871419714b778ebec2e22c7c53572b573706e. 
The list of files in the area can be found in the Appendix.  
## Objectives:
Verifying that the system has been implemented as intended and without unexpected edge cases.  
Identification of well-known vulnerabilities in smart contracts.  
Particular attention is paid to preventing attacks that can lead to the loss of system / user funds.  
## System Overview

https://github.com/77ph/stoner-cats-audit/blob/main/StonerCats-inheritance.png  
https://github.com/77ph/stoner-cats-audit/blob/main/StonerCats-graph.png  
https://github.com/77ph/stoner-cats-audit/blob/main/StonerCats-mdreport.md  

## Findings

Each issue is assigned a severity:  
⚠️Minor problems are subjective. Typically, these are suggestions for best practices or readability. It is up to the maintainers of the code to decide whether such problems should be addressed. Just "IMHO"-tag.  
❗Medium issues are objective, but not security vulnerabilities. They should be consulted unless there is a clear reason not to.  
❌The main issues are security vulnerabilities that cannot be exploited directly or may require specific conditions to be exploited.   
🔥Critical issues are security vulnerabilities that can be exploited directly and need to be fixed.  

### 1. ❗/⚠️ reentrancy-vulnerabilities in function reserveGiveaway()

    /*
    * Only valid before the sales starts, for giveaways/team thank you's
    */
    function  reserveGiveaway(uint256  numTokens) public  onlyOwner {

Not executable by an non-owner user, but an existing vulnerability "reentrancy".
Solution: Add modifier nonReentrant.  
Notes: From a practical point of view, ❗❗❗ it is not a vulnerability, since only owner can call it.
### 2. ❗ No change in function mewnt()

    function  mewnt(uint256  numTokens) external  payable  nonReentrant {
This issue is not an owner vulnerability. However, it can hurt the buyer.

    require(msg.value >= calculatePrice().mul(numTokens),"Ether value sent is below the price");

If `msg.value > calculatePrice().mul(numTokens)` then `change = msg.value - calculatePrice().mul(numTokens)` will not be returned to the buyer.
Solution: Add change in mewnt(). 

    uint256 change = msg.value.sub(calculatePrice().mul(numTokens));
    if(change > 0) {
	    require(payable(msg.sender).send(change));
    }

### 3. ⚠️. mewnt() vs reserveGiveaway()

    for (uint i = 0; i < numTokens; i++) {
	    uint mintId = nextTokenId++;
	    _safeMint(msg.sender, mintId);
	    _setTokenURI(mintId, Strings.strConcat(Strings.uint2str(mintId), "/index.json"));
    }
vs

    for (index = 0; index < numTokens; index++) {
        nextTokenId++;
        _safeMint(owner(), currentSupply + index);
        _setTokenURI(currentSupply + index, Strings.strConcat(Strings.uint2str(currentSupply + index), "/index.json"));
    }
It would be logical to unify this code, which does the same thing in this part.
owner() always equal msg.sender in reserveGiveaway() because "public onlyOwner".

### 4. ⚠️. issue in function calculatePrice()

    require(hasSaleStarted == true, "Sale hasn't started");
    require(totalSupply() < CURR_SUPPLY_CAP,"We're at max supply!");
Revert in "view" It does not seem like a good solution. It is not very logical, before the start of sales, not to give an answer what the price will be, especially since it is a constant.

    hasSaleStarted == true
Boolean constants can be used directly and do not need to be compare to `true` or `false`.

    return 350000000000000000; // 0.35 ETH
Literals with many digits are difficult to read and review.

    uint256 constant public price = (1 ether * 35)/ 100;
    ..
    return price;

### 5. ⚠️. issue in function tokensOfOwner()

     if (tokenCount == 0) {
	     // Return an empty array
	    return  new  uint256[](0);
    } else {
It is not obvious that special handling of this case is necessary, since the code below does the same for the case tokenCount == 0
### 6. ⚠️.  no function isSaleStarted()
It would be nice to have a function:

    function  isSaleStarted() public  view  returns (bool) {
    return hasSaleStarted;
    }
Then

    bool  public hasSaleStarted = false;

could be announced as "private".

## Token Metadata ERC721
Every NFT is identified by a unique uint256 ID inside the ERC-721 smart contract. 
The URI may point to a JSON file that conforms to the "ERC721 Metadata JSON Schema".

Not directly related to this contract (ERC721 standard), but a more general problem is the centralized presentation of meta information for a decentralized token.

https://nftschool.dev/reference/metadata-schemas/#linking-to-nft-assets  
https://docs.ipfs.io/how-to/best-practices-for-nft-data/#types-of-ipfs-links-and-when-to-use-them  

The only way to solve this contradiction within the framework of existing technologies is to use IPFS.
https://docs.ipfs.io/how-to/best-practices-for-nft-data/#metadata  

In the project under consideration, this is solved through a special header in the response of the centralized service:
< x-ipfs-path: /ipns/quick-senior-square-griffin.fission.app/json/1/index.json

Example:
curl -L -k -v https://go.fission.app/json/1/index.json  
curl -L -k -v https://ipfs.io/ipns/quick-senior-square-griffin.fission.app/json/1/index.json  

curl -L -k -v https://go.fission.app/json/1/image.jpg  
curl -L -k -v https://ipfs.io/ipns/quick-senior-square-griffin.fission.app/json/1/image.jpg  

Thus, it can be assumed that information can be accessed through IPFS. 

https://eips.ethereum.org/EIPS/eip-721  
Details of the problem here:
https://eips.ethereum.org/EIPS/eip-2477

# Appendix
https://github.com/77ph/stoner-cats-audit/blob/main/StonerCats-flattern.sol

