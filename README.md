# Yul (and Some Solidity) Optimizations and Tricks

Not an exhaustive list, but I tried to make one. Took some use of Twitter's API and lots of searching through DM's of tips I sent people. @controlcthenv on Twitter with any additions you have!

## Good General Resources

[Good Talk on Solidity Internals](https://www.youtube.com/watch?v=RxL_1AfV7N4)

[My Yul+ Toolchain](https://github.com/ControlCplusControlV/Yul-Log) (Working on Foundry support atm instead of Hardhat)

[Becoming a Daptools pilled chad in 30 minutes or less](https://www.youtube.com/watch?v=N9pJ9JieX10) - Then just download Foundry instead

[How Solidity Handles Data representation](https://ethdebug.github.io/solidity-data-representation/)

[Edgar Arout's Twitter Thread here](https://twitter.com/EdgarArout/status/1450825746679615492)

[Introduction To Yul+](https://www.youtube.com/watch?v=GTwqEYyYy6I)

[Yul Docs](https://docs.soliditylang.org/en/v0.8.9/yul.html)

[Every EVM Opcode and cost](https://www.evm.codes/)

[Yul+ Docs](https://github.com/FuelLabs/yulp)

[Vunerable Yul Patterns](https://github.com/Mikerah/solidity-bugs-and-vulns-in-yul)

[This Stackoverflow threading on calling contracts from Yul via call()](https://ethereum.stackexchange.com/questions/6354/how-do-i-construct-a-call-to-another-contract-using-inline-assembly)

## Tip - Utilize Access Lists for the Good of the Chain and Save gas
[Credit - @libevm](https://twitter.com/libevm/status/1523141360076812288)

Call `eth_createAccessList` on a node (probably Geth) and include your transaction blob, and include that access list when sending your transaction to save gas, especially helpful the more storage slots your write to.

*Important too for Ethereum's state management, and will eventually be used to do cool things

## Tip - Don't initialize Zero Values

[Credit](https://twitter.com/fiveoutofnine/status/1500502407712751618)

When writing a for loop instead of writing `uint256 index = 0;` instead write `uint256 index;` as being a uin256 it will be 0 by default so you can save some gas by avoiding initialization.

## Tip - Before Using Yul, Verify YOUR assembly is better than the compiler's

[Case and Point](https://twitter.com/fubuloubu/status/1453002622642819093) Just a reminder if you are a Yul Noob it may be worth testing against Solidity implementations to see if you are saving gas

## Tip - Overwrite new values onto old ones you're not using when Possible

Solidity doesn't garbage collect, and this is just cheaper in Yul, but write new values onto old unused ones to conserve memory and storage used saving gas!

## Tip - Keep Data in Calldata where possible

Since Calldata has already been paid for with the transaction, if you don't modify a parameter to a function, then don't bother copying the function to memory and just read the value from calldata.

## Tip - View Solidity Compiler Yul Output

If you want to see what your Solidity is doing under the hood, just add -yul and -ir to your solc compiler options. Very helpful to see how your code is working, see if you order operations unsafely, or just see how Solidity is beating your Yul in gas usage.

[Solc Compiler Options](https://docs.soliditylang.org/en/latest/using-the-compiler.html?highlight=yul)

## Tip - Using Vanity Addresses with lots of leading zeroes

Why? Well if you have 2 addresses - 0x000000a4323... and 0x0000000000f38210 because of the leading zeroes you can pack them both into the same storage slot, then just prepend the necessary amount of zeroes when using them. This saves you storage when doing things such as checking the owner of a contract.


## Tip - Using Sub 32 Byte values doesn't always save gas

Sub 32 byte values can save gas in the event of packing, but note that they require extra gas to decode and should be used on a case-by-case basis.

## Tip - Writing to an existing Storage Slot is cheaper than using a new one

Credit - @libevm

EIP - 2200 changed a lot with gas, and now if you hold 1 Wei of a token it's cheaper to use the token than if you hold 0. There is a lot to unpack here so just google EIP 2200 and learn if you want, but in general, if you need to use a storage slot, don't empty it if you plan to re-fill it later. Goes for all Yul+ and Yul contracts when managing memory.

## Tip - Negative values are more expensive in calldata

Credit - @[transmissions11](https://twitter.com/transmissions11/status/1482186615220887553)

Negative values have leading bytes of 0xfff while regular integers have zeri leading bytes, and in calldata non-zero bytes cost more than zero bytes, so negative ints end up consuming more gas in calldata.

## Tip - Using iszero() in a lot of places because the compiler is smart

Credit - @transmissions11

He explains it very well [here](https://twitter.com/transmissions11/status/1474465495243898885) but because the compiler knows how to optimize, putting it before some pieces of logic can end up reducing overall gas costs, so test out inserting it before JUMP opcodes.

## Tip - Use Gas() when using call() in Yul

Credit - @libevm

When using call() in Yul you can avoid manually counting out all the gas you need to perform the call, and just forward all available gas via using gas() for the gas parameter.

## Tip - A lot of Solmate is written in inline Yul, so if you're writing in Yul, you can just copy a lot of the assembly

Credit - @transmissions11 

[Solmate](https://github.com/Rari-Capital/solmate/) is written as very efficient Solidity, and because of this is mostly Yul. So if you don't know how to find a Sqrt in Yul for example, just go to Solmate and copy from within the assembly {} blocks for a working implementation, then add GPL-V3 to your SPDX license identifier!

## Tip - Store Storage in Code

Credit - @boredGenius

So [Zefram's blog](https://zefram.xyz/posts/how-i-almost-cheesed-the-evm/) explains this well, but you can save gas by deploying what you want to store in a new contract, and deploying that contract, then reading from that address. This adds a lot of complexity to code but if you need to cut costs and use SLOAD a lot, I recommend looking into SLOAD2.

## Tip - Half of the Zero Address Checks in the NFT spec aren't necessary

Credit - @transmissions11

Launching a new NFT collection and looking to cut minting and usage costs? Use Solmate's NFT contracts, as the standard needlessly prevents transfers to the void, unless someone can call a contract from the 0 address, and the 0 address has unique permissions, you don't need to check that the caller isn't the 0 address 90% of the time.

## Tip - If it can't overflow without uint256(-1) calls, you don't need to check for overflow

Save gas and avoid safemath with unchecked {} , this one is Solidity only but I wanted to include it, I was tired of seeing counters using Safemath, it is cost-prohibitive enough to call a contract billions of times to prevent an attack.

## Tip - If you are testing in Production, use Self-Destruct and Factory patterns for an Upgradeable contract

Credit - @libevm

Using a technique explained in this [Twitter thread](https://twitter.com/libevm/status/1468390867996086275?s=21) you can make it easily upgradeable and testable contracts with re-init and self-destruct. This mostly applied to MEV but if you are doing some cool factory-based programming it's worth trying out.

## Tip - Fallback Function Calls are cheaper than regular function calls

The Fallback function (and Sig-less functions in Yul) save gas because they don't require a function signature to call, for an example implementation I recommend looking at @libevm's [subway](https://github.com/libevm/subway/blob/master/contracts/src/Sandwich.yulp) which utilize's a sig-less function

## Tip - Pack Structs in Solidity

[Struct packing article here](https://dev.to/javier123454321/solidity-gas-optimizations-pt-3-packing-structs-23f4)

A basic optimization but important to know, structs should be organized so that they sequentially add up to multiples of 256 bits in size. So uint112 uint112 uint256 vs uint112 uint256 uint112

Saves read operations needed to get a value

## Tip - Making Solidity Values Constant Where Possible

They are replaced with literals at compile time and will prevent you from having to a read a value from memory or storage. For writing Yul - replace all known values and constant values with literals to save gas and comment what they are.

## Tip - Solidity Modifiers Increase Code Size, So sometimes make them functions

Credit - The Smart Contract Programmer on Youtube

When using modifiers, the code of the modifiers is inserted at the start of the function at compile time, which can massively ballon code size. So sometimes it makes sense to make a modifier a function call instead, as only the function call will be inserted at the start of the function.

## Tip - Trustless calls from L2 to L2 exist, and can be very useful for L2 based DAO's

Credit - Optimism and Arbitrum teams

The OVM and ArbOS have built-in functions on contract calls from L1 to L2 to verify msg.sender and vice versa. Therefore if you make an L1 contract that can only be called by a trusted party on one L2 before calling another L2, you can create a trustless bridge. Recommend reading about Hop for this, but a cool design choice for DAO building.
