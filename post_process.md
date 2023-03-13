# PoST Process

In a nutshell, the process for `winning post` and `window post` is as follows:
1. winning post:
    1. staged:
        1. Calculate `challenged_sectors` by [`generate_winning_post_sector_challenge`](https://github.com/filecoin-project/rust-fil-proofs/blob/7809b42417f923ced415115e745b43a7a88a3459/filecoin-proofs/tests/api.rs)
        2. Calculate `challenges` by [`generate_fallback_sector_challenges`](https://github.com/filecoin-project/rust-fil-proofs/blob/7809b42417f923ced415115e745b43a7a88a3459/filecoin-proofs/tests/api.rs#L901)
        3. Calculate `single_proof` by [`generate_single_vanilla_proof`](https://github.com/filecoin-project/rust-fil-proofs/blob/7809b42417f923ced415115e745b43a7a88a3459/filecoin-proofs/tests/api.rs#L907)
        4. Calculate groth16 proof by [`generate_winning_post_with_vanilla`](https://github.com/filecoin-project/rust-fil-proofs/blob/7809b42417f923ced415115e745b43a7a88a3459/filecoin-proofs/tests/api.rs#L916)
    2. all in one:
        1. Calculate groth16 proof by [`generate_winning_post`](https://github.com/filecoin-project/rust-fil-proofs/blob/7809b42417f923ced415115e745b43a7a88a3459/filecoin-proofs/tests/api.rs#L891) directly.
2. window post:
    1. staged:
        1. Calculate `challenges` by [`generate_fallback_sector_challenges`](https://github.com/filecoin-project/rust-fil-proofs/blob/7809b42417f923ced415115e745b43a7a88a3459/filecoin-proofs/tests/api.rs#L1364)
        2. Calculate `single_proof` by [`generate_single_vanilla_proof`](https://github.com/filecoin-project/rust-fil-proofs/blob/7809b42417f923ced415115e745b43a7a88a3459/filecoin-proofs/tests/api.rs#L1382)
        3. Calculate groth16 proof by [`generate_window_post_with_vanilla`](https://github.com/filecoin-project/rust-fil-proofs/blob/7809b42417f923ced415115e745b43a7a88a3459/filecoin-proofs/tests/api.rs#L1388)
    2. all in one:
        1. Calculate groth16 proof by [`generate_window_post`](https://github.com/filecoin-project/rust-fil-proofs/blob/7809b42417f923ced415115e745b43a7a88a3459/filecoin-proofs/tests/api.rs#L1353) directly.
