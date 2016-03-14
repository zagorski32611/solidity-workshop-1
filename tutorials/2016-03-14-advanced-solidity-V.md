# Advanced Solidity V - Solidity value-types

This post contains some information about basic Solidity types, their properties, and how they are translated into EVM bytecode. 

### int, uint, boolean

Integers in Solidity can be either signed or unsigned, and can be on the form `intN/uintN`, where `N = 8*n` for `n = 1, 2, ... , 32`. Some examples would be `uint8`, `int32`, `int128`, and `uint240`. Additionally, `int` and `uint` are shorthand for `int256` and `uint256`, respectively.
   
Signed integers does not require a lot of extra assembly work, because there are VM instructions for doing signed division, sign extension, and so on.

When dealing with numbers that are less then 256 bytes in size, there will be some conversions taking place. Below is a contract that has a single `uint` field. It was used in the previous tutorial to show how storage works, and will be used as an example here. Paste it into the browser compiler, and make sure `Enable Optimization` is un-checked.

```
contract C {
    uint x = 1;
}
```
 
As I point out in the previous tutorial, this will be stored at storage address `0x00`, and after cleaning up the assembly a bit, all it really does is `PUSH1 0x01 PUSH1 0x00 SSTORE`.
 
Now replace it with this:

```
contract C {
    uint8 x = 1;
}
```

You will notice that the bytecode increases in size. This is the relevant part of the assembly:

```
PUSH 1			1
PUSH 0			uint8 x = 1
PUSH 0			uint8 x = 1
PUSH 100		uint8 x = 1
EXP 			uint8 x = 1
DUP2 			uint8 x = 1
SLOAD 			uint8 x = 1
DUP2 			uint8 x = 1
PUSH FF			uint8 x = 1
MUL 			uint8 x = 1
NOT 			uint8 x = 1
AND 			uint8 x = 1
SWAP1 			uint8 x = 1
DUP4 			uint8 x = 1
MUL 			uint8 x = 1
OR 			    uint8 x = 1
SWAP1 			uint8 x = 1
SSTORE 			uint8 x = 1
POP 			uint8 x = 1
```

As we can see, these are a lot more instructions then for plain `uint`, since it has to make sure it only uses the first out of the 32 bytes. Even when turning optimization on, it will still have a lot more instructions. 

Before studying, we need to know what this code is doing. Its supposed to write only the first byte of the number 1 (which is 1), and put it in storage. Also, storage is automatically packed when values that occupy less then 32 bytes are used. Notice the `SLOAD` in there? It's needed because in general there could be other data stored in this slot, and if we write this value alone it will overwrite everything but the first byte (1) with zeroes. Instead we load the old value, use bit operations to manipulate only the first byte, then overwrite it with the new, modified value.

This is commonly used, but it needs to be able to handle any bitsize (that is a multiple of 8). Pseudo code for that procedure could look something like this:

```
function store(uint256 n, sizeInBits) {
    uint256 v = storage[0];
    uint256 mask = 1 << sizeInBits - 1; (0xFF in the case of uint8).
    storage[0] = (!mask & v) | (n & mask);
}
```

This ensures that only the first `sizeInBits` bits are read from `n`, and that those bits replace the ones currently stored in the first `sizeInBits` bits of `v`. That's why we see some bit operations in the assembly. The formula needs to be able to handle an offset though, so it would have to look like this.
  
```
function store(uint256 n, sizeInBits, offset) {
    uint256 v = storage[0];
    uint256 mask = (1 << sizeInBits - 1) << offset;
    storage[0] = (!mask & v) | (n & mask);
}
```

This can't be done in the EVM, however, since it does not have bit-shift instructions, so (integer) division and multiplication is used instead. 

With this in mind, let's start analyzing the assembly code.

First it will push four items onto the stack. The first is the value `x = 1`, the second is the assigned storage address (0), the third is the mask offset (0, since it's the first 8 bits), and the last is the multiplier used to shift the mask. Stack becomes: `[0x01, 0x00, 0x00, 0x0100]`.

Next it runs `EXP (0x0100 0x0)` turning the stack into: `[0x01, 0x00, 0x01]`. 

After that it continues by running `DUP2 (0x00)` and `SLOAD (0x00)`. `SLOAD` will push `0x00`, because this is new contract creation, and nothing has been written to storage previously in the code. Stack becomes: `[0x01, 0x00, 0x01, 0x00]`

It then does another `DUP2 (0x01)`, `PUSH1 0xFF` and `MUL (0xFF 0x01)`, making the stack: `[0x01, 0x00, 0x01, 0x00, 0xFF]`. `0xFF` is the mask, and the `MUL` is where it's offset but again - there's no offset here.
  
Next it runs `NOT (0xFF)`, which will inverse it, then `AND (0xFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFF00 0x00)`, which is `0x00`, so the stack becomes: `[0x01, 0x00, 0x01, 0x00]`. The value produced here is the masked old storage value.  

After that it does a `SWAP1 (0x00 0x01)`, `DUP4 (0x01)`, bringing the  value to the top of the stack, and then `MUL (0x01 0x01)`. Stack: `[0x01, 0x00, 0x00, 0x01]`
 
Next it does `OR (0x01 0x00)` (which pops the two top-most items and replace them with `0x01`), `SWAP1 (0x01 0x00)`, and finally `SSTORE (0x00 0x01)`. This stores `0x000......01` at storage address `0x00`, as expected. After that it pops the last remaining item (the original value item) and we're done. The end result is `storage[0] = 1`.

Booleans work the same way as `uint8` in this respect.

### Addresses and bytes

An address is 20 bytes. Let's change the contract to this.

```
CALLER 			msg.sender
PUSH 0			address addr = msg.sender
PUSH 0			address addr = msg.sender
PUSH 100		address addr = msg.sender
EXP 			address addr = msg.sender
DUP2 			address addr = msg.sender
SLOAD 			address addr = msg.sender
DUP2 			address addr = msg.sender
PUSH FFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFF			address addr = msg.sender
MUL 			address addr = msg.sender
NOT 			address addr = msg.sender
AND 			address addr = msg.sender
SWAP1 			address addr = msg.sender
DUP4 			address addr = msg.sender
MUL 			address addr = msg.sender
OR 			    address addr = msg.sender
SWAP1 			address addr = msg.sender
SSTORE 			address addr = msg.sender
POP 			address addr = msg.sender
```

This does not need a lot of explaining. It's the same as with the numbers. `CALLER` pushes the address of the caller onto the stack, and is the instruction behind `msg.sender`. After that we have the same thing, storage address, byte offset, etc. Later we can see the (20 byte) mask. Same principles at work.

### bytesN

`bytesN` works in much the same way, but are expressed in bytes and not bits (like the integers), so you have `bytesN`, where `N = 1, 2, ... , 32`. Let's check it for `bytes4`:

```
PUSH 12345678			0x12345678
PUSH 100000000000000000000000000000000000000000000000000000000			bytes4 bts = 0x12345678
MUL 			bytes4 bts = 0x12345678
PUSH 0			bytes4 bts = 0x12345678
PUSH 0			bytes4 bts = 0x12345678
PUSH 100			bytes4 bts = 0x12345678
EXP 			bytes4 bts = 0x12345678
DUP2 			bytes4 bts = 0x12345678
SLOAD 			bytes4 bts = 0x12345678
DUP2 			bytes4 bts = 0x12345678
PUSH FFFFFFFF			bytes4 bts = 0x12345678
MUL 			bytes4 bts = 0x12345678
NOT 			bytes4 bts = 0x12345678
AND 			bytes4 bts = 0x12345678
SWAP1 			bytes4 bts = 0x12345678
DUP4 			bytes4 bts = 0x12345678
PUSH 100000000000000000000000000000000000000000000000000000000			bytes4 bts = 0x12345678
SWAP1 			bytes4 bts = 0x12345678
DIV 			bytes4 bts = 0x12345678
MUL 			bytes4 bts = 0x12345678
OR 			bytes4 bts = 0x12345678
SWAP1 			bytes4 bts = 0x12345678
SSTORE 			bytes4 bts = 0x12345678
POP 			bytes4 bts = 0x12345678
```

Here we can see a difference, but that is because the bytes are aligned differently. Other then that, the same principles apply. 

### Coming up

More advanced types and instructions.