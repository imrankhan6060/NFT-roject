   

// SPDX-License-Identifier: MIT
pragma solidity ^0.8.9;

import "@openzeppelin/contracts/token/ERC721/ERC721.sol";
import "@openzeppelin/contracts/token/ERC721/extensions/ERC721Enumerable.sol";
import "@openzeppelin/contracts/token/ERC721/extensions/ERC721URIStorage.sol";
import "@openzeppelin/contracts/security/Pausable.sol";
import "@openzeppelin/contracts/access/Ownable.sol";
import "@openzeppelin/contracts/utils/Counters.sol";

contract NFTFriends is ERC721, ERC721Enumerable, ERC721URIStorage, Pausable, Ownable {
    using Counters for Counters.Counter;
  
    uint public maxSupply; 
    uint public maxMintLimint; // address limit
    uint public platformLimit;
    uint public userLimit;
    uint public allowlistLimit;
    bool allowlistmintopen = false;
    bool publicMintOpen = false;
    uint256 currentPhase;


    struct premiumUser{
        address userAdd;
        bool isRegister;
        
    }
    struct phase{
        uint256 reversedLimit;
        bool isActive;
        uint premiumLimit;
        uint normalLimit;
        mapping (address => uint )premiumUserBalance;
        mapping (address => uint )normalUserBalance;
    }

    mapping (uint => phase) public phaseMapping;
    mapping (address => premiumUser) public allowlistMapping;
    mapping (address => uint ) public addressTokenCount;
    mapping (address => bool) public adminAddresses; // Mapping to store admin addresses
    mapping(address => uint256[]) private nftsByAddress;



 Counters.Counter private _tokenIdCounter;
    
    constructor(uint _maxSupply, uint _platformLimit) ERC721("NFTFriends", "NFTF") {
        maxSupply = _maxSupply;
        platformLimit= _platformLimit;
        userLimit = _maxSupply - _platformLimit;
    }
    
    function pause() public onlyOwner {
        _pause();
    }

    function unpause() public onlyOwner {
        _unpause();
    }

function createPhase(uint _phaseReservedLi, uint permiumReservedLi, uint normalReservedLi ) public {
    require(phaseMapping[currentPhase].isActive ==false, "phase is already active");
    require(_phaseReservedLi < userLimit, "user limit exceed");
    require(phaseMapping[currentPhase].reversedLimit ==0 ,"phase already existed");
    phaseMapping[currentPhase].reversedLimit = _phaseReservedLi; 
    phaseMapping[currentPhase].premiumLimit = permiumReservedLi;
    phaseMapping[currentPhase].normalLimit = normalReservedLi; 

    }


    function activePhase()public onlyOwner{
      require(phaseMapping[currentPhase].isActive = true, "already active");
      require(phaseMapping[currentPhase].premiumLimit !=0, "phase not created");
      phaseMapping[currentPhase].isActive = true;
    
    }
    function deActivePhase()public  onlyOwner{
        require(phaseMapping[currentPhase].isActive == true, "phase not active");
      require(phaseMapping[currentPhase].premiumLimit !=0, "phase not created");
       phaseMapping[currentPhase].isActive = true;
       currentPhase++;
       
    }

    //modify the mint window
    function editMintWindow(bool _allowlistmintopen, bool _publicMintOpen )external onlyOwner{
        publicMintOpen = _publicMintOpen;
        allowlistmintopen = _allowlistmintopen;

    }



   function safeMint(string memory uri) public payable {
       uint256 tokenId = _tokenIdCounter.current();
       require(phaseMapping[currentPhase].isActive == true, "phase not active");
       require(maxMintLimint > 0, "address limit exceed ");
       
       if(allowlistMapping[msg.sender].isRegister){

             require(addressTokenCount[msg.sender] < maxMintLimint, "Address minted max NFT");
             require(phaseMapping[currentPhase].premiumLimit > phaseMapping[currentPhase].premiumUserBalance[msg.sender], "phase user limit exceed");
             phaseMapping[currentPhase].premiumUserBalance[msg.sender]++;
       }
       else{
         //  require(balanceOf(msg.sender) < normalUserBalance[msg.sender].maxMintLimint, "Address minted max NFT" );
       require(phaseMapping[currentPhase].premiumLimit > phaseMapping[currentPhase].normalUserBalance[msg.sender], "phase user limit exceed");
       phaseMapping[currentPhase].normalUserBalance[msg.sender]++;
       }
          internalMint();
        _setTokenURI(tokenId, uri);
        userLimit--;

   
   
   }
    






  
 //populate the allowlist or premium member
   function setAllowlist(address[] calldata addresses) external onlyOwner {
    for (uint256 i = 0; i < addresses.length; i++) {
         allowlistMapping[addresses[i]] = premiumUser({
            userAdd: addresses[i],
            isRegister: true
        });
    }
}

    
  







     // Add admin addresses who can mint up to the platform limit
    function addAdminAddresses(address[] calldata adminAddressesList) external onlyOwner {
        for (uint256 i = 0; i < adminAddressesList.length; i++) {
            adminAddresses[adminAddressesList[i]] = true;
        }
    }

    //function adminMint(string memory uri) public {
      //  require(adminAddresses[msg.sender], "Only admin addresses can call this function");
        //require(addressTokenCount[msg.sender] < platformLimit, "Admin address has reached the platform minting limit");

//        uint256 tokenId = _tokenIdCounter.current();
  //      _tokenIdCounter.increment();
    //    _safeMint(msg.sender, tokenId);
      //  _setTokenURI(tokenId, uri);
        //addressTokenCount[msg.sender]++;
   // }
    // Admin minting function for platform limit and bulk minting    // Admin minting function for platform limit and bulk minting
    function adminMint(uint256 numNFTs) public {
        require(adminAddresses[msg.sender], "Only admin addresses can call this function");
        require(numNFTs <= platformLimit, "Exceeded platform mint limit");
        require(addressTokenCount[msg.sender] + numNFTs <= platformLimit, "Exceeded platform mint limit for an address");

        for (uint256 i = 0; i < numNFTs; i++) {
            uint256 tokenId = _tokenIdCounter.current();
            require(totalSupply() < maxSupply, "Sold out");

            _tokenIdCounter.increment();
            _safeMint(msg.sender, tokenId);
            _setTokenURI(tokenId,"");
            addressTokenCount[msg.sender]++;
        }
    }

   function bulkMint(uint256 numNFTs, string[] memory uris) public {
    require(numNFTs <= userLimit, "Exceeded user mint limit");
    require(addressTokenCount[msg.sender] + numNFTs <= maxMintLimint, "Exceeded user mint limit for an address");

    uint256 tokenId = _tokenIdCounter.current();
    for (uint256 i = 0; i < numNFTs; i++) {
        internalMint();
        _setTokenURI(tokenId + i, uris[i]);
        userLimit--;
    }
}






  // globel user mint limint
    function mintLimit (uint limit)public onlyOwner{
         maxMintLimint = limit;
    }

    // fetch nft by address 

       function getNFTsByAddress(address _owner) public view returns (uint256[] memory) {
        return nftsByAddress[_owner];
    }

    //update hash in bulk
  /*  function updateHashes(bulkft[] memory dataArray){
        for (i=0; i< dataArray.length;i++){
            if(ownerOf(dataArray[i].id)==msg.sender){
                _setTokenURI(dataArray[i].id, dataArray[i].uri);
            }
        }
    }
*/
      function internalMint()internal {
        uint256 tokenId = _tokenIdCounter.current();
         require(totalSupply() < maxSupply,"sold out");
        _tokenIdCounter.increment();
        _safeMint(msg.sender, tokenId);
        addressTokenCount[msg.sender]++;
         nftsByAddress[msg.sender].push(tokenId);
        
    }
   
    







    //withdraw
        function withdrow(address _addr)external onlyOwner{
        uint256 balances = address(this).balance;
        payable (_addr).transfer(balances);
    }

    function _beforeTokenTransfer(address from, address to, uint256 tokenId, uint256 batchSize)
        internal
        whenNotPaused
        override(ERC721, ERC721Enumerable)
    {
        super._beforeTokenTransfer(from, to, tokenId, batchSize);
    }

    // The following functions are overrides required by Solidity.

    function _burn(uint256 tokenId) internal override(ERC721, ERC721URIStorage) {
        super._burn(tokenId);
    }

    function tokenURI(uint256 tokenId)
        public
        view
        override(ERC721, ERC721URIStorage)
        returns (string memory)
    {
        return super.tokenURI(tokenId);
    }

    function supportsInterface(bytes4 interfaceId)
        public
        view
        override(ERC721, ERC721Enumerable, ERC721URIStorage)
        returns (bool)
    {
        return super.supportsInterface(interfaceId);
    }
}
