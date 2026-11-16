One Signature, $55 Million: How a Wallet Drain Bypasses Approval Warnings
Onchain threat note 01 · Emmanuel Uzoekwe · November 2026
Summary
In August 2024, a crypto holder lost 55,473,618 DAI (about $55.4M) without ever approving a token to an attacker. A phishing site got them to sign one transaction that handed over ownership of their DSProxy, the smart contract that controlled their Maker vault. With ownership, the attacker took control of the vault and withdrew the DAI.
The case matters for anyone building wallet protection. Most approval warnings focus on token permissions such as `approve`, `setApprovalForAll` and `permit`. This drain used none of them. The dangerous action was an ownership transfer of a contract that itself held authority over funds.
This note reconstructs the attack from public reporting and onchain data, places it alongside the 2025 drainer data, and turns both into concrete detection signals and user guidance.
1. What happened
Item	Detail
Date	Tuesday, 20 August 2024 (first reported 21 August)
Loss	55,473,618 DAI, about $55.4M
Victim setup	An externally owned account (EOA) controlled a DSProxy, and the DSProxy controlled a Maker vault
Entry point	A phishing transaction signed by the victim
First flagged by	Onchain investigator ZachXBT
Attribution	CertiK linked the attack to the Inferno Drainer phishing kit
Transaction cited by The Block	`0xf70042bf3ae7c22f0680f8afa078c38989ed475dfbe5c8d8f30a50d4d2f45dc4`
The sequence, as reported:
The victim signed a transaction that transferred ownership of their DSProxy to an address Etherscan labels Fake_Phishing187019.
That address passed ownership on to a second attacker address (published in truncated form as `0x5D4b2…`).
Now in control of the proxy, the attacker set the vault's owner to their own wallet and minted 55,473,618 DAI out of it.
When the victim later tried to use their DSProxy, the call failed, because they no longer owned it.
Phishing or stolen key? BlockSec judged a signed phishing transaction more likely than a leaked private key, and Halborn reached the same view. The victim's later attempt to call the proxy fits someone who realised, too late, that a transaction they had signed was malicious.
2. Why it worked
No token left the wallet at the moment of signing. The victim's DAI sat inside a vault, reachable only through the DSProxy. Transferring the proxy's ownership did not move a single token, so a check that only asks "is this transaction moving or approving my tokens?" would see nothing alarming.
The authority was one layer removed. DSProxy is built on the ds-auth pattern, where a single `setOwner(address)` call decides who can act through the proxy. Whoever owns the proxy effectively owns everything the proxy controls. Changing the owner of a contract like this is equivalent to handing over the vault.
The prompt looked technical, not dangerous. A raw contract call to a proxy is hard to read in a wallet pop-up. Users who have learned to look out for the word "approve" have no reason to fear an unfamiliar function name on a contract they already use.
3. The wider pattern: drainers in 2025
Scam Sniffer's 2025 report on wallet-drainer phishing across EVM chains shows the same idea at scale: drainers get the victim to authorise the theft, so the blockchain treats it as legitimate.
Total losses: $83.85M across 106,106 victims, down 83% from $494M in 2024.
Seasonality: losses tracked market activity. Q3 2025 was the worst quarter, during ETH's strongest rally.
Large cases: 11 thefts of $1M or more, totalling $22.98M.
Large cases by signature type (2025, cases of $1M or more):
Signature type	Cases	Losses	What the victim actually authorised
Permit / Permit2	3	$8.72M	An off-chain signature letting a spender move tokens. Gasless to sign, so it feels harmless
Approve / increaseApproval	3	$5.62M	An onchain allowance for a spender, often unlimited
Transfer	2	$4.87M	A direct transfer of the asset
EIP-7702 batch	2	$2.54M	A delegation that bundles several malicious operations under one signature, new after the Pectra upgrade
setApprovalForAll	1	$1.23M	Control of an entire NFT collection, here Uniswap V3 liquidity positions
The largest 2025 theft was $6.5M in stETH and aEthWBTC via a single Permit signature in September. In 2024, Scam Sniffer recorded Permit and `setOwner` as the dominant signature types, which is exactly the family the DSProxy case belongs to.
4. Detection signals
These are the signals I would want a wallet guard to check before the user signs, with a suggested response.
#	Signal	Why it matters	Suggested action
1	Ownership or admin change (`setOwner`, `transferOwnership`, `setAuthority`) on a contract the user owns or uses	Hands over everything that contract controls. The DSProxy case	Block unless the new owner is an address the user already controls. Explain what the contract controls in plain words
2	Permit or Permit2 typed data to a spender with no history, or on a threat feed	Gasless signatures feel safe but authorise transfers	Block for flagged spenders. Otherwise warn, showing token, amount and deadline in plain language
3	Unlimited allowance (max uint256) or `increaseApproval` to a new spender	A standing allowance can be drained at any time later	Warn, show the exact amount, and suggest a limited amount
4	`setApprovalForAll(true)` to an operator that is not a known marketplace	Gives control of a whole NFT collection, including LP positions	Warn strongly or block
5	EIP-7702 authorisation delegating the account to an unverified contract	One signature can bundle many malicious calls	Block by default
6	Request targets the wallet's highest-value asset immediately after connecting	Drainer kits enumerate balances and ask for the biggest prize first	Raise the risk score on top of any other signal
7	Spender, recipient or new owner is newly created or carries a phishing label	Fresh or labelled addresses are a strong drainer indicator	Raise the risk score and name the label to the user
8	Domain is newly registered, a lookalike of a known protocol, or a known protocol domain serving unexpected scripts	Drainers live on cloned sites and sometimes on hijacked real ones	Warn on connect, block on sign
The product lesson from this case: approval checks should not stop at token permissions. Any call that changes who controls a contract holding user funds deserves the same, or higher, severity as an unlimited approval.
5. Guidance for users
Read what you are giving away, not just what you are paying. A signature that moves nothing can still hand over control.
Treat "free" signatures as seriously as transactions. Permit and Permit2 cost no gas to sign but can authorise transfers.
Be suspicious of ownership or admin changes. A normal dApp action almost never needs you to change the owner of your own contracts.
Avoid unlimited approvals, and review and revoke old ones regularly with a tool such as Revoke.cash.
Arrive at dApps from bookmarks, not from links in DMs, replies or search ads. Drainers rely on cloned sites.
Keep large holdings in a separate wallet that never connects to new or experimental sites.
6. Reproduce this onchain
Anyone can check the core facts with a block explorer:
Open the transaction above on Etherscan and note the sender, the contract called and the block time.
Under Input Data, use Decode Input Data to see which function was called and with which arguments.
Under Logs, look for ownership events (such as `LogSetOwner`) and DAI transfer or mint events, and note the addresses involved.
Open the addresses involved and check for Etherscan labels such as Fake_Phishing187019.
Follow the funds from the receiving address through its next transactions to see where the DAI went.
Sources
The Block: Crypto whale loses $55 million in Dai stablecoin to phishing attack (21 August 2024)
Halborn: Explained, the $55M whale phishing hack (August 2024)
Scam Sniffer: 2025 crypto phishing losses fall 83% to $84 million (January 2026)
Cointelegraph: Crypto phishing losses fell 83% in 2025
