# Storing NFT Data On-Chain

I do not really like the way NFTs are handled today with regards to the location of their storage.
As far as I know, only [Crypto Punks](https://www.larvalabs.com/blog/2021-8-18-18-0/on-chain-cryptopunks) and [On Chain Monkey](https://onchainmonkey.com/ )store data (+ algorithm) on-chain 
There are obviously some solutions such as [IPFS](https://ipfs.io/), [Filecoin](https://filecoin.io/) and others
like [NFT Storage](https://nft.storage/), but this blog has been largely inspired by Crypto Punks: how to generate images from Solidity.  

## EIP-2569
[EIP-2569](https://eips.ethereum.org/EIPS/eip-2569) "Saving and Displaying Image Onchain for Universal Tokens" defines a simple contract
that returns a string based on a token id:

```solidity
import "@openzeppelin/contracts/token/ERC721/IERC721.sol";

contract IERC721GetImageSvg is IERC721 {
    function getTokenImageSvg(uint256 tokenId) external view returns (string memory);
}
```

The above method returns a string, an [SVG](https://developer.mozilla.org/en-US/docs/Web/SVG), an 
"an XML-based markup language for describing two-dimensional based vector graphics".

## NFT Solidity Code
I have chosen to generate some random squares in the Solidity NFT code, resulting in something like:

![On-Chain NFT Image](./nft2.svg)

The size and complexity of computing has a direct impact on gas fees of course.

```solidity
// SPDX-License-Identifier: MIT
// Author: TJ
// https://eips.ethereum.org/EIPS/eip-2569
pragma solidity ^0.8.9;

import "@openzeppelin/contracts/token/ERC721/ERC721.sol";
import "@openzeppelin/contracts/utils/Strings.sol";
import "@openzeppelin/contracts/access/Ownable.sol";

contract OnChainNFT is ERC721, Ownable {
    uint256 internal constant SIZE = 24;

    struct Shape {
        uint256 x;
        uint256 y;
        uint256 w;
        uint256 h;
        uint256 r;
        uint256 g;
        uint256 b;
    }

    Shape[] _shapes;

    constructor(uint256 seed) ERC721("RNF", "Real NFT") {
        uint256 nb_rect = 5 + rnd(5, seed++);
        for (uint i = 0; i < nb_rect; i++) {
            uint256 x = rnd(50, seed++);
            uint256 y = rnd(50, seed++);
            uint256 w = rnd(30, seed++);
            uint256 h = rnd(30, seed++);

            uint256 r = rnd(256, seed++);
            uint256 g = rnd(256, seed++);
            uint256 b = rnd(256, seed++);
            _shapes.push(Shape(x, y, w, h, r, g, b));
        }
    }

    function tokenAsSVG() external view returns (string memory svg) {
        string memory svg_header = '<svg width="64" height="64" viewBox="0 0 64 64" xmlns="http://www.w3.org/2000/svg" xmlns:xlink="http://www.w3.org/1999/xlink">';
        svg = string(abi.encodePacked(svg_header));

        for (uint i = 0; i < _shapes.length; i++) {
            Shape memory s = _shapes[i];
            svg = string(abi.encodePacked(svg,
                        '<rect x="', toString(s.x), 
                        '" y="', toString(s.y),
                        '" width="',toString(s.w),
                        '" height="',toString(s.h),
                        '" style="fill:rgb(', toString(s.r), ',', toString(s.g), ',', toString(s.b), ')"',
                        '/>'));
        }

        svg = string(abi.encodePacked(svg, '</svg>'));
    }

    function rnd(uint256 upper, uint256 seed) private view returns (uint256) {
        return uint(keccak256(abi.encodePacked(block.timestamp, msg.sender, seed))) % upper;
    }

    function toString(uint256 value) internal pure returns (string memory) {
        return Strings.toString(value);
    }
}
```

It is as simple as building a string.
The contract deployed can be seen [here](https://ropsten.etherscan.io/address/0x9bb98c8ee130831ac4df4634c384f97297227bc8).

## Remote invocation of tokenAsSVG
The contract address is [0x9bb98c8ee130831ac4df4634c384f97297227bc8](https://ropsten.etherscan.io/address/0x9bb98c8ee130831ac4df4634c384f97297227bc8).  
I am using [Ethers](https://docs.ethers.io/) and [Node JS](https://nodejs.org/):

```javascript
const ethers = require('ethers');
const fs = require('fs')

// The Contract interface 'tokenAsSVG'
let abi = [
    "function tokenAsSVG() external view returns (string memory svg)"
];

// Connect to the network
let provider = ethers.getDefaultProvider('ropsten');

// The address from the above deployment example
let contractAddress = "0x9bb98c8ee130831ac4df4634c384f97297227bc8";

// We connect to the Contract using a Provider, so we will only
// have read-only access to the Contract
let contract = new ethers.Contract(contractAddress, abi, provider);
console.log("contract {}", contract);

async function get_svg() {
    let currentValue = await contract.tokenAsSVG();
    return currentValue;
}

console.log("Getting SVG...");
const svg_promise = get_svg();
console.log("Current contract value {}", svg_promise);

svg_promise.then(function(result) {
    console.log("result {}", result);
    fs.writeFile("real-nft.svg", result, function(err) {
        if(err) {
            return console.log(err);
        }
        console.log("The file was saved!");
    }); 
});
```

The file 'real-nft.svg' will be stored locally.
