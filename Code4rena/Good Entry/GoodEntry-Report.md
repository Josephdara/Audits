# [Good Entry Report](https://code4rena.com/reports/2023-08-goodentry)

## Findings by Josephdara
| Severity | Title | Count |
|:--:|:---|:--:|
| [M-01](#m-01-V3-Proxy-does-not-send-funds-to-the-recipient-instead-it-sends-to-the-msgSender)|V3 Proxy does not send funds to the recipient instead it sends to the msgSender| M-01 |
| [M-02](#m-02-Complete-Loss-of-funds-when-swapping-to-Ether-from-another-contract)| Complete Loss of funds when swapping to Ether from another contract| M-02 |


## [M-01] V3 Proxy does not send funds to the recipient instead it sends to the msgSender

## Impact and Details
https://github.com/code-423n4/2023-08-goodentry/blob/71c0c0eca8af957202ccdbf5ce2f2a514ffe2e24/contracts/helper/V3Proxy.sol#L112-L194

The functions above can be used to swap tokens, however the swaps are not sent to the provided address. Instead they are sent to the msg.sender.
This could cause issues if the user has been blacklisted on a token. Or if the user has a compromised signature/allowance of the target token and they attempt to swap to the token, the user looses all value even though they provided an destination adress

### Proof of Concept

```solidity 
//@audit-H does not send tokens to the required address

    function swapExactTokensForTokens(uint amountIn, uint amountOutMin, address[] calldata path, address to, uint deadline) external returns (uint[] memory amounts) {
        require(path.length == 2, "Direct swap only");
        ERC20 ogInAsset = ERC20(path[0]);
        ogInAsset.safeTransferFrom(msg.sender, address(this), amountIn);
        ogInAsset.safeApprove(address(ROUTER), amountIn);
        amounts = new uint[](2);
        amounts[0] = amountIn;         
        //@audit-issue it should be the to address not msg.sender
        amounts[1] = ROUTER.exactInputSingle(ISwapRouter.ExactInputSingleParams(path[0], path[1], feeTier, msg.sender, deadline, amountIn, amountOutMin, 0));
        ogInAsset.safeApprove(address(ROUTER), 0);
        emit Swap(msg.sender, path[0], path[1], amounts[0], amounts[1]); 
    }
```
Here is one of the many functions with this issue, As we can see after the swap is completed, tokens are sent back to the msg.sender from the router not to the to address

## Tools Used
Manual Review, Uniswap Router: https://github.com/Uniswap/v3-periphery/blob/main/contracts/SwapRouter.sol

## Recommended Mitigation Steps
The uniswap router supports inputting of a destination address. Hence the router should be called with the to address not the msg.sender.
Else remove the address to from the parameter list



## [M-02] Complete Loss of funds when swapping to Ether from another contract
## Impact and Details

https://github.com/code-423n4/2023-08-goodentry/blob/71c0c0eca8af957202ccdbf5ce2f2a514ffe2e24/contracts/helper/V3Proxy.sol#L160-L194

Ether swaps conducted with the V3proxy would be lost to another user forever.
This is because of 2 major issues
- unchecked return value
- wrong destination address being used

### Proof of Concept

When swapping for ETH with the function below:
```solidity
   function swapExactTokensForETH(uint amountIn, uint amountOutMin, address[] calldata path, address to, uint deadline) payable external returns (uint[] memory amounts) {
        require(path.length == 2, "Direct swap only");
        require(path[1] == ROUTER.WETH9(), "Invalid path");
        ERC20 ogInAsset = ERC20(path[0]);
        ogInAsset.safeTransferFrom(msg.sender, address(this), amountIn);
        ogInAsset.safeApprove(address(ROUTER), amountIn);
        amounts = new uint[](2);
        amounts[0] = amountIn;         
        amounts[1] = ROUTER.exactInputSingle(ISwapRouter.ExactInputSingleParams(path[0], path[1], feeTier, address(this), deadline, amountIn, amountOutMin, 0));
        ogInAsset.safeApprove(address(ROUTER), 0); 
        IWETH9 weth = IWETH9(ROUTER.WETH9());
        acceptPayable = true;
        weth.withdraw(amounts[1]);
        acceptPayable = false;
        payable(msg.sender).call{value: amounts[1]}("");
        emit Swap(msg.sender, path[0], path[1], amounts[0], amounts[1]);                 
    }
```
WETH is returned from the uniswap and sent to the contract. The ether value is sent to the contract from WETH, and the the value is sent to the user via the call() method.
However the calls can be made from a contract as the msg.sender in this case. This causes loss of funds in two scenarios

If the contract sending the transaction has no receive or fallback function, the transaction does not revert due to the call method but it returns false and a bytes32 value. However the bool value is uncaught and unchecked. Hence the ETH is left in the contract and no funds are received. This is an issue for the protocol because even if a destination address is entered as the address to, it still sends to the direct msg.sender.
The calling contract might have a receive or fallback function but no way to withdraw the eth. This is an issue for the protocol because even if a destination address is entered as the address to, it still sends to the direct msg.sender.

## Tools Used
Manual Review

## Recommended Mitigation Steps
Check the bool returned after the call() for success, to prevent loss of funds
Also send the ETH to the required destination
