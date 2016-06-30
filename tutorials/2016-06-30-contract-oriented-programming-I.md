# Contract-oriented programming - part I

This series is about [contract-oriented programming](https://en.wikipedia.org/wiki/Design_by_contract) and Solidity. "Contract" in this context is actually not an Ethereum contract, or a smart contract; it is the name used to describe interfaces for functions or other units of functionality. The basic idea is that pre- and post-conditions are first class citizens and should be formally specified and baked into the interfaces themselves.

There is obviously more to it then this, and we will get into more specifics later, but for now we will start by looking at some basic features of Solidity functions and interfaces, and how contract oriented techniques can be applied.

Despite being fairly simple, Solidity is not an easy language to write code in. There are contracts, libraries, functions, modifiers and events. Functions have lots of modifiers that must be chosen correctly, and it's not always obvious what some of them (such as `external` and `internal`) actually does. Certain modifiers are not yet enforced (for example `constant`), and also - even if you get all the modifiers right there are more issues still, for example that certain Solidity types cannot be used as in- or out-data in certain types of functions.

That being said, it is actually not that bad when you get used to it, and it keeps getting better all the time. The issues I mention here are not caused by bad language design, but because certain features (whether they're language or EVM specific) simply hasn't been added in yet. The language is still young. Still, I feel that some of it must be cleared up before going into the contract part.

### Functions and standard modifiers

Modifiers are added to variables and functions to affect their behavior. All the basic modifiers can be found [here](https://en.wikipedia.org/wiki/Design_by_contract). Most of the common modifiers (like public and private) is available, but some behave a bit differently then one might expect. 

Before describing all the modifiers in full I will have to go over `call` vs. `transaction` mechanics:

A `transaction` is a signed transaction sent by a user to a contract account, or external account, that changes (or at least tries to change) the world state. Transactions are placed in the tx queue and are not considered valid until they are eventually mined into a block. Transactions must always be used when sending Ether, or doing any form of **write** operation.

A `call` is used to **read** data from the chain, or do computations that does not change the world state, so it does not require a valid signature or consensus from the other users in the network. An example of when a `call` should be used is when checking the value of a contract field through its public accessor function.

#### `constant`

The `constant` modifier is not enforced by the Solidity compiler yet, but the purpose is to signal to the compiler and callers that the function does not mutate the world state. `constant` appears in the JSON ABI files description of the function, and is used by `web3.js` - the official javascript API - to decide whether it should invoke the function through a `transaction` or a `call`.

#### `external` and `public`

From a visibility perspective, `external` is essentially the same as `public`. When a contract containing a `public` or an `external` function has been deployed, the function can be called from other contracts, calls, and transaction.

The `external` modifier must not be confused with "external" as it's used in the white paper, i.e. "external accounts", which means accounts that are not contract accounts. `external` functions can be called from other contracts as well as through transactions or calls.

The main difference between `external` and `public` functions is the way they are called from the contract that contains them, and how the input parameters are handled. If you call a `public` function from another function in the same contract, the code will be executed using a `JUMP`, much like private and internal functions, whereas `external` functions must be invoked using the `CALL` instruction. Additionally, `external` functions does not copy the input data from the read-only calldata array into memory and/or stack variables, which can be used for optimization.

Finally, `public` is the default visibility, meaning functions will be public if nothing else is specified. There is a similar system for variables as well, but more about that can be found in the documentation.

#### `internal`

`internal` is essentially the same as `protected`. The function can't be called from other contracts, or by transacting to or calling the contract, but it can be called from other functions in the same contract and any contracts that extend it.

#### `private`

`private` functions can only be called from functions in the same contract.

#### Examples

This is a simple example contract with public and private functions. It can be copied and pasted into the online compiler.

```
contract HelloVisibility {
    function hello() constant returns (string) {
        return "hello";
    }

    function helloLazy() constant returns (string) {
        return hello();
    }

    function helloAgain() constant returns (string) {
        return helloQuiet();
    }

    function helloQuiet() constant private returns (string) {
        return "hello";
    }

}
```

The `hello` function can be called from other contracts, and can also be called from the contract itself, as demonstrated by the `helloLazy` function, which simply calls to the `hello` function. The `helloQuiet` function can be called from other functions, as demonstrated by the `helloAgain` function, but it cannot be called from other contracts or through external transactions/calls.

Notice that all functions are marked as `constant`, because none of them will change the world state.

Try adding `external` to the `hello` function, and you will see that the contract now fails to compile. If `hello` has to be external for some reason, we could try and fix the code by changing the call inside `helloLazy` to `this.hello()`, although doing that will not work either! This is because of another issue I mentioned, which is that certain types (dynamically sized arrays in particular) can't be used as input or output in certain functions.

The next two contracts demonstrates the difference between private and internal.

```
contract HelloGenerator {

    function helloQuiet() constant internal returns (string) {
        return "hello";
    }

}


contract Hello is HelloGenerator {
    function hello() constant external returns (string) {
        return helloQuiet();
    }
}
```

`Hello` extends `HelloGenerator` and uses its internal function to create the string. `HelloGenerator` has an empty JSON ABI, because it has no public functions. Try and change `internal` to `private` at `helloQuiet` and you will get a compiler error.

### Custom modifiers

The documentation on custom modifiers can be found [here](http://solidity.readthedocs.io/en/latest/contracts.html#function-modifiers). As an example of how modifiers can be utilized, here are three different ways to create the same function:

```
contract GuardedFunctionExample1 {

    uint public data = 1;

    function guardedFunction(uint _data) {
        if(_data == 0)
            throw;
        data = _data;
    }

}


contract GuardedFunctionExample2 {

    uint public data = 1;

    function guardedFunction(uint _data) {
        check(_data);
        data = _data;
    }

    function check(uint _data) private {
        if (_data == 0)
            throw;
    }

}


contract GuardedFunctionExample3 {

    uint public data = 1;

    modifier checked(uint _data) {
        if (_data == 0)
            throw;
        _
    }

    function guardedFunction(uint _data) checked(_data) {
        data = _data;
    }

}
```

NOTE: The modifier does not actually appear anywhere in the JSON ABI.

### Condition-oriented Programming

Looking at the previous section, one might ask "why complicate things using custom modifiers"? If I want to re-use the guard in different functions, normal decomposition would suffice (example 2). If I wanted to inline it, for added efficiency, I could just do that as well (example 1). This is true, but the modifier semantics is much better suited for something called condition-oriented programming. The basic idea behind COP is outlined in a [blog post](https://medium.com/@gavofyork/condition-orientated-programming-969f6ba0161a#.8dw7jp1gq) by Dr. Gavin Wood. The blog post highlights a number of ugly side effects that mixing the (pre)conditions in a function with the "business logic" itself may have, and shows how COP can be used to avoid that.

*Potential bugs hide when the programmer believes a conditional (and thus the state it projects onto) means one thing when in fact it means something subtly different.*

A good example of this is the code that caused "the DAO" to fail. It has been concluded that certain functions in the DAO contract was susceptible to so called reentrancy attacks, but this was not because of bugs in the EVM, or the Solidity language itself; it was because of a programming error that is very easy to make. More info about this weakness can be found in [this blog post](https://eng.erisindustries.com/programming/2016/06/18/lessons-learned-dao/) by core Solidity developer RJ Catalano.

Gavin goes on to explain that *Essentially, COP uses pre-conditions as a first-class citizen in programming* - which is essentially what the custom modifiers in Solidity are - then proceeds to give a few simple examples of how they can be used to write COP Solidity code.

#### COP applied

This section contains a simple application of the techniques described in the blog post. The following contract is a variation of the example token contract We will start without modifiers.

```
contract Token
{
    address public owner;

    // The balance of everyone
    mapping (address => uint) public balances;

    mapping (address => bool) public blacklisted;


    // Constructor - we're a millionaire!
    function Token() {
        owner = msg.sender;
        balances[msg.sender] = 1000000;
    }

    function blacklist(address _addr) {
        if(msg.sender != owner)
            return;
        blacklisted[_addr] = true;
    }

    function transfer(uint _amount, address _dest) {
        if(blacklisted[msg.sender])
            return;
        if(balances[msg.sender] >= _amount) {
            balances[msg.sender] -= _amount;
            balances[_dest] += _amount;
        }
    }

}
```

The first thing we will do is to break out the guard from the `blacklist` function into a modifier and add that modifier to the function, giving us this:

```
modifier isOwner {
    if (msg.sender != owner)
        return;
    _
}

function blacklist(address _addr) isOwner {
    blacklisted[_addr] = true;
}
```

Note that the modifier doesn't need a `()` if it doesn't take any arguments.

Fixing the transfer function is not very difficult. The same procedure as was used for `isOwner` can be used for both guards. The difference is we will add two modifiers to the function instead of one.

```
modifier notBlacklisted {
    if (blacklisted[msg.sender])
        return;
    _
}

modifier atLeast(uint x) {
    if (balances[msg.sender] < x)
        return;
    _
}

function transfer(uint _amount, address _dest) notBlacklisted atLeast(_amount) {
    balances[msg.sender] -= _amount;
    balances[_dest] += _amount;
}
```

Notice the order of the modifiers are left-to-right, so in order to have the exact same order the blacklist check must come first. Also, the balance check was changed slightly, although we could just as well have made it like this:

```
modifier atLeast(uint x) {
    if (balances[msg.sender] >= x)
      _
}
```

This is the resulting contract. Very neat.

```

contract COPToken
{
    address public owner;

    // The balance of everyone
    mapping (address => uint) public balances;

    mapping (address => bool) public blacklisted;


    // Constructor - we're a millionaire!
    function COPToken() {
        owner = msg.sender;
        balances[msg.sender] = 1000000;
    }

    modifier isOwner {
        if (msg.sender != owner)
            return;
        _
    }

    modifier notBlacklisted {
        if (blacklisted[msg.sender])
            return;
        _
    }

    modifier atLeast(uint x) {
        if (balances[msg.sender] < x)
            return;
        _
    }

    function blacklist(address _addr) isOwner {
        blacklisted[_addr] = true;
    }

    function transfer(uint _amount, address _dest) notBlacklisted atLeast(_amount) {
        balances[msg.sender] -= _amount;
        balances[_dest] += _amount;
    }

}
```

#### Unit testing

Now comes the interesting part: How do we make sure that these modifiers actually work? At this point, a few basic unit tests will have to do. We could just call the function with different params and check the results, but that is not particularly clean. Instead, we're going to use inheritance to create a contract with functions that are used to test a modifier, and separate functions for testing the modified function itself. This contract that is tested here is even simpler then the previous one.

```
contract Token
{

    // The balance of everyone
    mapping (address => uint) public balances;


    // Constructor - we're a millionaire!
    function Token() {
        balances[msg.sender] = 1000000;
    }

    modifier atLeast(uint x) {
        if (balances[msg.sender] < x)
            return;
        _
    }

    function transfer(uint _amount, address _dest) atLeast(_amount) {
        balances[msg.sender] -= _amount;
        balances[_dest] += _amount;
    }

}


contract TokenTest is Token {

    address constant EMPTY_ACCOUNT = 0xDEADBEA7;

    function atLeastTester(uint _amount) atLeast(_amount) constant private returns (bool) {
        return true;
    }

    function testAtLeastSuccess() returns (bool) {
        balances[msg.sender] = 1000;
        return atLeastTester(1000);
    }

    function testAtLeastFailBalanceTooLow() returns (bool) {
        balances[msg.sender] = 999;
        return !atLeastTester(1000);
    }

    // Test transferring to account with no money, then check their balance.
    function testTransfer() returns (bool) {
        balances[msg.sender] = 500;
        balances[EMPTY_ACCOUNT] = 0;
        transfer(500, EMPTY_ACCOUNT);
        return balances[msg.sender] == 0 && balances[EMPTY_ACCOUNT] == 500;
    }

    // Test transferring to account with no money, then check their balance.
    function testTransferFailBalanceTooLow() returns (bool) {
        balances[msg.sender] = 500;
        balances[EMPTY_ACCOUNT] = 0;
        transfer(600, EMPTY_ACCOUNT);
        return balances[msg.sender] == 500 && balances[EMPTY_ACCOUNT] == 0;
    }

}
```

The first function in the test contract simply runs the modifier and returns true if it passes. This means the function will return `true` if the balance of the caller is equal to or higher then the provided value, otherwise it returns false. Next, there are two simple functions to check if the modifier works as intended. Finally there are two functions to check that the body of the transfer function works as intended.

This makes testing the transfer function easier. If the modifier tests passes but the transfer function does not, it is clearly something wrong with the transfer function. If the modifier tests fail then we can't expect the transfer function to work properly until the modifier is fixed.


### Complications

The COP modifier approach does not come without issues.

When using normal functions, without modifiers, it is possible to return an error code when something goes wrong. In the example contract, we would perhaps have wanted to return a different error code depending on whether the transfer was successful, if it failed because the caller was blacklisted, or if it failed because the callers balance was too low. This can be very useful when a function is called by other contracts. Modifiers are a bit heavy handed in this respect because the only thing they can really do (at this point) is to throw, which terminates the VM and reverts all changes, or return, which returns the null value of all return params.

Another issue is of course the classical problem that arises when "things go formal" - code becomes much more difficult to write. An example would be a function that is made up of several nested conditionals, and each block contains a lot of logic. Gavin's example uses a vote rejection function that is called after the transfer logic (and its guards) has been run, but things can be a lot more complex; the vote function is called by default, and the custom modifier on that function takes no argument, but what if different functions should be called depending on whether or not the transaction was successful, and those in turn has similar conditions in them. It could get much more difficult to write that using proper COP.

### Conclusion

This is a very interesting approach to smart contract writing in Solidity. Together with formal verification, new unit testing and static analysis tools, new planned features like lambda functions, and by sticking to the safe language features that are already in place, we can hopefully avoid the "the DAO" problems from happening again.

It should also be mentioned that design-by-contract is not some obscure theoretical construct that is only applied in special niche languages; it is a well established programming paradigm, and there are several libraries that makes it possible in mainstream languages like C++ and Java (to some degree). Certain languages (like C++) even has full language support under way.

Finally, I really don't want to put personal opinion in here, but since I'm essentially holding the Slock.it contracts up as an example of what not to do, I'd like to point out that I don't blame their coders for the problems. Nobody can write 100% bug free, perfect code, and that's basically what they were supposed to do. At least they had the foresight to put a safeguard in place that prevent Ether from just being sucked out of the contract.

The next part will contain more advanced contracts and functionality. It will likely come after Gavin has posted part 2 of his tutorial.

Happy (and safe) smart contract writing!
