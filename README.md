# Sharpened

**Battle-tested skills library for AI coding agents.** A fork of [obra/superpowers](https://github.com/obra/superpowers) (MIT, © Jesse Vincent) with AI-specific hardening learned from real production sessions.

## Why fork

Superpowers gives coding agents a solid methodology: brainstorming → plan → subagent-driven execution → review → done, with TDD and YAGNI as principles. It works.

**But it was written for humans doing TDD.** In real production use with AI coding agents, we kept hitting failure modes the stock skills don't defend against:

1. **Tautological assertions** — `expect(output).not.toContain(orchestratorId)` passes trivially when the template never contains a UUID to begin with. Stock TDD's "watch it fail once" doesn't catch this because the test was always going to be green.
2. **Spy at wrong layer** — test asserts the idempotent guard in module A fires, but the actual guard is in module B. Removing guard A leaves the test green.
3. **Reporting drift** — agent claims "109 tests pass" when vitest actually reports 105. Claims "fixed" a field that never made it to schema. Deletes a test to stay green without saying so.
4. **Scope creep** — task says "extract a helper"; agent also fixes three unrelated bugs and refactors a sibling module. CI stays green, next PR becomes unreviewable.
5. **Anti-patterns as a fixed list** — stock skills ship with 5 anti-patterns frozen at author's last commit. Real projects need the list to grow every time a new failure mode appears.

Sharpened is the skill set that emerged after ~15 rounds of forcing fixes to these modes. The diffs vs. upstream aren't large; the philosophy is.

## Core additions / changes vs. superpowers

| Skill | Change |
|---|---|
| `test-driven-development` | **Reverse-regression is mandatory:** every new assertion must red-on-comment-out of the corresponding product-code line, green on restore. Integration tests forbid mock-pty-style chains. Anti-patterns list grows — project-local `anti-patterns.md` is versioned with the repo. |
| `verification-before-completion` | **Numbers are verbatim to tool output.** Deleted tests must be listed with reason. Completion is per-item verdict (done / partial / skipped + evidence), not a blanket "basically done". |
| `scope-guardrails` (new) | **PR-X only does PR-X.** Accidental fixes to unrelated issues go into a follow-up PR or get reverted. `git status` must match the task manifest. |

Everything else from superpowers is preserved.

## Install

### Claude Code
```bash
# From GitHub (once plugin marketplace supports custom sources)
/plugin marketplace add tt-a1i/sharpened
/plugin install sharpened

# Or local dev
git clone https://github.com/tt-a1i/sharpened.git ~/.claude/plugins/cache/sharpened/1.0.0
```

### Gemini CLI
```bash
gemini extensions install https://github.com/tt-a1i/sharpened
```

### OpenCode / Codex
See `.opencode/INSTALL.md` for bootstrap injection (inherits superpowers' multi-CLI support).

## Relationship to upstream

- **License:** MIT. Upstream copyright © Jesse Vincent; derivative work © Shaokun Tu.
- **Attribution:** every skill that is a modified version of an upstream skill retains a `> Adapted from obra/superpowers` note at top.
- **Sync policy:** we cherry-pick from upstream when skill content is updated; we do not auto-merge, because upstream's open PR backlog on TDD improvements suggests divergent quality bars.
- **Issues / PRs back upstream:** if a sharpened change is pure bug-fix (not an opinion), we file it upstream too.

## Philosophy

Human engineering conventions assume code is read many times, developers have cross-session memory, and code review catches drift. None of that is true for AI coding agents across sessions.

**Sharpened's thesis:** the rules that matter are the ones machines can verify (`grep`, `wc -l`, reverse-regression). Soft guidance ("mocks only if unavoidable") becomes hard rules ("`grep mock-node-pty tests/server/` returns zero"). The anti-patterns list grows every time an agent gets caught, and grows only. Nothing gets deleted because a future agent will always try the same trick again.

## Sponsorship

Upstream author accepts sponsorship: [obra on GitHub Sponsors](https://github.com/sponsors/obra). Sharpened has none.

## License

MIT. See [LICENSE](./LICENSE).
