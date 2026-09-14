# dsh-session-fixtures

Two **redacted, structurally complete** copies of real stored DeepSeek Harness session logs
(format `v0`) from a containerized web-profile deployment. Each one reproduces exactly one
migration refusal reported in
[deepseek-ai/deepseek-harness Discussion #6614](https://github.com/deepseek-ai/deepseek-harness/discussions/6614).

The logs are stored with `session.jsonl.zstd` as the file name (one directory per session), so a
plain `scan` over this repository root picks both of them up.

| directory | gate | refusal (verbatim from the official validator) |
|---|---|---|
| `v0-unknown-event-type/` | a plugin-registered event type outside the frozen v0 inventory | `format v0 contains unknown historical event type "notice/banner" at seq 26; migration refuses unknown historical events even when ignorable` |
| `v0-unknown-payload-member/` | a payload member the v0 disposition does not declare | `permission/preset 0 data has unexpected member "origin"` |

## How these were produced (redaction discipline)

Only **string values that carry free text** were replaced with `<REDACTED>`. Everything the
validator keys on was preserved byte-for-byte in shape:

* the session header (with `id` replaced by a synthetic fixture id, `cwd` normalized to `/workspace`,
  `parentSession`/`title` dropped),
* every event's `type`, `seq` and `time`,
* the exact **key set** and **value types** of every `data` object (including the offending
  `notice/banner` payload and the extra `origin` member),
* enum/literal values the validator compares against a fixed list (`form`, `action`, `reason`,
  `kind`, `role`, `mode`, `preset`, `policy`, `state`, `status`, …).

Message text, tool arguments/results, titles, prompts, paths, host names, provider identifiers and
credentials were all replaced. A three-layer sensitive scan (plaintext grep on the compressed
bytes, grep on the decompressed stream, and literal comparison against the deployment's credential
files) returned zero hits.

> If you re-redact these yourself: changing anything *other* than free text can make the log fail
> for a **different** reason (the validator checks key sets and enum literals). Both fixtures were
> verified to still fail for the *same* message after redaction, and to produce **no other**
> finding.

## Reproduce

```sh
# 1. read-only scan with the community tool that classifies these gates
npx -y dsh-session-check@0.1.3 scan .

# expected:
#   sessions scanned   : 2
#   loader would refuse: 2
#   gates hit: 1  v0-unknown-event-type  (official validator)
#              1  v0-unknown-payload-member  (official validator)
```

The two `v0-*` gates are produced by the **official** validator
(`@deepseek-ai/dsh-session-format-v0-to-v1`, `assertReleasedEventPayload`), so to see them the
scanner needs that package to be resolvable. On a host whose installed harness predates it, the
scanner prints `official validator : UNAVAILABLE` and those two gates are silently absent — install
the package (or point the scanner at a tree that contains it) before trusting a "0 refusals" result.

### Environments without a `zstd` CLI

`dsh-session-check` (like the harness's own migration path) shells out to `zstd -dc <file>`. A
stored log is **multi-frame**: first frame = header line, later frames = appended events. If your
`zstd` replacement tries a single-frame decompression first and only falls back on error, it will
"succeed" on the first frame alone and the log will look like a bare header — i.e. a **false clean
result**. Any stand-in must split on the zstd magic number (`28 B5 2F FD`) and decompress **every**
frame. Node's built-in `zlib.zstdDecompressSync` can do this:

```js
const M = Buffer.from([0x28, 0xb5, 0x2f, 0xfd])
const buf = fs.readFileSync(file)
const idx = []
let i = 0
while ((i = buf.indexOf(M, i)) !== -1) { idx.push(i); i += 4 }
idx.push(buf.length)
const out = []
for (let k = 0; k < idx.length - 1; k++) out.push(zlib.zstdDecompressSync(buf.subarray(idx[k], idx[k + 1])))
process.stdout.write(Buffer.concat(out))
```

## License / provenance

Log content is ours; it is published here only to let upstream maintainers reproduce the two
refusals. All message content has been removed.
