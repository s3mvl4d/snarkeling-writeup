---
layout: default
title: SNARKeling Treasure Hunt - Write-up
---

## Intro
First, I’d like to thank Cyfrin for accepting my codebase for this First Flight contest and for giving me the opportunity to act as a judge. It was a great experience reviewing submissions from participants with different perspectives and approaches, and it provided a valuable look into how developers reason about security in a composable system involving smart contracts, circuits, and generated verifiers.

For reference, the full codebase is available here:
👉 https://github.com/CodeHawks-Contests/2026-04-snarkeling

You can find the contest page here:
👉 https://codehawks.cyfrin.io/c/2026-04-snarkeling

The relevant Cyfrin Updraft course:
👉 https://updraft.cyfrin.io/courses/noir-programming-and-zk-circuits

If you’d like to follow my work or reach out, you can find me on Twitter:
👉 https://x.com/s3mvl4d


This write-up is my summary of the issues that mattered most, the submissions that held up, and the ones that did not. I’ve organized it the way I think these contests are best read: starting with the highest-severity breakages, then working down through medium and low severity issues, and finally covering a few disputed findings that did not survive closer inspection.

---

## Scope and context

The contest centered on a compact layered stack: the Solidity payout contract in `TreasureHunt.sol`, the Noir circuit in `circuits/src/main.nr`, the generated Honk verifier, and the surrounding deployment and fixture-generation scripts. The intended protocol flow was straightforward. A participant finds a real-world treasure, learns a secret, generates a proof off-chain, and submits a claim on-chain to receive a fixed ETH reward. The README explicitly frames the protocol around ten treasures, ten claims, and a funded reward pool, while the contract and circuit are supposed to enforce uniqueness, proof validity, and payout safety.

![alt text](treasure_hunt.png)

What made the contest interesting is that nearly every part of that story contained a fault line. Some were obvious once spotted, like the duplicate-claim bug. Others were more structural, like the gap between the circuit’s intended secrecy model and what the repository actually exposed. A few were operational rather than purely logical, but were still severe enough to matter because a protocol that cannot be deployed or safely operated is broken in a very practical sense. 

---

# High severity findings

## 1. The generated verifier made the protocol undeployable

Before getting into claim logic, it’s worth starting with the most fundamental failure mode of all: a protocol that cannot be deployed cannot secure anything. The verifier in this contest is generated as a monolithic Solidity contract from Barretenberg artifacts, and the attached verifier shows exactly the kind of structure that tends to bloat runtime bytecode - large embedded verification-key constants, extensive transcript logic, relation code, elliptic-curve helpers, and batching logic all packed into one generated contract. Standard EVM chains enforce the EIP-170 contract size cap of **24,576 bytes** for deployed runtime code, and if contract creation returns code larger than that limit, deployment fails.

```bash
forge script Deploy.s.sol --rpc-url http://127.0.0.1:8545  --broadcast
[⠊] Compiling...
No files changed, compilation skipped
Script ran successfully.

== Logs ==
  Chain ID: 31337
  Deployer: 0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266
  Deployer balance: 10002144599182000000000
  Initial funding: 100000000000000000000
  Verifier: 0xe7f1725E7734CE288F8367e1Bb143E90bb3F0512
  TreasureHunt: 0x9fE46736679d2D9a65F0992F2272dE9f3c7fa6e0
  Balance: 100000000000000000000
  Reward: 10000000000000000000
  Max treasures: 10
  Remaining treasures: 10
  Paused: false
Error: `HonkVerifier` is above the contract size limit (25716 > 24576).
Do you wish to continue? yes
```

```bash

    Transaction: 0xd46df53f637b31e5f2ba97ed0024efb1d470707b9bde78a23e84dad5c5ea942a
    Gas used: 1442233

    Block Number: 1
    Block Hash: 0xdd12f6e9429ec3b40d18c5a6f366e799bdbb07b6bdeba606647d0693e231b9e6
    Block Time: "Sun, 10 May 2026 09:19:46 +0000"


    Transaction: 0x24cdd1a786bb9d9303a083ad9e4f6777a1b13b66249e22f9cd4bce4cfe6ab419
    Gas used: 7289412
    Error: reverted with: EvmError: CreateContractSizeLimit
```

That matters here because the generated verifier is not an optional extra; it is a hard dependency of the entire payout system. The deployment script instantiates `new HonkVerifier()` first and then passes its address into the hunt contract, which means the whole protocol depends on the verifier being deployable as-is. If the generated verifier exceeds the EIP-170 size ceiling on the target chain, the deployment pipeline is dead on arrival: no verifier, no hunt contract, no claims, no protocol. In practical terms, that is not a cosmetic issue or a gas optimization note - it is an availability failure that bricks the project at deployment time and therefore deserves to be treated as high severity.

```javascript
verifier = new HonkVerifier();
hunt = new TreasureHunt{value: initialFunding}(address(verifier));
```

## 2. Broken double-claim protection
If the verifier issue was the first crack in the foundation, the duplicate-claim bug was the loudest and most immediately dangerous application-level break. At the heart of `TreasureHunt::claim()` is a uniqueness check that is supposed to ensure each treasure can only be redeemed once. But the contract checks `claimed[_treasureHash]` and later marks `claimed[treasureHash] = true`, meaning the read and the write are performed against different keys. The `_treasureHash` variable is not the caller-supplied treasure identifier being redeemed in the transaction, so the duplicate-claim guard is simply looking in the wrong place. 
That one-line mismatch is enough to break the entire economic promise of the hunt. A valid proof and `treasureHash` are supposed to represent a single discovered treasure, but because the guard never properly detects that the same `treasureHash` was already marked, the same treasure can be claimed again and again until the contract runs out of ETH or reaches its total claim counter. In other words, one successful claim can be turned into repeated withdrawals, draining rewards meant for other treasures and collapsing the “one treasure, one payout” model described in the README. That is exactly the sort of issue that deserves to sit near the top of any contest leaderboard. 

```javascript
if (claimed[_treasureHash]) revert AlreadyClaimed(treasureHash);
...
_markClaimed(treasureHash);
```

## 3. The secret treasures were stored in plain text
The protocol’s story depends on secrecy. The whole point of the Noir proof is that a participant can prove they know the right treasure secret without revealing it. But in this repository, the “secret” is not treated like a secret at all. The deployment script comments literally enumerate the treasure secrets, and the prover fixture file stores the entire treasure array in plaintext alongside the corresponding hashes. This is not an abstract leak or an accidental inference from metadata - the raw witness values are present in the repo.

Once those values are published, the trust model behind the treasure hunt breaks down. The circuit still proves knowledge of a witness, but if the witness is already sitting in source-controlled plaintext, anyone with repo access can produce a valid proof without ever discovering a treasure in the water. At that point, the ZK circuit is no longer protecting the real-world game; it is just certifying knowledge of data that the project itself disclosed. For a protocol whose entire premise depends on hidden secrets, that is not a footnote. It is a direct break of the core design assumption.

```bash
// Secret Treasures for the snorkeling hunt (not revealed to the public):
// 1, 2, 3, 4, 5, 6, 7, 8, 9, 10
```

## 4. The secrets were also brute-forceable even without the plaintext leak
Even if the plaintext disclosure had been cleaned up, the contest still had a deeper problem: the secret space itself was tiny. The circuit proves that the public `treasure_hash` matches `pedersen_hash([treasure])`, but the candidate treasure values were just small scalar integers. In effect, the supposedly hidden witness lives in a laughably small search space, which means the secrecy is not computationally meaningful. An attacker does not need privileged access if they can simply enumerate plausible values, hash them, and match the public hash offline.
That makes this bug distinct from the plaintext leak. The plaintext issue says, “the secret was already published.” The brute-force issue says, “even if it had not been published, it still would not have been secret enough.” Both conclusions independently damage the same trust assumption, and together they show that the protocol’s security did not rest on a serious witness design in the first place. When the core application model is “find a secret, prove you know it, get paid,” a witness space that can be exhaustively searched with negligible effort is a high-severity design failure.

```rust
assert(std::hash::pedersen_hash([treasure]) == treasure_hash);
```

## 5. The deployer private key was read from an unencrypted environment variable
The operational security story had its own major flaw. The deployment flow reads the raw deployer private key directly via `vm.envUint("PRIVATE_KEY")` and feeds that value into `vm.startBroadcast(deployerKey)`. In Foundry’s own scripting documentation, this pattern is possible, but it sits alongside safer alternatives like encrypted keystores, and Foundry maintainers have explicitly described storing plaintext private keys in working-directory files as an anti-pattern compared with keystore-backed flows. 
Why does this rise to high severity? Because the deployer key is not some harmless local convenience. It is the key that deploys the verifier, deploys and funds the hunt, and likely controls the operational lifecycle of the system. If that key is kept in a plaintext `.env` workflow, a commit mistake, a shell leak, a process inspection, or a compromised workstation can escalate directly into compromise of the deployer account. For a contest deployment that is expected to custody the hunt funding and administer sensitive flows like pausing, verifier updates, and withdrawals, exposing the signing key in this way is a significant operational risk, not a mere style complaint.

```javascript
uint256 deployerKey = vm.envUint("PRIVATE_KEY");
vm.startBroadcast(deployerKey);
```

## 6. One treasure was unclaimable, and the normal withdraw path could be bricked
Another high-severity issue came from the mismatch between what the circuit allowed and what the Solidity contract assumed. The contract hardcodes `MAX_TREASURES = 10` and expects the hunt to finish after ten successful claims, but the Noir circuit’s baked-in `ALLOWED_TREASURE_HASHES` array does not actually contain ten unique treasures. One value is duplicated and another expected hash is missing. The test data in the circuit was then bent around that mistake rather than detecting it, which masked the mismatch instead of surfacing it.
The result is two linked consequences from the same root cause. First, one treasure becomes effectively unclaimable because the circuit never recognizes its hash as a valid allowed secret. Second, the normal “hunt over” withdrawal path becomes unreachable under the intended one-claim-per-treasure model, because the contract still waits for ten claims even though the circuit only exposes nine unique claimable treasures. In an honest deployment without leaning on a separate double-claim bug, that means leftover ETH can remain stranded and the post-hunt accounting model no longer lines up with reality.


```rust
global ALLOWED_TREASURE_HASHES: [Field; 10] = [
  1505662313093145631275418581390771847921541863527840230091007112166041775502,
  -7876059170207639417138377068663245559360606207000570753582208706879316183353,
  -5602859741022561807370900516277986970516538128871954257532197637239594541050,
  2256689276847399345359792277406644462014723416398290212952821205940959307205,
  10311210168613568792124008431580767227982446451742366771285792060556636004770,
  -5697637861416433807484703347699404695743570043365849280798663758395067508,
  -2009295789879562882359281321158573810642695913475210803991480097462832104806,
  8931814952839857299896840311953754931787080333405300398787637512717059406908,
  -961435057317293580094826482786572873533235701183329831124091847635547871092,
  -961435057317293580094826482786572873533235701183329831124091847635547871092
];
```

# Medium severity findings

## 7. Liquidity enforcement was left to off-chain discipline
The hunt is built around a simple economic assumption: ten treasures, each worth ten ether, which means the contract should be fully funded if it is going to honor the advertised payout schedule. But that invariant is not enforced on-chain. The constructor accepts any msg.value and only validates the verifier address, while the deployment script merely defaults to 100 ether and still allows the value to be overridden through an environment variable. There is no contract-level guarantee that the hunt is fully liquid at deployment.
That turns funding into an honor system. If the operator misconfigures the environment or intentionally underfunds the deployment, the protocol can be launched in a state where valid claims eventually fail with `NotEnoughFunds()`. The code does have a manual owner funding path, so this is not the same class of break as the duplicate-claim issue, but it still undermines the advertised guarantee that a legitimate treasure finder can expect to be paid. For a reward-driven application, that is more than a deployment preference problem. It is an integrity problem in the economic configuration. 

```javascript
constructor(address _verifier) payable {
    if (_verifier == address(0)) revert InvalidVerifier();
    owner = msg.sender;
    verifier = IVerifier(_verifier);
}
```


```javascript
uint256 initialFunding = vm.envOr("INITIAL_FUNDING", DEFAULT_INITIAL_FUNDING);
hunt = new TreasureHunt{value: initialFunding}(address(verifier));
```

## 8. Front-running could steal on-chain attribution even if it could not steal ETH
One of the more interesting medium-severity findings sat in the gap between the payout logic and the event layer. The proof system binds to recipient, and the ETH transfer goes to recipient, which means a straightforward front-runner cannot redirect the money by merely changing the caller. But the Claimed event does not emit the recipient. It emits msg.sender, even though the event field is named recipient. That creates a split between who gets paid and who gets recorded as the apparent claimer. 
In a purely financial protocol, that might sound minor. But treasure-hunt style applications often care about recognition as much as payout: who found which treasure, who appears on the leaderboard, who earns downstream NFT rewards, who gets the bragging rights. In that setting, a mempool bot can copy the original claim transaction, preserve the same proof and bound recipient, and still win the race to emit the first Claimed event under its own sender address. The rightful recipient still receives the ETH, but the attribution layer is polluted. That is why I treated the bug as medium severity when paired with the realistic front-running impact, rather than as a mere low-severity logging mismatch.

```javascript
event Claimed(bytes32 indexed treasureHash, address indexed recipient);
...
emit Claimed(treasureHash, msg.sender);
```

# Low severity findings
## 9. The Claimed event parameter was simply wrong
Viewed in isolation, the `Claimed` event bug is still a valid low-severity issue even before you layer on the front-running attribution story. The event says its second parameter is recipient, but the implementation emits msg.sender. The proof system and ETH transfer still use the actual recipient, so the core state transition is not broken; what is broken is the metadata that off-chain consumers will rely on when reading the logs. 
That is why I considered the bare “incorrect event parameter” reports valid but low severity. By themselves, they prove an event/accounting mismatch. They do not automatically prove a larger exploit unless the reporter also connects that mismatch to something concrete, like attribution theft or a downstream integration that treats the event as canonical truth.

## 10. Non-owners could still fund the contract by sending ETH directly
The explicit `fund()` function is owner-only, but the contract also exposes a permissive `receive()` hook that accepts ETH from anyone and emits the same Funded event. That means a non-owner cannot use the formal owner-only entrypoint, but can still increase the contract balance by sending ETH directly to the contract address. It is an inconsistency between the apparent access-control model and the actual behavior of the contract. 
I treated this as low severity because it does not enable theft, griefing of accounting in any especially harmful way, or privilege escalation. If anything, unsolicited ETH improves the contract’s liquidity position. Still, the reports were correct to note that the “only owner can fund” policy is not consistently enforced across all paths that bring ETH into the contract. 

```javascript
function fund() external payable {
    require(msg.sender == owner, "ONLY_OWNER_CAN_FUND");
    require(msg.value > 0, "NO_ETH_SENT");
    emit Funded(msg.value, address(this).balance);
}

receive() external payable {
    emit Funded(msg.value, address(this).balance);
}
```

## 11. Claimants were unnecessarily forbidden from paying themselves
The claim path rejects any transaction where `recipient == msg.sender`. That means a legitimate finder cannot simply prove the claim and receive the reward into the same address that submitted the proof. Instead, they are forced to control and provide a different wallet. There is no clear security property gained by this restriction, especially because the proof is already bound to the recipient address and the payout follows that bound value. 
This is the kind of issue that would be easy to dismiss during implementation and obvious to users the moment they hit it. It adds friction, complicates the claiming experience, and does not meaningfully harden the system. That makes it a valid but low-severity logic/UX issue rather than a serious exploit. 

```javascript
if (
    recipient == address(0) ||
    recipient == address(this) ||
    recipient == owner ||
    recipient == msg.sender
) revert InvalidRecipient();
```

## 12. The owner could not emergency-withdraw to their own address
The emergency withdrawal path is owner-only and only available while the contract is paused, which means it is already an explicitly trusted recovery mechanism. Even so, the implementation rejects `recipient == owner`, forcing the owner to send emergency funds somewhere else. That is hard to justify as a meaningful security measure when the owner is already authorized to trigger the withdrawal and choose any other nonzero address. 
I marked this low because it is not attacker-driven and does not create direct fund loss. It is simply an unnecessary operational restriction on the very escape hatch that is supposed to be available when something goes wrong. Recovery paths should be practical, and this one was made less practical for no compelling reason. 

```javascript
require(
    recipient != address(0) &&
    recipient != address(this) &&
    recipient != owner,
    "INVALID_RECIPIENT"
);
```

## 13. `updateVerifier()` did not preserve the constructor’s zero-address invariant
The constructor sensibly rejects a zero verifier address, but `updateVerifier()` does not perform the same validation. That means the contract starts life with one configuration invariant and then quietly stops enforcing it on the admin update path. A paused owner can set the verifier to `address(0)` or another nonfunctional target, which can break all future claims until corrected. 
This is an admin-safety issue rather than a public exploit, so low severity felt appropriate. The important point is not that the owner is malicious; it is that the contract already recognized “zero verifier is invalid” as an invariant and then failed to preserve that invariant in the only code path that changes the verifier after deployment.

```javascript
function updateVerifier(IVerifier newVerifier) external {
    require(paused, "THE_CONTRACT_MUST_BE_PAUSED");
    require(msg.sender == owner, "ONLY_OWNER_CAN_UPDATE_VERIFIER");
    verifier = newVerifier;
    emit VerifierUpdated(address(newVerifier));
}
```

## 14. `withdraw()` lacked explicit owner-only access control
The withdrawal path that recovers the remaining balance after the hunt is over is described and clearly intended as an owner function, but the implementation never actually restricts the caller. Anyone can call it once `claimsCount >= MAX_TREASURES`, and the contract will send the funds to owner no matter who triggered it. 
This is low severity because the missing access control does not let an attacker redirect the funds to themselves. The controls that remain in place still prevent direct loss to an arbitrary caller. But it is a genuine access-control inconsistency, and it matters in a contest setting because it is exactly the sort of detail that separates a clean admin model from a sloppy one. 

```javascript
function withdraw() external {
    require(claimsCount >= MAX_TREASURES, "HUNT_NOT_OVER");
    uint256 balance = address(this).balance;
    require(balance > 0, "NO_FUNDS_TO_WITHDRAW");
    (bool sent, ) = owner.call{value: balance}("");
    require(sent, "ETH_TRANSFER_FAILED");
}
```


# Disputed/invalid findings
## 1. “State corruption” was not the right description of the duplicate-claim bug
A few reports tried to turn the duplicate-claim issue into something more dramatic by describing it as “state corruption” or a global contract bricking problem. That is not what the code does. The bug is that the contract reads the wrong mapping key when checking whether a treasure was already claimed. The state write itself still marks the correct user-supplied `treasureHash`. In other words, the logic is broken, but the mapping is not globally poisoned and future unrelated claims are not inherently bricked. 
That distinction mattered during judging. Broken uniqueness protection is already a critical bug; it does not need to be oversold as state corruption. Reports that correctly described the real impact were valid. Reports that claimed the contract became unusable for everyone after one claim were not describing what the code actually does. 

## 2. The input-aliasing claim did not survive testing
Another disputed thread involved an alleged `Circom-style` input aliasing or pseudonymous public-input attack. The idea was that if the duplicate-claim bug were fixed, a claimant might still bypass uniqueness by submitting an aliased `treasureHash + P value`. I reproduced the scenario inside the test suite after temporarily fixing the duplicate-claim guard to check the proper key, and the second claim failed with `SumcheckFailed()` instead of verifying successfully. 

```javascript
    function testDoubleClaimBypassWithInputAliasingAttackFails() public {
        (
            bytes memory proof,
            bytes32 treasureHash,
            address payable recipient
        ) = _loadFixture();

        // The user tries to double claim the treasure by reusing the same proof.
        uint256 P = 21888242871839275222246405745257275088548364400416034343698204186575808495617;
        bytes32 aliasedTreasureHash = bytes32(uint256(treasureHash) + P);

        vm.startPrank(participant);
        // First claim with the original treasureHash succeeds.
        hunt.claim(proof, treasureHash, recipient);
      
        // Second claim with the aliased treasureHash fails.
        vm.expectRevert(abi.encodeWithSelector(BaseZKHonkVerifier.SumcheckFailed.selector));
        hunt.claim(proof, aliasedTreasureHash, recipient);
      
        vm.stopPrank();
    }
```

Test Output:

```bash
000000001cdf38df59787b1a2ee0d3b59dd2ae99000000000000000000000000000000002eb9675261e18b851c023398401e3f6a0000000000000000000000000000000009090ad725fefb522c57358c4a2f886d0000000000000000000000000000000000000000000000000000000000000000
    │   │   ├─ [1349] PRECOMPILES::modexp(32, 32, 32, 0x133fc9f2336b8787a75e37259eb6a6515bdbd324feeea0fb94ebe51fe448c9c2, 0x30644e72e131a029b85045b68181585d2833e84879b9709143e1f593efffffff, 0x30644e72e131a029b85045b68181585d2833e84879b9709143e1f593f0000001) [staticcall]
    │   │   │   └─ ← [Return] 0x21b8168d0a173f0d4374a57fa60edf6d33815f10e7b46d057d173bd01496204e
    │   │   └─ ← [Revert] SumcheckFailed()
    │   └─ ← [Revert] SumcheckFailed()
    ├─ [0] VM::stopPrank()
    │   └─ ← [Return]
    └─ ← [Stop]

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 67.50ms (59.88ms CPU time)

Ran 1 test suite in 147.21ms (67.50ms CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)
```


That result lines up with the verifier code. Although some internal arithmetic reduces values modulo the field, the Fiat–Shamir transcript also hashes the raw `bytes32` public inputs into the challenge derivation. So changing the public input to an aliased representation does not preserve the same verified statement. It changes the transcript and invalidates the proof. That is why I judged the tag invalid for this codebase: the provided PoC did not establish a working exploit against the Noir/Barretenberg-generated verifier in front of us.


# Visual summary of high and medium findings

![alt text](attack_surface_mapped.png)

Why deployment is considered part of the on-chain attack surface? Although deployment is initiated off-chain by the operator, it is included in the on-chain attack surface because it directly determines the initial on-chain state, configuration, and security guarantees of the protocol.
In this system, deployment is not a passive step. It is an active transaction that creates and configures contracts on-chain, sets the verifier address, funds the contract, and establishes key invariants (such as reward availability and verifier correctness). Any mistake or weakness at this stage becomes permanently embedded into the deployed system.
From a security perspective, deployment behaves like the first state transition of the protocol, and therefore must be treated as part of the trusted on-chain environment rather than purely off-chain logic. Issues such as private key handling, incorrect funding amounts, or deploying non-functional contracts (e.g., an oversized verifier) all manifest as on-chain failures or unsafe states, even though they originate from deployment tooling.

# Closing thoughts

Overall, I highly recommend developers consider submitting their projects to the platform for a First Flight contest. It’s a great way to get fresh eyes on your code, surface important issues early, and engage with a security-focused community in a structured and accessible format.
From the judging side, the format encourages thoughtful submissions and creates a nice feedback loop between builders and auditors. I’ll definitely be submitting another project in the future, and I’m looking forward to seeing how the platform and its community continue to evolve.