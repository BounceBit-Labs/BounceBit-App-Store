# App Name
PicWe

## App Description
PicWe is An Omni-chain Decentralized Securities Trader Infra.Here`s the Highlights:
1.Create Public Goods for Massive Adoption by State Identification Service(SIS) which is the New Infrastructure for Layer 3. With SIS we can Make Builder Easier and Make Assets Safe.
2.Aggregate Trading with the Cex&Dex.
3.With our Starlight (AA) wallet users can easily enter the blockchain world.

### App Icon
[![2024-03-27-165141.png](https://i.postimg.cc/Ssd09ghD/2024-03-27-165141.png)](https://postimg.cc/WFdfL6tk)

#### App Link
[website](https://app.picwe.org/)

##### Smart Contract Source Code
```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import "@openzeppelin/contracts-upgradeable@5.0.1/utils/PausableUpgradeable.sol";
import "@openzeppelin/contracts-upgradeable@5.0.1/access/OwnableUpgradeable.sol";
import "@openzeppelin/contracts-upgradeable@5.0.1/proxy/utils/Initializable.sol";
import "@openzeppelin/contracts-upgradeable@5.0.1/proxy/utils/UUPSUpgradeable.sol";
import "@openzeppelin/contracts-upgradeable@5.0.1/utils/ContextUpgradeable.sol";
import "@openzeppelin/contracts@5.0.1/token/ERC20/IERC20.sol";
import "@openzeppelin/contracts@5.0.1/utils/cryptography/ECDSA.sol";

/**
 * @dev PicWe_StateIdentification contract.
 *      This contract is used for state identification in PicWe.
 */
contract PicWe_StateIdentification is Initializable, ContextUpgradeable, PausableUpgradeable, OwnableUpgradeable, UUPSUpgradeable{
    using ECDSA for bytes32;

    address public admin;
    IERC20 public pointToken;
    mapping(address => uint256) public balances;
    mapping(bytes32 => bool) public usedNonces;
    mapping(uint256 => mapping(address => bool)) public votes; 
    mapping(uint256 => uint256) public voteWeights;  
    mapping(uint256 => uint256) public totalWeights;  
    mapping(uint256 => address) private voteToParticipant;
    uint256 public nextVoteId;
    uint256 public reserveBalance;

    event VoteStarted(uint256 voteId, address indexed participant);
    event Voted(uint256 voteId, address indexed voter, uint256 weight);
    event AdminChanged(address indexed newAdmin);
    event ChannelOpened(address indexed participant, uint256 amount);
    event ChannelStateUpdated(address indexed participant, uint256 newBalance);
    event ChannelClosed(address indexed participant, uint256 amount);
    event ChannelClosedByDAO(address indexed participant, uint256 amount, uint256 voteId);
    event BalancesBatchUpdatedbyadmin(address[] participants, uint256[] newBalances);
    event BalancesBatchUpdatedbyuser(address[] participants, uint256[] newBalances);
    event DepositedToReserve(uint256 amount);
    event WithdrawnFromReserve(uint256 amount);

    /**
     * @dev Constructor for PicWe_StateIdentification contract.
     *      This is necessary for contract initialization.
     */
    constructor() {
        _disableInitializers();
    }

    /**
     * @dev Initialize the contract.
     * @param admin_ - The address of the admin
     * @param pointToken_ - The address of the point token
     * @param initialReserveBalance - The initial reserve balance
     */
    function initialize(address admin_, IERC20 pointToken_, uint256 initialReserveBalance) public initializer {
        __Context_init();
        __Pausable_init();
        __Ownable_init(admin_);
        __UUPSUpgradeable_init();

        admin = admin_;
        pointToken = pointToken_;
        nextVoteId = 1;
        reserveBalance = initialReserveBalance;
    }

    /**
     * @dev Pause the contract.
     *      Only the owner can call this function.
     */
    function pause() public onlyOwner {
        _pause();
    }

    /**
     * @dev Unpause the contract.
     *      Only the owner can call this function.
     */
    function unpause() public onlyOwner {
        _unpause();
    }

    /**
     * @dev Authorize the upgrade.
     *      Only the owner can call this function.
     * @param newImplementation - The address of the new implementation
     */
    function _authorizeUpgrade(address newImplementation)
        internal
        onlyOwner
        override
    {}
    
    /**
     * @dev Deposit to reserveBalance.
     *      Only the admin can call this function.
     * @param amount - The amount to deposit
     */
    function depositToReserve(uint256 amount) external onlyAdmin whenNotPaused {
        require(pointToken.transferFrom(_msgSender(), address(this), amount), "Transfer failed");
        reserveBalance += amount;
        emit DepositedToReserve(amount);
    }

    /**
     * @dev Withdraw from reserveBalance.
     *      Only the admin can call this function.
     * @param amount - The amount to withdraw
     */
    function withdrawFromReserve(uint256 amount) external onlyAdmin whenNotPaused {
        require(reserveBalance >= amount, "Insufficient reserve balance");
        require(pointToken.transfer(_msgSender(), amount), "Transfer failed");
        reserveBalance -= amount;
        emit WithdrawnFromReserve(amount);
    }
 
    /**
     * @dev Modifier to make a function callable only by the admin.
     */
    modifier onlyAdmin() {
        require(_msgSender() == admin, "Only admin can perform this action");
        _;
    }

    /**
     * @dev Change the admin.
     *      Only the owner can call this function.
     * @param newAdmin - The address of the new admin
     */
    function changeAdmin(address newAdmin) external onlyOwner whenNotPaused {
        require(newAdmin != address(0), "New admin address cannot be zero address");
        admin = newAdmin;
        emit AdminChanged(newAdmin);
    }

    /**
     * @dev Open a channel.
     * @param amount - The amount to open the channel with
     */
    function openChannel(uint256 amount) external whenNotPaused {
        require(pointToken.transferFrom(_msgSender(), address(this), amount), "Transfer failed");
        balances[_msgSender()] += amount;
        emit ChannelOpened(_msgSender(), amount);
    }

    /**
     * @dev Update the balance of a participant.
     * @param participant - The address of the participant
     * @param newBalance - The new balance of the participant
     */
    function _updateBalance(address participant, uint256 newBalance) internal {
        if (balances[participant] < newBalance) {
            uint256 shortfall = newBalance - balances[participant];
            require(reserveBalance >= shortfall, "Not enough reserve balance");
            reserveBalance -= shortfall;
            balances[participant] += shortfall;
        } else if (balances[participant] > newBalance) {
            uint256 excess = balances[participant] - newBalance;
            reserveBalance += excess;
            balances[participant] -= excess;
        }
    }

    /**
     * @dev Verify the batch signature.
     * @param participants - The addresses of the participants
     * @param newBalances - The new balances of the participants
     * @param nonce - The nonce
     * @param adminSignature - The signature of the admin
     */
    function _verifyBatchSignature(address[] calldata participants, uint256[] calldata newBalances, uint256 nonce, bytes calldata adminSignature) internal {
        bytes32 hash = keccak256(abi.encodePacked(participants, newBalances, nonce));
        require(admin == hash.recover(adminSignature), "Invalid signature");
        require(!usedNonces[hash], "Nonce already used");
        usedNonces[hash] = true;
    }

    /**
     * @dev Verify the signature.
     * @param participant - The address of the participant
     * @param newBalance - The new balance of the participant
     * @param nonce - The nonce
     * @param adminSignature - The signature of the admin
     */
    function _verifySignature(address participant, uint256 newBalance, uint256 nonce, bytes calldata adminSignature) internal {
        bytes32 hash = keccak256(abi.encodePacked(participant, newBalance, nonce));
        require(admin == hash.recover(adminSignature), "Invalid signature");
        require(!usedNonces[hash], "Nonce already used");
        usedNonces[hash] = true;
    }

    /**
     * @dev Update the channel state.
     * @param newBalance - The new balance of the channel
     * @param channel_nonce - The nonce of the channel
     * @param adminSignature - The signature of the admin
     */
    function updateChannelState(uint256 newBalance, uint256 channel_nonce, bytes calldata adminSignature) external whenNotPaused {
        _verifySignature(_msgSender(), newBalance, channel_nonce, adminSignature);

        _updateBalance(_msgSender(), newBalance);

        emit ChannelStateUpdated(_msgSender(), newBalance);
    }
    
    /**
     * @dev Batch update balances.
     * @param participants - The addresses of the participants
     * @param newBalances - The new balances of the participants
     * @param close_nonce - The nonce
     * @param adminSignature - The signature of the admin
     */
    function batchUpdateBalances(address[] calldata participants, uint256[] calldata newBalances, uint256 close_nonce, bytes calldata adminSignature) external whenNotPaused {
        require(participants.length == newBalances.length, "Participants and newBalances length must match");
        _verifyBatchSignature(participants, newBalances, close_nonce, adminSignature);

        for (uint i = 0; i < participants.length; i++) {
            _updateBalance(participants[i], newBalances[i]);
        }
        emit BalancesBatchUpdatedbyuser(participants, newBalances);
    }
    
    /**
     * @dev Close a channel.
     * @param newBalance - The new balance of the channel
     * @param close_nonce - The nonce
     * @param adminSignature - The signature of the admin
     */
    function closeChannel(uint256 newBalance, uint256 close_nonce, bytes calldata adminSignature) external whenNotPaused {
        _verifySignature(_msgSender(), newBalance, close_nonce, adminSignature);

        _updateBalance(_msgSender(), newBalance);

        require(pointToken.transfer(_msgSender(), newBalance), "Transfer failed");

        balances[_msgSender()] = 0;

        emit ChannelClosed(_msgSender(), newBalance);
    }

    /**
     * @dev Vote.
     * @param voteId - The ID of the vote
     * @param approve - Whether to approve the vote
     */
    function vote(uint256 voteId, bool approve) external whenNotPaused {
        require(balances[_msgSender()] > 0, "No balance to vote");
        require(!votes[voteId][_msgSender()], "Already voted");

        votes[voteId][_msgSender()] = true;
        uint256 weight = balances[_msgSender()];
        if (approve) {
            voteWeights[voteId] += weight;
        }

        emit Voted(voteId, _msgSender(), weight);
    }

    /**
     * @dev Start a vote to close a channel.
     * @param participant - The address of the participant
     */
    function startVoteToCloseChannel(address participant) external onlyAdmin whenNotPaused {
        require(balances[participant] > 0, "No balance to dispute");

        uint256 voteId = nextVoteId++;
        totalWeights[voteId] = pointToken.balanceOf(address(this));
        voteToParticipant[voteId] = participant; // 记录投票对应的参与者
        emit VoteStarted(voteId, participant);
    }


    /**
     * @dev Close a channel by DAO.
     * @param participant - The address of the participant
     * @param newBalance - The new balance of the participant
     * @param voteId - The ID of the vote
     */
    function closeChannelByDAO(address participant, uint256 newBalance, uint256 voteId) external onlyAdmin whenNotPaused {
        require(voteWeights[voteId] * 2 >= totalWeights[voteId], "Not enough votes to close channel");
        require(voteToParticipant[voteId] == participant, "Vote ID does not match participant"); // 确保投票ID对应于指定参与者

        _updateBalance(participant, newBalance);
        require(pointToken.transfer(participant, newBalance), "Transfer failed");
        balances[participant] = 0;

        emit ChannelClosedByDAO(participant, newBalance, voteId);
    }

    /**
     * @dev Batch update channel by admin.
     * @param participants - The addresses of the participants
     * @param newBalances - The new balances of the participants
     */
    function batchUpdateChannelbyadmin(address[] calldata participants, uint256[] calldata newBalances) external onlyAdmin whenNotPaused {
        require(participants.length == newBalances.length, "Array lengths must match");

        for (uint i = 0; i < participants.length; i++) {
            _updateBalance(participants[i], newBalances[i]);
        }
        emit BalancesBatchUpdatedbyadmin(participants, newBalances);
    }
}
