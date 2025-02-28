# Connect to program take home

We’ll want you to connect to a [version of the Solana progam we’re developing which is deployed on devnet](https://solscan.io/account/8icXpLgEgEVVbvhTAgL7W7AUMZbaUh1UJ1czMiQXCuVE?cluster=devnet).
The IDL is at `frontend/staking_devnet.json` and the auto-generated TS types are at `frontend/staking_devnet.ts`.

If you can configure the current frontend you already have to this program and add a form so users can call `stake()` and deposit their PGN ([as deployed here on devnet](https://solscan.io/token/6nTZUieY9foTJWxtXPtvDzDoocoCZYEsiRWcAhUg1WjC?cluster=devnet)) you’re all good.

It may help you to know the signature, in Rust, of the `stake()` function and a few relevant structs:

```rust
pub fn stake(ctx: Context<Stake>, amount: u64, timelock_boost: TimelockBoost) -> Result<()> {
    instructions::stake::handler(ctx, amount, timelock_boost)
}

/// Represents the different boost multipliers based on timelock duration
#[derive(AnchorSerialize, AnchorDeserialize, Clone, Copy, PartialEq, PartialOrd)]
pub enum TimelockBoost {
    /// No timelock (0 months) - 1x boost
    /// Applied to users where timelock has expired but remain staked
    NoLock = 1,
    /// 1 month timelock - 2x boost
    OneMonth = 2,
    /// 3 months timelock - 3x boost
    ThreeMonths = 3,
    /// 6 months timelock - 4x boost
    SixMonths = 4,
    /// 9 months timelock - 5x boost
    NineMonths = 5,
}

impl TimelockBoost {
    /// Get the boost multiplier value
    pub fn value(&self) -> u8 {
        *self as u8
    }

    /// Get the timelock duration in months
    pub fn duration_months(&self) -> u8 {
        match self {
            TimelockBoost::NoLock => 0,
            TimelockBoost::OneMonth => 1,
            TimelockBoost::ThreeMonths => 3,
            TimelockBoost::SixMonths => 6,
            TimelockBoost::NineMonths => 9,
        }
    }

    /// Get the boost from a duration in months
    pub fn from_months(months: u8) -> Option<Self> {
        match months {
            0 => Some(Self::NoLock),
            1 => Some(Self::OneMonth),
            3 => Some(Self::ThreeMonths),
            6 => Some(Self::SixMonths),
            9 => Some(Self::NineMonths),
            _ => None,
        }
    }
}

#[derive(Accounts)]
#[instruction(amount: u64, timelock_boost: TimelockBoost)]
pub struct Stake<'info> {
    #[account(
        mut,
        seeds = [b"program_state".as_ref()],
        bump = program_state.bump,
        constraint = !program_state.paused @ StakingError::ProgramPaused,
        constraint = program_state.can_stake @ StakingError::StakingDisabled,
    )]
    pub program_state: Account<'info, ProgramState>,

    #[account(
        init_if_needed,
        payer = user,
        space = UserStake::space(),
        seeds = [
            b"user_stake",
            user.key().as_ref(),
        ],
        bump,
    )]
    pub user_stake: Account<'info, UserStake>,

    #[account(
        mut,
        constraint = user_token_account.mint == program_state.pgn_mint @ StakingError::InvalidMint,
        constraint = user_token_account.owner == user.key() @ StakingError::InvalidTokenAccount
    )]
    pub user_token_account: Account<'info, TokenAccount>,

    #[account(
        mut,
        seeds = [b"vault".as_ref()],
        bump,
        constraint = vault.mint == program_state.pgn_mint @ StakingError::InvalidMint
    )]
    pub vault: Account<'info, TokenAccount>,

    #[account(mut)]
    pub user: Signer<'info>,

    pub token_program: Program<'info, Token>,
    pub system_program: Program<'info, System>,
}

#[account]
pub struct ProgramState {
    pub root_admin: Pubkey,
    /// Market Maker authority for creating reward batches
    pub market_maker: Pubkey,
    /// Address of the PGN token mint
    pub pgn_mint: Pubkey,
    /// Flag to enable/disable staking program-wide
    pub can_stake: bool,
    /// Flag for emergency pause of all operations
    pub paused: bool,
    /// Minimum lock period in seconds (1 month)
    pub minimum_lock_period: i64,
    /// Next available batch ID
    pub next_batch_id: u64,
    /// Bump seed for PDA derivation
    pub bump: u8,
}

/// Represents a user's stake in the program
#[account]
pub struct UserStake {
    /// The owner of this stake account
    pub owner: Pubkey,
    /// Amount of PGN tokens staked
    pub amount: u64,
    /// Unix timestamp when the timelock expires
    pub timelock_end: i64,
    /// Current timelock boost multiplier
    pub timelock_boost: TimelockBoost,
    /// Bump seed for PDA derivation
    pub bump: u8,
}

impl UserStake {
    /// Calculate space needed for account allocation
    pub fn space() -> usize {
        8 +     // discriminator
        32 +    // owner
        8 +     // amount
        8 +     // timelock_end
        1 +     // timelock_boost (enum)
        1       // bump
    }
    
    /// Get the seeds for PDA derivation
    pub fn get_pda_seeds<'a>(
        owner: &'a Pubkey,
        seeds: &'a mut [&'a [u8]; 2]
    ) {
        seeds[0] = b"user_stake";
        seeds[1] = owner.as_ref();
    }
}
```

It may also help you to see a happy path typescript test for the staking action:

```ts
const [userStakePDA] = PublicKey.findProgramAddressSync([Buffer.from('user_stake'), user.publicKey.toBuffer()], program.programId);

// Stake tokens
const tx = await program.methods
  .stake(
    new anchor.BN(100_000_000_000), // 100 tokens
    { oneMonth: {} }
  )
  .accounts({
    user: user.publicKey,
    userTokenAccount: userTokenAccount.address,
    vault: vaultPDA,
    userStake: userStakePDA,
    programState: programStatePDA,
    systemProgram: SystemProgram.programId,
    tokenProgram: TOKEN_PROGRAM_ID,
  })
  .signers([user])
  .rpc();
```

The test in typescript is not complete and you will have to learn (if you don’t already know) how to derive a PDA or an ATA on your own.

As far as libraries go, you will likely use `@coral-xyz/anchor` and `@solana/web3.js`, and potentially other respective @solana libs.

Hint: for the PDAs you will have to look at the seeds in the Rust code.

Any questions you may have along the way, feel free to ask me.

P.S.: Message me with your Solana devnet address so I can send you some PGN tokens to test with.