// =====================================================================================
// SAFE LAUNCH (SPARTA) - lib.rs (HARDENED MEV & ANTI-BUNDLE ENGINE) - FIXED
//
// FIXES IN THIS VERSION (search for "FIX:" to find them):
//   FIX 1: buy window baseline (buy_win_vsol) and sell window baseline (sell_win_vt) are now
//          refreshed whenever the window decays. Before, they could stay frozen at the launch
//          values and reject normal buys with BuyWindowCapExceeded.
//   FIX 2: a pending bond-slash notice only freezes `migrate` while the slash can still happen.
//          Once the bond release time has passed, the notice no longer blocks migration.
//   FIX 3: a new buy window never starts with a zero baseline (guards against divide/cap edge cases).
//
// WHALE CHANGE: WHALE_MAX_BPS is now 2_000 (20%) and WHALE_LOCK_SECS is now 18 days.
//
// Account sizes and error numbers are unchanged from the previous version.
// =====================================================================================
use anchor_lang::prelude::*;
use anchor_lang::solana_program::{
    instruction::{AccountMeta, Instruction},
    program::invoke_signed,
    pubkey,
};
use anchor_lang::system_program::{transfer, Transfer};
use anchor_spl::metadata::{
    create_metadata_accounts_v3, mpl_token_metadata::types::DataV2, CreateMetadataAccountsV3,
    Metadata,
};
use anchor_spl::token::{
    self, spl_token::instruction::AuthorityType, spl_token::native_mint, Mint, MintTo,
    CloseAccount, SetAuthority, SyncNative, Token, TokenAccount, Transfer as SplTransfer,
};

declare_id!("5LwkPaq754fX9MtxFoMe3xiVtpPMtZRfyHSqXhwSodoS");

// ---- SECURITY: only this wallet may run init_config ----
const INIT_AUTHORITY: Pubkey = pubkey!("Dmo3PTbgAQfYvegqBUA89av4SkF8LDYpvnxtXpQLxFNq");

const TOTAL_SUPPLY: u64 = 1_000_000_000_000_000;
const CURVE_SUPPLY: u64 = 800_000_000_000_000;
const START_VIRTUAL_SOL: u64 = 30_000_000_000;
const START_VIRTUAL_TOKENS: u64 = 1_073_000_000_000_000;
const MIN_TRADE_LAMPORTS: u64 = 1_000_000;
const SELL_FEE_BPS: u64 = 100;
const MAX_EARLY_TOTAL: u64 = 5_000_000_000; // 5 SOL
const MIN_SLOT_DELAY: u64 = 25;
const MAX_OWNER_FEE_BPS: u64 = 300; // 3% hard cap
const MAX_CREATOR_FEE_BPS: u64 = 100; // 1% hard cap

// ---- ANTI-BUNDLE & ANTI-SNIPE PARAMETERS ----
const STRICT_LAUNCH_SLOTS: u64 = 150; // First 150 slots (~60s): strictly 1 buy per slot launch-wide
const WALLET_SELL_COOLDOWN_SLOTS: u64 = 15; // Minimum 15 slots (~6s) between sells per wallet

// ---- SECURITY: hard ceilings the admin can never go past ----
const MAX_LOCK_SECS: i64 = 90 * 24 * 60 * 60;
const MAX_BOND_RELEASE_SECS: i64 = 30 * 24 * 60 * 60;
const MAX_EARLY_WINDOW_SECS: i64 = 24 * 60 * 60;
const MAX_END_FEE_BPS: u64 = 1_000;
const MAX_BOND_LAMPORTS: u64 = 10_000_000_000;
const MIN_MAX_WALLET_BPS: u64 = 50;

// ---- creator rules ----
const MAX_CREATOR_SHARE_BPS: u64 = 500;
const CREATOR_CLIFF_DIV: i64 = 4;

// ---- graduation rewards ----
const GRADUATION_CREATOR_REWARD: u64 = 500_000_000;
const GRADUATION_PLATFORM_FEE: u64 = 500_000_000;

// ---- MAINNET VALUES ----
const MIN_CREATOR_VEST_SECS: i64 = 18 * 24 * 60 * 60;
const MIN_EARLY_LOCK_SECS: i64 = 60 * 60;
const CONFIG_TIMELOCK_SECS: i64 = 24 * 60 * 60;
const MIN_BOND_RELEASE_SECS: i64 = 7 * 24 * 60 * 60;
const FLAG_LIFETIME_SECS: i64 = 7 * 24 * 60 * 60;
const PERMANENT_FLAG_CLAIM_BLOCK_SECS: i64 = 90 * 24 * 60 * 60;

// ---- protections ----
const SETTLE_SECS: i64 = 120;
const SLOT_SELL_CAP_BPS: u64 = 300; // 3% per slot to block fast multi-slot dumps
const DUMP_FEE_START_BPS: u64 = 300;
const DUMP_FEE_SECS: i64 = 600;
const POST_WINDOW_LOCK_SECS: i64 = 20 * 60;

// ---- whale rule ----
const WHALE_MAX_BPS: u64 = 2_000; // 20% of supply per wallet (was 1_500)
const WHALE_LOCK_SECS: i64 = 18 * 24 * 60 * 60; // 18 days (was 30)
const WHALE_TOTAL_BPS: u64 = 5_000; // 50% launch-wide whale-locked cap (was 4_000)

// ---- permanent launch-wide buy limit per slot ----
const SLOT_BUY_IMPACT_BPS: u64 = 500; // 5%
const GUARDIAN_PAUSE_SECS: i64 = 2 * 60 * 60;

// ---- sell-side fees ----
const FLIP_FEE_START_BPS: u64 = 500;
const FLIP_FULL_SECS: i64 = 60 * 60;
const FLIP_END_SECS: i64 = 24 * 60 * 60;
const IMPACT_FREE_BPS: u64 = 500;
const IMPACT_FEE_MAX_BPS: u64 = 500;
const MAX_SELL_EXTRA_BPS: u64 = 1_500;
const LOCKZONE_VEST_SECS: i64 = 7 * 24 * 60 * 60;

// ---- rolling decay rates ----
const BUY_WINDOW_SECS: i64 = 60;
const SELL_WINDOW_SECS: i64 = 60;
const BUY_WINDOW_IMPACT_BPS: u64 = 1_000;
const WINDOW_FREE_BPS: u64 = 200;
const WINDOW_FEE_MAX_BPS: u64 = 1_000;
const SLASH_DELAY_SECS: i64 = 48 * 60 * 60;

// ---- graduation & Meteora pool ----
const GRADUATE_AT_REMAINING: u64 = CURVE_SUPPLY / 100;
const POST_GRAD_LOCK_SECS: i64 = 24 * 60 * 60;
const MIGRATE_RENT_BUDGET: u64 = 100_000_000;

const DAMM_ID: Pubkey = pubkey!("cpamdpZCGKUy5JxQXB4dcpGPiikHawvSWAd6mEn1sGG");
const DISC_INIT_CUSTOMIZABLE_POOL: [u8; 8] = [20, 161, 241, 24, 189, 221, 180, 2];
const DISC_PERMANENT_LOCK_POSITION: [u8; 8] = [165, 176, 125, 6, 231, 171, 186, 213];
const POOL_FEE_START_NUM: u64 = 500_000_000;
const POOL_FEE_STEPS: u16 = 49;
const POOL_FEE_STEP_SECS: u64 = 6;
const POOL_FEE_STEP_DROP: u64 = 10_000_000;
const POOL_COMPOUNDING_FEE_BPS: u16 = 5_000;
const PLATFORM_FEE_SHARE_BPS: u64 = 6_000;
const DISC_CLAIM_POSITION_FEE: [u8; 8] = [180, 38, 154, 17, 133, 33, 162, 211];

fn to_u64(x: u128) -> Result<u64> {
    u64::try_from(x).map_err(|_| error!(ErrorCode::MathOverflow))
}

fn is_flag_active(flag: &AccountInfo, now: i64, for_claim: bool) -> Result<bool> {
    if flag.data_is_empty() {
        return Ok(false);
    }
    let d = flag.try_borrow_data()?;
    if d.len() < 8 + 32 + 8 + 1 {
        return Ok(true);
    }
    let mut b = [0u8; 8];
    b.copy_from_slice(&d[40..48]);
    let at = i64::from_le_bytes(b);
    if d[48] != 0 {
        if !for_claim {
            return Ok(true);
        }
        let until = at
            .checked_add(PERMANENT_FLAG_CLAIM_BLOCK_SECS)
            .ok_or(ErrorCode::MathOverflow)?;
        return Ok(now < until);
    }
    let until = at
        .checked_add(FLAG_LIFETIME_SECS)
        .ok_or(ErrorCode::MathOverflow)?;
    Ok(now < until)
}

fn quick_flip_bps(last_buy_at: i64, now: i64) -> u64 {
    if last_buy_at <= 0 {
        return 0;
    }
    let since = now.saturating_sub(last_buy_at).max(0);
    if since < FLIP_FULL_SECS {
        FLIP_FEE_START_BPS
    } else if since < FLIP_END_SECS {
        FLIP_FEE_START_BPS * ((FLIP_END_SECS - since) as u64)
            / ((FLIP_END_SECS - FLIP_FULL_SECS) as u64)
    } else {
        0
    }
}

fn validate_params(p: &ConfigParams) -> Result<()> {
    require!(p.start_fee_bps >= p.end_fee_bps, ErrorCode::BadConfig);
    require!(p.start_fee_bps <= 10_000, ErrorCode::BadConfig);
    require!(p.end_fee_bps <= MAX_END_FEE_BPS, ErrorCode::BadConfig);
    require!(
        p.early_window_secs > 0 && p.early_window_secs <= MAX_EARLY_WINDOW_SECS,
        ErrorCode::BadConfig
    );
    require!(
        p.bond_release_secs >= MIN_BOND_RELEASE_SECS
            && p.bond_release_secs <= MAX_BOND_RELEASE_SECS,
        ErrorCode::BadConfig
    );
    require!(
        p.early_lock_secs >= MIN_EARLY_LOCK_SECS && p.early_lock_secs <= MAX_LOCK_SECS,
        ErrorCode::BadConfig
    );
    require!(
        p.creator_vest_secs >= MIN_CREATOR_VEST_SECS && p.creator_vest_secs <= MAX_LOCK_SECS,
        ErrorCode::BadConfig
    );
    require!(p.bond_lamports <= MAX_BOND_LAMPORTS, ErrorCode::BadConfig);
    require!(
        p.max_early_buy_lamports <= MAX_EARLY_TOTAL,
        ErrorCode::BadConfig
    );
    require!(p.owner_fee_bps <= MAX_OWNER_FEE_BPS, ErrorCode::BadConfig);
    require!(
        p.creator_fee_bps <= MAX_CREATOR_FEE_BPS,
        ErrorCode::BadConfig
    );
    require!(p.max_wallet_bps <= 10_000, ErrorCode::BadConfig);
    require!(
        p.max_wallet_bps == 0 || p.max_wallet_bps >= MIN_MAX_WALLET_BPS,
        ErrorCode::BadConfig
    );
    require!(
        p.max_slot_buy_lamports >= MIN_TRADE_LAMPORTS
            && p.max_slot_buy_lamports <= MAX_EARLY_TOTAL,
        ErrorCode::BadConfig
    );
    Ok(())
}

fn apply_params(c: &mut Config, p: &ConfigParams) {
    c.bond_lamports = p.bond_lamports;
    c.bond_release_secs = p.bond_release_secs;
    c.early_window_secs = p.early_window_secs;
    c.early_lock_secs = p.early_lock_secs;
    c.creator_vest_secs = p.creator_vest_secs;
    c.max_early_buy_lamports = p.max_early_buy_lamports;
    c.start_fee_bps = p.start_fee_bps;
    c.end_fee_bps = p.end_fee_bps;
    c.owner_fee_bps = p.owner_fee_bps;
    c.creator_fee_bps = p.creator_fee_bps;
    c.max_wallet_bps = p.max_wallet_bps;
    c.max_slot_buy_lamports = p.max_slot_buy_lamports;
}

fn pay_from_config<'info>(
    from: &AccountInfo<'info>,
    to: &AccountInfo<'info>,
    amount: u64,
) -> Result<()> {
    let min = Rent::get()?.minimum_balance(from.data_len());
    let left = from
        .lamports()
        .checked_sub(amount)
        .ok_or(ErrorCode::MathOverflow)?;
    require!(left >= min, ErrorCode::InsufficientPool);
    let new_to = to
        .lamports()
        .checked_add(amount)
        .ok_or(ErrorCode::MathOverflow)?;
    **from.try_borrow_mut_lamports()? = left;
    **to.try_borrow_mut_lamports()? = new_to;
    Ok(())
}

pub mod pool_math {
    pub const DEAD_LIQUIDITY: u128 = 100u128 << 64;

    pub fn isqrt(n: u128) -> u128 {
        if n < 2 {
            return n;
        }
        let bits = 128 - n.leading_zeros();
        let mut x: u128 = 1u128 << ((bits + 1) / 2);
        loop {
            let y = (x + n / x) / 2;
            if y >= x {
                return x;
            }
            x = y;
        }
    }

    pub fn plan(a: u64, b: u64) -> Option<(u128, u128)> {
        if a == 0 || b == 0 {
            return None;
        }
        let (a, b) = (a as u128, b as u128);
        let shift: u32 = if b < (1u128 << 40) { 88 } else { 64 };
        let num = b.checked_mul(1u128 << shift)?;
        let root = isqrt(num / a);
        let sqrt_price = root << ((128 - shift) / 2);
        if sqrt_price == 0 {
            return None;
        }
        let liquidity = a.checked_mul(sqrt_price)?;
        if liquidity <= DEAD_LIQUIDITY {
            return None;
        }
        Some((sqrt_price, liquidity))
    }

    pub fn amounts(sqrt_price: u128, liquidity: u128) -> Option<(u128, u128)> {
        let a = (liquidity + sqrt_price - 1) / sqrt_price;
        let hi = (liquidity >> 64) * (sqrt_price >> 64);
        let m1 = (liquidity >> 64) * (sqrt_price & u64::MAX as u128);
        let m2 = (liquidity & u64::MAX as u128) * (sqrt_price >> 64);
        let lo = (liquidity & u64::MAX as u128) * (sqrt_price & u64::MAX as u128);
        let mid = m1.checked_add(m2)?;
        let mut q = hi.checked_add(mid >> 64)?;
        let rem_nonzero = (mid & u64::MAX as u128) != 0 || lo != 0;
        if rem_nonzero {
            q = q.checked_add(1)?;
        }
        Some((a, q))
    }
}

const TOKEN_2022_ID: Pubkey = pubkey!("TokenzQdBNbLqP5VEhdkAS6EPFLC1PHnBqCXEpPxuEb");

#[program]
pub mod safe_launch {
    use super::*;

    pub fn init_config(ctx: Context<InitConfig>, p: ConfigParams) -> Result<()> {
        require_keys_eq!(
            ctx.accounts.admin.key(),
            INIT_AUTHORITY,
            ErrorCode::NotInitAuthority
        );
        validate_params(&p)?;
        let c = &mut ctx.accounts.config;
        c.admin = ctx.accounts.admin.key();
        c.pending_admin = Pubkey::default();
        c.guardian = Pubkey::default();
        c.flagger = Pubkey::default();
        c.paused = false;
        c.owner_fees_owed = 0;
        c.pending = p.clone();
        c.pending_ready_at = 0;
        apply_params(c, &p);
        emit!(ConfigUpdated {
            admin: c.admin,
            slot: Clock::get()?.slot,
        });
        Ok(())
    }

    pub fn propose_config(ctx: Context<AdminOnly>, p: ConfigParams) -> Result<()> {
        validate_params(&p)?;
        let clock = Clock::get()?;
        let ready_at = clock
            .unix_timestamp
            .checked_add(CONFIG_TIMELOCK_SECS)
            .ok_or(ErrorCode::MathOverflow)?;
        let c = &mut ctx.accounts.config;
        c.pending = p;
        c.pending_ready_at = ready_at;
        emit!(ConfigProposed {
            admin: ctx.accounts.admin.key(),
            ready_at,
            slot: clock.slot,
        });
        Ok(())
    }

    pub fn apply_config(ctx: Context<AdminOnly>) -> Result<()> {
        let clock = Clock::get()?;
        let c = &mut ctx.accounts.config;
        require!(c.pending_ready_at > 0, ErrorCode::NoPendingConfig);
        require!(
            clock.unix_timestamp >= c.pending_ready_at,
            ErrorCode::TimelockActive
        );
        let p = c.pending.clone();
        validate_params(&p)?;
        apply_params(c, &p);
        c.pending_ready_at = 0;
        emit!(ConfigUpdated {
            admin: ctx.accounts.admin.key(),
            slot: clock.slot,
        });
        Ok(())
    }

    pub fn cancel_config(ctx: Context<AdminOnly>) -> Result<()> {
        ctx.accounts.config.pending_ready_at = 0;
        emit!(ConfigCancelled {
            admin: ctx.accounts.admin.key(),
            slot: Clock::get()?.slot,
        });
        Ok(())
    }

    pub fn set_paused(ctx: Context<AdminOnly>, paused: bool) -> Result<()> {
        ctx.accounts.config.paused = paused;
        emit!(PausedChanged {
            by: ctx.accounts.admin.key(),
            paused,
            slot: Clock::get()?.slot,
        });
        Ok(())
    }

    pub fn set_guardian(ctx: Context<AdminOnly>, guardian: Pubkey) -> Result<()> {
        ctx.accounts.config.guardian = guardian;
        emit!(GuardianSet {
            admin: ctx.accounts.admin.key(),
            guardian,
            slot: Clock::get()?.slot,
        });
        Ok(())
    }

    pub fn set_flagger(ctx: Context<AdminOnly>, flagger: Pubkey) -> Result<()> {
        ctx.accounts.config.flagger = flagger;
        emit!(FlaggerSet {
            admin: ctx.accounts.admin.key(),
            flagger,
            slot: Clock::get()?.slot,
        });
        Ok(())
    }

    pub fn guardian_pause_launch(ctx: Context<GuardianPause>) -> Result<()> {
        let clock = Clock::get()?;
        let by = ctx.accounts.signer.key();
        let is_admin = by == ctx.accounts.config.admin;
        let launch_key = ctx.accounts.launch.key();
        let l = &mut ctx.accounts.launch;
        if is_admin {
            l.paused = true;
        } else {
            require!(!l.guardian_used, ErrorCode::GuardianPauseUsed);
            let until = clock
                .unix_timestamp
                .checked_add(GUARDIAN_PAUSE_SECS)
                .ok_or(ErrorCode::MathOverflow)?;
            l.guardian_used = true;
            l.paused_until = until;
            emit!(GuardianPaused {
                launch: launch_key,
                by,
                until,
                slot: clock.slot,
            });
        }
        emit!(LaunchPausedChanged {
            launch: launch_key,
            by,
            paused: true,
            slot: clock.slot,
        });
        Ok(())
    }

    pub fn set_launch_paused(ctx: Context<AdminLaunch>, paused: bool) -> Result<()> {
        ctx.accounts.launch.paused = paused;
        if !paused {
            ctx.accounts.launch.paused_until = 0;
        }
        emit!(LaunchPausedChanged {
            launch: ctx.accounts.launch.key(),
            by: ctx.accounts.admin.key(),
            paused,
            slot: Clock::get()?.slot,
        });
        Ok(())
    }

    pub fn propose_admin(ctx: Context<AdminOnly>, new_admin: Pubkey) -> Result<()> {
        require!(new_admin != Pubkey::default(), ErrorCode::BadConfig);
        ctx.accounts.config.pending_admin = new_admin;
        emit!(AdminProposed {
            admin: ctx.accounts.admin.key(),
            pending_admin: new_admin,
            slot: Clock::get()?.slot,
        });
        Ok(())
    }

    pub fn accept_admin(ctx: Context<AcceptAdmin>) -> Result<()> {
        let c = &mut ctx.accounts.config;
        let old_admin = c.admin;
        c.admin = ctx.accounts.new_admin.key();
        c.pending_admin = Pubkey::default();
        emit!(AdminChanged {
            old_admin,
            new_admin: c.admin,
            slot: Clock::get()?.slot,
        });
        Ok(())
    }

    pub fn withdraw_fees(ctx: Context<AdminOnly>) -> Result<()> {
        let amount = ctx.accounts.config.owner_fees_owed;
        require!(amount > 0, ErrorCode::TooSmall);
        ctx.accounts.config.owner_fees_owed = 0;
        pay_from_config(
            &ctx.accounts.config.to_account_info(),
            &ctx.accounts.admin.to_account_info(),
            amount,
        )?;
        emit!(FeesWithdrawn {
            admin: ctx.accounts.admin.key(),
            amount,
            slot: Clock::get()?.slot,
        });
        Ok(())
    }

    pub fn claim_creator_fees(ctx: Context<ClaimCreatorFees>) -> Result<()> {
        let amount = ctx.accounts.launch.creator_fees_owed;
        require!(amount > 0, ErrorCode::TooSmall);
        ctx.accounts.launch.creator_fees_owed = 0;
        pay_from_config(
            &ctx.accounts.config.to_account_info(),
            &ctx.accounts.creator.to_account_info(),
            amount,
        )
    }

    pub fn create_launch(
        ctx: Context<CreateLaunch>,
        name: String,
        symbol: String,
        uri: String,
    ) -> Result<()> {
        require!(
            !name.is_empty()
                && name.len() <= 32
                && !symbol.is_empty()
                && symbol.len() <= 10
                && uri.len() <= 200,
            ErrorCode::BadMetadata
        );
        require!(!ctx.accounts.config.paused, ErrorCode::Paused);
        let clock = Clock::get()?;
        let now = clock.unix_timestamp;
        let bond = ctx.accounts.config.bond_lamports;
        let early_lock_secs = ctx.accounts.config.early_lock_secs;
        let creator_vest_secs = ctx.accounts.config.creator_vest_secs;
        let bond_release_secs = ctx.accounts.config.bond_release_secs;

        transfer(
            CpiContext::new(
                ctx.accounts.system_program.to_account_info(),
                Transfer {
                    from: ctx.accounts.creator.to_account_info(),
                    to: ctx.accounts.bond_vault.to_account_info(),
                },
            ),
            bond,
        )?;

        let bump = ctx.bumps.launch;
        {
            let l = &mut ctx.accounts.launch;
            l.creator = ctx.accounts.creator.key();
            l.mint = ctx.accounts.mint.key();
            l.created_at = now;
            l.virtual_sol = START_VIRTUAL_SOL;
            l.virtual_tokens = START_VIRTUAL_TOKENS;
            l.real_sol = 0;
            l.real_tokens = CURVE_SUPPLY;
            l.bond_slashed = false;
            l.bond_claimed = false;
            l.early_total = 0;
            l.created_slot = clock.slot;
            l.creator_fees_owed = 0;
            l.bond_amount = bond;
            l.bump = bump;
            l.graduated = false;
            l.graduated_at = 0;
            l.migrated = false;
            l.pool = Pubkey::default();
            l.early_lock_secs = early_lock_secs;
            l.creator_vest_secs = creator_vest_secs;
            l.bond_release_secs = bond_release_secs;
            l.paused = false;
            l.last_slot = 0;
            l.slot_spent = 0;
            l.position = Pubkey::default();
            l.position_nft_account = Pubkey::default();
            l.last_sell_slot = 0;
            l.slot_sold = 0;
            l.whale_total = 0;
            l.paused_until = 0;
            l.guardian_used = false;
            l.cap_slot = 0;
            l.cap_slot_vsol = 0;
            l.cap_slot_net = 0;
            l.buy_win_start = now;
            l.buy_win_vsol = START_VIRTUAL_SOL;
            l.buy_win_net = 0;
            l.sell_win_start = now;
            l.sell_win_sold = 0;
            l.sell_win_vt = START_VIRTUAL_TOKENS;
            l.slash_ready_at = 0;
        }

        let mint_key = ctx.accounts.mint.key();
        let seeds: &[&[u8]] = &[b"launch", mint_key.as_ref(), &[bump]];
        let signer = &[seeds];

        token::mint_to(
            CpiContext::new_with_signer(
                ctx.accounts.token_program.to_account_info(),
                MintTo {
                    mint: ctx.accounts.mint.to_account_info(),
                    to: ctx.accounts.token_vault.to_account_info(),
                    authority: ctx.accounts.launch.to_account_info(),
                },
                signer,
            ),
            TOTAL_SUPPLY,
        )?;

        create_metadata_accounts_v3(
            CpiContext::new_with_signer(
                ctx.accounts.token_metadata_program.to_account_info(),
                CreateMetadataAccountsV3 {
                    metadata: ctx.accounts.metadata.to_account_info(),
                    mint: ctx.accounts.mint.to_account_info(),
                    mint_authority: ctx.accounts.launch.to_account_info(),
                    payer: ctx.accounts.creator.to_account_info(),
                    update_authority: ctx.accounts.launch.to_account_info(),
                    system_program: ctx.accounts.system_program.to_account_info(),
                    rent: ctx.accounts.rent.to_account_info(),
                },
                signer,
            ),
            DataV2 {
                name,
                symbol,
                uri,
                seller_fee_basis_points: 0,
                creators: None,
                collection: None,
                uses: None,
            },
            false,
            true,
            None,
        )?;

        token::set_authority(
            CpiContext::new_with_signer(
                ctx.accounts.token_program.to_account_info(),
                SetAuthority {
                    current_authority: ctx.accounts.launch.to_account_info(),
                    account_or_mint: ctx.accounts.mint.to_account_info(),
                },
                signer,
            ),
            AuthorityType::MintTokens,
            None,
        )?;

        emit!(LaunchCreated {
            launch: ctx.accounts.launch.key(),
            mint: mint_key,
            creator: ctx.accounts.creator.key(),
            slot: clock.slot,
        });
        Ok(())
    }

    pub fn buy(ctx: Context<Buy>, sol_in: u64, min_tokens_out: u64) -> Result<()> {
        require!(sol_in >= MIN_TRADE_LAMPORTS, ErrorCode::TooSmall);
        let cfg: Config = Config::clone(&ctx.accounts.config);
        require!(!cfg.paused, ErrorCode::Paused);
        let clock = Clock::get()?;
        let now = clock.unix_timestamp;
        let buyer = ctx.accounts.buyer.key();
        let l = &mut ctx.accounts.launch;
        require!(!l.graduated, ErrorCode::Graduated);
        require!(!l.paused && now >= l.paused_until, ErrorCode::LaunchPaused);

        let elapsed = now
            .checked_sub(l.created_at)
            .ok_or(ErrorCode::MathOverflow)?;
        let in_early = elapsed < cfg.early_window_secs;
        let in_lock_zone = elapsed < cfg.early_window_secs.saturating_add(POST_WINDOW_LOCK_SECS);
        let is_creator = buyer == l.creator;

        // HARDENED DEFENSE 1: NO CREATOR EXEMPTION.
        // Everyone must respect the slot launch delay to stop slot 0 creation+buy bundles
        let ready = l
            .created_slot
            .checked_add(MIN_SLOT_DELAY)
            .ok_or(ErrorCode::MathOverflow)?;
        require!(clock.slot >= ready, ErrorCode::TooEarly);

        // HARDENED DEFENSE 2: STRICT 1-BUY-PER-SLOT ANTI-BUNDLE LOCK
        // Disables multi-wallet atomic Jito bundles in the first STRICT_LAUNCH_SLOTS slots
        if clock.slot < l.created_slot.saturating_add(STRICT_LAUNCH_SLOTS) {
            require!(l.last_slot != clock.slot, ErrorCode::AntiBundleSlotOccupied);
        }

        let guard_secs = cfg.early_window_secs.saturating_add(SETTLE_SECS);
        if elapsed < guard_secs {
            if clock.slot != l.last_slot {
                l.last_slot = clock.slot;
                l.slot_spent = 0;
            }
            l.slot_spent = l
                .slot_spent
                .checked_add(sol_in)
                .ok_or(ErrorCode::MathOverflow)?;
            require!(
                l.slot_spent <= cfg.max_slot_buy_lamports,
                ErrorCode::SlotCapExceeded
            );
        } else {
            l.last_slot = clock.slot;
        }

        let owner_fee = sol_in
            .checked_mul(cfg.owner_fee_bps)
            .ok_or(ErrorCode::MathOverflow)?
            / 10_000;
        let creator_fee = sol_in
            .checked_mul(cfg.creator_fee_bps)
            .ok_or(ErrorCode::MathOverflow)?
            / 10_000;
        let fees = owner_fee
            .checked_add(creator_fee)
            .ok_or(ErrorCode::MathOverflow)?;
        let base = sol_in.checked_sub(fees).ok_or(ErrorCode::MathOverflow)?;

        let fee_bps = current_fee_bps(&cfg, elapsed)?;
        let fee = base.checked_mul(fee_bps).ok_or(ErrorCode::MathOverflow)? / 10_000;
        let net = base.checked_sub(fee).ok_or(ErrorCode::MathOverflow)?;

        // per slot limits
        if clock.slot != l.cap_slot {
            l.cap_slot = clock.slot;
            l.cap_slot_vsol = l.virtual_sol;
            l.cap_slot_net = 0;
        }
        l.cap_slot_net = l
            .cap_slot_net
            .checked_add(net)
            .ok_or(ErrorCode::MathOverflow)?;
        require!(
            (l.cap_slot_net as u128) * 10_000
                <= (l.cap_slot_vsol as u128) * (SLOT_BUY_IMPACT_BPS as u128),
            ErrorCode::SlotCapExceeded
        );

        // HARDENED DEFENSE 3: CONTINUOUS SMOOTH DECAY (no 60-second cliff gaming)
        // FIX 1: the baseline (buy_win_vsol) is refreshed on every decay step, so it can no
        // longer stay frozen at an old, smaller value and wrongly reject normal buys.
        let decay_seconds = now.saturating_sub(l.buy_win_start).max(0);
        if decay_seconds >= BUY_WINDOW_SECS {
            l.buy_win_start = now;
            l.buy_win_vsol = l.virtual_sol;
            l.buy_win_net = 0;
        } else if decay_seconds > 0 {
            let retain_bps = 10_000u128
                .saturating_sub((decay_seconds as u128 * 10_000) / (BUY_WINDOW_SECS as u128));
            l.buy_win_net = to_u64((l.buy_win_net as u128 * retain_bps) / 10_000)?;
            l.buy_win_start = now;
            l.buy_win_vsol = l.virtual_sol; // FIX 1
        }
        // FIX 3: never leave a zero baseline (old launches may carry zeros)
        if l.buy_win_vsol == 0 {
            l.buy_win_vsol = l.virtual_sol;
        }
        l.buy_win_net = l
            .buy_win_net
            .checked_add(net)
            .ok_or(ErrorCode::MathOverflow)?;
        require!(
            (l.buy_win_net as u128) * 10_000
                <= (l.buy_win_vsol as u128) * (BUY_WINDOW_IMPACT_BPS as u128),
            ErrorCode::BuyWindowCapExceeded
        );

        let k = (l.virtual_sol as u128)
            .checked_mul(l.virtual_tokens as u128)
            .ok_or(ErrorCode::MathOverflow)?;
        let new_vs = (l.virtual_sol as u128)
            .checked_add(net as u128)
            .ok_or(ErrorCode::MathOverflow)?;
        let new_vt = k.checked_add(new_vs - 1).ok_or(ErrorCode::MathOverflow)? / new_vs;
        let tokens_out = to_u64(
            (l.virtual_tokens as u128)
                .checked_sub(new_vt)
                .ok_or(ErrorCode::MathOverflow)?,
        )?;
        require!(
            tokens_out > 0 && tokens_out <= l.real_tokens,
            ErrorCode::CurveExhausted
        );
        require!(tokens_out >= min_tokens_out, ErrorCode::Slippage);

        let pos = &mut ctx.accounts.position;
        if pos.owner == Pubkey::default() {
            pos.owner = buyer;
            pos.launch = l.key();
        }

        let mut excess: u64 = 0;
        let mut new_whale_total: u64 = l.whale_total;

        if is_creator {
            require!(in_early, ErrorCode::CreatorBuyWindow);
            let limit = (TOTAL_SUPPLY as u128) * (MAX_CREATOR_SHARE_BPS as u128) / 10_000;
            let held = (ctx.accounts.buyer_ata.amount as u128)
                + (pos.locked_amount as u128)
                + (pos.released as u128)
                + (tokens_out as u128);
            require!(held <= limit, ErrorCode::CreatorShareCap);
        } else if cfg.max_wallet_bps > 0 {
            let free_limit = (TOTAL_SUPPLY as u128) * (cfg.max_wallet_bps as u128) / 10_000;
            let held_before = (ctx.accounts.buyer_ata.amount as u128)
                + (pos.locked_amount as u128)
                + (pos.whale_total.saturating_sub(pos.whale_released) as u128);
            let held_after = held_before + (tokens_out as u128);
            if in_lock_zone {
                require!(held_after <= free_limit, ErrorCode::MaxWallet);
            } else {
                let hard = (TOTAL_SUPPLY as u128)
                    * (WHALE_MAX_BPS.max(cfg.max_wallet_bps) as u128)
                    / 10_000;
                require!(held_after <= hard, ErrorCode::MaxWallet);
                excess = to_u64(held_after.saturating_sub(free_limit.max(held_before)))?;
                if excess > 0 {
                    new_whale_total = l
                        .whale_total
                        .checked_add(excess)
                        .ok_or(ErrorCode::MathOverflow)?;
                    require!(
                        (new_whale_total as u128) * 10_000
                            <= (TOTAL_SUPPLY as u128) * (WHALE_TOTAL_BPS as u128),
                        ErrorCode::WhaleLimit
                    );
                }
            }
        }

        let locked = in_lock_zone || is_creator;
        require!(
            !is_flag_active(&ctx.accounts.flag.to_account_info(), now, false)?,
            ErrorCode::WalletFlagged
        );
        if in_early {
            pos.early_spent = pos
                .early_spent
                .checked_add(sol_in)
                .ok_or(ErrorCode::MathOverflow)?;
            require!(
                pos.early_spent <= cfg.max_early_buy_lamports,
                ErrorCode::EarlyCapExceeded
            );
            l.early_total = l
                .early_total
                .checked_add(sol_in)
                .ok_or(ErrorCode::MathOverflow)?;
            require!(
                l.early_total <= MAX_EARLY_TOTAL,
                ErrorCode::EarlyCapExceeded
            );
        }

        transfer(
            CpiContext::new(
                ctx.accounts.system_program.to_account_info(),
                Transfer {
                    from: ctx.accounts.buyer.to_account_info(),
                    to: ctx.accounts.sol_vault.to_account_info(),
                },
            ),
            base,
        )?;
        if fees > 0 {
            transfer(
                CpiContext::new(
                    ctx.accounts.system_program.to_account_info(),
                    Transfer {
                        from: ctx.accounts.buyer.to_account_info(),
                        to: ctx.accounts.config.to_account_info(),
                    },
                ),
                fees,
            )?;
        }

        l.virtual_sol = to_u64(new_vs)?;
        l.virtual_tokens = to_u64(new_vt)?;
        l.real_sol = l
            .real_sol
            .checked_add(base)
            .ok_or(ErrorCode::MathOverflow)?;
        l.real_tokens = l
            .real_tokens
            .checked_sub(tokens_out)
            .ok_or(ErrorCode::MathOverflow)?;
        l.creator_fees_owed = l
            .creator_fees_owed
            .checked_add(creator_fee)
            .ok_or(ErrorCode::MathOverflow)?;
        ctx.accounts.config.owner_fees_owed = ctx
            .accounts
            .config
            .owner_fees_owed
            .checked_add(owner_fee)
            .ok_or(ErrorCode::MathOverflow)?;

        pos.last_buy_at = now;
        pos.bought_total = pos
            .bought_total
            .checked_add(tokens_out)
            .ok_or(ErrorCode::MathOverflow)?;

        let mint_key = l.mint;
        let seeds: &[&[u8]] = &[b"launch", mint_key.as_ref(), &[l.bump]];
        let signer = &[seeds];
        let to_lock = if locked { tokens_out } else { excess };
        let to_free = tokens_out
            .checked_sub(to_lock)
            .ok_or(ErrorCode::MathOverflow)?;
        for (amount, target) in [
            (to_lock, ctx.accounts.escrow.to_account_info()),
            (to_free, ctx.accounts.buyer_ata.to_account_info()),
        ] {
            if amount > 0 {
                token::transfer(
                    CpiContext::new_with_signer(
                        ctx.accounts.token_program.to_account_info(),
                        SplTransfer {
                            from: ctx.accounts.token_vault.to_account_info(),
                            to: target,
                            authority: l.to_account_info(),
                        },
                        signer,
                    ),
                    amount,
                )?;
            }
        }

        if locked {
            pos.locked_amount = pos
                .locked_amount
                .checked_add(tokens_out)
                .ok_or(ErrorCode::MathOverflow)?;
            let lock_secs = if is_creator {
                l.creator_vest_secs
            } else {
                l.early_lock_secs
                    .max(cfg.early_window_secs.saturating_add(POST_WINDOW_LOCK_SECS))
            };
            let unlock = l
                .created_at
                .checked_add(lock_secs)
                .ok_or(ErrorCode::MathOverflow)?;
            if unlock > pos.unlock_at {
                pos.unlock_at = unlock;
            }
        }

        if excess > 0 {
            let remaining = pos.whale_total.saturating_sub(pos.whale_released);
            pos.whale_total = remaining
                .checked_add(excess)
                .ok_or(ErrorCode::MathOverflow)?;
            pos.whale_released = 0;
            pos.whale_start = now;
            pos.whale_end = now
                .checked_add(WHALE_LOCK_SECS)
                .ok_or(ErrorCode::MathOverflow)?;
            l.whale_total = new_whale_total;
            emit!(WhaleLocked {
                launch: l.key(),
                wallet: buyer,
                amount: excess,
                wallet_total_locked: pos.whale_total,
                launch_total_locked: l.whale_total,
                release_end: pos.whale_end,
                slot: clock.slot,
            });
        }

        emit!(Traded {
            launch: l.key(),
            trader: buyer,
            is_buy: true,
            sol_amount: sol_in,
            token_amount: tokens_out,
            virtual_sol: l.virtual_sol,
            virtual_tokens: l.virtual_tokens,
            ts: now,
        });

        if l.real_tokens <= GRADUATE_AT_REMAINING {
            l.graduated = true;
            l.graduated_at = now;
            emit!(Graduated {
                launch: l.key(),
                remaining_tokens: l.real_tokens,
                sol: l.real_sol,
                ts: now,
            });
        }
        Ok(())
    }

    pub fn sell(ctx: Context<Sell>, tokens_in: u64, min_sol_out: u64) -> Result<()> {
        require!(tokens_in > 0, ErrorCode::TooSmall);
        let cfg: Config = Config::clone(&ctx.accounts.config);

        let clock = Clock::get()?;
        let now = clock.unix_timestamp;

        let pos = &mut ctx.accounts.position;

        // HARDENED DEFENSE 4: CONSECUTIVE-SLOT SELL COOLDOWN PER WALLET
        // Neutralizes stair-case bot dumping across consecutive 400ms blocks
        if pos.last_sell_slot > 0 {
            require!(
                clock.slot >= pos.last_sell_slot.saturating_add(WALLET_SELL_COOLDOWN_SLOTS),
                ErrorCode::SellCooldownActive
            );
        }
        pos.last_sell_slot = clock.slot;

        let allowance = pos.bought_total.saturating_sub(pos.sold_total);
        require!(tokens_in <= allowance, ErrorCode::NotYourTokens);
        pos.sold_total = pos
            .sold_total
            .checked_add(tokens_in)
            .ok_or(ErrorCode::MathOverflow)?;
        let last_buy_at = pos.last_buy_at;

        let l = &mut ctx.accounts.launch;
        require!(!l.migrated, ErrorCode::Migrated);

        if clock.slot != l.last_sell_slot {
            l.last_sell_slot = clock.slot;
            l.slot_sold = 0;
        }
        l.slot_sold = l
            .slot_sold
            .checked_add(tokens_in)
            .ok_or(ErrorCode::MathOverflow)?;
        require!(
            (l.slot_sold as u128) * 10_000 <= (l.virtual_tokens as u128) * (SLOT_SELL_CAP_BPS as u128),
            ErrorCode::SlotSellCapExceeded
        );

        let after_window = now
            .saturating_sub(l.created_at)
            .saturating_sub(cfg.early_window_secs)
            .max(0);
        let dump_bps: u64 = if after_window < DUMP_FEE_SECS {
            DUMP_FEE_START_BPS * ((DUMP_FEE_SECS - after_window) as u64) / (DUMP_FEE_SECS as u64)
        } else {
            0
        };

        let flip_bps = quick_flip_bps(last_buy_at, now);

        let impact_bps = to_u64(
            (tokens_in as u128) * 10_000 / ((l.virtual_tokens as u128) + (tokens_in as u128)),
        )?;
        let impact_fee_bps = impact_bps
            .saturating_sub(IMPACT_FREE_BPS)
            .min(IMPACT_FEE_MAX_BPS);

        // Smooth decaying sell window
        // FIX 1: the baseline (sell_win_vt) is refreshed on every decay step.
        let sell_decay_seconds = now.saturating_sub(l.sell_win_start).max(0);
        if sell_decay_seconds >= SELL_WINDOW_SECS {
            l.sell_win_start = now;
            l.sell_win_sold = 0;
            l.sell_win_vt = l.virtual_tokens;
        } else if sell_decay_seconds > 0 {
            let retain_bps = 10_000u128
                .saturating_sub((sell_decay_seconds as u128 * 10_000) / (SELL_WINDOW_SECS as u128));
            l.sell_win_sold = to_u64((l.sell_win_sold as u128 * retain_bps) / 10_000)?;
            l.sell_win_start = now;
            l.sell_win_vt = l.virtual_tokens; // FIX 1
        }
        l.sell_win_sold = l
            .sell_win_sold
            .checked_add(tokens_in)
            .ok_or(ErrorCode::MathOverflow)?;
        let win_bps = to_u64(
            (l.sell_win_sold as u128) * 10_000 / (l.sell_win_vt.max(1) as u128),
        )?;
        let window_fee_bps = win_bps
            .saturating_sub(WINDOW_FREE_BPS)
            .saturating_mul(2)
            .min(WINDOW_FEE_MAX_BPS);

        let extra_bps = dump_bps
            .checked_add(flip_bps)
            .and_then(|x| x.checked_add(impact_fee_bps))
            .and_then(|x| x.checked_add(window_fee_bps))
            .ok_or(ErrorCode::MathOverflow)?
            .min(MAX_SELL_EXTRA_BPS);

        let k = (l.virtual_sol as u128)
            .checked_mul(l.virtual_tokens as u128)
            .ok_or(ErrorCode::MathOverflow)?;
        let new_vt = (l.virtual_tokens as u128)
            .checked_add(tokens_in as u128)
            .ok_or(ErrorCode::MathOverflow)?;
        let new_vs = k.checked_add(new_vt - 1).ok_or(ErrorCode::MathOverflow)? / new_vt;
        let gross = to_u64(
            (l.virtual_sol as u128)
                .checked_sub(new_vs)
                .ok_or(ErrorCode::MathOverflow)?,
        )?;

        let pool_fee = gross
            .checked_mul(SELL_FEE_BPS + extra_bps)
            .ok_or(ErrorCode::MathOverflow)?
            / 10_000;
        let owner_fee = gross
            .checked_mul(cfg.owner_fee_bps)
            .ok_or(ErrorCode::MathOverflow)?
            / 10_000;
        let creator_fee = gross
            .checked_mul(cfg.creator_fee_bps)
            .ok_or(ErrorCode::MathOverflow)?
            / 10_000;
        let fees = owner_fee
            .checked_add(creator_fee)
            .ok_or(ErrorCode::MathOverflow)?;
        let total_out = gross.checked_sub(pool_fee).ok_or(ErrorCode::MathOverflow)?;
        let sol_out = total_out.checked_sub(fees).ok_or(ErrorCode::MathOverflow)?;
        require!(sol_out >= min_sol_out, ErrorCode::Slippage);
        require!(total_out <= l.real_sol, ErrorCode::InsufficientPool);

        token::transfer(
            CpiContext::new(
                ctx.accounts.token_program.to_account_info(),
                SplTransfer {
                    from: ctx.accounts.seller_ata.to_account_info(),
                    to: ctx.accounts.token_vault.to_account_info(),
                    authority: ctx.accounts.seller.to_account_info(),
                },
            ),
            tokens_in,
        )?;

        l.virtual_sol = to_u64(new_vs)?;
        l.virtual_tokens = to_u64(new_vt)?;
        l.real_sol = l
            .real_sol
            .checked_sub(total_out)
            .ok_or(ErrorCode::MathOverflow)?;
        l.real_tokens = l
            .real_tokens
            .checked_add(tokens_in)
            .ok_or(ErrorCode::MathOverflow)?;
        l.creator_fees_owed = l
            .creator_fees_owed
            .checked_add(creator_fee)
            .ok_or(ErrorCode::MathOverflow)?;
        ctx.accounts.config.owner_fees_owed = ctx
            .accounts
            .config
            .owner_fees_owed
            .checked_add(owner_fee)
            .ok_or(ErrorCode::MathOverflow)?;

        let launch_key = l.key();
        let sv_bump = ctx.bumps.sol_vault;
        let seeds: &[&[u8]] = &[b"sol_vault", launch_key.as_ref(), &[sv_bump]];
        transfer(
            CpiContext::new_with_signer(
                ctx.accounts.system_program.to_account_info(),
                Transfer {
                    from: ctx.accounts.sol_vault.to_account_info(),
                    to: ctx.accounts.seller.to_account_info(),
                },
                &[seeds],
            ),
            sol_out,
        )?;
        if fees > 0 {
            transfer(
                CpiContext::new_with_signer(
                    ctx.accounts.system_program.to_account_info(),
                    Transfer {
                        from: ctx.accounts.sol_vault.to_account_info(),
                        to: ctx.accounts.config.to_account_info(),
                    },
                    &[seeds],
                ),
                fees,
            )?;
        }

        emit!(Traded {
            launch: launch_key,
            trader: ctx.accounts.seller.key(),
            is_buy: false,
            sol_amount: sol_out,
            token_amount: tokens_in,
            virtual_sol: l.virtual_sol,
            virtual_tokens: l.virtual_tokens,
            ts: now,
        });
        Ok(())
    }

    pub fn migrate(ctx: Context<Migrate>) -> Result<()> {
        require!(ctx.accounts.launch.graduated, ErrorCode::NotGraduated);
        require!(!ctx.accounts.launch.migrated, ErrorCode::AlreadyMigrated);

        // HARDENED DEFENSE 5: PREVENT EVADING A BOND SLASH BY RUSHING MIGRATION
        // FIX 2: the notice only blocks migration while the slash can still happen
        // (slash_bond is impossible once the bond release time has passed).
        {
            let lr = &ctx.accounts.launch;
            let release = lr
                .created_at
                .checked_add(lr.bond_release_secs)
                .ok_or(ErrorCode::MathOverflow)?;
            let now_ts = Clock::get()?.unix_timestamp;
            require!(
                lr.slash_ready_at == 0 || now_ts >= release,
                ErrorCode::SlashNoticePending
            );
        }

        let launch_key = ctx.accounts.launch.key();
        let mint_key = ctx.accounts.mint.key();
        let wsol_key = native_mint::ID;
        let nft_mint = ctx.accounts.position_nft_mint.key();

        let (hi, lo) = if mint_key > wsol_key {
            (mint_key, wsol_key)
        } else {
            (wsol_key, mint_key)
        };
        let find = |seeds: &[&[u8]]| Pubkey::find_program_address(seeds, &DAMM_ID).0;
        let pool_key = find(&[b"cpool", hi.as_ref(), lo.as_ref()]);
        require_keys_eq!(ctx.accounts.pool.key(), pool_key, ErrorCode::BadDexAccount);
        require_keys_eq!(
            ctx.accounts.position.key(),
            find(&[b"position", nft_mint.as_ref()]),
            ErrorCode::BadDexAccount
        );
        require_keys_eq!(
            ctx.accounts.position_nft_account.key(),
            find(&[b"position_nft_account", nft_mint.as_ref()]),
            ErrorCode::BadDexAccount
        );
        require_keys_eq!(
            ctx.accounts.dex_token_a_vault.key(),
            find(&[b"token_vault", mint_key.as_ref(), pool_key.as_ref()]),
            ErrorCode::BadDexAccount
        );
        require_keys_eq!(
            ctx.accounts.dex_token_b_vault.key(),
            find(&[b"token_vault", wsol_key.as_ref(), pool_key.as_ref()]),
            ErrorCode::BadDexAccount
        );
        require_keys_eq!(
            ctx.accounts.pool_authority.key(),
            find(&[b"pool_authority"]),
            ErrorCode::BadDexAccount
        );
        require_keys_eq!(
            ctx.accounts.event_authority.key(),
            find(&[b"__event_authority"]),
            ErrorCode::BadDexAccount
        );

        let tokens = ctx.accounts.token_vault.amount;
        let sol_total = ctx.accounts.sol_vault.to_account_info().lamports();
        let creator_reward = if ctx.accounts.launch.bond_slashed {
            0
        } else {
            GRADUATION_CREATOR_REWARD
        };
        let platform_fee = GRADUATION_PLATFORM_FEE;
        let payouts = creator_reward
            .checked_add(platform_fee)
            .ok_or(ErrorCode::MathOverflow)?;
        let sol = sol_total
            .checked_sub(payouts)
            .ok_or(ErrorCode::BadPoolPlan)?;
        let (sqrt_price, liquidity) = pool_math::plan(tokens, sol).ok_or(ErrorCode::BadPoolPlan)?;
        let (need_a, need_b) =
            pool_math::amounts(sqrt_price, liquidity).ok_or(ErrorCode::BadPoolPlan)?;
        require!(
            need_a <= tokens as u128 && need_b <= sol as u128,
            ErrorCode::BadPoolPlan
        );
        let locked_liquidity = liquidity
            .checked_sub(pool_math::DEAD_LIQUIDITY)
            .ok_or(ErrorCode::BadPoolPlan)?;

        let launch_bump = ctx.accounts.launch.bump;
        let launch_seeds: &[&[u8]] = &[b"launch", mint_key.as_ref(), &[launch_bump]];
        let sol_vault_seeds: &[&[u8]] =
            &[b"sol_vault", launch_key.as_ref(), &[ctx.bumps.sol_vault]];
        let mig_seeds: &[&[u8]] = &[b"mig", launch_key.as_ref(), &[ctx.bumps.mig]];

        transfer(
            CpiContext::new(
                ctx.accounts.system_program.to_account_info(),
                Transfer {
                    from: ctx.accounts.caller.to_account_info(),
                    to: ctx.accounts.mig.to_account_info(),
                },
            ),
            MIGRATE_RENT_BUDGET,
        )?;

        token::transfer(
            CpiContext::new_with_signer(
                ctx.accounts.token_program.to_account_info(),
                SplTransfer {
                    from: ctx.accounts.token_vault.to_account_info(),
                    to: ctx.accounts.mig_token_a.to_account_info(),
                    authority: ctx.accounts.launch.to_account_info(),
                },
                &[launch_seeds],
            ),
            tokens,
        )?;
        transfer(
            CpiContext::new_with_signer(
                ctx.accounts.system_program.to_account_info(),
                Transfer {
                    from: ctx.accounts.sol_vault.to_account_info(),
                    to: ctx.accounts.mig_token_b.to_account_info(),
                },
                &[sol_vault_seeds],
            ),
            sol,
        )?;

        if payouts > 0 {
            transfer(
                CpiContext::new_with_signer(
                    ctx.accounts.system_program.to_account_info(),
                    Transfer {
                        from: ctx.accounts.sol_vault.to_account_info(),
                        to: ctx.accounts.config.to_account_info(),
                    },
                    &[sol_vault_seeds],
                ),
                payouts,
            )?;
            ctx.accounts.config.owner_fees_owed = ctx
                .accounts
                .config
                .owner_fees_owed
                .checked_add(platform_fee)
                .ok_or(ErrorCode::MathOverflow)?;
            ctx.accounts.launch.creator_fees_owed = ctx
                .accounts
                .launch
                .creator_fees_owed
                .checked_add(creator_reward)
                .ok_or(ErrorCode::MathOverflow)?;
            emit!(GraduationPaid {
                launch: launch_key,
                creator_reward,
                platform_fee,
                slot: Clock::get()?.slot,
            });
        }
        token::sync_native(CpiContext::new(
            ctx.accounts.token_program.to_account_info(),
            SyncNative {
                account: ctx.accounts.mig_token_b.to_account_info(),
            },
        ))?;

        cpi_create_pool(&ctx.accounts, liquidity, sqrt_price, mig_seeds)?;
        cpi_lock_position(&ctx.accounts, locked_liquidity, mig_seeds)?;

        let left = ctx.accounts.mig.to_account_info().lamports();
        if left > 0 {
            transfer(
                CpiContext::new_with_signer(
                    ctx.accounts.system_program.to_account_info(),
                    Transfer {
                        from: ctx.accounts.mig.to_account_info(),
                        to: ctx.accounts.caller.to_account_info(),
                    },
                    &[mig_seeds],
                ),
                left,
            )?;
        }

        let now = Clock::get()?.unix_timestamp;
        let l = &mut ctx.accounts.launch;
        l.migrated = true;
        l.pool = pool_key;
        l.position = ctx.accounts.position.key();
        l.position_nft_account = ctx.accounts.position_nft_account.key();
        l.real_sol = 0;
        l.real_tokens = 0;
        emit!(Migrated {
            launch: launch_key,
            pool: pool_key,
            position: ctx.accounts.position.key(),
            token_amount: need_a as u64,
            sol_amount: need_b as u64,
            liquidity,
            ts: now,
        });
        Ok(())
    }

    pub fn claim_pool_fees(ctx: Context<ClaimPoolFees>) -> Result<()> {
        require!(ctx.accounts.launch.migrated, ErrorCode::NotMigrated);
        let launch_key = ctx.accounts.launch.key();
        let mint_key = ctx.accounts.mint.key();
        let wsol_key = native_mint::ID;
        let pool_key = ctx.accounts.launch.pool;

        require_keys_eq!(ctx.accounts.pool.key(), pool_key, ErrorCode::BadDexAccount);
        require_keys_eq!(
            ctx.accounts.position.key(),
            ctx.accounts.launch.position,
            ErrorCode::BadDexAccount
        );
        require_keys_eq!(
            ctx.accounts.position_nft_account.key(),
            ctx.accounts.launch.position_nft_account,
            ErrorCode::BadDexAccount
        );
        let find = |seeds: &[&[u8]]| Pubkey::find_program_address(seeds, &DAMM_ID).0;
        require_keys_eq!(
            ctx.accounts.dex_token_a_vault.key(),
            find(&[b"token_vault", mint_key.as_ref(), pool_key.as_ref()]),
            ErrorCode::BadDexAccount
        );
        require_keys_eq!(
            ctx.accounts.dex_token_b_vault.key(),
            find(&[b"token_vault", wsol_key.as_ref(), pool_key.as_ref()]),
            ErrorCode::BadDexAccount
        );
        require_keys_eq!(
            ctx.accounts.pool_authority.key(),
            find(&[b"pool_authority"]),
            ErrorCode::BadDexAccount
        );
        require_keys_eq!(
            ctx.accounts.event_authority.key(),
            find(&[b"__event_authority"]),
            ErrorCode::BadDexAccount
        );

        let mig_seeds: &[&[u8]] = &[b"mig", launch_key.as_ref(), &[ctx.bumps.mig]];

        cpi_claim_fee(&ctx.accounts, mig_seeds)?;
        ctx.accounts.mig_token_b.reload()?;
        let fee = ctx.accounts.mig_token_b.amount;
        require!(fee > 0, ErrorCode::NothingToClaim);

        token::close_account(CpiContext::new_with_signer(
            ctx.accounts.token_program.to_account_info(),
            CloseAccount {
                account: ctx.accounts.mig_token_b.to_account_info(),
                destination: ctx.accounts.caller.to_account_info(),
                authority: ctx.accounts.mig.to_account_info(),
            },
            &[mig_seeds],
        ))?;
        transfer(
            CpiContext::new(
                ctx.accounts.system_program.to_account_info(),
                Transfer {
                    from: ctx.accounts.caller.to_account_info(),
                    to: ctx.accounts.config.to_account_info(),
                },
            ),
            fee,
        )?;

        let mut platform_part = to_u64((fee as u128) * (PLATFORM_FEE_SHARE_BPS as u128) / 10_000)?;
        let mut creator_part = fee.checked_sub(platform_part).ok_or(ErrorCode::MathOverflow)?;
        if ctx.accounts.launch.bond_slashed {
            platform_part = fee;
            creator_part = 0;
        }
        ctx.accounts.config.owner_fees_owed = ctx
            .accounts
            .config
            .owner_fees_owed
            .checked_add(platform_part)
            .ok_or(ErrorCode::MathOverflow)?;
        ctx.accounts.launch.creator_fees_owed = ctx
            .accounts
            .launch
            .creator_fees_owed
            .checked_add(creator_part)
            .ok_or(ErrorCode::MathOverflow)?;
        emit!(PoolFeesClaimed {
            launch: launch_key,
            amount: fee,
            platform_part,
            creator_part,
            slot: Clock::get()?.slot,
        });
        Ok(())
    }

    pub fn claim_locked(ctx: Context<ClaimLocked>) -> Result<()> {
        let now = Clock::get()?.unix_timestamp;
        require!(
            !is_flag_active(&ctx.accounts.flag.to_account_info(), now, true)?,
            ErrorCode::WalletFlagged
        );
        let l = &ctx.accounts.launch;
        let pos = &mut ctx.accounts.position;

        if l.graduated {
            let after = l
                .graduated_at
                .checked_add(POST_GRAD_LOCK_SECS)
                .ok_or(ErrorCode::MathOverflow)?;
            require!(now >= after, ErrorCode::StillLocked);
        }

        let amount: u64;
        if pos.owner == l.creator {
            let total = (pos.locked_amount as u128)
                .checked_add(pos.released as u128)
                .ok_or(ErrorCode::MathOverflow)?;
            let elapsed = now
                .checked_sub(l.created_at)
                .ok_or(ErrorCode::MathOverflow)?
                .max(0);
            let vest = l.creator_vest_secs.max(1);
            let vested: u128 = if elapsed >= vest {
                total
            } else if elapsed < vest / CREATOR_CLIFF_DIV {
                0
            } else {
                total
                    .checked_mul(elapsed as u128)
                    .ok_or(ErrorCode::MathOverflow)?
                    / (vest as u128)
            };
            let claimable = vested.saturating_sub(pos.released as u128);
            amount = to_u64(claimable)?;
            require!(amount > 0, ErrorCode::StillLocked);
            pos.released = pos
                .released
                .checked_add(amount)
                .ok_or(ErrorCode::MathOverflow)?;
            pos.locked_amount = pos
                .locked_amount
                .checked_sub(amount)
                .ok_or(ErrorCode::MathOverflow)?;
        } else {
            require!(now >= pos.unlock_at, ErrorCode::StillLocked);
            let total = (pos.locked_amount as u128)
                .checked_add(pos.released as u128)
                .ok_or(ErrorCode::MathOverflow)?;
            let passed = now.saturating_sub(pos.unlock_at).max(0) as u128;
            let vested: u128 = if passed >= LOCKZONE_VEST_SECS as u128 {
                total
            } else {
                total
                    .checked_mul(passed)
                    .ok_or(ErrorCode::MathOverflow)?
                    / (LOCKZONE_VEST_SECS as u128)
            };
            amount = to_u64(vested.saturating_sub(pos.released as u128))?;
            require!(amount > 0, ErrorCode::StillLocked);
            pos.released = pos
                .released
                .checked_add(amount)
                .ok_or(ErrorCode::MathOverflow)?;
            pos.locked_amount = pos
                .locked_amount
                .checked_sub(amount)
                .ok_or(ErrorCode::MathOverflow)?;
        }

        let mint_key = l.mint;
        let seeds: &[&[u8]] = &[b"launch", mint_key.as_ref(), &[l.bump]];
        token::transfer(
            CpiContext::new_with_signer(
                ctx.accounts.token_program.to_account_info(),
                SplTransfer {
                    from: ctx.accounts.escrow.to_account_info(),
                    to: ctx.accounts.owner_ata.to_account_info(),
                    authority: ctx.accounts.launch.to_account_info(),
                },
                &[seeds],
            ),
            amount,
        )
    }

    pub fn claim_whale_locked(ctx: Context<ClaimWhaleLocked>) -> Result<()> {
        let clock = Clock::get()?;
        let now = clock.unix_timestamp;
        require!(
            !is_flag_active(&ctx.accounts.flag.to_account_info(), now, true)?,
            ErrorCode::WalletFlagged
        );
        let mint_key = ctx.accounts.launch.mint;
        let bump = ctx.accounts.launch.bump;
        if ctx.accounts.launch.graduated {
            let after = ctx
                .accounts
                .launch
                .graduated_at
                .checked_add(POST_GRAD_LOCK_SECS)
                .ok_or(ErrorCode::MathOverflow)?;
            require!(now >= after, ErrorCode::StillLocked);
        }

        let pos = &mut ctx.accounts.position;
        require!(pos.whale_total > pos.whale_released, ErrorCode::NoWhaleLock);
        let duration = pos
            .whale_end
            .checked_sub(pos.whale_start)
            .ok_or(ErrorCode::MathOverflow)?
            .max(1);
        let vested: u128 = if now >= pos.whale_end {
            pos.whale_total as u128
        } else {
            let passed = now.saturating_sub(pos.whale_start).max(0);
            (pos.whale_total as u128)
                .checked_mul(passed as u128)
                .ok_or(ErrorCode::MathOverflow)?
                / (duration as u128)
        };
        let amount = to_u64(vested.saturating_sub(pos.whale_released as u128))?;
        require!(amount > 0, ErrorCode::StillLocked);
        pos.whale_released = pos
            .whale_released
            .checked_add(amount)
            .ok_or(ErrorCode::MathOverflow)?;
        let launch_total = ctx
            .accounts
            .launch
            .whale_total
            .checked_sub(amount)
            .ok_or(ErrorCode::MathOverflow)?;
        ctx.accounts.launch.whale_total = launch_total;

        let seeds: &[&[u8]] = &[b"launch", mint_key.as_ref(), &[bump]];
        token::transfer(
            CpiContext::new_with_signer(
                ctx.accounts.token_program.to_account_info(),
                SplTransfer {
                    from: ctx.accounts.escrow.to_account_info(),
                    to: ctx.accounts.owner_ata.to_account_info(),
                    authority: ctx.accounts.launch.to_account_info(),
                },
                &[seeds],
            ),
            amount,
        )?;
        emit!(WhaleClaimed {
            launch: ctx.accounts.launch.key(),
            wallet: ctx.accounts.owner.key(),
            amount,
            launch_total_locked: launch_total,
            slot: clock.slot,
        });
        Ok(())
    }

    pub fn flag_wallet(ctx: Context<FlagWallet>) -> Result<()> {
        let now = Clock::get()?.unix_timestamp;
        let f = &mut ctx.accounts.flag;
        f.wallet = ctx.accounts.wallet.key();
        if !f.permanent {
            f.flagged_at = now;
        }
        emit!(WalletFlagged {
            wallet: ctx.accounts.wallet.key(),
            admin: ctx.accounts.signer.key(),
            slot: Clock::get()?.slot,
        });
        Ok(())
    }

    pub fn flag_permanent(ctx: Context<FlagPermanent>) -> Result<()> {
        let now = Clock::get()?.unix_timestamp;
        let f = &mut ctx.accounts.flag;
        f.wallet = ctx.accounts.wallet.key();
        if !f.permanent {
            f.flagged_at = now;
            f.permanent = true;
        }
        emit!(WalletFlaggedPermanent {
            wallet: ctx.accounts.wallet.key(),
            admin: ctx.accounts.admin.key(),
            slot: Clock::get()?.slot,
        });
        Ok(())
    }

    pub fn clear_expired_flag(ctx: Context<ClearExpiredFlag>) -> Result<()> {
        let f = &ctx.accounts.flag;
        require!(!f.permanent, ErrorCode::FlagNotExpired);
        let now = Clock::get()?.unix_timestamp;
        let until = f
            .flagged_at
            .checked_add(FLAG_LIFETIME_SECS)
            .ok_or(ErrorCode::MathOverflow)?;
        require!(now >= until, ErrorCode::FlagNotExpired);
        Ok(())
    }

    pub fn unflag_wallet(ctx: Context<UnflagWallet>) -> Result<()> {
        emit!(WalletUnflagged {
            wallet: ctx.accounts.flag.wallet,
            admin: ctx.accounts.admin.key(),
            slot: Clock::get()?.slot,
        });
        Ok(())
    }

    pub fn propose_slash_bond(ctx: Context<AdminLaunch>) -> Result<()> {
        let clock = Clock::get()?;
        let launch_key = ctx.accounts.launch.key();
        let l = &mut ctx.accounts.launch;
        require!(!l.bond_slashed && !l.bond_claimed, ErrorCode::BondGone);
        require!(!l.migrated, ErrorCode::Migrated);
        let ready_at = clock
            .unix_timestamp
            .checked_add(SLASH_DELAY_SECS)
            .ok_or(ErrorCode::MathOverflow)?;
        l.slash_ready_at = ready_at;
        emit!(SlashProposed {
            launch: launch_key,
            admin: ctx.accounts.admin.key(),
            ready_at,
            slot: clock.slot,
        });
        Ok(())
    }

    pub fn cancel_slash_bond(ctx: Context<AdminLaunch>) -> Result<()> {
        ctx.accounts.launch.slash_ready_at = 0;
        Ok(())
    }

    pub fn slash_bond(ctx: Context<SlashBond>) -> Result<()> {
        let amount = ctx.accounts.launch.bond_amount;
        let l = &mut ctx.accounts.launch;
        require!(!l.bond_slashed && !l.bond_claimed, ErrorCode::BondGone);
        require!(!l.migrated, ErrorCode::Migrated);
        let now = Clock::get()?.unix_timestamp;
        require!(
            l.slash_ready_at > 0 && now >= l.slash_ready_at,
            ErrorCode::TimelockActive
        );
        let release = l
            .created_at
            .checked_add(l.bond_release_secs)
            .ok_or(ErrorCode::MathOverflow)?;
        require!(now < release, ErrorCode::StillLocked);
        l.bond_slashed = true;
        l.slash_ready_at = 0;
        l.real_sol = l
            .real_sol
            .checked_add(amount)
            .ok_or(ErrorCode::MathOverflow)?;

        let launch_key = l.key();
        let seeds: &[&[u8]] = &[b"bond", launch_key.as_ref(), &[ctx.bumps.bond_vault]];
        transfer(
            CpiContext::new_with_signer(
                ctx.accounts.system_program.to_account_info(),
                Transfer {
                    from: ctx.accounts.bond_vault.to_account_info(),
                    to: ctx.accounts.sol_vault.to_account_info(),
                },
                &[seeds],
            ),
            amount,
        )?;
        emit!(BondSlashed {
            launch: launch_key,
            admin: ctx.accounts.admin.key(),
            amount,
            slot: Clock::get()?.slot,
        });
        Ok(())
    }

    pub fn claim_bond(ctx: Context<ClaimBond>) -> Result<()> {
        let now = Clock::get()?.unix_timestamp;
        let amount = ctx.accounts.launch.bond_amount;
        let l = &mut ctx.accounts.launch;
        require!(!l.bond_slashed && !l.bond_claimed, ErrorCode::BondGone);
        let ready = l
            .created_at
            .checked_add(l.bond_release_secs)
            .ok_or(ErrorCode::MathOverflow)?;
        require!(now >= ready, ErrorCode::StillLocked);
        l.bond_claimed = true;

        let launch_key = l.key();
        let seeds: &[&[u8]] = &[b"bond", launch_key.as_ref(), &[ctx.bumps.bond_vault]];
        transfer(
            CpiContext::new_with_signer(
                ctx.accounts.system_program.to_account_info(),
                Transfer {
                    from: ctx.accounts.bond_vault.to_account_info(),
                    to: ctx.accounts.creator.to_account_info(),
                },
                &[seeds],
            ),
            amount,
        )
    }
}

#[inline(never)]
fn cpi_create_pool<'info>(
    a: &Migrate<'info>,
    liquidity: u128,
    sqrt_price: u128,
    mig_seeds: &[&[u8]],
) -> Result<()> {
    let mint_key = a.mint.key();
    let wsol_key = native_mint::ID;
    let pool_key = a.pool.key();

    let mut data = Vec::with_capacity(8 + 27 + 4 + 16 + 16 + 1 + 16 + 16 + 3);
    data.extend_from_slice(&DISC_INIT_CUSTOMIZABLE_POOL);
    data.extend_from_slice(&POOL_FEE_START_NUM.to_le_bytes());
    data.extend_from_slice(&POOL_FEE_STEPS.to_le_bytes());
    data.extend_from_slice(&POOL_FEE_STEP_SECS.to_le_bytes());
    data.extend_from_slice(&POOL_FEE_STEP_DROP.to_le_bytes());
    data.push(0u8);
    data.extend_from_slice(&POOL_COMPOUNDING_FEE_BPS.to_le_bytes());
    data.push(0u8);
    data.push(0u8);
    data.extend_from_slice(&0u128.to_le_bytes());
    data.extend_from_slice(&u128::MAX.to_le_bytes());
    data.push(0u8);
    data.extend_from_slice(&liquidity.to_le_bytes());
    data.extend_from_slice(&sqrt_price.to_le_bytes());
    data.push(1u8);
    data.push(2u8);
    data.push(0u8);

    let mig_info = a.mig.to_account_info();
    let ix = Instruction {
        program_id: DAMM_ID,
        accounts: vec![
            AccountMeta::new_readonly(*mig_info.key, false),
            AccountMeta::new(a.position_nft_mint.key(), true),
            AccountMeta::new(a.position_nft_account.key(), false),
            AccountMeta::new(*mig_info.key, true),
            AccountMeta::new_readonly(a.pool_authority.key(), false),
            AccountMeta::new(pool_key, false),
            AccountMeta::new(a.position.key(), false),
            AccountMeta::new_readonly(mint_key, false),
            AccountMeta::new_readonly(wsol_key, false),
            AccountMeta::new(a.dex_token_a_vault.key(), false),
            AccountMeta::new(a.dex_token_b_vault.key(), false),
            AccountMeta::new(a.mig_token_a.key(), false),
            AccountMeta::new(a.mig_token_b.key(), false),
            AccountMeta::new_readonly(a.token_program.key(), false),
            AccountMeta::new_readonly(a.token_program.key(), false),
            AccountMeta::new_readonly(a.token_2022_program.key(), false),
            AccountMeta::new_readonly(a.system_program.key(), false),
            AccountMeta::new_readonly(a.event_authority.key(), false),
            AccountMeta::new_readonly(DAMM_ID, false),
        ],
        data,
    };
    invoke_signed(
        &ix,
        &[
            mig_info.clone(),
            a.position_nft_mint.to_account_info(),
            a.position_nft_account.to_account_info(),
            mig_info.clone(),
            a.pool_authority.to_account_info(),
            a.pool.to_account_info(),
            a.position.to_account_info(),
            a.mint.to_account_info(),
            a.wsol_mint.to_account_info(),
            a.dex_token_a_vault.to_account_info(),
            a.dex_token_b_vault.to_account_info(),
            a.mig_token_a.to_account_info(),
            a.mig_token_b.to_account_info(),
            a.token_program.to_account_info(),
            a.token_program.to_account_info(),
            a.token_2022_program.to_account_info(),
            a.system_program.to_account_info(),
            a.event_authority.to_account_info(),
            a.dex_program.to_account_info(),
        ],
        &[mig_seeds],
    )?;
    Ok(())
}

#[inline(never)]
fn cpi_lock_position<'info>(
    a: &Migrate<'info>,
    locked_liquidity: u128,
    mig_seeds: &[&[u8]],
) -> Result<()> {
    let mig_info = a.mig.to_account_info();
    let mut data = Vec::with_capacity(8 + 16);
    data.extend_from_slice(&DISC_PERMANENT_LOCK_POSITION);
    data.extend_from_slice(&locked_liquidity.to_le_bytes());
    let ix = Instruction {
        program_id: DAMM_ID,
        accounts: vec![
            AccountMeta::new(a.pool.key(), false),
            AccountMeta::new(a.position.key(), false),
            AccountMeta::new_readonly(a.position_nft_account.key(), false),
            AccountMeta::new_readonly(*mig_info.key, true),
            AccountMeta::new_readonly(a.event_authority.key(), false),
            AccountMeta::new_readonly(DAMM_ID, false),
        ],
        data,
    };
    invoke_signed(
        &ix,
        &[
            a.pool.to_account_info(),
            a.position.to_account_info(),
            a.position_nft_account.to_account_info(),
            mig_info.clone(),
            a.event_authority.to_account_info(),
            a.dex_program.to_account_info(),
        ],
        &[mig_seeds],
    )?;
    Ok(())
}

#[inline(never)]
fn cpi_claim_fee<'info>(a: &ClaimPoolFees<'info>, mig_seeds: &[&[u8]]) -> Result<()> {
    let mig_info = a.mig.to_account_info();
    let ix = Instruction {
        program_id: DAMM_ID,
        accounts: vec![
            AccountMeta::new_readonly(a.pool_authority.key(), false),
            AccountMeta::new_readonly(a.pool.key(), false),
            AccountMeta::new(a.position.key(), false),
            AccountMeta::new(a.mig_token_a.key(), false),
            AccountMeta::new(a.mig_token_b.key(), false),
            AccountMeta::new(a.dex_token_a_vault.key(), false),
            AccountMeta::new(a.dex_token_b_vault.key(), false),
            AccountMeta::new_readonly(a.mint.key(), false),
            AccountMeta::new_readonly(a.wsol_mint.key(), false),
            AccountMeta::new_readonly(a.position_nft_account.key(), false),
            AccountMeta::new_readonly(*mig_info.key, true),
            AccountMeta::new_readonly(a.token_program.key(), false),
            AccountMeta::new_readonly(a.token_program.key(), false),
            AccountMeta::new_readonly(a.event_authority.key(), false),
            AccountMeta::new_readonly(DAMM_ID, false),
        ],
        data: DISC_CLAIM_POSITION_FEE.to_vec(),
    };
    invoke_signed(
        &ix,
        &[
            a.pool_authority.to_account_info(),
            a.pool.to_account_info(),
            a.position.to_account_info(),
            a.mig_token_a.to_account_info(),
            a.mig_token_b.to_account_info(),
            a.dex_token_a_vault.to_account_info(),
            a.dex_token_b_vault.to_account_info(),
            a.mint.to_account_info(),
            a.wsol_mint.to_account_info(),
            a.position_nft_account.to_account_info(),
            mig_info.clone(),
            a.token_program.to_account_info(),
            a.token_program.to_account_info(),
            a.event_authority.to_account_info(),
            a.dex_program.to_account_info(),
        ],
        &[mig_seeds],
    )?;
    Ok(())
}

fn current_fee_bps(cfg: &Config, elapsed: i64) -> Result<u64> {
    if elapsed >= cfg.early_window_secs {
        return Ok(cfg.end_fee_bps);
    }
    let span = cfg
        .start_fee_bps
        .checked_sub(cfg.end_fee_bps)
        .ok_or(ErrorCode::MathOverflow)?;
    let drop = span
        .checked_mul(elapsed.max(0) as u64)
        .ok_or(ErrorCode::MathOverflow)?
        / (cfg.early_window_secs as u64);
    Ok(cfg.start_fee_bps - drop)
}

#[event]
pub struct LaunchCreated {
    pub launch: Pubkey,
    pub mint: Pubkey,
    pub creator: Pubkey,
    pub slot: u64,
}

#[event]
pub struct Traded {
    pub launch: Pubkey,
    pub trader: Pubkey,
    pub is_buy: bool,
    pub sol_amount: u64,
    pub token_amount: u64,
    pub virtual_sol: u64,
    pub virtual_tokens: u64,
    pub ts: i64,
}

#[event]
pub struct Graduated {
    pub launch: Pubkey,
    pub remaining_tokens: u64,
    pub sol: u64,
    pub ts: i64,
}

#[event]
pub struct Migrated {
    pub launch: Pubkey,
    pub pool: Pubkey,
    pub position: Pubkey,
    pub token_amount: u64,
    pub sol_amount: u64,
    pub liquidity: u128,
    pub ts: i64,
}

#[event]
pub struct ConfigUpdated {
    pub admin: Pubkey,
    pub slot: u64,
}

#[event]
pub struct PausedChanged {
    pub by: Pubkey,
    pub paused: bool,
    pub slot: u64,
}

#[event]
pub struct LaunchPausedChanged {
    pub launch: Pubkey,
    pub by: Pubkey,
    pub paused: bool,
    pub slot: u64,
}

#[event]
pub struct GuardianSet {
    pub admin: Pubkey,
    pub guardian: Pubkey,
    pub slot: u64,
}

#[event]
pub struct FlaggerSet {
    pub admin: Pubkey,
    pub flagger: Pubkey,
    pub slot: u64,
}

#[event]
pub struct AdminProposed {
    pub admin: Pubkey,
    pub pending_admin: Pubkey,
    pub slot: u64,
}

#[event]
pub struct AdminChanged {
    pub old_admin: Pubkey,
    pub new_admin: Pubkey,
    pub slot: u64,
}

#[event]
pub struct FeesWithdrawn {
    pub admin: Pubkey,
    pub amount: u64,
    pub slot: u64,
}

#[event]
pub struct WalletFlagged {
    pub wallet: Pubkey,
    pub admin: Pubkey,
    pub slot: u64,
}

#[event]
pub struct WalletFlaggedPermanent {
    pub wallet: Pubkey,
    pub admin: Pubkey,
    pub slot: u64,
}

#[event]
pub struct WalletUnflagged {
    pub wallet: Pubkey,
    pub admin: Pubkey,
    pub slot: u64,
}

#[event]
pub struct BondSlashed {
    pub launch: Pubkey,
    pub admin: Pubkey,
    pub amount: u64,
    pub slot: u64,
}

#[event]
pub struct PoolFeesClaimed {
    pub launch: Pubkey,
    pub amount: u64,
    pub platform_part: u64,
    pub creator_part: u64,
    pub slot: u64,
}

#[event]
pub struct ConfigProposed {
    pub admin: Pubkey,
    pub ready_at: i64,
    pub slot: u64,
}

#[event]
pub struct ConfigCancelled {
    pub admin: Pubkey,
    pub slot: u64,
}

#[event]
pub struct GraduationPaid {
    pub launch: Pubkey,
    pub creator_reward: u64,
    pub platform_fee: u64,
    pub slot: u64,
}

#[event]
pub struct WhaleLocked {
    pub launch: Pubkey,
    pub wallet: Pubkey,
    pub amount: u64,
    pub wallet_total_locked: u64,
    pub launch_total_locked: u64,
    pub release_end: i64,
    pub slot: u64,
}

#[event]
pub struct WhaleClaimed {
    pub launch: Pubkey,
    pub wallet: Pubkey,
    pub amount: u64,
    pub launch_total_locked: u64,
    pub slot: u64,
}

#[event]
pub struct GuardianPaused {
    pub launch: Pubkey,
    pub by: Pubkey,
    pub until: i64,
    pub slot: u64,
}

#[event]
pub struct SlashProposed {
    pub launch: Pubkey,
    pub admin: Pubkey,
    pub ready_at: i64,
    pub slot: u64,
}

#[account]
#[derive(InitSpace)]
pub struct Config {
    pub admin: Pubkey,
    pub pending_admin: Pubkey,
    pub guardian: Pubkey,
    pub flagger: Pubkey,
    pub bond_lamports: u64,
    pub bond_release_secs: i64,
    pub early_window_secs: i64,
    pub early_lock_secs: i64,
    pub creator_vest_secs: i64,
    pub max_early_buy_lamports: u64,
    pub start_fee_bps: u64,
    pub end_fee_bps: u64,
    pub owner_fee_bps: u64,
    pub creator_fee_bps: u64,
    pub max_wallet_bps: u64,
    pub max_slot_buy_lamports: u64,
    pub owner_fees_owed: u64,
    pub paused: bool,
    pub pending: ConfigParams,
    pub pending_ready_at: i64,
}

#[derive(AnchorSerialize, AnchorDeserialize, Clone, InitSpace)]
pub struct ConfigParams {
    pub bond_lamports: u64,
    pub bond_release_secs: i64,
    pub early_window_secs: i64,
    pub early_lock_secs: i64,
    pub creator_vest_secs: i64,
    pub max_early_buy_lamports: u64,
    pub start_fee_bps: u64,
    pub end_fee_bps: u64,
    pub owner_fee_bps: u64,
    pub creator_fee_bps: u64,
    pub max_wallet_bps: u64,
    pub max_slot_buy_lamports: u64,
}

#[account]
#[derive(InitSpace)]
pub struct Launch {
    pub creator: Pubkey,
    pub mint: Pubkey,
    pub created_at: i64,
    pub virtual_sol: u64,
    pub virtual_tokens: u64,
    pub real_sol: u64,
    pub real_tokens: u64,
    pub early_total: u64,
    pub created_slot: u64,
    pub creator_fees_owed: u64,
    pub bond_amount: u64,
    pub bond_slashed: bool,
    pub bond_claimed: bool,
    pub bump: u8,
    pub graduated: bool,
    pub graduated_at: i64,
    pub migrated: bool,
    pub pool: Pubkey,
    pub early_lock_secs: i64,
    pub creator_vest_secs: i64,
    pub bond_release_secs: i64,
    pub paused: bool,
    pub last_slot: u64,
    pub slot_spent: u64,
    pub position: Pubkey,
    pub position_nft_account: Pubkey,
    pub last_sell_slot: u64,
    pub slot_sold: u64,
    pub whale_total: u64,
    pub paused_until: i64,
    pub guardian_used: bool,
    pub cap_slot: u64,
    pub cap_slot_vsol: u64,
    pub cap_slot_net: u64,
    pub buy_win_start: i64,
    pub buy_win_vsol: u64,
    pub buy_win_net: u64,
    pub sell_win_start: i64,
    pub sell_win_sold: u64,
    pub sell_win_vt: u64,
    pub slash_ready_at: i64,
    pub _reserved: [u8; 8],
}

#[account]
#[derive(InitSpace)]
pub struct Position {
    pub owner: Pubkey,
    pub launch: Pubkey,
    pub early_spent: u64,
    pub locked_amount: u64,
    pub unlock_at: i64,
    pub released: u64,
    pub whale_total: u64,
    pub whale_released: u64,
    pub whale_start: i64,
    pub whale_end: i64,
    pub last_buy_at: i64,
    pub bought_total: u64,
    pub sold_total: u64,
    pub last_sell_slot: u64, // per-wallet sell cooldown (taken from _reserved)
    pub _reserved: [u8; 8],  // was 16, total account size unchanged
}

#[account]
#[derive(InitSpace)]
pub struct Flag {
    pub wallet: Pubkey,
    pub flagged_at: i64,
    pub permanent: bool,
}

#[derive(Accounts)]
pub struct InitConfig<'info> {
    #[account(init, payer = admin, space = 8 + Config::INIT_SPACE, seeds = [b"config"], bump)]
    pub config: Account<'info, Config>,
    #[account(mut)]
    pub admin: Signer<'info>,
    pub system_program: Program<'info, System>,
}

#[derive(Accounts)]
pub struct AdminOnly<'info> {
    #[account(mut, seeds = [b"config"], bump, has_one = admin)]
    pub config: Account<'info, Config>,
    #[account(mut)]
    pub admin: Signer<'info>,
}

#[derive(Accounts)]
pub struct AdminLaunch<'info> {
    #[account(seeds = [b"config"], bump, has_one = admin)]
    pub config: Account<'info, Config>,
    #[account(mut, seeds = [b"launch", launch.mint.as_ref()], bump = launch.bump)]
    pub launch: Account<'info, Launch>,
    pub admin: Signer<'info>,
}

#[derive(Accounts)]
pub struct GuardianPause<'info> {
    #[account(seeds = [b"config"], bump,
        constraint = (signer.key() == config.admin
            || (config.guardian != Pubkey::default() && signer.key() == config.guardian))
            @ ErrorCode::NotGuardian)]
    pub config: Account<'info, Config>,
    #[account(mut, seeds = [b"launch", launch.mint.as_ref()], bump = launch.bump)]
    pub launch: Account<'info, Launch>,
    pub signer: Signer<'info>,
}

#[derive(Accounts)]
pub struct AcceptAdmin<'info> {
    #[account(mut, seeds = [b"config"], bump,
        constraint = config.pending_admin == new_admin.key() @ ErrorCode::NotPendingAdmin)]
    pub config: Account<'info, Config>,
    pub new_admin: Signer<'info>,
}

#[derive(Accounts)]
pub struct ClaimCreatorFees<'info> {
    #[account(mut, seeds = [b"config"], bump)]
    pub config: Account<'info, Config>,
    #[account(mut, seeds = [b"launch", launch.mint.as_ref()], bump = launch.bump,
        has_one = creator)]
    pub launch: Account<'info, Launch>,
    #[account(mut)]
    pub creator: Signer<'info>,
}

#[derive(Accounts)]
pub struct CreateLaunch<'info> {
    #[account(seeds = [b"config"], bump)]
    pub config: Box<Account<'info, Config>>,
    #[account(init, payer = creator, space = 8 + Launch::INIT_SPACE,
        seeds = [b"launch", mint.key().as_ref()], bump)]
    pub launch: Box<Account<'info, Launch>>,
    #[account(init, payer = creator, mint::decimals = 6, mint::authority = launch)]
    pub mint: Box<Account<'info, Mint>>,
    #[account(init, payer = creator, token::mint = mint, token::authority = launch,
        seeds = [b"vault", launch.key().as_ref()], bump)]
    pub token_vault: Box<Account<'info, TokenAccount>>,
    #[account(init, payer = creator, token::mint = mint, token::authority = launch,
        seeds = [b"escrow", launch.key().as_ref()], bump)]
    pub escrow: Box<Account<'info, TokenAccount>>,
    #[account(mut, seeds = [b"bond", launch.key().as_ref()], bump)]
    pub bond_vault: SystemAccount<'info>,
    /// CHECK: created by Metaplex
    #[account(mut, seeds = [b"metadata", token_metadata_program.key().as_ref(), mint.key().as_ref()],
        bump, seeds::program = token_metadata_program.key())]
    pub metadata: UncheckedAccount<'info>,
    #[account(mut)]
    pub creator: Signer<'info>,
    pub token_metadata_program: Program<'info, Metadata>,
    pub token_program: Program<'info, Token>,
    pub system_program: Program<'info, System>,
    pub rent: Sysvar<'info, Rent>,
}

#[derive(Accounts)]
pub struct Buy<'info> {
    #[account(mut, seeds = [b"config"], bump)]
    pub config: Box<Account<'info, Config>>,
    #[account(mut, seeds = [b"launch", launch.mint.as_ref()], bump = launch.bump)]
    pub launch: Box<Account<'info, Launch>>,
    #[account(init_if_needed, payer = buyer, space = 8 + Position::INIT_SPACE,
        seeds = [b"pos", launch.key().as_ref(), buyer.key().as_ref()], bump)]
    pub position: Box<Account<'info, Position>>,
    /// CHECK: must not be flagged
    #[account(seeds = [b"flag", buyer.key().as_ref()], bump)]
    pub flag: UncheckedAccount<'info>,
    #[account(mut, seeds = [b"vault", launch.key().as_ref()], bump)]
    pub token_vault: Box<Account<'info, TokenAccount>>,
    #[account(mut, seeds = [b"escrow", launch.key().as_ref()], bump)]
    pub escrow: Box<Account<'info, TokenAccount>>,
    #[account(mut, token::mint = launch.mint, token::authority = buyer)]
    pub buyer_ata: Box<Account<'info, TokenAccount>>,
    #[account(mut, seeds = [b"sol_vault", launch.key().as_ref()], bump)]
    pub sol_vault: SystemAccount<'info>,
    #[account(mut)]
    pub buyer: Signer<'info>,
    pub token_program: Program<'info, Token>,
    pub system_program: Program<'info, System>,
}

#[derive(Accounts)]
pub struct Sell<'info> {
    #[account(mut, seeds = [b"config"], bump)]
    pub config: Box<Account<'info, Config>>,
    #[account(mut, seeds = [b"launch", launch.mint.as_ref()], bump = launch.bump)]
    pub launch: Box<Account<'info, Launch>>,
    #[account(mut, seeds = [b"vault", launch.key().as_ref()], bump)]
    pub token_vault: Box<Account<'info, TokenAccount>>,
    #[account(mut, token::mint = launch.mint, token::authority = seller)]
    pub seller_ata: Box<Account<'info, TokenAccount>>,
    #[account(mut, seeds = [b"sol_vault", launch.key().as_ref()], bump)]
    pub sol_vault: SystemAccount<'info>,
    #[account(mut, seeds = [b"pos", launch.key().as_ref(), seller.key().as_ref()], bump,
        constraint = position.owner == seller.key())]
    pub position: Box<Account<'info, Position>>,
    #[account(mut)]
    pub seller: Signer<'info>,
    pub token_program: Program<'info, Token>,
    pub system_program: Program<'info, System>,
}

#[derive(Accounts)]
pub struct ClaimLocked<'info> {
    #[account(seeds = [b"launch", launch.mint.as_ref()], bump = launch.bump)]
    pub launch: Box<Account<'info, Launch>>,
    #[account(mut, seeds = [b"pos", launch.key().as_ref(), owner.key().as_ref()], bump,
        has_one = owner)]
    pub position: Box<Account<'info, Position>>,
    #[account(mut, seeds = [b"escrow", launch.key().as_ref()], bump)]
    pub escrow: Box<Account<'info, TokenAccount>>,
    #[account(mut, token::mint = launch.mint, token::authority = owner)]
    pub owner_ata: Box<Account<'info, TokenAccount>>,
    /// CHECK: checked by helper
    #[account(seeds = [b"flag", owner.key().as_ref()], bump)]
    pub flag: UncheckedAccount<'info>,
    pub owner: Signer<'info>,
    pub token_program: Program<'info, Token>,
}

#[derive(Accounts)]
pub struct ClaimWhaleLocked<'info> {
    #[account(mut, seeds = [b"launch", launch.mint.as_ref()], bump = launch.bump)]
    pub launch: Box<Account<'info, Launch>>,
    #[account(mut, seeds = [b"pos", launch.key().as_ref(), owner.key().as_ref()], bump,
        has_one = owner)]
    pub position: Box<Account<'info, Position>>,
    #[account(mut, seeds = [b"escrow", launch.key().as_ref()], bump)]
    pub escrow: Box<Account<'info, TokenAccount>>,
    #[account(mut, token::mint = launch.mint, token::authority = owner)]
    pub owner_ata: Box<Account<'info, TokenAccount>>,
    /// CHECK: checked by helper
    #[account(seeds = [b"flag", owner.key().as_ref()], bump)]
    pub flag: UncheckedAccount<'info>,
    pub owner: Signer<'info>,
    pub token_program: Program<'info, Token>,
}

#[derive(Accounts)]
pub struct FlagWallet<'info> {
    #[account(seeds = [b"config"], bump,
        constraint = (signer.key() == config.admin
            || (config.flagger != Pubkey::default() && signer.key() == config.flagger))
            @ ErrorCode::NotFlagger)]
    pub config: Account<'info, Config>,
    /// CHECK: wallet flagged
    pub wallet: UncheckedAccount<'info>,
    #[account(init_if_needed, payer = signer, space = 8 + Flag::INIT_SPACE,
        seeds = [b"flag", wallet.key().as_ref()], bump)]
    pub flag: Account<'info, Flag>,
    #[account(mut)]
    pub signer: Signer<'info>,
    pub system_program: Program<'info, System>,
}

#[derive(Accounts)]
pub struct FlagPermanent<'info> {
    #[account(seeds = [b"config"], bump, has_one = admin)]
    pub config: Account<'info, Config>,
    /// CHECK: wallet flagged
    pub wallet: UncheckedAccount<'info>,
    #[account(init_if_needed, payer = admin, space = 8 + Flag::INIT_SPACE,
        seeds = [b"flag", wallet.key().as_ref()], bump)]
    pub flag: Account<'info, Flag>,
    #[account(mut)]
    pub admin: Signer<'info>,
    pub system_program: Program<'info, System>,
}

#[derive(Accounts)]
pub struct ClearExpiredFlag<'info> {
    #[account(mut, close = caller, seeds = [b"flag", flag.wallet.as_ref()], bump)]
    pub flag: Account<'info, Flag>,
    #[account(mut)]
    pub caller: Signer<'info>,
}

#[derive(Accounts)]
pub struct UnflagWallet<'info> {
    #[account(seeds = [b"config"], bump, has_one = admin)]
    pub config: Account<'info, Config>,
    #[account(mut, close = admin, seeds = [b"flag", flag.wallet.as_ref()], bump)]
    pub flag: Account<'info, Flag>,
    #[account(mut)]
    pub admin: Signer<'info>,
}

#[derive(Accounts)]
pub struct SlashBond<'info> {
    #[account(seeds = [b"config"], bump, has_one = admin)]
    pub config: Account<'info, Config>,
    #[account(mut, seeds = [b"launch", launch.mint.as_ref()], bump = launch.bump)]
    pub launch: Account<'info, Launch>,
    #[account(mut, seeds = [b"bond", launch.key().as_ref()], bump)]
    pub bond_vault: SystemAccount<'info>,
    #[account(mut, seeds = [b"sol_vault", launch.key().as_ref()], bump)]
    pub sol_vault: SystemAccount<'info>,
    pub admin: Signer<'info>,
    pub system_program: Program<'info, System>,
}

#[derive(Accounts)]
pub struct ClaimBond<'info> {
    #[account(seeds = [b"config"], bump)]
    pub config: Account<'info, Config>,
    #[account(mut, seeds = [b"launch", launch.mint.as_ref()], bump = launch.bump,
        has_one = creator)]
    pub launch: Account<'info, Launch>,
    #[account(mut, seeds = [b"bond", launch.key().as_ref()], bump)]
    pub bond_vault: SystemAccount<'info>,
    #[account(mut)]
    pub creator: Signer<'info>,
    pub system_program: Program<'info, System>,
}

#[derive(Accounts)]
pub struct Migrate<'info> {
    #[account(mut, seeds = [b"config"], bump)]
    pub config: Box<Account<'info, Config>>,
    #[account(mut, seeds = [b"launch", launch.mint.as_ref()], bump = launch.bump)]
    pub launch: Box<Account<'info, Launch>>,
    #[account(address = launch.mint)]
    pub mint: Box<Account<'info, Mint>>,
    #[account(address = native_mint::ID)]
    pub wsol_mint: Box<Account<'info, Mint>>,
    #[account(mut, seeds = [b"vault", launch.key().as_ref()], bump)]
    pub token_vault: Box<Account<'info, TokenAccount>>,
    #[account(mut, seeds = [b"sol_vault", launch.key().as_ref()], bump)]
    pub sol_vault: SystemAccount<'info>,
    #[account(mut, seeds = [b"mig", launch.key().as_ref()], bump)]
    pub mig: SystemAccount<'info>,
    #[account(init, payer = caller, token::mint = mint, token::authority = mig,
        seeds = [b"mig_a", launch.key().as_ref()], bump)]
    pub mig_token_a: Box<Account<'info, TokenAccount>>,
    #[account(init, payer = caller, token::mint = wsol_mint, token::authority = mig,
        seeds = [b"mig_b", launch.key().as_ref()], bump)]
    pub mig_token_b: Box<Account<'info, TokenAccount>>,
    #[account(mut)]
    pub caller: Signer<'info>,
    #[account(mut)]
    pub position_nft_mint: Signer<'info>,
    /// CHECK: verified in handler
    #[account(mut)]
    pub position_nft_account: UncheckedAccount<'info>,
    /// CHECK: verified in handler
    #[account(mut)]
    pub pool: UncheckedAccount<'info>,
    /// CHECK: verified in handler
    #[account(mut)]
    pub position: UncheckedAccount<'info>,
    /// CHECK: verified in handler
    #[account(mut)]
    pub dex_token_a_vault: UncheckedAccount<'info>,
    /// CHECK: verified in handler
    #[account(mut)]
    pub dex_token_b_vault: UncheckedAccount<'info>,
    /// CHECK: verified in handler
    pub pool_authority: UncheckedAccount<'info>,
    /// CHECK: verified in handler
    pub event_authority: UncheckedAccount<'info>,
    /// CHECK: verified in handler
    #[account(address = DAMM_ID, executable)]
    pub dex_program: UncheckedAccount<'info>,
    /// CHECK: verified in handler
    #[account(address = TOKEN_2022_ID, executable)]
    pub token_2022_program: UncheckedAccount<'info>,
    pub token_program: Program<'info, Token>,
    pub system_program: Program<'info, System>,
}

#[derive(Accounts)]
pub struct ClaimPoolFees<'info> {
    #[account(mut, seeds = [b"config"], bump)]
    pub config: Box<Account<'info, Config>>,
    #[account(mut, seeds = [b"launch", launch.mint.as_ref()], bump = launch.bump)]
    pub launch: Box<Account<'info, Launch>>,
    #[account(address = launch.mint)]
    pub mint: Box<Account<'info, Mint>>,
    #[account(address = native_mint::ID)]
    pub wsol_mint: Box<Account<'info, Mint>>,
    #[account(mut, seeds = [b"mig", launch.key().as_ref()], bump)]
    pub mig: SystemAccount<'info>,
    #[account(mut, seeds = [b"mig_a", launch.key().as_ref()], bump)]
    pub mig_token_a: Box<Account<'info, TokenAccount>>,
    #[account(init_if_needed, payer = caller, token::mint = wsol_mint, token::authority = mig,
        seeds = [b"mig_b", launch.key().as_ref()], bump)]
    pub mig_token_b: Box<Account<'info, TokenAccount>>,
    #[account(mut)]
    pub caller: Signer<'info>,
    /// CHECK: verified in handler
    pub pool: UncheckedAccount<'info>,
    /// CHECK: verified in handler
    #[account(mut)]
    pub position: UncheckedAccount<'info>,
    /// CHECK: verified in handler
    pub position_nft_account: UncheckedAccount<'info>,
    /// CHECK: verified in handler
    #[account(mut)]
    pub dex_token_a_vault: UncheckedAccount<'info>,
    /// CHECK: verified in handler
    #[account(mut)]
    pub dex_token_b_vault: UncheckedAccount<'info>,
    /// CHECK: verified in handler
    pub pool_authority: UncheckedAccount<'info>,
    /// CHECK: verified in handler
    pub event_authority: UncheckedAccount<'info>,
    /// CHECK: verified in handler
    #[account(address = DAMM_ID, executable)]
    pub dex_program: UncheckedAccount<'info>,
    pub token_program: Program<'info, Token>,
    pub system_program: Program<'info, System>,
}

// IMPORTANT: Anchor numbers errors 6000, 6001, ... in the order they are written here.
// NEVER insert or reorder: only add new errors at the END.
#[error_code]
pub enum ErrorCode {
    #[msg("Amount too small")]
    TooSmall,
    #[msg("Slippage exceeded")]
    Slippage,
    #[msg("Curve exhausted")]
    CurveExhausted,
    #[msg("Wallet flagged: blocked from early buys")]
    WalletFlagged,
    #[msg("Early-window buy cap exceeded")]
    EarlyCapExceeded,
    #[msg("Still locked")]
    StillLocked,
    #[msg("Not enough SOL in pool")]
    InsufficientPool,
    #[msg("Bond already slashed or claimed")]
    BondGone,
    #[msg("Too early after launch")]
    TooEarly,
    #[msg("Invalid config")]
    BadConfig,
    #[msg("Math overflow")]
    MathOverflow,
    #[msg("Trading is paused")]
    Paused,
    #[msg("Max wallet limit exceeded")]
    MaxWallet,
    #[msg("Not the pending admin")]
    NotPendingAdmin,
    #[msg("Bad token name, symbol or link")]
    BadMetadata,
    #[msg("Curve is closed: buying has stopped, the pool is being opened")]
    Graduated,
    #[msg("Curve is closed and the pool is open: trade on Meteora")]
    Migrated,
    #[msg("Not graduated yet")]
    NotGraduated,
    #[msg("Already migrated")]
    AlreadyMigrated,
    #[msg("Wrong DEX account passed to migrate")]
    BadDexAccount,
    #[msg("Could not work out the pool opening price")]
    BadPoolPlan,
    #[msg("Only the set init wallet can run init_config")]
    NotInitAuthority,
    #[msg("Not the admin or guardian")]
    NotGuardian,
    #[msg("Buying is paused on this launch")]
    LaunchPaused,
    #[msg("Per-slot buy cap exceeded: too much bought in this slot")]
    SlotCapExceeded,
    #[msg("The creator may only buy during the early window")]
    CreatorBuyWindow,
    #[msg("Creator share cap exceeded")]
    CreatorShareCap,
    #[msg("No config change is waiting")]
    NoPendingConfig,
    #[msg("Config change is still in its waiting period")]
    TimelockActive,
    #[msg("Not migrated yet: there is no pool")]
    NotMigrated,
    #[msg("No pool fees to claim yet")]
    NothingToClaim,
    #[msg("Not the admin or flagger")]
    NotFlagger,
    #[msg("Flag has not expired")]
    FlagNotExpired,
    #[msg("Too many tokens sold in this slot: try again in a moment")]
    SlotSellCapExceeded,
    // ---- added earlier (6034, 6035, 6036) ----
    #[msg("Too many tokens above the free limit are already locked on this launch")]
    WhaleLimit,
    #[msg("The guardian already used its one pause on this launch")]
    GuardianPauseUsed,
    #[msg("No whale-locked tokens to release")]
    NoWhaleLock,
    // ---- appended (6037, 6038) ----
    #[msg("You can only sell tokens this wallet bought. Send received tokens back to the wallet that bought them")]
    NotYourTokens,
    #[msg("Too much boughtA on this launch in the last minute: try again shortly")]
    BuyWindowCapExceeded,
    // ---- HARDENED MEV DEFENSE ERRORS (appended at end: 6039-6042) ----
    #[msg("Strict genesis period: only 1 buy per slot is allowed across this launch")]
    AntiBundleSlotOccupied,
    #[msg("Sell cooldown active: please wait a few seconds before selling again")]
    SellCooldownActive,
    #[msg("Migration blocked while an active bond slash notice is pending")]
    SlashNoticePending,
    #[msg("Rate limit exceeded: excessive pressure on bonding curve")]
    RateLimitExceeded,
}
