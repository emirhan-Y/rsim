# PC-Based DKM Simulator â€” Requirements Document

## 1. Purpose

The DKM (Downloadable Kernel Module) running on the VxWorks target receives multiple types of input messages, processes them, and produces output messages. Today, the only way to exercise the DKM is to write full binary files of input messages, load them onto the target, and inspect the resulting output binaries in a separate analysis application. This makes it impractical to:

- Hand-craft or tweak individual input messages for targeted testing
- Construct edge-case / unusual inputs that don't exist in captured binaries
- Iterate quickly (each test cycle requires binary editing + a full DKM run)

This document defines requirements for a **PC-based simulator** that connects to the DKM over TCP, lets an engineer view and edit input messages in a human-readable form, sends them to the DKM, and captures/displays the DKM's output messages in real time.

## 2. Goals

- **G1 â€” Struct independence:** The simulator's core logic must not need to change when a message struct is added or modified. Only a schema description needs to change.
- **G2 â€” Human-editable messages:** Input messages (including dynamically-sized fields) must be viewable and editable field-by-field in a UI, not as raw bytes.
- **G3 â€” Live stimulation:** The simulator must be able to send messages to the DKM over TCP, replacing (or supplementing) the binary-file workflow.
- **G4 â€” Output capture:** The simulator must listen for and decode DKM output over TCP, displaying messages the same way input messages are shown.
- **G5 â€” Binary file compatibility:** The simulator must still be able to load and parse the existing input/output binary files, for continuity with current data and workflows.

## 3. Non-Goals / Out of Scope (proposed â€” confirm)

- Replacing the existing analysis application's deeper analytics features (this tool is for stimulation and quick inspection, not the full analysis suite).
- Modifying the DKM itself.
- Automated test-case generation / fuzzing (could be a future extension, not a v1 requirement).

## 4. Key Architectural Decision: Schema-Driven Message Handling

This is the mechanism that delivers Goal G1, and needs to be nailed down early since it drives everything else.

**Problem:** Message struct definitions live in C/C++ headers on the DKM side. The simulator is to be written primarily in Java, which cannot consume C/C++ headers directly, and hand-porting struct layouts to Java for every message type would defeat the "generalizes automatically" goal.

**Proposed approach:**

1. Define an external, human-readable **message schema** (e.g. JSON or YAML) that is the single source of truth the simulator reads at runtime. For each message type, the schema describes:
   - Message type ID / name
   - Ordered list of fixed fields (name, primitive type, size, endianness)
   - Any dynamic/repeating groups: the count field that determines repetition, and the field layout of each repeated element (as in the `POINT_MSG` example: N IDs followed by N data points, each with its own fields)
2. Build a small **schema-generation tool** that parses the C/C++ headers and produces/updates this schema file, so the headers remain the true origin of struct definitions but the simulator never touches C++ directly. (This tool can be a separate utility â€” Python or a lightweight C++/clang-based parser â€” run manually or as part of a build step whenever headers change.)
3. The Java simulator loads only the schema file. Adding or changing a message type means regenerating (or hand-editing, if the codegen tool doesn't yet support a particular pattern) the schema â€” **no simulator code changes required**.

**Open question for you:** Are you open to building/maintaining a small codegen tool to keep the schema in sync with the headers, or would you rather hand-author and hand-maintain the schema file directly (simpler to build, but a second place to update when structs change)? The rest of this document assumes the schema-file approach either way â€” this only affects how the schema gets populated.

## 5. Functional Requirements

### 5.1 Message Schema Engine
- FR-1: Load a schema file describing all known message types, their fixed fields, and any dynamic/repeating field groups.
- FR-2: Given a schema and a raw byte buffer, decode it into a structured, named-field representation.
- FR-3: Given a schema and a structured, named-field representation (including user edits), re-encode it into the correct raw byte layout (correct sizes, byte order, and repeat counts).
- FR-4: Support the field types actually used by the DKM (e.g. int8/16/32/64, uint variants, float, double, fixed-size arrays/strings, etc. â€” to be enumerated against your real structs).
- FR-5: Validate edited values against field constraints (type range, array length vs. declared count field) before allowing a message to be sent or saved.

### 5.2 Input Binary File Handling
- FR-6: Load an existing input binary file and split it into its constituent messages using the schema (message type identification, e.g. via a header/type ID field â€” needs confirming against your file format).
- FR-7: Display the parsed messages as a list (by type, sequence, timestamp if present).
- FR-8: Allow selecting a message from the list and viewing/editing every field, including add/remove of repeated elements (e.g. adding a 3rd data point to a `POINT_MSG`).
- FR-9: Allow creating a brand-new message from scratch (not just editing an existing one), for constructing test cases that don't exist in any captured binary.
- FR-10: Allow saving the (possibly edited) message set back out as a binary file, in the original format, for continuity with existing tools.

### 5.3 TCP Communication â€” Stimulus (Simulator â†’ DKM)
- FR-11: Connect to the DKM over TCP (configurable host/port).
- FR-12: Send one or more selected/edited messages to the DKM, encoded per the wire protocol.
- FR-13: Support sending messages individually (on demand) and as a scripted/batch sequence (e.g. replay a whole loaded file, optionally with timing/delay control between messages).
- FR-14: Surface connection state and send errors in the UI.

### 5.4 TCP Communication â€” Output Capture (DKM â†’ Simulator)
- FR-15: Listen on TCP for output messages from the DKM (configurable host/port, or reuse of the stimulus connection if the DKM uses a single bidirectional socket â€” **needs confirming**).
- FR-16: Decode incoming output messages using the same schema engine as input.
- FR-17: Display captured output messages live, in the same field-exposed format as input messages.
- FR-18: Allow saving captured output to a binary file compatible with the existing output-binary format, for continued use with the existing analysis app.

### 5.5 UI
- FR-19: List view of loaded/captured messages (input and output), filterable/sortable by type.
- FR-20: Detail/edit view showing all fields of a selected message, with appropriate input controls per field type (numeric fields, repeating group add/remove, etc.).
- FR-21: Clear visual distinction between input (to be sent / editable) and output (received / read-only, or edit only for re-injection experiments) messages.
- FR-22: Basic session/log view showing what was sent and received, in order, with timestamps.

## 6. Non-Functional Requirements

- NFR-1: **Extensibility** â€” adding a new message type must require only a schema update (per Section 4), not simulator code changes, in the common case.
- NFR-2: **Platform** â€” runs on the PC used for analysis (confirm OS â€” Windows/Linux) with Java as primary implementation language.
- NFR-3: **Performance** â€” must handle the message volumes seen in real capture files/sessions without UI lag (rough volume/rate numbers TBD â€” see open questions).
- NFR-4: **Data fidelity** â€” encode/decode round-trip must be byte-exact for unmodified fields, so re-saved binaries remain valid for existing downstream tools.
- NFR-5: **Robustness** â€” malformed/unknown messages (e.g. from a schema mismatch) should be reported clearly rather than silently corrupted or crashing the tool.

## 7. Assumptions

- A1: Message type can be identified from a message's header/leading bytes without needing full context (i.e., message boundaries and types are self-describing in both the binary files and the TCP stream).
- A2: The dynamic-count fields (e.g. "N" in a POINT_MSG) are always explicit fields within the message itself, not implied by external context or total message length alone.
- A3: The DKM's wire protocol and the binary file format are the same encoding (or a documented mapping exists between them).

## 8. Open Questions / Risks

- **TCP protocol details (partially known):** framing (length-prefixed? delimiter? fixed header with type + length?), byte order, whether input and output use separate sockets/ports or one bidirectional connection, and how a message boundary is detected on the wire. This needs to be nailed down (or reverse-engineered from captured traffic) before FR-11â€“FR-18 can be finalized.
- **Binary file format:** is there an existing spec for how messages are delimited/typed inside the input/output binary files, or does this also need reverse engineering?
- **Schema source-of-truth process:** hand-maintained schema file vs. a header-parsing codegen tool (Section 4) â€” needs a decision, as it affects how much tooling this project includes.
- **Message volume/performance targets:** expected number of messages per session/file, and any real-time rate requirements for the output listener.
- **Scripting/automation needs:** is a GUI-only tool sufficient, or is there a need to eventually script test sequences (e.g. for regression suites) â€” relevant to whether the schema/engine should be usable headlessly, separate from the UI.

## 9. Suggested Phasing (proposed)

1. **Phase 1 â€” Core engine:** schema format definition + decode/encode engine, proven against a couple of real message types (including a dynamic-field type like POINT_MSG) using existing binary files (no TCP yet).
2. **Phase 2 â€” File UI:** load/view/edit/save binary files through the UI (FR-6â€“FR-10, FR-19â€“FR-21).
3. **Phase 3 â€” TCP stimulus:** connect and send to the DKM (FR-11â€“FR-14), once protocol details are confirmed.
4. **Phase 4 â€” TCP capture:** listen for and display output (FR-15â€“FR-18).
5. **Phase 5 â€” polish:** batch/scripted sends, session logging, validation refinements.