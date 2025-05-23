# Chapter 11: The Attacker's Gambit - Calculations and Security

The Bitcoin software was stabilizing. The simulations ran smoothly, the P2P network exchanged blocks, and the incentive structure seemed sound. Yet, Satoshi, ever the meticulous engineer and skeptical academic, couldn't shake a persistent unease. The double-spend attack, the Achilles' heel of any digital cash system, needed to be not just addressed, but mathematically dissected and quantified. It wasn't enough to *believe* that an attacker trying to reverse a transaction by creating a longer secret chain would fail; Satoshi needed to *prove* it, or at least calculate the odds with cold, hard numbers.

Their small apartment, already a testament to focused intellectual labor, now transformed into the den of a probability theorist. Stacks of books appeared on the desk: Feller's "An Introduction to Probability Theory and Its Applications," Knuth's "The Art of Computer Programming" (for its rigorous mathematical underpinnings), and various academic papers on random walks and statistical modeling. The whiteboard, previously covered in network diagrams and code snippets, was now a canvas for equations, probability trees, and distribution curves.

Satoshi focused on the scenario: an attacker makes a payment to a merchant. The merchant waits for some number of confirmations (`z` blocks) before considering the payment final. In the meantime, the attacker is secretly trying to build an alternative chain, starting from the block *before* their payment, in which the payment to the merchant is replaced with a payment back to themselves. If the attacker can build their secret chain longer than the honest chain *before* the merchant releases the goods or services, they succeed in defrauding the merchant.

"It's a race," Satoshi muttered, pacing the worn floorboards. "A race between the honest network and the attacker." They recognized the pattern: it was analogous to the classic "Gambler's Ruin" problem. Imagine two gamblers, one with a finite amount of money (the attacker, who is `z` blocks behind) and one with an effectively infinite amount (the honest chain, which will keep growing). What's the probability the gambler with finite resources can overcome their deficit and "ruin" the other?

Satoshi defined the variables:
*   `p`: the probability that an honest node finds the next block.
*   `q`: the probability that the attacker (or a coalition of attackers) finds the next block.
*   `z`: the number of blocks the attacker is currently behind (i.e., the number of confirmations the merchant is waiting for).

If `p > q` – meaning the honest hashing power is greater than the attacker's – intuition suggested the attacker should eventually fall further and further behind. But "eventually" wasn't good enough. Satoshi needed specifics. They recalled a formula from Feller's work, or perhaps derived it anew from first principles on their whiteboard: the probability that an attacker, starting `z` blocks behind, *ever* catches up to the honest chain is `(q/p)^z`.

"If the attacker controls 10% of the network's hashing power (`q = 0.1`), and the honest network controls the rest (`p = 0.9`)," Satoshi calculated, scribbling on a notepad, "then `q/p` is approximately `0.111`. If the merchant waits for just `z = 1` confirmation, the attacker's chance of ever catching up (and thus potentially reversing the transaction in the *next* block) is about 11.1%. Not great."

But what if `z` increased?
*   For `z = 2`, `(0.111)^2` is about 1.23%.
*   For `z = 6` (a number that was beginning to feel like a good rule of thumb), `(0.111)^6` is approximately 0.00017% – less than two in a million.

This was promising. The probability dropped exponentially with each confirmation. However, this calculation was for the attacker *ever* catching up. A merchant is concerned about the attacker catching up *within a specific timeframe* – specifically, while the attacker is still `z` blocks behind.

Satoshi realized that the number of blocks an attacker could generate while the honest network generated `z` blocks would follow a Poisson distribution. The expected number of blocks the attacker would find is `lambda = z * (q/p)`. The probability of them finding `k` blocks is `(lambda^k * e^-lambda) / k!`.

The final piece of the puzzle was to combine these two insights. For each possible number of blocks (`k`) that the attacker might find while the honest chain grows by `z` blocks, what is the probability that the attacker, now `z-k` blocks ahead of where they started (relative to their own chain), can catch up from that new position?

This led to the formulation of the sum published in the whitepaper:

`P = 1 - Sum_{k=0}^{z} [ ( (z * q/p)^k * e^(-z*q/p) ) / k! * (1 - (q/p)^(z-k) if k < z else 1 if k >= z) ]`
(Satoshi simplified this in the paper, focusing on the probability the attacker *can still catch up*).

Late nights were spent translating this mathematical formula into C code. The glow of the monitor reflected in Satoshi's focused eyes as they typed, compiled, and ran simulations. They created a small program that took `q` (attacker's fraction of hash power) and `z` (confirmations) as input and spat out the probability of a successful double-spend.

```c
// (A simplified representation of Satoshi's thought process for the code)
double AttackerSuccessProbability(double q, int z) {
    double p = 1.0 - q;
    double lambda = z * (q / p);
    double sum = 1.0; // Summing the probabilities of the attacker *not* catching up
    for (int k = 0; k <= z; k++) {
        double poisson = exp(-lambda);
        for (int i = 1; i <= k; i++) {
            poisson *= lambda / i;
        }
        // Probability attacker *cannot* catch up if they're (z-k) behind
        double prob_attacker_fails_eventually = 1.0 - pow(q / p, z - k);
        if (k >= z) { // If attacker found as many or more blocks
             prob_attacker_fails_eventually = 0.0; // They have caught up or surpassed
        }
        sum -= poisson * prob_attacker_fails_eventually;
    }
    return sum; // This would be probability of success
}
```
(The actual code in the whitepaper is more direct in calculating the success probability from the Poisson distribution values where k > z).

Satoshi meticulously generated the tables that would appear in the whitepaper. For `q = 0.1`:
*   `z=0, P=1.0000000` (Attacker always wins if merchant accepts 0 confirmations)
*   `z=1, P=0.2045873`
*   `z=5, P=0.0024295`
*   `z=10, P=0.0000056` (This is the P_sub_10 value from the paper)

For `q = 0.3` (a significant attacker, controlling 30% of the network):
*   `z=5, P=0.0416822`
*   `z=10, P=0.0049419`
*   `z=15, P=0.0007082`

A profound sense of relief and vindication washed over Satoshi. The numbers didn't lie. Even with a substantial fraction of the network's hashing power, an attacker faced rapidly diminishing odds of success with each confirmation. The "wait for 6 confirmations" rule of thumb wasn't arbitrary; it was grounded in solid probability. For typical transaction values, waiting for 5 or 6 blocks made a successful double-spend incredibly unlikely, provided the honest network held the majority of hashing power.

The system was not just intuitively secure; it was *demonstrably* secure against this specific, critical attack vector. The gambler, if rational and facing these odds, would not take the bet. Satoshi saved the C code and the generated tables. This analysis would be a cornerstone of the whitepaper, a clear signal to the world that Bitcoin was built on a foundation of mathematical rigor. The hum of the computer fans seemed a little quieter, the weight on Satoshi's shoulders a little lighter. The attacker's gambit was, by the numbers, a losing one.
