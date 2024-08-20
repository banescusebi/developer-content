---
sidebarLabel: Deploying Programs
title: Deploying Your First Solana Program
sidebarSortOrder: 3
---

In this section, we'll build, deploy, and test a simple Solana program using the
Anchor framework. By the end, you'll have deployed your first program to the
Solana blockchain!

The purpose of this section is to familiarize you with the Solana Playground.
We'll walk through a more detailed example in the PDA and CPI sections. For more
details, refer to the [Programs on Solana](/docs/core/programs) page.

<Steps>

### Create Anchor Project

First, open https://beta.solpg.io in a new browser tab.

- Click the "Create a new project" button on the left-side panel.

- Enter a project name, select Anchor as the framework, then click the "Create"
  button.

![New Project](/assets/docs/intro/quickstart/pg-new-project.gif)

You'll see a new project created with the program code in the `src/lib.rs` file.

```rust filename="lib.rs"
use anchor_lang::prelude::*;

// This is your program's public key and it will update
// automatically when you build the project.
declare_id!("11111111111111111111111111111111");

#[program]
mod hello_anchor {
    use super::*;
    pub fn initialize(ctx: Context<Initialize>, data: u64) -> Result<()> {
        ctx.accounts.new_account.data = data;
        msg!("Changed data to: {}!", data); // Message will show up in the tx logs
        Ok(())
    }
}

#[derive(Accounts)]
pub struct Initialize<'info> {
    // We must specify the space in order to initialize an account.
    // First 8 bytes are default account discriminator,
    // next 8 bytes come from NewAccount.data being type u64.
    // (u64 = 64 bits unsigned integer = 8 bytes)
    #[account(init, payer = signer, space = 8 + 8)]
    pub new_account: Account<'info, NewAccount>,
    #[account(mut)]
    pub signer: Signer<'info>,
    pub system_program: Program<'info, System>,
}

#[account]
pub struct NewAccount {
    data: u64
}
```

<Accordion>
<AccordionItem title="Explanation">

For now, we'll only cover the high-level overview of the program code:

- The `declare_id!` macro specifies the on-chain address of your program. It
  will be automatically updated when we build the program in the next step.

  ```rs
  declare_id!("11111111111111111111111111111111");
  ```

- The `#[program]` macro annotates a module containing functions that represent
  the program's instructions.

  ```rs
  #[program]
  mod hello_anchor {
      use super::*;
      pub fn initialize(ctx: Context<Initialize>, data: u64) -> Result<()> {
          ctx.accounts.new_account.data = data;
          msg!("Changed data to: {}!", data); // Message will show up in the tx logs
          Ok(())
      }
  }
  ```

  In this example, the `initialize` instruction takes two parameters:

  1. `ctx: Context<Initialize>` - Provides access to the accounts required for
     this instruction, as specified in the `Initialize` struct.
  2. `data: u64` - An instruction parameter that will be passed in when the
     instruction is invoked.

  The function body sets the `data` field of `new_account` to the provided
  `data` argument and then prints a message to the program logs.

- The `#[derive(Accounts)]` macro is used to define a struct that specifies the
  accounts required for a particular instruction, where each field represents a
  separate account.

  The field types (ex. `Signer<'info>`) and constraints (ex. `#[account(mut)]`)
  are used by Anchor to automatically handle common security checks related to
  account validation.

  ```rs
  #[derive(Accounts)]
  pub struct Initialize<'info> {
      #[account(init, payer = signer, space = 8 + 8)]
      pub new_account: Account<'info, NewAccount>,
      #[account(mut)]
      pub signer: Signer<'info>,
      pub system_program: Program<'info, System>,
  }
  ```

- The `#[account]` macro is used to define a struct that represents the data
  structure of an account created and owned by the program.

  ```rs
  #[account]
  pub struct NewAccount {
    data: u64
  }
  ```

</AccordionItem>
</Accordion>

### Build and Deploy Program

To build the program, simply run `build` in the terminal.

```shell filename="Terminal"
build
```

Notice that the address in `declare_id!()` has been updated. This is your
program's on-chain address.

<Accordion>
<AccordionItem title="Output">

```shell filename="Terminal"
$ build
Building...
Build successful. Completed in 1.46s.
```

</AccordionItem>
</Accordion>

Once the program is built, run `deploy` in the terminal to deploy the program to
the network (devnet by default). To deploy a program, SOL must be allocated to
the on-chain account that stores the program.

Before deployment, ensure you have enough SOL. You can get devnet SOL by either
running `solana airdrop 5` in the Playground terminal or using the
[Web Faucet](https://faucet.solana.com/).

```shell filename="Terminal"
deploy
```

<Accordion>
<AccordionItem title="Output">

```shell filename="Terminal"
$ deploy
Deploying... This could take a while depending on the program size and network conditions.
Warning: 1 transaction not confirmed, retrying...
Deployment successful. Completed in 19s.
```

</AccordionItem>
</Accordion>

Alternatively, you can also use the `Build` and `Deploy` buttons on the
left-side panel.

![Build and Deploy](/assets/docs/intro/quickstart/pg-build-deploy.png)

Once the program is deployed, you can now invoke its instructions.

### Test Program

Included with the starter code is a test file found in `tests/anchor.test.ts`.
This file demonstrates how to invoke the `initialize` instruction on the starter
program from the client.

```ts filename="anchor.test.ts"
// No imports needed: web3, anchor, pg and more are globally available

describe("Test", () => {
  it("initialize", async () => {
    // Generate keypair for the new account
    const newAccountKp = new web3.Keypair();

    // Send transaction
    const data = new BN(42);
    const txHash = await pg.program.methods
      .initialize(data)
      .accounts({
        newAccount: newAccountKp.publicKey,
        signer: pg.wallet.publicKey,
        systemProgram: web3.SystemProgram.programId,
      })
      .signers([newAccountKp])
      .rpc();
    console.log(`Use 'solana confirm -v ${txHash}' to see the logs`);

    // Confirm transaction
    await pg.connection.confirmTransaction(txHash);

    // Fetch the created account
    const newAccount = await pg.program.account.newAccount.fetch(
      newAccountKp.publicKey,
    );

    console.log("On-chain data is:", newAccount.data.toString());

    // Check whether the data on-chain is equal to local 'data'
    assert(data.eq(newAccount.data));
  });
});
```

To run the test file once the program is deployed, run `test` in the terminal.

```shell filename="Terminal"
test
```

You should see an output indicating that the test passed successfully.

<Accordion>
<AccordionItem title="Output">

```shell filename="Terminal"
$ test
Running tests...
  hello_anchor.test.ts:
  hello_anchor
    Use 'solana confirm -v 3TewJtiUz1EgtT88pLJHvKFzqrzDNuHVi8CfD2mWmHEBAaMfC5NAaHdmr19qQYfTiBace6XUmADvR4Qrhe8gH5uc' to see the logs
    On-chain data is: 42
    ✔ initialize (961ms)
  1 passing (963ms)
```

</AccordionItem>
</Accordion>

You can also use the `Test` button on the left-side panel.

![Run Test](/assets/docs/intro/quickstart/pg-test.png)

You can then view the transaction logs by running the `solana confirm -v`
command and specifying the transaction hash (signature) from the test output:

```shell filename="Terminal"
solana confirm -v [TxHash]
```

For example:

```shell filename="Terminal"
solana confirm -v 3TewJtiUz1EgtT88pLJHvKFzqrzDNuHVi8CfD2mWmHEBAaMfC5NAaHdmr19qQYfTiBace6XUmADvR4Qrhe8gH5uc
```

<Accordion>
<AccordionItem title="Output">

```shell filename="Terminal" {29-35}
$ solana confirm -v 3TewJtiUz1EgtT88pLJHvKFzqrzDNuHVi8CfD2mWmHEBAaMfC5NAaHdmr19qQYfTiBace6XUmADvR4Qrhe8gH5uc
RPC URL: https://api.devnet.solana.com
Default Signer: Playground Wallet
Commitment: confirmed

Transaction executed in slot 308150984:
  Block Time: 2024-06-25T12:52:05-05:00
  Version: legacy
  Recent Blockhash: 7AnZvY37nMhCybTyVXJ1umcfHSZGbngnm4GZx6jNRTNH
  Signature 0: 3TewJtiUz1EgtT88pLJHvKFzqrzDNuHVi8CfD2mWmHEBAaMfC5NAaHdmr19qQYfTiBace6XUmADvR4Qrhe8gH5uc
  Signature 1: 3TrRbqeMYFCkjsxdPExxBkLAi9SB2pNUyg87ryBaTHzzYtGjbsAz9udfT9AkrjSo1ZjByJgJHBAdRVVTZv6B87PQ
  Account 0: srw- 3z9vL1zjN6qyAFHhHQdWYRTFAcy69pJydkZmSFBKHg1R (fee payer)
  Account 1: srw- c7yy8zdP8oeZ2ewbSb8WWY2yWjDpg3B43jk3478Nv7J
  Account 2: -r-- 11111111111111111111111111111111
  Account 3: -r-x 2VvQ11q8xrn5tkPNyeraRsPaATdiPx8weLAD8aD4dn2r
  Instruction 0
    Program:   2VvQ11q8xrn5tkPNyeraRsPaATdiPx8weLAD8aD4dn2r (3)
    Account 0: c7yy8zdP8oeZ2ewbSb8WWY2yWjDpg3B43jk3478Nv7J (1)
    Account 1: 3z9vL1zjN6qyAFHhHQdWYRTFAcy69pJydkZmSFBKHg1R (0)
    Account 2: 11111111111111111111111111111111 (2)
    Data: [175, 175, 109, 31, 13, 152, 155, 237, 42, 0, 0, 0, 0, 0, 0, 0]
  Status: Ok
    Fee: ◎0.00001
    Account 0 balance: ◎5.47001376 -> ◎5.46900152
    Account 1 balance: ◎0 -> ◎0.00100224
    Account 2 balance: ◎0.000000001
    Account 3 balance: ◎0.00139896
  Log Messages:
    Program 2VvQ11q8xrn5tkPNyeraRsPaATdiPx8weLAD8aD4dn2r invoke [1]
    Program log: Instruction: Initialize
    Program 11111111111111111111111111111111 invoke [2]
    Program 11111111111111111111111111111111 success
    Program log: Changed data to: 42!
    Program 2VvQ11q8xrn5tkPNyeraRsPaATdiPx8weLAD8aD4dn2r consumed 5661 of 200000 compute units
    Program 2VvQ11q8xrn5tkPNyeraRsPaATdiPx8weLAD8aD4dn2r success

Confirmed
```

</AccordionItem>
</Accordion>

Alternatively, you can view the transaction details on
[SolanaFM](https://solana.fm/) or
[Solana Explorer](https://explorer.solana.com/?cluster=devnet) by searching for
the transaction signature (hash).

<Callout>
  Reminder to update the cluster (network) connection on the Explorer you are
  using to match Solana Playground. Solana Playground's default cluster is devnet.
</Callout>

### Close Program

Lastly, the SOL allocated to the on-chain program can be fully recovered by
closing the program.

You can close a program by running the following command and specifying the
program address found in `declare_id!()`:

```shell filename="Terminal"
solana program close [ProgramID]
```

For example:

```shell filename="Terminal"
solana program close 2VvQ11q8xrn5tkPNyeraRsPaATdiPx8weLAD8aD4dn2r
```

<Accordion>
<AccordionItem title="Output">

```shell filename="Terminal"
$ solana program close 2VvQ11q8xrn5tkPNyeraRsPaATdiPx8weLAD8aD4dn2r
Closed Program Id 2VvQ11q8xrn5tkPNyeraRsPaATdiPx8weLAD8aD4dn2r, 2.79511512 SOL reclaimed
```

</AccordionItem>
<AccordionItem title="Explanation">

Only the upgrade authority of the program can close it. The upgrade authority is
set when the program is deployed, and it's the only account with permission to
modify or close the program. If the upgrade authority is revoked, then the
program becomes immutable and can never be closed or upgraded.

When deploying programs on Solana Playground, your Playground wallet is the
upgrade authority for all your programs.

</AccordionItem>
</Accordion>

Congratulations! You've just built and deployed your first Solana program using
the Anchor framework!

</Steps>