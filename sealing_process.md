# Sealing Process

In a nutshell the process is splitted into 4 phases:
1. [`seal_pre_commit_phase1`](https://github.com/filecoin-project/rust-filecoin-proofs-api/blob/fba94e039c140698fef692ba5399c925b9b31acf/src/seal.rs#L334)(P1)
    1. The sector data is stored as leaves of a merkle tree, resulting in the merkle tree `tree_d` and the merkle root `comm_d`.
    2. The miner derives a unique `replica_id` for the current sealing sector.
    3. A layered structure called `StackedDrg` that has the same number of nodes in each layer as `#leaves` of `tree_d`, is labeled based on `replica_id`, resulting in layered `labels`.
2. [`seal_pre_commit_phase2`](https://github.com/filecoin-project/rust-filecoin-proofs-api/blob/fba94e039c140698fef692ba5399c925b9b31acf/src/seal.rs#L416)(P2)
    1. The layered `labels` are first aggregated into column commitments, then the column commitments are stored as leaves of a merkle tree, resulting in the merkle tree `tree_c` and the merkle root `tree_c_root`.
    2. The labels of the last layer are used as the keys for encoding the leaves of `tree_d`, the encoding results are stored as leaves of a merkle tree, resulting in the merkle tree `tree_r_last` and the merkle root `tree_r_last_root`.
    3. `tree_c_root` and `tree_r_last_root` are further hashed into `comm_r`.
3. [`seal_commit_phase1`](https://github.com/filecoin-project/rust-filecoin-proofs-api/blob/fba94e039c140698fef692ba5399c925b9b31acf/src/seal.rs#L508)(C1)
    1. The idea is that, in order to prove that the miner actually stored the sector data, the miner has to come up with the merkle proofs of randomly challenged leaves of the above mentioned trees.
    2. The challenge is divided into some specific number of partitions(basically some hardcoded rounds of challenges, where each round challenges several nodes from various trees above), the actual number is [decided](https://github.com/filecoin-project/rust-filecoin-proofs-api/blob/fba94e039c140698fef692ba5399c925b9b31acf/src/registry.rs#L122) by the sector size.
    3. A single partition proof shows that:
        1. The prover knows valid merkle paths for the challenged leaves in `tree_d` that is consistent with `comm_d`.
        2. The prover knows valid merkle paths for the challenged leaves in `tree_c` and `tree_r_last` which are consistent with `comm_r`.
        3. The prover knows the challenged label and its parents' labels in each `StackedDrg` layer.
        4. The prover knows the challenged label and its parents' labels in the last `StackedDrg` layer, which are used as encoding key.
4. [`seal_commit_phase2`](https://github.com/filecoin-project/rust-filecoin-proofs-api/blob/fba94e039c140698fef692ba5399c925b9b31acf/src/seal.rs#L600)(C2)
    1. As the above proof is quite big, the purpose of this phase is to convert the above proofs into zkSNARK proofs, which will be small.
    2. After [all](https://github.com/filecoin-project/rust-fil-proofs/blob/1680ad08f607512e0c2f9d13c330d07f2b2ca081/storage-proofs-porep/src/stacked/circuit/proof.rs#L206) the public input are specified, the private witness and constraints are synthesized, with the help of setup parameters.
    3. Finally the public input plus the `A/B/C` as specified in the groth16 protocol are verified onchain as witness proof to show the miner can answer all challenges, thus must have stored the whole sector with great possibility.

The details of each phase are explained below.

## P1

The purpose of this phase is to generate `tree_d` and `labals` mentioned above.

`tree_d` is generated as follows:
1. The sector data is stored as [leaves](https://github.com/filecoin-project/rust-fil-proofs/blob/1680ad08f607512e0c2f9d13c330d07f2b2ca081/filecoin-proofs/src/api/seal.rs#L92) of the merkle tree, [padding](https://github.com/filecoin-project/rust-fil-proofs/blob/1680ad08f607512e0c2f9d13c330d07f2b2ca081/filecoin-proofs/src/api/seal.rs#L107) if needed.
2. Then a binary merkle tree is [created](https://github.com/filecoin-project/rust-fil-proofs/blob/1680ad08f607512e0c2f9d13c330d07f2b2ca081/filecoin-proofs/src/api/seal.rs#L153) based on leaves.
    1. Some notations: `tree_size` = `#leaf` + `#internal_node`.
    2. Then `#leaf` = `sector_size`/`size_of(domain)`, as per [here](https://github.com/filecoin-project/rust-fil-proofs/blob/1680ad08f607512e0c2f9d13c330d07f2b2ca081/filecoin-proofs/src/api/util.rs#L31).
    3. Then `tree_size` is calculated from `#leaf` [here](https://github.com/filecoin-project/rust-fil-proofs/blob/1680ad08f607512e0c2f9d13c330d07f2b2ca081/filecoin-proofs/src/api/util.rs#L35), basically the size for each layer shrinks by a factor of 2 until 1, then sum up all layers.
    4. Every 32 bytes of leaf data is [converted](https://github.com/filecoin-project/rust-fil-proofs/blob/1680ad08f607512e0c2f9d13c330d07f2b2ca081/storage-proofs-core/src/merkle/builders.rs#L211) into a domain element.
    5. Then each leaf element is [hashed and concatenated](https://github.com/filecoin-project/merkletree/blob/a81abcf3854f96c08de0aada69e597daf11aa757/src/merkle.rs#L2074), then persisted on disk.
    6. From the leaf nodes, the internal nodes are then [built](https://github.com/filecoin-project/merkletree/blob/a81abcf3854f96c08de0aada69e597daf11aa757/src/merkle.rs#L1633) layer by layer from the bottom to top.
    7. The resulting merkle tree is stored under the cache path with [`tree-d`](https://github.com/filecoin-project/rust-fil-proofs/blob/1680ad08f607512e0c2f9d13c330d07f2b2ca081/storage-proofs-core/src/cache_key.rs#L17) in its name.

`labels` is generated as follows:
1. The [`ProofScheme`](https://github.com/filecoin-project/rust-fil-proofs/blob/1680ad08f607512e0c2f9d13c330d07f2b2ca081/storage-proofs-core/src/proof.rs#L9) for the sealing process is [`StackedDrg`](https://github.com/filecoin-project/rust-fil-proofs/blob/1680ad08f607512e0c2f9d13c330d07f2b2ca081/storage-proofs-porep/src/stacked/vanilla/proof_scheme.rs#L16).
2. First the [`PublicParams`](https://github.com/filecoin-project/rust-fil-proofs/blob/1680ad08f607512e0c2f9d13c330d07f2b2ca081/storage-proofs-porep/src/stacked/vanilla/params.rs#L53) for `StackedDrg` is [setup](https://github.com/filecoin-project/rust-fil-proofs/blob/1680ad08f607512e0c2f9d13c330d07f2b2ca081/storage-proofs-porep/src/stacked/vanilla/proof_scheme.rs#L26), which includes `StackedBucketGraph` and `LayerChallenges`, triggered [here](https://github.com/filecoin-project/rust-fil-proofs/blob/1680ad08f607512e0c2f9d13c330d07f2b2ca081/filecoin-proofs/src/api/seal.rs#L129).
    1. `StackedBucketGraph` is an [alias](https://github.com/filecoin-project/rust-fil-proofs/blob/1680ad08f607512e0c2f9d13c330d07f2b2ca081/storage-proofs-porep/src/stacked/vanilla/graph.rs#L61) for `StackedGraph<H, BucketGraph<H>>`, a layered graph with the same number of nodes in each layer as `#leaves` in `tree_d`, the actual `#layer` is [decided](https://github.com/filecoin-project/rust-fil-proofs/blob/1680ad08f607512e0c2f9d13c330d07f2b2ca081/filecoin-proofs/src/constants.rs#L108) by sector size.
    2. The core function of this structure is for easily generating base parents and expander parents for nodes.
        1. For base parents(parents in the same layer), it [delegates](https://github.com/filecoin-project/rust-fil-proofs/blob/1680ad08f607512e0c2f9d13c330d07f2b2ca081/storage-proofs-porep/src/stacked/vanilla/graph.rs#L418) the the work to the [`BucketGraph::parents`](https://github.com/filecoin-project/rust-fil-proofs/blob/1680ad08f607512e0c2f9d13c330d07f2b2ca081/storage-proofs-core/src/drgraph.rs#L138) directly.
            1. Basically this function randomly but deterministically(`seed` is given) chooses nodes with smaller index as parents, but guarantees that the immediate predecessor is among them so that labels can only be computed sequentially.
        2. For expander parents(parents in the previous layer), it calls [`correspondent`](https://github.com/filecoin-project/rust-fil-proofs/blob/128f7209ec583e023f04630102ef1dd17fbe2370/storage-proofs-porep/src/stacked/vanilla/graph.rs#L341) to compute parents one by one, basically using feistel encoding to randomly but deterministically choose nodes with smaller index.
        3. Both base parents and expander parents may contain duplicate nodes.
3. Then a unique `replica_id` is [generated](https://github.com/filecoin-project/rust-fil-proofs/blob/1680ad08f607512e0c2f9d13c330d07f2b2ca081/filecoin-proofs/src/api/seal.rs#L176) for the current sealing sector.
4. Then the layered labels for `StackedDrg` is generated based on `PublicParams` and `replica_id`.
    1. For the first layer, each node only has base parents, [`create_label`](https://github.com/filecoin-project/rust-fil-proofs/blob/1680ad08f607512e0c2f9d13c330d07f2b2ca081/storage-proofs-porep/src/stacked/vanilla/create_label/single.rs#L173) is [called](https://github.com/filecoin-project/rust-fil-proofs/blob/1680ad08f607512e0c2f9d13c330d07f2b2ca081/storage-proofs-porep/src/stacked/vanilla/create_label/single.rs#L54) to generate labels for each node.
        1. The final node label is computed from `sha256(layer index, node index, replica id, all base parents)`
        2. The base parents are hashed for [7 rounds](https://github.com/filecoin-project/rust-fil-proofs/blob/128f7209ec583e023f04630102ef1dd17fbe2370/storage-proofs-porep/src/stacked/vanilla/graph.rs#L242).
    2. For all other layers, each node has both base parents and expander parents, [`create_label_exp`](https://github.com/filecoin-project/rust-fil-proofs/blob/master/storage-proofs-porep/src/stacked/vanilla/create_label/single.rs#L210) is [called](https://github.com/filecoin-project/rust-fil-proofs/blob/master/storage-proofs-porep/src/stacked/vanilla/create_label/single.rs#L65) to generate labels for each node.
        1. The final node label is computed from `sha256(layer index, node index, replica id, all base and expander parents)`
        2. The base and expander parents are hashed for [3 rounds](https://github.com/filecoin-project/rust-fil-proofs/blob/128f7209ec583e023f04630102ef1dd17fbe2370/storage-proofs-porep/src/stacked/vanilla/graph.rs#L213).
    3. The resulting labels of each layer are stored separately under the cache path with [`layer-N`](https://github.com/filecoin-project/rust-fil-proofs/blob/1680ad08f607512e0c2f9d13c330d07f2b2ca081/storage-proofs-porep/src/stacked/vanilla/create_label/mod.rs#L26) in its name.



## P2

The purpose of this phase is to generate `tree_c` and `tree_r_last` mentioned above.


`tree_c` is generated as follows:
1. The shape of `tree_c` is not vanilla binary merkle tree, but [decided](https://github.com/filecoin-project/rust-fil-proofs/blob/1680ad08f607512e0c2f9d13c330d07f2b2ca081/filecoin-proofs/src/constants.rs#L223) by sector size, basically it's a 3-layered merkle tree: base tree -> sub tree -> top tree, where each tree can have different arity(branches).
    1. For 32GB sector, the shape is `(U8, U8, U0)`, meaning the base tree arity is 8, sub tree arity is also 8, but there's no top tree.
    2. For 64GB sector, the shape is `(U8, U8, U2)`, meaning the base tree arity is 8, sub tree arity is also 8, and the top tree arity is 2.
    3. The max height for top tree and sub tree is 1, so that the base tree count can be computed like [this](https://github.com/filecoin-project/rust-fil-proofs/blob/1680ad08f607512e0c2f9d13c330d07f2b2ca081/storage-proofs-core/src/merkle/builders.rs#L374), resulting in `tree_count`.
2. The `node_count` for base tree is computed by [`graph.size() / tree_count`](https://github.com/filecoin-project/rust-fil-proofs/blob/1680ad08f607512e0c2f9d13c330d07f2b2ca081/storage-proofs-porep/src/stacked/vanilla/proof.rs#L1285).
3. Imagine the layered `StackedGraph` mentioned above, it's cut into chunks of width `node_count` vertically, then for each chunk, the nodes of the same column are [hashed](https://github.com/filecoin-project/rust-fil-proofs/blob/1680ad08f607512e0c2f9d13c330d07f2b2ca081/storage-proofs-porep/src/stacked/vanilla/proof.rs#L813) into a single value, resulting in `node_count` hash values. These `node_count` hash values are then used as leaves to [build](https://github.com/filecoin-project/rust-fil-proofs/blob/1680ad08f607512e0c2f9d13c330d07f2b2ca081/storage-proofs-porep/src/stacked/vanilla/proof.rs#L821) the base merkle tree.
4. In total, the above step generates `tree_count` base merkle trees, which are then [aggregated](https://github.com/filecoin-project/rust-fil-proofs/blob/1680ad08f607512e0c2f9d13c330d07f2b2ca081/storage-proofs-porep/src/stacked/vanilla/proof.rs#L830) into the 3-layered merkle tree.

`tree_r_last` is generated as follows:
1. `labels` of the last layer are [used as encoding keys](https://github.com/filecoin-project/rust-fil-proofs/blob/1680ad08f607512e0c2f9d13c330d07f2b2ca081/storage-proofs-porep/src/stacked/vanilla/proof.rs#L1413) for the replica.
2. Then `tree_r_last` is generated in a similar fasion as `tree_c` except that each element is computed by [combining](https://github.com/filecoin-project/rust-fil-proofs/blob/1680ad08f607512e0c2f9d13c330d07f2b2ca081/storage-proofs-porep/src/stacked/vanilla/proof.rs#L897) corresponding elements of sector data and label data of the last layer.

The roots of `tree_c` and `tree_r_last` are then [hashed](https://github.com/filecoin-project/rust-fil-proofs/blob/1680ad08f607512e0c2f9d13c330d07f2b2ca081/storage-proofs-porep/src/stacked/vanilla/proof.rs#L1437) into `comm_r`.

## P3

The purpose of this phase is to generate proofs for all challenges.

The process is as follows:
1. The proofs are computed partition by partition.
2. For each partition, all challenged leaf indices are computed [in a batch](https://github.com/filecoin-project/rust-fil-proofs/blob/1680ad08f607512e0c2f9d13c330d07f2b2ca081/storage-proofs-porep/src/stacked/vanilla/proof.rs#L154).
3. For each challenged leaf index `c`:
    1. The merkle proof of leaf `c` in `tree_d` is [computed](https://github.com/filecoin-project/rust-fil-proofs/blob/1680ad08f607512e0c2f9d13c330d07f2b2ca081/storage-proofs-porep/src/stacked/vanilla/proof.rs#L166).
    2. All labels of index `c` are [fetched and converted](https://github.com/filecoin-project/rust-fil-proofs/blob/1680ad08f607512e0c2f9d13c330d07f2b2ca081/storage-proofs-porep/src/stacked/vanilla/proof.rs#L177) into a column proof of `tree_c`.
    3. All base parents of node `c` are [fetched and converted](https://github.com/filecoin-project/rust-fil-proofs/blob/1680ad08f607512e0c2f9d13c330d07f2b2ca081/storage-proofs-porep/src/stacked/vanilla/proof.rs#L181) into a column proof of `tree_c`.
    4. All expander parents of node `c` are [fetched and converted](https://github.com/filecoin-project/rust-fil-proofs/blob/1680ad08f607512e0c2f9d13c330d07f2b2ca081/storage-proofs-porep/src/stacked/vanilla/proof.rs#L188) into a column proof of `tree_c`.
    5. The merkle proof of leaf `c` in `tree_r_last` is [computed](https://github.com/filecoin-project/rust-fil-proofs/blob/1680ad08f607512e0c2f9d13c330d07f2b2ca081/storage-proofs-porep/src/stacked/vanilla/proof.rs#L205).
    6. For each layer:
        1. The parent labels are challenged node `c` are fetched and [repeated](https://github.com/filecoin-project/rust-fil-proofs/blob/1680ad08f607512e0c2f9d13c330d07f2b2ca081/storage-proofs-porep/src/stacked/vanilla/proof.rs#L248) as `LabelingProof`.
        2. The `LabelingProof` of the last layer is redundantly saved as `EncodingProof`.

## P4

The purpose of this phase is to make the above proofs succinct using zkSNARK circuit(groth16 circuit actually).

Central to the circuit is the [`ConstraintSystem`](https://github.com/zhiqiangxu/bellperson/blob/42b3aa8a34bedbeb3145729882a671635cb288fb/src/constraint_system.rs#L71) trait:
1. [`alloc_input`](https://github.com/zhiqiangxu/bellperson/blob/42b3aa8a34bedbeb3145729882a671635cb288fb/src/constraint_system.rs#L99) is called to allocate a public input.
2. [`alloc`](https://github.com/zhiqiangxu/bellperson/blob/42b3aa8a34bedbeb3145729882a671635cb288fb/src/constraint_system.rs#L91) is called to allocate a private witness.
3. [`enforce`](https://github.com/zhiqiangxu/bellperson/blob/42b3aa8a34bedbeb3145729882a671635cb288fb/src/constraint_system.rs#L107) is called to enforce a constraint of the form `A * B = C`.

The process is as follows:
1. For [each](https://github.com/filecoin-project/rust-fil-proofs/blob/1680ad08f607512e0c2f9d13c330d07f2b2ca081/storage-proofs-core/src/compound_proof.rs#L245) partition proof, [construct](https://github.com/filecoin-project/rust-fil-proofs/blob/1680ad08f607512e0c2f9d13c330d07f2b2ca081/storage-proofs-porep/src/stacked/circuit/proof.rs#L288) an instance of [`StackedCircuit`](https://github.com/filecoin-project/rust-fil-proofs/blob/1680ad08f607512e0c2f9d13c330d07f2b2ca081/storage-proofs-porep/src/stacked/circuit/proof.rs#L28).
    1. Internally the partition proof is [converted](https://github.com/filecoin-project/rust-fil-proofs/blob/1680ad08f607512e0c2f9d13c330d07f2b2ca081/storage-proofs-porep/src/stacked/circuit/proof.rs#L320) into challenge proofs.
2. The logic to process groth16 circuit is common(in fact the original author of `bellman` is planning to move it into a separate crate in the future):
    1. Generate two random variables `r` and `s` [nondeterministically](https://github.com/zhiqiangxu/bellperson/blob/42b3aa8a34bedbeb3145729882a671635cb288fb/src/groth16/prover.rs#L235).
    2. Each circuit is synthesized in a batch via [`synthesize_circuits_batch`](https://github.com/zhiqiangxu/bellperson/blob/42b3aa8a34bedbeb3145729882a671635cb288fb/src/groth16/prover.rs#L648).
        1. The first public input is [always `1`](https://github.com/zhiqiangxu/bellperson/blob/42b3aa8a34bedbeb3145729882a671635cb288fb/src/groth16/prover.rs#L669).
        2. Then [`circuit::synthesize`](https://github.com/zhiqiangxu/bellperson/blob/42b3aa8a34bedbeb3145729882a671635cb288fb/src/groth16/prover.rs#L671) is called to generate all assignments and constraints.
        3. The `A/B/C` polynomial evaluations as specified in the groth16 paper and assignments for public input and witness are returned for further processing.
    3. A bunch of tricks are further carried out to extract a proof as specified in the groth16 paper.

### Arithmetization

The process of converting general computation to circuit is called arithmetization, which is pretty complicated in the general case (e.g. zkevm), but much easier for a specific computation, like sealing, and it's more relevant for developers actually writing circuits, so it's worth to dive into the details of [`StackedCircuit::synthesize`](https://github.com/filecoin-project/rust-fil-proofs/blob/1680ad08f607512e0c2f9d13c330d07f2b2ca081/storage-proofs-porep/src/stacked/circuit/proof.rs#L92):
1. A witness for `replica_id` is allocated, which is then constrained to be equal to a input.
2. Then the `replica_id` witness is [converted](https://github.com/filecoin-project/rust-fil-proofs/blob/1680ad08f607512e0c2f9d13c330d07f2b2ca081/storage-proofs-porep/src/stacked/circuit/proof.rs#L114) to bit witness of size `Scalar::NUM_BITS`(required by [`sha256 circuit`](https://github.com/filecoin-project/bellperson/blob/950829bba6504a806958877451480ef597d6bd73/src/gadgets/sha256.rs#L50)).
3. A witness for `comm_d` is allocated and assigned, which is then constrained to be equal to a input.
4. A witness for `comm_r` is allocated and assigned, which is then constrained to be equal to a input.
5. A witness for `comm_r_last` is allocated and assigned.
6. A witness for `comm_c` is allocated and assigned.
7. `comm_r` is computed and saved in another witness via [`hash2_circuit(comm_c, comm_r_last)`](https://github.com/filecoin-project/rust-fil-proofs/blob/1680ad08f607512e0c2f9d13c330d07f2b2ca081/storage-proofs-porep/src/stacked/circuit/proof.rs#L153) and constrained to be equal to the public input `comm_r`.
8. Then for each challenge proof, [`Proof::synthesize`](https://github.com/filecoin-project/rust-fil-proofs/blob/1680ad08f607512e0c2f9d13c330d07f2b2ca081/storage-proofs-porep/src/stacked/circuit/proof.rs#L169) is called to generate a subcircuit.
    1. `data_leaf` opening:
        1. A witness for `data_leaf` is allocated and assigned.
        2. [`enforce_inclusion`](https://github.com/filecoin-project/rust-fil-proofs/blob/1680ad08f607512e0c2f9d13c330d07f2b2ca081/storage-proofs-porep/src/stacked/circuit/params.rs#L130) is called to ensure existence of `data_leaf` in `comm_d`, essentially an arithmetization for merkle leaf inclusion proof.
    3. `replica column` opening:
        1. For each base/expander parent column:
            1. A witness is allocated and assigned for each node in the column, then [`poseidon_hash`](https://github.com/filecoin-project/neptune/blob/5abfd12ae72f19d7a2a491307e51b3bef2c95214/src/circuit.rs#L376) is called to allocate and assign the [column hash witness](https://github.com/filecoin-project/rust-fil-proofs/blob/1680ad08f607512e0c2f9d13c330d07f2b2ca081/storage-proofs-porep/src/stacked/circuit/params.rs#L148), resulting in `val`.
            2. [`enforce_inclusion`](https://github.com/filecoin-project/rust-fil-proofs/blob/1680ad08f607512e0c2f9d13c330d07f2b2ca081/storage-proofs-porep/src/stacked/circuit/params.rs#L130) is called to ensure existence of `val` in `comm_c`.
    4. A witness for `challenge` is allocated and assigned, which is then [`pack_into_input`](https://github.com/filecoin-project/rust-fil-proofs/blob/1680ad08f607512e0c2f9d13c330d07f2b2ca081/storage-proofs-porep/src/stacked/circuit/params.rs#L186).
    5. labeling opening:
        1. For each layer, the labeling is verified:
            1. Parent labels are converted and constrained into bit values, and then duplicated according to the hashing algorithm.
                1. The reason for this step is that the [`sha256 circuit`](https://github.com/filecoin-project/bellperson/blob/950829bba6504a806958877451480ef597d6bd73/src/gadgets/sha256.rs#L50) expects the input to be in this form.
            2. [`create_label_circuit`](https://github.com/filecoin-project/rust-fil-proofs/blob/1680ad08f607512e0c2f9d13c330d07f2b2ca081/storage-proofs-porep/src/stacked/circuit/params.rs#L235) is called to actually allocate and assign the label for the challenged node, resulting in `label`.
                1. `create_label_circuit` prepares the input for `sha256 circuit`, pads with `false` if necesary, then calls `sha256 circuit` to do the heavy part.
        2. Each layer's `label` is stored into `column_labels`, and finally `column_hash` is computed from `column_labels` via `poseidon_hash` and ensured to exist in `comm_c` via `enforce_inclusion`.
    6. encoding node opening:
        1. The label of of last layer is used as encoding key to encode the above `data_leaf`, resulting in `encoded_node`.
        2. [`enforce_inclusion`](https://github.com/filecoin-project/rust-fil-proofs/blob/1680ad08f607512e0c2f9d13c330d07f2b2ca081/storage-proofs-porep/src/stacked/circuit/params.rs#L130) is called to ensure existence of `encoded_node` in `comm_r_last`.
    

So by here we can see that circuit is composed of assignments and constraints, and how the circuit for sealing is actually spelled out.



## Verify

The entry point of proof verification is [`batch_verify_seals`](https://github.com/filecoin-project/ref-fvm/blob/146d6bc10c80001bc772ca55abcd56cc8120cbb5/fvm/src/kernel/default.rs#L574):
1. which then calls [`verify_seal`](https://github.com/filecoin-project/ref-fvm/blob/146d6bc10c80001bc772ca55abcd56cc8120cbb5/fvm/src/kernel/default.rs#L1137)
    1. which then calls [`CompoundProof::verify`](https://github.com/filecoin-project/rust-fil-proofs/blob/1680ad08f607512e0c2f9d13c330d07f2b2ca081/storage-proofs-core/src/compound_proof.rs#L140) to verify the proof for a single sector.
        1. For a single partition proof, [`verify_proof`](https://github.com/filecoin-project/bellperson/blob/950829bba6504a806958877451480ef597d6bd73/src/groth16/verifier.rs#L38) is called for verification.
        2. For multiple partition proofs, a [combining technique](https://github.com/filecoin-project/bellperson/blob/950829bba6504a806958877451480ef597d6bd73/src/groth16/verifier.rs#L137) is used to do batch verification.



