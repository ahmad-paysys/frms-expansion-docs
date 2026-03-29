# FRMS-COE-LIB Two-Week Change Inventory — Compact Table

## Scope

- Repository: `frms-coe-lib`
- Window: last 2 weeks (`git log --since="2 weeks ago"`)
- Commit filter: non-merge commits
- **Author filter: Ahmad Khalid commits only**
- Exclusions applied:
  - `src/helpers/proto/Full.proto`
  - `src/helpers/safeObjectFromSchema.ts`
  - `src/helpers/transactionTypeGuards.ts`
  - `src/index.ts`
  - `__tests__/safeObjectFromSchema.test.ts`
  - `__tests__/safeObjectFromSchema.redisBootstrap.test.ts`
  - `__tests__/transactionTypeGuards.test.ts`

---

## Source & Interface Changes

| File | Methods / Interfaces / Members changed | Commit | Commit message |
|---|---|---|---|
| `src/services/redis.ts` | Added `getEndpointSchemaBundle(endpointPath)`; introduced constants `PRIMARY_SERVER_INDEX`, `EMPTY_MEMBER_COUNT`, `MULTI_RESULT_COMMAND_INDEX`, `MULTI_RESULT_VALUE_INDEX`; updated internals of constructor, `getMemberValues`, `setJson`, `set`, `setAdd`, `addOneGetAll`, `addOneGetCount` | `f2d2828` | feat: updated the lib with safeObject implementation tied to schema from Redis |
| `src/interfaces/BaseMessage.ts` | Refactor from generic to explicit `BaseMessage`; introduced `BaseType`; added required `MsgId`; standardized `Payload`; moved/added optional `endpointPath`; adjusted `SupportedTransactionMessage` union | `372b3f9` | First version of the lib. Need to take a look at frms-coe-startup-lib right after. |
| `src/interfaces/BaseMessage.ts` | Added `MsgId`; refactor continued around BaseMessage shape | `95468e6` | Updated the lib with MsgId for BaseMessage and added an optional field endpointPath in the MetaData interface/descriptor for schema registery implementation |
| `src/interfaces/BaseMessage.ts` | Added `endpointPath?: string` in BaseMessage (moved from metadata) | `961f5d9` | Update: Updated the lib to move endpointPath out of MetaData and into the BaseMessage interface/descriptor. also, removed the token added by accident. |
| `src/interfaces/metaData.ts` | Added `endpointPath?: string` | `95468e6` | Updated the lib with MsgId for BaseMessage and added an optional field endpointPath in the MetaData interface/descriptor for schema registery implementation |
| `src/interfaces/metaData.ts` | Removed `endpointPath?: string` (moved to BaseMessage) | `961f5d9` | Update: Updated the lib to move endpointPath out of MetaData and into the BaseMessage interface/descriptor. also, removed the token added by accident. |
| `src/interfaces/index.ts` | Commented out export of `TransferAmount` | `372b3f9` | First version of the lib. Need to take a look at frms-coe-startup-lib right after. |
| `src/interfaces/TransferAmount.ts` | File removed (name-status showed deletion in period) | `372b3f9` | First version of the lib. Need to take a look at frms-coe-startup-lib right after. |
| `src/helpers/protobuf.ts` | Added `normaliseBaseMessagePayload` and `denormaliseBaseMessagePayload`; updated `createMessageBuffer` normalization path; added/updated `decodeMessageBuffer` denormalization path | `372b3f9` | First version of the lib. Need to take a look at frms-coe-startup-lib right after. |
| `src/helpers/protobuf.ts` | Refactored normalise/denormalise to use `data.transaction` key (removed deprecated `data.BaseMessage`/`data.baseMessage`); added `isRecord` helper; imported `isBaseMessageTransaction`/`isPacs002Transaction` type guards; non-Pacs002 transactions missing required fields now throw | `d62b436` | Feat: added type guards for Pacs002 and BaseMessage, cleaned up protobuf.ts for BaseMessage to make the behaviour consistent, made some changes to SafeObject but SafeObject is on hold until after Build 1.1 |

---

## Test & Documentation Changes

| File | Test blocks / sections changed | Commit | Commit message |
|---|---|---|---|
| `__tests__/utils.test.ts` | Added `createMessageBuffer` and `decodeMessageBuffer` imports; added BaseMessage serialization/deserialization describe block + two test cases | `372b3f9` | First version of the lib. Need to take a look at frms-coe-startup-lib right after. |
| `__tests__/utils.test.ts` | Refactored tests to use `transaction` key instead of `BaseMessage`; added `MsgId`/`endpointPath` to test data; added three new test cases: missing `MsgId` rejection, missing `Payload` rejection, deprecated top-level `BaseMessage` shape rejection | `d62b436` | Feat: added type guards for Pacs002 and BaseMessage, cleaned up protobuf.ts for BaseMessage to make the behaviour consistent, made some changes to SafeObject but SafeObject is on hold until after Build 1.1 |
| `README.md` | Added ToC entry and section `Schema-Safe Dot Notation Access` with usage, endpoint key requirements, fail-fast policy | `f2d2828` | feat: updated the lib with safeObject implementation tied to schema from Redis |
| `README.md` | Expanded section 5: `EndpointPath Contract`, `Runtime Guarantees (Fail-Fast)`, `Temporary POC Compatibility Note` (EMS Redis fallback), canonical BaseMessage JSON example, required vs optional field prose | `d62b436` | Feat: added type guards for Pacs002 and BaseMessage, cleaned up protobuf.ts for BaseMessage to make the behaviour consistent, made some changes to SafeObject but SafeObject is on hold until after Build 1.1 |
| `.npmrc` | Registry auth token template changed from `${GH_TOKEN}` to `{{GH_TOKEN}}` | `372b3f9` | First version of the lib. Need to take a look at frms-coe-startup-lib right after. |

---

## Package Metadata / Lock Changes

| File | Change type | Commits |
|---|---|---|
| `package.json` | Version bump to `7.0.1-rc.ak.17`; `publish:dev` tag changed to `psl` (no runtime symbols) | `372b3f9`, `961f5d9`, `f2d2828`, `d62b436` |
| `package-lock.json` | Dependency lock-state updates (no runtime symbols) | `372b3f9`, `961f5d9`, `f2d2828`, `d62b436` |

---

## Notes

- This compact report is derived from git history and the detailed inventory report.
- It is optimized for quick review in PRs and status threads.
- Only commits by Ahmad Khalid are included.
