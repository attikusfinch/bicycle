# TON payment processor
[![Based on TON][ton-svg]][ton]
[![Go](https://github.com/gobicycle/bicycle/actions/workflows/go.yml/badge.svg)](https://github.com/gobicycle/bicycle/actions/workflows/go.yml)
[![Telegram][telegram-svg]][telegram-url]

Microservice for accepting payments and making withdrawals to wallets in TON blockchain.  
Supports TON coins and Jettons (conforming certain criteria)  
Provides REST API for integration.
Service is ADNL based and interacts directly with node and do not use any third party API.

**Warning** The project is in the testing and proof-of-concepts phase. Use with caution. Suggestions are welcome.

- [How it works](#How-it-works)
  - [Features](#Features)
- [Glossary](#Glossary)
- [Prerequisites](#Prerequisites)
  - [Criteria for valid Jettons](#Criteria-for-valid-Jettons)
- [Deployment](#Deployment)
  - [Configurable parameters](#Configurable-parameters)
  - [Service deploy](#Service-deploy)
- [REST API](https://gobicycle.github.io/bicycle/)
- [Technical notes](/technical_notes.md)
- [Threat model](/threat_model.md)
- [TODO list](/todo_list.md)

![test_dashboard](https://user-images.githubusercontent.com/120649456/211955983-698b12b8-eccf-45c5-85bb-f8f6364c154e.png)
![db_dashboard](https://user-images.githubusercontent.com/120649456/211955998-749772a6-10d2-4594-96f1-be6ab6051be5.png)

## How it works

The service provides the following functionality:

- `generate new deposit address` of wallet (TON or Jetton) for TON blockchain. This address you provide to customer for payment. Payments are accumulated in the hot-wallet.
- `make withdrawal` of TONs or Jettons from hot-wallet to customer wallet at TON blockchain.

### Features
* Deposit addresses can be reused for multiple payments
* Sends withdrawals with comment
* Sends withdrawals in batches using highload wallet
* Aggregates part of TONs or Jettons at cold-wallet
* Supports authorization by Bearer token
* Generates and monitor deposits with 8-bit address prefix same as hot-wallet (in the same shard)
* Service withdrawals (cancellation of incorrect payments)

## Glossary

- `deposit-address` - address generated by the service to which users send payments.
- `deposit` - blockchain account with `deposit-address`
- `hot-wallet` - wallet for aggregation all incoming TONs and Jettons from deposit-addresses.
- `cold-wallet` - wallet to which part of the funds from the hot wallet is sent for security. Cold-wallet seed phrase is not used by the service.
- `user_id` - unique text value to identify deposit-addresses or withdrawal request for a specific user.
- `query_id` - unique text value to identify withdrawal request for a specific user to prevent double spending.
- `basic unit` - minimum indivisible unit for TON (e.g. for TON `basic unit` = nanoTONs) or Jetton.
- `hot_wallet_minimum_balance` - minimum TON balance in nanoTONs at hot wallet to start service.
- `hot_wallet_maximum_balance` - minimum balance (of TONs or Jettons) in basic units at hot wallet. Anything more than this amount will be withdrawn to a cold wallet.
- `minimum_withdrawal_amount` - minimum balance (of TONs or Jettons) in basic units at deposit account to make withdrawal to hot wallet. It is necessary to prevent the case when the withdrawal fee will be close to the balance on the deposit.

## Prerequisites
- Need minimum (configured) amount of TONs at HighloadV2 wallet address correlated with seed phrase or already deployed HighloadV2 wallet. 
- To ensure the reliability and security of the service, you need to provide your own TON node (with lite server) on the same machine as the service.
- Jettons used must meet certain criteria

### Criteria for valid Jettons
- conforming to the standard [TEP-74](https://github.com/ton-blockchain/TEPs/blob/master/text/0074-jettons-standard.md)
- the Jetton wallet should not spontaneously change its balance, only with transfer.
- fee for the withdrawal of Jettons from the wallet should not be too high and meet the internal setting of the service

## Deployment

### Configurable parameters
| ENV variable           | Description                                                                                                                                                                                                                                                                                                           |
|------------------------|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `LITESERVER`           | IP and port of lite server, example: `185.86.76.183:5815`                                                                                                                                                                                                                                                             |
| `LITESERVER_KEY`       | public key of lite server `5v2dHtSclsGsZVbNVwTj4hQDso5xvQjzL/yPEHJevHk=`. <br/>Be careful with base64 encoding and ENV var. Use ''                                                                                                                                                                                    |
| `SEED`                 | seed phrase for main hot wallet. 24 words compatible with standard TON wallets                                                                                                                                                                                                                                        |
| `DB_URI`               | URI for DB connection, example: <br/>`postgresql://db_user:db_password@localhost:5432/payment_processor`                                                                                                                                                                                                              |
| `API_HOST`             | host for REST API, example `localhost:8081`, default `0.0.0.0:8081`                                                                                                                                                                                                                                                   |
| `API_TOKEN`            | Bearer token for REST API, example `123`                                                                                                                                                                                                                                                                              |
| `IS_TESTNET`           | `true` if service works in TESTNET, `false` - for MAINNET. Default: `true`.                                                                                                                                                                                                                                           |
| `JETTONS`              | list of Jettons, processed by service in format: <br/>`JETTON_SYMBOL_1:MASTER_CONTRACT_ADDR_1:hot_wallet_max_balance:min_withdrawal_amount, JETTON_SYMBOL_2:MASTER_CONTRACT_ADDR_2:hot_wallet_max_balance:min_withdrawal_amount`, <br/>example: `TGR:kQCKt2WPGX-fh0cIAz38Ljd_OKQjoZE_cqk7QrYGsNP6wfP0:1000000:100000` |
| `TON_CUTOFFS`          | cutoffs in nanoTONs in format: <br/>`hot_wallet_min_balance:hot_wallet_max_balance:min_withdrawal_amount`, <br/> example `1000000000:100000000000:1000000000`                                                                                                                                                         |
| `COLD_WALLET`          | cold-wallet address, example `kQCdyiS-fIV9UVfI9Phswo4l2MA-hm8YseH3XZ_YiH9Y1ufw`                                                                                                                                                                                                                                       |
| `DEPOSIT_SIDE_BALANCE` | `true` - service calculates user balance by deposit incoming, `false` - by hot wallet incoming. Default: `false`.                                                                                                                                                                                                     |
| `QUEUE_ENABLED`        | `true` - service sends incoming notifications to queue, `false` - sending disabled. Default: `false`.                                                                                                                                                                                                                 |
| `QUEUE_URI`            | URI for queue client connection, example `amqp://guest:guest@payment_rabbitmq:5672/`                                                                                                                                                                                                                                  |
| `QUEUE_NAME`           | name of exchange                                                                                                                                                                                                                                                                                                      |

**! Be careful with `IS_TESTNET` variable.** This does not guarantee that a testnet node is being used. It is only for address checking purposes.

There are also internal service settings (fees and timeouts) that are specified in the source code in the [Config](/config/config.go) package.
Calibration parameters recommendations in [Technical notes](/technical_notes.md).

### Service deploy

**Do not use same `.env` file for `payment-processor` and other services!**

1. Build docker images from makefile 
```console
make -f Makefile
```
2. Prepare `.env` file for `payment-postgres` service or fill environment variables in `docker-compose-main.yml` file.
Database scheme automatically init.
```console
docker-compose -f docker-compose-main.yml up -d payment-postgres
```
3. Prepare `.env` file for `payment-processor` service or fill environment variables in `docker-compose-main.yml` file.
```console
docker-compose -f docker-compose-main.yml up -d payment-processor
```
4. Optionally you can start Grafana for service monitoring. Prepare `.env` file for `payment-grafana` service or 
fill environment variables in `docker-compose-main.yml` file.
```console
docker-compose -f docker-compose-main.yml up -d payment-grafana
```
5. Optionally you can start RabbitMQ to collect payment notifications (if `QUEUE_ENABLED` env var is `true` for payment-processor). 
Prepare `.env` file for `payment-rabbitmq` service or fill environment variables in `docker-compose-main.yml` file.
```console
docker-compose -f docker-compose-main.yml up -d payment-rabbitmq
```

<!-- Badges -->
[ton-svg]: https://img.shields.io/badge/Based%20on-TON-blue
[ton]: https://ton.org
[telegram-url]: https://t.me/tonbicycle
[telegram-svg]: https://img.shields.io/badge/telegram-chat-blue?color=blue&logo=telegram&logoColor=white
