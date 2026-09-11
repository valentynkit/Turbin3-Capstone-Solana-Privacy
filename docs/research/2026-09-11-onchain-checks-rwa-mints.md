# On-chain verification (mainnet-beta RPC, 2026-09-11)

Question: do live tokenized-equity issuers on Solana actually use Token-2022 compliance extensions (hooks / DefaultAccountState=Frozen / ACL)?

| Mint | Issuer | Extensions | transferHook program | defaultAccountState | permanentDelegate |
|---|---|---|---|---|---|
| XsDoVfqeBukxuZHWhdvWHBhgEHjGNst4MLodqsJHzoB (TSLAx) | xStocks/Backed | metadataPointer, permanentDelegate, defaultAccountState, scaledUiAmountConfig, pausableConfig, confidentialTransferMint, transferHook, tokenMetadata | **null** (armed, disabled) | **initialized** (not frozen) | 5aMNNLQJwAEeoemTEMkv5NVjqKwvvefRYCQ5Z67HFvEq |
| XsbEhLAtcf6HdfpFZ5xEMdqW8nfAvcsP5bdudRLJzJp (AAPLx) | xStocks/Backed | same | null | initialized | same |
| KeGv7bsfR4MheC1CkmnAVceoApjrkvBhHYjWb67ondo (TSLAon) | Ondo Global Markets | scaledUiAmountConfig, metadataPointer, pausableConfig, defaultAccountState, confidentialTransferMint, transferHook, tokenMetadata | null | initialized | none |
| gEGtLTPNQ7jcg25zTetkbmF7teoDLcrfTnQfmn2ondo (NVDAon) | Ondo Global Markets | same | null | initialized | none |

Both issuers have freeze authorities set (JDq14…/51QVC…).

## Reading
1. Compliance is **armed but switched off**: the TransferHook extension is initialized with no program, DefaultAccountState is `initialized`. Flipping either on today would get the mint rejected by Raydium's `is_supported_mint()` and de-badged on Orca/Meteora → liquidity dies. This is the concrete, verifiable version of the T1 problem: issuers want hooks/ACL, the DEX layer forbids them, so they ship without.
2. Both already use **ScaledUiAmount** (splits) and **Pausable** (halts) and **confidentialTransferMint** — so T2's "splits via ScaledUiAmount" is not novel; issuers do it. T2's remaining novelty is only distributions + record-date snapshots + event log.
3. Neither uses Token ACL (would show DAS=frozen). ACL adoption is zero among the two biggest RWA issuers as of today.

Command used:
```
curl -s https://api.mainnet-beta.solana.com -X POST -H 'content-type: application/json' \
  -d '{"jsonrpc":"2.0","id":1,"method":"getAccountInfo","params":["<MINT>",{"encoding":"jsonParsed"}]}' \
  | jq '.result.value.data.parsed.info.extensions'
```
