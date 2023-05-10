---
title: "Stricter Types in TypeScript"
tags: [TypeScript]
author: "Yunier"
date: "2023-02-17"
description: "A quick look at opaque/branded types in TypeScript."
---

Recently TypeScript [wizard](https://www.totaltypescript.com/) [Matt Pocock](https://twitter.com/mattpocockuk) made a [Twitter thread](https://twitter.com/mattpocockuk/status/1625173884885401600) on branded types. At first, I did not know what he was talking about, I thought it was a new TypeScript feature being introduced in [TypeScript 5](https://devblogs.microsoft.com/typescript/announcing-typescript-5-0-beta/) but upon closer look, I realized that it was not a new feature but rather a technique that I already knew, opaque types. 

I first learned about opaque types from [Evert Pot](https://evertpot.com/opaque-ts-types/) in his blog post [Implementing an opaque type in typescript](https://evertpot.com/opaque-ts-types/), though I guess now the TypeScript community prefers to call them branded types, the name doesn't matter, the problem being solved is the same, preventing types from being interchangeable.

For example, take a look at the following code.

```TypeScript
type DepositAmount = number;
type WithdrawAmout = number;

function deposit(amount: DepositAmount)
{
    console.log(`Adding ${amount} money...`)   
}

const depositAmount: DepositAmount = 100

deposit(depositAmount);
```

In the code above a DepositAmount and WithdrawAmout type are declared, assume that the deposit function is part of some Web API run by a bank. In this scenario, we create a variable depositAmount, assign it a value of 100 and pass it as a parameter to the deposit function. Pretty standard stuff, nothing new here. The code does have one flaw, it is possible for a developer to accidentally do the following.

```TypeScript
const depositAmount: WithdrawAmout = 100

deposit(depositAmount);
```

The variable depositAmount is now of type WithdrawAmout while the deposit function still expects a type of DepositAmount. Changing the variable depositAmount to be of type WithdrawAmout has no impact on the program, the result is the same, try it yourself. The reason why this change has no impact on the way the API runs is due to how TypeScript infers type, see to TypeScript the type DepositAmount and WithdrawAmout are the same. While the code continues to work as before, we as humans know that the functionality is wrong, a type WithdrawAmout being passed to a function that expects a type of DepositAmount doesn't make a whole lot of sense.

This is where Branded/Opaque types come in handy, a branded type is not interchangeable. Here is how it would work in the example above. First, declare a brand as a unique symbol type.

```TypeScript
declare const brand: unique symbol; 
```

A symbol is a type that can never be created because it is a primitive type, just like numbers and strings. See [Symbols](https://www.typescriptlang.org/docs/handbook/symbols.html)

Now I'll declare my branded type and I'll update DepositAmount and WithdrawAmout.

```TypeScript
type Brand<T, TBran extends string> = T & {[brand] : TBran};

type DepositAmount = Brand<number, "DepositAmount">;
type WithdrawAmout = Brand<number, "WithdrawAmout">;
```

The original DepositAmount & WithdrawAmout are now branded types. The rest of the code is updated as shown below.

```TypeScript
function deposit(amount: DepositAmount)
{
    console.log(`Adding ${amount} money...`)   
}

const depositAmount: DepositAmount = 100 as DepositAmount;

deposit(depositAmount);
```

Now if someone attempts to use the deposit function but with a variable of type WithdrawAmout, TypeScript will produce the following error.

```TypeScript 
// Argument of type 'WithdrawAmout' is not assignable to parameter of type 'DepositAmount'.
```

Thus ensuring that only a variable of type DepositAmount is ever passed to the deposit function.

If you prefer to not have to cast then use a type predicate function as shown below.

```TypeScript
const isValidDepositAmount = (amount: number) : amount is DepositAmount => { return amount > 0 }
const isValidWithdrawAmount = (amount: number) : amount is WithdrawAmount => { return amount < 0}
```

Then the code can be updated.

```TypeScript
const depositAmount = 100;

if (isValidDepositAmount(depositAmount)){
    deposit(depositAmount); // variable depositAmount is valid.
}
```

And if someone were to assign an invalid value like -100 to depositAmount then the deposit method will never be invoked. 

```TypeScript
const depositAmount = -100;

if (isValidDepositAmount(depositAmount)){
    deposit(depositAmount); // will never be executed because -100 cannot be cast to DepositAmount in isValidDepositAmount.
}
```

If you are interested in learning more, I recommend becoming a [TypeScript Wizard](https://www.totaltypescript.com/).