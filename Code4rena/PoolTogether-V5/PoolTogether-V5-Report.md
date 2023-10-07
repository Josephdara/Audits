 # [PoolTogether Report](https://code4rena.com/reports/2023-08-pooltogether)

## Findings by Josephdara
| Severity | Title | Count |
|:--:|:---|:--:|
| [H-01](#h-01-function-rngComplete-is-unpprotected)| function rngComplete is unpprotected | H-01 |


## [H-01] function rngComplete is unpprotected 

## Impact and Details
https://github.com/GenerationSoftware/pt-v5-draw-auction/blob/f1c6d14a1772d6609de1870f8713fb79977d51c1/src/RngRelayAuction.sol#L131-L176


## Impact
The rngComplete is a function Called by the relayer to complete the Rng relay auction.
However it has zero access control.

## Proof of Concept
The function makes calls to the prizepool to close a draw, it also withdraws from a reserve. All these are done with the data passed in as parameters. Hence it should not be left unguarded
```solidity 
 function rngComplete(
    uint256 _randomNumber,
    uint256 _rngCompletedAt,
    address _rewardRecipient,
    uint32 _sequenceId,
    AuctionResult calldata _rngAuctionResult
  ) external returns (bytes32) {
    if (_sequenceHasCompleted(_sequenceId)) revert SequenceAlreadyCompleted();
    uint64 _auctionElapsedSeconds = uint64(block.timestamp < _rngCompletedAt ? 0 : block.timestamp - _rngCompletedAt);
    if (_auctionElapsedSeconds > (_auctionDurationSeconds-1)) revert AuctionExpired();
    // Calculate the reward fraction and set the draw auction results
    UD2x18 rewardFraction = _fractionalReward(_auctionElapsedSeconds);
    _auctionResults.rewardFraction = rewardFraction;
    _auctionResults.recipient = _rewardRecipient;
    _lastSequenceId = _sequenceId;

    AuctionResult[] memory auctionResults = new AuctionResult[](2);
    auctionResults[0] = _rngAuctionResult;
    auctionResults[1] = AuctionResult({
      rewardFraction: rewardFraction,
      recipient: _rewardRecipient
    });

    uint32 drawId = prizePool.closeDraw(_randomNumber);

    uint256 futureReserve = prizePool.reserve() + prizePool.reserveForOpenDraw();
    uint256[] memory _rewards = RewardLib.rewards(auctionResults, futureReserve);

    emit RngSequenceCompleted(
      _sequenceId,
      drawId,
      _rewardRecipient,
      _auctionElapsedSeconds,
      rewardFraction
    );

    for (uint8 i = 0; i < _rewards.length; i++) {
      uint104 _reward = uint104(_rewards[i]);
      if (_reward > 0) {
        prizePool.withdrawReserve(auctionResults[i].recipient, _reward);
        emit AuctionRewardDistributed(_sequenceId, auctionResults[i].recipient, i, _reward);
      }
    }

    return bytes32(uint(drawId));
  }

  ```
## Tools Used
Manual Review

## Recommendation
Since it says that it is a relayer function. Then require that the msg.sender is the rngAuctionRelayer address

