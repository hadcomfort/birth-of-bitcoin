# Bitcoin: A Peer-to-Peer Electronic Cash System

Satoshi Nakamoto  
[satoshin@gmx.com](mailto:satoshin@gmx.com)  
www.bitcoin.org

**Abstract.** A purely peer-to-peer version of electronic cash would allow online payments to be sent directly from one party to another without going through a financial institution. Digital signatures provide part of the solution, but the main benefits are lost if a trusted third party is still required to prevent double-spending. We propose a solution to the double-spending problem using a peer-to-peer network. The network timestamps transactions by hashing them into an ongoing chain of hash-based proof-of-work, forming a record that cannot be changed without redoing the proof-of-work. The longest chain not only serves as proof of the sequence of events witnessed, but proof that it came from the largest pool of CPU power. As long as a majority of CPU power is controlled by nodes that are not cooperating to attack the network, they'll generate the longest chain and outpace attackers. The network itself requires minimal structure. Messages are broadcast on a best effort basis, and nodes can leave and rejoin the network at will, accepting the longest proof-of-work chain as proof of what happened while they were gone.

## 1. Introduction

Commerce on the Internet has come to rely almost exclusively on financial institutions serving as trusted third parties to process electronic payments. While the system works well enough for most transactions, it still suffers from the inherent weaknesses of the trust based model. Completely non-reversible transactions are not really possible, since financial institutions cannot avoid mediating disputes. The cost of mediation increases transaction costs, limiting the minimum practical transaction size and cutting off the possibility for small casual transactions, and there is a broader cost in the loss of ability to make non-reversible payments for non-reversible services. With the possibility of reversal, the need for trust spreads. Merchants must be wary of their customers, hassling them for more information than they would otherwise need. A certain percentage of fraud is accepted as unavoidable. These costs and payment uncertainties can be avoided in person by using physical currency, but no mechanism exists to make payments over a communications channel without a trusted party.

What is needed is an electronic payment system based on cryptographic proof instead of trust, allowing any two willing parties to transact directly with each other without the need for a trusted third party. Transactions that are computationally impractical to reverse would protect sellers from fraud, and routine escrow mechanisms could easily be implemented to protect buyers. In this paper, we propose a solution to the double-spending problem using a peer-to-peer distributed timestamp server to generate computational proof of the chronological order of transactions. The system is secure as long as honest nodes collectively control more CPU power than any cooperating group of attacker nodes.

## 2. Transactions

We define an electronic coin as a chain of digital signatures. Each owner transfers the coin to the next by digitally signing a hash of the previous transaction and the public key of the next owner and adding these to the end of the coin. A payee can verify the signatures to verify the chain of ownership.

```
      ┌─────────────────────┐               ┌─────────────────────┐              ┌─────────────────────┐
      │                     │               │                     │              │                     │
      │    Transaction      │               │    Transaction      │              │    Transaction      │
      │                     │               │                     │              │                     │
      │   ┌─────────────┐   │               │   ┌─────────────┐   │              │   ┌─────────────┐   │
      │   │ Owner 1's   │   │               │   │ Owner 2's   │   │              │   │ Owner 3's   │   │
      │   │ Public Key  │   │               │   │ Public Key  │   │              │   │ Public Key  │   │
      │   └───────┬─────┘   │               │   └───────┬─────┘   │              │   └───────┬─────┘   │
      │           │    .    │               │           │    .    │              │           │         │
──────┼─────────┐ │    .    ├───────────────┼─────────┐ │    .    ├──────────────┼─────────┐ │         │
      │         │ │    .    │               │         │ │    .    │              │         │ │         │
      │      ┌──▼─▼──┐ .    │               │      ┌──▼─▼──┐ .    │              │      ┌──▼─▼──┐      │
      │      │ Hash  │ .    │               │      │ Hash  │ .    │              │      │ Hash  │      │
      │      └───┬───┘ .    │    Verify     │      └───┬───┘ .    │    Verify    │      └───┬───┘      │
      │          │     ............................    │     ...........................    │          │
      │          │          │               │     │    │          │              │     │    │          │
      │   ┌──────▼──────┐   │               │   ┌─▼────▼──────┐   │              │   ┌─▼────▼──────┐   │
      │   │ Owner 0's   │   │      Sign     │   │ Owner 1's   │   │      Sign    │   │ Owner 2's   │   │
      │   │ Signature   │   │      ...........─►│ Signature   │   │     ...........─►│ Signature   │   │
      │   └─────────────┘   │      .        │   └─────────────┘   │     .        │   └─────────────┘   │
      │                     │      .        │                     │     .        │                     │
      └─────────────────────┘      .        └─────────────────────┘     .        └─────────────────────┘
                                   .                                    .
          ┌─────────────┐          .            ┌─────────────┐         .            ┌─────────────┐
          │ Owner 1's   │...........            │ Owner 2's   │..........            │ Owner 3's   │
          │ Private Key │                       │ Private Key │                      │ Private Key │
          └─────────────┘                       └─────────────┘                      └─────────────┘
```

The problem of course is the payee can't verify that one of the owners did not double-spend the coin. A common solution is to introduce a trusted central authority, or mint, that checks every transaction for double spending. After each transaction, the coin must be returned to the mint to issue a new coin, and only coins issued directly from the mint are trusted not to be double-spent. The problem with this solution is that the fate of the entire money system depends on the company running the mint, with every transaction having to go through them, just like a bank.

We need a way for the payee to know that the previous owners did not sign any earlier transactions. For our purposes, the earliest transaction is the one that counts, so we don't care about later attempts to double-spend. The only way to confirm the absence of a transaction is to be aware of all transactions. In the mint based model, the mint was aware of all transactions and decided which arrived first. To accomplish this without a trusted party, transactions must be publicly announced [^1], and we need a system for participants to agree on a single history of the order in which they were received. The payee needs proof that at the time of each transaction, the majority of nodes agreed it was the first received.

## Part 1, Chapter 3: The Unwavering Clock (Timestamp Server)

Elara, a historian of digital ages, often found herself lost in the labyrinthine archives of the early internet. Her current obsession: the elusive concept of a truly decentralized currency. She'd read countless theories, but none had truly addressed the fundamental flaw: how to prevent someone from spending the same digital coin twice. The banks, with their centralized ledgers, solved it easily enough, but at what cost? Trust. A trust that could be betrayed, manipulated, or simply fail.

Then she found it, buried deep in a forgotten corner of a digital library: a proposal for a "timestamp server." At first, it seemed deceptively simple. A server that would take a "hash" – a unique digital fingerprint – of a block of items, say, a collection of transactions, and then publish that hash widely. Like a newspaper printing the day's headlines, but for data. "But what does that prove?" she muttered to herself, tracing the diagrams on her screen.

The genius, she soon realized, lay in the chain. Each new hash wouldn't just be a snapshot of its own block; it would also incorporate the hash of the *previous* block. This created an unbreakable, chronological chain. If you changed anything in an old block, even a single digit, its hash would change. That would, in turn, change the hash of the next block, and the next, all the way to the present. It was like a digital domino effect. To alter history, you'd have to re-calculate every single hash that came after your alteration. The timestamp wasn't just a mark in time; it was a commitment, a digital anchor that proved the data existed *at that moment*, and that it hadn't been tampered with since. Elara felt a thrill. This wasn't just a server; it was an unwavering clock, a digital witness to the unfolding of events, built not on trust, but on cryptographic certainty.

## Part 1, Chapter 4: The Labor of Trust (Proof-of-Work)

The timestamp server was a revelation, but a new question emerged: who would run these servers? And how could one ensure they were honest? A centralized timestamp server would simply reintroduce the very problem they were trying to solve. The answer, Elara discovered, lay in a concept as ancient as human endeavor: labor. But this was labor of a new kind – computational labor.

The whitepaper spoke of "proof-of-work," a system where the right to add a new block to the chain wasn't granted by authority, but earned through effort. Imagine, the text suggested, a colossal digital puzzle. To solve it, you had to find a specific number, a "nonce," that when combined with the block's data and then hashed, would produce a result with a very particular characteristic – for instance, starting with a certain number of zeros. This wasn't about intelligence; it was about brute force, about trying countless numbers until you stumbled upon the right one.

"It's like digging for gold," Elara mused, "but instead of a pickaxe, you have a supercomputer, and instead of gold, you're digging for a specific pattern in a hash." The average work required to find this "golden nonce" was immense, exponentially increasing with the number of zeros required. But once found, verifying it was trivial – a single hash calculation. This asymmetry was key. It was hard to create, but easy to check.

This computational labor, this "proof-of-work," served a dual purpose. Firstly, it made altering past blocks incredibly difficult. If an attacker wanted to change a transaction in an old block, they wouldn't just have to re-calculate that block's hash; they'd have to re-do the proof-of-work for *every single block* that came after it. The sheer computational power required would be astronomical, making such an attack practically impossible as the chain grew longer. Secondly, it solved the problem of majority decision-making. In a network without central control, how do you decide what the "truth" is? The whitepaper's answer was elegant: "one-CPU-one-vote." The longest chain, the one with the most accumulated proof-of-work, was the one that represented the majority consensus, the one that had the most computational effort invested in it. It was a democratic system, not of people, but of processing power, ensuring that as long as honest participants controlled the majority of the network's computing strength, the true history would always prevail.

## Part 1, Chapter 5: The Whispering Network (Network)

The pieces were beginning to fall into place for Elara. The digital signatures provided ownership, the timestamp server ensured chronology, and proof-of-work secured the history. But how did all these disparate elements coalesce into a functioning system? The answer lay in the "network" itself – a decentralized web of interconnected nodes, each playing a vital role.

She imagined a bustling digital marketplace. When a new transaction occurred – say, Alice sending a coin to Bob – it wasn't sent to a bank. Instead, it was "broadcast" across this network, a whisper carried on digital winds to every listening node. Each node, like a diligent scribe, collected these new transactions, bundling them together into a "block."

Then came the race. Every node, armed with its computational power, began the arduous task of finding the difficult proof-of-work for its own block. It was a silent, relentless competition. When one node, through sheer computational luck and effort, finally found the elusive nonce, it didn't keep it secret. No, it immediately broadcast its newly validated block to all other nodes.

The other nodes, upon receiving this new block, performed a quick verification: were all the transactions valid? Had any of the coins already been spent? If everything checked out, they accepted the block. Their acceptance wasn't just passive; it was active. They immediately began working on the *next* block in the chain, using the hash of the newly accepted block as their starting point. This act of building upon the longest chain was their "vote" of confidence, their affirmation of the network's shared history.

What if two nodes found a valid block at roughly the same time? Elara wondered. The whitepaper explained that some nodes might receive one version first, others the other. They'd work on the first one they received, but keep the alternative in mind. The tie would be broken when the *next* proof-of-work was found. Whichever branch became longer would be adopted by all nodes, and those working on the shorter branch would gracefully switch over. It was a self-correcting mechanism, ensuring that the network always converged on a single, agreed-upon history. The beauty, Elara realized, was in its resilience. Messages didn't need to reach every single node instantly; as long as they reached many, they'd eventually be included. And if a node missed a block, it would simply request it when it saw the next one in the chain. This wasn't a fragile, centralized system; it was a robust, self-organizing organism.

## Part 1, Chapter 6: The Golden Reward (Incentive)

Elara had traced the technical marvel of this new system, but a nagging question remained: why? Why would anyone dedicate their precious computational resources, their electricity, their time, to maintaining this intricate network? The whitepaper, with its characteristic directness, addressed this head-on: "Incentive."

It was a stroke of genius, she thought. The very first transaction within each newly validated block wasn't a payment from one user to another. Instead, it was a special transaction that "started a new coin," a fresh piece of the digital currency, and awarded it directly to the creator of that block – the node that had successfully found the proof-of-work. This was the "mining" process, analogous to gold miners expending their resources to unearth precious metals. Here, the resource expended was CPU time and electricity, transformed into new digital wealth.

"So, they're paid to secure the network," Elara murmured, a smile spreading across her face. This wasn't charity; it was a carefully designed economic engine. The steady, predictable addition of new coins into circulation provided a powerful motivation for nodes to participate, to dedicate their computing power, and crucially, to play by the rules.

Beyond the newly minted coins, there was another layer of incentive: transaction fees. If a user sent a payment where the input value was slightly higher than the output value, the difference wasn't lost; it was added to the incentive value of the block containing that transaction. This meant that as the system matured and perhaps the rate of new coin creation diminished, the network could transition to being entirely supported by these fees, becoming "completely inflation free" once a predetermined number of coins had entered circulation.

The incentive mechanism also served as a powerful deterrent against malicious behavior. If a "greedy attacker" managed to amass more computational power than all the honest nodes combined, they would face a choice. They could use their immense power to try and defraud the system, perhaps by double-spending their own coins. Or, they could use that same power to honestly generate new coins, earning a larger share of the rewards than anyone else. The whitepaper argued that it would be "more profitable to play by the rules," as undermining the system would ultimately devalue their own wealth. The very design of the system encouraged honesty, aligning the self-interest of participants with the integrity and security of the entire network. It was a self-sustaining ecosystem, powered by the promise of a golden reward.

## 7. Reclaiming Disk Space

Once the latest transaction in a coin is buried under enough blocks, the spent transactions before it can be discarded to save disk space. To facilitate this without breaking the block's hash, transactions are hashed in a Merkle Tree [^7] [^2] [^5], with only the root included in the block's hash. Old blocks can then be compacted by stubbing off branches of the tree. The interior hashes do not need to be stored.

```
┌──────────────────────────────────────────┐    ┌──────────────────────────────────────────┐
│                                          │    │                                          │
│ Block ┌─────────────────────────────┐    │    │ Block ┌─────────────────────────────┐    │
│       │  Block Header (Block Hash)  │    │    │       │  Block Header (Block Hash)  │    │
│       │ ┌────────────┐ ┌─────────┐  │    │    │       │ ┌────────────┐ ┌─────────┐  │    │
│       │ │ Prev Hash  │ │ Nonce   │  │    │    │       │ │ Prev Hash  │ │ Nonce   │  │    │
│       │ └────────────┘ └─────────┘  │    │    │       │ └────────────┘ └─────────┘  │    │
│       │                             │    │    │       │                             │    │
│       │     ┌─────────────┐         │    │    │       │     ┌─────────────┐         │    │
│       │     │  Root Hash  │         │    │    │       │     │  Root Hash  │         │    │
│       │     └─────▲─▲─────┘         │    │    │       │     └─────▲─▲─────┘         │    │
│       │           │ │               │    │    │       │           │ │               │    │
│       │           │ │               │    │    │       │           │ │               │    │
│       └───────────┼─┼───────────────┘    │    │       └───────────┼─┼───────────────┘    │
│                   │ │                    │    │                   │ │                    │
│     ..........    │ │     ..........     │    │     ┌────────┐    │ │     ..........     │
│     .        ─────┘ └─────.        .     │    │     │        ├────┘ └─────.        .     │
│     . Hash01 .            . Hash23 .     │    │     │ Hash01 │            . Hash23 .     │
│     .▲.....▲..            .▲.....▲..     │    │     │        │            .▲.....▲..     │
│      │     │               │     │       │    │     └────────┘             │     │       │
│      │     │               │     │       │    │                            │     │       │
│      │     │               │     │       │    │                            │     │       │
│ .....│.. ..│.....     .....│.. ..│.....  │    │                       ┌────┴─┐ ..│.....  │
│ .      . .      .     .      . .      .  │    │                       │      │ .      .  │
│ .Hash0 . .Hash1 .     .Hash2 . .Hash3 .  │    │                       │Hash2 │ .Hash3 .  │
│ ...▲.... ...▲....     ...▲.... ...▲....  │    │                       │      │ .      .  │
│    │        │            │        │      │    │                       └──────┘ ...▲....  │
│    │        │            │        │      │    │                                   │      │
│    │        │            │        │      │    │                                   │      │
│ ┌──┴───┐ ┌──┴───┐     ┌──┴───┐ ┌──┴───┐  │    │                                ┌──┴───┐  │
│ │ Tx0  │ │ Tx1  │     │ Tx2  │ │ Tx3  │  │    │                                │ Tx3  │  │
│ └──────┘ └──────┘     └──────┘ └──────┘  │    │                                └──────┘  │
│                                          │    │                                          │
└──────────────────────────────────────────┘    └──────────────────────────────────────────┘
     Transactions Hashed in a Merkle Tree              After Pruning Tx0-2 from the Block
```

A block header with no transactions would be about 80 bytes. If we suppose blocks are generated every 10 minutes, 80 bytes * 6 * 24 * 365 = 4.2MB per year. With computer systems typically selling with 2GB of RAM as of 2008, and Moore's Law predicting current growth of 1.2GB per year, storage should not be a problem even if the block headers must be kept in memory.

## 8. Simplified Payment Verification

It is possible to verify payments without running a full network node. A user only needs to keep a copy of the block headers of the longest proof-of-work chain, which he can get by querying network nodes until he's convinced he has the longest chain, and obtain the Merkle branch linking the transaction to the block it's timestamped in. He can't check the transaction for himself, but by linking it to a place in the chain, he can see that a network node has accepted it, and blocks added after it further confirm the network has accepted it.

```
     Longest Proof-of-Work Chain
        ┌────────────────────────────────────────┐      ┌────────────────────────────────────────┐       ┌────────────────────────────────────────┐
        │   Block Header                         │      │   Block Header                         │       │   Block Header                         │
        │  ┌──────────────────┐ ┌──────────────┐ │      │  ┌──────────────────┐ ┌──────────────┐ │       │  ┌──────────────────┐ ┌──────────────┐ │
 ───────┼─►│ Prev Hash        │ │ Nonce        │ ├──────┼─►│ Prev Hash        │ │ Nonce        │ ├───────┼─►│ Prev Hash        │ │ Nonce        │ ├────────►
        │  └──────────────────┘ └──────────────┘ │      │  └──────────────────┘ └──────────────┘ │       │  └──────────────────┘ └──────────────┘ │
        │                                        │      │                                        │       │                                        │
        │     ┌───────────────────┐              │      │    ┌────────────────────┐              │       │     ┌───────────────────┐              │
        │     │   Merkle Root     │              │      │    │   Merkle Root      │              │       │     │   Merkle Root      │              │
        │     └───────────────────┘              │      │    └────────▲─▲─────────┘              │       │     └───────────────────┘              │
        │                                        │      │             │ │                        │       │                                        │
        └────────────────────────────────────────┘      └─────────────┼─┼────────────────────────┘       └────────────────────────────────────────┘
                                                                      │ │
                                                                      │ │
                                                        ┌────────┐    │ │     ..........
                                                        │        ├────┘ └─────.        .
                                                        │ Hash01 │            . Hash23 .
                                                        │        │            .▲.....▲..
                                                        └────────┘             │     │
                                                                               │     │
                                                                               │     │   Merkle Branch for Tx3
                                                                               │     │
                                                                         ┌─────┴─┐ ..│.....
                                                                         │       │ .      .
                                                                         │ Hash2 │ .Hash3 .
                                                                         │       │ .      .
                                                                         └───────┘ ...▲....
                                                                                      │
                                                                                      │
                                                                                  ┌───┴───┐
                                                                                  │  Tx3  │
                                                                                  └───────┘
```

As such, the verification is reliable as long as honest nodes control the network, but is more vulnerable if the network is overpowered by an attacker. While network nodes can verify transactions for themselves, the simplified method can be fooled by an attacker's fabricated transactions for as long as the attacker can continue to overpower the network. One strategy to protect against this would be to accept alerts from network nodes when they detect an invalid block, prompting the user's software to download the full block and alerted transactions to confirm the inconsistency. Businesses that receive frequent payments will probably still want to run their own nodes for more independent security and quicker verification.

## 9. Combining and Splitting Value

Although it would be possible to handle coins individually, it would be unwieldy to make a separate transaction for every cent in a transfer. To allow value to be split and combined, transactions contain multiple inputs and outputs. Normally there will be either a single input from a larger previous transaction or multiple inputs combining smaller amounts, and at most two outputs: one for the payment, and one returning the change, if any, back to the sender.

```
     ┌──────────────────────┐
     │ Transaction          │
     │                      │
     │   ┌─────┐  ┌─────┐   │
─────┼──►│ in  │  │ out │ ──┼─────►
     │   └─────┘  └─────┘   │
     │                      │
     │                      │
     │   ┌─────┐  ┌─────┐   │
─────┼──►│ in  │  │ ... │ ──┼─────►
     │   └─────┘  └─────┘   │
     │                      │
     │                      │
     │   ┌─────┐            │
─────┼──►│...  │            │
     │   └─────┘            │
     │                      │
     └──────────────────────┘
```
It should be noted that fan-out, where a transaction depends on several transactions, and those transactions depend on many more, is not a problem here. There is never the need to extract a complete standalone copy of a transaction's history.

## 10. Privacy

The traditional banking model achieves a level of privacy by limiting access to information to the parties involved and the trusted third party. The necessity to announce all transactions publicly precludes this method, but privacy can still be maintained by breaking the flow of information in another place: by keeping public keys anonymous. The public can see that someone is sending an amount to someone else, but without information linking the transaction to anyone. This is similar to the level of information released by stock exchanges, where the time and size of individual trades, the "tape", is made public, but without telling who the parties were.

```
Traditional Privacy Models                                                │
                                      ┌─────────────┐   ┌──────────────┐  │  ┌────────┐
┌──────────────┐  ┌──────────────┐    │  Trusted    │   │              │  │  │        │
│  Identities  ├──┤ Transactions ├───►│ Third Party ├──►│ Counterparty │  │  │ Public │
└──────────────┘  └──────────────┘    │             │   │              │  │  │        │
                                      └─────────────┘   └──────────────┘  │  └────────┘

New Privacy Model
                                       ┌────────┐
┌──────────────┐ │ ┌──────────────┐    │        │
│  Identities  │ │ │ Transactions ├───►│ Public │
└──────────────┘ │ └──────────────┘    │        │
                                       └────────┘
```
As an additional firewall, a new key pair should be used for each transaction to keep them from being linked to a common owner. Some linking is still unavoidable with multi-input transactions, which necessarily reveal that their inputs were owned by the same owner. The risk is that if the owner of a key is revealed, linking could reveal other transactions that belonged to the same owner.

## 11. Calculations
We consider the scenario of an attacker trying to generate an alternate chain faster than the honest chain. Even if this is accomplished, it does not throw the system open to arbitrary changes, such as creating value out of thin air or taking money that never belonged to the attacker. Nodes are not going to accept an invalid transaction as payment, and honest nodes will never accept a block containing them. An attacker can only try to change one of his own transactions to take back money he recently spent.

The race between the honest chain and an attacker chain can be characterized as a Binomial Random Walk. The success event is the honest chain being extended by one block, increasing its lead by +1, and the failure event is the attacker's chain being extended by one block, reducing the gap by -1.

The probability of an attacker catching up from a given deficit is analogous to a Gambler's Ruin problem. Suppose a gambler with unlimited credit starts at a deficit and plays potentially an infinite number of trials to try to reach breakeven. We can calculate the probability he ever reaches breakeven, or that an attacker ever catches up with the honest chain, as follows [^8]:

```plaintext
p = probability an honest node finds the next block<
q = probability the attacker finds the next block
q = probability the attacker will ever catch up from z blocks behind
``````
     
$$
qz = 
\begin{cases} 
1 & \text{if } p \leq q \\
\left(\frac{q}{p}\right) z & \text{if } p > q 
\end{cases}
$$

Given our assumption that p > q, the probability drops exponentially as the number of blocks the attacker has to catch up with increases. With the odds against him, if he doesn't make a lucky lunge forward early on, his chances become vanishingly small as he falls further behind. 

We now consider how long the recipient of a new transaction needs to wait before being sufficiently certain the sender can't change the transaction. We assume the sender is an attacker who wants to make the recipient believe he paid him for a while, then switch it to pay back to himself after some time has passed. The receiver will be alerted when that happens, but the sender hopes it will be too late.

The receiver generates a new key pair and gives the public key to the sender shortly before signing. This prevents the sender from preparing a chain of blocks ahead of time by working on it continuously until he is lucky enough to get far enough ahead, then executing the transaction at that moment. Once the transaction is sent, the dishonest sender starts working in secret on a parallel chain containing an alternate version of his transaction.

The recipient waits until the transaction has been added to a block and z blocks have been linked after it. He doesn't know the exact amount of progress the attacker has made, but assuming the honest blocks took the average expected time per block, the attacker's potential progress will be a Poisson distribution with expected value:

$$
\lambda = z\frac{q}{p}
$$

To get the probability the attacker could still catch up now, we multiply the Poisson density for each amount of progress he could have made by the probability he could catch up from that point:

$$
\sum_{k=0}^{\infty} \frac{\lambda^k e^{-\lambda}}{k!} \cdot \left\{ 
\begin{array}{cl} 
\left(\frac{q}{p}\right)^{(z-k)} & \text{if } k \leq z \\
1 & \text{if } k > z 
\end{array}
\right.
$$

Rearranging to avoid summing the infinite tail of the distribution...

$$
1 - \sum_{k=0}^{z} \frac{\lambda^k e^{-\lambda}}{k!} \left(1-\left(\frac{q}{p}\right)^{(z-k)}\right)
$$

Converting to C code...

```c
#include <math.h>

double AttackerSuccessProbability(double q, int z)
{
    double p = 1.0 - q;
    double lambda = z * (q / p);
    double sum = 1.0;
    int i, k;
    for (k = 0; k <= z; k++)
    {
        double poisson = exp(-lambda);
        for (i = 1; i <= k; i++)
            poisson *= lambda / i;
        sum -= poisson * (1 - pow(q / p, z - k));
    }
    return sum;
}
```
Running some results, we can see the probability drop off exponentially with z.

```plaintext
q=0.1
z=0 P=1.0000000
z=1 P=0.2045873
z=2 P=0.0509779
z=3 P=0.0131722
z=4 P=0.0034552
z=5 P=0.0009137
z=6 P=0.0002428
z=7 P=0.0000647
z=8 P=0.0000173
z=9 P=0.0000046
z=10 P=0.0000012

q=0.3
z=0 P=1.0000000
z=5 P=0.1773523
z=10 P=0.0416605
z=15 P=0.0101008
z=20 P=0.0024804
z=25 P=0.0006132
z=30 P=0.0001522
z=35 P=0.0000379
z=40 P=0.0000095
z=45 P=0.0000024
z=50 P=0.0000006
```
Solving for P less than 0.1%...
```plaintext
P < 0.001
q=0.10 z=5
q=0.15 z=8
q=0.20 z=11
q=0.25 z=15
q=0.30 z=24
q=0.35 z=41
q=0.40 z=89
q=0.45 z=340
```
## 12. Conclusion
We have proposed a system for electronic transactions without relying on trust. We started with the usual framework of coins made from digital signatures, which provides strong control of ownership, but is incomplete without a way to prevent double-spending. To solve this, we proposed a peer-to-peer network using proof-of-work to record a public history of transactions that quickly becomes computationally impractical for an attacker to change if honest nodes control a majority of CPU power. The network is robust in its unstructured simplicity. Nodes work all at once with little coordination. They do not need to be identified, since messages are not routed to any particular place and only need to be delivered on a best effort basis. Nodes can leave and rejoin the network at will, accepting the proof-of-work chain as proof of what happened while they were gone. They vote with their CPU power, expressing their acceptance of valid blocks by working on extending them and rejecting invalid blocks by refusing to work on them. Any needed rules and incentives can be enforced with this consensus mechanism.
<br>

### References
---
[^1]: W. Dai, "b-money," http://www.weidai.com/bmoney.txt, 1998.
[^2]: H. Massias, X.S. Avila, and J.-J. Quisquater, "Design of a secure timestamping service with minimal
trust requirements," In 20th Symposium on Information Theory in the Benelux, May 1999.
[^3]: S. Haber, W.S. Stornetta, "How to time-stamp a digital document," In Journal of Cryptology, vol 3, no
2, pages 99-111, 1991.
[^4]: D. Bayer, S. Haber, W.S. Stornetta, "Improving the efficiency and reliability of digital time-stamping,"
In Sequences II: Methods in Communication, Security and Computer Science, pages 329-334, 1993.
[^5]: S. Haber, W.S. Stornetta, "Secure names for bit-strings," In Proceedings of the 4th ACM Conference
on Computer and Communications Security, pages 28-35, April 1997.
[^6]: A. Back, "Hashcash - a denial of service counter-measure,"
http://www.hashcash.org/papers/hashcash.pdf, 2002.
[^7]: R.C. Merkle, "Protocols for public key cryptosystems," In Proc. 1980 Symposium on Security and
Privacy, IEEE Computer Society, pages 122-133, April 1980.
[^8]: W. Feller, "An introduction to probability theory and its applications," 1957.
