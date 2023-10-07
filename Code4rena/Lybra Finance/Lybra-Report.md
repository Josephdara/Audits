# [Lybra Finance Report](https://code4rena.com/reports/2023-06-lybra)

## Findings by Josephdara
| Severity | Title | Count |
|:--:|:---|:--:|
| [H-01](#h-01-Invalid-Access-Control-Modifiers)| Invalid Access Control Modifiers| H-01 |
| [H-02](#h-02-_voteSucceeded-check-is-invalid)| _voteSucceeded check is invalid| H-02 |
| [H-03](#h-03-_quorumReached-does-not-add-all-votes)| _quorumReached does not add all votes| H-03 |
| [M-01](#m-01-Wrong-validation-when-setting-BadCollateralRatio)| Wrong validation when setting BadCollateralRatio| M-01 |
| [M-02](#m-02-Voting-period-hardcoded-to-3blocks)| Voting period hardcoded to 3 blocks| M-02 |

## [H-01] Invalid Access Control Modifiers

## Impact and Details
https://github.com/code-423n4/2023-06-lybra/blob/7b73ef2fbb542b569e182d9abf79be643ca883ee/contracts/lybra/configuration/LybraConfigurator.sol#L85-L93

The LybraConfigurator is the contract in charge of all core functionality in the Lybra ecosystem. However, the modifiers checks here are invalid. So anybody could call any function in the protocol. All funds could be stolen and governance overturned

Proof of Concept
```solidity 
//@audit-issue modifier does not revert  
   modifier onlyRole(bytes32 role) {
       GovernanceTimelock.checkOnlyRole(role, msg.sender);
       _;
   }

   modifier checkRole(bytes32 role) {
       GovernanceTimelock.checkRole(role, msg.sender);
       _;
   }
```
This modifier here makes a call to the GovernanceTimelock smart contract to fetch the roles there. In the GovernanceTimelock smart contract however, it is visible that this checks are invalid.
```solidity 
    function checkRole(bytes32 role, address _sender) public view  returns(bool){
        return hasRole(role, _sender) || hasRole(DAO, _sender);
    }

    function checkOnlyRole(bytes32 role, address _sender) public view  returns(bool){
        return hasRole(role, _sender);
    }
```
This makes a call to the inherited hasRole function which does not revert, but instead returns a bool, True if the address has the role, and false if the person does not have the role.
```solidity 
    function hasRole(bytes32 role, address account) public view virtual override returns (bool) {
        return _roles[role].members[account];
    }
```
Hence the modifier only gets a false Boolean value but does not revert because there is no require statement involved.

Tools Used
Manual Review

Recommended Mitigation Steps
Update the modifiers to have require statements:
```solidity
//@audit-issue modifier modifier would revert if false is returned
   modifier onlyRole(bytes32 role) {
       require(GovernanceTimelock.checkOnlyRole(role, msg.sender));
       _;
   }

   modifier checkRole(bytes32 role) {
       require(GovernanceTimelock.checkRole(role, msg.sender));
       _;
   }
```


## [H-02] _voteSucceeded check is invalid

## Impact and Details
https://github.com/code-423n4/2023-06-lybra/blob/7b73ef2fbb542b569e182d9abf79be643ca883ee/contracts/lybra/governance/LybraGovernance.sol#L63-L68
 ```solidity 
    //@audit-issue wrong check vote succeeded is when 0 > 1
    function _voteSucceeded(uint256 proposalId) internal view override returns (bool){
        return proposalData[proposalId].supportVotes[1] > proposalData[proposalId].supportVotes[0];
    } 
 ```
_voteSucceeded is a function that checks if the votes have succeeded in favor of the proposal, however, this function returns true if proposal fails and false if proposal fails

Proof of Concept
According to the contract

```solidity 
        forVotes =  proposalData[proposalId].supportVotes[0];
        againstVotes =  proposalData[proposalId].supportVotes[1];
        abstainVotes =  proposalData[proposalId].supportVotes[2];
```
That means if proposalData[proposalId].supportVotes[1] > proposalData[proposalId].supportVotes[0] then the proposal was defeated but if  proposalData[proposalId].supportVotes[0] > proposalData[proposalId].supportVotes[1] then the proposal succeeded

## Tools Used
Manual Review

## Recommended Mitigation Steps
Revert the values to appropriately check


## [H-03] _quorumReached does not add all votes

## Impact and Details
https://github.com/code-423n4/2023-06-lybra/blob/7b73ef2fbb542b569e182d9abf79be643ca883ee/contracts/lybra/governance/LybraGovernance.sol#L57-L61

_quorumReached is a function that checks if the Amount of votes already cast passes the threshold limit.
But the function does not add all votes
```solidity

     //@audit-issue quorum reached does not add all votes
    function _quorumReached(uint256 proposalId) internal view override returns (bool){
        return proposalData[proposalId].supportVotes[1] + proposalData[proposalId].supportVotes[2] >= quorum(proposalSnapshot(proposalId));
    }
```
Proof of Concept
As seen below in the contract
```solidity
    forVotes =  proposalData[proposalId].supportVotes[0];
        againstVotes =  proposalData[proposalId].supportVotes[1];
        abstainVotes =  proposalData[proposalId].supportVotes[2];
 ```
This means supportVotes[0], supportVotes[1], and supportVotes[2] are all valid votes. But the _quorumReached function only checks the last 2 against one third of all possible votes

## Tools Used
Manual review

## Recommended Mitigation Steps
Add all votes in the function before checking

## [M-01] Wrong validation when setting BadCollateralRatio

## Impact and Details
https://github.com/code-423n4/2023-06-lybra/blob/7b73ef2fbb542b569e182d9abf79be643ca883ee/contracts/lybra/configuration/LybraConfigurator.sol#L123-L130

Setting of BadCollateralRatio has a slight bug
```solidity 
  //@audit-issue bug here, should be - 1e19
    function setBadCollateralRatio(address pool, uint256 newRatio) external onlyRole(DAO) {
        require(newRatio >= 130 * 1e18 && newRatio <= 150 * 1e18 && newRatio <= vaultSafeCollateralRatio[pool] + 1e19, "LNA");
        vaultBadCollateralRatio[pool] = newRatio;
        emit SafeCollateralRatioChanged(pool, newRatio);
    }
 ```
this check for newRatio <= vaultSafeCollateralRatio[pool] + 1e19 should be newRatio <= vaultSafeCollateralRatio[pool] - 1e19

## Proof of Concept
Going by the design pattern of the codebase, it is show that the BadCollateralRatio should be at least 1e19 less than vaultSafeCollateralRatio at all times.
It is even shown here that for the LybraEUSDVault with the hardcoded bad collateral of 150e18, the safe ratio has to be atleast 1e19 greater which is newRatio >= 160 * 1e18, for the LybraPeUSDVault, it gets the BadCollateralRatio from the mapping and ensure that it is greater than it by atleast 1e19

    function setSafeCollateralRatio(address pool, uint256 newRatio) external checkRole(TIMELOCK) {
        if(IVault(pool).vaultType() == 0) {
            require(newRatio >= 160 * 1e18, "eUSD vault safe collateralRatio should more than 160%");
        } else {
            require(newRatio >= vaultBadCollateralRatio[pool] + 1e19, "PeUSD vault safe collateralRatio should more than bad collateralRatio");
        }
        vaultSafeCollateralRatio[pool] = newRatio;
        emit SafeCollateralRatioChanged(pool, newRatio);
    }
This has led me to conclude that newRatio <= vaultSafeCollateralRatio[pool] + 1e19 is indeed a bug in the code

## Tools Used
Manual Review

## Recommended Mitigation Steps
Change + to -




## [M-02] Voting period hardcoded to 3 blocks
## Impact and Details

https://github.com/code-423n4/2023-06-lybra/blob/7b73ef2fbb542b569e182d9abf79be643ca883ee/contracts/lybra/governance/LybraGovernance.sol#L143-L145

Here in the Governance contract, the voting period is locked to 3 blocks.
```solidity 
     function votingPeriod() public pure override returns (uint256){
         return 3;
    }

     function votingDelay() public pure override returns (uint256){
         return 1;
    }
```
This is a direct bug because if we take a look at the inherited Governor contract we see where the proposal is being created and the voting limits are set
```solidity 
 function propose(
        address[] memory targets,
        uint256[] memory values,
        bytes[] memory calldatas,
        string memory description
    ) public virtual override returns (uint256) {
        address proposer = _msgSender();
        require(_isValidDescriptionForProposer(proposer, description), "Governor: proposer restricted");

        uint256 currentTimepoint = clock();
        require(
            getVotes(proposer, currentTimepoint - 1) >= proposalThreshold(),
            "Governor: proposer votes below proposal threshold"
        );

        uint256 proposalId = hashProposal(targets, values, calldatas, keccak256(bytes(description)));

        require(targets.length == values.length, "Governor: invalid proposal length");
        require(targets.length == calldatas.length, "Governor: invalid proposal length");
        require(targets.length > 0, "Governor: empty proposal");
        require(_proposals[proposalId].voteStart == 0, "Governor: proposal already exists");

        uint256 snapshot = currentTimepoint + votingDelay();
        uint256 deadline = snapshot + votingPeriod();

        _proposals[proposalId] = ProposalCore({
            proposer: proposer,
            voteStart: SafeCast.toUint64(snapshot),
            voteEnd: SafeCast.toUint64(deadline),
            executed: false,
            canceled: false,
            __gap_unused0: 0,
            __gap_unused1: 0
        });
    }
```
However after reaching out to the sponsor, I was informed that the voting periods are supposed to be 2 weeks long. Instead the voteEnd is set to 36 secs after proposal.

## Tools Used
Manual Review

## Recommended Mitigation Steps
I suggest the voting period be set to a changeable variable

