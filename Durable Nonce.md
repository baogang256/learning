# 使用 Solana 的 Durable Nonce 向多个 Landing Service 发送同一笔交易

在 Solana 区块链中，Durable Nonce 是一种强大的工具，可以用于创建和签名可以在未来任意时间提交的交易。与普通的基于区块哈希的交易不同，Durable Nonce 交易不会因为区块哈希过期而失效，这使得它们特别适合需要离线签名或延迟提交的场景。本文将详细介绍如何使用 Durable Nonce 向多个 Landing Service 发送同一笔交易，并解释这样做的好处。

## 1\. Durable Nonce 的工作原理

在 Solana 中，普通的交易使用 Recent Blockhash 来防止双重支付（Double-Spend）问题。然而，Recent Blockhash 只在最近 150 个区块内有效（大约 80-90 秒），这限制了交易的提交时间。Durable Nonce 通过引入一个特殊的账户（Nonce Account）来解决这个问题。Nonce Account 存储一个唯一的 Nonce 值，该值可以替代 Recent Blockhash，从而确保每笔交易的唯一性。

Durable Nonce 交易的另一个关键特性是，每笔交易必须以 `nonceAdvance` 指令作为第一条指令。这个指令会更新 Nonce Account 中的 Nonce 值，确保每次交易的 Nonce 都是唯一的。即使交易失败，Nonce 也会被推进，从而防止重复使用同一个 Nonce。

## 2\. 使用 Durable Nonce 向多个 Landing Service 发送交易的好处

向多个 Landing Service 发送同一笔交易的主要目的是提高交易的总成功率。在实际应用中，可能会遇到以下情况：

- **网络延迟或不稳定**：某些 Landing Service 可能因为网络问题而无法及时提交交易。
- **费用问题**：某些 Landing Service 可能因为费用不足而拒绝交易。
- **交易拥堵**：某些 Landing Service 可能因为交易拥堵而无法及时处理交易。

通过向多个 Landing Service 发送同一笔交易，可以显著提高交易成功提交的可能性。Durable Nonce 的持久性和唯一性确保了即使某些 Landing Service 失败，其他 Landing Service 仍然可以成功提交交易。

## 3\. 实现步骤

以下是使用 Durable Nonce 向多个 Landing Service 发送同一笔交易的实现步骤：

### 3.1 创建 Nonce 账户

首先，需要创建一个 Nonce 账户，并为其提供足够的资金以满足租金豁免的要求。
#### 创建 Solana Nonce 账户的命令行步骤
##### 生成 Nonce 账户密钥对
```bash
solana-keygen new -o nonce-account.json
```
**说明**：  
  - 此命令会生成一个新的密钥对，并保存到 `nonce-account.json` 文件中。
  - 密钥对包含公钥和私钥，用于标识 Nonce 账户。

##### 使用发送者账户创建 Nonce 账户
```bash
solana -k sender.json create-nonce-account nonce-account.json 0.0015
```
**参数解析**：  
  - `-k sender.json`：指定支付交易费用的发送者账户密钥文件。
  - `nonce-account.json`：上一步生成的 Nonce 账户密钥文件。
  - `0.0015`：分配给 Nonce 账户的初始 SOL 余额（单位：SOL）。

**功能**：  
  - 该命令会从发送者账户扣除指定 SOL，创建并初始化 Nonce 账户。
  - Nonce 账户用于存储链上动态随机数（Durable Nonce），支持离线交易。

### 3.2 使用Durable Nonce的完整示例（Python）

接下来，我们将通过Python代码，演示如何利用同一个Nonce Instruction，将同一笔交易，面向不同的Landing Service分别构建不同的Transaction并提交。我们使用了两个Landing Service: de.0slot.trade (Germany endpoint)和ny.0slot.trade ( New York endpoint), 以及上面已经创建好的Nonce账号。

我们首先展示完整的代码，你可以复制并保存为py文件运行。接下去我们会对关键代码进行说明。

#### 3.2.1 完整代码

```python
import base58
import asyncio
import argparse
from solders.hash import Hash
from solders.pubkey import Pubkey
from solders.keypair import Keypair
from solders.message import Message
from solders.transaction import Transaction
from solders.system_program import TransferParams, transfer
from solana.rpc.async_api import AsyncClient

async def send_solana_transaction(api_key, private_key, nonce_public_key, to_public_key):
    # Initialize clients for different regions
    client_for_blockhash = AsyncClient(f"https://api.mainnet-beta.solana.com")
    de_sender = AsyncClient(f"https://de.0slot.trade?api-key=" + api_key)  # Germany endpoint
    ny_sender = AsyncClient(f"https://ny.0slot.trade?api-key=" + api_key)  # New York endpoint

    # Create keypair from private key
    sender = Keypair.from_bytes(base58.b58decode(private_key))

    # Main recipient address
    receiver = Pubkey.from_string(to_public_key)
    # Our TIP receiving addresses - using different addresses for DE and NY to simulate multiple Landing Services
    de_tip_receiver = Pubkey.from_string("6fQaVhYZA4w3MBSXjJ81Vf6W1EDYeUPXpgVQ6UQyU1Av")
    ny_tip_receiver = Pubkey.from_string("4HiwLEP2Bzqj3hM2ENxJuzhcPCdsafwiet3oGkMkuQY4")
    # Nonce account public key created by sender - must be associated
    # solana-keygen new -o nonce-account.json
    # solana -k sender.json create-nonce-account nonce-account.json 0.0015
    nonce_account_pubkey = Pubkey.from_string(nonce_public_key)

    # Get nonce account info to extract the current nonce value
    get_account_resp = await client_for_blockhash.get_account_info(nonce_account_pubkey)
    await client_for_blockhash.close()

    # The nonce is stored in the account data at bytes 40-72
    # This matches the Rust NonceState structure layout:
    # struct NonceState {
    #     version: u32,          // 4 bytes (offset 0-3)
    #     state: u32,            // 4 bytes (offset 4-7)
    #     authorized_pubkey: Pubkey,  // 32 bytes (offset 8-39)
    #     nonce: Pubkey,         // 32 bytes (offset 40-71) ← we need this
    #     fee_calculator: FeeCalculator,  // 8 bytes (offset 72-79)
    # }
    nonce = get_account_resp.value.data[40:72]
    nonce_hash = Hash.from_bytes(nonce)

    # Create transaction instructions for Germany endpoint
    # Includes:
    # 1. Main transfer (1 lamport)
    # 2. Tip transfer (100,000 lamports = 0.0001 SOL)
    de_instructions = [
        transfer(
            TransferParams(
                from_pubkey=sender.pubkey(),
                to_pubkey=receiver,
                lamports=1
            )
        ),
        transfer(
            TransferParams(
                from_pubkey=sender.pubkey(),
                to_pubkey=de_tip_receiver,
                lamports=100000
            )
        )
    ]
    
    # Create transaction instructions for New York endpoint (same structure)
    ny_instructions = [
        transfer(
            TransferParams(
                from_pubkey=sender.pubkey(),
                to_pubkey=receiver,
                lamports=1
            )
        ),
        transfer(
            TransferParams(
                from_pubkey=sender.pubkey(),
                to_pubkey=ny_tip_receiver,
                lamports=100000
            )
        )
    ]
    
    # Create messages with nonce for each transaction
    de_message = Message.new_with_nonce(
        de_instructions,
        payer=sender.pubkey(),
        nonce_account_pubkey=nonce_account_pubkey,
        nonce_authority_pubkey=sender.pubkey(),
    )
    ny_message = Message.new_with_nonce(
        ny_instructions,
        payer=sender.pubkey(),
        nonce_account_pubkey=nonce_account_pubkey,
        nonce_authority_pubkey=sender.pubkey(),
    )

    # Create unsigned transactions
    de_transaction = Transaction.new_unsigned(de_message)
    ny_transaction = Transaction.new_unsigned(ny_message)

    # Sign transactions with the same nonce
    de_transaction.sign([sender], nonce_hash)
    ny_transaction.sign([sender], nonce_hash)

    try:
        # Send both transactions concurrently using asyncio
        de_task = de_sender.send_transaction(de_transaction)
        ny_task = ny_sender.send_transaction(ny_transaction)
        results = await asyncio.gather(
            de_task,
            ny_task,
            return_exceptions=True  # Don't fail if one transaction fails
        )
        
        # Process results
        for i, result in enumerate(results):
            if isinstance(result, Exception):
                print(f"Error sending to {'DE' if i == 0 else 'NY'}: {str(result)}")
            else:
                print(f"Transaction signature ({'DE' if i == 0 else 'NY'}):", result.value)
    except Exception as e:
        print("Error:", str(e))
    finally:
        # Clean up connections
        await de_sender.close()
        await ny_sender.close()

async def main():
    # Set up command line argument parser
    parser = argparse.ArgumentParser(description="Send a Solana transaction")
    parser.add_argument("--api_key", required=True, help="Solana API key for accessing the network.")
    parser.add_argument("--private_key", required=True, help="Sender's private key for signing the transaction.")
    parser.add_argument("--nonce_public_key", required=True, help="Sender's nonce account public key")
    parser.add_argument("--to_public_key", required=True, help="Public key of the main receiver.")

    args = parser.parse_args()

    await send_solana_transaction(
        args.api_key,
        args.private_key,
        args.nonce_public_key,
        args.to_public_key,
    )

if __name__ == "__main__":
    asyncio.run(main())

```

#### 3.2.2 针对不同Landing Serivce创建transaction message
  
上述完整的代码中，使用Nonce Instruction针对Landing Service: de.0slot.trade (Germany endpoint)创建transaction messages的关键代码如下：

```python
# Create transaction instructions for Germany endpoint
    # Includes:
    # 1. Main transfer (1 lamport)
    # 2. Tip transfer (100,000 lamports = 0.0001 SOL)
    de_instructions = [
        transfer(
            TransferParams(
                from_pubkey=sender.pubkey(),
                to_pubkey=receiver,
                lamports=1
            )
        ),
        transfer(
            TransferParams(
                from_pubkey=sender.pubkey(),
                to_pubkey=de_tip_receiver,
                lamports=100000
            )
        )
    ]
    # Create messages with nonce for each transaction
    de_message = Message.new_with_nonce(
        de_instructions,
        payer=sender.pubkey(),
        nonce_account_pubkey=nonce_account_pubkey,
        nonce_authority_pubkey=sender.pubkey(),
    )
```
针对Landing Service: ny.0slot.trade ( New York endpoint)的代码和上述基本一致。

#### 3.2.31 同时发送所有的transactions

```python
# Send both transactions concurrently using asyncio
        de_task = de_sender.send_transaction(de_transaction)
        ny_task = ny_sender.send_transaction(ny_transaction)
        results = await asyncio.gather(
            de_task,
            ny_task,
            return_exceptions=True  # Don't fail if one transaction fails
        )
```

## 4\. 总结

通过使用 Solana 的 Durable Nonce，我们可以创建和签名可以在未来任意时间提交的交易。向多个 Landing Service 发送同一笔交易可以显著提高交易的总成功率，尤其是在网络不稳定或交易拥堵的情况下。Durable Nonce 的持久性和唯一性确保了即使某些 Landing Service 失败，其他 Landing Service 仍然可以成功提交交易。

## 参考信息

- [Solana Durable Nonces - 官方文档](https://solana.com/developers/guides/advanced/introduction-to-durable-nonces)
- [Durable & Offline Transaction Signing using Nonces - GitHub](https://github.com/0xproflupin/solana-durable-nonces)

---

希望这篇文章对你有所帮助！如果有任何问题或需要进一步的解释，请随时联系我们。
Our support team is available through these channels:  

- **Discord Support**: [Join our community](https://discord.com/invite/Qd6txfyS)  
- **Twitter/X**: [Follow @0slot_trade](https://x.com/0slot_trade)  
- **Telegram**: [Message @kurt0slot](https://t.me/kurt0slot)  
