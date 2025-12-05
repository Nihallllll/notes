

> [!info] About This Guide

> A comprehensive guide to mastering Solana blockchain concepts for interviews and production development

  

---

  

## 📚 Table of Contents

  

> [!abstract]- Quick Navigation

> 1. [[#🏗️ 1. SOLANA ARCHITECTURE & FUNDAMENTALS|Solana Architecture & Fundamentals]]

> 2. [[#💾 2. ACCOUNTS & PROGRAM MODEL|Accounts & Program Model]]

> 3. [[#🔐 3. PROGRAM DERIVED ADDRESSES (PDAs)|Program Derived Addresses (PDAs)]]

> 4. [[#💰 4. RENT & LAMPORTS|Rent & Lamports]]

> 5. [[#⚡ 5. THE SEALEVEL RUNTIME|The Sealevel Runtime]]

> 6. [[#📦 6. TRANSACTIONS, INSTRUCTIONS & COMPOSABILITY|Transactions, Instructions & Composability]]

> 7. [[#🔍 7. SYSVARS (SYSTEM VARIABLES)|Sysvars (System Variables)]]

> 8. [[#🧮 8. COMPUTE UNITS & RESOURCE CONSTRAINTS|Compute Units & Resource Constraints]]

> 9. [[#🧠 9. MEMORY MODEL & SECURITY|Memory Model & Security]]

> 10. [[#🔀 10. CROSS-PROGRAM INVOCATION (CPI)|Cross-Program Invocation (CPI)]]

> 11. [[#🎪 11. TOKEN PROGRAM & SPL TOKENS|Token Program & SPL Tokens]]

> 12. [[#💳 12. ASSOCIATED TOKEN ACCOUNTS (ATAs)|Associated Token Accounts (ATAs)]]

> 13. [[#🖼️ 13. METADATA PROGRAM (NFT METADATA)|Metadata Program]]

> 14. [[#🔐 14. SIGNER RULES & AUTHORIZATION|Signer Rules & Authorization]]

> 15. [[#⚓ 15. ANCHOR FRAMEWORK FUNDAMENTALS|Anchor Framework Fundamentals]]

> 16. [[#🛡️ 16. SECURITY PATTERNS & BEST PRACTICES|Security Patterns & Best Practices]]

> 17. [[#🔄 17. PROGRAM UPGRADEABILITY|Program Upgradeability]]

> 18. [[#🚀 18. ADVANCED TOPICS & OPTIMIZATION|Advanced Topics & Optimization]]

  

---

  

## 🏗️ 1. SOLANA ARCHITECTURE & FUNDAMENTALS

  

### 1.1 What Makes Solana Different?

  

> [!tip] Key Concept

> Solana is a **high-performance blockchain** that achieves **65,000+ transactions per second** through parallel processing and a unique timestamp system.

  

**Key architectural innovations:**

  

| Innovation | Purpose | Impact |

|-----------|---------|--------|

| **Proof of History (PoH)** | Cryptographic clock before consensus | Fast finality without heavy consensus overhead |

| **Sealevel Runtime** | Parallel smart contract execution | Multi-threaded state machine vs single-threaded competitors |

| **Explicit Account Dependencies** | Each instruction lists accounts it touches | Runtime can schedule non-overlapping transactions in parallel |

| **Stateless Programs** | Code and data separation | Composability and zero-knowledge friendly |

| **BPF VM** | Rust-based execution environment | Safe, efficient, auditable bytecode |

  

### 1.2 Proof of History (PoH): The Secret Sauce

  

**Problem:** Traditional blockchains need consensus to agree on the order of transactions. Consensus is slow.

  

**Solution:** Create a **verifiable timestamp** BEFORE consensus happens.

  

**How PoH works:**

  

```

Input: Message 1 → SHA256 → Output 1 (includes timestamp)

           ↓

Input: Output 1 + Message 2 → SHA256 → Output 2

           ↓

Input: Output 2 + Message 3 → SHA256 → Output 3

           ...and so on

```

  

**Key insights:**

- The hash chain **proves** message order and timing

- Validators can verify this proof **independently and in parallel**

- If you want to rewrite history, you'd have to recompute the ENTIRE chain (proof-of-work level difficulty)

- Creates a **cryptographic clock** that doesn't require consensus

  

> [!question] Interview Question Context

> This allows Solana to reach ~400ms block times while other chains need seconds/minutes for consensus.

  

### 1.3 The Account Model: Separation of Code and Data

  

**Ethereum model:**

- Smart contracts are both **code** AND **data storage**

- State lives IN the contract

- State changes happen through function calls

  

**Solana model:**

- **Programs** (accounts with `executable=true`) contain CODE ONLY

- **Data accounts** hold all user data separately

- Programs are stateless – they execute logic but don't store results

- Programs modify data accounts they're given permission to touch

  

> [!success] This Enables

> - ✅ Parallel execution (different programs can touch different data)

> - ✅ Composability (data is portable between programs)

> - ✅ Explicitness (you know exactly which state is affected)

  

---

  

## 💾 2. ACCOUNTS & PROGRAM MODEL

  

### 2.1 Anatomy of a Solana Account

  

Every account on Solana has this structure:

  

```rust

pub struct AccountInfo {

    pub lamports: u64,           // SOL balance in lamports

    pub data: Vec<u8>,           // Actual data stored (max 10MB)

    pub owner: Pubkey,           // Which program owns this account

    pub executable: bool,        // Is this a program? (if true = code account)

    pub rent_epoch: Epoch,       // When next rent is owed

}

```

  

**What each field means:**

  

| Field | Meaning | Example |

|-------|---------|---------|

| `lamports` | Amount of SOL (1 SOL = 1 billion lamports) | 5,000,000 lamports = 0.005 SOL |

| `data` | Raw bytes (could be program code OR user data) | Account struct serialized as bytes |

| `owner` | Program that can write to this account | Token Program, Your Program, System Program |

| `executable` | True if account contains program code | True for programs, False for data |

| `rent_epoch` | Epoch when rent was last collected/exemption checked | Used to track rent collection |

  

### 2.2 Types of Accounts

  

**1. System Accounts (wallets)**

- Owned by **System Program**

- Hold SOL

- Can sign transactions

- Cannot have data written to them

  

**2. Program Accounts (smart contracts)**

- Owned by **BPF Loader**

- `executable = true`

- Contains compiled program code

- Data section contains the actual bytecode

- Cannot be modified (immutable after deployment)

  

**3. Data Accounts**

- Owned by **your program** (or another program)

- `executable = false`

- Store user data (counters, balances, metadata)

- Can only be modified by their owner program

- Created by System Program, assigned to your program

  

**4. Sysvar Accounts (special)**

- Read-only system accounts

- Maintained by Solana runtime

- Contain cluster info (clock, rent rates, etc.)

- We'll cover these in detail later

  

### 2.3 Account Ownership & Authorization Rules

  

**Golden Rule:** Only a program can modify an account it owns.

  

```

┌─────────────────────────────────────────┐

│ Account X (owner = Program A)           │

├─────────────────────────────────────────┤

│ ONLY Program A can write to this        │

│ Program B cannot modify it              │

│ Program A can modify if given in instruction │

└─────────────────────────────────────────┘

```

  

**Implications:**

- To modify data, the transaction must invoke the **owning program**

- The owning program decides **when** and **how** data changes

- This is the basis of all **smart contract security**

  

### 2.4 Creating and Closing Accounts

  

**Creating accounts:**

- Only the **System Program** can create accounts

- Your program must **CPI (call) the System Program**

- You must provide: address, owner, space (in bytes)

- You must transfer enough lamports for **rent exemption**

  

**Closing accounts:**

- Your program can close accounts it owns

- Closes mean: set data = empty, set lamports = 0, transfer lamports to beneficiary

- This reclaims storage space and frees up lamports

  

---

  

## 🔐 3. PROGRAM DERIVED ADDRESSES (PDAs)

  

### 3.1 What is a PDA?

  

> [!note] Definition

> A PDA is an account address that is **deterministically derived** from a program ID and optional seeds. Unlike regular keypairs, PDAs have NO private key – only the **program** can sign on their behalf.

  

> [!success] Why PDAs Matter

> - ✅ **Deterministic:** Same seeds + program ID = same address (always)

> - ✅ **Program Authority:** The program can sign transactions as the PDA

> - ✅ **No Private Key:** Can't be compromised (no key to steal)

> - ✅ **Unique Associations:** Create unique accounts per user/entity

  

### 3.2 How PDAs are Derived

  

**Formula:**

```

PDA = SHA256(program_id || seed1 || seed2 || ... || seedN || bump)

      if result is on the ed25519 curve

      else

      try bump - 1, bump - 2, ... until valid

```

  

**The "Bump Seed":**

- A value (0-255) that ensures the derived address is NOT on the ed25519 curve

- Required because PDAs must be **off-curve** (so they have no private key)

- Always use the **canonical bump** (highest valid bump) for security

- Current bump is usually 254 or 253

  

**Process:**

```

1. Try bump = 255 with your seeds

   → Check if address is off-curve

   → If YES, return this PDA and bump

   → If NO, try bump = 254

2. Keep trying lower bumps until you find an off-curve address

3. Return first valid PDA found (canonical bump)

```

  

### 3.3 Common PDA Seeds Patterns

  

**Pattern 1: Per-User Account**

```

Seeds: [b"user_vault", user_pubkey.as_ref()]

Result: Each user gets a unique vault

```

  

**Pattern 2: Collection Items**

```

Seeds: [b"nft", collection_id, item_index]

Result: Each NFT in collection has unique address

```

  

**Pattern 3: Multi-Party Escrow**

```

Seeds: [b"escrow", buyer_pubkey, seller_pubkey, order_id]

Result: Specific agreement between parties

```

  

**Pattern 4: Time-Based**

```

Seeds: [b"reward", epoch, user_pubkey]

Result: Different rewards per epoch per user

```

  

### 3.4 PDA Signing (invoke_signed)

  

**Problem:** PDAs have no private key, so how do they sign transactions?

  

**Answer:** The program invokes with `invoke_signed`, passing the seeds.

  

```rust

// Inside your program

invoke_signed(

    &instruction,

    &account_infos,

    &[&[b"vault", user.key().as_ref(), &[bump]]]  // Seeds + bump

)?;

```

  

**What happens:**

1. Runtime sees `invoke_signed`

2. Runtime checks: "Does PDA derived from these seeds + bump match the signer?"

3. If YES, treat it as signed by that PDA

4. If NO, reject the transaction

  

> [!warning] Security Rule

> Only call `invoke_signed` if:

> - You verified the PDA is correctly derived

> - You verified the bump is canonical

> - Prevents attackers from creating fake PDAs

  

---

  

## 💰 4. RENT & LAMPORTS

  

### 4.1 What is Rent?

  

> [!note] Definition

> Rent is a fee to store data on Solana. Accounts must maintain a minimum balance (2 years of rent) to be "rent-exempt" and never have their lamports deducted.

  

> [!question] Why Does Rent Exist?

> - Validators maintain account data – they need compensation

> - Encourages efficient use of storage (remove unused accounts)

> - Time-based + space-based fee model

  

### 4.2 Rent Calculation

  

**Key constants:**

```

Lamports per byte per year: 3,480

Exemption threshold: 2 years

Formula: bytes * 3,480 * 2 = minimum_balance

```

  

**Example:**

```

256 byte account:

256 * 3,480 * 2 = 1,781,760 lamports = ~0.00178 SOL

```

  

**Current rates (as of 2025):**

- 3,480 lamports per byte per year

- Must pay 2 years upfront = 6,960 lamports per byte

- No ongoing rent deduction if rent-exempt

  

### 4.3 Lamports vs SOL

  

| Unit | Value |

|------|-------|

| 1 SOL | 1,000,000,000 lamports |

| 1 lamport | 0.000000001 SOL |

  

**Why lamports?**

- All on-chain calculations in lamports (integers)

- Avoids floating-point errors

- 1 lamport is the smallest unit

  

### 4.4 Creating Rent-Exempt Accounts

  

**You MUST provide enough lamports when creating an account:**

  

```rust

// Example in Anchor

#[derive(Accounts)]

pub struct Initialize<'info> {

    #[account(init, payer = user, space = 8 + UserData::INIT_SPACE)]

    pub data_account: Account<'info, UserData>,

    #[account(mut)]

    pub user: Signer<'info>,

    pub system_program: Program<'info, System>,

}

```

  

**What Anchor does automatically:**

1. Calculates required space: `8 (discriminator) + data_size`

2. Queries rent for this size

3. Transfers lamports from payer to new account

4. Account is now rent-exempt

  

### 4.5 Rent Exemption Edge Cases

  

> [!faq] What if I want to store 100MB of data?

> Solana caps initial account creation at 10,240 bytes. You can realloc (grow) later.

  

**Closing accounts and reclaiming rent:**

```

Close account → Transfer all lamports → Account deleted

```

  

---

  

## ⚡ 5. THE SEALEVEL RUNTIME

  

### 5.1 What is Sealevel?

  

> [!note] Definition

> Sealevel is Solana's parallel smart contract runtime. It can execute **thousands of contracts simultaneously** on multi-core processors, not sequentially like Ethereum.

  

**Traditional blockchain execution (EVM):**

```

Block N:

  Instruction 1 → executes

  Instruction 2 → waits

  Instruction 3 → waits

  Instruction 4 → waits

Then all done

```

  

**Solana with Sealevel:**

```

Block N:

  Instruction 1 (touches Account A) ⟶ ✓

  Instruction 2 (touches Account B) ⟶ ✓ (parallel!)

  Instruction 3 (touches Account C) ⟶ ✓ (parallel!)

  Instruction 4 (touches Account A) ⟶ ✗ (conflicts with #1, queued)

```

  

### 5.2 How Sealevel Enables Parallelism

  

**Key requirement:** Each instruction must **explicitly declare** which accounts it reads/writes.

  

```rust

pub fn transfer(

    from: AccountInfo,      // will write

    to: AccountInfo,        // will write

    amount: u64

) {

    // Modify accounts

}

```

  

**Sealevel scheduler:**

1. Receives all pending transactions

2. Analyzes account dependencies for each transaction

3. Finds **non-overlapping** transactions

4. Executes them in parallel on different CPU cores

5. If conflict detected, queues transaction

  

**Example:**

```

Tx 1: Transfer A → B (writes to A, B)

Tx 2: Transfer C → D (writes to C, D)

Result: Can run in parallel (no account overlap)

  

Tx 3: Transfer A → E (writes to A, E)

Result: Must wait after Tx 1 (both write to A)

```

  

### 5.3 Limits to Parallelism

  

**Current limits:**

- Max 200,000 compute units per transaction (default)

- Can increase to 1.4 million CUs at extra cost

- Max 60 million compute units per block

- Max 12 million compute units per account per block

  

**Why the per-account limit?**

- Prevents single "hot" account from becoming bottleneck

- Encourages distributed design

  

---

  

## 📦 6. TRANSACTIONS, INSTRUCTIONS & COMPOSABILITY

  

### 6.1 Transactions vs Instructions

  

**Transaction:**

- Signed by user(s)

- Contains **one or more instructions**

- Atomic: all succeed or all fail

- Executes in sequential order

  

**Instruction:**

- Function call to a program

- Specifies: program ID, accounts, data

- Can invoke another instruction (CPI)

- Has access to accounts listed in instruction

  

### 6.2 Transaction Structure

  

```

┌─ Transaction ─────────────────────┐

│                                    │

│  Signatures: [sig1, sig2, ...]    │ ← Who signed

│  Message:                          │

│  ├─ Accounts: [acct1, acct2, ...] │ ← All accounts used

│  ├─ Instructions: [instr1,        │   ← The calls

│  │                  instr2, ...]  │

│  └─ Recent Blockhash              │ ← Anti-replay

│                                    │

└────────────────────────────────────┘

```

  

**What makes it valid?**

- ✅ Signatures match signers

- ✅ Recent blockhash not expired

- ✅ Instructions execute in order

- ✅ Programs don't modify unauthorized accounts

  

### 6.3 Instruction Metadata

  

Each instruction specifies accounts with metadata:

  

```

Account Name    │ Read/Write │ Signer │ Description

────────────────┼────────────┼────────┼──────────────────

Payer           │ Write      │ Yes    │ Pays for rent

Token Mint      │ Read       │ No     │ Which token?

Token Account   │ Write      │ No     │ Where balance lives

Program         │ Read       │ No     │ Which program?

```

  

**Why this matters:**

- Validator schedules based on **write conflicts**

- Two transactions can run parallel if **no write-write conflicts**

- Reads don't block (multiple readers OK)

  

### 6.4 Atomicity & Failure Semantics

  

**Solana transactions are ATOMIC at the instruction level:**

  

```

Transaction:

  ├─ Instruction 1: Transfer SOL (succeeds)

  ├─ Instruction 2: Mint token (FAILS)

  └─ Instruction 3: Would run here

Result: Instruction 1 reverts, Transaction fails

        No partial state changes

```

  

**Key rule:** If any instruction fails, entire transaction reverts.

  

### 6.5 Composability: Chaining Instructions

  

**You can build complex operations by chaining instructions:**

  

```

1. Create a PDA vault    (System Program)

2. Deposit tokens        (Token Program)

3. Claim rewards         (Your Program)

4. Transfer to wallet    (Token Program)

   → All in ONE transaction

   → All atomic

   → All parallel (if non-overlapping)

```

  

**Benefits:**

- ✅ No race conditions between steps

- ✅ Transparent to users (one signature)

- ✅ Efficient (batched)

  

---

  

## 🔍 7. SYSVARS (SYSTEM VARIABLES)

  

### 7.1 What are Sysvars?

  

> [!note] Definition

> Sysvars are **read-only system accounts** at known addresses that provide cluster state to programs. They're maintained by the Solana runtime.

  

> [!example] Analogy

> Like global constants in a program, but blockchain version.

  

### 7.2 Common Sysvars

  

| Sysvar | Address | Purpose | Example Use |

|--------|---------|---------|-------------|

| **Clock** | `SysvarC1...` | Current slot, epoch, time | Time-locked contracts |

| **Rent** | `SysvarRe...` | Rent rates & exemption threshold | Calculate rent needed |

| **EpochSchedule** | `SysvarEp...` | Slots per epoch, warmup info | Epoch-based logic |

| **RecentBlockhashes** | `SysvarRe...` | Last 400 slot hashes | Anti-replay checks |

| **SlotHashes** | `SysvarSloth...` | Historical slot hashes | Verify historical data |

| **Fees** | `SysvarFe...` | Fee calculator | ⚠️ Deprecated |

| **Instructions** | `Sysvar1nstr...` | All instructions in transaction | Instruction introspection |

  

### 7.3 Accessing Clock Sysvar

  

**In Anchor (easy way):**

```rust

use anchor_lang::solana_program::sysvar::clock::Clock;

  

#[program]

pub mod my_program {

    pub fn my_instruction(ctx: Context<MyContext>) -> Result<()> {

        let clock = Clock::get()?;

        msg!("Current slot: {}", clock.slot);

        msg!("Current epoch: {}", clock.epoch);

        msg!("Unix timestamp: {}", clock.unix_timestamp);

        Ok(())

    }

}

```

  

**Fields in Clock:**

```rust

pub struct Clock {

    pub slot: u64,              // Current slot number

    pub epoch: u64,             // Current epoch

    pub epoch_start_timestamp: i64,  // When epoch started

    pub leader_schedule_epoch: u64,  // Epoch for leader schedule

    pub unix_timestamp: i64,    // Unix timestamp (provider oracle)

}

```

  

### 7.4 Time on Solana

  

**Two ways to measure time:**

  

| Metric | Precision | Use Case |

|--------|-----------|----------|

| **Slot** | ~400ms per slot | Fine-grained timing, exact ordering |

| **Unix Timestamp** | 1 second granularity | Human-readable times, deadlines |

  

> [!warning] Important

> Unix timestamp is **provided by validators** (not cryptographic), so:

> - ✅ Good enough for deadlines

> - ❌ Not good for sub-second precision

> - ⚠️ Can be influenced by validator clocks

  

### 7.5 Rent Sysvar

  

**Accessing rent rates:**

```rust

use anchor_lang::solana_program::sysvar::rent::Rent;

  

let rent = Rent::get()?;

msg!("Lamports per byte per year: {}", rent.lamports_per_byte_year);

msg!("Exemption threshold (years): {}", rent.exemption_threshold);

// rent.burn_percent (deprecated)

```

  

---

  

## 🧮 8. COMPUTE UNITS & RESOURCE CONSTRAINTS

  

### 8.1 What are Compute Units (CU)?

  

> [!note] Definition

> Compute Units are a **measure of computational work**. Each operation costs some CUs. Running out of CUs = transaction fails.

  

> [!question] Why Not Just Use Gas Like Ethereum?

> - Ethereum gas: directly tied to fees

> - Solana CUs: measured separately, decoupled from fees

> - Allows for cheaper batch execution

  

### 8.2 Compute Unit Limits

  

| Scope | Limit |

|-------|-------|

| **Per Transaction (default)** | 200,000 CU |

| **Per Transaction (max)** | 1,400,000 CU |

| **Per Block** | 60,000,000 CU |

| **Per Account per Block** | 12,000,000 CU |

  

**Why a per-account limit?**

- Prevents single "hot" account from consuming entire block

- Encourages distributed system design

  

### 8.3 Common Operation Costs

  

| Operation | CU Cost | Notes |

|-----------|---------|-------|

| Base function call | ~221 CU | Overhead |

| msg!() output | ~226 CU | Per print statement |

| if statement | ~100 CU | Conditional branch |

| Loop (1 iteration) | ~300 CU | Roughly |

| Borsh serialization (simple) | ~2,000-5,000 CU | Depends on size |

| Keccak256 hash | ~2,700 CU | Hash operation |

| Verifiy ed25519 signature | ~24,000 CU | Expensive! |

| Larger data types | More CU | u64 costs more than u32 |

  

### 8.4 Optimizing Compute Usage

  

**Rules of thumb:**

  

1. **Use smaller data types where possible**

   ```rust

   u8:  cheaper than u16, u32, u64

   u32: often good balance

   ```

  

2. **Avoid loops in tight code paths**

   ```rust

   // ❌ Expensive

   for i in 0..1000 { expensive_operation(); }

   // ✅ Better: do computation off-chain

   // Pass result on-chain

   ```

  

3. **Minimize print statements**

   ```rust

   // ❌ Each msg!() costs ~226 CU

   msg!("Value: {}", x);

   msg!("Amount: {}", y);

   // ✅ Better: single combined message

   msg!("Value: {}, Amount: {}", x, y);

   ```

  

4. **Use zero-copy for large accounts**

   ```rust

   // ❌ Borsh deserialization: ~5,000+ CU

   let data = ctx.accounts.large_account.data.borrow();

   // ✅ Zero-copy: ~100 CU

   let data = ctx.accounts.large_account.load()?;

   ```

  

5. **Batch operations across transactions**

   ```rust

   // Instead of many small transactions,

   // combine into single transaction with multiple instructions

   ```

  

### 8.5 Increasing Compute Limit

  

**If you need more compute units:**

  

```rust

use solana_program::compute_budget::ComputeBudgetInstruction;

  

// Set limit to 1M CU (costs extra)

let increase_cu = ComputeBudgetInstruction::set_compute_unit_limit(1_000_000);

```

  

> [!warning] Cost

> Higher limits cost more in fees (priority fees).

  

---

  

## 🧠 9. MEMORY MODEL & SECURITY

  

### 9.1 Solana's Memory Model

  

**Stack:**

- 4KB fixed size

- Local variables, function arguments

- Allocated on-the-fly

  

**Heap:**

- 32KB maximum

- Dynamic allocations (Vec, HashMap)

- Uses bump allocator (no deallocation)

  

**Account Data:**

- Up to 10MB per account

- Persistent across transactions

- Serialized/deserialized on each access

  

### 9.2 Accessing Account Data

  

**Two approaches:**

  

**1. Borsh (standard):**

```rust

let data = MyData::try_from_slice(&account.data.borrow())?;

// Now you have Rust struct

data.value = 42;

```

  

**Pros:** Type-safe, convenient

**Cons:** Expensive (2,000-5,000 CU for deserialization)

  

**2. Zero-Copy:**

```rust

let mut data = account.load_mut::<MyData>()?;

// Mutable reference to raw data

data.value = 42;

```

  

**Pros:** Cheap (~100 CU)

**Cons:** More dangerous, must be careful with lifetime

  

### 9.3 Account Data Serialization

  

**Anchor uses Borsh by default:**

  

```rust

#[account]

pub struct MyData {

    pub count: u64,      // 8 bytes

    pub name: String,    // 4 bytes (len) + string bytes

    pub owner: Pubkey,   // 32 bytes

}

  

// Anchor also adds 8-byte discriminator

// Total: 8 + 8 + (4 + name) + 32

```

  

**Alignment matters:**

```rust

// This is efficient:

struct Efficient {

    value: u64,    // 8 bytes

    flag: u8,      // 1 byte

    another: u64,  // 8 bytes

}

// Total: 17 bytes (aligned)

  

// This wastes space:

struct Wasteful {

    value: u64,    // 8 bytes

    flag: u8,      // 1 byte (padding: 7 bytes wasted!)

    another: u64,  // 8 bytes

}

```

  

### 9.4 Access Control & Security

  

**The security model:**

  

```

┌─ Program A ──────┐

│ (owner program)  │

│ Can write data   │

└──────────────────┘

         ↑

         │ owns

         ↓

┌─ Account X ──────┐

│ (data account)   │

│ (owner = A)      │

│ data = [...]     │

└──────────────────┘

         ↓

         │ only Program A can write

         ↓

Program B cannot access ❌

```

  

**Access control rules:**

- ✅ Program A (owner) can read/write Account X

- ❌ Program B cannot write Account X (different owner)

- ❌ User cannot directly modify Account X

- ✅ User can invoke Program A to modify Account X (if authorized)

  

---

  

## 🔀 10. CROSS-PROGRAM INVOCATION (CPI)

  

### 10.1 What is CPI?

  

> [!note] Definition

> CPI (Cross-Program Invocation) is when one program calls another program. It's how Solana programs compose and build on each other.

  

> [!example] Analogy

> Like importing a library function, but on-chain.

  

### 10.2 CPI vs System Program Calls

  

**Regular instruction:**

```

User → Creates transaction → Calls Program A

```

  

**CPI:**

```

Program A → invoke!() → Program B

```

  

**Key difference:** Program A is the "signer" of the CPI (not the user).

  

### 10.3 CPI Patterns

  

**Pattern 1: Invoke System Program (transfer SOL)**

```rust

invoke(

    &system_instruction::transfer(

        &from.key(),

        &to.key(),

        amount

    ),

    &[from.clone(), to.clone(), system_program.clone()],

)?;

```

  

**Pattern 2: Invoke Token Program (transfer tokens)**

```rust

invoke(

    &token::transfer(

        &token_program.key(),

        &source.key(),

        &destination.key(),

        &authority.key(),

        &[],

        amount

    )?,

    &[source.clone(), destination.clone(), authority.clone()],

)?;

```

  

**Pattern 3: Invoke Another Program**

```rust

let cpi_instruction = program_b::instruction::some_function(param);

invoke(&cpi_instruction, &account_infos)?;

```

  

### 10.4 Signed vs Unsigned CPIs

  

**Unsigned CPI (PDA doesn't sign):**

```rust

invoke(

    &instruction,

    &account_infos,

)?;

```

  

**Signed CPI (PDA signs via invoke_signed):**

```rust

invoke_signed(

    &instruction,

    &account_infos,

    &[&[b"seed", &[bump]]], // PDA derives from these

)?;

```

  

**When to use:**

- **Unsigned:** When instruction needs regular signer (user)

- **Signed:** When instruction needs PDA to sign (e.g., escrow release)

  

### 10.5 CPI Security Concerns

  

> [!danger] ⚠️ Signer Privilege Reuse

> ```rust

> // DANGEROUS! Signer status flows through

> invoke(

>     &untrusted_instruction,

>     &[user_signer.clone(), ...] // User is signer

> )?;

> // Attacker-controlled program can abuse user's signature!

> ```

  

> [!success] ✅ Safe Approach

> ```rust

> // Only pass specific accounts needed

> // Don't pass signer unless necessary

> invoke(

>     &my_trusted_instruction,

>     &[vault.clone(), destination.clone()],

> )?;

> ```

  

> [!danger] ⚠️ Arbitrary Program Invocation

> ```rust

> // DANGEROUS!

> let program = // user-provided

> invoke(&instruction, &[program.clone(), ...])?;

> // Attacker can pass malicious program!

> ```

  

> [!success] ✅ Safe Approach

> ```rust

> // Hardcode expected program ID

> require_keys_eq!(program.key(), EXPECTED_PROGRAM_ID)?;

> invoke(&instruction, &[program.clone(), ...])?;

> ```

  

---

  

## 🎪 11. TOKEN PROGRAM & SPL TOKENS

  

### 11.1 What is SPL Token Program?

  

> [!note] Definition

> SPL = Solana Program Library. The Token Program is the standard interface for creating and managing tokens on Solana (like ERC-20 on Ethereum).

  

**Key concepts:**

  

| Concept | Purpose |

|---------|---------|

| **Mint** | Token definition (name, decimals, total supply) |

| **Token Account** | User's balance in a specific token |

| **Authority** | Who can mint/burn/freeze tokens |

  

### 11.2 Mint Account

  

**Stores:**

```rust

pub struct Mint {

    pub supply: u64,                    // Total tokens minted

    pub decimals: u8,                   // Decimal places (0-18)

    pub is_initialized: bool,

    pub owner: Option<Pubkey>,          // Mint authority

    pub freeze_authority: Option<Pubkey>, // Can freeze accounts

}

```

  

**Example:**

```

USDC Mint:

├─ Supply: 25 billion (in atomic units)

├─ Decimals: 6

└─ Owner: Circle (can mint more)

```

  

### 11.3 Token Accounts

  

**Each user holds tokens in a Token Account:**

  

```rust

pub struct Account {

    pub mint: Pubkey,                // Which token

    pub owner: Pubkey,               // Who owns this account

    pub amount: u64,                 // Balance

    pub delegate: COption<Pubkey>,   // Delegated to?

    pub delegated_amount: u64,       // How much delegated

    pub is_initialized: bool,

    pub is_native: bool,

    pub state: AccountState,         // Frozen?

}

```

  

**Example transaction:**

```

Alice's USDC Account: 1000 USDC

      ↓ (transfer 100)

Bob's USDC Account: +100 USDC

```

  

### 11.4 Token Transfer Mechanics

  

**Step-by-step:**

  

1. User signs transaction

2. Invoke Token Program with `transfer` instruction

3. Token Program checks:

   - Does source account hold enough tokens?

   - Is owner (or delegate) a signer?

4. Token Program modifies:

   - Reduce source account amount

   - Increase destination account amount

5. Transaction confirms

  

### 11.5 Token Program Authorities

  

**Mint Authority:**

- Can mint new tokens

- Can burn tokens

- Usually held by project team

  

**Freeze Authority:**

- Can freeze/unfreeze accounts

- Prevents transfers from frozen accounts

- Used for compliance (e.g., sanctions)

  

**Owner (Account Authority):**

- Can authorize transfers

- Can approve delegations

- Is owner of Token Account

  

---

  

## 💳 12. ASSOCIATED TOKEN ACCOUNTS (ATAs)

  

### 12.1 What is an ATA?

  

> [!note] Definition

> An ATA is a **deterministically derived** Token Account that belongs to a user for a specific token. One ATA per user per token.

  

| Before ATAs | With ATAs |

|-------------|------------|

| Users needed to create token accounts manually → confusing | Standardized, deterministic addresses → seamless |

  

### 12.2 ATA Derivation

  

**Formula:**

```

ATA_Address = SHA256(

    program_id=TokenMetadataProgram ||

    owner_pubkey ||

    mint ||

    ""  // empty seed

)

```

  

**Key insight:** Same owner + same mint = same ATA always.

  

**Example:**

```

User: Alice

Mint: USDC

ATA: [deterministic address]

  

Later: Alice wants USDC account

Same ATA address derives again

→ Can reuse or initialize if needed

```

  

### 12.3 Creating ATAs

  

**Before:**

```rust

// Manual Token Account creation (many steps)

let create_account_ix = system_instruction::create_account(...);

let init_token_account_ix = token_instruction::init_account(...);

```

  

**With ATAs (standard way):**

```rust

let ata = get_associated_token_address(&user, &mint);

// If doesn't exist, create it (atomic)

let create_ata_ix = create_associated_token_account(

    &payer,

    &user,

    &mint,

);

```

  

### 12.4 ATA Standards

  

> [!success] Advantages

> - ✅ Same address every time

> - ✅ Clients know user's token account without query

> - ✅ Can batch-initialize ATAs

> - ✅ Reduces transaction size

  

> [!warning] Disadvantages

> - ❌ Only one ATA per token per user (can't have multiple)

> - ❌ Requires ATA Program for atomicity

  

---

  

## 🖼️ 13. METADATA PROGRAM (NFT METADATA)

  

### 13.1 The Metadata Program

  

> [!note] Definition

> The Metadata Program (by Metaplex) attaches extra data to SPL tokens, turning them into NFTs with name, symbol, image, etc.

  

| Without Metadata | With Metadata |

|------------------|---------------|

| Just a number (token amount) | Full NFT with image, description, royalties |

  

### 13.2 Metadata Account Structure

  

**Creates a PDA:**

```

Seeds: [

    b"metadata",

    metadata_program_id,

    mint_pubkey

]

```

  

**Stores:**

```rust

pub struct Metadata {

    pub key: MetadataKey,

    pub update_authority: Pubkey,  // Can update

    pub mint: Pubkey,              // Which token?

    pub data: Data,                // Name, symbol, URI

    pub primary_sale_happened: bool,

    pub is_mutable: bool,

    pub edition: Option<MasterEdition>,  // For NFTs

}

  

pub struct Data {

    pub name: String,

    pub symbol: String,

    pub uri: String,  // Link to off-chain JSON

    pub seller_fee_basis_points: u16,  // Royalties

    pub creators: Vec<Creator>,

}

```

  

### 13.3 Metadata Workflow

  

**Creating an NFT:**

  

```

1. Create Mint (SPL Token Program)

   └─ Result: Mint Account

  

2. Create Metadata (Metadata Program)

   └─ PDA: derived from mint

   └─ Stores: name, symbol, image URL, royalties

  

3. Create Master Edition (Metadata Program)

   └─ Marks mint as "1 of 1" (unique)

```

  

### 13.4 Off-Chain Metadata JSON

  

**Metadata URI points to JSON:**

  

```json

{

  "name": "Cool NFT #1",

  "symbol": "COOL",

  "description": "An awesome NFT",

  "image": "https://example.com/nft1.png",

  "attributes": [

    {

      "trait_type": "Rarity",

      "value": "Legendary"

    },

    {

      "trait_type": "Color",

      "value": "Blue"

    }

  ],

  "external_url": "https://example.com",

  "properties": {

    "files": [

      {

        "uri": "https://example.com/nft1.png",

        "type": "image/png"

      }

    ],

    "category": "image",

    "creators": [

      {

        "address": "...",

        "verified": true,

        "share": 100

      }

    ]

  }

}

```

  

---

  

## 🔐 14. SIGNER RULES & AUTHORIZATION

  

### 14.1 The Signer Concept

  

> [!note] Definition

> A signer is an account that has authorized a transaction by providing a valid signature with its private key. Programs verify this signature was provided.

  

**Example:**

```

Alice's keypair secret: [private key]

Transaction signed: signature = sign(transaction, private_key)

Blockchain verifies: valid_signature(transaction, Alice_pubkey)?

```

  

### 14.2 Signer vs Authority

  

| Term | Meaning | Who |

|------|---------|-----|

| **Signer** | Provided valid signature for transaction | User/wallet |

| **Authority** | Authorized by program to do something | Can be user, PDA, or other program |

  

**Example:**

```rust

// User is signer (signed the transaction)

pub fn transfer(

    ctx: Context<Transfer>,

    amount: u64,

) -> Result<()> {

    // But program checks if user is "authorized" to transfer

    // (e.g., is user the token account owner?)

    require_keys_eq!(ctx.accounts.token_account.owner, ctx.accounts.user.key())?;

    // Then: user is both signer AND authorized

    Ok(())

}

```

  

### 14.3 Signer Checks

  

**Every privileged operation needs signer verification:**

  

```rust

// ❌ WRONG: No signer check

pub fn withdraw(ctx: Context<Withdraw>) -> Result<()> {

    ctx.accounts.vault.amount = 0;

    Ok(())

}

// Anyone can call this!

  

// ✅ RIGHT: Check signer

pub fn withdraw(ctx: Context<Withdraw>) -> Result<()> {

    require!(ctx.accounts.user.is_signer, CustomError::MustBeSigner)?;

    require_keys_eq!(ctx.accounts.vault.owner, ctx.accounts.user.key())?;

    ctx.accounts.vault.amount = 0;

    Ok(())

}

// Only owner can withdraw

```

  

### 14.4 Signer Privilege Reuse (Security Risk)

  

> [!danger] The Vulnerability

> ```rust

> // DANGEROUS CODE

> pub fn execute(

>     ctx: Context<Execute>,

>     instruction: Instruction,

> ) -> Result<()> {

>     // User signed this transaction

>     // But we're passing their signer status to untrusted program!

>     invoke(&instruction, &[ctx.accounts.user.clone(), ...])?;

>     Ok(())

> }

>

> // Attacker:

> // 1. Calls execute() with malicious instruction

> // 2. Attacker's program receives user as signer

> // 3. Attacker's program can now perform actions as if user authorized them!

> ```

  

> [!success] Prevention

> ```rust

> // ✅ SAFE: Don't pass user signer to untrusted program

> pub fn execute(

>     ctx: Context<Execute>,

>     instruction: Instruction,

> ) -> Result<()> {

>     // Don't include user in CPI accounts!

>     // User signer status doesn't propagate

>     invoke(&instruction, &[vault.clone(), ...])?;

>     Ok(())

> }

> ```

  

### 14.5 PDA Signing (No Private Key)

  

**Special case: PDAs can "sign" without private key**

  

```rust

// PDA is signer in a different sense:

// Program invokes_signed with correct seeds

invoke_signed(

    &instruction,

    &accounts,

    &[&[b"seed", &[bump]]], // Seeds that derive PDA

)?;

  

// Runtime verifies:

// "Does this PDA derive from these seeds?"

// If yes, treat as valid "signer"

```

  

---

  

## ⚓ 15. ANCHOR FRAMEWORK FUNDAMENTALS

  

### 15.1 What is Anchor?

  

> [!note] Definition

> Anchor is a **framework that simplifies Solana program development** by handling boilerplate like serialization, account validation, and error handling.

  

| Before Anchor | With Anchor |

|---------------|-------------|

| 100 lines of manual account validation | 10 lines of attributes |

  

### 15.2 Anchor's Core Components

  

| Component | Purpose |

|-----------|---------|

| **#[derive(Accounts)]** | Auto-validate account inputs |

| **#[account]** | Define data structures |

| **#[program]** | Define instruction handlers |

| **Constraints** | Declarative security checks |

| **Error Codes** | Custom error types |

| **Events** | On-chain logging/indexing |

  

### 15.3 Accounts Macro

  

```rust

#[derive(Accounts)]

pub struct Initialize<'info> {

    #[account(init, payer = payer, space = 8 + size)]

    pub data_account: Account<'info, MyData>,

    #[account(mut)]

    pub payer: Signer<'info>,

    pub system_program: Program<'info, System>,

}

```

  

**What it does:**

- ✅ Verifies accounts passed are as expected

- ✅ Deserializes data structures

- ✅ Checks ownership

- ✅ Validates signer status

  

### 15.4 Core Constraints

  

| Constraint | Effect |

|-----------|--------|

| `#[account(init)]` | Create new account |

| `#[account(mut)]` | Account data will be modified |

| `#[account(signer)]` | Account must have signed |

| `#[account(seeds, bump)]` | Validate PDA derivation |

| `#[account(close = recipient)]` | Close and reclaim rent |

| `#[account(has_one = field)]` | Verify relationship |

  

### 15.5 Program Module

  

```rust

#[program]

pub mod my_program {

    use super::*;

  

    pub fn initialize(ctx: Context<Initialize>, param: u64) -> Result<()> {

        // Your logic

        msg!("Initialized!");

        Ok(())

    }

}

```

  

**Structure:**

- Each function = one instruction

- Takes `Context<T>` where T = accounts struct

- Returns `Result<()>`

  

### 15.6 Error Handling

  

```rust

#[error_code]

pub enum MyError {

    #[msg("Unauthorized access")]

    Unauthorized,

    #[msg("Invalid amount")]

    InvalidAmount,

}

  

pub fn transfer(ctx: Context<Transfer>, amount: u64) -> Result<()> {

    require!(amount > 0, MyError::InvalidAmount)?;

    require!(ctx.accounts.user.is_signer, MyError::Unauthorized)?;

    Ok(())

}

```

  

---

  

## 🛡️ 16. SECURITY PATTERNS & BEST PRACTICES

  

### 16.1 The Security Model

  

**Solana's security is based on:**

  

```

┌─ Explicit Account Dependencies ─┐

│ (declared upfront)              │

├─────────────────────────────────┤

│ Runtime can enforce:             │

│ ✓ No unauthorized access         │

│ ✓ Parallelization safety         │

│ ✓ Predictable behavior           │

└─────────────────────────────────┘

```

  

### 16.2 The Golden Rules

  

**Rule 1: Always verify account ownership**

```rust

require_keys_eq!(account.owner, my_program_id)?;

```

  

**Rule 2: Always verify signers**

```rust

require!(account.is_signer)?;

```

  

**Rule 3: Always verify data freshness**

```rust

if account.data_is_stale() {

    return Err(ProgramError::InvalidData);

}

```

  

**Rule 4: Always use canonical bump**

```rust

require_eq!(stored_bump, canonical_bump)?;

```

  

### 16.3 Common Vulnerabilities

  

| Vulnerability | Risk | Fix |

|---------------|------|-----|

| Missing signer check | Unauthorized access | Check `is_signer` |

| Missing ownership check | Data tampering | Verify `owner` field |

| Unvalidated PDA | Wrong account used | Verify bump is canonical |

| Arbitrary CPI | Signer privilege reuse | Hardcode program IDs |

| Missing rent-exemption check | Account garbage collection | Verify rent-exempt |

  

### 16.4 Validation Checklist

  

> [!todo] Before Calling a Function, Verify:

> - [ ] Account ownership (is it owned by my program?)

> - [ ] Account type (is it the right data structure?)

> - [ ] Signer status (did the right account sign?)

> - [ ] PDA derivation (is bump canonical?)

> - [ ] Data consistency (is data what we expect?)

> - [ ] Rent exemption (won't be collected?)

  

### 16.5 Defense in Depth

  

```rust

pub fn sensitive_operation(

    ctx: Context<SensitiveOp>,

) -> Result<()> {

    // Check 1: Ownership

    require_keys_eq!(ctx.accounts.vault.owner, crate::ID)?;

    // Check 2: Signer

    require!(ctx.accounts.authority.is_signer)?;

    // Check 3: Authority matches

    require_keys_eq!(

        ctx.accounts.vault.authority,

        ctx.accounts.authority.key()

    )?;

    // Check 4: PDA validation

    require_eq!(ctx.accounts.vault.bump, canonical_bump)?;

    // THEN do operation

    ctx.accounts.vault.balance = 0;

    Ok(())

}

```

  

---

  

## 🔄 17. PROGRAM UPGRADEABILITY

  

### 17.1 How Programs Can Be Upgraded

  

**Solana programs CAN be upgraded:**

  

1. Program account allocated with **2x** the original size

2. New bytecode written to account

3. Authority transferred to new code (or locked)

  

### 17.2 Upgrade Authority

  

```bash

# Transfer upgrade authority

solana program set-upgrade-authority <PROGRAM_ID> --new-upgrade-authority <NEW_AUTHORITY>

  

# Make program immutable (no future upgrades)

solana program set-upgrade-authority <PROGRAM_ID> --final

```

  

### 17.3 Upgrade Risks

  

> [!danger] ⚠️ State Breaking Changes

> ```rust

> // OLD data structure

> pub struct OldData {

>     pub value: u64,

> }

>

> // NEW data structure (incompatible!)

> pub struct NewData {

>     pub value: u32,  // Changed size!

>     pub new_field: u32,

> }

>

> // Old data can't deserialize to new structure

> // Accounts become unusable!

> ```

  

> [!success] ✅ Backward Compatible Upgrades

> ```rust

> // Add fields at the end

> pub struct Data {

>     pub value: u64,           // Original field

>     pub new_field: Option<u32>, // New field (optional)

> }

>

> // Old data still deserializes (new_field = None)

> // New data deserializes fine

> ```

  

### 17.4 Best Practices

  

> [!tip] Best Practices for Upgrades

> 1. **Plan for upgrades:** Design data structures to be extensible

> 2. **Use versioning:** Include version in data structures

> 3. **Deploy to devnet first:** Test upgrades before mainnet

> 4. **Document changes:** Clear migration path for users

> 5. **Consider immutability:** For critical programs, lock authority

  

---

  

## 🚀 18. ADVANCED TOPICS & OPTIMIZATION

  

### 18.1 Instruction Introspection

  

> [!note] Definition

> A program can inspect other instructions in the same transaction using the `Instruction` Sysvar.

  

**Use case:**

```

Program A: "Before executing my instruction,

           verify that a signature verification

           happened in Instruction 1"

  

Instruction 1: ed25519 signature verification

Instruction 2: Program A checks Instruction 1 happened

```

  

**How it works:**

```rust

use solana_program::sysvar::instructions;

  

pub fn my_instruction(ctx: Context<MyContext>) -> Result<()> {

    // Get current instruction index

    let current_index = instructions::load_current_index_checked()?;

    // Load previous instruction

    let prev_instruction = instructions::load_instruction_at_checked(

        current_index - 1,

        &ctx.accounts.instructions_sysvar

    )?;

    // Verify it's what we expect

    require_keys_eq!(prev_instruction.program_id, EXPECTED_PROGRAM)?;

    Ok(())

}

```

  

### 18.2 Zero-Copy Deserialization

  

> [!tip] For very large accounts, Borsh is too expensive:

  

```rust

#[account(zero_copy)]

pub struct LargeData {

    pub items: [Item; 10000],  // Huge!

}

  

pub fn read_large_data(ctx: Context<ReadLargeData>) -> Result<()> {

    let data = ctx.accounts.large_account.load()?;

    // Direct access to data, no deserialization!

    let first_item = data.items[0];

    Ok(())

}

```

  

> [!info] Cost Comparison

> | Method | CU Cost |

> |--------|--------|

> | Borsh | 5,000-10,000 CU |

> | Zero-copy | 100 CU |

  

### 18.3 Reallocation & Account Growth

  

```rust

#[derive(Accounts)]

pub struct GrowAccount<'info> {

    #[account(mut, realloc = new_size, realloc::payer = payer, realloc::zero = false)]

    pub account: Account<'info, MyData>,

    #[account(mut)]

    pub payer: Signer<'info>,

    pub system_program: Program<'info, System>,

}

```

  

**How it works:**

1. Calculate new rent needed

2. Payer transfers difference to account

3. Solana expands account data

  

### 18.4 Optimizing for Parallel Execution

  

**Design accounts so instructions don't conflict:**

  

```rust

// ❌ WRONG: All users write to shared counter

struct GlobalState {

    counter: u64,  // Bottleneck!

}

  

// ✅ RIGHT: Each user has own counter

struct UserState {

    pubkey: Pubkey,

    counter: u64,

}

  

// Different instructions can now run in parallel!

```

  

### 18.5 MEV & Frontrunning Mitigation

  

**Solana doesn't have mempool, but CPIs can still be exploited:**

  

```rust

// ❌ Vulnerable: Price at time of execution

pub fn swap(ctx: Context<Swap>, min_received: u64) -> Result<()> {

    let amount_out = calculate_output()?;

    require!(amount_out >= min_received)?;

    Ok(())

}

  

// ✅ Safer: Include timestamp check

pub fn swap(ctx: Context<Swap>, min_received: u64, max_slot_diff: u64) -> Result<()> {

    let clock = Clock::get()?;

    require!(clock.slot - last_known_slot < max_slot_diff)?;

    let amount_out = calculate_output()?;

    require!(amount_out >= min_received)?;

    Ok(())

}

```

  

---

  

## 📖 KEY TAKEAWAYS

  

> [!abstract]+ For Interviews

> 1. **Understand why Solana is fast:** PoH + Sealevel + parallel execution

> 2. **Know the account model:** Programs don't store data; accounts do

> 3. **PDAs are crucial:** They enable trustless operations

> 4. **Security is explicit:** Validators can't hide things; it's in the code

> 5. **Composability matters:** Programs build on each other; transactions are atomic

  

> [!abstract]+ For Production

> 6. **Always validate accounts:** Check owner, signer, existence

> 7. **Use Anchor constraints:** They're security-tested

> 8. **Optimize compute:** Measure before optimizing

> 9. **Design for parallelism:** Avoid hot accounts

> 10. **Test thoroughly:** Especially error cases and security scenarios

  

> [!important] Critical Rules

> - ✅ Check signer for privileged operations

> - ✅ Verify account ownership

> - ✅ Use canonical bump for PDAs

> - ✅ Validate all external program calls

> - ✅ Ensure rent-exemption for important accounts

> - ✅ Batch operations to save fees

> - ✅ Write tests for security scenarios

  

---

  

## 🎓 NEXT STEPS

  

> [!rocket] What's Next?

> Now that you understand Solana's architecture and concepts, you're ready for the **practical coding section** where we'll build real Anchor programs using these patterns.

  

---

  

> [!info] Metadata

> **Last Updated:** December 5, 2025

> **Format:** Obsidian Markdown with Emojis, Tables, Callouts, and Nested Structure

>

> #solana #blockchain #web3 #anchor #rust
