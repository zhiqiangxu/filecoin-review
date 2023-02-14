# FVM

## Basic

The function to create a `FVM` instance is [`create_fvm_machine`](https://github.com/filecoin-project/filecoin-ffi/blob/master/rust/src/fvm/machine.rs#L108).

The main logic is: `MultiEngineContainer` [maintains](https://github.com/filecoin-project/filecoin-ffi/blob/master/rust/src/fvm/machine.rs#L95) a single `MultiEngine` for each specific network version, while `MultiEngine` [maintains](https://github.com/filecoin-project/ref-fvm/blob/master/fvm/src/engine.rs#L71) a single `EnginePool` for each specific `EngineConfig`. The `EnginePool` together with [`Machine`](https://github.com/filecoin-project/ref-fvm/blob/master/fvm/src/machine/default.rs#L32) are further [wrapped up](https://github.com/filecoin-project/filecoin-ffi/blob/master/rust/src/fvm/engine.rs#L180) as `Executor`, which is the core of `FVM`.  This `Executor` [preloads](https://github.com/filecoin-project/ref-fvm/blob/master/fvm/src/executor/default.rs#L318) builtin actors on instantiation. 

The function to actually apply a transaction upon the created `FVM` instance is [`fvm_machine_execute_message`](https://github.com/filecoin-project/filecoin-ffi/blob/master/rust/src/fvm/machine.rs#L183).

The main logic is: it calls the [`execute_message`](https://github.com/filecoin-project/filecoin-ffi/blob/master/rust/src/fvm/machine.rs#L204) method of the previously created executor, which finally calls [`DefaultExecutor::execute_message`](https://github.com/filecoin-project/ref-fvm/blob/master/fvm/src/executor/default.rs#L61). It then acquires an [`FVM Engine`](https://github.com/filecoin-project/ref-fvm/blob/master/fvm/src/engine.rs#L296) by calling [`engine_pool.acquire`](https://github.com/filecoin-project/ref-fvm/blob/master/fvm/src/executor/default.rs#L99), this method will honor the concurrency limit configuration. 

Before actually diving into the WebAssembly execution details, it's better to clarify some concepts from `wasmtime`:
1. `Module`: a compiled in-memory representation of an input WebAssembly binary, ready to be instantiated.
2. `Instance`: An instantiated WebAssembly module, ready to be called.
3. `Store`: a container for WebAssembly instances and **host-defined state**.

The **host-defined state** is the key for actually importing customized syscalls into WebAssembly for module model.(There's also a [component model](https://github.com/WebAssembly/component-model/blob/main/design/mvp/Explainer.md) which is [not used by `FVM`](https://github.com/filecoin-project/ref-fvm/blob/master/fvm/src/engine.rs#L392).)

For `FVM` the host-defined state is [`InvocationData`](https://github.com/filecoin-project/ref-fvm/blob/master/fvm/src/syscalls/mod.rs#L32). When WebAssembly calls the imported syscall, `wasmtime` will [callback to the implementation with the `InvocationData` attached](https://github.com/filecoin-project/ref-fvm/blob/master/fvm/src/syscalls/bind.rs#L117).

OK let's talk back to `FVM Engine`.

On a top level, `FVM Engine` does this:
1. [Creates a `CallManager`](https://github.com/filecoin-project/ref-fvm/blob/master/fvm/src/executor/default.rs#L104) for that message, giving itself to the `CallManager`.
2. [Call the specified actor/method](https://github.com/filecoin-project/ref-fvm/blob/master/fvm/src/executor/default.rs#L144) using `CallManager::send`
3. The `CallManager` then [constructs a `Kernel`](https://github.com/filecoin-project/ref-fvm/blob/master/fvm/src/call_manager/default.rs#L695), which actually implements the FVM interface as presented to the actors, [detaching itself](https://github.com/filecoin-project/ref-fvm/blob/master/fvm/src/call_manager/default.rs#L691) to the `Kernel`.
4. The `CallManager` then [constructs a `Store`](https://github.com/filecoin-project/ref-fvm/blob/master/fvm/src/call_manager/default.rs#L706), first wrapping up the `Kernel` into `InvocationData`, then giving the `InvocationData` to the `Store`.
5. The `CallManager` then [instantiates a WebAssembly instance](https://github.com/filecoin-project/ref-fvm/blob/master/fvm/src/call_manager/default.rs#L711) 
6. The `CallManager` then [invokes the above WebAssembly instance](https://github.com/filecoin-project/ref-fvm/blob/master/fvm/src/call_manager/default.rs#L735).
    1. For builtin actors, it calls the exported [`invoke` function](https://github.com/filecoin-project/builtin-actors/blob/master/runtime/src/lib.rs#L39)
    2. The `invoke` function then calls the framework function [`trampoline`](https://github.com/filecoin-project/builtin-actors/blob/master/runtime/src/runtime/fvm.rs#L565)
    3. `trampoline` fetches `method` and `params` then calls [`invoke_method`](https://github.com/filecoin-project/builtin-actors/blob/master/runtime/src/runtime/fvm.rs#L578).
    4. `invoke_method` [calls](https://github.com/filecoin-project/builtin-actors/blob/master/runtime/src/runtime/actor_code.rs#L15) the current actor/method being called and returns.  
7. The `CallManager` then returns the `InvocationResult` and re-attaches itself.

If a actor calls another actor:
1. The [`send`](https://github.com/filecoin-project/ref-fvm/blob/master/fvm/src/syscalls/send.rs#L16) syscall will be called.
2. [`kernel.send`](https://github.com/filecoin-project/ref-fvm/blob/master/fvm/src/syscalls/send.rs#L46) will be called.
3. The `Kernel` then calls [`CallManager::send`](https://github.com/filecoin-project/ref-fvm/blob/master/fvm/src/kernel/default.rs#L393)
4. Similar to the 3-7 steps above, then the final result is returned to `wasmtime` which continues the WebAssembly execution.

This summarizes the basic process of instantiating and executing a `FVM`.

## Advanced

`FVM` automatically applies two instruments to every WebAssembly module before execution:
1. [`stack limiter`](https://github.com/filecoin-project/ref-fvm/blob/master/fvm/src/engine.rs#L404)
2. [`gas metering`](https://github.com/filecoin-project/ref-fvm/blob/master/fvm/src/engine.rs#L419)

---
The `stack limiter` instrument basically inserts pre/post-ambles before and after each WebAssembly call opcode, aiming to limit the stack depth. But there're 3 corner cases to fix:
1. exported function
2. start function
3. table indirect function

For the above functions, a wrapper function is generated, which adds pre/post-ambles before and after calling the target function, all external references to these functions are then replaced with corresponding wrapper function.

---

The `gas metering` instrument aims to "gas" each instruction, but doing it cleverly: instead of doing it opcode by opcode, it's doing it block by block. It partitions the opcodes into blocks which is guaranteed to run from start to end, namely [`MeteredBlock`](https://github.com/filecoin-project/fvm-wasm-instrument/blob/master/src/gas_metering/mod.rs#L161). `MeteredBlock` is similar to [basic block](https://en.wikipedia.org/wiki/Basic_block) in compiler theory.

This kind of task is very hard to implement for instruction set with `goto`-alike instruction, which WebAssembly avoids by design. Moreover WebAssembly natively has the concept of blocks, corresponding to the [`ControlBlock`](https://github.com/filecoin-project/fvm-wasm-instrument/blob/master/src/gas_metering/mod.rs#L138) in code. There're 4 kinds of `ControlBlock`:
1. function body block(implicitly created on entry)
2. `block`/`if`/`loop` opcode initiated block

These `ControlBlock`s can only be ended by `end` opcode. Thus we can identify all `ControlBlock`s.

The `MeteredBlock` initially starts at the same position as the containing `ControlBlock`, but can be ended in 7 opcodes: `else`/`br`/`brif`/`brtable`/`return`/`end`. The situation with `end` opcode is the most tricky: when `end` opcode is found, and if the **current** `ControlBlock` contains opcodes which may jump over the **last** `ControlBlock`, then should also mark the active `MeteredBlock` of the last `ControlBlock` as finished. The reason is crystal clear if you draw the situation on the paper. It is worth mentioning that the end position of the previous `MeteredBlock` is also the start position of the next `MeteredBlock` in the same `ControlBlock`.

Thus we can identify all `MeteredBlock`s. But there's a corner case to fix: there're some opcodes whose gas can't be calculated statically(e.g. `memory.grow` with non constant parameter), corresponding to the [`MeteredInstruction`](https://github.com/filecoin-project/fvm-wasm-instrument/blob/master/src/gas_metering/mod.rs#L170) in code. These opcodes have linear gas cost and are identified as `MeteredInstruction`s.

In order to handle the identified `ControlBlock`s and `MeteredInstruction`s, `FVM` will inject a `gas_func(cost i64)` function to the end of the current WebAssembly module, and an imported `gas_counter` global variable to record the remaining gas. The [logic](https://github.com/filecoin-project/fvm-wasm-instrument/blob/master/src/gas_metering/mod.rs#L735) of `gas_func` is to subtract cost from `gas_counter`, and panics if negative. Because of the imported `gas_counter` global variable, the pointers in Code/Export/Element/Data sections also need to be adjusted.

After these preparations, for each `MeteredBlock`, a [call](https://github.com/filecoin-project/fvm-wasm-instrument/blob/master/src/gas_metering/mod.rs#L814) to `gas_func` with cost `charge_cost+MeteredBlock.cost` is inserted right before where `MeteredBlock` start; For each `MeteredInstruction`, a [call](https://github.com/filecoin-project/fvm-wasm-instrument/blob/master/src/gas_metering/mod.rs#L826) to `gas_func` with cost `stack top * MeteredBlock.unit_cost` is inserted right before where `MeteredInstruction` start(The gas for the base part is already charged in the `MeteredBlock` that contains it).

