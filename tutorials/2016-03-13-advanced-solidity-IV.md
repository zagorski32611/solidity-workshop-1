# Advanced Solidity IV - Storage

This post is on contract storage, and how to work with it.

Storage, unlike memory and calldata, is a map. It stores one word per address, which is also different from memory and calldata where each address correspond to 1 byte. For example, `storage[0x0] = 0x755921` would store 32 bytes at address `0x0` (`0x0000...755921`), and `storage[0x1] = 0x55` would store 32 bytes at `0x1`. In memory, we would have to store the first word at `0x0` and the second at `0x20`.

The instructions used to operate on storage is `SSTORE` and `SLOAD`. They work exactly like `MSTORE` and `MLOAD`, except uses storage instead of memory. `SSTORE` has two operands: address and value. `SLOAD` has one operand: The address.
 
Here's the example from the tutorial on memory, but with storage instead.

`PUSH1 0x40 PUSH1 0x20 SSTORE PUSH1 0x20 SLOAD STOP`

Bytecode: `604060205560205400`

Output:

```
/usr/local/bin/node runcode.js
PC: 0
Opcode: PUSH1
Stack: []
Storage:  {}
PC: 2
Opcode: PUSH1
Stack: [ '40' ]
Storage:  {}
PC: 4
Opcode: SSTORE
Stack: [ '40', '20' ]
Storage:  {}
PC: 5
Opcode: PUSH1
Stack: []
Storage:  { '0000000000000000000000000000000000000000000000000000000000000020': '40' }
PC: 7
Opcode: SLOAD
Stack: [ '20' ]
Storage:  { '0000000000000000000000000000000000000000000000000000000000000020': '40' }
PC: 8
Opcode: STOP
Stack: [ '40' ]
Storage:  { '0000000000000000000000000000000000000000000000000000000000000020': '40' }

Process finished with exit code 0
```

*(Note that by default you don't see the storage key because secure trie hashes it)*

### Storage in Solidity
 
Storage is used for fields in Solidity contracts. This is automatic, so if you declare a field, the value will automatically be put in storage rather then memory. Solidity will also generate the storage addresses automatically according to [these rules](http://solidity.readthedocs.org/en/latest/miscellaneous.html#layout-of-state-variables-in-storage). Let's watch it in action.

First, copy this contract into the browser compiler. Make sure 'Enable Optimization' is checked.

```
contract C {
    uint x = 1;
}
```

Now look at the assembly.

```
.code
  PUSH 60			contract C {\n    uint x = 1;
...
  PUSH 40			contract C {\n    uint x = 1;
...
  MSTORE 			contract C {\n    uint x = 1;
...
  PUSH 1			1
  PUSH 0			uint x = 1
  PUSH 0			uint x = 1
  POP 			uint x = 1
  SSTORE 			uint x = 1
  PUSH #[$] 0000000000000000000000000000000000000000000000000000000000000000			contract C {\n    uint x = 1;
...
  DUP1 			contract C {\n    uint x = 1;
...
  PUSH [$] 0000000000000000000000000000000000000000000000000000000000000000			contract C {\n    uint x = 1;
...
  PUSH 0			contract C {\n    uint x = 1;
...
  CODECOPY 			contract C {\n    uint x = 1;
...
  PUSH 0			contract C {\n    uint x = 1;
...
  RETURN 			contract C {\n    uint x = 1;
...
.data
  0:
    .code
      PUSH 60			contract C {\n    uint x = 1;
...
      PUSH 40			contract C {\n    uint x = 1;
...
      MSTORE 			contract C {\n    uint x = 1;
...
      STOP 			contract C {\n    uint x = 1;
...
    .data
```

The part different from an empty contract init section is this:

```
  PUSH 1			1
  PUSH 0			uint x = 1
  PUSH 0			uint x = 1
  POP 			uint x = 1
  SSTORE 			uint x = 1
```

Notice there is an unnecessary push and pop in there, and if we remove them we get this:

```
  PUSH 1			1
  PUSH 0			uint x = 1
  SSTORE 			uint x = 1
```

This is the instruction to write `1` to storage address `0`, which is in accordance with the spec. `x` is the first field, and it's an `uint`.

Next we add another field.

```
contract C {
    uint x = 1;
    int y = 6;
}
```

The relevant assembly, with the unnecessary push/pops taken out:

```
  PUSH 1			1
  PUSH 0			uint x = 1
  SSTORE 			uint x = 1
  PUSH 6			6
  PUSH 1			int y = 6
  SSTORE 			int y = 6
```

Notice that it adds `x` at storage address `0x0`, and `y` at `0x1`.

Finally let's look at a mapping. 'Enable Optimization' must be unchecked this time.

```
contract C {
    
    mapping (uint => uint) map;
    
    function C() {
        map[1] = 2;
    }
    
}
```

The assembly:

```
.code
  PUSH 60			contract C {\n    
    mapping...
  PUSH 40			contract C {\n    
    mapping...
  MSTORE 			contract C {\n    
    mapping...
tag 1			function C() {\n        map[1]...
  JUMPDEST 			function C() {\n        map[1]...
  PUSH 2			2
  PUSH 0			map
  PUSH 0			map
  POP 			map
  PUSH 0			map[1]
  PUSH 1			1
  DUP2 			map[1]
  MSTORE 			map[1]
  PUSH 20			map[1]
  ADD 			map[1]
  SWAP1 			map[1]
  DUP2 			map[1]
  MSTORE 			map[1]
  PUSH 20			map[1]
  ADD 			map[1]
  PUSH 0			map[1]
  SHA3 			map[1]
  PUSH 0			map[1]
  POP 			map[1] = 2
  DUP2 			map[1] = 2
  SWAP1 			map[1] = 2
  SSTORE 			map[1] = 2
  POP 			map[1] = 2
tag 2			function C() {\n        map[1]...
  JUMPDEST 			function C() {\n        map[1]...
  PUSH #[$] 0000000000000000000000000000000000000000000000000000000000000000			contract C {\n    
    mapping...
  DUP1 			contract C {\n    
    mapping...
  PUSH [$] 0000000000000000000000000000000000000000000000000000000000000000			contract C {\n    
    mapping...
  PUSH 0			contract C {\n    
    mapping...
  CODECOPY 			contract C {\n    
    mapping...
  PUSH 0			contract C {\n    
    mapping...
  RETURN 			contract C {\n    
    mapping...
.data
  0:
    .code
      PUSH 60			contract C {\n    
    mapping...
      PUSH 40			contract C {\n    
    mapping...
      MSTORE 			contract C {\n    
    mapping...
      PUSH [tag] 1			contract C {\n    
    mapping...
      JUMP 			contract C {\n    
    mapping...
    tag 1			contract C {\n    
    mapping...
      JUMPDEST 			contract C {\n    
    mapping...
      STOP 			contract C {\n    
    mapping...
    .data
```

This is strange-looking. First of all, there are a number of `JUMPDEST`s, and it almost looks like it's jumping back and forth between the init and data sections. That's not the case, though. Solidity code-generation adds some (seemingly) redundant instructions here and there, but the optimizer deals with most of it. In this case we need to run un-optimized to show how SHA3 is used to generate addresses.

The relevant part is this:

```
  PUSH 2			2
  PUSH 0			map
  PUSH 0			map
  POP 			map
  PUSH 0			map[1]
  PUSH 1			1
  DUP2 			map[1]
  MSTORE 			map[1]
  PUSH 20			map[1]
  ADD 			map[1]
  SWAP1 			map[1]
  DUP2 			map[1]
  MSTORE 			map[1]
  PUSH 20			map[1]
  ADD 			map[1]
  PUSH 0			map[1]
  SHA3 			map[1]
  PUSH 0			map[1]
  POP 			map[1] = 2
  DUP2 			map[1] = 2
  SWAP1 			map[1] = 2
  SSTORE 			map[1] = 2
```

After some cleaning up, we get this:

```
  PUSH 2			2
  PUSH 0			map
  PUSH 0			map[1]
  PUSH 1			1
  DUP2 			map[1]
  MSTORE 			map[1]
  PUSH 20			map[1]
  ADD 			map[1]
  SWAP1 			map[1]
  DUP2 			map[1]
  MSTORE 			map[1]
  PUSH 20			map[1]
  ADD 			map[1]
  PUSH 0			map[1]
  SHA3 			map[1]
  DUP2 			map[1] = 2
  SWAP1 			map[1] = 2
  SSTORE 			map[1] = 2
  POP 			map[1] = 2
```

Like it says in the docs, the storage address algorithm is based on `SHA3`, which takes data from memory and hashes it. There are two inputs:

`SHA3 memoryOffset length`

`memoryOffset` is the first stack item, and tells it where in memory to start reading. 

`length` is the second stack item, and tells it how many bytes to read.

The solidity docs explains how the memory address of a map element is computed:

*... the value corresponding to a mapping key k is located at sha3(k . p) where . is concatenation ...*

In this case we know that `k = 1` and `p = 0`, so we expect the assembly code to write 1 and then 0 into memory (two words), and then run `SHA3` with an offset that matches the memory location of the first param, and a length of 64. Let's check.

After the first `MSTORE` we will have `memory[0x0] = 0x01`, and the stack will be: `[0x02, 0x00, 0x00]` (`0x02` is the value). Since `0x01` is the key, `SHA3` will have offset `0`. 

Next it pushes `0x20` and runs `ADD`, making the stack: `[0x02, 0x00, 0x20]`

After the next `MSTORE` we will have added `memory[0x20] = 0` and the stack becomes: `[0x02, 0x20]`.

Next it pushes `0x20` and adds, turning the stack to: `[0x02, 0x40]`. `0x40` is the number of bytes to read from memory (64).

Then it pushes `0x00`, which is the offset, and runs `SHA3`, meaning the stack is left with 2 items: `[0x02, SHA3_output ]`.

Finally it duplicates `0x02` and swaps it with the `SHA3` result and runs `SSTORE`. This will store `0x02` at storage address `SHA3_result`. `0x02` is the only remaining stack item, and is popped right after. 

Here is the output from the EVM:

```
/usr/local/bin/node runcode.js
PC: 0
Opcode: PUSH1
Stack: []
Memory: []
Storage:  {}
PC: 2
Opcode: PUSH1
Stack: [ '60' ]
Memory: []
Storage:  {}
PC: 4
Opcode: MSTORE
Stack: [ '60', '40' ]
Memory: []
Storage:  {}
PC: 5
Opcode: JUMPDEST
Stack: []
Memory: [00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,60]
Storage:  {}
PC: 6
Opcode: PUSH1
Stack: []
Memory: [00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,60]
Storage:  {}
PC: 8
Opcode: PUSH1
Stack: [ '02' ]
Memory: [00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,60]
Storage:  {}
PC: 10
Opcode: PUSH1
Stack: [ '02', '00' ]
Memory: [00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,60]
Storage:  {}
PC: 12
Opcode: POP
Stack: [ '02', '00', '00' ]
Memory: [00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,60]
Storage:  {}
PC: 13
Opcode: PUSH1
Stack: [ '02', '00' ]
Memory: [00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,60]
Storage:  {}
PC: 15
Opcode: PUSH1
Stack: [ '02', '00', '00' ]
Memory: [00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,60]
Storage:  {}
PC: 17
Opcode: DUP2
Stack: [ '02', '00', '00', '01' ]
Memory: [00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,60]
Storage:  {}
PC: 18
Opcode: MSTORE
Stack: [ '02', '00', '00', '01', '00' ]
Memory: [00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,60]
Storage:  {}
PC: 19
Opcode: PUSH1
Stack: [ '02', '00', '00' ]
Memory: [00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,01,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,60]
Storage:  {}
PC: 21
Opcode: ADD
Stack: [ '02', '00', '00', '20' ]
Memory: [00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,01,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,60]
Storage:  {}
PC: 22
Opcode: SWAP1
Stack: [ '02', '00', '20' ]
Memory: [00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,01,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,60]
Storage:  {}
PC: 23
Opcode: DUP2
Stack: [ '02', '20', '00' ]
Memory: [00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,01,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,60]
Storage:  {}
PC: 24
Opcode: MSTORE
Stack: [ '02', '20', '00', '20' ]
Memory: [00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,01,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,60]
Storage:  {}
PC: 25
Opcode: PUSH1
Stack: [ '02', '20' ]
Memory: [00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,01,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,60]
Storage:  {}
PC: 27
Opcode: ADD
Stack: [ '02', '20', '20' ]
Memory: [00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,01,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,60]
Storage:  {}
PC: 28
Opcode: PUSH1
Stack: [ '02', '40' ]
Memory: [00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,01,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,60]
Storage:  {}
PC: 30
Opcode: SHA3
Stack: [ '02', '40', '00' ]
Memory: [00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,01,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,60]
Storage:  {}
PC: 31
Opcode: PUSH1
Stack: [ '02',
  'ADA5013122D395BA3C54772283FB069B10426056EF8CA54750CB9BB552A59E7D' ]
Memory: [00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,01,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,60]
Storage:  {}
PC: 33
Opcode: POP
Stack: [ '02',
  'ADA5013122D395BA3C54772283FB069B10426056EF8CA54750CB9BB552A59E7D',
  '00' ]
Memory: [00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,01,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,60]
Storage:  {}
PC: 34
Opcode: DUP2
Stack: [ '02',
  'ADA5013122D395BA3C54772283FB069B10426056EF8CA54750CB9BB552A59E7D' ]
Memory: [00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,01,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,60]
Storage:  {}
PC: 35
Opcode: SWAP1
Stack: [ '02',
  'ADA5013122D395BA3C54772283FB069B10426056EF8CA54750CB9BB552A59E7D',
  '02' ]
Memory: [00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,01,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,60]
Storage:  {}
PC: 36
Opcode: SSTORE
Stack: [ '02',
  '02',
  'ADA5013122D395BA3C54772283FB069B10426056EF8CA54750CB9BB552A59E7D' ]
Memory: [00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,01,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,60]
Storage:  {}
PC: 37
Opcode: POP
Stack: [ '02' ]
Memory: [00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,01,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,60]
Storage:  { ada5013122d395ba3c54772283fb069b10426056ef8ca54750cb9bb552a59e7d: '02' }
PC: 38
Opcode: JUMPDEST
Stack: []
Memory: [00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,01,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,60]
Storage:  { ada5013122d395ba3c54772283fb069b10426056ef8ca54750cb9bb552a59e7d: '02' }
PC: 39
Opcode: PUSH1
Stack: []
Memory: [00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,01,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,60]
Storage:  { ada5013122d395ba3c54772283fb069b10426056ef8ca54750cb9bb552a59e7d: '02' }
PC: 41
Opcode: DUP1
Stack: [ '0A' ]
Memory: [00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,01,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,60]
Storage:  { ada5013122d395ba3c54772283fb069b10426056ef8ca54750cb9bb552a59e7d: '02' }
PC: 42
Opcode: PUSH1
Stack: [ '0A', '0A' ]
Memory: [00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,01,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,60]
Storage:  { ada5013122d395ba3c54772283fb069b10426056ef8ca54750cb9bb552a59e7d: '02' }
PC: 44
Opcode: PUSH1
Stack: [ '0A', '0A', '32' ]
Memory: [00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,01,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,60]
Storage:  { ada5013122d395ba3c54772283fb069b10426056ef8ca54750cb9bb552a59e7d: '02' }
PC: 46
Opcode: CODECOPY
Stack: [ '0A', '0A', '32', '00' ]
Memory: [00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,01,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,60]
Storage:  { ada5013122d395ba3c54772283fb069b10426056ef8ca54750cb9bb552a59e7d: '02' }
PC: 47
Opcode: PUSH1
Stack: [ '0A' ]
Memory: [60,60,60,40,52,60,08,56,5B,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,01,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,60]
Storage:  { ada5013122d395ba3c54772283fb069b10426056ef8ca54750cb9bb552a59e7d: '02' }
PC: 49
Opcode: RETURN
Stack: [ '0A', '00' ]
Memory: [60,60,60,40,52,60,08,56,5B,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,01,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,60]
Storage:  { ada5013122d395ba3c54772283fb069b10426056ef8ca54750cb9bb552a59e7d: '02' }

Returned: NaN

Process finished with exit code 0
```

Finally, I mentioned that the optimizer has to be disabled for this, and the reason is because some of the code will otherwise disappear. Enable the optimizer now and look at the output again, and you will see that the jumps and all of that is gone, and the code is much, much cleaner.

### Coming up

This concludes the basics, having covered the stack, flow control, calldata, memory and storage. The following posts will be mostly about Ethereum specific instructions, and Solidity related assembly code (such as its handling of types). There will also be some info about the optimizer, and case studies.