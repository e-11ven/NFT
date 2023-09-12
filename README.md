ERC721 unrevealed with timestamp


// SPDX-License-Identifier: MIT

pragma solidity >=0.7.0 <0.9.0;

import "t@openzeppelin/contracts/token/ERC721/extensions/ERC721Enumerable.sol";
import "@openzeppelin/contracts/access/Ownable.sol";
import "@openzeppelin/contracts/utils/Strings.sol";

contract Hude is ERC721Enumerable, Ownable {
    using Strings for uint256;

    string public baseURI;
    string public baseExtension = ".json";
    uint256 public cost = 0.01 ether;
    uint256 public maxSupply = 111;
    uint256 public maxMintAmount = 2;
    bool public paused = true;
    bool public revealed = false;
    string public notRevealedUri;
    uint256 public mintWindowOpens = 1692369240;
    uint256 public mintWindowCloses = 1692425853;

    constructor(
        string memory _name,
        string memory _symbol,
        string memory _initBaseURI,
        string memory _initNotRevealedUri
    ) ERC721(_name, _symbol) {
        setBaseURI(_initBaseURI);
        setNotRevealedURI(_initNotRevealedUri);
    }

    function _baseURI() internal view virtual override returns (string memory) {
        return baseURI;
    }

    function mint(uint256 _mintAmount) public payable {
        uint256 supply = totalSupply();
        require(!paused, "Minting is paused");
        require(
            _mintAmount > 0 && _mintAmount <= maxMintAmount,
            "Invalid mint amount"
        );
        require(supply + _mintAmount <= maxSupply, "Exceeds max supply");
        require(
            block.timestamp >= mintWindowOpens &&
                block.timestamp <= mintWindowCloses,
            "Outside minting window"
        );

        if (msg.sender != owner()) {
            require(msg.value >= cost * _mintAmount, "Insufficient payment");
        }

        for (uint256 i = 1; i <= _mintAmount; i++) {
            _safeMint(msg.sender, supply + i);
        }
    }

    function walletofOwner(address _owner)
        public
        view
        returns (uint256[] memory)
    {
        uint256 ownerTokenCount = balanceOf(_owner);
        uint256[] memory tokenIds = new uint256[](ownerTokenCount);
        for (uint256 i = 0; i < ownerTokenCount; i++) {
            tokenIds[i] = tokenOfOwnerByIndex(_owner, i);
        }
        return tokenIds;
    }

    function tokenURI(uint256 tokenId)
        public
        view
        virtual
        override
        returns (string memory)
    {
        require(_exists(tokenId), "Token does not exist");

        if (!revealed) {
            return notRevealedUri;
        }

        string memory currentBaseURI = _baseURI();
        return
            bytes(currentBaseURI).length > 0
                ? string(
                    abi.encodePacked(
                        currentBaseURI,
                        tokenId.toString(),
                        baseExtension
                    )
                )
                : "";
    }

    function reveal() public onlyOwner {
        revealed = true;
    }

    function setMaxMintAmount(uint256 _newMaxMintAmount) public onlyOwner {
        maxMintAmount = _newMaxMintAmount;
    }

    function setNotRevealedURI(string memory _newNotRevealedURI)
        public
        onlyOwner
    {
        notRevealedUri = _newNotRevealedURI;
    }

    function setBaseURI(string memory _newBaseURI) public onlyOwner {
        baseURI = _newBaseURI;
    }

    function setBaseExtension(string memory _newBaseExtension)
        public
        onlyOwner
    {
        baseExtension = _newBaseExtension;
    }

    function pause(bool _state) public onlyOwner {
        paused = _state;
    }

    function withdraw() public payable onlyOwner {
        uint256 contractBalance = address(this).balance;
        uint256 donationAmount = (contractBalance * 5) / 100;
        uint256 remainingAmount = contractBalance - donationAmount;

        require(
            payable(0xD81116096A86FeE95dA2aAc5646A66EA7d9b9e10).send(
                donationAmount
            ),
            "Donation transfer failed"
        );
        require(
            payable(owner()).send(remainingAmount),
            "Withdrawal transfer failed"
        );
    }

    receive() external payable {}
}
