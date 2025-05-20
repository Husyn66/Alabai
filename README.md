// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import "@openzeppelin/contracts/token/ERC20/ERC20.sol";
import "@openzeppelin/contracts/token/ERC20/extensions/ERC20Burnable.sol";
import "@openzeppelin/contracts/token/ERC20/extensions/ERC20Pausable.sol";
import "@openzeppelin/contracts/access/AccessControl.sol";
import "@openzeppelin/contracts/security/ReentrancyGuard.sol";

contract AlabaiToken is ERC20, ERC20Burnable, ERC20Pausable, AccessControl, ReentrancyGuard {
    bytes32 public constant PAUSER_ROLE = keccak256("PAUSER_ROLE");
    bytes32 public constant MINTER_ROLE = keccak256("MINTER_ROLE");
    bytes32 public constant BLACKLISTER_ROLE = keccak256("BLACKLISTER_ROLE");
    
    uint256 public immutable MAX_SUPPLY = 1_000_000_000 * 10**18;
    uint256 public maxTransferAmount = MAX_SUPPLY / 100;
    bool public maxTransferAmountEnabled = true;
    
    mapping(address => bool) private _blacklisted;
    mapping(address => uint256) private _lastTransferTimestamp;
    uint256 public cooldownTime = 30;
    bool public cooldownEnabled = true;
    
    uint256 public feeBasisPoints = 0;
    address public feeCollector;
    bool public feeEnabled = false;
    
    event BlacklistUpdated(address indexed account, bool blacklisted);
    event MaxTransferAmountUpdated(uint256 amount);
    event CooldownTimeUpdated(uint256 time);
    event FeeUpdated(uint256 feeBasisPoints);
    event FeeCollectorUpdated(address feeCollector);
    event MaxTransferAmountToggled(bool enabled);
    event CooldownToggled(bool enabled);
    event FeeToggled(bool enabled);

    constructor() ERC20("Alabai", "ABI") {
        _grantRole(DEFAULT_ADMIN_ROLE, msg.sender);
        _grantRole(PAUSER_ROLE, msg.sender);
        _grantRole(MINTER_ROLE, msg.sender);
        _grantRole(BLACKLISTER_ROLE, msg.sender);
        feeCollector = msg.sender;
        _mint(msg.sender, 500_000_000 * 10**18);
    }

    function _update(address from, address to, uint256 amount)
        internal
        virtual
        override(ERC20, ERC20Pausable)
    {
        // Apply blacklist check
        if (from != address(0)) {
            require(!_blacklisted[from], "AlabaiToken: sender is blacklisted");
        }
        
        if (to != address(0)) {
            require(!_blacklisted[to], "AlabaiToken: recipient is blacklisted");
        }

        // Apply max transfer limit
        if (from != address(0) && to != address(0)) {
            if (maxTransferAmountEnabled && !hasRole(DEFAULT_ADMIN_ROLE, from) && !hasRole(DEFAULT_ADMIN_ROLE, to)) {
                require(amount <= maxTransferAmount, "AlabaiToken: transfer amount exceeds maximum");
            }

            // Apply cooldown
            if (cooldownEnabled && !hasRole(DEFAULT_ADMIN_ROLE, from) && !hasRole(DEFAULT_ADMIN_ROLE, to)) {
                require(_lastTransferTimestamp[from] + cooldownTime <= block.timestamp, "AlabaiToken: cooldown active");
                _lastTransferTimestamp[from] = block.timestamp;
            }
        }

        // Apply fee if enabled
        if (from != address(0) && to != address(0) && feeEnabled && feeBasisPoints > 0) {
            require(feeCollector != address(0), "AlabaiToken: fee collector not set");
            uint256 feeAmount = amount * feeBasisPoints / 10000;
            uint256 transferAmount = amount - feeAmount;
            
            // Call parent implementation for fee transfer
            super._update(from, feeCollector, feeAmount);
            
            // Call parent for remaining amount
            if (transferAmount > 0) {
                super._update(from, to, transferAmount);
            }
        } else {
            // Standard transfer without fee
            super._update(from, to, amount);
        }
    }

    function mint(address to, uint256 amount) public onlyRole(MINTER_ROLE) {
        require(to != address(0), "AlabaiToken: mint to zero address");
        require(totalSupply() + amount <= MAX_SUPPLY, "AlabaiToken: exceeds maximum supply");
        _mint(to, amount);
    }

    function pause() public onlyRole(PAUSER_ROLE) {
        _pause();
    }

    function unpause() public onlyRole(PAUSER_ROLE) {
        _unpause();
    }

    function updateBlacklist(address account, bool blacklisted) public onlyRole(BLACKLISTER_ROLE) {
        _blacklisted[account] = blacklisted;
        emit BlacklistUpdated(account, blacklisted);
    }

    function isBlacklisted(address account) public view returns (bool) {
        return _blacklisted[account];
    }

    function setMaxTransferAmount(uint256 amount) public onlyRole(DEFAULT_ADMIN_ROLE) {
        maxTransferAmount = amount;
        emit MaxTransferAmountUpdated(amount);
    }

    function setMaxTransferAmountEnabled(bool enabled) public onlyRole(DEFAULT_ADMIN_ROLE) {
        maxTransferAmountEnabled = enabled;
        emit MaxTransferAmountToggled(enabled);
    }

    function setCooldownTime(uint256 time) public onlyRole(DEFAULT_ADMIN_ROLE) {
        cooldownTime = time;
        emit CooldownTimeUpdated(time);
    }

    function setCooldownEnabled(bool enabled) public onlyRole(DEFAULT_ADMIN_ROLE) {
        cooldownEnabled = enabled;
        emit CooldownToggled(enabled);
    }

    function setFee(uint256 _feeBasisPoints) public onlyRole(DEFAULT_ADMIN_ROLE) {
        require(_feeBasisPoints <= 1000, "AlabaiToken: fee cannot exceed 10%");
        feeBasisPoints = _feeBasisPoints;
        emit FeeUpdated(_feeBasisPoints);
    }

    function setFeeCollector(address _feeCollector) public onlyRole(DEFAULT_ADMIN_ROLE) {
        require(_feeCollector != address(0), "AlabaiToken: fee collector cannot be zero address");
        feeCollector = _feeCollector;
        emit FeeCollectorUpdated(_feeCollector);
    }

    function setFeeEnabled(bool enabled) public onlyRole(DEFAULT_ADMIN_ROLE) {
        feeEnabled = enabled;
        emit FeeToggled(enabled);
    }
}

