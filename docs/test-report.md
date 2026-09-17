# Test Report

## Scope

Manual integration and regression testing of the Resort Guest Operations Copilot across Feishu, n8n, Google Gemini, Supabase PostgreSQL, and pgvector.

- Test date: 2026-09-17
- Environment: portfolio demo
- Knowledge source: `Harborlight_Bay_Resort_Operations_SOP.pdf`, version 1.1
- Test style: node-level checks, sub-workflow integration tests, and end-to-end Feishu tests

## Test strategy

The project uses four layers of verification:

1. **Transformation checks** - Code nodes receive representative inputs and produce the expected typed fields.
2. **Sub-workflow integration checks** - RAG retrieval, structured analysis, audit persistence, and booking handling are tested independently.
3. **End-to-end checks** - A Feishu message is processed through the gateway and produces both a Feishu reply and the expected Supabase state.
4. **Safety and negative-path checks** - Approval, emergency, missing-information, duplicate-event, and invalid-reference behavior is verified explicitly.

## Executed scenarios

| ID | Scenario | Expected route/status | Database assertion | Result |
| --- | --- | --- | --- | --- |
| OPS-01 | Room 615 requests two dental kits | `routine` / `acknowledged` | One operations audit row | Pass |
| OPS-02 | Free late checkout until 14:00 | `manager_approval` / `pending_approval` | Approval flag true; SOP evidence stored | Pass |
| OPS-03 | Strong gas smell in kitchen | `human_escalation` / `escalated` | Escalation flag true; critical safety reply | Pass |
| OPS-04 | Guest asks for a crib | `routine` / `acknowledged` | Reply requires stock check and staff installation | Pass |
| RAG-01 | Late-checkout question | Grounded response | Relevant front-office chunks retrieved | Pass |
| BKG-01 | Complete spa request | `create` / `pending_availability` | New booking-request row | Pass |
| BKG-02 | Spa request missing service/time details | `create` / `pending_information` | Missing fields stored; no confirmation | Pass |
| BKG-03 | Modify an existing booking | `modify` / `modification_requested` | Original retained; linked request appended | Pass |
| BKG-04 | Cancel an existing booking | `cancel` / `cancellation_requested` | Original retained; linked request appended | Pass |
| BKG-05 | Cancel unknown booking reference | `reference_not_found` response | No booking row inserted | Pass |
| MIX-01 | Room odor plus breakfast request | Operational response + booking flow | Operations audit and booking row both created | Pass |

## Representative assertions

### Approval boundary

- The response states that approval is required.
- No refund, compensation, free late checkout, or exception is represented as approved.
- The audit row is stored with `requires_approval = true` and `status = 'pending_approval'`.

### Emergency boundary

- The response prioritizes immediate safety actions.
- `needs_human_escalation` is true only for immediate safety/security hazards.
- The workflow instructs the employee to contact the responsible human authority.

### Booking boundary

- A booking request is never represented as confirmed without external availability data.
- Modify and cancel requests require an existing target reference.
- A nonexistent reference produces a safe response and does not create a booking row.
- Original booking rows are not overwritten by later requests.

### Multi-intent isolation

- The operational reply excludes false booking-status claims.
- The booking workflow receives only the booking-specific request plus limited original context for shared identifiers.
- One consolidated Feishu reply contains clearly separated operational and booking guidance.

## Evidence policy

The public repository contains the reproducible inputs and expected assertions rather than raw production-style execution logs. This avoids publishing chat IDs, user identifiers, webhook paths, or credential metadata. Workflow screenshots demonstrate architecture; the test matrix documents behavior.

## Limitations of this report

- These are manual integration tests, not a continuous automated test suite.
- External provider availability is intentionally not simulated as a successful booking confirmation.
- Load, concurrency, retry, and long-term retention behavior are outside the portfolio-demo scope.

