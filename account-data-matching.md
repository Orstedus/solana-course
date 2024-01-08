
Заголовок: Перевірка даних акаунту на валідність
Цілі:
- Пояснити ризики у безпеці, пов'язані з відсутністю перевірки даних
- Здійснення перевірки даних на валідність, використовуючи повноформатний RUST
- Здійснення перевірки даних на валідність за допомогою обмежуючих Анчорів


# TL;DR

- Використовуйте **перевірки даних** для підтвердження того, що дані акаунту співпадають з тими, які повинні бути**.** Без відповідної перевірки даних можуть з'явитися неочікувані акаунти.
- Щоб виконати перевірку даних у Rust, просто порівняйте дані акаунту з значенням, яке повинно бути.
    
    ```rust
    if ctx.accounts.user.key() != ctx.accounts.user_data.user {
        return Err(ProgramError::InvalidAccountData.into());
    }
    ```
    
- У Анчорах, ви можете використовувати `constraint` щоб перевірити, чи є даний вираз правдивим. Крім того, можна використовувати `has_one` щоб перевірити, чи поле цільового облікового запису зберігається в акаунті, який відповідає ключу у структурі `Accounts`.

# Загалом

Співпадіння даних акаунту відноситься до перевірки даних на валідність, які використовуються задля підтвердження, що дані акаунту збігаються з даними, які очікує програма. Перевірка даних додає додаткові обмеження, щоб забезпечити вхід відповідних акаунтів до системи. 

Це корисно, коли облікові записи залежать від значень інших облікових записів, або якщо система залежить від даних, збережених в акаунті. 

### Відсутність перевірки даних

Приклад нижче включає в себе функцію `update_admin`, яка оновлює поле `admin`, що вберігається у акаунті `admin_config`. 

У функції бракує перевірки даних, щоб співставити, чи акаунт `admin`, підписуючий транзакцію, відповідає акаунту `admin`, збереженому у `admin_config`. Це означає, що будь-який акаунт, який підписав транзакцію і пройшов у функцію як `admin`, може редагувати `admin_config`.

```rust
use anchor_lang::prelude::*;

declare_id!("Fg6PaFpoGXkYsidMpWTK6W2BeZ7FEfcYkg476zPFsLnS");

#[program]
pub mod data_validation {
    use super::*;
    ...
    pub fn update_admin(ctx: Context<UpdateAdmin>) -> Result<()> {
        ctx.accounts.admin_config.admin = ctx.accounts.new_admin.key();
        Ok(())
    }
}

#[derive(Accounts)]
pub struct UpdateAdmin<'info> {
    #[account(mut)]
    pub admin_config: Account<'info, AdminConfig>,
    #[account(mut)]
    pub admin: Signer<'info>,
    pub new_admin: SystemAccount<'info>,
}

#[account]
pub struct AdminConfig {
    admin: Pubkey,
}
```

### Додавання перевірки даних 

Базовий підхід для вирішення цієї проблеми у Rust це порівняти вхідний ключ з ключом `admin`, розташованим у `admin_config`, виводимо помилку якщо вони не співпадають.

```rust
if ctx.accounts.admin.key() != ctx.accounts.admin_config.admin {
    return Err(ProgramError::InvalidAccountData.into());
}
```

Після додавання перевірки даних, функція `update_admin` буде продовжуватися тільки у тому випадку, коли акаунт `admin`, підписуючий транзакцію, співпадає з `admin`, розташованому у `admin_config`.

```rust
use anchor_lang::prelude::*;

declare_id!("Fg6PaFpoGXkYsidMpWTK6W2BeZ7FEfcYkg476zPFsLnS");

#[program]
pub mod data_validation {
    use super::*;
    ...
    pub fn update_admin(ctx: Context<UpdateAdmin>) -> Result<()> {
      if ctx.accounts.admin.key() != ctx.accounts.admin_config.admin {
            return Err(ProgramError::InvalidAccountData.into());
        }
        ctx.accounts.admin_config.admin = ctx.accounts.new_admin.key();
        Ok(())
    }
}

#[derive(Accounts)]
pub struct UpdateAdmin<'info> {
    #[account(mut)]
    pub admin_config: Account<'info, AdminConfig>,
    #[account(mut)]
    pub admin: Signer<'info>,
    pub new_admin: SystemAccount<'info>,
}

#[account]
pub struct AdminConfig {
    admin: Pubkey,
}
```

### Використання обмежуючих анчорів

Анчор облегшує цю задачу за допомогою обмежувача `has_one`. Ви можете використовувати `has_one`, щоб перенести перевірку даних з функції до структури `UpdateAdmin`.

На прикаді нижче показано, як `has_one = admin` уточнює, чи акаунт `admin`, підписуючий транзакцію, співпадає з акаунтом `admin`, розташованому у `admin_config`. Щоб використовувати обмежувач `has_one`, конвенція найменування поля в акаунті повинна відповідати конвенції найменування у структурі валідації акаунту. 

```rust
use anchor_lang::prelude::*;

declare_id!("Fg6PaFpoGXkYsidMpWTK6W2BeZ7FEfcYkg476zPFsLnS");

#[program]
pub mod data_validation {
    use super::*;
    ...
    pub fn update_admin(ctx: Context<UpdateAdmin>) -> Result<()> {
        ctx.accounts.admin_config.admin = ctx.accounts.new_admin.key();
        Ok(())
    }
}

#[derive(Accounts)]
pub struct UpdateAdmin<'info> {
    #[account(
        mut,
        has_one = admin
    )]
    pub admin_config: Account<'info, AdminConfig>,
    #[account(mut)]
    pub admin: Signer<'info>,
    pub new_admin: SystemAccount<'info>,
}

#[account]
pub struct AdminConfig {
    admin: Pubkey,
}
```

Як варіант, ви можете використовувати `constraint` для додавання виразу, який буде виходити у позитивне значення. Це корисно, коли, з якоїсь причини, ви не можете зробити конвенції найменування однаковими чи вам треба більш складний вираз для повної валідації вхідних даних.

```rust
#[derive(Accounts)]
pub struct UpdateAdmin<'info> {
    #[account(
        mut,
        constraint = admin_config.admin == admin.key()
    )]
    pub admin_config: Account<'info, AdminConfig>,
    #[account(mut)]
    pub admin: Signer<'info>,
    pub new_admin: SystemAccount<'info>,
}
```

# Створення демо

У цьому демо ми зробимо просту программу "сховище", схожу на ту, яку ми робили на уроці авторизації підписувача та на уроці перевірки власника. Так само, як на ціх демо, ми покажемо як відсутність перевірки даних на валідність може призвести до руйнування нашого сховища.

### 1. Starter

Для початку, встановіть початковий код з гілки `starter` [цієї репозиторії](https://github.com/Unboxed-Software/solana-account-data-matching). Початковий код включає в себе дві функції та шаблонні налаштування для тестового файлу. 

Функція `initialize_vault` ініціалізує новій акаунт `Vault` та `TokenAccount`. Акаунт `Vault` буде зберігати адрес токен акаунту, повноваження сховища, та призначення зняття коштів з токен акаунту.

Повноваження нового токен акаунту будуть призначені на `vault`, PDA програми. Це дозволяє акаунту `vault` підписувати перенесення токенів з токен акаунту. 

Функція `insecure_withdraw` переносить усі токени токен акаунту `vault` до токен акаунту `withdraw_destination`. 

Звурніть увагу, що ця функція ****має**** перевірку підписувача `authority` та перевірку власника `vault`. Проте, у валідації акаунту або логіці функції ніде нема коду, який перевіряє, що `authority` акаунту, який пройшов у функцію, співпадає з `authority` акаунту у `vault`.

```rust
use anchor_lang::prelude::*;
use anchor_spl::token::{self, Mint, Token, TokenAccount};

declare_id!("Fg6PaFpoGXkYsidMpWTK6W2BeZ7FEfcYkg476zPFsLnS");

#[program]
pub mod account_data_matching {
    use super::*;

    pub fn initialize_vault(ctx: Context<InitializeVault>) -> Result<()> {
        ctx.accounts.vault.token_account = ctx.accounts.token_account.key();
        ctx.accounts.vault.authority = ctx.accounts.authority.key();
        ctx.accounts.vault.withdraw_destination = ctx.accounts.withdraw_destination.key();
        Ok(())
    }

    pub fn insecure_withdraw(ctx: Context<InsecureWithdraw>) -> Result<()> {
        let amount = ctx.accounts.token_account.amount;

        let seeds = &[b"vault".as_ref(), &[*ctx.bumps.get("vault").unwrap()]];
        let signer = [&seeds[..]];

        let cpi_ctx = CpiContext::new_with_signer(
            ctx.accounts.token_program.to_account_info(),
            token::Transfer {
                from: ctx.accounts.token_account.to_account_info(),
                authority: ctx.accounts.vault.to_account_info(),
                to: ctx.accounts.withdraw_destination.to_account_info(),
            },
            &signer,
        );

        token::transfer(cpi_ctx, amount)?;
        Ok(())
    }
}

#[derive(Accounts)]
pub struct InitializeVault<'info> {
    #[account(
        init,
        payer = authority,
        space = 8 + 32 + 32 + 32,
        seeds = [b"vault"],
        bump,
    )]
    pub vault: Account<'info, Vault>,
    #[account(
        init,
        payer = authority,
        token::mint = mint,
        token::authority = vault,
        seeds = [b"token"],
        bump,
    )]
    pub token_account: Account<'info, TokenAccount>,
    pub withdraw_destination: Account<'info, TokenAccount>,
    pub mint: Account<'info, Mint>,
    #[account(mut)]
    pub authority: Signer<'info>,
    pub token_program: Program<'info, Token>,
    pub system_program: Program<'info, System>,
    pub rent: Sysvar<'info, Rent>,
}

#[derive(Accounts)]
pub struct InsecureWithdraw<'info> {
    #[account(
        seeds = [b"vault"],
        bump,
    )]
    pub vault: Account<'info, Vault>,
    #[account(
        mut,
        seeds = [b"token"],
        bump,
    )]
    pub token_account: Account<'info, TokenAccount>,
    #[account(mut)]
    pub withdraw_destination: Account<'info, TokenAccount>,
    pub token_program: Program<'info, Token>,
    pub authority: Signer<'info>,
}

#[account]
pub struct Vault {
    token_account: Pubkey,
    authority: Pubkey,
    withdraw_destination: Pubkey,
}
```

### 2. Тест функції `insecure_withdraw`

Щоб показати, що це проблема, давайте напишемо тест, де акаунт з іншим від сховища значенням `authority` намагається зняти кошти з сховища.

Тестовий файл включає в себе код, який виконує функцію `initialize_vault`, використовуючи гаманець як `authority` а потім переводить 100 токенів на токен акаунт `vault`.

Додайте тест, який буде викликати функцію `insecure_withdraw`. Використовуйте `withdrawDestinationFake` як акаунт `withdrawDestination` та `walletFake` як `authority`. Потім здійсніть транзакцію від імені `walletFake`.

Так як у нас немає перевірок для підтвердження акаунту `authority`, він проходить у функцію під співпадаючими значеннями з акаунту `vault`, які ми ініціалізували на першому тесті, функція спрацює успішно і кошти будуть відправлені до акаунту `withdrawDestinationFake`.

```tsx
describe("account-data-matching", () => {
  ...
  it("Insecure withdraw", async () => {
    const tx = await program.methods
      .insecureWithdraw()
      .accounts({
        vault: vaultPDA,
        tokenAccount: tokenPDA,
        withdrawDestination: withdrawDestinationFake,
        authority: walletFake.publicKey,
      })
      .transaction()

    await anchor.web3.sendAndConfirmTransaction(connection, tx, [walletFake])

    const balance = await connection.getTokenAccountBalance(tokenPDA)
    expect(balance.value.uiAmount).to.eq(0)
  })
})
```

Запустіть `anchor test` щоб побачити, що обидві транзакції будуть успішні.

```bash
account-data-matching
  ✔ Initialize Vault (811ms)
  ✔ Insecure withdraw (403ms)
```

### 3. Додавання функції `secure_withdraw`

Давайте реалізуємо безпечну версію цієї функції, яку назвемо `secure_withdraw`.

Ця функція буде ідентична до `insecure_withdraw`, але ми використаємо обмежувач `has_one` у структурі валідації акаунту (`SecureWithdraw`) для того, щоб перевірити, що `authority` акаунту, який прошов у функцію, співпадає з `authority` акаунту у `vault`. Цього разу тільки уповноважений акаунт зможе вивести кошти з сховища.

```rust
use anchor_lang::prelude::*;
use anchor_spl::token::{self, Mint, Token, TokenAccount};

declare_id!("Fg6PaFpoGXkYsidMpWTK6W2BeZ7FEfcYkg476zPFsLnS");

#[program]
pub mod account_data_matching {
    use super::*;
    ...
    pub fn secure_withdraw(ctx: Context<SecureWithdraw>) -> Result<()> {
        let amount = ctx.accounts.token_account.amount;

        let seeds = &[b"vault".as_ref(), &[*ctx.bumps.get("vault").unwrap()]];
        let signer = [&seeds[..]];

        let cpi_ctx = CpiContext::new_with_signer(
            ctx.accounts.token_program.to_account_info(),
            token::Transfer {
                from: ctx.accounts.token_account.to_account_info(),
                authority: ctx.accounts.vault.to_account_info(),
                to: ctx.accounts.withdraw_destination.to_account_info(),
            },
            &signer,
        );

        token::transfer(cpi_ctx, amount)?;
        Ok(())
    }
}

#[derive(Accounts)]
pub struct SecureWithdraw<'info> {
    #[account(
        seeds = [b"vault"],
        bump,
        has_one = token_account,
        has_one = authority,
        has_one = withdraw_destination,

    )]
    pub vault: Account<'info, Vault>,
    #[account(
        mut,
        seeds = [b"token"],
        bump,
    )]
    pub token_account: Account<'info, TokenAccount>,
    #[account(mut)]
    pub withdraw_destination: Account<'info, TokenAccount>,
    pub token_program: Program<'info, Token>,
    pub authority: Signer<'info>,
}
```

### 4. Тест функції `secure_withdraw`

А тепер давайте зробимо тест функції `secure_withdraw` двома шляхами: один в них використовує `walletFake`, а інший `wallet`. Ми очікуємо, що перший не спрацює і виведе помилку, а другий буде виконан успішно.

```tsx
describe("account-data-matching", () => {
  ...
  it("Secure withdraw, expect error", async () => {
    try {
      const tx = await program.methods
        .secureWithdraw()
        .accounts({
          vault: vaultPDA,
          tokenAccount: tokenPDA,
          withdrawDestination: withdrawDestinationFake,
          authority: walletFake.publicKey,
        })
        .transaction()

      await anchor.web3.sendAndConfirmTransaction(connection, tx, [walletFake])
    } catch (err) {
      expect(err)
      console.log(err)
    }
  })

  it("Secure withdraw", async () => {
    await spl.mintTo(
      connection,
      wallet.payer,
      mint,
      tokenPDA,
      wallet.payer,
      100
    )

    await program.methods
      .secureWithdraw()
      .accounts({
        vault: vaultPDA,
        tokenAccount: tokenPDA,
        withdrawDestination: withdrawDestination,
        authority: wallet.publicKey,
      })
      .rpc()

    const balance = await connection.getTokenAccountBalance(tokenPDA)
    expect(balance.value.uiAmount).to.eq(0)
  })
})
```

Запустіть `anchor test` щоб побачити, що транзакція, яка зроблена за допомогою невірного акаунту зараз повертає Анчорну помилку, а та, котра використовує вірний акаунт - виконується без проблем.

```bash
'Program Fg6PaFpoGXkYsidMpWTK6W2BeZ7FEfcYkg476zPFsLnS invoke [1]',
'Program log: Instruction: SecureWithdraw',
'Program log: AnchorError caused by account: vault. Error Code: ConstraintHasOne. Error Number: 2001. Error Message: A has one constraint was violated.',
'Program log: Left:',
'Program log: DfLZV18rD7wCQwjYvhTFwuvLh49WSbXFeJFPQb5czifH',
'Program log: Right:',
'Program log: 5ovvmG5ntwUC7uhNWfirjBHbZD96fwuXDMGXiyMwPg87',
'Program Fg6PaFpoGXkYsidMpWTK6W2BeZ7FEfcYkg476zPFsLnS consumed 10401 of 200000 compute units',
'Program Fg6PaFpoGXkYsidMpWTK6W2BeZ7FEfcYkg476zPFsLnS failed: custom program error: 0x7d1'
```

Зверніть увагу, що Анчор уточнює у консолі той акаунт, який і спричинив помилку (`AnchorError caused by account: vault`).

```bash
✔ Secure withdraw, expect error (77ms)
✔ Secure withdraw (10073ms)
```

І саме так ви закрили прогалину у безпеці системи. Головний момент у більшості таких потенційних подвигів полягає у тому, що вони доволі прості. Тим паче, чим більше та складніше ваша програма - тим легше стає пропустити можливі дірки у безпеці. Доволі гарною звичкою є написання тестів з умовою, яка  *не повинна* працювати. Чим більше - тим краще. Таким чином ви знаходите проблеми ще до кінцевого запуску проекту в світ.

Якщо ви хочете подивитись на фінальний варіант коду, ви можете знайти його у гілці `solution` [цієї репозиторії](https://github.com/Unboxed-Software/solana-account-data-matching/tree/solution).

# Challenge

Так само, як і в інших уроках, ваша відповідальність полягає в тому, щоб ви практикувалися уникати цих дір безпеки у своїх та чужих проектах.

Виділіть трохи часу, щоб подивитись хоча б на одну програму і переконатися, що там присутня правильна перевірка даних для запобігання взлому.

Пам'ятайте! Якщо ви знайшли помилку у чужій програмі - швидко попередьте її власника про це. Якщо знайшли у себе - полагодьте якнайшвидше.
