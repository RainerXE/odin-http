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

## Keeping this fork current

```sh
git fetch upstream
git rebase upstream/main        # keeps the patch as the only local commit
git push origin main
```

Then bump `vendor/odin-http` in OdinLLM to the new commit. The fork has no other local
branches and no other local changes, so a rebase should always apply cleanly or be a
one-hunk conflict.
