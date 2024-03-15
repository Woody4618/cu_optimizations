# Optimizing CU in programs 

## Introduction

Every block on Solana currently has a blockspace limit of 48 million CU and a 12 million CU per write lock. If you exhaust the CU limit your transaction will fail. Optimizing your program CUs has many advantages.

Currently every transactions on Solana costs 5000 lamports independant on the compute units used. 
Four reasons on why to optimize CU anyway:

1. A smaller transaction is more likely to be included in a block.
2. Currently every transaction costs 5000 lamports/sig no matter the CU. This may change in the future. Better be prepared.
3. It makes your program more composable, because when another program does a CPI in your program it also need to cover your CU.
4. More block space for every one. One block could theoretically hold around tens of thousands of simple transfer transactions for example. So the less intensive each transaction is the more we can get in a block. 

By default every transaction on solana requests 200.000 CUs. 
With the call setComputeLimit this can be increased to a maximum of 1.4 million.

If you are only doing a simple transfer, you can set the CU limit to 300 for example.

```js
  const computeLimitIx = ComputeBudgetProgram.setComputeUnitLimit({
    units: 300,
  });

```

There can also be priority fees which can be set like this: 

```js
  const computePriceIx = ComputeBudgetProgram.setComputeUnitPrice({
    microLamports: 1,
  });
```

This means for every requested CU, 1 microLamport is paid. This would result in a fee of 0.2 lamports.
These instructions can be put into a transaction at any position.

```js
const transaction = new Transaction().add(computePriceIx, computeLimitIx, ...);
```

Find Compute Budget code here: 
https://github.com/solana-labs/solana/blob/090e11210aa7222d8295610a6ccac4acda711bb9/program-runtime/src/compute_budget.rs#L26-L87


Blocks are packed using the real CU used and not the requested CU.

There may be a CU cost to loaded account size as well soon with a maximum of 16.000 CU which would be charges heap space at rate of 8cu per 32K. (Max loaded accounts per transaction is 64Mb)
https://github.com/solana-labs/solana/issues/29582 

## How to measure CU

The best way to measure CU is to use the solana program log. 

For that you can use this macro: 

```rust
/// Total extra compute units used per compute_fn! call 409 CU
/// https://github.com/anza-xyz/agave/blob/d88050cda335f87e872eddbdf8506bc063f039d3/programs/bpf_loader/src/syscalls/logging.rs#L70
/// https://github.com/anza-xyz/agave/blob/d88050cda335f87e872eddbdf8506bc063f039d3/program-runtime/src/compute_budget.rs#L150
#[macro_export]
macro_rules! compute_fn {
    ($msg:expr=> $($tt:tt)*) => {
        ::solana_program::msg!(concat!($msg, " {"));
        ::solana_program::log::sol_log_compute_units();
        let res = { $($tt)* };
        ::solana_program::log::sol_log_compute_units();
        ::solana_program::msg!(concat!(" } // ", $msg));
        res
    };
}
```

You put it in front and after the code block you want to measure like so: 

```rust

compute_fn!("My message" => {
    // Your code here
});

```

Then you can paste the logs into chatGPT and let i calculate the CU for you if you dont want to do it yourself.
Here is the original from @thlorenz https://github.com/thlorenz/sol-contracts/blob/master/packages/sol-common/rust/src/lib.rs

# Optimizations 

## 1 Logging 
Logging is very expensive. Especially logging pubkeys and concatenating strings. Use pubkey.log instead if you need it and only log what is really necessary. 

```rust
// 11962 CU !!
// Base58 encoding is expensive, concatenation is expensive
compute_fn! { "Log a pubkey to account info" =>
    msg!("A string {0}", ctx.accounts.counter.to_account_info().key());
}

// 262 cu
compute_fn! { "Log a pubkey" =>
    ctx.accounts.counter.to_account_info().key().log();
}

let pubkey = ctx.accounts.counter.to_account_info().key();

// 206 CU
compute_fn! { "Pubkey.log" =>
    pubkey.log();
}

// 357 CU - string concatenation is expensive
compute_fn! { "Log a pubkey simple concat" =>
    msg!("A string {0}", "5w6z5PWvtkCd4PaAV7avxE6Fy5brhZsFdbRLMt8UefRQ");
}
```

## 2 Data Types
Bigger data types cost more CU. Use the smallest data type possible. 

```rust
// 357
compute_fn! { "Push Vector u64 " =>
    let mut a: Vec<u64> = Vec::new();
    a.push(1);
    a.push(1);
    a.push(1);
    a.push(1);
    a.push(1);
    a.push(1);
}

// 211 CU
compute_fn! { "Vector u8 " =>
    let mut a: Vec<u8> = Vec::new();
    a.push(1);
    a.push(1);
    a.push(1);
    a.push(1);
    a.push(1);
    a.push(1);
}

```

## 3 Serialization
Borsh serialization can be expensive depending on the account structs, use zero copy when possible and directly interact with the memory. It also saves more stack space than boxing. Operations on the stack are slightly more efficient though.  

```rust
// 6302 CU
pub fn initialize(_ctx: Context<InitializeCounter>) -> Result<()> {
    Ok(())
}

// 5020 CU
pub fn initialize_zero_copy(_ctx: Context<InitializeCounterZeroCopy>) -> Result<()> {
    Ok(())
}
```

```rust 
// 108 CU - total CU including serialization 2600 
compute_fn! { "Borsh Serialize" =>
    let counter = &mut ctx.accounts.counter;
    counter.count = counter.count.checked_add(1).unwrap();
}

// 151 CU - total CU including serialization 1254 
compute_fn! { "Zero Copy Serialize" =>
    let counter = &mut ctx.accounts.counter_zero_copy.load_mut()?;
    counter.count = counter.count.checked_add(1).unwrap();
}
```

## 4 PDAs
Depending on the seeds find_program_address can use multiple loops and become very expensive. You can save the bump in an account or pass it in from the client to remove this overhead: 

```rust
pub fn pdas(ctx: Context<PdaAccounts>) -> Result<()> {
    let program_id = Pubkey::from_str("5w6z5PWvtkCd4PaAV7avxE6Fy5brhZsFdbRLMt8UefRQ").unwrap();

    // 12,136 CUs
    compute_fn! { "Find PDA" =>
        Pubkey::find_program_address(&[b"counter"], ctx.program_id);
    }

    // 1,651 CUs
    compute_fn! { "Find PDA" =>
        Pubkey::create_program_address(&[b"counter", &[248_u8]], &program_id).unwrap();
    }

    Ok(())
}

#[derive(Accounts)]
pub struct PdaAccounts<'info> {
    #[account(mut)]
    pub counter: Account<'info, CounterData>,
    // 12,136 CUs when not defining the bump
    #[account(
        seeds = [b"counter"],
        bump
    )]
    pub counter_checked: Account<'info, CounterData>,
}

#[derive(Accounts)]
pub struct PdaAccounts<'info> {
    #[account(mut)]
    pub counter: Account<'info, CounterData>,
    // only 1600 if using the bump that is saved in the counter_checked account
    #[account(
        seeds = [b"counter"],
        bump = counter_checked.bump
    )]
    pub counter_checked: Account<'info, CounterData>,
}

````

## 5 Closures and function

During the tests it looks like that closures, function calls and inlining have a similar cost and were well optimized by the compiler. 

## 6 Native vs Anchor

Anchor is a great tool for writing programs, but it comes with a cost. Every check that anchor does costs CU. While most checks are useful, there may be room for improvement. The anchor generated code is not optimized for CU. 
The tests for native programs are not yet done. If you want to help with that, please let me know or open a PR. 

## 7 Analyze and optimize yourself 

Most important here is actually to know that every check and every serialization costs compute and how to profile and optimize it yourself since every program is different. Profile and optimize your programs today! 

Feel free to play around with it and add more examples. Also here is a very nice article from @RareSkills_io https://www.rareskills.io/post/solana-compute-unit-price on that topic. I replicated some of the examples :) 
