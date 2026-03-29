# Knowledge Transfer: BaseMessage Wiring in FRMS-COE-LIB

## Introduction

The `FRMS-COE-LIB` library is the shared foundation on which all Tazama fraud-risk-management services depend. Historically, the only transaction shape the library understood was `Pacs002` -- the ISO 20022 Financial Institution to Financial Institution Payment Status Report. Every interface, protobuf definition, Redis serialisation routine, and type guard was built around Pacs002 message type.

The `BaseMessage` initiative changes this. It introduces a flexible, schema-agnostic transaction envelope that can carry **any** non-`Pacs002` message type. Rather than modelling every conceivable ISO 20022 variant (or any other format TAZAMA users would want to use) as a first-class TypeScript interface with a matching protobuf definition, `BaseMessage` treats the transaction payload as an opaque `Record<string, unknown>`. Shape validation is deferred to runtime, where a schema retrieved from Redis governs which fields may be accessed and what types they must conform to. The result is a system that can on-board a new message format without touching library code -- only the schema registry needs updating.

This document traces the `BaseMessage` wiring across four phases of work, with references to the files that were changed and the reasoning behind each decision.

---

## Phase 1 -- Expansion of Interfaces and Protobuf

Phase 1 established the foundational contracts that allow `BaseMessage` to coexist with `Pacs002`. The guiding principle was **surgical change**: every modification had to be non-breaking for downstream services that still operate exclusively on `Pacs002` messages.

### 1.1 The `BaseMessage` and `SupportedTransactionMessage` Interfaces

**File:** `src/interfaces/BaseMessage.ts`

```typescript
export interface BaseMessage {
  TxTp: string;
  TenantId: string;
  MsgId: string;
  Payload: Record<string, unknown>;
  endpointPath?: string;
  DataCache?: DataCache;
}

export type SupportedTransactionMessage = BaseMessage | Pacs002;
```

The interface deliberately mirrors the top-level fields that `Pacs002` already carries (`TxTp`, `TenantId`) so that generic pipeline code can continue to destructure those fields regardless of which variant it is handling. The critical differentiator is `Payload`: a `Record<string, unknown>` that replaces the deeply-typed ISO structure (`FIToFIPmtSts`) with a bag of arbitrary key-value data. This is the core of the flexibility -- any JSON document can be placed inside `Payload`, and the library neither validates nor constrains its shape at compile time.

#### MsgId
The `MsgId` field was added shortly after the initial commit (commit `95468e6`) to ensure every `BaseMessage` carries a unique identifier, which downstream services such as `TypologyProcessor` and `TADP` rely on for deduplication and audit-trail correlation. Initial idea was to keep MsgId for BaseMessage types wherever it is in the Payload. At connection creation time, the users would map the MsgId (or its equivalent) to the transactiondetails.MsgId field, and at ingestion time in DEMS, this transactiondetails.MsgId field would be populated for downstream consumption. At a later discussion, the team decided to introduce a new root level field of MsgId, to be populated by DEMS with the correct value of MsgId from the user's mapping. This was done to clarify the intent of the BaseMessage contract, allowing any future expansion of downstream services from DEMS to retain the clarity on BaseMessage types and keeping code changes to a minimum. However, for Build 1.1, MsgId root level field would be provided by the user, to keep DEMS changes for Build 1.1 to a minimum.

#### endpointPath
The `endpointPath` field was originally introduced on the `MetaData` interface in commit `95468e6`, but it was later moved onto `BaseMessage` itself in commit `961f5d9`.

The reason for this change is that `endpointPath` identifies the schema registry entry for the payload and is therefore an intrinsic property of the transaction itself, rather than part of the surrounding processing metadata. By placing it on `BaseMessage`, any service that receives the transaction can resolve the payload schema directly, without also depending on the `MetaData` envelope.

For Build 1.1, however, `endpointPath` is not applicable. Since SafeObject is no longer in scope for Build 1.1, there is currently no need to use `endpointPath` as part of transaction handling in this delivery.

#### SupportedTransactionMessage
The `SupportedTransactionMessage` union type is the glue that makes the rest of the library polymorphic. Wherever the library previously accepted `Pacs002`, it can now accept `SupportedTransactionMessage`, and type guards (discussed below) narrow the union at runtime.

### 1.2 Modification of the `RuleRequest` Interface

**File:** `src/interfaces/rule/RuleRequest.ts`

```typescript
export interface RuleRequest {
  transaction: SupportedTransactionMessage;
  networkMap: NetworkMap;
  DataCache: DataCache;
  metaData?: MetaData;
}
```

Before this change the `transaction` field was typed as `Pacs002`, which provided no expansion avenue for any non-Pacs002 message types. The Phase 1 update replaced it with `SupportedTransactionMessage`. This is a contract-level change that propagates to every rule executor, typology processor, and TADP consumer: all of them now receive either a `BaseMessage` or a `Pacs002` in a type-safe manner and are expected to use the type guards to determine which variant they have.

The `DataCache` field is worth noting here. Both `BaseMessage` and `Pacs002` transactions carry pre-extracted financial identifiers (debtor/creditor IDs, amounts, exchange rates) in the `DataCache` structure, which is defined in `src/interfaces/rule/DataCache.ts`. For `Pacs002`, the data-preparation service extracts these values from the deeply-typed ISO structure. For `BaseMessage`, if applicable, the extraction would be driven by the schema and occur upstream in the pipeline. The `RuleRequest` therefore remains structurally identical regardless of the underlying message type.

### 1.3 Protobuf Schema Expansion

**File:** `src/helpers/proto/Full.proto`

The protobuf `Transaction` message was extended to carry `BaseMessage` fields alongside the existing ISO structures:

```protobuf
message Transaction {
    string TxTp = 1;
    string TenantId = 6;
    Fitofipmtsts FIToFIPmtSts = 2;       // Pacs002
    Fitoficstmrcdttrf FIToFICstmrCdtTrf = 3;  // Pacs008
    Cstmrcdttrfinitn CstmrCdtTrfInitn = 4;    // Pain001
    Cdtrpmtactvtnreq CdtrPmtActvtnReq = 5;    // Pain013
    string MsgId = 7;
    Payload Payload = 8;
    string endpointPath = 9;
    dataCache DataCache = 10;
}
```

Field numbers `7` through `10` are the new additions. The `Payload` message is deliberately minimal:

```protobuf
message Payload {
    string Json = 1;
}
```

This is an intentional design choice. Protobuf requires a fixed schema, but `BaseMessage` payloads are schema-free at the protobuf layer. The solution is to serialise the entire `Payload` object as a JSON string inside a wrapper message. This means:

- The protobuf wire format gains a single `string` field rather than a recursive definition for every possible payload shape.
- Encoding and decoding must normalise/denormalise the JSON string to/from a plain object (handled in `protobuf.ts`).
- Existing `Pacs002` fields (`FIToFIPmtSts`) remain at their original field numbers, so messages encoded before the expansion can still be decoded without error.

This backward-compatible field numbering is critical. Protobuf's wire format is additive -- unknown fields are silently skipped by older decoders. A downstream service that has not yet upgraded will simply ignore `MsgId`, `Payload`, `endpointPath`, and `DataCache` when decoding a buffer that contains them.

### 1.4 Normalisation and Denormalisation in `protobuf.ts`

**File:** `src/helpers/protobuf.ts`

Three internal helpers were added to handle the impedance mismatch between TypeScript's `Record<string, unknown>` and protobuf's `Payload { string Json }` wrapper:

**`isRecord`** -- a simple type guard that checks `typeof value === 'object' && value !== null`. Used throughout the normalisation logic to safely narrow `unknown` values.

**`normaliseBaseMessagePayload`** -- takes the `data` object that is about to be encoded, inspects `data.transaction`, and if it is a `BaseMessage` (as determined by `isBaseMessageTransaction`), wraps the `Payload` field in `{ Json: JSON.stringify(payload) }`. If the transaction is `Pacs002`, the function returns `data` unmodified. This ensures the existing `Pacs002` encoding path is completely untouched.

```typescript
const normaliseBaseMessagePayload = (data: Record<string, unknown>): Record<string, unknown> => {
  // ...
  if (isPacs002Transaction(transaction)) {
    return data;  // Pacs002 path is untouched
  }
  // ...
  return {
    ...data,
    transaction: {
      ...transaction,
      Payload: { Json: payloadJson },
    },
  };
};
```

The function also guards against double-normalisation: if `Payload` already contains a `Json` key, it returns `data` as-is. This defensive check protects against scenarios where a buffer is re-encoded after partial processing.

**`denormaliseBaseMessagePayload`** -- the inverse operation. After protobuf decoding, if the transaction is not `Pacs002` and `Payload.Json` is a string, it parses the JSON and replaces the `Payload` wrapper with the resulting plain object. A `try/catch` ensures that malformed JSON does not crash the decode pipeline -- instead, the raw transaction is returned unmodified.

These two functions are wired into the public API:

- **`createMessageBuffer`** (lines 102-111) calls `normaliseBaseMessagePayload` before `FRMSMessage.create/encode`.
- **`decodeMessageBuffer`** (lines 119-127) calls `denormaliseBaseMessagePayload` after `FRMSMessage.decode/toObject`.

The `decodeMessageBuffer` function itself is new -- previously, decoding lived in `FRMS-COE-STARTUP-LIB`. Moving it into `FRMS-COE-LIB` alongside the encoding logic ensures that the normalisation and denormalisation are co-located and cannot drift out of sync.

### 1.5 Transaction Type Guards

**File:** `src/helpers/transactionTypeGuards.ts`

Two runtime type guards provide the mechanism for narrowing `SupportedTransactionMessage` at any point in the pipeline:

**`isPacs002Transaction`** checks for `TxTp` (string), `TenantId` (string), and the `FIToFIPmtSts` root object containing both `GrpHdr` and `TxInfAndSts`. This is a structural check, not a nominal one -- it does not rely on `TxTp` having a specific value like `"pacs.002"`, but instead verifies the presence of the deeply-nested ISO structure.

**`isBaseMessageTransaction`** checks for `TxTp`, `TenantId`, `MsgId` (all strings), and `Payload` (object), then **excludes** any object that has a `FIToFIPmtSts` field. The exclusion clause (`!('FIToFIPmtSts' in transaction)`) is the key discriminator: a `Pacs002` message will always have `FIToFIPmtSts`, so any transaction that has `Payload` but lacks `FIToFIPmtSts` is definitively a `BaseMessage`.

This mutual-exclusion design means that the two guards are safe to use in if/else chains without risk of ambiguity. It also means that a malformed object that happens to carry both `Payload` and `FIToFIPmtSts` will be classified as `Pacs002` (since `isPacs002Transaction` does not check for the absence of `Payload`), which is the safer default for backward compatibility.

### 1.6 Non-Breaking Nature of Phase 1

All Phase 1 changes were designed to be non-breaking:

- The `RuleRequest.transaction` field moved from `Pacs002` to `SupportedTransactionMessage`. This ensures that existing code that was already passing `Pacs002` objects continues to compile and function.
- The protobuf `Transaction` message gained new fields at previously unused field numbers. Existing encoded buffers decode without error; existing decoders ignore the new fields.
- `createMessageBuffer` calls `normaliseBaseMessagePayload`, but for `Pacs002` transactions the function is a no-op that returns the input unchanged.
- The new `decodeMessageBuffer` is additive -- it does not replace any existing function.

---

## Phase 2 -- Post-Phase-1 Additions: SafeObject, Schema Enforcement, and Redis Extensions

Phase 2 addressed the question that Phase 1 deliberately left open: if `BaseMessage.Payload` is an opaque `Record<string, unknown>`, how do rule executors safely access its fields without losing type safety or silently reading `undefined` values from missing paths?

### 2.1 The SafeObject Proxy Pattern (Not Applicable for Build 1.1)

**File:** `src/helpers/safeObjectFromSchema.ts`

The `createSafeObjectFromEndpoint` function (line 310) is the centrepiece of Phase 2. It takes an `endpointPath` string and a `payload` object, retrieves the corresponding JSON Schema from Redis, compiles the schema into a path-type map, and returns a `Proxy`-wrapped object (a `SafeObject`) that enforces schema compliance on every property access.

#### 2.1.1 Schema Retrieval (Not Applicable for Build 1.1)

The function first obtains a Redis connection via `getOrCreateRedisService` (lines 266-296). This function implements a two-tier fallback strategy:

1. **Primary:** Uses the `frms-coe-lib` Redis configuration obtained via `validateRedisConfig(false)`. This is the standard path for services that are bootstrapped through `FRMS-COE-STARTUP-LIB`. At the moment this will always fail, because DEMS is configured in a non-standard way to set up the Redis service. When DEMS is patched (details below) this should be considered the default option for Schema retrieval.

2. **Fallback (DEMS-compatible):** If `validateRedisConfig` throws an error (e.g., because the service is EMS and doesn't use the standard configuration), the function will read the following environment variables: `REDIS_HOST`, `REDIS_PORT`, `REDIS_PASSWORD`, `REDIS_DB`, and `REDIS_IS_CLUSTER`. It will then construct a `RedisConfig` from these variables. This approach ensures that EMS, which initializes Redis independently, can still use the SafeObject facility without requiring the full startup library configuration. 
**Note:** In the future, the DEMS behavior should be updated to use the standardized Redis contracts from FRMS-COE-LIB. While this update is not part of Build 1.1, it is considered critical and should be prioritized before the full handover of DEMS to TAZAMA.

Once the Redis connection is established, the schema bundle is retrieved via `getEndpointSchemaBundle` on the `RedisService` (discussed below), or, for lightweight Redis adapters that only expose `getJson`, via direct key lookup and JSON parsing.

#### 2.1.2 Schema Compilation (Not Applicable for Build 1.1)

The retrieved schema bundle is validated by `parseSchemaBundle` (lines 152-166), which checks that the bundle has a `schema` field and that its `publishing_status` (if present) is `"active"`. The schema itself is then compiled by `compileSchemaPaths` (lines 109-150), which performs a recursive depth-first traversal of the JSON Schema tree, recording every reachable property path and its expected type in a `Map<string, string>`:

```
"storyamount"         -> "object"
"storyamount.amount"  -> "number"
"storyamount.ccy"     -> "string"
```

The traversal handles `object` nodes (via `properties`), as well as `anyOf`, `oneOf`, and `allOf` composition keywords. Type inference (`inferNodeType`, lines 82-107) handles arrays, objects, and union types (where the first non-`"null"` type in a `type` array is selected).

Compiled schemas are cached in a module-level `Map` keyed by `endpointPath` (lines 298-308). The cache entry includes a `schemaSignature` (the JSON-stringified schema), so a changed schema triggers recompilation even for the same endpoint path.

#### 2.1.3 Proxy-Based Access Enforcement (Not Applicable for Build 1.1)

The heart of the SafeObject is `createSafeProxy` (lines 219-264). It wraps the payload in a `Proxy` whose `get` trap performs three checks on every property access:

1. **Path existence:** If the accessed path does not appear in the `pathTypeMap`, an error is thrown immediately: `"Path 'x.y.z' does not exist in configured schema"`. This prevents rule authors from accidentally reading a field that the schema does not declare.
2. **Value presence:** If the path exists in the schema but the corresponding value in the payload is `undefined` or `null`, an error is thrown: `"Path 'x.y.z' was not found in payload"`. This catches missing data at the point of access rather than allowing `undefined` to propagate silently through arithmetic or string operations.
3. **Type coercion:** The raw value is passed through `coerceLeafValue` (lines 168-217), which attempts to coerce the value to the expected type. For example, a string `"123"` is coerced to the number `123` if the schema declares `"number"`, and a string `"true"` becomes the boolean `true`. If coercion fails, an error is thrown with a descriptive message.

For `object`-typed paths, the proxy recursively wraps the child object in another `createSafeProxy`, so the enforcement extends to arbitrarily deep nesting. Special `Proxy` traps for `toJSON`, `then`, `catch`, and `finally` ensure the proxy behaves correctly when serialised or when used in `await` expressions.

This fail-fast approach is a deliberate architectural choice. Rules that operate on `Pacs002` benefit from compile-time type safety because every field is modelled in TypeScript. Rules that operate on `BaseMessage` lose that compile-time safety, but the SafeObject recovers equivalent runtime safety. The cost is an additional Redis lookup and proxy indirection; the benefit is that a rule cannot silently produce incorrect results because of a missing or mistyped field.

### 2.2 Redis `getEndpointSchemaBundle` (Not Applicable for Build 1.1)

**File:** `src/services/redis.ts`

The `getEndpointSchemaBundle` method (lines 94-125) was added to `RedisService` to provide a dedicated, validated retrieval path for schema bundles. Rather than leaving callers to call `getJson` and parse the result themselves, this method:

1. Calls `getJson(endpointPath)` to retrieve the raw cached value.
2. Parses the JSON and validates that the result is an object.
3. Checks the `publishing_status` field, rejecting any bundle whose status is not `"active"`.
4. Validates that a `schema` sub-object exists.

By centralising this validation in the `RedisService`, the library ensures that every consumer of schema bundles -- whether `safeObjectFromSchema.ts` or a future caller -- receives a consistent, validated result.

### 2.3 Interface Barrel Export Updates

**File:** `src/interfaces/index.ts`

The barrel file was updated to export `BaseMessage` and `SupportedTransactionMessage` from `./BaseMessage`. The previously exported `TransferAmount` interface was removed (the `TransferAmount.ts` file was deleted entirely), reflecting a cleanup of interfaces that are no longer needed in the expanded message model.

### 2.4 MetaData Interface Simplification

**File:** `src/interfaces/metaData.ts`

The `endpointPath` field was added to `MetaData` in commit `95468e6` and then removed in commit `961f5d9`, leaving `MetaData` with only its original processing-time and trace-parent fields. The move was motivated by the realisation that `endpointPath` is a property of the transaction itself (it determines which schema governs the payload), not of the processing context. Keeping it on `MetaData` would have required every service to forward the metadata envelope alongside the transaction, whereas placing it on `BaseMessage` means the transaction is self-describing.

---

## Phase 3 -- Rule Executor updates with optional Redis Integration

### Note: SafeObject and Schema retrieval, including endpointPath are no longer considered relevant for Build 1.1, meaning some parts of section 3 are not relevant for this build goals

### 3.1 Goals

Phase 3 focuses on connecting the rule executor runtime to the SafeObject facility introduced in Phase 2. The objective is to allow rule code to access `BaseMessage` payload fields through the schema-enforced proxy rather than through raw property access.

In a typical rule executor (`Rule.ts`), the flow is:

1. The rule receives a `RuleRequest` containing a `SupportedTransactionMessage`.
2. If the transaction is a `BaseMessage`, the rule calls `createSafeObjectFromEndpoint` with the transaction's `endpointPath` and `Payload`.
3. The returned `SafeObject` is used for all subsequent field access, ensuring that every read is schema-validated and type-coerced.

This integration means that the rule executor must have access to a Redis connection capable of retrieving schema bundles. The two-tier fallback in `getOrCreateRedisService` (primary `frms-coe-lib` config, fallback EMS-compatible env vars) ensures that the rule executor works in both traditional Tazama deployments and EMS-hosted environments.

### 3.2 Phase 3 Alt -- MsgId Population via EMS

An alternative track within Phase 3 addresses the population of the `MsgId` field on `BaseMessage` transactions. When a transaction originates from EMS, the `MsgId` is generated at ingestion time and attached to the `BaseMessage` before it enters the Tazama pipeline. This is significant because downstream services such as `TypologyProcessor` and `TADP` use `MsgId` as a correlation key when aggregating rule results and producing evaluation reports.

For `Pacs002` transactions, the `MsgId` has always been extracted from the ISO structure (`FIToFIPmtSts.GrpHdr.MsgId`). However, for `BaseMessage`, there is no canonical location for the identifier within the payload, so it must be supplied externally. During connection creation, it is the user's responsibility to map the `MsgId` (or its equivalent) to an appropriate field. Currently, this is mapped to `transactiondetails.MsgId`, but in the future, when the approach is revised to eliminate dependencies on `Pacs002` terminology and behavior, it will be mapped to `transaction.MsgId`.
In any case, DEMS will use this mapping to populate the `MsgId` at the root of the transaction. This ensures that every `BaseMessage` enters the pipeline with a populated `MsgId` before it reaches any rule executor, typology processor, or TADP instance.

This design preserves the invariant that all services downstream of the data-preparation layer can rely on `MsgId` being present and non-empty, regardless of whether the transaction is `Pacs002` or `BaseMessage`.

---

## Phase 4 -- Expansion to CIMS Service

Phase 4 should extend the `BaseMessage` wiring to the **CIMS** layer. The details of this phase are not readily apparent to the author. I am not familiar with CIMS code base, and cannot advise on the adoption path here. However, Reeba can lead the efforts, keeping the intent of this expansion as a guideline on how best to plan and implement this expansion.

## Summary for Build 1.1

Phases 1 through 4 are intended to be part of the Build 1.1 and should therefore be finished by Monday morning. Details of work done/work in progress, and work remaining for each vertical of the ecosystem are given below.

### Event Monitoring Service

No changes needed at the moment. EMS should be moved to the latest versions of frms-coe-lib and frms-coe-startup-lib, but no expansion or patches needed for Build 1.1 here.

### Event Director

Previously, we incorrectly assumed that Event Director would not need to be moved to its own branch. Although the Event Director codebase does not currently require any new features or patches, it still needs to be updated to use the latest `frms-*` libraries.

Without the updated contracts, Event Director silently drops fields during ingress and egress when messages are transported through NATS. These new contracts are already incorporated into the FRMS-COE-STARTUP-LIB `natsService`, and the implementation is designed to minimize downstream code changes.

As a result, Event Director only needs to be moved to the `feat-paysys-poc` branch and have its `package.json` updated to use the newer versions of `FRMS-COE-LIB` and `FRMS-COE-STARTUP-LIB`.

### Rule Executor, including Rule Configs, `Rule.ts` written in Rule Studio, and the Rule Template

Work on the Rule Executor has been completed. Further details are documented elsewhere in this report and in the accompanying reports in the repository. At this stage, only limited changes have been made to the Rule Executor, and the update mainly includes version bumps for `FRMS-COE-LIB` and `FRMS-COE-STARTUP-LIB`.

One small but intentional change was the addition of a log statement at line 32 in `execute.ts` to capture and display incoming transaction details. This log should remain in place until our processes are more robust and we have standardized our approach to handling transport-layer changes.

A more significant change was the addition of Redis configuration to the database manager composition in the Rule Executor. Reeba has temporarily removed this change because of the current mismatch between how EMS sets up and uses Redis and how `FRMS-COE-LIB` defines Redis setup and usage through its contracts. Because of this mismatch, SafeObject is not being included in Build 1.1.

Once the EMS codebase has been cleaned up, we can revisit the SafeObject approach. There are also broader semantic issues with SafeObject that we plan to address as part of the same post-Build 1.1 cleanup work.

`Rule.ts` has now been largely standardized for Build 1.1 and can be treated as functionally complete. Because SafeObject is not being used in this build, the current approach is intentionally straightforward.

Step 1 is to import the new guards from `FRMS-COE-LIB`:

```typescript
import { isPacs002Transaction, isBaseMessageTransaction } from '@tazama-lf/frms-coe-lib';
```

Step 2 is to extract the transaction object from the `RuleRequest`:
```typescript
const transaction = req.transaction;
```

Step 3 is to branch based on the transaction type:
- If the transaction is a `Pacs002` transaction, it can be processed using the existing approach, with the addition of the `Pacs002` guard:
```typescript
    if (isPacs002Transaction(transaction)) {
        const amountVal = transaction.storyamount.amount;
        return determineOutcome(amountVal, ruleConfig, ruleRes);
    }
```
If the transaction is not a `Pacs002` transaction, it should be treated as a `BaseMessage` transaction. In that case, use the `BaseMessage` guard, extract the payload from the transaction, cast it to `Record<string, unknown>`, and then read the required fields in a controlled way. Because `SafeObject` is not currently in use, you must still perform an explicit type check on any primitive field you read. This is a temporary measure:
```typescript
    if (isBaseMessageTransaction(transaction)) {
        const payload = transaction.Payload as Record<string, unknown>;
        const amountRaw = payload.amount;
        if (typeof amountRaw !== 'number') {
            throw new Error('BaseMessage payload missing numeric storyamount.amount');
        }
        return determineOutcome(amountRaw, ruleConfig, ruleRes);
    }
```

Putting it all together, the implementation should look like this:
```typescript
import type { DatabaseManagerInstance, LoggerService, ManagerConfig } from '@tazama-lf/frms-coe-lib';
import type { RuleConfig, RuleRequest, RuleResult } from '@tazama-lf/frms-coe-lib/lib/interfaces';
// import { createSafeObjectFromEndpoint } from '@tazama-lf/frms-coe-lib'; // This will be utilized after Build 1.1
import { isPacs002Transaction, isBaseMessageTransaction } from '@tazama-lf/frms-coe-lib';

export type RuleExecutorConfig = Required<Pick<ManagerConfig, 'rawHistory' | 'eventHistory' | 'configuration' | 'localCacheConfig'>>; // redisCache to be added after Build 1.1

export async function handleTransaction(
    req: RuleRequest,
    determineOutcome: (value: number, ruleConfig: RuleConfig, ruleResult: RuleResult) => RuleResult,
    ruleRes: RuleResult,
    loggerService: LoggerService,
    ruleConfig: RuleConfig,
    databaseManager: DatabaseManagerInstance<RuleExecutorConfig>,
): Promise<RuleResult> {

    if (!ruleConfig.config.bands) {
        throw new Error('Invalid config provided - bands not provided');
    }
    if (!ruleConfig.config.exitConditions) {
        throw new Error('Invalid config provided - exitConditions not provided');
    }
    if (!ruleConfig.config.parameters || typeof ruleConfig.config.parameters.tolerance != 'number') {
        throw new Error('Invalid config provided - tolerance parameter not provided or invalid type');
    }

    // // This was the intended approach for reading BaseMessage payloads using SafeObject.
    // // It will be revisited after Build 1.1 and is expected to become the canonical approach,
    // // with more guards moved into FRMS-COE-LIB.
    //
    // const endpointPath = '/cbe/3.26.015/frms-sessions/fable003';
    // const safeObject = await createSafeObjectFromEndpoint(endpointPath, req.transaction);
    // loggerService.log("${JSON.Stringify(safeObject)}");
    // const amountVal = req.transaction.storyamount.amount;

const transaction = req.transaction;

    // // Pacs002 transactions continue to follow the legacy path, with the addition of the guard.
    // if (isPacs002Transaction(transaction)) {
    //     // Pacs002 path (leave legacy behavior exactly as-is)
    //     // Existing logic continues unchanged
    //     const amountVal = transaction.storyamount.amount; // or your current legacy Pacs002 read path
    //     return determineOutcome(amountVal, ruleConfig, ruleRes);
    // }

    if (isBaseMessageTransaction(transaction)) {
        // BaseMessage transactions are handled by reading directly from Payload for now,
        // since the SafeObject path is deferred until after Build 1.1.
        const payload = transaction.Payload as Record<string, unknown>;

        const amountVal = payload.amount;

        if (typeof amountVal !== 'number') {
            throw new Error('BaseMessage payload missing numeric storyamount.amount');
        }

        return determineOutcome(amountVal, ruleConfig, ruleRes);
    }

    throw new Error('Unsupported transaction type: expected Pacs002 or BaseMessage');

}
```

The Rule Template should always be kept up to date with the latest versions of `FRMS-COE-LIB` and `FRMS-COE-STARTUP-LIB`. Any change to contracts in `FRMS-COE-LIB`, or to the transport mechanism in `FRMS-COE-STARTUP-LIB`, must also be reflected in the Rule Template.

In the past, failing to update the Rule Template accordingly caused significant issues with rule container behavior. This dependency should be clearly understood across the team and communicated wherever appropriate.

## Typology Processor

At present, the Typology Processor remains on the `dev` branch. A new `feat-paysys-poc` branch should be created to keep Build 1.1 deliverables consistent.

This branch should introduce guards for both `Pacs002` and `BaseMessage` transactions. The existing `MsgId` handling for `Pacs002` should remain unchanged. In addition, new logic should be added to read and process `MsgId` for `BaseMessage` transactions.

Because DEMS is not currently configured to populate the `MsgId` field, Build 1.1 will require users to provide the `MsgId` themselves at the root level of the transaction message, as shown below. However, this will change after Build 1.1, to be populated by DEMS after being mapped by the user at connection creation time in Connection Studio:
```JSON
{
  "TxTp": "fable003",
  "TenantId": "cbe",
  "MsgId": "msg009",
  "Payload": {
    "cnic": "1234-5678-910",
    "dates": {
      "storydate": "1970-01-01"
    },
    "msgid": "msg009",
    "amount": 99.9999999,
    "country": "PK",
    "currency": "PKR"
  }
}
```

## Transaction Aggregation Decisioning Processor

### TADP

The changes required in TADP are expected to closely mirror those in the Typology Processor. For consistency with the rest of the Build 1.1 deliverables, a new `feat-paysys-poc` branch should also be created for TADP.

At a minimum, this branch should introduce support for both `Pacs002` and `BaseMessage` transactions through the appropriate transaction guards. The existing handling for `Pacs002` should remain unchanged wherever possible, while equivalent support should be added for `BaseMessage` transactions. In practice, this means ensuring that TADP can correctly recognize the incoming transaction type, preserve the current `Pacs002` behavior, and add the corresponding logic needed to process `BaseMessage` transactions safely and consistently.

Any assumptions already described for the Typology Processor should also be treated as applicable here. In particular, if `MsgId` handling is required in TADP, the `Pacs002` path should continue to use the existing logic, while the `BaseMessage` path should rely on the `MsgId` being supplied at the root of the transaction for Build 1.1. Since DEMS does not yet populate this field automatically, this remains a temporary requirement for users until the wider message-handling approach is cleaned up.

As with the Typology Processor, the goal for Build 1.1 should be to make the smallest possible change set needed to support the new transaction shape without introducing unnecessary divergence in behavior. The implementation should favor targeted updates, maintain backward compatibility for `Pacs002`, and align with the same contract and transport expectations being introduced across the rest of the platform.

---

## Summary of File Responsibilities

| File | Role |
|---|---|
| `src/interfaces/BaseMessage.ts` | Defines the `BaseMessage` interface and the `SupportedTransactionMessage` union type |
| `src/interfaces/rule/RuleRequest.ts` | Consumes `SupportedTransactionMessage` as the `transaction` field for rule evaluation |
| `src/helpers/proto/Full.proto` | Protobuf schema for wire-format serialisation; carries `BaseMessage` fields via `Payload { string Json }` |
| `src/helpers/protobuf.ts` | Encode/decode logic with normalisation and denormalisation of `BaseMessage` payloads |
| `src/helpers/transactionTypeGuards.ts` | Runtime type guards for discriminating `Pacs002` from `BaseMessage` |
| `src/helpers/safeObjectFromSchema.ts` | Schema-driven proxy that enforces field existence, type correctness, and fail-fast access |
| `src/services/redis.ts` | Redis client with `getEndpointSchemaBundle` for schema registry lookups |
