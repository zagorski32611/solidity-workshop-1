# Contract-oriented programming - part II

This is the second part of the contract-oriented programming tutorial. Here we're going to look at post-conditions, and a few functions that are more complex then the ones in the previous post.

NOTE: Like in the first post, this is just a few examples of how to apply some simple contract-oriented techniques in Solidity using custom modifiers. We will get into more nuances in later posts. This is mostly for experimentation and exploring.

### Post-conditions

Post conditions can be used to ensure that certain things actually happened as a result of running a given function. Usually these things will be assertions about some state, such as the caller balance, or a contract field. We do not limit the type of states may (or should) be checked here.

A very simple example of a post-condition would be this:

```
contract PostCheck {

    uint public data = 0;

    // Check that the 'data' field was set to the value of '_data'.
    modifier data_is_valid(uint _data) {
        _
        if (_data != data)
            throw;
    }

    function setData(uint _data) data_is_valid(_data) {
        data = _data;
    }

}
```

Notice the position of the `_` inside the post-condition modifier; it will execute the modified function *before* doing the check, as opposed to in a pre-condition modifier where the check is done first.

It is possible to combine pre- and post-conditions. Here is an example:

```
contract PrePostCheck {

    uint public data = 0;

    // Check that the input '_data' value is not the same as the value
    // already stored in 'data'.
    modifier data_is_valid(uint _data) {
        if (_data == data)
           throw;
        _
    }

    // Check that the 'data' field was set to the value of '_data'.
    modifier data_was_updated(uint _data) {
        _
        if (_data != data)
            throw;
    }

    function setData(uint _data) data_is_valid(_data) data_was_updated(_data) {
        data = _data;
    }

}
```

This is a very safe function. Not only does it check that the input data is valid (which in this case means it's not the same as the data that's already stored), but it also checks that the data variable was actually changed before returning. We are not going to unit test this contract, like in part 1, since the tests would essentially be the same.

### Ordering of modifiers

When adding modifiers to a function, it is very important to get the ordering right. Modifiers should be added left-to-right, in the order they are meant to be executed. This means that if both pre- and post-conditions are used, we must put the pre-conditions first. There is no semantic difference between pre- and post-condition modifiers, so the best thing to do is (in my opinion) to use a good naming strategy. One way would be to prepend all modifier names with either `pre` or `post`, e.g. `pre_data_is_zero`. There are no recommendations in the official Solidity style guide.

### `return` vs. `throw`

It can be hard to decide how to escape from a function if pre- or post-conditions are not met. Given how Solidity works at this point, it really boils down to whether you allow `return` in modifiers, or if they have to `throw`.

Personally, I'm not sure what the best solution is. I'm leaning towards always throwing. This approach treats modifiers as traditional assertions, and if an assertion fails, that means we're guaranteed (in theory) that the code execution doesn't have unintended consequences. What I mean by "in theory" is that it is true as long as the modifiers are properly written and added to the functions in the correct way, and that execution is normal (i.e. no weird EVM exploits are used). This seems like the most contract-oriented way of doing things. The downside is that `throw` does not allow any form of recovery in calling functions, because there is no way to `catch`, but even if you could, there is still no error types, but like I point out in part 1 - it is also not possible to return error codes when using modifiers so this doesn't really matter at this point.

### Separating function logic from conditions

Sometimes it can be hard to know whether a conditional should be put inside a modifier (as a pre- or post-condition), or be part of the function body itself. Here is a contract with a function that is a bit trickier then the ones in part 1:

```
contract Token {

    // The balance of everyone
    mapping (address => uint) public balances;

    // Blacklisted accounts
    mapping (address => bool) public blacklisted;


    // Constructor - we're a millionaire!
    function Token() {
        balances[msg.sender] = 1000000;
    }

    // Transfer funds.
    // If the caller is blacklisted, this will fail.
    // If the receiver is not blacklisted and the caller has the funds, 
    // they will be transferred to the receiver, otherwise the caller is blacklisted 
    // and their account is emptied.
    function transfer(uint _amount, address _dest) {
        if (blacklisted[msg.sender])
            return;
        if (!blacklisted[_dest] && balances[msg.sender] >= _amount) {
            balances[msg.sender] -= _amount;
            balances[_dest] += _amount;
        }
        else {
            balances[msg.sender] = 0;
            blacklisted[msg.sender] = true;
        }
    }

}
```

Here we see a number of conditionals being used, but it is different from the contracts we've looked at thus far in that `transfer` can do two completely different things based on the state of the contract - one is to transfer funds from one account to another, and one is to empty the callers account and blacklist them. So, how do we decompose this?

Let's start by looking at the conditionals. We have the following three:

1. `blacklisted[msg.sender] == true`

2. `blacklisted[_dest] == false`

3. `balance[msg.sender] >= _amount`

Out of these, I would argue that `1` is the only pre-condition. It asserts that the caller is not blacklisted, and if they are, the rest of the body will not be run. The other two conditions should not be broken out, because they are part of the function's logic. It doesn't matter if one or both of them fail; the function will still work as intended.

What we have then is one pre-condition and no post-conditions. With modifiers it should look something like this:

```
contract Token {

    // The balance of everyone
    mapping (address => uint) public balances;

    // Blacklisted accounts
    mapping (address => bool) public blacklisted;


    // Constructor - we're a millionaire!
    function Token() {
        balances[msg.sender] = 1000000;
    }

    // msg.sender cannot be blacklisted
    modifier not_blacklisted {
        if (blacklisted[msg.sender])
            throw;
        _
    }

    // Transfer funds.
    // If the caller is blacklisted, this will fail.
    // If the receiver is not blacklisted and the caller has the funds, 
    // they will be transferred to the receiver, otherwise the caller is blacklisted 
    // and their account is emptied.
    function transfer(uint _amount, address _dest) not_blacklisted {
        if (!blacklisted[_dest] && balances[msg.sender] >= _amount) {
            balances[msg.sender] -= _amount;
            balances[_dest] += _amount;
        }
        else {
            balances[msg.sender] = 0;
            blacklisted[msg.sender] = true;
        }
    }

}
```

Now imagine that we change the function so that a low balance does not cause the sender to be blacklisted, but only prevent them from transacting (which honestly seems a bit more sensible). In that case we change the balance check to a pre-condition as well.

(Note that we will also update the documentation)

```
contract Token {

    // The balance of everyone
    mapping (address => uint) public balances;

    // Blacklisted accounts
    mapping (address => bool) public blacklisted;


    // Constructor - we're a millionaire!
    function Token() {
        balances[msg.sender] = 1000000;
    }

    // msg.sender cannot be blacklisted
    modifier not_blacklisted {
        if (blacklisted[msg.sender])
            throw;
        _
    }

    // msg.sender must have a balance of at least 'x' tokens.
    modifier at_least(uint x) {
        if (balances[msg.sender] < x)
            throw;
        _
    }

    // Transfer funds.
    // If the caller is blacklisted or does not have enough funds,
    // the transfer will fail.
    // If the receiver is not blacklisted, the funds are transferred
    // to the receiver, otherwise the caller is blacklisted and their
    // account emptied.
    function transfer(uint _amount, address _dest) not_blacklisted at_least(_amount) {
        if (!blacklisted[_dest]) {
            balances[msg.sender] -= _amount;
            balances[_dest] += _amount;
        }
        else {
            balances[msg.sender] = 0;
            blacklisted[msg.sender] = true;
        }
    }

}
```

This looks a lot neater, but we can do more. We have actually separated the balance adjustment logic from the blacklisting logic, which means we can put that code in a different function, and add the `at_least` modifier to the new function instead. The balance adjustment function would be private, so that only the `transfer` function can access it. Also, to be extra neat, we can break out the blacklisting code as well (the code inside the `else` block).

```
contract Token {

    // The balance of everyone
    mapping (address => uint) public balances;

    // Blacklisted accounts
    mapping (address => bool) public blacklisted;


    // Constructor - we're a millionaire!
    function Token() {
        balances[msg.sender] = 1000000;
    }

    // msg.sender cannot be blacklisted
    modifier not_blacklisted {
        if (blacklisted[msg.sender])
            throw;
        _
    }

    // msg.sender must have a balance of at least 'x' tokens.
    modifier at_least(uint x) {
        if (balances[msg.sender] < x)
            throw;
        _
    }

    // Transfer funds. This fails if the caller does not have enough funds.
    function __transfer(uint _amount, address _dest) private at_least(_amount) {
        balances[msg.sender] -= _amount;
        balances[_dest] += _amount;
    }

    // Blacklist an account.
    function __blacklist() private {
        balances[msg.sender] = 0;
        blacklisted[msg.sender] = true;
    }

    // Transfer funds.
    // If the caller is blacklisted or does not have enough funds,
    // the transfer will fail.
    // If the receiver is not blacklisted, the funds are transferred
    // to the receiver, otherwise the caller is blacklisted and their
    // account emptied.
    function transfer(uint _amount, address _dest) not_blacklisted {
        if(!blacklisted[_dest])
            __transfer(_amount, _dest);
        else
            __blacklist();
    }

}
```

Note that we could just use the ternary operator in the body of `transfer` now, but it makes the code a bit harder to read so we won't.

Also, notice that we have picked the interface apart. The `transfer` function no longer has the balance check; it has been moved to the private `__transfer` function instead. If we look a bit closer, though, we will see that this actually makes sense, because if the target account is blacklisted then the `transfer` function will not even call the balance transfer logic. This is an important detail.

### Contracts and smart-contracts

In these tutorials, there are two different types of contracts at work, and the differences between the two must be made clear.

- The `contract` here - in terms of contract-oriented programming - is the definition of functionality, i.e. what a function does, and the conditions that must to be met in order for the function to carry out its work. It is informal, meaning there is not full language support for it.

- The balance criterion, in this case, is a pre-condition in the (informal) transactional contract definition.

- The `at_least` modifier is used to do the balance check in the code.

- The `transfer` function is where the functions described in the contract is carried out. It is part of a `smart-contract`, which is the name used for code that runs in the EVM (Ethereum Virtual Machine).

- The `transfer` function is the entry point, but it defers some of the work to other functions.

Note that the `contract` (in this context) is also not the JSON ABI of the smart-contract, and nor is it the JSON ABI of the `transfer` function, although the JSON ABI is part of the contract definition since it contains specifications for the input and output data, and some other things. There is no way to formally specify these type of contracts in Solidity. Most of the contract related validation work must be done manually, but it is still good to work like this because (like Gavin points out) it makes the code safer, and easier to validate and test.

Finally, I will not unit test these contracts either. The example in part 1 is enough for now. I will probably add a part that focuses only on that when the basics has been sorted.

### Conclusion

This part showed how post-conditions can be created as custom modifiers, and how more advanced code can be structured. The point of the advanced example was to show that contract-oriented programming can't be applied simply by moving conditionals out into custom modifiers, but requires some thinking.

If someone wants to dig deeper into contract-oriented programming in general, a good document can be found [here](http://se.ethz.ch/~meyer/publications/computer/contract.pdf). It explains a few of the concepts, and how they can be applied.

Next post will probably be about documentation and more about how code can be structured. It depends on when Gavin produces his next blog-post.
