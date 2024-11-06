# BIT-000X: Proof of Weights (Working Paper)

- **BIT Number:** TBD
- **Title:** Proof of Weights
- **Author(s):** Inference Labs (https://github.com/inference-labs-inc/)
- **Discussions-to:** [URL for discussion thread]
- **Status:** Draft
- **Type:** Core, Subtensor
- **Created:** 2024-09-09
- **Updated:** 2024-09-09
- **Requires:** None
- **Replaces:** None

## Abstract

This proposal introduces Proof of Weights, a real-time cryptographic solution for Bittensor's ecosystem. It ensures subnet owners' incentivization mechanisms are faithfully executed by validators, safeguarding intended behavior and reward distribution. By leveraging scalable, configurable zero-knowledge proofs, Proof of Weights maintains network bandwidth while rewarding honest validators for their genuine contributions, providing subnet operators with enhanced tools to direct validator behavior.

## Motivation

In the Bittensor documentation, subnet owners are burdened with two key responsibilities:

- Defining the specific digital task to be performed by the subnet miners.
- Implementing an incentive mechanism that aligns miners with the desired task outcomes.

The existing protocol, however, does not offer subnet owners any method to verify that their specific digital task is being completed by the network, or that their incentive mechanism is being used by validators to score miner’s work. This inability to verify the behavior of subnet validators has lead to the existence of weight copying, whereby validators can avoid doing any of the work defined by the subnet owner, and gain higher profit by copying the weights distributed to miners by other validators who are (presumably) running the correct code. The lack of verification present in the protocol today could lead to situations where a subnet effectively goes rogue; validators could choose to simply not run the code provided by the subnet owner, modify the owner’s code, or run their own code to hijack the subnet entirely. This is a serious issue for builders entering the Bittensor ecosystem, as no certainty is guaranteed in the code they deploy for use in their subnet.

Proof of Weights is a necessary addition to the Bittensor network because it allows subnet owners to verify usage of their defined incentive mechanisms by validators participating in their subnet. Specifically, this addition to the protocol will allow subnet owners to define a penalty for validators that cannot cryptographically prove unique and genuine weight calculations conducted during runtime against the specific functionality of the subnet. This is a benefit to the Bittensor ecosystem as it provides subnet owners with the ability to verify their responsibilities are being carried out within the subnet they operate in, in a way that is trustless and transparent.

## Specification

Proof of Weights exclusively covers subnet incentive mechanisms. Where a subnet incentive mechanism is defined as follows.

> A mechanism which converts input vectors which represent qualitative signals about a miner’s work into succinct weight vectors which represent the outcoming score of that miner on a normal distribution.

### Zero Knowledge Circuits

Each subnet owner that intends to use Proof of Weights must supply a custom zero knowledge circuit representing their incentive mechanism. Technically, asking each subnet owner to implement their own zero knowledge circuit from scratch would be prohibitive in terms of development effort. To ease this barrier to entry, conversion of Python based incentive designs to Halo2 proof circuits is adopted through the use of EZKL.

High level, the process for conversion of the incentive mechanism into a zero knowledge circuit involves the following steps.

1. Implementation of the incentive mechanism as a torch neural network module
2. Exportation of the incentive mechanism to ONNX format
3. Conversion of the ONNX representation to the Halo2 circuit

After this process, the subnet owner will provide the exported Halo2 verification key to the chain as discussed in the hyperparameters section.

### Hyperparameters

Two new hyperparameters are required, each can be configured by the subnet owner.

- `weights_verification_key`

  This hyperparameter stores the cryptographic verification key unique to the incentive mechanism created by the owner of the subnet. Blockchain validators will use the value of this hyperparameter during proof verification, to attest validator supplied proofs against.

- `valid_proof_value`

  This hyperparameter is a float value between 0 and 1. It represents the value that the subnet owner places on validators supplying cryptographic proofs alongside their weights within the subnet. When set to zero, validators that do not provide proofs alongside their weights are valued the same as validators that do. When set to 1, validators that provide proofs are maximally incentivized.

### Additional Weight Set Method

One new function is required, to allow validators running on proof of weights subnets to commit proofs and input signals along with the weights they set. This function has a signature as follows, and must include functionality to verify incoming proofs and provided signals against the verification_key hyperparameter, using the metadata parameters provided within the function call.

```rust
set_weights_with_proof(self,
    wallet: "bittensor.wallet",
    netuid: int,
    uids: Union[NDArray[np.int64], "torch.LongTensor", list],
    weights: Union[NDArray[np.float32], "torch.FloatTensor", list],
    proof: bytes,
    signals: list[Any],
    wait_for_inclusion: bool = False,
    wait_for_finalization: bool = False,
    prompt: bool = False,
    max_retries: int = 5,
    parameters: dict = None
)
```

### Yuma Consensus

Changes to Yuma Consensus must be made to implement a penalty for validators that cannot provide a proof with their weights. Let Vp represent an array of proof validity signals for each validator such that Vp[i] ∈ {1, 0}. A value of 1 indicates the validator `i` provided a valid proof with their weights during the previous tempo. To implement the valid_proof_value hyperparameter, the following modification to clipped weights within Yuma consensus is proposed. The core functionality of this change involves boosting the weights of validators who supply valid proofs with their weights. When set to 1, the subnet owner places maximum value in validators who provide proof. This means that validators who supply proof with their weights will have their weights clipped to consensus without further changes, and validators who are unable to supply proofs along with their weights will be clipped to zero.

```rust
W_clipped *= (Vp + (1 - Vp) * (1 - hyperparameter(valid_proof_value)))
```

<details>
<summary>Simulation of Proof of Weights in Action</summary>

![PoW Simulation](https://github.com/user-attachments/assets/e6ef0e53-9d96-43ce-9cdf-df77f82be0ad)

</details>

## Rationale

In response to the weight copying problem, Bittensor has recently released Commit Reveal to combat this behavior by which in specific conditions the network would reduce the rewards of the copying actors. If enabled, Commit Reveal addresses this problem by enforcing all subnet validators to commit a hash of their responses on chain. Validators would then reveal these responses after the expiry of an interval that has been set as a parameter by the subnet owner.

Although Commit Reveal is functionally effective, the time interval does introduce latency, delaying important feedback to miners throughout the network. Bittensor’s primary goals are to constantly incentivize the development of the state-of-the-art with economic incentive. Commit Reveal in its current state slows down this feedback loop. This approach slows down the speed of innovation of the network and creates additional frustrations for miners by reducing the real time visibility of their performance. This also creates a new strategy for weight copiers, weight prediction, whereby a dishonest validator can instead predict what the scores will be once revealed.

With Proof of Weights, validators are required to prove that they've done the correct scoring work to evaluate miners within the subnet based on the subnet owner's defined incentive mechanism. This opens the door for new strategies which can be used in conjunction with commit reveal to further mitigate the issue of weight copying.

### Alternatives Considered

#### Circom

The [Circom] proof system using [groth16] proofs was considered as an alternative to [EZKL]’s Halo2 implementation for several reasons listed below, however the barrier to developer adoption is significant considering it uses a DSL ([circom]) to define constraints. Migration to this proof system is recommended in the future as tooling surrounding the proof system improves, lowering the barrier to entry for developers.

- Proof sizes remain a fixed size of approximately 800 bytes
- Proving times have shown significant improvements within the [Omron] subnet, with 1024 scoring changes proven in under 1.5s for Subnet 2.

Benchmarks taken from an Apple M1 machine with 10 CPUs and 32GB of RAM, comparing the performance of [EZKL] and [Circom] for several Proof of Weights circuits are show below.

| Proof System | Circuit   | Batch Size | Proof Size | Public Signals Size | Verification Time (per proof) |
| ------------ | --------- | ---------- | ---------- | ------------------- | ----------------------------- |
| EZKL         | Subnet 2  | 1024       | 90kb       | 1024                | 0.7s                          |
| EZKL         | Subnet 27 | 1024       | 147kb      | 1024                | 0.1s                          |
| EZKL         | Subnet 48 | 1024       | 77kb       | 1024                | 0.6s                          |
| Circom       | Subnet 2  | 1024       | ~616 bytes | 166kb               | 0.38s                         |
| Circom       | Subnet 48 | 1024       | ~616 bytes | 65kb                | 0.31s                         |

#### Jolt

Jolt is a novel proof system developed by a16z, which provides a fully featured zkVM. This virtual machine allows users to define their programs in the widely adopted rust programming language. Ultimately, despite the ease of development, the proof size and verification times are not competitive with [EZKL] and [Circom]. For these reasons, it does not currently offer advantages over existing proof systems and thus was rejected for use in Proof of Weights. The Jolt proof system continues to develop rapidly, and future progress will be reevaluated on an ongoing bases for use with Proof of Weights.

> For reference, metrics containing `PoW` refer to Proof of Weights circuits developed using the Jolt proof system. Metrics containing `Proof of Weights` refer to circuits developed using the Circom proof system.

[See proving benchmarks on the Omron Subnet →](https://wandb.ai/inferencelabs/omron/reports/SN2-PoW-256-min_response_time-SN2-Proof-of-Weights-1-56-min_response_time-SN27-PoW-256-min_response_time-24-11-06-19-39-03---VmlldzoxMDA1Nzk0MQ)

### Impact Assessment

#### Performance Metrics

- Proof verification times vary but are generally under 1s per proof
- Proof sizes vary depending on circuit complexity, in the range of 50kb-1mb
- Trusted setup files (KZG commitments) must be present on each subtensor node for the purposes of proof verification. In total, 16GB of persistent storage is required for the KZG files.

> Note that each circuit must be contrained in terms of maximum complexity to prevent the circuit from becoming too large to be practical for the network.

#### Scaling Considerations

Based on the average proof sizes and verification times from the benchmarks above, scaled load is estimated as follows.

| Scenario             | Validators | Subnets | Total Verification Time per Tempo | Total Proof Size per Tempo |
| -------------------- | ---------- | ------- | --------------------------------- | -------------------------- |
| Current              | 20/subnet  | 64      | 597s                              | 134mb                      |
| Current with groth16 | 20/subnet  | 64      | 384s                              | 788kb                      |
| Future\*             | 10/subnet  | 64      | 298s                              | 67mb                       |

(\*) As Bittensor progresses through the dTAO upgrade, validator consolidation is anticipated and the number of active validators on each subnet will decrease. Yuma consensus is less effective as the marginal validator is removed, and what happens when a single validator remains? Who will ensure they are honest and will treat all miners fairly? Proof of Weights is designed such that a single validator can prove they are honest and fair.

## Backwards Compatibility

To support validator submission of weights accompanied by cryptographic proofs, a new version of Bittensor will be released. Subnet owners that choose to enable this hyperparameter will require all validators participating in that subnet to upgrade to this new version in order to successfully commit proofs and public signals alongside their weights.

Similarly to how commit-reveal is implemented as an optional feature to be enabled at the discretion of a subnet owner, Proof of Weights is also optional. Subnet owners who choose not to enable Proof of Weights will not have any impact on the operation of their subnet, miners, or validators. Proof of Weights can be enabled or disabled at any time without impact to the subnets operations (minus the typical inconvenience of deploying a subnet software update across miners/validators).
Reference Implementation (Optional)
If applicable, include a link to a reference implementation that demonstrates the feasibility of the proposal. This implementation may be partial or fully complete.

## Reference Implementation

[Omron] has served as a testing ground for proof of weights for the past four months, and through it many iterations have been implemented, tested and improved. Today, the subnet provides three proof of weights circuits for subnets 2, 27 and 48, with many more on the way. Please find attached reference implementations of each.

| Subnet    | Circuit Link                                                                                                                                                                                     |
| --------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| Subnet 2  | https://github.com/inference-labs-inc/omron-subnet/tree/0627356363d2745d7511c1e9a28e27df7e242261/neurons/deployment_layer/model_55de10a6bcf638af4bc79901d63204a9e5b1c6534670aa03010bae6045e3d0e8 |
| Subnet 27 | https://github.com/inference-labs-inc/omron-subnet/tree/0627356363d2745d7511c1e9a28e27df7e242261/neurons/deployment_layer/model_b7d33e7c19360c042d94c5a7360d7dc68c36dd56c449f7c49164a0098769c01f |
| Subnet 48 | https://github.com/inference-labs-inc/omron-subnet/blob/d7d61c665835b21497f8e909de7aa51134f13a93/neurons/deployment_layer/sn48/src/main.circom#L1                                                |

Additionally, a fork of the developer documentation repository has been created to provide simulations of changes to Yuma consensus and the below referenced approaches to mitigate the issue of weight copying.

[Developer Documentation Simulation](https://github.com/inference-labs-inc/pow-simulations)

## Security Considerations

Though this proposal primarily exists to increase the level of security throughout the network, there are important considerations regarding the security of this update itself.

Note that an additional concern would normally be the ability of validators to copy eachother’s proofs from extrinsics submitted to the chain, however this is made impossible through the use of the validator’s UID as one of the input signals. The blockchain will check the UID included in the proof against the submitting key to verify a match and mitigate this risk.

An additional layer of security is provided by the recently executed Master Services Agreement with [ImmuneFi], which establishes a $100,000 bug bounty program aimed at incentivizing white-hat hackers to identify and report vulnerabilities within the [Omron] codebase. By proactively identifying potential flaws through external audits and community-driven bounty programs, the subnet will ensure continuous security vigilance.

A significant concern related to the issue of weight copying exists in a situation where a dishonest validator employs artificially generated input data designed to mimic real-world data, in an effort to supply cryptographic proof that they operated within the subnet and effectively weight copy by generating valid proofs based on false pretenses. As described, this situation could present a challenge in that validators could observe resulting weight vectors across submissions and pre-emptively calculate consensus weights, using these calculations to craft input data designed to provide proven “perfect” consensus weights.

In terms of weight copying, four potential solutions are proposed, all of which build upon advancements made in commit reveal and proof of weights as outlined in this proposal. The emphasis in all of these initiatives is on balancing instant feedback with risk of exposing critical data to weight copying parties.

### Countering Weight Copying

Proof of Weights used in conjunction with the following commit-reveal related strategies can provide a powerful and demonstrable defense against weight copying.

#### Peer Prediction

This strategy was first introduced in the first BT weight copier paper and proposes the integration of peer prediction to determine how likely weights are to have been generated authentically based on a non-deterministic case of scoring. This strategy has the potential to integrate into proof of weights through comparison of circuit inputs.

#### Range Proofs

Through the use of ZKRPs, validators can prove that their weights fall within a specific range on the incentive curve. For example, the proof can contain buckets of UIDs for percentile ranges on the incentive curve. This allows miners to receive instantaneous feedback without allowing copiers full knowledge of weight vectors, only that each miner is between two points on the incentive curve.

#### Rank Proofs

Similar to the range proof approach, proofs could output the rank order of each miner rather than their actual weight vector, again this would provide sufficient information to the miner in terms of how they fare in terms of position in the incentive curve without exposing critical weight vectors.

#### Partial revelation

In this strategy, subtensor nodes select a random subset of validators based on a stake threshold percentage, compelling a subset of validators to expose their weight vectors immediately. Miners will receive partial feedback instantaneously, allowing them to gauge their incentive without full visibility. During revelation, the remaining validators are compelled to reveal their previously proven weights and consensus is then calculated for the previous tempo. Weight copiers aren’t provided enough information to calculate consensus values, though miners are provided with a reasonably high degree of visibility into their incentive instantly.

## Copyright

This document is licensed under The Unlicense.

- Public signals increase to a dynamic size in the case that input signals are used in a peer prediction style approach to weight copying. To reduce the amount of load on the chain to a smaller, fixed size and for the purposes of this BIT, output signals (weights and validator UID) are sufficient.

[BT Weight Copier Paper]: https://docs.bittensor.com/papers/BT_Weight_Copier-29May2024.pdf
[EZKL]: https://github.com/zkonduit/ezkl
[Circom]: https://github.com/iden3/circom
[ZKRPs]: https://eprint.iacr.org/2024/430.pdf
[groth16]: https://eprint.iacr.org/2016/260.pdf
[Omron]: https://github.com/inference-labs-inc/omron-subnet
[ONNX]: https://onnx.ai/
[Yuma Consensus]: https://docs.bittensor.com/yuma-consensus
[ImmuneFi]: https://immunefi.com/
