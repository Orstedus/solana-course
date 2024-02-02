---
Заголовок: Довільний CPI
Цілі:
- Поясніть ризики безпеки, пов'язані з викликом CPI до невідомої програми
- Покажіть, як модуль CPI Anchor запобігає цьому, коли ви викликаєте CPI з однієї програми Anchor в іншу
- Безпечно та надійно здійсніть CPI з програми Anchor в довільну не-Anchor програму
---

# Зміст

- Для генерації CPI цільова програма повинна бути передана в інструкцію виклику як обліковий запис. Це означає, що в інструкцію може бути передана будь-яка цільова програма. Ваша програма повинна перевіряти неправильні або неочікувані програми.
- Виконуйте перевірки програм у нативних програмах, просто порівнюючи публічний ключ переданої програми з програмою, яку ви очікували.
- Якщо програма написана на Anchor, то вона може мати публічно доступний модуль CPI. Це спрощує та забезпечує безпеку виклику програми з іншої програми Anchor. Модуль CPI Anchor автоматично перевіряє, що адреса програми, передана відповідно до адреси програми, збереженої в модулі.

# Огляд

Міжпрограмний виклик (CPI) - це коли одна програма викликає інструкцію в іншій програмі. "Довільний CPI" - це коли програма структурована для видачі CPI до будь-якої програми, яка передається в інструкцію, а не для виклику CPI до однієї конкретної програми. Оскільки викликачі інструкцій вашої програми можуть передавати будь-яку програму, яку вони бажають у список облікових записів інструкції, не вдаючись до перевірки адреси переданої програми, результатом буде виконання CPI до довільних програм.

Цей відсутність перевірок програм створює можливість для зловмисного користувача передати програму, відмінну від очікуваної, що призводить до того, що початкова програма викликає інструкцію в цю загадкову програму. Неможливо передбачити наслідки цього CPI. Це залежить від логіки програми (як оригінальної, так і неочікуваної), а також від того, які інші облікові записи передаються в оригінальну інструкцію.

## Відсутність перевірок програм

Візьміть наступну програму як приклад. Інструкція `cpi` викликає інструкцію `transfer` в програмі `token_program`, але немає коду, який перевіряє, чи є обліковий запис `token_program`, переданий в інструкцію, насправді програмою SPL Token.

```rust
use anchor_lang::prelude::*;
use anchor_lang::solana_program;

declare_id!("Fg6PaFpoGXkYsidMpWTK6W2BeZ7FEfcYkg476zPFsLnS");

#[program]
pub mod arbitrary_cpi_insecure {
    use super::*;

    pub fn cpi(ctx: Context<Cpi>, amount: u64) -> ProgramResult {
        solana_program::program::invoke(
            &spl_token::instruction::transfer(
                ctx.accounts.token_program.key,
                ctx.accounts.source.key,
                ctx.accounts.destination.key,
                ctx.accounts.authority.key,
                &[],
                amount,
            )?,
            &[
                ctx.accounts.source.clone(),
                ctx.accounts.destination.clone(),
                ctx.accounts.authority.clone(),
            ],
        )
    }
}

#[derive(Accounts)]
pub struct Cpi<'info> {
    source: UncheckedAccount<'info>,
    destination: UncheckedAccount<'info>,
    authority: UncheckedAccount<'info>,
    token_program: UncheckedAccount<'info>,
}
```

Хакер може легко викликати цю інструкцію та передати дублікат програми токенів, яку вони створили та контролюють.

## Додавання перевірку програми

Цю вразливість можна виправити, просто додавши кілька рядків до інструкції `cpi`, щоб перевірити, чи публічний ключ `token_program` належить програмі SPL Token.


```rust
pub fn cpi_secure(ctx: Context<Cpi>, amount: u64) -> ProgramResult {
    if &spl_token::ID != ctx.accounts.token_program.key {
        return Err(ProgramError::IncorrectProgramId);
    }
    solana_program::program::invoke(
        &spl_token::instruction::transfer(
            ctx.accounts.token_program.key,
            ctx.accounts.source.key,
            ctx.accounts.destination.key,
            ctx.accounts.authority.key,
            &[],
            amount,
        )?,
        &[
            ctx.accounts.source.clone(),
            ctx.accounts.destination.clone(),
            ctx.accounts.authority.clone(),
        ],
    )
}
```

Тепер, якщо хакер передасть іншу програму токенів, інструкція поверне помилку `ProgramError::IncorrectProgramId`.

Залежно від програми, яку ви викликаєте за допомогою вашого CPI, ви можете або захардкодити адресу очікуваного ідентифікатора програми, або використовувати кристал Rust програми, щоб отримати адресу програми, якщо це можливо. У вищезазначеному прикладі `spl_token` надає адресу програми SPL Token.

## Використання модуля CPI Anchor

Простіший спосіб керувати перевірками програми - використовувати модулі CPI Anchor. Ми вивчили в [попередньому уроці](https://github.com/Unboxed-Software/solana-course/blob/main/content/anchor-cpi), що Anchor може автоматично генерувати модулі CPI, щоб спростити CPI у програму. Ці модулі також підвищують безпеку, перевіряючи публічний ключ програми, яка передається в одну з її публічних інструкцій.

Кожна програма Anchor використовує макрос `declare_id()` для визначення адреси програми. Коли для певної програми генерується модуль CPI, він використовує адресу, передану у цей макрос, як "джерело правди" і автоматично перевіряє, що всі CPI, виконані за допомогою її модуля CPI, націлені на цей ідентифікатор програми.

Хоча на самому ядрі нічим не відрізняється від ручних перевірок програм, використання модулів CPI дозволяє уникнути можливості забути виконати перевірку програми або випадково ввести неправильний ідентифікатор програми під час захардкодування його.

Програма нижче показує приклад використання модуля CPI для програми SPL Token для виконання трансферу, показаного в попередніх прикладах.


```rust
use anchor_lang::prelude::*;
use anchor_spl::token::{self, Token, TokenAccount};

declare_id!("Fg6PaFpoGXkYsidMpWTK6W2BeZ7FEfcYkg476zPFsLnS");

#[program]
pub mod arbitrary_cpi_recommended {
    use super::*;

    pub fn cpi(ctx: Context<Cpi>, amount: u64) -> ProgramResult {
        token::transfer(ctx.accounts.transfer_ctx(), amount)
    }
}

#[derive(Accounts)]
pub struct Cpi<'info> {
    source: Account<'info, TokenAccount>,
    destination: Account<'info, TokenAccount>,
    authority: Signer<'info>,
    token_program: Program<'info, Token>,
}

impl<'info> Cpi<'info> {
    pub fn transfer_ctx(&self) -> CpiContext<'_, '_, '_, 'info, token::Transfer<'info>> {
        let program = self.token_program.to_account_info();
        let accounts = token::Transfer {
            from: self.source.to_account_info(),
            to: self.destination.to_account_info(),
            authority: self.authority.to_account_info(),
        };
        CpiContext::new(program, accounts)
    }
}
```

Зверніть увагу, що, подібно до прикладу вище, Anchor створив кілька [обгорток для популярних внутрішніх програм](https://github.com/coral-xyz/anchor/tree/master/spl/src), які дозволяють вам викликати CPI в них так, якщо б вони були програмами Anchor.

Додатково, залежно від програми, до якої ви виконуєте CPI, ви можете використовувати тип облікового запису Anchor [`Program`](https://docs.rs/anchor-lang/latest/anchor_lang/accounts/program/struct.Program.html), щоб перевірити передану програму в вашій структурі валідації облікового запису. Між [`anchor_lang`](https://docs.rs/anchor-lang/latest/anchor_lang) та [`anchor_spl`](https://docs.rs/anchor_spl/latest/), наступні типи `Program` надаються з:

- [`System`](https://docs.rs/anchor-lang/latest/anchor_lang/struct.System.html)
- [`AssociatedToken`](https://docs.rs/anchor-spl/latest/anchor_spl/associated_token/struct.AssociatedToken.html)
- [`Token`](https://docs.rs/anchor-spl/latest/anchor_spl/token/struct.Token.html)

Якщо у вас є доступ до модуля CPI Anchor програми, ви зазвичай можете імпортувати її тип програми з наступним, заміняючи назву програми на фактичну назву програми:

```rust
use other_program::program::OtherProgram;
```

# Лабораторна робота

Щоб продемонструвати важливість перевірки програми, яку ви використовуєте для CPI, ми працюватимемо з спрощеною та трохи придуманою грою. Ця гра представляє персонажів з обліковими записами PDA та використовує окрему програму "метаданих", щоб керувати метаданими персонажа та атрибутами, такими як здоров'я та потужність.

Хоча цей приклад трохи умовний, він фактично є практично таким самим архітектурним рішенням, як працюють NFT на Solana: програма управління токенами SPL керує мінтами токенів, розподілом та передачею, а окрема програма метаданих використовується для призначення метаданих токенам. Таким чином, вразливість, про яку ми розповідаємо тут, також може бути застосована до реальних токенів.

### 1. Підготовка

Ми розпочнемо з гілки `starter` у [цьому репозиторії](https://github.com/Unboxed-Software/solana-arbitrary-cpi/tree/starter). Клонуйте репозиторій, а потім відкрийте його на гілці `starter`.

Зверніть увагу, що є три програми:

1. `gameplay`
2. `character-metadata`
3. `fake-metadata`

Додатково, вже є тест у каталозі `tests`.

Перша програма, `gameplay`, - це та програма, яку безпосередньо використовує наш тест. Подивіться на програму. Вона має дві інструкції:

1. `create_character_insecure` - створює нового персонажа та викликає CPI в програму метаданих, щоб налаштувати початкові атрибути персонажа
2. `battle_insecure` - ставить двох персонажів одного проти одного, призначаючи "перемогу" персонажу з найвищими атрибутами

Друга програма, `character-metadata`, призначена для "схваленої" програми для обробки метаданих персонажа. Подивіться на цю програму. У неї є одна інструкція для `create_metadata`, яка створює новий PDA та призначає псевдовипадкове значення від 0 до 20 для здоров'я та потужності персонажа.

Остання програма, `fake-metadata`, є "фальшивою" програмою метаданих, яка призначена для ілюстрації того, що атакувальник може створити для експлуатації нашої програми `gameplay`. Ця програма практично ідентична програмі метаданих `character-metadata`, тільки вона призначає початкове здоров'я та потужність персонажа для максимально допустимих: 255.

### 2. Тест інструкції `create_character_insecure`

У каталозі `tests` вже є тест для цього. Він довгий, але витратьте хвилину, щоб розглянути його перед тим, як ми запустимо його разом:


```typescript
it("Insecure instructions allow attacker to win every time", async () => {
    // Ініціалізуємо гравця з реальними метаданими
    await gameplayProgram.methods
      .createCharacterInsecure()
      .accounts({
        metadataProgram: metadataProgram.programId,
        authority: playerOne.publicKey,
      })
      .signers([playerOne])
      .rpc()

    // Ініціалізуємо хакера з його фальшивими метаданими
    await gameplayProgram.methods
      .createCharacterInsecure()
      .accounts({
        metadataProgram: fakeMetadataProgram.programId,
        authority: attacker.publicKey,
      })
      .signers([attacker])
      .rpc()

    // Отримуємо дані акаунту обидвох гравців
    const [playerOneMetadataKey] = getMetadataKey(
      playerOne.publicKey,
      gameplayProgram.programId,
      metadataProgram.programId
    )

    const [attackerMetadataKey] = getMetadataKey(
      attacker.publicKey,
      gameplayProgram.programId,
      fakeMetadataProgram.programId
    )

    const playerOneMetadata = await metadataProgram.account.metadata.fetch(
      playerOneMetadataKey
    )

    const attackerMetadata = await fakeMetadataProgram.account.metadata.fetch(
      attackerMetadataKey
    )

    // Звичайний гравець повинен мати здоров'я від 0 до 20
    expect(playerOneMetadata.health).to.be.lessThan(20)
    expect(playerOneMetadata.power).to.be.lessThan(20)

    // Хакер буде мати силу на здоровья у 255
    expect(attackerMetadata.health).to.equal(255)
    expect(attackerMetadata.power).to.equal(255)
})
```

Цей тест описує сценарій, в якому звичайний гравець та хакер обидва створюють своїх персонажів. Тільки хакер передає ідентифікатор програми фальшивих метаданих, а не фактичної програми метаданих. І оскільки інструкція `create_character_insecure` не має перевірок програми, вона все ще виконується.

Результатом є те, що у звичайного персонажа правильна кількість здоров'я та потужності: кожен з них має значення від 0 до 20. Але здоров'я та потужність хакера - кожен по 255, що робить атакувальника непереможним.

Якщо ви ще цього не зробили, запустіть `anchor test`, щоб перевірити, чи відбувається тест так, як описано.

### 3. Створення інструкції `create_character_secure`

Давайте виправимо це, створивши безпечну інструкцію для створення нового персонажа. Ця інструкція повинна виконувати належні перевірки програм та використовувати програму `character-metadata` для виконання CPI замість простого використання `invoke`.

Якщо ви хочете випробувати свої навички, спробуйте це самостійно, перш ніж продовжити далі.

Ми почнемо з оновлення нашого оператора `use` у верхній частині файлу `lib.rs` програми `gameplay`. Ми дамо собі доступ до типу програми для перевірки облікового запису та допоміжної функції для видачі CPI `create_metadata`.


```rust
use character_metadata::{
    cpi::accounts::CreateMetadata,
    cpi::create_metadata,
    program::CharacterMetadata,
};
```

Далі давайте створимо нову структуру перевірки облікового запису з назвою `CreateCharacterSecure`. Цього разу ми робимо `metadata_program` типом `Program`:

```rust
#[derive(Accounts)]
pub struct CreateCharacterSecure<'info> {
    #[account(mut)]
    pub authority: Signer<'info>,
    #[account(
        init,
        payer = authority,
        space = 8 + 32 + 32 + 64,
        seeds = [authority.key().as_ref()],
        bump
    )]
    pub character: Account<'info, Character>,
    #[account(
        mut,
        seeds = [character.key().as_ref()],
        seeds::program = metadata_program.key(),
        bump,
    )]
    /// Перевірки
    pub metadata_account: AccountInfo<'info>,
    pub metadata_program: Program<'info, CharacterMetadata>,
    pub system_program: Program<'info, System>,
}
```

Нарешті, ми додаємо інструкцію `create_character_secure`. Вона буде такою самою, як і раніше, але буде використовувати повний функціонал CPI Anchor замість безпосереднього використання `invoke`:

```rust
pub fn create_character_secure(ctx: Context<CreateCharacterSecure>) -> Result<()> {
    let character = &mut ctx.accounts.character;
    character.metadata = ctx.accounts.metadata_account.key();
    character.auth = ctx.accounts.authority.key();
    character.wins = 0;

    let context = CpiContext::new(
        ctx.accounts.metadata_program.to_account_info(),
        CreateMetadata {
            character: ctx.accounts.character.to_account_info(),
            metadata: ctx.accounts.metadata_account.to_owned(),
            authority: ctx.accounts.authority.to_account_info(),
            system_program: ctx.accounts.system_program.to_account_info(),
        },
    );

    create_metadata(context)?;

    Ok(())
}
```

### 4. Тест `create_character_secure`

Тепер, коли у нас є безпечний спосіб ініціалізації нового персонажа, давайте створимо новий тест. Цей тест просто має спробувати ініціалізувати персонажа Хакера і очікувати, що буде викинута помилка.

```typescript
it("Secure character creation doesn't allow fake program", async () => {
    try {
      await gameplayProgram.methods
        .createCharacterSecure()
        .accounts({
          metadataProgram: fakeMetadataProgram.programId,
          authority: attacker.publicKey,
        })
        .signers([attacker])
        .rpc()
    } catch (error) {
      expect(error)
      console.log(error)
    }
})
```

Якщо ви ще цього не зробили, запустіть `anchor test`. Зверніть увагу, що, як і очікувалося, була викинута помилка, де деталізовано, що ідентифікатор програми, переданий в інструкцію, не є очікуваним ідентифікатором програми:

```bash
'Program log: AnchorError caused by account: metadata_program. Error Code: InvalidProgramId. Error Number: 3008. Error Message: Program ID was not as expected.',
'Program log: Left:',
'Program log: FKBWhshzcQa29cCyaXc1vfkZ5U985gD5YsqfCzJYUBr',
'Program log: Right:',
'Program log: D4hPnYEsAx4u3EQMrKEXsY3MkfLndXbBKTEYTwwm25TE'
```

Це завершує процес захисту від довільних CPI!

Хоча можуть бути випадки, коли вам потрібна більша гнучкість у CPI вашої програми, важливо вживати всі можливі заходи для запобігання уразливостям у вашому коді.

Якщо вас цікавить ознайомлення з остаточним кодом, ви можете знайти його на гілці `solution` [в тому ж репозиторії](https://github.com/Unboxed-Software/solana-arbitrary-cpi/tree/solution).

# Виклик

Як і з іншими уроками цього розділу, ви маєте можливість попрактикуватися у уникненні цього вразливості з точки зору безпеки, аналізуючи власні або інші програми.

Візьміть трохи часу, щоб переглянути принаймні одну програму і переконатися, що перевірки програми виконуються для кожної програми, переданої в інструкції, особливо тих, які викликаються через CPI.

Пам'ятайте, якщо ви виявите помилку або вразливість у програмі когось іншого, будь ласка, повідомте його! Якщо ви знайдете її у власній програмі, обов'язково виправте її якнайшвидше.

## Завершили лабораторну роботу?

Завантажте свій код на GitHub і [поділіться своїми враженнями від цього уроку](https://form.typeform.com/to/IPH0UGz7#answers-lesson=5bcaf062-c356-4b58-80a0-12cca99c29b0)!

