# SmartContractSecurity

Smart contract security: common vulnerabilities, audits, and safe patterns. Most crypto hacks do not break the blockchain — they exploit bugs in individual smart contracts. This guide covers the bug classes that have caused the largest losses, and the patterns that prevent them.

> This guide is educational only, not financial or legal advice. No audit or pattern guarantees safety, and unaudited changes after an audit are not covered by that audit.

## 1. Why smart contracts are hard to secure

- **Immutability:** once deployed, code often cannot be changed. A bug found after deployment cannot always be patched — funds may be stuck in a buggy contract forever, or must be migrated through a risky upgrade.
- **Adversarial environment:** code is public, anyone can interact with it, and attackers are financially motivated and highly skilled. A contract is attacked by the whole world, continuously.
- **Composability:** contracts call other contracts. A secure contract can become insecure when combined with a malicious or buggy one.
- **Money at stake:** unlike most software bugs, which cause crashes or data loss, smart contract bugs cause immediate, irreversible financial loss.

Security in smart contracts is therefore not a feature added at the end. It is the design discipline from the first line.

## 2. The most costly vulnerability classes

### Reentrancy
A contract sends funds (or calls another contract) before updating its own balances. The receiving contract calls back into the original contract before the balance update, and withdraws again — and again.

- **Famous example:** the 2016 DAO hack, which drained about $60 million at the time and led to the Ethereum / Ethereum Classic split.
- **Fix (checks-effects-interactions):** update balances first, then interact with external contracts. Also use a reentrancy guard (a lock that prevents nested calls).
- **User lesson:** you cannot detect reentrancy from a UI. You rely on audits, battle-testing, and time — prefer protocols that have survived heavy use.

### Integer overflow and underflow
In older Solidity versions, arithmetic that exceeded a number type's range wrapped around silently: 0 − 1 became the maximum value, creating tokens from nothing.

- **Fix:** modern Solidity (0.8+) reverts on overflow by default. Libraries like OpenZeppelin's SafeMath provided the same protection for older versions.
- **User lesson:** this class is largely solved for new contracts, but old, unaudited contracts may still carry it.

### Access control failures
Functions that should be restricted (minting, pausing, upgrading, draining) are callable by anyone, because the developer forgot a permission check, or the owner key was a single wallet that got compromised.

- **Fix:** explicit access control (such as OpenZeppelin's Ownable or AccessControl), multisig admin keys, timelocks for sensitive changes, and the principle of least privilege — no function should have more power than it needs.
- **User lesson:** check who the owner is and what owner-only functions exist, on the explorer, before trusting a contract (see SmartContractBasics, section 4).

### Oracle manipulation
A contract relies on a price feed (oracle) to decide liquidations, swaps, or rewards. An attacker manipulates that price — often with a flash loan (a loan borrowed and repaid within one transaction) that temporarily moves a thin pool's price — and the contract acts on the false price.

- **Fix:** use time-weighted average prices (TWAPs) instead of spot prices, use multiple decentralised oracles, and add sanity checks and circuit breakers for extreme price moves.
- **User lesson:** protocols that price assets from a single, thin liquidity pool are fragile. Prefer protocols using robust oracle designs.

### Flash loan attacks
Flash loans themselves are not a vulnerability — they are a legitimate DeFi tool. But they let an attacker borrow enormous capital for one transaction, with no collateral, which amplifies every other bug: oracle manipulation, reentrancy, and governance attacks all become cheaper and larger with flash loans.

- **Fix:** the same fixes as for the underlying bugs, plus flash-loan-resistant design (for example, not allowing governance votes with flash-loaned tokens in the same transaction).
- **User lesson:** if a protocol's security depends on attackers not having capital, it is not secure — flash loans remove that barrier entirely.

### Logic errors and economic attacks
Not every exploit is a coding bug. Some are economic: a reward formula that can be gamed, a liquidation bonus that is too generous, an incentive that pays more than the value it creates. These are harder to audit, because the code does exactly what was written — the design itself is wrong.

- **Fix:** economic modelling, adversarial review ("how would I game this?"), staged rollouts with caps, and bug bounties that reward finding economic exploits, not just code bugs.

### Front-running and MEV (contract level)
Contracts that reveal profitable actions publicly (liquidations, arbitrage, auctions) invite bots to frontrun ordinary users. See MEVExplained for the full picture.

### Denial of service and gas griefing
A contract that loops over an unbounded list (for example, paying out to every holder in one transaction) can become too expensive to execute as the list grows, freezing funds. Or an attacker can force a contract into a state where legitimate transactions always fail.

- **Fix:** pull payments (users withdraw individually) instead of push payments (contract pays everyone at once), and bounded loops.

## 3. Audits: what they do and do not do

An **audit** is a professional review of a contract's code by a security firm, producing a report of findings, severity levels, and fixes.

What audits do:

- Catch known bug classes, logic errors, and access control mistakes that automated tools and reviewers can find.
- Force teams to think adversarially and document their assumptions.
- Provide a public signal of diligence — a team that publishes full audits is more accountable than one that hides them.

What audits do not do:

- **Guarantee safety.** Audits are time-limited, human reviews. They miss bugs, especially novel and economic ones.
- **Cover future code.** An audit of version 1 does not cover version 2, or a proxy upgrade made after the audit.
- **Cover the whole system.** Audits usually cover specific contracts, not the full stack: frontends, bridges, oracles, governance, and operational security (key management) are often out of scope.
- **Prove the team is honest.** An audited contract can still have an admin backdoor that the audit noted and the team kept, or a team that later acts maliciously.

**How to read an audit (beginner version):**

1. Is the full report public, not just a badge or a summary?
2. How many findings were critical or high severity, and were they fixed or merely "acknowledged"?
3. Does the audited code match the deployed code? (Check commit hashes and deployment addresses where the report provides them.)
4. How many firms audited it, and are they reputable? One audit from an unknown firm is weak evidence.
5. When was the audit? An audit from two years ago, before major upgrades, is stale.

## 4. Safe patterns developers use (and users should recognise)

- **Checks-Effects-Interactions:** validate inputs, update state, then interact externally. Prevents reentrancy.
- **Reentrancy guards:** locks that block nested calls into sensitive functions.
- **Pull over push payments:** users withdraw funds themselves, instead of the contract pushing payments to many users at once.
- **Least privilege access control:** each role can do only what it needs, enforced by code, not by trust.
- **Timelocks:** sensitive changes take effect only after a delay (for example, 48 hours), giving users time to exit if they disagree.
- **Multisig admin:** critical actions require multiple independent signers, so one compromised key is not enough.
- **Pausability with limits:** the ability to pause a contract during an attack, balanced against the risk that a malicious admin pauses to trap funds. Who can pause, and who can unpause, matters.
- **Upgrade transparency:** upgrades announced in advance, with code published and a timelock, so users are not surprised by new logic.
- **Bug bounties:** public rewards for finding vulnerabilities, which attract friendly researchers before hostile ones find the same bugs.
- **Formal verification (advanced):** mathematical proof that code satisfies specified properties. Powerful for critical invariants, but expensive and limited to what was specified.

## 5. Operational security: the non-code half of contract security

Many "hacks" are not code exploits at all:

- **Compromised deployer or admin keys:** an attacker steals a developer's key and upgrades a contract or drains a treasury. Multisigs, hardware wallets, and key rotation prevent most of this.
- **Phished frontend:** users visit the real protocol's website, but a compromised frontend sends their transactions to an attacker's contract. Always verify contract addresses and transaction details in your wallet before signing.
- **Malicious governance proposals:** an attacker passes a proposal that looks harmless but changes a contract's rules. Read governance proposals, or delegate to someone who does.
- **Supply chain attacks:** a compromised dependency, library, or deployment tool injects malicious code. Reproducible builds and dependency pinning reduce this risk.

As a user, you cannot control a team's operational security — but you can prefer teams that publish their multisig addresses, timelock parameters, key management practices, and incident response history.

## 6. User checklist: before trusting a contract with meaningful funds

- [ ] Contract source code is verified on the explorer, and I (or someone I trust) have skimmed its key functions.
- [ ] Full audit reports are public, recent, and cover the deployed version; critical findings are fixed, not just acknowledged.
- [ ] Owner and admin controls are understood: who can mint, pause, upgrade, and change fees? Is it a multisig with a timelock?
- [ ] The protocol has operated through a market crash or stress event without failing.
- [ ] A bug bounty exists, and the team has a public incident response history.
- [ ] Total value locked is real and distributed, not inflated by the team's own tokens.
- [ ] My exposure is capped: I am not putting funds I cannot afford to lose into a single protocol.

If most boxes are unchecked, the protocol is not ready for meaningful funds, however attractive its yield looks.

## 7. If a protocol you use is hacked

1. **Do not panic-transact.** Attackers often deploy phishing sites within minutes of a hack announcement, targeting users trying to "rescue" funds.
2. **Revoke approvals** for the compromised contract, from a clean device, after verifying the revocation tool's address.
3. **Move unaffected funds** to a fresh wallet if you suspect your wallet or approvals are compromised.
4. **Follow only official announcements** from the project's verified channels. Ignore DMs offering recovery.
5. **Preserve evidence:** transaction hashes, timestamps, and screenshots, in case of later compensation or investigation.
6. **Do not pay a "recovery service".** It is almost certainly a second scam.

## 8. Frequently asked questions

**Does "audited by a famous firm" mean safe?**
No. It means a reputable firm reviewed a specific version at a specific time and found (and hopefully fixed) certain issues. It is a positive signal, not a guarantee.

**Can I audit a contract myself without coding?**
Partially. You can check verification, owner functions, holder distribution, audit reports, and operational transparency, using an explorer — the checks in section 6. Full code auditing requires programming and security expertise.

**Are newer contracts riskier than older ones?**
Generally, yes. Older contracts that have held large value through crashes have survived real adversarial testing. New contracts have not, however clean their audits look. This is the "battle-tested" principle.

**What is the single most common user-side mistake?**
Signing approvals and transactions without reading them, on sites reached from links. Contract security cannot protect a user who approves a malicious contract.

> Educational content only. This is not financial, legal, or investment advice.
