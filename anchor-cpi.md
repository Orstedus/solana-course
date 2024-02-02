---
Заголовок: Anchor CPI та помилки в них
Цілі:
- Реалізувати міжпрограмні виклики (CPIs) з програми на Anchor
- Використати `cpi`, щоб згенерувати допоміжні функції для виклику методів у існуючій програмі на Anchor.
- Використати `invoke` та `invoke_signed`, щоб залучити CPIs там, де допоміжні функції CPI не доступні.
- Генерувати та повертати настроювані помилки.
---

# Зміст

- Anchor надає спрощенний шлях для створення CPI, використовуючи **`CpiContext`**
- Acnhor функція **`cpi`** створює допоміжні функції CPI для виклику методів у існуючій Anchor програмі.
- Якщо у вас немає доступу до допоміжних функцій CPI, ви все ще можете беспосередньо використовувати `invoke` та `invoke_signed`
- Макрос атрібуту **`error_code`** використовується для створення настроюваних анкорних помилок is used to create custom Anchor Errors

# Загалом

Якщо ви повернетеся до [першого уроку по CPI](cpi), ви можете пам'ятати, що розробка CPI на звичайному Rust може буде дуже непередбачуваною. Саме тому Anchor робить її трохи простішою, особливо коли програма, котру ви викликаєте, є програмою Anchor, до якої ви маєте доступ.

У цьому уроці ви дізнаєтесь як розробляти Anchor CPI. Крім того, ви дізнаєтесь як розробити настроювані помилки у Anchor, тож ви зможете писати більш витончений код на ньому.

## Міжпрограмні виклики (CPI) у Anchor

CPI дозволяє програмам викликати методи в інших програмах, використовуючи функції `invoke` та `invoke_signed`. Це дозволяє новим програмам будуватися на основі вже існуючих програмах (ми називаємо це композиційність).

Незважаючи на те, що створення CPI за допомогою `invoke` або `invoke_signed` є шляхом, Anchor також надає спрощений спосіб створення CPI, використовуючи `CpiContext`.

У цьому уроці ви будете вмикористовувати `anchor_spl`, щоб зробити CPI для програми токенів SPL. Також [ви можете дізнатися що доступно у `anchor_spl`](https://docs.rs/anchor-spl/latest/anchor_spl/#).

### `CpiContext`

Перший крок у створенні CPI це створення екземпляру `CpiContext`. `CpiContext` дуже схожий на `Context`, перший тип аргументу, який потрібен у функціях Anchor. Вони обидва продекларовані у одному і тому самому модулі та мають схожий функціонал.

Тип `CpiContext` запроваждує введення у міжпрограмних викликах без аргументів:

- `accounts` - список уккаунтів, який потрібен для виклику функції
- `remaining_accounts` - аккаунти, які залишилися
- `program` - ID програми, яка викликається
- `signer_seeds` - якщо PDA підписує, то введіть значення, які потрібні PDA

```rust
pub struct CpiContext<'a, 'b, 'c, 'info, T>
where
    T: ToAccountMetas + ToAccountInfos<'info>,
{
    pub accounts: T,
    pub remaining_accounts: Vec<AccountInfo<'info>>,
    pub program: AccountInfo<'info>,
    pub signer_seeds: &'a [&'b [&'c [u8]]],
}
```

Використайте `CpiContext::new` щоб створити новий екземпляр для передачі оригінального підпису транзакції

```rust
CpiContext::new(cpi_program, cpi_accounts)
```

```rust
pub fn new(
        program: AccountInfo<'info>,
        accounts: T
    ) -> Self {
    Self {
        accounts,
        program,
        remaining_accounts: Vec::new(),
        signer_seeds: &[],
    }
}
```

Використайте `CpiContext::new_with_signer`, щоб свторити новий екземпляр, коли здійснюється підпис від імені PDA для CPI.

```rust
CpiContext::new_with_signer(cpi_program, cpi_accounts, seeds)
```

```rust
pub fn new_with_signer(
    program: AccountInfo<'info>,
    accounts: T,
    signer_seeds: &'a [&'b [&'c [u8]]],
) -> Self {
    Self {
        accounts,
        program,
        signer_seeds,
        remaining_accounts: Vec::new(),
    }
}
```

### Акаунти CPI

Одна з найголовніших речей у `CpiContext`, який спрощує міжпрограмні виклики це те, що аргумент `accounts` це загальний тип, який дозволяє вставити його в будь-який об'єкт, який приймає властивості `ToAccountMetas` та `ToAccountInfos<'info>`.

Ці властивості додані атрібутним макросом `#[derive(Accounts)]`, який ви використовували раніше для того, щоб представити функції акаунтів. Це означає, що ви можете використовувати схожі структури з `CpiContext`.

Це допоможе вам з організацією коду та безпекою типів.

### Виклик функції в іншій програмі Anchor

Коли програма, яку ви викликаєте, є опублікованою програмою Anchor, Anchor Може згенерувати створювач функцій та допоміжні функції CPI для вас.

Просто задекларуйте залежніть вашої програми від програми, яку ви викликаєте, у файлі `Cargo.toml`, як тут:

```
[dependencies]
callee = { path = "../callee", features = ["cpi"]}
```

Додавши `features = ["cpi"]`, ви активуєте функціонал `cpi`, і ваша програма отримує доступ до модулю `callee::cpi`.

Модуль `cpi` викриває функцію `callee` як функцію Rust, що приймає в якості аргументів `CpiContext` та будь-які додаткові дані. Ці функції використовують той самий формат, що й функції інструкцій у ваших програмах Anchor, лише з `CpiContext` замість `Context`. Модуль `cpi` також викриває структури облікових записів, необхідні для виклику інструкцій.

Наприклад, якщо `callee` має інструкцію `do_something`, яка вимагає облікові записи, визначені в структурі `DoSomething`, ви можете викликати `do_something` наступним чином:

```rust
use anchor_lang::prelude::*;
use callee;
...

#[program]
pub mod lootbox_program {
    use super::*;

    pub fn call_another_program(ctx: Context<CallAnotherProgram>, params: InitUserParams) -> Result<()> {
        callee::cpi::do_something(
            CpiContext::new(
                ctx.accounts.callee.to_account_info(),
                callee::DoSomething {
                    user: ctx.accounts.user.to_account_info()
                }
            )
        )
        Ok(())
    }
}
...
```

### Виклик функції в програмі, яка не є програмою Anchor

Коли програма, яку ви викликаєте, *не* є програмою Anchor, існують два можливі варіанти:

1. Можливо, розробники програми опублікували кейс зі своїми власними допоміжними функціями для виклику їх програми. Наприклад, код `anchor_spl` надає допоміжні функції, які практично ідентичні з точки зору виклику на місці тому, що ви отримаєте з модулем `cpi` програми Anchor. Наприклад, ви можете створити новий токен за допомогою [функції `mint_to`](https://docs.rs/anchor-spl/latest/src/anchor_spl/token.rs.html#36-58) та використовувати структуру облікових записів [`MintTo`](https://docs.rs/anchor-spl/latest/anchor_spl/token/struct.MintTo.html).
    ```rust
    token::mint_to(
        CpiContext::new_with_signer(
            ctx.accounts.token_program.to_account_info(),
            token::MintTo {
                mint: ctx.accounts.mint_account.to_account_info(),
                to: ctx.accounts.token_account.to_account_info(),
                authority: ctx.accounts.mint_authority.to_account_info(),
            },
            &[&[
                "mint".as_bytes(),
                &[*ctx.bumps.get("mint_authority").unwrap()],
            ]]
        ),
        amount,
    )?;
    ```
2. Якщо для програми, чиї функції вам потрібно викликати, не існує допоміжного модуля, ви можете скористатися функціями `invoke` та `invoke_signed`. Насправді, вихідний код допоміжної функції `mint_to`, на яку було посилання вище, показує приклад використання `invoke_signed`, коли задається `CpiContext`. Ви можете слідувати схожому шаблону, якщо вирішите використовувати структуру облікових записів та `CpiContext` для організації та підготовки вашого CPI.
    ```rust
    pub fn mint_to<'a, 'b, 'c, 'info>(
        ctx: CpiContext<'a, 'b, 'c, 'info, MintTo<'info>>,
        amount: u64,
    ) -> Result<()> {
        let ix = spl_token::instruction::mint_to(
            &spl_token::ID,
            ctx.accounts.mint.key,
            ctx.accounts.to.key,
            ctx.accounts.authority.key,
            &[],
            amount,
        )?;
        solana_program::program::invoke_signed(
            &ix,
            &[
                ctx.accounts.to.clone(),
                ctx.accounts.mint.clone(),
                ctx.accounts.authority.clone(),
            ],
            ctx.signer_seeds,
        )
        .map_err(Into::into)
    }
    ```

## Генерація помилок в Anchor

Ми вже досить глибоко занурилися в Anchor на цьому етапі, тому важливо знати, як створювати власні помилки.

У кінцевому підсумку, всі програми повертають один і той самий тип помилок: [`ProgramError`](https://docs.rs/solana-program/latest/solana_program/program_error/enum.ProgramError.html). Проте, при написанні програми за допомогою Anchor ви можете використовувати `AnchorError` як абстракцію над `ProgramError`. Ця абстракція надає додаткову інформацію при невдачі програми, зокрема:

- Назва та номер помилки
- Місце в коді, де сталася помилка
- Обліковий запис, який порушив обмеження

```rust
pub struct AnchorError {
    pub error_name: String,
    pub error_code_number: u32,
    pub error_msg: String,
    pub error_origin: Option<ErrorOrigin>,
    pub compared_values: Option<ComparedValues>,
}
```

Помилки Anchor можна поділити на:

- Внутрішні помилки Anchor, які фреймворк повертає зі свого власного коду
- Власні помилки, які ви, розробник, можете створити

Ви можете додати унікальні для вашої програми помилки, використовуючи атрибут `error_code`. Просто додайте цей атрибут до власного типу `enum`. Потім ви можете використовувати варіанти `enum` як помилки у вашій програмі. Крім того, ви можете додати повідомлення про помилку до кожного варіанту за допомогою атрибуту `msg`. Клієнти можуть відображати це повідомлення про помилку, якщо помилка виникає.

```rust
#[error_code]
pub enum MyError {
    #[msg("MyAccount may only hold data below 100")]
    DataTooLarge
}
```

Для повернення власної помилки ви можете використовувати макроси [err](https://docs.rs/anchor-lang/latest/anchor_lang/macro.err.html) або [error](https://docs.rs/anchor-lang/latest/anchor_lang/prelude/macro.error.html) з функції інструкції. Вони додають інформацію про файл та рядок до помилки, яка потім реєструється Anchor для допомоги вам з налагодженням.

```rust
#[program]
mod hello_anchor {
    use super::*;
    pub fn set_data(ctx: Context<SetData>, data: MyAccount) -> Result<()> {
        if data.data >= 100 {
            return err!(MyError::DataTooLarge);
        }
        ctx.accounts.my_account.set_inner(data);
        Ok(())
    }
}

#[error_code]
pub enum MyError {
    #[msg("MyAccount may only hold data below 100")]
    DataTooLarge
}
```

Також, ви можете скористатися макросом [require](https://docs.rs/anchor-lang/latest/anchor_lang/macro.require.html), щоб спростити повернення помилок. Код вище може бути перероблений наступним чином:

```rust
#[program]
mod hello_anchor {
    use super::*;
    pub fn set_data(ctx: Context<SetData>, data: MyAccount) -> Result<()> {
        require!(data.data < 100, MyError::DataTooLarge);
        ctx.accounts.my_account.set_inner(data);
        Ok(())
    }
}

#[error_code]
pub enum MyError {
    #[msg("MyAccount may only hold data below 100")]
    DataTooLarge
}
```

# Лабораторна робота

Давайте практикувати концепції, які ми розглянули на цьому уроці, розширивши програму для огляду фільмів з попередніх уроків.

У цій лабораторній роботі ми оновимо програму, щоб видавати токени користувачам, коли вони подають новий відгук про фільм.

### 1. Початок

Для початку ми будемо використовувати кінцевий стан програми Anchor для огляду фільмів з попереднього уроку. Отже, якщо ви щойно завершили той урок, то у вас все налаштовано і готово. Якщо ви тільки приєдналися, не хвилюйтеся, ви можете [завантажити початковий код](https://github.com/Unboxed-Software/anchor-movie-review-program/tree/solution-pdas). Ми використовуватимемо гілку `solution-pdas` як наш вихідний пункт.

### 2. Додайте залежності до `Cargo.toml`

Перш ніж ми розпочнемо, нам потрібно активувати функціонал `init-if-needed` та додати `anchor-spl` до залежностей у файлі `Cargo.toml`. Якщо вам потрібно освіжити функціонал `init-if-needed`, перегляньте [урок Anchor PDA та акаунти](anchor-pdas).

```rust
[dependencies]
anchor-lang = { version = "0.25.0", features = ["init-if-needed"] }
anchor-spl = "0.25.0"
```

### 3. Ініціалізація токену винагороди

Далі перейдіть до `lib.rs` та створіть інструкцію для ініціалізації нового мінта токенів. Це буде токен, який видаватиметься кожного разу, коли користувач залишає відгук. Зверніть увагу, що нам не потрібно включати жодної власної логіки інструкції, оскільки ініціалізацію можна повністю обробити за допомогою обмежень Anchor.

```rust
pub fn initialize_token_mint(_ctx: Context<InitializeMint>) -> Result<()> {
    msg!("Token mint initialized");
    Ok(())
}
```

Тепер реалізуйте тип контексту `InitializeMint` та перерахуйте облікові записи і обмеження, які вимагає інструкція. Тут ми ініціалізуємо новий обліковий запис `Mint`, використовуючи PDA з рядком "mint" як початкові дані. Зверніть увагу, що ми можемо використовувати той самий PDA як для адреси облікового запису `Mint`, так і для прав на мінтування. Використання PDA як права на мінтування дозволяє нашій програмі підписувати мінтування токенів.

Для ініціалізації облікового запису `Mint` нам потрібно включити `token_program`, `rent` та `system_program` до списку облікових записів.


```rust
#[derive(Accounts)]
pub struct InitializeMint<'info> {
    #[account(
        init,
        seeds = ["mint".as_bytes()],
        bump,
        payer = user,
        mint::decimals = 6,
        mint::authority = mint,
    )]
    pub mint: Account<'info, Mint>,
    #[account(mut)]
    pub user: Signer<'info>,
    pub token_program: Program<'info, Token>,
    pub rent: Sysvar<'info, Rent>,
    pub system_program: Program<'info, System>
}
```

Можливо, є деякі обмеження вище, які ви ще не бачили. Додавання `mint::decimals` та `mint::authority` разом із `init` забезпечує, що обліковий запис ініціалізується як новий мінт токенів з відповідними десятковими розрядами та правами на мінтування.

### 4. Помилка Anchor

Далі давайте створимо помилку Anchor, яку ми використовуватимемо при перевірці `rating`, що передається до функції `add_movie_review` або `update_movie_review`.


```rust
#[error_code]
enum MovieReviewError {
    #[msg("Rating must be between 1 and 5")]
    InvalidRating
}
```

### 5. Оновлення функції `add_movie_review`

Тепер, коли ми вже зробили деяку підготовку, давайте оновимо функцію `add_movie_review` та тип контексту `AddMovieReview`, щоб мінтувати токени для рецензента.

Далі оновіть тип контексту `AddMovieReview`, щоб додати наступні облікові записи:

- `token_program` - ми будемо використовувати програму Token для мінтування токенів
- `mint` - обліковий запис мінта для токенів, які ми будемо мінтувати користувачам під час додавання відгуку про фільм
- `token_account` - пов'язаний обліковий запис токена для зазначеного вище `mint` та рецензента
- `associated_token_program` - потрібно, оскільки ми будемо використовувати обмеження `associated_token` для `token_account`
- `rent` - потрібно, оскільки ми використовуємо обмеження `init-if-needed` для `token_account`


```rust
#[derive(Accounts)]
#[instruction(title: String, description: String)]
pub struct AddMovieReview<'info> {
    #[account(
        init,
        seeds=[title.as_bytes(), initializer.key().as_ref()],
        bump,
        payer = initializer,
        space = 8 + 32 + 1 + 4 + title.len() + 4 + description.len()
    )]
    pub movie_review: Account<'info, MovieAccountState>,
    #[account(mut)]
    pub initializer: Signer<'info>,
    pub system_program: Program<'info, System>,
    // Нижче додані акаунти
    pub token_program: Program<'info, Token>,
    #[account(
        seeds = ["mint".as_bytes()]
        bump,
        mut
    )]
    pub mint: Account<'info, Mint>,
    #[account(
        init_if_needed,
        payer = initializer,
        associated_token::mint = mint,
        associated_token::authority = initializer
    )]
    pub token_account: Account<'info, TokenAccount>,
    pub associated_token_program: Program<'info, AssociatedToken>,
    pub rent: Sysvar<'info, Rent>
}
```

Знову ж таки, деякі з вищезазначених обмежень можуть бути вам незнайомі. Обмеження `associated_token::mint` та `associated_token::authority`, разом із обмеженням `init_if_needed`, забезпечує, що якщо обліковий запис ще не був ініціалізований, він буде ініціалізований як пов'язаний обліковий запис токена для зазначеного мінта та прав.

Далі давайте оновимо функцію `add_movie_review`, щоб вона робила наступне:

- Перевіряємо, що `rating` є дійсним. Якщо це не дійсний рейтинг, повертаємо помилку `InvalidRating`.
- Виконуємо CPI до інструкції `mint_to` програми токенів, використовуючи PDA прав мінтування як підписанта. Зауважте, що ми мінтуємо 10 токенів користувачу, але потрібно врахувати десяткові розряди мінта, роблячи це `10*10^6`.

На щастя, ми можемо використовувати `anchor_spl`, щоб отримати доступ до допоміжних функцій та типів, таких як `mint_to` та `MintTo`, для побудови нашого CPI до програми токенів. `mint_to` приймає контекст `CpiContext` та ціле число як аргументи, де ціле число представляє кількість токенів для мінтування. `MintTo` може бути використаний для списку облікових записів, які потрібні для інструкції мінту.

```rust
pub fn add_movie_review(ctx: Context<AddMovieReview>, title: String, description: String, rating: u8) -> Result<()> {
    msg!("Movie review account created");
    msg!("Title: {}", title);
    msg!("Description: {}", description);
    msg!("Rating: {}", rating);

    require!(rating >= 1 && rating <= 5, MovieReviewError::InvalidRating);

    let movie_review = &mut ctx.accounts.movie_review;
    movie_review.reviewer = ctx.accounts.initializer.key();
    movie_review.title = title;
    movie_review.description = description;
    movie_review.rating = rating;

    mint_to(
        CpiContext::new_with_signer(
            ctx.accounts.token_program.to_account_info(),
            MintTo {
                authority: ctx.accounts.mint.to_account_info(),
                to: ctx.accounts.token_account.to_account_info(),
                mint: ctx.accounts.mint.to_account_info()
            },
            &[&[
                "mint".as_bytes(),
                &[*ctx.bumps.get("mint").unwrap()]
            ]]
        ),
        10*10^6
    )?;

    msg!("Minted tokens");

    Ok(())
}
```

### 6. Оновлення функції `update_movie_review`

Тут ми тільки додаємо перевірку того, що `rating` є дійсним.

```rust
pub fn update_movie_review(ctx: Context<UpdateMovieReview>, title: String, description: String, rating: u8) -> Result<()> {
    msg!("Movie review account space reallocated");
    msg!("Title: {}", title);
    msg!("Description: {}", description);
    msg!("Rating: {}", rating);

    require!(rating >= 1 && rating <= 5, MovieReviewError::InvalidRating);

    let movie_review = &mut ctx.accounts.movie_review;
    movie_review.description = description;
    movie_review.rating = rating;

    Ok(())
}
```

### 7. Тест

Отже, це всі зміни, які нам потрібно внести до програми! Тепер давайте оновимо наші тести.

Почніть з того, щоб переконатися, що ваші імпорти та функція `describe` виглядають так:

```typescript
import * as anchor from "@project-serum/anchor"
import { Program } from "@project-serum/anchor"
import { expect } from "chai"
import { getAssociatedTokenAddress, getAccount } from "@solana/spl-token"
import { AnchorMovieReviewProgram } from "../target/types/anchor_movie_review_program"

describe("anchor-movie-review-program", () => {
  // Налаштуйте клієнт на роботу з локальним кластером
  const provider = anchor.AnchorProvider.env()
  anchor.setProvider(provider)

  const program = anchor.workspace
    .AnchorMovieReviewProgram as Program<AnchorMovieReviewProgram>

  const movie = {
    title: "Just a test movie",
    description: "Wow what a good movie it was real great",
    rating: 5,
  }

  const [movie_pda] = anchor.web3.PublicKey.findProgramAddressSync(
    [Buffer.from(movie.title), provider.wallet.publicKey.toBuffer()],
    program.programId
  )

  const [mint] = anchor.web3.PublicKey.findProgramAddressSync(
    [Buffer.from("mint")],
    program.programId
  )
...
}
```

Зробивши це, додайте тест для інструкції `initializeTokenMint`:

```typescript
it("Initializes the reward token", async () => {
    const tx = await program.methods.initializeTokenMint().rpc()
})
```

Зверніть увагу, що нам не потрібно додавати `.accounts`, оскільки вони можуть бути виведені, включаючи обліковий запис `mint` (якщо у вас ввімкнута автоінференція даних).

Далі оновіть тест для інструкції `addMovieReview`. Основні додатки:
1. Отримати адресу пов'язаного токену, яка повинна бути передана в інструкцію як обліковий запис, який не може бути виведений
2. Перевірте в кінці тесту, що обліковий запис пов'язаного токену має 10 токенів

```typescript
it("Movie review is added`", async () => {
  const tokenAccount = await getAssociatedTokenAddress(
    mint,
    provider.wallet.publicKey
  )
  
  const tx = await program.methods
    .addMovieReview(movie.title, movie.description, movie.rating)
    .accounts({
      tokenAccount: tokenAccount,
    })
    .rpc()
  
  const account = await program.account.movieAccountState.fetch(movie_pda)
  expect(movie.title === account.title)
  expect(movie.rating === account.rating)
  expect(movie.description === account.description)
  expect(account.reviewer === provider.wallet.publicKey)

  const userAta = await getAccount(provider.connection, tokenAccount)
  expect(Number(userAta.amount)).to.equal((10 * 10) ^ 6)
})
```

Після цього тест ні для `updateMovieReview`, ні для `deleteMovieReview` не потребує жодних змін.

На цьому етапі запустіть `anchor test`, і ви повинні побачити наступний вивід

```console
anchor-movie-review-program
    ✔ Initializes the reward token (458ms)
    ✔ Movie review is added (410ms)
    ✔ Movie review is updated (402ms)
    ✔ Deletes a movie review (405ms)

  5 passing (2s)
```

Якщо вам потрібен більше часу з концепціями з цього уроку або ви застрягли по дорозі, не соромтеся [заглянути до коду рішення](https://github.com/Unboxed-Software/anchor-movie-review-program/tree/solution-add-tokens). Зверніть увагу, що рішення цієї лабораторної роботи знаходиться на гілці `solution-add-tokens`.

# Виклик

Щоб застосувати те, що ви вивчили про CPI в цьому уроці, подумайте, як ви можете використовувати їх у програмі "Вступ для студентів". Ви можете зробити щось подібне до того, що ми зробили у лабораторній роботі тут і додати функціональність для мінтування токенів користувачам, коли вони представляються.

Спробуйте зробити це самостійно, якщо можете! Але якщо ви застрягли, не соромтеся звернутися до цього [коду рішення](https://github.com/Unboxed-Software/anchor-student-intro-program/tree/cpi).

## Завершили лабораторну роботу?

Завантажте ваш код на GitHub і [розкажіть нам, що ви думали про цей урок](https://form.typeform.com/to/IPH0UGz7#answers-lesson=21375c76-b6f1-4fb6-8cc1-9ef151bc5b0a)!
