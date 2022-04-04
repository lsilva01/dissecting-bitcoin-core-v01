# How wallets identify relevant transactions

## 1. Receiveing notifications about new transactions or new blocks

When a Bitcoin Core node learns about a new transaction, how does it know if it is related to one or more of its wallets?

The first thing to notice is that the `class CWallet` implements the `interfaces::Chain::Notifications`.

```c++
class CWallet final : public WalletStorage, public interfaces::Chain::Notifications
{
    // ...
}
```

This interface givers the wallet the ability to access to receive a series of notifications, such as `transactionAddedToMempool`, `transactionRemovedFromMempool`, `blockConnected` and so on. The names of these methods are self-explanatory.

To register itself as notification client, the wallet has the `std::unique_ptr<interfaces::Handler> m_chain_notifications_handler` attribute and it is initialized in `CWallet::AttachChain(...)` method.

This method updates the wallet according to the current chain, scanning new blocks, updating the best block locator, and registering for notifications about new blocks and transactions. This is called when the wallet is created or loaded (`CWallet::Create(...)`).

```c++
bool CWallet::AttachChain(const std::shared_ptr<CWallet>& walletInstance, interfaces::Chain& chain, const bool rescan_required, bilingual_str& error, std::vector<bilingual_str>& warnings)
{
    LOCK(walletInstance->cs_wallet);
    // allow setting the chain if it hasn't been set already but prevent changing it
    assert(!walletInstance->m_chain || walletInstance->m_chain == &chain);
    walletInstance->m_chain = &chain;

    walletInstance->m_chain_notifications_handler = walletInstance->chain().handleNotifications(walletInstance);
    // ...
}
```

This briefly explains how the wallet is able to listen to new transactions or blocks. More information about the notification mechanism can be seen in the [Notifications Mechanism (ValidationInterface)](https://github.com/chaincodelabs/bitcoin-core-onboarding/blob/main/1.0_bitcoin_core_architecture.asciidoc#notifications-mechanism-validationinterface) section of [Bitcoin Architecture](https://github.com/chaincodelabs/bitcoin-core-onboarding/blob/main/1.0_bitcoin_core_architecture.asciidoc) article.

## 2. Notification Handlers

The next step is to filter which transactions interest the wallet.

Four of these notification handlers are the ones that are relevant to filter transactions. All of them call `CWallet::SyncTransaction(...)`.

```c++
// src/wallet/wallet.h
void SyncTransaction(const CTransactionRef& tx, const SyncTxState& state, bool update_tx = true, bool rescanning_old_block = false) EXCLUSIVE_LOCKS_REQUIRED(cs_wallet);

// src/wallet/wallet.cpp
void CWallet::SyncTransaction(const CTransactionRef& ptx, const SyncTxState& state, bool update_tx, bool rescanning_old_block)
{
    if (!AddToWalletIfInvolvingMe(ptx, state, update_tx, rescanning_old_block))
        return; // Not one of ours

    // If a transaction changes 'conflicted' state, that changes the balance
    // available of the outputs it spends. So force those to be
    // recomputed, also:
    MarkInputsDirty(ptx);
}

void CWallet::transactionAddedToMempool(const CTransactionRef& tx, uint64_t mempool_sequence) {
    LOCK(cs_wallet);
    SyncTransaction(tx, TxStateInMempool{});
    // ...
}

void CWallet::transactionRemovedFromMempool(const CTransactionRef& tx, MemPoolRemovalReason reason, uint64_t mempool_sequence) {
    // ...
    if (reason == MemPoolRemovalReason::CONFLICT) {
        // ...
        SyncTransaction(tx, TxStateInactive{});
    }
}

void CWallet::blockConnected(const CBlock& block, int height)
{
    // ...
    for (size_t index = 0; index < block.vtx.size(); index++) {
        SyncTransaction(block.vtx[index], TxStateConfirmed{block_hash, height, static_cast<int>(index)});
        transactionRemovedFromMempool(block.vtx[index], MemPoolRemovalReason::BLOCK, 0 /* mempool_sequence */);
    }
}

void CWallet::blockDisconnected(const CBlock& block, int height)
{
    // ...
    for (const CTransactionRef& ptx : block.vtx) {
        SyncTransaction(ptx, TxStateInactive{});
    }
}
```

Note that `CWallet::SyncTransaction(...)` adds the transactions to wallet if it is relevant and then marks each input of the transaction (`const std::vector<CTxIn> CTransaction::vin`) as dirty so they can be recalculated.

## 3. Scanning the block chain

Another method that calls `CWallet::SyncTransaction(...)` is the `CWallet::ScanForWalletTransactions(...)`, which scans the block chain (starting in `start_block` parameter) for transactions relevant to the wallet.

This method is called when manually requesting a rescan (`rescanblockchain` RPC), when adding a new descriptor or when a new key is added to the wallet.

```c++
CWallet::ScanResult CWallet::ScanForWalletTransactions(const uint256& start_block, int start_height, std::optional<int> max_height, const WalletRescanReserver& reserver, bool fUpdate)
{
    // ...
    for (size_t posInBlock = 0; posInBlock < block.vtx.size(); ++posInBlock) {
        SyncTransaction(block.vtx[posInBlock], TxStateConfirmed{block_hash, block_height, static_cast<int>(posInBlock)}, fUpdate, /*rescanning_old_block=*/true);
    }
    // ...
}
```

## 4. `AddToWalletIfInvolvingMe(...)`

`CWallet::AddToWalletIfInvolvingMe` basically perfoms the following steps:

* If the transaction is confirmed, it checks if it conflicts with another. If so, marks the transaction (and its in-wallet descendants) as conflicting with a particular block (`if (auto* conf = std::get_if<TxStateConfirmed>(&state))`).

* It checks if the wallet already contains the transaction. If so, updates if requested in the `fUpdate` parameter or finishes the execution (`if (fExisted && !fUpdate) return false;`).

* It checks if the transaction interests the wallet (`if (fExisted || IsMine(tx) || IsFromMe(tx))`)

* If so, it checks if any keys in the wallet keypool that were supposed to be unused have appeared in a new transaction. If so, removes those keys from the keypool (`for (auto &dest : spk_man->MarkUnusedAddresses(txout.scriptPubKey))`).

* Finally, it adds the transaction to the wallet (`AddToWallet(...)`). This function inserts the new transaction in `CWallet::mapWallet`, updates it with relevant information such as `CWalletTx::nTimeReceived` (time it was received by the node), `CWalletTx::nOrderPos` (position in ordered transaction list) and so on. This function also writes the transaction to database (`batch.WriteTx(wtx)`) and mark the transaction as dirty to recalculate balance.

```c++
// src/wallet/wallet.cpp
bool CWallet::AddToWalletIfInvolvingMe(const CTransactionRef& ptx, const SyncTxState& state, bool fUpdate, bool rescanning_old_block)
{
    const CTransaction& tx = *ptx;
    {
        AssertLockHeld(cs_wallet);

        if (auto* conf = std::get_if<TxStateConfirmed>(&state)) {
            // ...
        }

        bool fExisted = mapWallet.count(tx.GetHash()) != 0;
        if (fExisted && !fUpdate) return false;
        if (fExisted || IsMine(tx) || IsFromMe(tx))
        {
            for (const CTxOut& txout: tx.vout) {
                for (const auto& spk_man : GetScriptPubKeyMans(txout.scriptPubKey)) {
                    for (auto &dest : spk_man->MarkUnusedAddresses(txout.scriptPubKey)) {
                        // ...
                    }
                }
            }

            TxState tx_state = std::visit([](auto&& s) -> TxState { return s; }, state);
            return AddToWallet(MakeTransactionRef(tx), tx_state, /*update_wtx=*/nullptr, /*fFlushOnClose=*/false, rescanning_old_block);
        }
    }
    return false;
}

CWalletTx* CWallet::AddToWallet(CTransactionRef tx, const TxState& state, const UpdateWalletTxFn& update_wtx, bool fFlushOnClose, bool rescanning_old_block)
{
    LOCK(cs_wallet);

    WalletBatch batch(GetDatabase(), fFlushOnClose);

    uint256 hash = tx->GetHash();

    // ...

    auto ret = mapWallet.emplace(std::piecewise_construct, std::forward_as_tuple(hash), std::forward_as_tuple(tx, state));
    CWalletTx& wtx = (*ret.first).second;
    // ...
    if (fInsertedNew) {
        wtx.nTimeReceived = GetTime();
        wtx.nOrderPos = IncOrderPosNext(&batch);
        // ...
    }

    // ...

    // Write to disk
    if (fInsertedNew || fUpdated)
        if (!batch.WriteTx(wtx))
            return nullptr;

    // Break debit/credit balance caches:
    wtx.MarkDirty();

    // ...

    return &wtx;
}
```

## 5. `CWallet::IsMine(...)`

As the name implies, the method that actually identifies which transactions belong to the wallet is `IsMine()`.

```c++
isminetype CWallet::IsMine(const CScript& script) const
{
    AssertLockHeld(cs_wallet);
    isminetype result = ISMINE_NO;
    for (const auto& spk_man_pair : m_spk_managers) {
        result = std::max(result, spk_man_pair.second->IsMine(script));
    }
    return result;
}
```

Note the `CWallet::IsMine(const CScript& script)` is just a proxy to the `ScriptPubKeyMan::IsMine(const CScript &script)`. This is an important distiction, because in Bitcoin Core the class `CWallet` does not manage the keys. This work is done by `ScriptPubKeyMan` subclasses: `DescriptorScriptPubKeyMan` and `LegacyScriptPubKeyMan`. All `ScriptPubKeyMan` instances belonging to the wallet are stored in `CWallet::m_spk_managers`.

Another important aspect of that method is the return type, the `enum isminetype`. This type is defined in `src/wallet/ismine.h`.

```c++
enum isminetype : unsigned int {
    ISMINE_NO         = 0,
    ISMINE_WATCH_ONLY = 1 << 0,
    ISMINE_SPENDABLE  = 1 << 1,
    ISMINE_USED       = 1 << 2,
    ISMINE_ALL        = ISMINE_WATCH_ONLY | ISMINE_SPENDABLE,
    ISMINE_ALL_USED   = ISMINE_ALL | ISMINE_USED,
    ISMINE_ENUM_ELEMENTS,
};
```

The code depends on `ScriptPubKeyMan` implementation. Not every `ScriptPubKeyMan` covers all types.

For `LegacyScriptPubKeyMan`:
* `ISMINE_NO`: the scriptPubKey is not in the wallet;
* `ISMINE_WATCH_ONLY`: the scriptPubKey has been imported into the wallet;
* `ISMINE_SPENDABLE`: the scriptPubKey corresponds to an address owned by the wallet user (who can spend with the private key);
* `ISMINE_USED`: the scriptPubKey corresponds to a used address owned by the wallet user;
* `ISMINE_ALL`: all ISMINE flags except for USED;
* `ISMINE_ALL_USED`: all ISMINE flags including USED;
* `ISMINE_ENUM_ELEMENTS`: the number of isminetype enum elements.

For `DescriptorScriptPubKeyMan` and future `ScriptPubKeyMan`:
* `ISMINE_NO`: the scriptPubKey is not in the wallet;
* `ISMINE_SPENDABLE`: the scriptPubKey matches a scriptPubKey in the wallet.
* `ISMINE_USED`: the scriptPubKey corresponds to a used address owned by the wallet user.

## 6. `DescriptorScriptPubKeyMan::IsMine(...)`

`DescriptorScriptPubKeyMan::IsMine(...)` basically checks if `DescriptorScriptPubKeyMan::m_map_script_pub_keys` contains the `CScript scriptPubKey` passed in parameter.

```c++
isminetype DescriptorScriptPubKeyMan::IsMine(const CScript& script) const
{
    LOCK(cs_desc_man);
    if (m_map_script_pub_keys.count(script) > 0) {
        return ISMINE_SPENDABLE;
    }
    return ISMINE_NO;
}
```

`DescriptorScriptPubKeyMan::m_map_script_pub_keys` is a `std::map<CScript, int32_t>` type (a map of scripts to the descriptor range index).

## 7. `LegacyScriptPubKeyMan::IsMine(...)`

`LegacyScriptPubKeyMan::IsMine(...)` is only a proxy for `IsMineResult IsMineInner(...)`.

```c++
isminetype LegacyScriptPubKeyMan::IsMine(const CScript& script) const
{
    switch (IsMineInner(*this, script, IsMineSigVersion::TOP)) {
    case IsMineResult::INVALID:
    case IsMineResult::NO:
        return ISMINE_NO;
    case IsMineResult::WATCH_ONLY:
        return ISMINE_WATCH_ONLY;
    case IsMineResult::SPENDABLE:
        return ISMINE_SPENDABLE;
    }
    assert(false);
}
```

`IsMineResult IsMineInner(...)` is only used by `LegacyScriptPubKeyMan` (which should be deprecated at some point) and is considerably more complex than its equivalent in the more modern `DescriptorScriptPubKeyMan`.

The first step is to call `Solver(scriptPubKey, vSolutions)` method, which parses a scriptPubKey and identifies the script type for standard scripts. If successful, returns the script type and parsed pubkeys or hashes, depending on the type. For example, for a P2SH script, `vSolutionsRet` will contain the script hash, for P2PKH it will contain the key hash, an so on.

```c++
IsMineResult IsMineInner(const LegacyScriptPubKeyMan& keystore, const CScript& scriptPubKey, IsMineSigVersion sigversion, bool recurse_scripthash=true)
{
    IsMineResult ret = IsMineResult::NO;

    std::vector<valtype> vSolutions;
    TxoutType whichType = Solver(scriptPubKey, vSolutions);
    // ...
}
```

The next step is to handle each script type separately. Note that if it is a Taproot transaction, it will not be considered spendable by legacy wallets. They purposely do not support Taproot as they are marked for deprecation.

```c++
IsMineResult IsMineInner(...)
{
    // ...
    TxoutType whichType = Solver(scriptPubKey, vSolutions);

    CKeyID keyID;
    switch (whichType) {
    case TxoutType::NONSTANDARD:
    case TxoutType::NULL_DATA:
    case TxoutType::WITNESS_UNKNOWN:
    case TxoutType::WITNESS_V1_TAPROOT:
        break;
    case TxoutType::PUBKEY:
        // ...
    case TxoutType::WITNESS_V0_KEYHASH:
        // ...
    case TxoutType::PUBKEYHASH:
        // ...
    case TxoutType::SCRIPTHASH:
        // ...
    case TxoutType::WITNESS_V0_SCRIPTHASH:
        // ...
    case TxoutType::MULTISIG:
        // ...
    }
    } // no default case, so the compiler can warn about missing cases

    if (ret == IsMineResult::NO && keystore.HaveWatchOnly(scriptPubKey)) {
        ret = std::max(ret, IsMineResult::WATCH_ONLY);
    }
    return ret;
}
```

If no script type conditions are met for a `scriptPubKey`, the function checks at the end if it is a watch-only script in the wallet.

```c++
IsMineResult IsMineInner(...)
{
    // ...
    switch (whichType) {
        // ...
        case TxoutType::PUBKEY:
        keyID = CPubKey(vSolutions[0]).GetID();
        if (!PermitsUncompressed(sigversion) && vSolutions[0].size() != 33) {
            return IsMineResult::INVALID;
        }
        if (keystore.HaveKey(keyID)) {
            ret = std::max(ret, IsMineResult::SPENDABLE);
        }
        break;
        // ...
    }
    // ...
}
```

When the script type is a public key, the function first checks if it is a `P2PK` (uncompressed public key), otherwise it must be 33 bytes (compressed format).

It then checks if the wallet keystore has the key. In this case, it means the script can be spent by the wallet.

---
**NOTE**

In the early days of Bitcoin, the transactions were of type `P2PK`, which were specified in uncompressed format.
However using this format turned out to be both wasteful for storing unspent transaction outputs (UTXOs) and a compressed format was adopted for `P2PKH` and `P2WPKH`.

Uncompressed format has:

* `04` - Marker
* x coordinate - 32 bytes, big endian
* y coordinate - 32 bytes, big endian

And the compressed has:

* `02` if y is even, `03` if odd - Marker
* x coordinate - 32 bytes, big endian

Note that the compressed format has a total of 33 bytes (x coordinate + marker).

More recently, taproot address `P2TR` was introduced and it uses a format called `x-only`, with only x coordinate - 32 bytes, big endian.

---

The next step is the segwit format (`P2WPKH`). First the function invalidates the script if this has a `P2WPKH` nested inside `P2WSH`. It then checks that the script is in the expected format with the `OP_0` before the witness output.

If these two validations pass, the script will be recreated as Public Key Hash and the function will be called recursively. Note that in this second call, the script will be handled as `TxoutType::PUBKEYHASH`.

```c++
IsMineResult IsMineInner(...)
{
    // ...
    case TxoutType::WITNESS_V0_KEYHASH:
    {
        if (sigversion == IsMineSigVersion::WITNESS_V0) {
            // P2WPKH inside P2WSH is invalid.
            return IsMineResult::INVALID;
        }
        if (sigversion == IsMineSigVersion::TOP && !keystore.HaveCScript(CScriptID(CScript() << OP_0 << vSolutions[0]))) {
            // We do not support bare witness outputs unless the P2SH version of it would be
            // acceptable as well. This protects against matching before segwit activates.
            // This also applies to the P2WSH case.
            break;
        }
        ret = std::max(ret, IsMineInner(keystore, GetScriptForDestination(PKHash(uint160(vSolutions[0]))), IsMineSigVersion::WITNESS_V0));
        break;
    }
    // ...
}
```

The `TxoutType::PUBKEYHASH` logic is very similar to the `TxoutType::PUBKEY`: it checks if the wallet keystore has the key, which means the script can be spent by the wallet.

Before that, however, the function validates whether the key must be compressed.

```c++
IsMineResult IsMineInner(...)
{
    // ...
    case TxoutType::PUBKEYHASH:
        keyID = CKeyID(uint160(vSolutions[0]));
        if (!PermitsUncompressed(sigversion)) {
            CPubKey pubkey;
            if (keystore.GetPubKey(keyID, pubkey) && !pubkey.IsCompressed()) {
                return IsMineResult::INVALID;
            }
        }
        if (keystore.HaveKey(keyID)) {
            ret = std::max(ret, IsMineResult::SPENDABLE);
        }
        break;
    // ...
}
```

The next item to be dealt with is `TxoutType::SCRIPTHASH`. The logic is very similiar to the one seen before. First the script is validated (`P2SH` inside `P2WSH` or `P2SH` is invalid) and the function checks if the script exists in THE wallet keystore. As with `TxoutType::WITNESS_V0_KEYHASH`, the function will recurse into nested p2sh and p2wsh scripts or will simply treat any script that has been stored in the keystore as spendable.


```c++
IsMineResult IsMineInner(...)
{
    // ...
    case TxoutType::SCRIPTHASH:
    {
        if (sigversion != IsMineSigVersion::TOP) {
            // P2SH inside P2WSH or P2SH is invalid.
            return IsMineResult::INVALID;
        }
        CScriptID scriptID = CScriptID(uint160(vSolutions[0]));
        CScript subscript;
        if (keystore.GetCScript(scriptID, subscript)) {
            ret = std::max(ret, recurse_scripthash ? IsMineInner(keystore, subscript, IsMineSigVersion::P2SH) : IsMineResult::SPENDABLE);
        }
        break;
    }
    // ...
}
```

`TxoutType::WITNESS_V0_SCRIPTHASH` has the same logic seen in the previous item. The only difference is that the has `Hash160` is recreated with the solved script hash, since `P2SH-P2WSH` is allowed.

```c++
IsMineResult IsMineInner(...)
{
    // ...
    case TxoutType::WITNESS_V0_SCRIPTHASH:
    {
        if (sigversion == IsMineSigVersion::WITNESS_V0) {
            // P2WSH inside P2WSH is invalid.
            return IsMineResult::INVALID;
        }
        if (sigversion == IsMineSigVersion::TOP && !keystore.HaveCScript(CScriptID(CScript() << OP_0 << vSolutions[0]))) {
            break;
        }
        uint160 hash;
        CRIPEMD160().Write(vSolutions[0].data(), vSolutions[0].size()).Finalize(hash.begin());
        CScriptID scriptID = CScriptID(hash);
        CScript subscript;
        if (keystore.GetCScript(scriptID, subscript)) {
            ret = std::max(ret, recurse_scripthash ? IsMineInner(keystore, subscript, IsMineSigVersion::WITNESS_V0) : IsMineResult::SPENDABLE);
        }
        break;
    }
    // ...
}
```

The last type of script is `TxoutType ::MULTISIG`, whose logic is straightforward. `Solver (...)` returns all the keys of the script and then they are validated in the same way as the previous scripts. Transactions are only considered `ISMINE_SPENDABLE` if the node has all keys.

```c++
IsMineResult IsMineInner(...)
{
    // ...
    case TxoutType::MULTISIG:
    {
        if (sigversion == IsMineSigVersion::TOP) {
            break;
        }

        std::vector<valtype> keys(vSolutions.begin()+1, vSolutions.begin()+vSolutions.size()-1);
        if (!PermitsUncompressed(sigversion)) {
            for (size_t i = 0; i < keys.size(); i++) {
                if (keys[i].size() != 33) {
                    return IsMineResult::INVALID;
                }
            }
        }
        if (HaveKeys(keys, keystore)) {
            ret = std::max(ret, IsMineResult::SPENDABLE);
        }
        break;
    }
    // ...
}
```

Thus, we cover most of the code responsible for identifying which transactions belong to the wallet. The code related to `IsMine(...)` or `IsMineInner(...)` is used either when the transactions arrive through the mempool, or by blocks or through a scan in the UTXO Set.
