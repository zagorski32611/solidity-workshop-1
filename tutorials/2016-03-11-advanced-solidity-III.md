# Advanced Solidity III - Memory and Calldata

This post is about memory and calldata.

### Memory

Memory is an expandable byte-array used to store data during program execution. It can be accessed using the `MSTORE` and `MLOAD` instructions. Here is an example of how writing to memory can look:

`PUSH1 0x40 PUSH1 0x20 MSTORE`

`MSTORE` is described on page 26 of the yellow-paper. I mention it in the first part of this tutorial series as well. It takes two stack-items; the first is the offset, and the second is the value that should be stored.
 
`MSTORE offset value`.

In array notation, this means the sequence above means `memory[0x20] = 0x40`. As usual, the instruction clears the operands from the stack.

Reading from memory is done using `MLOAD`. To read the value stored at `0x20` we would write:

`PUSH1 0x20 MLOAD`

To illustrate this, let's combine the write and the read, convert it to bytecode and run it in the javascript-vm.

`PUSH1 0x40 PUSH1 0x20 MSTORE PUSH1 0x20 MLOAD STOP`

`MSTORE` is `0x52`, `MLOAD` is `0x51`, `PUSH1`is `0x60`, and `STOP` is `0x00`, so the code becomes:

`604060205260205100`

This is the output:

```
/usr/local/bin/node runcode.js
PC: 0
Opcode: PUSH1
Stack: []
Memory: []
PC: 2
Opcode: PUSH1
Stack: [ '40' ]
Memory: []
PC: 4
Opcode: MSTORE
Stack: [ '40', '20' ]
Memory: []
PC: 5
Opcode: PUSH1
Stack: []
Memory: [00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,40]
PC: 7
Opcode: MLOAD
Stack: [ '20' ]
Memory: [00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,40]
PC: 8
Opcode: STOP
Stack: [ '40' ]
Memory: [00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,40]
returned: 

Process finished with exit code 0
```

Looking at the stack, it looks about right. First it pushes `0x40` and `0x20`, runs `MSTORE` which clears the stack. At 5 it pushes `0x20`, then runs `MLOAD` which pops `0x20` and adds `0x40` instead - which is what is stored at memory address `0x20`.
  
The memory looks a bit weird though. The byte `0x40` is there, but there are a lot of zeroes as well. The reason is that when writing to memory using `MSTORE`, it will write a full word (32 bytes). It will also load a full 32 bytes when using `MLOAD`. The length of the array is 64 bytes, and the `40` is at index 63 because the values are right-oriented. 

This storing and loading mechanics is why offsets of 32 (`0x20`) are often used when writing, to avoid values overlapping. If writes aren't done right, it may result in bytes being overwritten.

Finally, there's also an `MSTORE8` which allows you to write a single byte. This will store one byte at memory address 0:

`PUSH1 0x40 PUSH1 0x00 MSTORE8 STOP`

Bytecode: `604060005300`

Output:

```
/usr/local/bin/node runcode.js
PC: 0
Opcode: PUSH1
Stack: []
Memory: []
PC: 2
Opcode: PUSH1
Stack: [ '40' ]
Memory: []
PC: 4
Opcode: MSTORE8
Stack: [ '40', '00' ]
Memory: []
PC: 5
Opcode: STOP
Stack: []
Memory: [40]
returned: 

Process finished with exit code 0
```

### Calldata

Calldata is a byte-array, just like memory, but it is read-only. It contains the `data` from the transaction that triggered the code execution.
 
The way you access calldata is through the `CALLDATALOAD` instruction (YP, page 32). It takes one parameter, which is the offset, and just like memory it will read a full 32 bytes starting from that offset. If the calldata does not have the full amount of bytes, or the index is out-of-bounds, it will simply be padded with zeroes.

For this example, I'm going to pass the value `0x40` into the javascript VM as calldata, although that can't be seen here. The code for reading it is:

`PUSH1 0x00 CALLDATALOAD STOP`

`CALLDATALOAD` has value `0x35`, so the bytecode becomes:

`60003500`

Output:

```
/usr/local/bin/node runcode.js
PC: 0
Opcode: PUSH1
Stack: []
PC: 2
Opcode: CALLDATALOAD
Stack: [ '00' ]
PC: 3
Opcode: STOP
Stack: [ '4000000000000000000000000000000000000000000000000000000000000000' ]
returned: 

Process finished with exit code 0

```

There are two more instructions related to calldata, `CALLDATASIZE`, which takes no input and pushes the length of the calldata array onto the stack, and `CALLDATACOPY` which copies calldata to memory. `CALLDATACOPY` works in much the same way as `CODECOPY`. It has three operands:

`CALLDATACOPY memoryOffset callDataOffset callDataLength`

Finally, it's worth mentioning that calldata is not passed to functions, but to the VM. It is passed to the VM when execution begins, and remains static. This means if you want to read function input params you need to take this into account and start reading from the fifth byte and onwards (as mentioned in part 1, the first four bytes of calldata is reserved for function identifiers).
 
### Example of calldata, memory, and RETURN.

To finish this up, we will create a program that takes the first 32 bytes of call-data, copies it to memory, and returns it. This will require `CALLDATACOPY` and `RETURN`. I mentioned return in the first tutorial. It is listed on page 29 in the yellow paper. The input is:
 
`RETURN memOffset length`

The first param specifies the memory address where it should start reading, and the length is the total amount of bytes it should read.

--

**WARNING**: *In low-level code I tend to keep return-data length at multiples of 32. Even though it's not the most efficient way of doing things it adds consistency, and reduces the chance of errors. This is important when calling contracts from other contracts, because as we will see later, the `CALL` instruction requires that you allocate memory space for the return data beforehand. For efficiency reasons (memory expansion costs gas) you usually want to re-use old memory when possible - so if you get something like 50 bytes returned and use two full words of memory for it (which is normal), it will not overwrite all of the old bytes. Forgetting to clear manually before the `CALL` could lead to data being corrupted. Returning data in multiples of 32 bytes is safer, cleaner, and is therefore usually better IMO.*

--

Let's start with the program. It is simple in theory - we do a `CALLDATACOPY` followed by a `RETURN`. No manual memory writes is required. We want to copy 32 bytes from calldata, starting at address 0, and put it in memory. We will start writing at memory address 0, so according to the "definition" above, this is what we get:

`CALLDATACOPY 0x00 0x00 0x20`

Next we want to return those bytes from memory, and since we started at address 0 and wrote 32 bytes, this is what we get:

`RETURN 0x00 0x20`

Now all we need to do is to reverse it so that the data is added to the stack in the proper order.

`PUSH1 0x20 PUSH1 0x00 PUSH1 0x00 CALLDATACOPY PUSH1 0x20 PUSH1 0x0 RETURN`

Bytecode: `6020600060003760206000F3`

Output:

```
/usr/local/bin/node runcode.js
PC: 0
Opcode: PUSH1
Stack: []
Memory: []
PC: 2
Opcode: PUSH1
Stack: [ '20' ]
Memory: []
PC: 4
Opcode: PUSH1
Stack: [ '20', '00' ]
Memory: []
PC: 6
Opcode: CALLDATACOPY
Stack: [ '20', '00', '00' ]
Memory: []
PC: 7
Opcode: PUSH1
Stack: []
Memory: [40,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00]
PC: 9
Opcode: PUSH1
Stack: [ '20' ]
Memory: [40,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00]
PC: 11
Opcode: RETURN
Stack: [ '20', '00' ]
Memory: [40,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00,00]
returned:  4000000000000000000000000000000000000000000000000000000000000000

Process finished with exit code 0
```

Notice that I use the same calldata as in the previous example - the first (and only) byte being `0x40`. The reason there are so many zeroes is (again) because it is padded to make it a full 32 bytes.

### Coming up

The next post will be about storage. I will explain how to read and write, and point out some of the main differences between storage and memory/calldata. After that it will be mostly about specific instructions, and examples of how they are used.