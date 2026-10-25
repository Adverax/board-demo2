# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## State of the repo

Design-only so far: there is no Go code, `go.mod`, or Makefile yet. The source of truth is `docs/board-service.md` (English; `docs/board-service.ru.md` is the Russian copy — keep both in sync). The brief is "frozen for the live demo": anything not written there is out of scope, so don't add features beyond it.

Because nothing is built yet, there are no established build/lint/test commands. Once `go.mod` exists, standard Go tooling applies (`go build ./...`, `go test ./...`, `go test -run TestName ./path`, `go test -race ./...` — the concurrency test warrants `-race`).

## What is being built

A small kanban board: one Go binary serving a JSON API under `/api` plus a single embedded web page. One shared board, no auth, no users.

## Decided architecture (from the brief)

- Go 1.26, standard library `net/http` only for HTTP.
- In-memory storage guarded by a mutex; state is lost on restart (no persistence).
- Static page is plain HTML/CSS/JS with no build step, embedded via `go:embed` and served by the same process.
- Port from `PORT` env var, default 8080.
- Endpoints: `GET /api/board`, `POST /api/cards`, `PATCH /api/cards/{id}`, `POST /api/cards/{id}/move` (body `{"to": "<column>"}`), `DELETE /api/cards/{id}`.
- Errors are a JSON object with a stable `code`: `validation_failed`, `not_found`, `wip_limit_exceeded`, `invalid_move`. The page shows the reason of a refused action.

## Invariants that must hold after every operation

- Three fixed columns in order: `todo`, `doing`, `done`; the board read returns cards grouped in that order.
- Title: non-empty after trimming whitespace, at most 80 characters. Description is optional.
- New cards go to the end of `todo`; a move lands at the end of the target column.
- Moves are to an adjacent column only (`todo`↔`doing`↔`done`); `todo`↔`done` is refused (`invalid_move`).
- `doing` holds at most 3 cards; a move into a full `doing` is refused and changes nothing (`wip_limit_exceeded`).
- Positions within each column are dense (0..n-1, no gaps or duplicates), including after delete and move.
- The WIP check and the move must happen atomically under the same lock — 20 concurrent moves into `doing` must never exceed 3 cards.

## Required tests (release success criteria)

- A property test: positions stay dense after any random sequence of add/move/edit/delete.
- A concurrency test: 20 simultaneous moves into `doing`, column never exceeds 3.
- Every rule above is covered by at least one named test.

## Out of scope

Auth, multiple boards, persistence, custom columns or configurable WIP limit, drag and drop, keyboard shortcuts, animations, comments/labels/assignees/due dates, archive (the last two groups are explicitly "later, not now").
