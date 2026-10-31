# Commonplace boards: design

Design, 2026-10-30. Nothing built yet. Milestone B in [PRODUCT-DESIGN.md](../../PRODUCT-DESIGN.md) §15.

Files:

- this doc: what the board is, how cards move, what Avery sees
- [schema.sql](schema.sql): the data model (runs as is on SQLite 3.35+)
- [API.md](API.md): the MCP tools agents call and the HTTP calls the web UI makes

This builds on PRODUCT-DESIGN.md §10 and keeps all of it: kanzo's design and none of its code, `boards.sqlite3` in the workspace palace, the §7 stamp on every write, memory scopes as board scopes, `backlog` as the boards' inbox, the change counter, links to drawers. Where they differ, this doc wins. It adds three things §10 left open:

1. **Claims that expire.** An agent that crashes mid-task no longer holds its card forever.
2. **Handoffs.** An agent can pass a card to another agent, back to the pool, or to a person, with a note.
3. **A `review` column and a decide queue.** Anything waiting on Avery sits in one place, and he answers it in one click.

## Goals

1. Agents from any provider pick up work, report progress and hand it on without a person relaying messages.
2. Two agents never work on the same card.
3. Everything waiting on Avery is on one screen, oldest first, and each item takes one decision.
4. Every card, move and comment shows who did it, stamped by the server.
5. A task filed by an untrusted bot never reaches an agent with a shell until a person lets it through.

**Non-goals for v1:** sub-tasks, due dates, labels, estimates, swimlanes, per-board permissions beyond the wing, the sealed tier for boards, starting agents from a card (that is module A).

## Columns

```
backlog │ todo │ doing │ review │ done
```

| Column | Holds | Who puts cards here |
|---|---|---|
| `backlog` | Ideas, and anything filed by an `inbox` token. Trusted, unrestricted board writers may promote or explicitly take available cards of unrestricted origin without a pending person decision. | Untrusted tokens and cloud clients (always), anyone (on purpose) |
| `todo` | Ready work. A card may be reserved for one agent. | People (from backlog or new), trusted and standard agents, decisions |
| `doing` | Held by exactly one actor. An agent's hold expires. | `task_take_next`, or a person dragging a card |
| `review` | An open ask to a person: a question, or finished work to check. | `ask`, `task_update`, or `task_handoff` to a person |
| `done` | Closed. Never deleted. | The holder, a person, or a decision |

`review` is the one column added to kanzo's set. A question to Avery could have been a flag on a `doing` card, but then waiting work looks like moving work, and the claim keeps ticking on a card nobody can move. A column makes waiting visible on the board and gives the decide queue a single query.

## Card lifecycle

```
            person                 task_take_next           task_update done
 backlog ───────────▶ todo ───────────────────────▶ doing ───────────────────▶ done
    ▲                ▲  ▲                            │ │
    │                │  │ claim expires              │ │ task_handoff to a person
    │                │  └────────────────────────────┤ ▼
    │                │    task_handoff anyone/agent  │ review ──decide──▶ todo (back to asker) │ done │ backlog
    └────────────────┴───────────────────────────────┘
```

The full table of who may make each move is in [API.md](API.md#move-rules). Three rules carry the design:

- **People and trusted, unrestricted board writers may move available cards out of `backlog`; restricted-origin submissions and cards waiting on a person require a person.** Origin restrictions are recorded when a card is filed, including trusted inbox submissions. Legacy agent backlog uses its trust stamp and the stamping connection's current policy in the same workspace; untrusted stamps and missing, inactive or ambiguous origins remain person-only. Automatic pickup stays on `todo`, and backlog promotion preserves the caller's reservation. Only the asking agent may close a `review` card via `task_update`; person-directed review returns require `decide`. Withdrawal handling is prepared for future agent-directed asks under the rules below. An agent can never answer its own question. Untrusted and inbox-restricted connections still only submit cards and comments.
- **Only the holder changes a `doing` card.** Others can comment.
- **Cards are never deleted.** Closing one with a reason is a move to `done`. The event log is append-only (enforced by triggers in `schema.sql`).

Trusted, unrestricted board writers may edit title, body, checks, labels and
memory links on review cards they requested, preserving the open ask. The
edit event keeps prior changed values as JSON, including visible link IDs.
MCP decisions may supply the version read to reject a stale answer after an edit.

Current tools and HTTP routes create only person-directed reviews and clear
the assignee. Assignee edits and withdrawal are prepared for agent-to-agent
review asks once a creation path exists. For such asks, the assignee or
requester may return an agent-directed review
to `todo` or `doing` by withdrawing its ask in the same transaction. Person
recipients (including the default owner), person assignees and person return
targets block withdrawal and keep their decision queue intact. Only the
requester may complete a review card, and `decide` still refuses an agent's
own ask.

Withdrawal preserves the review/open-ask database constraint. A `release`
event stores the original ask and optional verbatim reason, with actor, time,
recipient, options and statuses. `card.ask_withdrawn` notifies readers alongside
the normal move event; it does not record or imply an answer. Standard-trust,
untrusted and inbox-restricted agents gain no feedback permissions.

## How agents claim

`task_take_next` is the only way an agent starts a card. One `BEGIN IMMEDIATE` transaction:

1. Expires stale claims on the board, logging an `expire` event that names the agent that lost it.
2. Claims the caller's reserved card with the lowest position, or else the free card with the lowest position.
3. Logs a `claim` event.

The race check behind this: four processes claiming from 2,001 cards, interleaved (1 / 11 / 653 / 1,336 claims each), got 2,001 distinct cards and expired the one stale claim. B's done test repeats this with real Claude Code and Codex sessions.

**Claims expire.** A claim lasts `claim_minutes` (30 by default, per board). Any write by the holder renews it: a comment, an update, a handoff. Expiry is lazy: the next board transaction frees stale cards, so no background job is needed. An expired card goes back to the lane it was claimed from (`todo` or `backlog`), free, with an `expire` event naming its former holder. Explicit backlog selection cannot turn into automatic work through expiry. Claims created before lane tracking return to `todo` as before.

People's holds never expire. Avery dragging a card into `doing` means he is on it.

**Picking the board.** The hooks pass the git remote, and `board_repos` maps it to a board (kanzo's project detection). Agents outside a repo name the board.

## How agents report and hand off

- **Progress:** `task_comment`. Short, verbatim, stamped. It is the log the next agent reads.
- **Finish:** `task_update status=done`, optionally with `lesson`, which files a drawer through the normal memory write path and links it both ways. On boards with `done_needs_review`, this lands in `review` as "Done?" instead.
- **Hand off** with `task_handoff to=… note=…`. The note is required.
  - `to=anyone`: back to `todo`, free.
  - `to=codex:vm`: back to `todo`, reserved for that connection. Only it (or a person) can start it.
  - `to=avery`: into `review`. The note becomes the question, `options` become buttons, and the card remembers who asked (`return_to`).
- **Ask before starting:** an agent can hand a free or self-reserved `todo` card to a person. That is how it says "this is unclear" without claiming.

After a decision, the card goes back to `todo`, reserved for the agent that asked. Its next `task_take_next` returns that card first, with the decision on a line of its own. The agent does not wait or poll: it hands off, ends its turn, and the session-start hook line tells it when something is back (API.md, "Hook line").

## What Avery sees

Two screens in the Commonplace web app, same login. The rule from the product brief applies: no taglines, no helper text, no secondary labels. Where a screen would need them, the layout carries the meaning.

### Decide

The screen he opens first. The nav shows its count. One list across all boards, oldest first, in two groups: cards in `review` that ask him, then `backlog` cards filed by agents.

```
Decide 4
┌──────────────────────────────────────────────────────────────────────────┐
│ mempalace  #145  Pick the token expiry default         claude-code:vm 2h │
│ "1 year or 90 days for personal tokens?"                                 │
│ [ 1y ]  [ 90d ]  [________________________________]  ↩                   │
├──────────────────────────────────────────────────────────────────────────┤
│ mempalace  #151  Done?  Split the hook output          codex:vm      40m │
│ [ ship ]  [ redo ]  [________________________________]  ↩                │
├──────────────────────────────────────────────────────────────────────────┤
│ mempalace  #153  Add Grok calendar sync                coordinator ⚠ 1d  │
│ [ todo ]  [ close ]                                                      │
└──────────────────────────────────────────────────────────────────────────┘
```

- The question is quoted verbatim, the way recall shows memories.
- A button decides. Typing a reply and pressing Enter decides with free text. Both send the card back to the agent that asked.
- `⚠` marks §9 flags. An untrusted author shows in a muted colour, not with a label.
- Keyboard: `j`/`k` move, `1`–`6` pick an option, `Enter` sends the reply, `o` opens the card.
- Opening a row shows the card panel (below) beside the list. Other outcomes (free for any agent, close, park) are in that panel.

### Board

```
mempalace                                                           ⌘K
┌─┬────────────────┬────────────────┬────────────────┬────────────────┐
│9│ todo 3         │ doing 1        │ review 2       │ done 41        │
│ │ #147 Canary …  │ #142 Fix the … │ #145 Pick the… │ #139 …         │
│ │ #148 Split … ● │ codex:vm 12m   │ #151 Done? …   │ #137 …         │
│ │ #149 …         │                │                │                │
└─┴────────────────┴────────────────┴────────────────┴────────────────┘
```

- `backlog` is the narrow strip on the left with only its count. Clicking it opens it as a column.
- `review` is tinted. It is the only column that waits on a person.
- A card shows its title and one line: holder and time held in `doing`, the asker in `review`, `●` when reserved for an agent in `todo`. Linked memories show as a small count, flags as `⚠`.
- `done` shows the last 20.
- Drag and drop calls the same code as `task_update`, sending the card's `version`. If an agent moved the card in the meantime, the card snaps to where it really is.
- The page polls the change counter (2 s visible, 30 s hidden), so agent moves show within seconds.

### Card panel

Opens over the right side of either screen: title, body, the ask with its buttons if in `review`, linked drawers (first line each, `invalidated` if soft-deleted), then the event log. Every log line is stamp, time, verbatim text. No other chrome.

## Trust and scopes

All from PRODUCT-DESIGN.md §6–§10, nothing new:

- A board belongs to one wing. A token sees it only with that wing in `read_wings`, writes only with `memory:write`. Private wings need `private:read`.
- Untrusted tokens and cloud clients (write wing `inbox`) add cards and comments only, and their cards land in `backlog`. They never get `task_take_next`, `task_update` or `task_handoff`.
- The §9 check flags titles, bodies, comments, handoff notes and asks. Flags show; text is never changed.
- Board tools are tagged in our capability table, so fail-closed and the auto-generated canary suite cover them.
- Agents can hand off to people who are members of the workspace only. A handoff to anyone else is refused.

## Tests (in the pre-push hook and nightly on the test host, no GitHub Actions)

- **Race:** N agents call `task_take_next` together; every card is claimed once.
- **Expiry:** a claim past `claim_until` is freed on the next board call, with one `expire` event.
- **Move rules:** every row of the API.md table, allowed and refused, over MCP and over HTTP, against the same service code.
- **Canary:** a board, card, comment and ask in a forbidden wing never appear in any tool reply or HTTP response for a scoped token.
- **Backlog guard:** a card from an untrusted token is never returned by `task_take_next` until a person triages it.
- **B's done test** (PRODUCT-DESIGN.md §15) extended: the winning agent hands the card to Avery with options, he decides on the Decide screen, and the same agent's next `task_take_next` returns it with the decision.

## Decisions

Taken in this doc, easy to overrule:

- [x] Fifth column `review`, instead of a waiting flag on `doing`.
- [x] Agent claims expire after 30 minutes without a write; people's holds never do.
- [x] One extra tool, `task_handoff` (8 in total), instead of overloading `task_update`.
- [x] Backlog pickup requires a person or a trusted, unrestricted board writer; person decisions remain protected.
- [x] Cards are never deleted; the event log is append-only.
- [x] After a decision, the card goes back to the agent that asked by default.

Open:

- [ ] **Notify Avery outside the web app?** The Decide count is the only signal in v1. A push to his phone needs a channel that keeps the "no external service" promise (a self-hosted notification service would). Decide before B ships.
- [ ] **Reserve for an agent kind** (`to=codex`, any Codex connection) as well as one connection. Wait for real use.
