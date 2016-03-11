# Advanced Solidity II - EVM Stack and Assembler

This post is about how to write basic assembly code for the EVM. It shows how to manually work with the stack and do some basic flow-control. There is also some info about the bytecode format, and how to manually turn assembly code into bytecode.

### Stack-machines

The Ethereum VM is stack-based. This means the operands for the instructions are taken from the stack, and it is also where the results are added.

Here is a very simple example of how adding 5 and 10 would work in a stack machine. Assume there is a `PUSH` instruction that adds an integer to the stack, and an `ADD` instruction that pops the topmost two items on the stack, adds them, and puts the sum back on top of the stack. This is how adding would look:

```
0: PUSH 5       Stack: [ 5 ] 

1: PUSH 10      Stack: [ 5, 10 ]

2: ADD          Stack: [ 15 ]
```

The stack now has one item on it: 15.

If we wanted to add 5, 10, and 20, it could be done in a similar way:

```
0: PUSH 5       Stack: [ 5 ] 

1: PUSH 10      Stack: [ 5, 10 ]

2: ADD          Stack: [ 15 ]

3: PUSH 20      Stack: [ 15, 20 ]

4: ADD          Stack: [ 35 ]
```

There are other ways of doing it as well, for example we could push all three numbers on the stack directly and just run `ADD` twice.

```
0: PUSH 5       Stack: [ 5 ] 

1: PUSH 10      Stack: [ 5, 10 ]

2: PUSH 20      Stack: [ 5, 10, 20 ]

3: ADD          Stack: [ 5, 30 ]

4: ADD          Stack: [ 35 ]
```

The data will also have to come from somewhere. The way it could be done is by giving each instruction a byte-code, i.e. a number from 0 to 255. By doing so, we can pass instructions and input values in a simple byte array. We can use these rules:
 
1. `PUSH` is the byte `0x01`

2. `ADD` is the byte `0x02`

3. Numbers are interpreted as regular 32 bit integers meaning they will occupy 4 bytes, e.g. 5 is `0x00000005`.

We also add a few rules, so that the bytecode can be interpreted.

1. The first byte is always an instruction.

2. The byte following an `ADD` is always an instruction.

2. The four bytes following a `PUSH` is to be interpreted as a number.

Now we can write byte-code for the first program. 

```
PUSH    5           PUSH    10          ADD
0x01    0x00000005  0x01    0x0000000A  0x02

Result:

0100000005010000000A02

```

Note that it could still fail because for example an `ADD` instruction could be added before there are two items on the stack, but that is not important now.

Either way, if we run this now it will be possible to push and add numbers; for example, this javascript function can do it (unless numbers get too big; there's no proper handling of integer over/underflow):

```
function run(bytecode) {
    var len = bytecode.length;
    if (!len || len % 2)
        throw new Error("Bad input");
    var hl = len >> 1;
    
    var code = new Array(hl);
    for (var i = 0; i < hl; i++) {
        code[i] = parseInt(bytecode.substr(i * 2, 2), 16);
    }

    var codeSize = code.length;
    var stack = [];
    var pc = -1;

    while (++pc < codeSize) {
        switch (code[pc]) {
            case 1:
                var num = (code[++pc] << 24) | (code[++pc] << 16) | (code[++pc] << 8) | code[++pc];
                stack.push(num);
                console.log("PUSH: %d", num);
                break;
            case 2:
                if (stack.length < 2)
                    throw new Error("Stack underflow");
                var a = stack.pop(), b = stack.pop();
                var sum = a + b;
                stack.push(sum);
                console.log("ADD: %d + %d = %d", a, b, sum);
                break;
            default:
                throw new Error("Bad instruction");
        }
        console.log("Stack:", stack);
    }
    console.log("Done");
}

run("0100000005010000000A02");
```

### The EVM Stack

The EVM is much more complex then this simple example, obviously, but addition would actually be done in much the same way. One big difference is that the EVM has a word-size of 32 bytes instead of the 4 bytes I used in the example. It also has several `PUSH` instructions - one for each possible number of bytes (`PUSH1` to `PUSH32`). This is how it would look:

```
PUSH1 0x01 PUSH1 0x03 ADD STOP
```

This should add 1 and 3.

`PUSH1` is `0x60` and `ADD` is `0x01`, and we will add a `STOP` at the end `0x00`, so the bytecode array becomes:

`600160030100`

Running this in the JavaScript VM (using my own tools) produces this:

```
/usr/local/bin/node runcode.js
PC: 0
Opcode: PUSH1
Stack: []
PC: 2
Opcode: PUSH1
Stack: [ '01' ]
PC: 4
Opcode: ADD
Stack: [ '01', '03' ]
PC: 5
Opcode: STOP
Stack: [ '04' ]
returned: 

Process finished with exit code 0
```

Most other arithmetic operations (and similar) works in pretty much the same way.

#### Manual stack manipulation

It is possible to manipulate the items on the stack using `POP`, `SWAP` and `DUP`.

##### POP

Pop will remove the top item. I will now add it to the code from the previous example, right before the `STOP`.

```
PUSH1 0x01 PUSH1 0x03 ADD POP STOP
```

`POP` is `0x50`, so the bytecode is: `60016003015000`

Output:

```
/usr/local/bin/node runcode.js
PC: 0
Opcode: PUSH1
Stack: []
PC: 2
Opcode: PUSH1
Stack: [ '01' ]
PC: 4
Opcode: ADD
Stack: [ '01', '03' ]
PC: 5
Opcode: POP
Stack: [ '04' ]
PC: 6
Opcode: STOP
Stack: []
returned: 

Process finished with exit code 0
```

##### SWAP

`SWAP` has several versions, depending on which items you want to swap. `SWAP1` will swap the first with the second, `SWAP2` the first with the third, and so on until `SWAP16`.

I will run the original example except change `ADD` to `SWAP1`. This means changing `0x01` to `0x90`.

```
PUSH1 0x01 PUSH1 0x03 SWAP1 STOP
```

Bytecode: `600160039000`

Output:

```
/usr/local/bin/node runcode.js
PC: 0
Opcode: PUSH1
Stack: []
PC: 2
Opcode: PUSH1
Stack: [ '01' ]
PC: 4
Opcode: SWAP1
Stack: [ '01', '03' ]
PC: 5
Opcode: STOP
Stack: [ '03', '01' ]
returned: 

Process finished with exit code 0
```

##### DUP

`DUP` will copy a stack item and put the copy on top of the stack. It has several versions as well, and the principle is the same. `DUP1` will duplicate the first item, `DUP2` the third, and so on until `DUP16`.

I will run the original example except change `ADD` to `DUP2`, copying the second stack item. This means changing `0x01` to `0x81`.

```
PUSH1 0x01 PUSH1 0x03 DUP2 STOP
```

Bytecode:  `600160038100`

Output:

```
/usr/local/bin/node runcode.js
PC: 0
Opcode: PUSH1
Stack: []
PC: 2
Opcode: PUSH1
Stack: [ '01' ]
PC: 4
Opcode: DUP2
Stack: [ '01', '03' ]
PC: 5
Opcode: STOP
Stack: [ '01', '03', '01' ]
returned: 

Process finished with exit code 0
```

### Flow control

Flow control is done through jumps and conditional statements. A plain`JUMP` will set the program counter to the value that's currently on top of the stack. `JUMPI` is a conditional jump. It will set the program counter to the value on top of the stack, if the next stack item is not zero. It is often used in combination with conditionals like `EQ` and `LT`.

It is not possible to set the PC to any value, the code at that point has to be a `JUMPDEST`. I will now add a number of items to the stack, but do a jump that will skip some.

```
PUSH1 0x01 PUSH1 0x02 PUSH1 0x0B JUMP PUSH1 0x03 PUSH1 0x04 JUMPDEST STOP
```

This should add `0x01` and `0x02` to the stack, jump past the next two `PUSH1`s to index 11, then stop.

Bytecode: `60016002600B56600360045B00`

```
/usr/local/bin/node runcode.js
PC: 0
Opcode: PUSH1
Stack: []
PC: 2
Opcode: PUSH1
Stack: [ '01' ]
PC: 4
Opcode: PUSH1
Stack: [ '01', '02' ]
PC: 6
Opcode: JUMP
Stack: [ '01', '02', '0b' ]
PC: 11
Opcode: JUMPDEST
Stack: [ '01', '02' ]
PC: 12
Opcode: STOP
Stack: [ '01', '02' ]
returned: 

Process finished with exit code 0
```

`JUMPI` can be done in the same way, but without input it gets complicated, so a true conditional jump will instead be done in a later post about calldata. Now I will simply make two examples - one where I push 0 as the "condition", and one where I pass 1.
 
Successful conditional jump:

```
PUSH1 0x01 PUSH1 0x01 PUSH1 0x0B JUMPI PUSH1 0x03 PUSH1 0x04 JUMPDEST STOP
```

Bytecode: `60016001600B57600360045B00`

```
/usr/local/bin/node runcode.js
PC: 0
Opcode: PUSH1
Stack: []
PC: 2
Opcode: PUSH1
Stack: [ '01' ]
PC: 4
Opcode: PUSH1
Stack: [ '01', '01' ]
PC: 6
Opcode: JUMPI
Stack: [ '01', '01', '0b' ]
PC: 11
Opcode: JUMPDEST
Stack: [ '01' ]
PC: 12
Opcode: STOP
Stack: [ '01' ]
returned: 

Process finished with exit code 0
```

Failed conditional jump:

```
PUSH1 0x01 PUSH1 0x00 PUSH1 0x0B JUMPI PUSH1 0x03 PUSH1 0x04 JUMPDEST STOP
```

Bytecode: `60016000600B57600360045B00`

```
/usr/local/bin/node runcode.js
PC: 0
Opcode: PUSH1
Stack: []
PC: 2
Opcode: PUSH1
Stack: [ '01' ]
PC: 4
Opcode: PUSH1
Stack: [ '01', '00' ]
PC: 6
Opcode: JUMPI
Stack: [ '01', '00', '0b' ]
PC: 7
Opcode: PUSH1
Stack: [ '01' ]
PC: 9
Opcode: PUSH1
Stack: [ '01', '03' ]
PC: 11
Opcode: JUMPDEST
Stack: [ '01', '03', '04' ]
PC: 12
Opcode: STOP
Stack: [ '01', '03', '04' ]
returned: 

Process finished with exit code 0
```

Notice it will pass `JUMPDEST` in the failed example as well. This is fine, because `JUMPDEST` does not affect normal execution; it simply moves past it to the next instruction.

### Coming posts

Knowing how to work with the stack and program counter is a good start. Arithmetic, bitwise, comparison operations is easy to use, and does not need a lot of explaining. Same thing with operations like `ADDRESS`, `CALLER`, `BLOCKHASH`, etc. Instead, the coming posts will likely be on calldata and memory, since memory makes it possible to return data, and calldata makes it possible to provide input. Storage will also be treated. 

Once the basics are covered I will move on to more advanced things like calling contracts from contracts, creating new contracts, manually calling libraries, copying bytecode, etc.