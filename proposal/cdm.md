# Capped Duration Multiplier


## Background

Hybrid Minting Model:
1. A portion of minting rewards come from simple exponential decay (simple minting) and the remainder from network baseline minting.
2. The network disburses maximum rewards when network RBP exceeds the baseline storage target, a key performance indicator (KPI).
3. The baseline storage capacity is currently targeted to double network storage capacity annually, but this growth rate can be modified through approved Filecoin Improvement Proposals (FIPs).

Baseline Crossing:
1. When the network’s storage capacity either exceeds or falls below the network’s baseline storage target.
2. The first baseline crossing, occured in April 2021 when the network’s RBP exceeded the baseline storage target.
3. However, on February 9, 2023, the Filecoin network’s capacity fell below the storage capacity target function, “crossing the baseline from above”. 

Reward Calculation:
```rust
/// The projected block reward a sector would earn over some period.
/// Also known as "BR(t)".
/// BR(t) = ProjectedRewardFraction(t) * SectorQualityAdjustedPower
/// ProjectedRewardFraction(t) is the sum of estimated reward over estimated total power
/// over all epochs in the projection period [t t+projectionDuration]
pub fn expected_reward_for_power(
    reward_estimate: &FilterEstimate,
    network_qa_power_estimate: &FilterEstimate,
    qa_sector_power: &StoragePower,
    projection_duration: ChainEpoch,
) -> TokenAmount {
    let network_qa_power_smoothed = network_qa_power_estimate.estimate();

    if network_qa_power_smoothed.is_zero() {
        return TokenAmount::from_atto(reward_estimate.estimate());
    }

    let expected_reward_for_proving_period = smooth::extrapolated_cum_sum_of_ratio(
        projection_duration,
        0,
        reward_estimate,
        network_qa_power_estimate,
    );
    let br128 = qa_sector_power * expected_reward_for_proving_period; // Q.0 * Q.128 => Q.128
    TokenAmount::from_atto(std::cmp::max(br128 >> PRECISION, Default::default()))
}

/// DealWeight and VerifiedDealWeight are spacetime occupied by regular deals and verified deals in a sector.
/// Sum of DealWeight and VerifiedDealWeight should be less than or equal to total SpaceTime of a sector.
/// Sectors full of VerifiedDeals will have a SectorQuality of VerifiedDealWeightMultiplier/QualityBaseMultiplier.
/// Sectors full of Deals will have a SectorQuality of DealWeightMultiplier/QualityBaseMultiplier.
/// Sectors with neither will have a SectorQuality of QualityBaseMultiplier/QualityBaseMultiplier.
/// SectorQuality of a sector is a weighted average of multipliers based on their proportions.
pub fn quality_for_weight(
    size: SectorSize,
    duration: ChainEpoch,
    deal_weight: &DealWeight,
    verified_weight: &DealWeight,
) -> SectorQuality {
    let sector_space_time = BigInt::from(size as u64) * BigInt::from(duration);
    let total_deal_space_time = deal_weight + verified_weight;

    let weighted_base_space_time =
        (&sector_space_time - total_deal_space_time) * &*QUALITY_BASE_MULTIPLIER;
    let weighted_deal_space_time = deal_weight * &*DEAL_WEIGHT_MULTIPLIER;
    let weighted_verified_space_time = verified_weight * &*VERIFIED_DEAL_WEIGHT_MULTIPLIER;
    let weighted_sum_space_time =
        weighted_base_space_time + weighted_deal_space_time + weighted_verified_space_time;
    let scaled_up_weighted_sum_space_time: SectorQuality =
        weighted_sum_space_time << SECTOR_QUALITY_PRECISION;

    scaled_up_weighted_sum_space_time
        .div_floor(&sector_space_time)
        .div_floor(&QUALITY_BASE_MULTIPLIER)
}

/// Returns the power for a sector size and weight.
pub fn qa_power_for_weight(
    size: SectorSize,
    duration: ChainEpoch,
    deal_weight: &DealWeight,
    verified_weight: &DealWeight,
) -> StoragePower {
    let quality = quality_for_weight(size, duration, deal_weight, verified_weight);
    (BigInt::from(size as u64) * quality) >> SECTOR_QUALITY_PRECISION
}

// compute initial pledge
let qa_pow = qa_power_for_weight(
    info.sector_size,
    duration,
    &new_sector_info.deal_weight,
    &new_sector_info.verified_deal_weight,
);

new_sector_info.expected_day_reward = expected_reward_for_power(
    &rew.this_epoch_reward_smoothed,
    &pow.quality_adj_power_smoothed,
    &qa_pow,
    fil_actors_runtime::network::EPOCHS_IN_DAY,
);
```

This proposal is essentially about changing the calculation of `qa_pow` above.

## Specification

1. A strict superset of [FIP56](https://github.com/filecoin-project/FIPs/blob/e80e4277a7592e1ce2b9f57a8e5d069251e97372/FIPS/fip-0056.md).
    1. SDM
        1. code:
            ```
            fn sdm(duration_commitment: float) -> float {
                if duration_commitment <= (3/2 * EPOCHS_IN_YEAR) {
                    1
                } else {
                    (duration_commitment - EPOCHS_IN_YEAR/2) / EPOCHS_IN_YEAR
                }
            }        

            ...

            new_sector_info.expected_day_reward = expected_reward_for_power(
                &rew.this_epoch_reward_smoothed,
                &pow.quality_adj_power_smoothed,
                &qa_pow,
                fil_actors_runtime::network::EPOCHS_IN_DAY,
            ) * sdm(duration);

            new_sector_info.expected_storage_pledge = expected_reward_for_power(
                &rew.this_epoch_reward_smoothed,
                &pow.quality_adj_power_smoothed,
                &qa_pow,
                INITIAL_PLEDGE_PROJECTION_PERIOD,
            ) * sdm(duration);        

            ...

            let penalized_reward =  max(
                SP(t), 
                BR(StartEpoch, 20d) + BR(StartEpoch, 1d) * terminationRewardFactor * min(SectorAgeInDays, 140*sdm(duration_commitment))
            )        
            ```
        2. formula:
            ```math
            sectorQuality = (avgQuality \cdot sectorSize) \cdot SDM
            ```
2. Increase [MAX_SECTOR_EXPIRATION_EXTENSION](https://github.com/filecoin-project/builtin-actors/blob/38a7cd55936467cc91376aded392611f4faa3dc9/runtime/src/runtime/policy.rs#L367), and the corresponding [max_prove_commit_duration](https://github.com/filecoin-project/builtin-actors/blob/38a7cd55936467cc91376aded392611f4faa3dc9/actors/miner/src/policy.rs#L84-L85) to 3700 fildays (~10.13 years).
3. Cap the maximum QAP available to a sector, by defining CDM as:
    ```
    FILP_QAP_FACTOR=10          // same as mainnet
    MAX_SECTOR_QAP=10           // !!!NEW!!!
    SDM_SLOPE=1x-per-360days    // same as FIP56/36
    SDM_LAG=540days             // same as FIP56
    MIN_SECTOR_DURATION=360days // same as FIP56

    SECTOR_QAP = MIN(
        MAX_SECTOR_QAP,
        (
        MAX( MIN_SECTOR_DURATION, ( SECTOR_DURATION - SDM_LAG ) )
            *
        SDM_SLOPE
            *
        (
            SECTOR_SIZE - FILP_PIECES_SIZE      // CC or Data
                +
            FILP_QAP_FACTOR * FILP_PIECES_SIZE  // Fil+ client data
        )
        )
    )
    ```

## Incentive Considerations

[solution space of combining the two modes of operation](https://www.wolframalpha.com/input?i=3d+plot&assumption=%7B%22F%22%2C+%223DPlot%22%2C+%223dplotupperrange2%22%7D+-%3E%221%22&assumption=%7B%22F%22%2C+%223DPlot%22%2C+%223dplotupperrange1%22%7D+-%3E%2210.15%22&assumption=%7B%22MC%22%2C+%223d+plot%22%7D+-%3E+%7B%22Calculator%22%7D&assumption=%7B%22F%22%2C+%223DPlot%22%2C+%223dplotfunction%22%7D+-%3E%22%3D+MIN%28+10%2C+%28+max%281%2C+y-1.5%29+*+%28+1-x+%2B+10*x+%29+%29+%29%22&assumption=%7B%22F%22%2C+%223DPlot%22%2C+%223dplotvariable1%22%7D+-%3E%22y%22&assumption=%7B%22F%22%2C+%223DPlot%22%2C+%223dplotlowerrange1%22%7D+-%3E%221%22&assumption=%7B%22F%22%2C+%223DPlot%22%2C+%223dplotvariable2%22%7D+-%3E%22x%22&assumption=%7B%22F%22%2C+%223DPlot%22%2C+%223dplotlowerrange2%22%7D+-%3E%220%22)

|Fil+ Exposure|Min Rational Duration|Effective QAP|
|---|---|---|
|100%|MIN_SECTOR_DURATION|10x|
|80%|2.72 years|10x|
|75%|2.80 years|10x|
|50%|3.32 years|10x|
|33%|4.02 years|10x|
|25%|4.58 years|10x|
|20%|5.08 years|10x|
|15%|5.76 years|10x|
|10%|6.77 years|10x|
|5%|8.40 years|10x|
|2%|9.98 years|10x|
|1%|MAX_SECTOR_DURATION|9.43x|
|0%|MAX_SECTOR_DURATION|8.65x|