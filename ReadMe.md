# FRMS Expansion Reports

This directory contains documentation and change reports for the BaseMessage expansion in `FRMS-COE-LIB`. The expansion introduces `BaseMessage` -- a flexible, schema-agnostic transaction envelope that allows Tazama to process any non-`Pacs002` message type without requiring new TypeScript interfaces or protobuf definitions per format. Payload validation is deferred to runtime via schemas stored in Redis, enabling new message formats to be on-boarded through configuration alone.

---

## Documents

### [KT-FRMS-COE-LIB.md](KT-FRMS-COE-LIB.md)

Knowledge transfer document covering the full BaseMessage wiring across four phases: interface and protobuf expansion, SafeObject schema enforcement, rule executor Redis integration, and CIMS service extension. Explains the design decisions, type guards, protobuf normalisation, and the SafeObject proxy pattern with references to each source file involved.

### [SundayStatus.md](SundayStatus.md)

Everything in the works for Build 1.1, and what we got done over the weekend.

### [Report-FRMS-COE-LIB-2Week-FunctionalDelta.md](Report-FRMS-COE-LIB-2Week-FunctionalDelta.md)

Focused summary of runtime-impacting changes only. Lists each modified source file alongside its functional delta and associated commits. Includes risk notes highlighting integration-sensitive contract changes and breaking wire-format shifts. Designed for quick assessment of what changed and what could break.

### [Report-FRMS-COE-LIB-2Week-ChangeInventory.md](Report-FRMS-COE-LIB-2Week-ChangeInventory.md)

Detailed file-by-file inventory of all changes within the two-week window. Each section lists the specific methods, interfaces, exports, or test blocks that were added, modified, or removed, with full commit references and dates. Covers source, tests, documentation, and package metadata.

### [Report-FRMS-COE-LIB-2Week-ChangeInventory-Compact.md](Report-FRMS-COE-LIB-2Week-ChangeInventory-Compact.md)

Condensed table version of the full change inventory. Presents the same information in three tables -- source/interface changes, test/documentation changes, and package metadata -- for quick scanning in PR reviews or status updates.
