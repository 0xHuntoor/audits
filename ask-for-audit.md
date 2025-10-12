## Auditing Process

My auditing process involves multiple phases to improve the safety of the protocol as much as I can.

You can message me on **[Telegram](https://t.me/huntoor)** to request it.

### Pre-Audit Phase
This process may take a few hours or a day at most. In this phase, I review the protocol and check its structure and code.
- The degree of decentralization of the protocol.
- Code complexity.
- Access control logic.

Then, I search for common bugs like simple re-entrancy, unsafe casting, weird ERC20 cases not handled, etc. This gives me an impression of how secure the protocol is (will it contain many bugs, or is its code robust with fewer bugs).

Finally, I give the sponsor (protocol developers) my view of the protocol codebase, the common bugs that I found, and the price of the audit (yes, this process is free for all protocols).

**NOTE:**
- Some types of protocols I do not accept can be found [here](#protocols-i-do-not-accept).
- To know how much it will cost to audit your protocol, check this [link](#pricing-and-duration).

### Auditing Phase
This is the longest process. If we reach an agreement on the price and duration, the auditing process starts as follows:
- I make a security review of the codebase.
- When auditing, I can contact protocol developers to ask about something unclear, abnormal behavior, or an issue I’m not sure about.
- After finishing, I submit all issues I found in GitRep/issues, labeled by the severity of the issue.
- Developers can discuss the issue’s validity, severity, and anything related to it on the issue’s page.
- Developers will fix issues that have impact and can acknowledge some issues if they see it has no major impact.

### Post-Audit Phase
After fixing the issues, I make another audit of the protocol to check the issue mitigations.
- Check that the issues are fixed successfully without introducing further issues.
- Give my final thoughts about the project.
- Confirm if I see that the protocol is safe to go for Mainnet or if another audit is better before launching.

This is how the auditing process occurs. Feel free to drop a message, and I will be more than happy to secure the protocol.

---

## Pricing And Duration

### Pricing:

The price to audit the protocol depends on two different things:
1. The number of Solidity code lines (the more code, the more expensive).
2. The code complexity (is there an integration with another protocol, heavy math, YUL code, etc.).

The price ranges between: 
- [5, 10] USDC per LOC based on https://github.com/Consensys/solidity-metrics.
- 4-8K/week

Whatever rating method is more convenient for you.

The price is adjusted according to the complexity of the code.

### Duration:

The duration is not a constant period for all protocols with the same SLOCs, as complexity and conditions vary. But in normal cases, here is a table of the durations of the protocol according to the SLOC, where it should not exceed these periods.

|SLOC|Duration|
|:--:|:--------:|
| SLOC <= 500 | 4 days|
| 1000 >= SLOC > 500| 7 days|
| 1500 >= SLOC > 1000| 10 days|
| 2000 >= SLOC > 1500| 14 days|

This duration may change from one protocol to another, but it is a good approximation.

---

## Important Things to know
- The payment is fully delivered after the Pre-Audit Phase. In case of agreement, the auditing process will only start after the price we agreed on is delivered.
- The mitigation process should not exceed 2 weeks (after finishing the auditing process, mitigation should take 2 weeks at most). If it takes longer than 2 weeks, a fine is applied.
- The mitigation process should fix only the issues found; any new features added are not part of the audit responsibility.
- Mitigation should not require too many modifications. Most issues are mitigated by a simple check or something like that. But sometimes there may be an issue in the design itself, which requires changing the overall protocol structure. In case the mitigation is too complex, there is an additional quote for reviewing it (like a mitigation of an issue that requires adding `200` SLOC, for example).

---

## Protocols I do not accept

### Protocols used for frauding and stealing

If the protocol will be used to steal users’ funds, drain wallets, or deceive users, then I will not accept auditing this protocol.

### Gambling and Lottery Protocols

I do not accept auditing protocols that deal with lotteries and gambling, like casino protocols where people bid with their money and may be the winner.

### Lending/Borrowing with interest rate > 0

Lending/borrowing protocols like AAVE and Compound are some of the most famous protocols in the DeFi space, and they are used a lot. In most cases, the borrower will return the value he took from the lender and pay fees for this.
- For example, if he borrowed 100 ETH, he will pay 100.5 ETH when giving it back to the lender.

This type of lending/borrowing protocol, I do not accept.

If there are no fees (interest rate) accumulated by the lender, then I can accept the protocol.

If you are not sure about your protocol type, interest rate mechanism, or its integrations, you can message me, and I will tell you whether I can accept it or not.

### Leverage Protocols with fees paid to the Lender

In most cases, if the protocol deals with leverage, I can’t audit it. This includes leverage in borrowing/lending and leverage on trading.

### Perpetual/Options trading

In the case of forex perpetual/options, it is done by simulating the buying and selling process—i.e., the user is not actually buying the tokens he wants; the system just records this.

If the users are not actually taking their tokens when making a trade and do not own them themselves, then I cannot accept this protocol.

**In Brief**
- If the user owns the (tokens) position, meaning he can either sell it or transfer it to another one, then I can accept the protocol.
- If the user position is known using the smart contract storage, and users cannot see their token value, nor transfer the position, and can only close the position from the contract, then I cannot accept the protocol.

Some protocols are complex, and some protocols can integrate with other protocols that do one of the things I do not accept. So if you’re not sure about the protocol type, you can message me, and I will tell you whether I can accept the protocol or not.

### Trading Protocols with Shorts
If the protocol is like a trading protocol that implements a shorting mechanism, I am not accepting it.

### Yield Farming Strategies
If a yield farming protocol depends on strategies or vaults that gain profits from one of the protocol types I do not accept, then I do not accept the protocol.

For example, if the protocol interacts with an AAVE vault, then I cannot accept the audit, as AAVE vaults gain profit from lending/borrowing.

### Integrations
If the protocol integrates with one of the protocol types that I do not accept, then I may not accept the audit; this depends on how it integrates with it.

## Disclaimer

Smart contract auditing cannot guarantee the safety of the protocol 100%. I try my best to find as many issues as I can that can put the protocol in an abnormal state. But I cannot be sure that I found all issues and attacks that can occur.
