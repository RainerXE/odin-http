# Local patches on top of upstream odin-http

This is a fork of <https://github.com/laytan/odin-http> that exists so OdinLLM can
carry changes upstream does not have yet. It is vendored at `OdinLLM/vendor/odin-http`
and fetched by CI from `RainerXE/odin-http`.

Base: upstream `main` at `26da442` ("openssl: update bundled libraries to
openssl-3.6.5", 2026-11-09). Everything below that is upstream history, unmodified.

## `respond_json`: `json.Marshal_Options` moved to pass-by-value

Odin changed `core:encoding/json` in `dev-2026-11`: `marshal_to_writer` and
`marshal_to_builder` take `Marshal_Options` **by value** now, where `dev-2026-10` and
earlier took `^Marshal_Options`. Upstream still calls

```odin
json.marshal_to_writer(rw.w, v, &opt)
```

so the package stops compiling on `dev-2026-11`:

```text
responses.odin(151:43) Error: Cannot assign value '&opt' of type '^Marshal_Options'
                                        to 'Marshal_Options' in a procedure argument
```

Because `src/api_openai` imports `vendor/odin-http`, that one call site takes OdinLLM's
whole gate down with it.

The patch calls `json.marshal` instead, which has taken the options by value in every
compiler version this project supports, and writes the resulting body. Two
consequences worth knowing:

- the body is materialised once per JSON response instead of streamed through the
  response writer's 128-byte buffer;
- the marshal error is now known **before** the first byte is written, so a failure
  sets a clean 500 instead of appending to a partial body.

### Why not the one-character upstream fix

`json.marshal_to_writer(rw.w, v, opt)` is what upstream needs, and
[upstream PR #124](https://github.com/laytan/odin-http/pull/124) (opened 2026-11-07)
proposes exactly that. It is still open, and it only compiles on `dev-2026-11` and
newer.

The patch below is also offered upstream as
[PR #126](https://github.com/laytan/odin-http/pull/126) (branch
`fix/respond-json-portable`, `responses.odin` only — no `PATCHES.md`). If upstream
takes either PR, the reason to keep this fork shrinks to "not released yet"; if it
takes #124, the portability argument disappears with it.

OdinLLM's fleet is mid-migration: the macOS runner moved to `dev-2026-11a`, but the
Linux runner image was last built with `dev-2026-10` (the toolchain pins moved to
`dev-2026-11` on 2026-11-08 and the image is rebuilt by hand). Going through
`json.marshal` keeps both ends building, instead of making a dependency bump depend on
an unrelated image rebuild.

**Drop this patch** — go back to plain upstream — once upstream #124 is merged *and*
every runner image builds with `dev-2026-11`. Check `responses.odin` on `origin/main`
before the next rebase; if it says `opt` rather than `&opt`, the patch is obsolete.

## `Response_Writer`: flush did not reach the socket

`response_writer_init` installs an `io.Stream` whose `.Flush` case moved the pending bytes
into the response's own buffer (`r._buf`). Only `.Close` wrote that buffer to the
connection, through a single `nbio.send` that also ends the request. So `io.flush` was a
no-op on the wire: a handler streaming an event per token delivered nothing until it
returned.

Measured with a two-event SSE handler (write, flush, sleep 500 ms, write, flush, close):

| | first byte | framing |
|---|---:|---|
| upstream `26da442` | 502.7 ms | both events plus the terminator in one write |
| this patch | 0.2 ms | first event in its own chunk, second at 500.9 ms |

That is the difference between a streaming endpoint and a buffered one, and it is what
made OdinLLM's SSE surface unusable behind a proxy.

The patch keeps one send in flight per response: bytes written while a send runs stay in
`_buf`, and that send's callback issues the next one, so chunk order is preserved and the
buffer a send reads from is never reallocated underneath it. Streamed bytes are copied
into the connection's request arena, which outlives the send and is freed when the
response completes. `response_status` stops rewriting the status line after the first
streamed byte, because the heading is already on the wire.

Offered upstream as [PR #127](https://github.com/laytan/odin-http/pull/127) (branch
`feat/writer-flush-to-socket`). **Drop this patch** when upstream takes it: check that
`response.odin` on `origin/main` sends from the `.Flush` case before the next rebase.

## Keeping this fork current

```sh
git fetch upstream
git rebase upstream/main        # keeps the patches as the only local commits
git push origin main
```

Then bump `vendor/odin-http` in OdinLLM to the new commit. The fork carries no local
branches beyond the two patch branches that back the upstream PRs, so a rebase should
apply cleanly or be a small conflict at the files named above.
