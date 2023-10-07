# [Stader Labs Report](https://code4rena.com/reports/2023-06-stader)

## Findings by Josephdara
| Severity | Title | Count |
|:--:|:---|:--:|
| [M-01](#m-01-Owner-in-VaultProxy-is-address(0)) | Owner in VaultProxy is address(0)| M-01 |
| [M-02](#m-02-Pausability-unimplemented)| Pausability unimplemented | M-02 |


## [M-01] Owner in VaultProxy is address(0)

## Impact and Details

https://github.com/code-423n4/2023-06-stader/blob/7566b5a35f32ebd55d3578b8bd05c038feb7d9cc/contracts/VaultProxy.sol#L20-L35
The owner variable in the ```VaultProxy.sol``` is going to be zero because of a bug in staderConfig.
The ```intialize``` function in ```staderConfig``` grants th admin role to an admin passed in via the native ```_grantRole()```, but does not update the mapping that ```getAdmin``` reads from.
```solidity
function initialise(
    bool _isValidatorWithdrawalVault,
    uint8 _poolId,
    uint256 _id,
    address _staderConfig
) external {
    if (isInitialized) {
        revert AlreadyInitialized();
    }
    UtilLib.checkNonZeroAddress(_staderConfig);
    isValidatorWithdrawalVault = _isValidatorWithdrawalVault;
    isInitialized = true;
    poolId = _poolId;
    id = _id;
    staderConfig = IStaderConfig(_staderConfig);

    //@audit-issue Admin is zero on initialize
    //address(0) is going to  be returned, wrt stader.initialize
    owner = staderConfig.getAdmin();
}
```
https://github.com/code-423n4/2023-06-stader/blob/7566b5a35f32ebd55d3578b8bd05c038feb7d9cc/contracts/StaderConfig.sol#L85-L103
https://github.com/code-423n4/2023-06-stader/blob/7566b5a35f32ebd55d3578b8bd05c038feb7d9cc/contracts/StaderConfig.sol#L361-L363

The intialization of the vaults would not revert and all functions restricted to the owner would be inaccessible unless function updateAdmin is called.
https://github.com/code-423n4/2023-06-stader/blob/7566b5a35f32ebd55d3578b8bd05c038feb7d9cc/contracts/StaderConfig.sol#L176-L183

The update admin does not revoke functions for the current admin in the AccessControl contract too because the address returned is address zero
### Proof of Concept
```solidity

pragma solidity 0.8.16;

import '../../contracts/library/UtilLib.sol';

import '../../contracts/StaderConfig.sol';
import '../../contracts/VaultProxy.sol';
import '../../contracts/ValidatorWithdrawalVault.sol';
import '../../contracts/OperatorRewardsCollector.sol';

import './mocks/PoolUtilsMock.sol';
import './mocks/PenaltyMockForVault.sol';
import './mocks/SDCollateralMock.sol';
import './mocks/StakePoolManagerMock.sol';

import 'forge-std/Test.sol';
import '@openzeppelin/contracts/proxy/transparent/TransparentUpgradeableProxy.sol';
import '@openzeppelin/contracts/proxy/transparent/ProxyAdmin.sol';

contract ValidatorWithdrawalVaultTest is Test {
address staderAdmin;
address staderManager;
address staderTreasury;
PoolUtilsMock poolUtils;

uint8 poolId;
uint256 validatorId;

StaderConfig staderConfig;
VaultProxy withdrawVault;
OperatorRewardsCollector operatorRC;

function setUp() public {
    poolId = 1;
    validatorId = 1;

    staderAdmin = vm.addr(100);
    staderManager = vm.addr(101);
    address ethDepositAddr = vm.addr(102);

    ProxyAdmin proxyAdmin = new ProxyAdmin();

    StaderConfig configImpl = new StaderConfig();
    TransparentUpgradeableProxy configProxy = new TransparentUpgradeableProxy(
        address(configImpl),
        address(proxyAdmin),
        ''
    );
    staderConfig = StaderConfig(address(configProxy));
    staderConfig.initialize(staderAdmin, ethDepositAddr);

    OperatorRewardsCollector operatorRCImpl = new OperatorRewardsCollector();
    TransparentUpgradeableProxy operatorRCProxy = new TransparentUpgradeableProxy(
        address(operatorRCImpl),
        address(proxyAdmin),
        ''
    );
    operatorRC = OperatorRewardsCollector(address(operatorRCProxy));
    operatorRC.initialize(staderAdmin, address(staderConfig));

    poolUtils = new PoolUtilsMock(address(staderConfig));
    PenaltyMockForVault penaltyContract = new PenaltyMockForVault();
    SDCollateralMock sdCollateral = new SDCollateralMock();
    ValidatorWithdrawalVault withdrawVaultImpl = new ValidatorWithdrawalVault();

    vm.startPrank(staderAdmin);
    staderConfig.updatePoolUtils(address(poolUtils));
    staderConfig.updatePenaltyContract(address(penaltyContract));
    staderConfig.updateSDCollateral(address(sdCollateral));
    staderConfig.updateOperatorRewardsCollector(address(operatorRC));
    staderConfig.updateValidatorWithdrawalVaultImplementation(address(withdrawVaultImpl));

    vm.stopPrank();

    withdrawVault = new VaultProxy();
    withdrawVault.initialise(true, poolId, validatorId, address(staderConfig));
}

function testAdminRights() public {
   vm.startPrank(staderAdmin);
  staderConfig.grantRole(staderConfig.MANAGER(), staderManager);
  vm.stopPrank();
}
function testFetchAdminAndVaultOwner() public{
    address _admin = staderConfig.getAdmin();
    assertEq(_admin, address(0));
    address _owner = withdrawVault.owner();
    assertEq(_owner, address(0));
}

}
```

## Tools Used
Foundry, manual review

## Recommended Mitigation Steps
```StaderConfig.initialize``` should add ```setAccount(ADMIN, _admin);``` a last line




## [M-02] Pausability unimplemented 
## Impact and Details
https://github.com/code-423n4/2023-06-stader/blob/7566b5a35f32ebd55d3578b8bd05c038feb7d9cc/contracts/SocializingPool.sol#L1-L228


The SocializingPool.sol imports and inherits the Openzeppelin Pausable contract
```solidity
import '@openzeppelin/contracts-upgradeable/access/AccessControlUpgradeable.sol';
import '@openzeppelin/contracts-upgradeable/security/PausableUpgradeable.sol';
import '@openzeppelin/contracts-upgradeable/security/ReentrancyGuardUpgradeable.sol';
import '@openzeppelin/contracts-upgradeable/utils/cryptography/MerkleProofUpgradeable.sol';
import '@openzeppelin/contracts/token/ERC20/IERC20.sol';

contract SocializingPool is
ISocializingPool,
Initializable,
AccessControlUpgradeable,
PausableUpgradeable,
ReentrancyGuardUpgradeable
{
```

But there is no function in the contract that implements the pausabiility functionality. Even though some functions implement the modifier.
```solidity
function claim(
    uint256[] calldata _index,
    uint256[] calldata _amountSD,
    uint256[] calldata _amountETH,
    bytes32[][] calldata _merkleProof
) external override nonReentrant whenNotPaused {
```
Therefore this contract cannot be paused
## Proof of Concept
Below is a snapshot of the PausableUpgradeable, and it implements no public/external pause and unpause functions

```solidity
abstract contract PausableUpgradeable is Initializable, ContextUpgradeable {
/**
 * @dev Emitted when the pause is triggered by `account`.
 */
event Paused(address account);

/**
 * @dev Emitted when the pause is lifted by `account`.
 */
event Unpaused(address account);

bool private _paused;

/**
 * @dev Initializes the contract in unpaused state.
 */
function __Pausable_init() internal onlyInitializing {
    __Pausable_init_unchained();
}

function __Pausable_init_unchained() internal onlyInitializing {
    _paused = false;
}

/**
 * @dev Modifier to make a function callable only when the contract is not paused.
 *
 * Requirements:
 *
 * - The contract must not be paused.
 */
modifier whenNotPaused() {
    _requireNotPaused();
    _;
}

/**
 * @dev Modifier to make a function callable only when the contract is paused.
 *
 * Requirements:
 *
 * - The contract must be paused.
 */
modifier whenPaused() {
    _requirePaused();
    _;
}

/**
 * @dev Returns true if the contract is paused, and false otherwise.
 */
function paused() public view virtual returns (bool) {
    return _paused;
}

/**
 * @dev Throws if the contract is paused.
 */
function _requireNotPaused() internal view virtual {
    require(!paused(), "Pausable: paused");
}

/**
 * @dev Throws if the contract is not paused.
 */
function _requirePaused() internal view virtual {
    require(paused(), "Pausable: not paused");
}

/**
 * @dev Triggers stopped state.
 *
 * Requirements:
 *
 * - The contract must not be paused.
 */
function _pause() internal virtual whenNotPaused {
    _paused = true;
    emit Paused(_msgSender());
}

/**
 * @dev Returns to normal state.
 *
 * Requirements:
 *
 * - The contract must be paused.
 */
function _unpause() internal virtual whenPaused {
    _paused = false;
    emit Unpaused(_msgSender());
}

/**
 * @dev This empty reserved space is put in place to allow future versions to add new
 * variables without shifting down storage in the inheritance chain.
 * See https://docs.openzeppelin.com/contracts/4.x/upgradeable#storage_gaps
 */
uint256[49] private __gap;
}
```

## Tools Used
Manual review

## Recommended Mitigation Steps
Implement an access-controlled pause and unpause function

