# MonethaBuyer contract bug bounty

  * https://www.reddit.com/r/ethdev/comments/6x5j33/bug_bounty_for_monetha_mth_ico_buyer_contract/
  * Date: 2017-08-31
  * Reward: 10 ETH

## Contracts

[MonthaBuyer.sol](./contracts/MonthaBuyer.sol)

## Report

If the token buy fails and I call the withdraw function from a contract with nonzero ETH balance on the buyer contract, then after receiving the bounty deposit it back on the buyer contract (which is possible because there is no "!(now > earliest_buy_time + 1 hours)" check in the default payable function), I can call the withdraw again. If I do this recursively, I can claim most of the remaining withdrawal bounty in one transaction without performing any useful function.

```javascript
  // Withdraws all ETH deposited or tokens purchased by the given user and rewards the caller.
  function withdraw(address user){
    // Only allow withdrawals after the contract has had a chance to buy in.
    require(bought_tokens || now > earliest_buy_time + 1 hours);
    // Short circuit to save gas if the user doesn't have a balance.
    if (balances[user] == 0) return;
    // If the contract failed to buy into the sale, withdraw the user's ETH.
    if (!bought_tokens) {
      // Store the user's balance prior to withdrawal in a temporary variable.
      uint256 eth_to_withdraw = balances[user];
      // Update the user's balance prior to sending ETH to prevent recursive call.
      balances[user] = 0;
      // Return the user's funds.  Throws on failure to prevent loss of funds.
      user.transfer(eth_to_withdraw);
    }
    // Withdraw the user's tokens if the contract has purchased them.
    else {
      // Retrieve current token balance of contract.
      uint256 contract_token_balance = token.balanceOf(address(this));
      // Disallow token withdrawals if there are no tokens to withdraw.
      require(contract_token_balance != 0);
      // Store the user's token balance in a temporary variable.
      uint256 tokens_to_withdraw = (balances[user] * contract_token_balance) / contract_eth_value;
      // Update the value of tokens currently held by the contract.
      contract_eth_value -= balances[user];
      // Update the user's balance prior to sending to prevent recursive call.
      balances[user] = 0;
      // 1% fee if contract successfully bought tokens.
      uint256 fee = tokens_to_withdraw / 100;
      // Send the fee to the developer.
      require(token.transfer(developer, fee));
      // Send the funds.  Throws on failure to prevent loss of funds.
      require(token.transfer(user, tokens_to_withdraw - fee));
    }
    // Each withdraw call earns 1% of the current withdraw bounty.
    uint256 claimed_bounty = withdraw_bounty / 100;
    // Update the withdraw bounty prior to sending to prevent recursive call.
    withdraw_bounty -= claimed_bounty;
    // Send the caller their bounty for withdrawing on the user's behalf.
    msg.sender.transfer(claimed_bounty);
  }

```

```javascript
  // Default function.  Called when a user sends ETH to the contract.
  function () payable {
    // Disallow deposits if kill switch is active.
    require(!kill_switch);
    // Only allow deposits if the contract hasn't already purchased the tokens.
    require(!bought_tokens);
    // Only allow deposits that won't exceed the contract's ETH cap.
    require(this.balance < eth_cap);
    // Update records of deposited ETH to include the received amount.
    balances[msg.sender] += msg.value;
  }
```

