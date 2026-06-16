# Solana Mini Launchpad

Учебный мини-лаунчпад на Solana + Anchor: два on-chain контракта (SOL/USD oracle и token minter), Rust backend для обновления цены и прослушки событий, а также Remix фронтенд (папка `frontend/`).

## Структура

- `program/` — Anchor workspace
  - `programs/sol_usd_oracle` — хранит цену SOL/USD (decimals = 6)
  - `programs/token_minter` — минтит SPL токены за комиссию в SOL, используя цену из oracle
  - `tests/` — Anchor TS тесты
- `backend/` — Rust сервис, который обновляет цену и слушает события `TokenCreated`
- `frontend/` — Remix hello-world (React Router)

## Быстрый старт (локально)

1. **Validator**: запустить `solana-test-validator` (или `make validator`). Для отображения имени, тикера и картинки токена в кошельке используйте валидатор с клоном Metaplex: `make validator-metaplex` (клон программы Token Metadata с mainnet). Убедитесь, что `~/.config/solana/id.json` есть и профинансирован (`solana airdrop 1000` при необходимости).

2. **Программы**: собрать и задеплоить (ID программ берутся из keypair в `program/target/deploy/`; при первом деплое выполните `anchor keys sync`, затем пересоберите):

   ```bash
   make build
   make deploy
   ```

3. **Инициализация**: один раз после деплоя инициализировать oracle и minter (скрипт выведет `ORACLE_STATE_PUBKEY` для `.env`):
   ```bash
   make init
   ```

## Деплой на Devnet

На фронте есть переключатель **Localnet / Devnet**. Для тестов на devnet:

1. Переключить CLI на devnet и пополнить кошелёк:

   ```bash
   solana config set --url devnet
   solana airdrop 2
   ```

2. Собрать и задеплоить на devnet:

   ```bash
   make deploy-devnet
   ```

3. Инициализировать оракул и минтер на devnet (один раз):

   ```bash
   make init-devnet
   ```

4. В приложении выбрать сеть **Devnet**, в кошельке переключиться на Devnet — можно минтить. На devnet Metaplex уже есть, картинка в кошельке может отображаться (если URI доступен по HTTPS).

5. **Backend**: скопировать `backend/.env.example` в `backend/.env`, подставить `ORACLE_STATE_PUBKEY` из вывода init-скрипта. Путь `BACKEND_KEYPAIR_PATH` поддерживает `~`:

   ```bash
   cd backend
   cargo run
   ```

   Сервис будет периодически вызывать `update_price` и слушать события `TokenCreated`, выводя их в stdout в JSON.

6. **Фронтенд** (опционально):

   ```bash
   cd frontend
   npm install && npm run dev
   ```

   Открыть http://localhost:7001.

7. **Тесты** (LiteSVM, без сети):
   ```bash
   cd program
   anchor test
   ```
   Или `yarn litesvm` для запуска только тестов в `tests/*.litesvm.ts`.

## Переменные окружения для backend

См. `backend/.env.example`. Основные:

- `SOLANA_RPC_HTTP`, `SOLANA_RPC_WS` — RPC локального валидатора или devnet/mainnet.
- `ORACLE_PROGRAM_ID`, `MINTER_PROGRAM_ID` — из `anchor keys list` (после деплоя).
- `ORACLE_STATE_PUBKEY` — PDA от seed `"oracle_state"`; выводится скриптом `program/scripts/init-local.js`.
- `BACKEND_KEYPAIR_PATH` — keypair администратора оракула (поддерживается `~`).
- Опционально: `MOCK_PRICE`, `PRICE_API_URL`, `PRICE_POLL_INTERVAL_SEC`.

## Метаданные токена (Metaplex)

При минте можно передать `name`, `symbol` и `uri` — контракт создаёт запись Metaplex Token Metadata (имя, тикер, картинка в кошельке). Если передать пустое имя, метаданные не создаются (подходит для localnet без Metaplex). Для отображения в кошельке поднимайте валидатор с клоном Metaplex: `make validator-metaplex`, затем деплой и `init` как обычно.

## Основные ограничения

- Все вычисления комиссии — integer math, `fee_lamports = mint_fee_usd * LAMPORTS_PER_SOL / price`.
- Oracle price и mint_fee_usd хранятся с точностью 10^6.
- Доступ к `update_price` только у oracle admin (backend keypair).
- `mint_token` падает, если `price == 0` или fee/supply некорректны.

---

## Порядок запуска (локально)

1. `solana-test-validator`
2. `cd program && anchor build && anchor deploy --provider.cluster localnet`
3. `cd program && node scripts/init-local.js` — скопировать `ORACLE_STATE_PUBKEY` в `backend/.env`
4. `cd backend && cargo run`
5. `cd frontend && npm run dev` — открыть в браузере и покликать.

## Результаты деплоя на devnet

### Адреса контрактов

#### Адресс sol usd oracle

3EKbZCxvUXzrZaVKYHc7BcyqrVQPpMmN3vysSxPfZ6iE

##### Транзакции

1. https://explorer.solana.com/tx/46MnvZR8BHkqPZEcvdPBwh3wB9DPMFJ188BaszPUKh1Boi7XvA3Cs6GTz2QtajL6aoAedFgJux6c4RGxbsgP9iWt?cluster=devnet
2. https://explorer.solana.com/tx/4DLddmefJKDDFxkymUtmLuQR8aKK7xopva8TgjRtm8JNnxomfpJPZaMJK8A161EsrhU1ybzuPAq2Ld3DSah31L35?cluster=devnet

#### Адрес token minter

CuDJpGcDkPPUyFXQrGV6x3WZGADdiVE3tcK7WA4j5vFa

##### Транзакции

1. https://explorer.solana.com/tx/4FpKMbMt3DmHpdT4gHx55VrhCyjd2z6MTYNShbxyxBFmq7vTLTgZCfcmzqQKjYGYguWXKfEt5p2LA3myUK5UZDvd?cluster=devnet
2. https://explorer.solana.com/tx/ii8K5NVY692V25kZo42enAfJ1MscZ2VJWY7ggzMZEnV4oZBipYDeea7h7DewArQeaWgV28n8LFGfNxdcmCN7KgH?cluster=devnet

### Oracle state pubkey

EwWo2g3crUNLeXPRCA1VSRXLEBmXLe6MZgSzLYRhjPHS

### Адрес минта

1. 6vjErcmq6BhYYhj9mNmWWZsHXADK99bLqKJ9D8w6bdzR
2. BEbtCXwcZQFyAtpS56i2THh2CEbY16S6Mq6Wqt8yqSWF
3. GiTKgwP8kmoQc8Z35EE2qFhAgK1ugjf1kZaDPp3eGDRD

https://explorer.solana.com/address/CC7m6vdDZV7m6WDCqg9YyBoKdjFeMNugQerobQ3DxQYt/tokens?cluster=devnet

### Кошелек с которого был выпущен токен

CC7m6vdDZV7m6WDCqg9YyBoKdjFeMNugQerobQ3DxQYt

https://explorer.solana.com/address/CC7m6vdDZV7m6WDCqg9YyBoKdjFeMNugQerobQ3DxQYt?cluster=devnet

Транзакции:

1. https://explorer.solana.com/tx/3YKGqVwfaWTysmtLJHiVi6y2gqCDMoyjj8WhvHnQKumxDYe6YzcvZ2z1ALwMk9oXcT887zSDUxXNHrdnCxtkyJPM?cluster=devnet
2. https://explorer.solana.com/tx/3Jp84e7qmph8w2KkZSsc9yut8xuQCJqiBEz4snBeitTQC1qFeRen75W5UwLJvcuKio8my9WTVWx3ycDyu6zd9Nxw?cluster=devnet

## TODO

1. Было бы здорово добавить проверку актуальности цены оракула в mint_token, раз мы её имеем в стейте (например, require!(Clock::get()?.slot - oracle_state.last_updated_slot <= MAX_STALENESS)).
2. В launchpad можно стейт и события вынести в отдельный файл — так же, как в оракле. Это немного добавит удобства в работе с большими контрактами.
3. В аннотациях ///CHECK: для Metaplex Token Metadata Program и Metadata PDA‑аккаунтов можно указать, что их проверка проведена в коде метода контракта. Это немного ускорит проверку безопасности контракта.
4. При применении защищенного умножения считается правильным преобразовывать тип цены к u128 и обратно. Это может помочь избежать ошибки на граничных значениях.
