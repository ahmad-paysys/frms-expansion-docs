# FRMS-COE-LIB Two-Week Functional Delta

## Scope

This appendix contains only **functional/runtime-impact** changes in `frms-coe-lib` over the last two weeks.

Included:
- source files affecting runtime behavior (builders/helpers/services/interfaces)

Excluded from this appendix:
- tests
- docs/README
- `.npmrc`
- `package.json`
- `package-lock.json`
- previously requested path exclusions:
  - `src/helpers/proto/Full.proto`
  - `src/helpers/safeObjectFromSchema.ts`
  - `src/helpers/transactionTypeGuards.ts`
  - `src/index.ts`

**Author filter: Ahmad Khalid commits only**

---

## Runtime-impact summary

| File | Runtime-impact delta | Commit(s) |
|---|---|---|
| `src/services/redis.ts` | Added schema-bundle retrieval API `getEndpointSchemaBundle(endpointPath)`; internal behavior cleanup via constants for indexing and empty checks; removed explicit `'OK'` validation in `setJson`/`set` paths | `f2d2828` |
| `src/helpers/protobuf.ts` | Added BaseMessage payload normalization/denormalization in buffer encode/decode flow (`createMessageBuffer`, `decodeMessageBuffer`) | `372b3f9` |
| `src/interfaces/BaseMessage.ts` | Contract-level transaction shape changes (required `MsgId`, optional `endpointPath`, payload shape/union evolution) impacting consumers at compile/runtime integration boundaries | `372b3f9`, `95468e6`, `961f5d9` |
| `src/interfaces/metaData.ts` | `endpointPath` field added then removed (moved to BaseMessage), changing where endpoint identity is expected | `95468e6`, `961f5d9` |
| `src/interfaces/index.ts` | Removed `TransferAmount` export from barrel, affecting import availability for downstream consumers | `372b3f9` |
| `src/interfaces/TransferAmount.ts` | Removed file, affecting compile-time contract availability where referenced | `372b3f9` |

---

## Functional risk notes (quick read)

- **High integration sensitivity:** `BaseMessage`/`MetaData` contract moves (`MsgId`, `endpointPath`) can break consumers expecting prior shape.
- **Transport payload handling:** `protobuf.ts` normalization/denormalization modifies effective encoded/decoded shape for BaseMessage-style payloads.

---

## Commit index (runtime files only)

- `f2d2828` — feat: updated the lib with safeObject implementation tied to schema from Redis
- `961f5d9` — Update: Updated the lib to move endpointPath out of MetaData and into the BaseMessage interface/descriptor. also, removed the token added by accident.
- `95468e6` — Updated the lib with MsgId for BaseMessage and added an optional field endpointPath in the MetaData interface/descriptor for schema registery implementation
- `372b3f9` — First version of the lib. Need to take a look at frms-coe-startup-lib right after.
