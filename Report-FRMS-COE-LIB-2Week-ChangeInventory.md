# FRMS-COE-LIB Two-Week Change Inventory

## Scope

This report is a **read-only** inventory of changes in `frms-coe-lib` for the last two weeks.

Time window used:
- `git log --since="2 weeks ago"`

Commit filtering:
- non-merge commits only
- **Author filter: Ahmad Khalid commits only**

Requested exclusions applied:
- `src/helpers/proto/Full.proto`
- `src/helpers/safeObjectFromSchema.ts`
- `src/helpers/transactionTypeGuards.ts`
- `src/index.ts`
- `__tests__/safeObjectFromSchema.test.ts`
- `__tests__/safeObjectFromSchema.redisBootstrap.test.ts`
- `__tests__/transactionTypeGuards.test.ts`

---

## File-by-file change inventory

## 1) `src/services/redis.ts`

### Methods / members changed
- Added constants:
  - `PRIMARY_SERVER_INDEX`
  - `EMPTY_MEMBER_COUNT`
  - `MULTI_RESULT_COMMAND_INDEX`
  - `MULTI_RESULT_VALUE_INDEX`
- Added method:
  - `getEndpointSchemaBundle(endpointPath: string): Promise<Record<string, unknown>>`
- Updated method internals:
  - constructor server indexing now uses `PRIMARY_SERVER_INDEX`
  - `getMemberValues` empty check uses `EMPTY_MEMBER_COUNT`
  - `setJson` removed explicit `'OK'` response assertion
  - `set` removed explicit `'OK'` response assertion
  - `setAdd` duplicate-member check uses `EMPTY_MEMBER_COUNT`
  - `addOneGetAll` multi-result indexing now uses named constants
  - `addOneGetCount` multi-result indexing now uses named constants

### Commits
- `f2d2828` (2026-03-27) — feat: updated the lib with safeObject implementation tied to schema from Redis

---

## 2) `src/interfaces/BaseMessage.ts`

### Interfaces / types changed
- Refactor introduced `BaseType` and non-generic `BaseMessage`
- `BaseMessage` now explicitly includes `MsgId: string`
- `Payload` standardized to `Record<string, unknown>` in this period's refactor path
- `DataCache?: DataCache` retained in `BaseMessage`
- `endpointPath?: string` added to `BaseMessage` (moved from metadata)
- `SupportedTransactionMessage` changed to `BaseMessage | Pacs002`
- Legacy/alternative payload type aliases were removed/commented out in this period

### Commits
- `372b3f9` (2026-03-24) — First version of the lib. Need to take a look at frms-coe-startup-lib right after.
- `95468e6` (2026-03-25) — Updated the lib with MsgId for BaseMessage and added an optional field endpointPath in the MetaData interface/descriptor for schema registery implementation
- `961f5d9` (2026-03-25) — Update: Updated the lib to move endpointPath out of MetaData and into the BaseMessage interface/descriptor. also, removed the token added by accident.

---

## 3) `src/interfaces/metaData.ts`

### Interfaces / fields changed
- `endpointPath?: string` was added, then removed from `MetaData`

### Commits
- `95468e6` (2026-03-25) — Updated the lib with MsgId for BaseMessage and added an optional field endpointPath in the MetaData interface/descriptor for schema registery implementation
- `961f5d9` (2026-03-25) — Update: Updated the lib to move endpointPath out of MetaData and into the BaseMessage interface/descriptor. also, removed the token added by accident.

---

## 4) `src/interfaces/index.ts`

### Exports changed
- Export of `TransferAmount` was commented out:
  - `export type * from './TransferAmount'`

### Commits
- `372b3f9` (2026-03-24) — First version of the lib. Need to take a look at frms-coe-startup-lib right after.

---

## 5) `src/interfaces/TransferAmount.ts`

### File-level change
- File deleted in this period (per commit-level name-status)

### Commits
- `372b3f9` (2026-03-24) — First version of the lib. Need to take a look at frms-coe-startup-lib right after.

---

## 6) `src/helpers/protobuf.ts`

### Methods / behavior changed
- Added helper methods:
  - `normaliseBaseMessagePayload`
  - `denormaliseBaseMessagePayload`
- `createMessageBuffer` changed to normalize BaseMessage payload before encoding
- `decodeMessageBuffer` added/updated to denormalize BaseMessage payload after decoding
- Documentation comment added for decode path in this period
- Refactored `normaliseBaseMessagePayload` and `denormaliseBaseMessagePayload` to use `data.transaction` key instead of deprecated `data.BaseMessage`/`data.baseMessage` top-level keys
- Added `isRecord` local helper
- Imported and integrated `isBaseMessageTransaction` and `isPacs002Transaction` type guards from `./transactionTypeGuards`
- Non-Pacs002 transactions missing required BaseMessage fields (`TxTp`, `TenantId`, `MsgId`, `Payload`) now cause `createMessageBuffer` to throw (returns `undefined`) instead of silently encoding

### Commits
- `372b3f9` (2026-03-24) — First version of the lib. Need to take a look at frms-coe-startup-lib right after.
- `d62b436` (2026-03-29) — Feat: added type guards for Pacs002 and BaseMessage, cleaned up protobuf.ts for BaseMessage to make the behaviour consistent, made some changes to SafeObject but SafeObject is on hold until after Build 1.1

---

## 7) `__tests__/utils.test.ts`

### Test blocks changed
- Added imports for:
  - `createMessageBuffer`
  - `decodeMessageBuffer`
- Added describe block:
  - `Should serialise/deserialise BaseMessage payload`
- Added test cases:
  - `se/deserialise BaseMessage with dynamic payload object`
  - `keeps raw payload when BaseMessage.Payload.Json is malformed`
- Refactored existing test cases to use `transaction` key instead of `BaseMessage` top-level key
- Renamed test descriptions to reference "transaction BaseMessage"
- Test data updated to include `MsgId` and `endpointPath` fields
- Assertions updated to verify round-trip via `decoded?.transaction` instead of `decoded?.BaseMessage`
- Added new test cases:
  - `rejects non-pacs002 transaction when MsgId is missing`
  - `rejects non-pacs002 transaction when Payload is missing`
  - `rejects deprecated top-level BaseMessage shape`

### Commits
- `372b3f9` (2026-03-24) — First version of the lib. Need to take a look at frms-coe-startup-lib right after.
- `d62b436` (2026-03-29) — Feat: added type guards for Pacs002 and BaseMessage, cleaned up protobuf.ts for BaseMessage to make the behaviour consistent, made some changes to SafeObject but SafeObject is on hold until after Build 1.1

---

## 8) `README.md`

### Documentation sections changed
- Added table-of-contents entry for:
  - `5. Schema-Safe Dot Notation Access`
- Added new section documenting:
  - `createSafeObjectFromEndpoint` usage
  - endpoint key requirements
  - fail-fast behavior expectations
- Expanded section 5 with:
  - `EndpointPath Contract` subsection (replacing shorter "Endpoint key requirements")
  - `Runtime Guarantees (Fail-Fast)` subsection (replacing "Failure policy")
  - `Temporary POC Compatibility Note` documenting EMS-style Redis env var fallback
  - `Canonical BaseMessage transaction shape` JSON example showing required `transaction` envelope with `TxTp`, `TenantId`, `MsgId`, `endpointPath`, and `Payload`
  - Updated usage example to include `currency` access
  - Added prose clarifying required vs optional BaseMessage fields for non-Pacs002 flows

### Commits
- `f2d2828` (2026-03-27) — feat: updated the lib with safeObject implementation tied to schema from Redis
- `d62b436` (2026-03-29) — Feat: added type guards for Pacs002 and BaseMessage, cleaned up protobuf.ts for BaseMessage to make the behaviour consistent, made some changes to SafeObject but SafeObject is on hold until after Build 1.1

---

## 9) `.npmrc`

### Config change
- Registry auth token template changed from `${GH_TOKEN}` to `{{GH_TOKEN}}`

### Commits
- `372b3f9` (2026-03-24) — First version of the lib. Need to take a look at frms-coe-startup-lib right after.

---

## 10) `package.json`

### Metadata / scripts changed
- Version bumps in this period (latest: `7.0.1-rc.ak.17`)
- `publish:dev` tag changed from `psl-dev-ahmad-khalid` to `psl`
- Changes tracked as package metadata/script updates (no runtime method/interface symbols)

### Commits
- `372b3f9` (2026-03-24) — First version of the lib. Need to take a look at frms-coe-startup-lib right after.
- `961f5d9` (2026-03-25) — Update: Updated the lib to move endpointPath out of MetaData and into the BaseMessage interface/descriptor. also, removed the token added by accident.
- `f2d2828` (2026-03-27) — feat: updated the lib with safeObject implementation tied to schema from Redis
- `d62b436` (2026-03-29) — Feat: added type guards for Pacs002 and BaseMessage, cleaned up protobuf.ts for BaseMessage to make the behaviour consistent, made some changes to SafeObject but SafeObject is on hold until after Build 1.1

---

## 11) `package-lock.json`

### Metadata / dependency lock changes

- Lockfile updates corresponding to package/version/script/dependency-state changes in the same commit stream
- Changes are lock-state only (no runtime method/interface symbols)

### Commits

- `372b3f9` (2026-03-24) — First version of the lib. Need to take a look at frms-coe-startup-lib right after.
- `961f5d9` (2026-03-25) — Update: Updated the lib to move endpointPath out of MetaData and into the BaseMessage interface/descriptor. also, removed the token added by accident.
- `f2d2828` (2026-03-27) — feat: updated the lib with safeObject implementation tied to schema from Redis
- `d62b436` (2026-03-29) — Feat: added type guards for Pacs002 and BaseMessage, cleaned up protobuf.ts for BaseMessage to make the behaviour consistent, made some changes to SafeObject but SafeObject is on hold until after Build 1.1

---

## Notes

- This report is based strictly on git history within the specified two-week window and the exclusions provided.
- Merge commits were intentionally omitted to avoid duplicate/cherry-picked noise and keep the inventory focused on direct code-change commits.
- Only commits by Ahmad Khalid are included.
