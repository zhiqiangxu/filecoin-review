# FEVM

`FEVM` is filecoin's first implemented foreign runtime for ecosystem compatibility as depicted in [FIP30](https://github.com/filecoin-project/FIPs/blob/6499f9ef4522e8bbdf718d96070a60e920c89f01/FIPS/fip-0030.md#abstract) and [fvm spec](https://github.com/filecoin-project/fvm-specs/blob/main/04-evm-mapping.md).

Broadly speaking, FEVM related change can be grouped into 3 classes:
1. At node layer, ethereum's eip1559 tx is [converted](https://github.com/filecoin-project/venus/blob/13ed0531a8d27dddd397ceed58a88cd3b3d85541/venus-shared/actors/types/eth_transactions.go#L199) into a filecoin tx.
    1. Depending on whether `to` is nil, [`EamActor.CreateExternal`](https://github.com/filecoin-project/builtin-actors/blob/424669ba4e38a1a8557626d411cd0c89389f4fcd/actors/eam/src/lib.rs#L277-L288) or [`EvmContractActor.InvokeContract`](https://github.com/filecoin-project/builtin-actors/blob/424669ba4e38a1a8557626d411cd0c89389f4fcd/actors/evm/src/lib.rs#L232-L262) is [called](https://github.com/filecoin-project/venus/blob/13ed0531a8d27dddd397ceed58a88cd3b3d85541/venus-shared/actors/types/eth_transactions.go#L174-L179).
2. At fvm layer, an `Ethereum Externally Owned Address` or a [placeholder actor](https://docs.filecoin.io/smart-contracts/filecoin-evm-runtime/actor-types/#placeholder) that has an f4 address in the EAM's namespace are additionally [regarded](https://github.com/filecoin-project/ref-fvm/blob/c1bcbff0e1e22a7eed27590e521bd90093e2c78e/fvm/src/executor/default.rs#L410-L428) as a valid sender.
    1. The placeholder actor is automatically [upgraded](https://github.com/filecoin-project/ref-fvm/blob/c1bcbff0e1e22a7eed27590e521bd90093e2c78e/fvm/src/executor/default.rs#L420-L428) into a ethaccount actor upon first tx.
3. At builtin actors layer, the entire evm runtime is implemented as a builtin actor on top of FVM through runtime emulation.


The bulk of evm interpreter logic is implemented by [`EvmContractActor`](https://github.com/filecoin-project/builtin-actors/blob/424669ba4e38a1a8557626d411cd0c89389f4fcd/actors/evm/src/lib.rs#L405), while [`EamActor`](https://github.com/filecoin-project/builtin-actors/blob/424669ba4e38a1a8557626d411cd0c89389f4fcd/actors/eam/src/lib.rs#L291) is responsible for managing ethereum accounts.

Let's dive into the details of `EamActor.CreateExternal` and `EvmContractActor.InvokeContract` to have a taste of both.

## `EamActor.CreateExternal`

1. [checks](https://github.com/filecoin-project/builtin-actors/blob/424669ba4e38a1a8557626d411cd0c89389f4fcd/actors/eam/src/lib.rs#L283) if the caller is the same as the origin.
2. [computes](https://github.com/filecoin-project/builtin-actors/blob/424669ba4e38a1a8557626d411cd0c89389f4fcd/actors/eam/src/lib.rs#L286) the address of the actor to be created, obeying the same rule as `CREATE`.
3. [call](https://github.com/filecoin-project/builtin-actors/blob/424669ba4e38a1a8557626d411cd0c89389f4fcd/actors/eam/src/lib.rs#L287) `create_actor`.
    1. A `Exec4Params` struct is [filled out](https://github.com/filecoin-project/builtin-actors/blob/424669ba4e38a1a8557626d411cd0c89389f4fcd/actors/eam/src/lib.rs#L159-L163), specifying the `EvmContractActor` as the code of the actor to be created(`Exec4Params.code_cid`), the construct parameters, and the target address.
    2. [Call](https://github.com/filecoin-project/builtin-actors/blob/424669ba4e38a1a8557626d411cd0c89389f4fcd/actors/eam/src/lib.rs#L165-L170) [InitActor.exec4](https://github.com/filecoin-project/builtin-actors/blob/424669ba4e38a1a8557626d411cd0c89389f4fcd/actors/init/src/lib.rs#L111) with said `Exec4Params`.
        1. `InitActor.exec4` [checks](https://github.com/filecoin-project/builtin-actors/blob/424669ba4e38a1a8557626d411cd0c89389f4fcd/actors/init/src/lib.rs#L112) the caller is `EamActor`.
        2. A delegated f4 address is [computed](https://github.com/filecoin-project/builtin-actors/blob/424669ba4e38a1a8557626d411cd0c89389f4fcd/actors/init/src/lib.rs#L115-L118) associating the target address and `EamActor`.
        3. [Computes](https://github.com/filecoin-project/builtin-actors/blob/424669ba4e38a1a8557626d411cd0c89389f4fcd/actors/init/src/lib.rs#L126) an f2 address from `sender || nonce || # of actors created during message execution`.
        4. [Allocates](https://github.com/filecoin-project/builtin-actors/blob/424669ba4e38a1a8557626d411cd0c89389f4fcd/actors/init/src/lib.rs#L132) an f0 address.
        5. [Persists](https://github.com/filecoin-project/builtin-actors/blob/424669ba4e38a1a8557626d411cd0c89389f4fcd/actors/init/src/state.rs#L37) the map `f2 => f0` and `f4 => f0`.
        6. Actually [creates](https://github.com/filecoin-project/builtin-actors/blob/424669ba4e38a1a8557626d411cd0c89389f4fcd/actors/init/src/lib.rs#L152) an empty actor with `Exec4Params.code_cid`, f0 address and delegated f4 address.
        7. Call [`EvmContractActor.constructor`](https://github.com/filecoin-project/builtin-actors/blob/424669ba4e38a1a8557626d411cd0c89389f4fcd/actors/evm/src/lib.rs#L191) with `Exec4Params.constructor_params`.
            1. Notice that each time an actor is called, the context automatically [switches](https://github.com/filecoin-project/ref-fvm/blob/c1bcbff0e1e22a7eed27590e521bd90093e2c78e/fvm/src/call_manager/default.rs#L694-L703) to that actor, e.g, `self` changes.
            2. `EvmContractActor.constructor` [checks](https://github.com/filecoin-project/builtin-actors/blob/424669ba4e38a1a8557626d411cd0c89389f4fcd/actors/evm/src/lib.rs#L196) the calls is `InitActor`.
            3. [Initializes](https://github.com/filecoin-project/builtin-actors/blob/424669ba4e38a1a8557626d411cd0c89389f4fcd/actors/evm/src/lib.rs#L197) the actor state.
                1. The succesful output of the initcode is [saved](https://github.com/filecoin-project/builtin-actors/blob/424669ba4e38a1a8557626d411cd0c89389f4fcd/actors/evm/src/lib.rs#L135-L140) into the actor state as evm runtime code.
        8. Return f0 and f2 address created above.


## `EvmContractActor.InvokeContract`

1. [Loads](https://github.com/filecoin-project/builtin-actors/blob/424669ba4e38a1a8557626d411cd0c89389f4fcd/actors/evm/src/lib.rs#L242) the state saved by `EvmContractActor.constructor`.
2. [Calls](https://github.com/filecoin-project/builtin-actors/blob/424669ba4e38a1a8557626d411cd0c89389f4fcd/actors/evm/src/lib.rs#L254) [`invoke_contract_inner`](https://github.com/filecoin-project/builtin-actors/blob/424669ba4e38a1a8557626d411cd0c89389f4fcd/actors/evm/src/lib.rs#L151).
    1. [Fetches](https://github.com/filecoin-project/builtin-actors/blob/424669ba4e38a1a8557626d411cd0c89389f4fcd/actors/evm/src/lib.rs#L162) runtime code.
    2. [Executes](https://github.com/filecoin-project/builtin-actors/blob/424669ba4e38a1a8557626d411cd0c89389f4fcd/actors/evm/src/lib.rs#L175) runtime code.
        1. The execution [honestly](https://github.com/filecoin-project/builtin-actors/blob/424669ba4e38a1a8557626d411cd0c89389f4fcd/actors/evm/src/interpreter/execution.rs#L258-L275) evaluates each opcode and advances to the next.
        2. There's no dedicated gas calculation for evm opcodes, as the entire wasm code has already been [metered](https://github.com/zhiqiangxu/filecoin-review/blob/main/fvm.md#advanced).
    3. If succesful, state is [persisted](https://github.com/filecoin-project/builtin-actors/blob/424669ba4e38a1a8557626d411cd0c89389f4fcd/actors/evm/src/lib.rs#L179).

