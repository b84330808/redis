# Report: two bug fixes for redis/redis

- Base: `redis/redis` `unstable` at `8e2ba8c` ("Bound redis-cli legacy help arity expansion (#16117)").
- Fork: `b84330808/redis`. Nothing was opened, commented or pushed upstream.
- Both bugs were found by reading the code and testing on `unstable`. Neither has an upstream
  issue. I searched upstream PRs and issues in all states and found no competing PR (details in
  each section).
- Author on both commits: `b84330808 <b84330808@gmail.com>`. Each message ends with
  `Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>`.

| | Fix A (higher impact) | Fix B |
|---|---|---|
| Bug | `ACL SETUSER` accepts a first-arg rule with a space or quote; `ACL LIST`/`GETUSER`/`SAVE`, `CONFIG REWRITE` or `MODULE UNLOAD` then crash the server | `redis-cli --quoted-json` prints invalid JSON for any string containing `"` |
| Branch | `fix-acl-firstarg-quotes-crash` | `fix-redis-cli-quoted-json-quotes` |
| SHA | `1a4e5d35b125a54d9f53ad4affe4cd91924cf27e` | `50222f3803f8c2b929f29a9027ba4c4e48ec2eb9` |
| Files | `src/acl.c` (+11), `tests/unit/acl.tcl` (+17) | `src/redis-cli.c` (+5 −9), `tests/integration/redis-cli.tcl` (+18) |

---

## Fix A: ACL first-arg with spaces or quotes crashes the server

### How it was found

I was checking how ACL rule text is serialized and parsed back. `ACLSetSelector()` stores a
`+cmd|first-arg` rule verbatim in `selector->command_rules`, a space-separated string.
`ACLDescribeSelectorCommandRules()` (acl.c:814) and
`ACLRecomputeCommandBitsFromCommandRulesAllUsers()` (acl.c:716) later split that string again
with the quote-aware `sdssplitargs()`, and assert on the result. Nothing validates the
first-arg, so:

| Rule passed to `ACL SETUSER` | Result on unstable |
|---|---|
| `+select\|a"b`, `+select\|it's` | `OK`, then `ACL LIST` crashes: `acl.c:815 'argv != NULL' is not true` |
| `'+select\|a b'` (one argument), `(+select\|"a b")` | `OK`, then `ACL LIST` crashes: `acl.c:819 'res == C_OK' is not true` |
| `+select\|"0"` | `OK`; after any `MODULE UNLOAD` the rule silently becomes `+select\|0` and the user can now run `SELECT 0` |

Every path that describes a user hits this: `ACL LIST`, `ACL GETUSER`, `ACL SAVE`,
`CONFIG REWRITE`, `RM_GetModuleUserACLString`, and the recompute on `MODULE LOAD`/`UNLOAD`. An
aclfile line holding such a rule loads fine, because the aclfile loader splits on plain spaces,
and the server crashes later.

### Reproduction on unstable (8e2ba8c)

```bash
make -j
```

```bash
./src/redis-server --port 7777 --save "" --daemonize yes --logfile /tmp/r.log
```

```bash
./src/redis-cli -p 7777 acl setuser u on nopass '+select|a"b'
```

```bash
./src/redis-cli -p 7777 acl list
```

Output:

```
OK
Error: Server closed the connection
```

Server log:

```
# === ASSERTION FAILED ===
# ==> acl.c:815 'argv != NULL' is not true
...
redis-server *:7777(ACLDescribeSelectorCommandRules+0x278)
redis-server *:7777(ACLDescribeSelector+0x264)
redis-server *:7777(aclCommand+0x851)
```

Selector form, same crash path: `acl setuser fu on nopass '(+select|"a b")'` returns `OK`, then
`acl getuser fu` gives `Error: Server closed the connection`, and the log shows
`==> acl.c:819 'res == C_OK' is not true`.

Silent permission change, on a server started with `--enable-module-command yes` (requires
`make -C tests/modules`):

```
> ACL SETUSER u on nopass -@all '+select|"0"'
OK
$ redis-cli --user u --pass x SELECT 0
NOPERM User u has no permissions to run the 'select' command
> MODULE LOAD /path/to/tests/modules/aclcheck.so
OK
> MODULE UNLOAD aclcheck
OK
> ACL GETUSER u      ->  commands: "-@all +select|0"
$ redis-cli --user u --pass x SELECT 0
OK
```

### Fix

In the first-arg branch of `ACLSetSelector()`, reject a first-arg that contains whitespace
(`ACLStringHasSpaces`) or a `"`/`'` with `EINVAL`. This gives the usual `Syntax error`, the same
treatment that key and channel patterns with spaces already get. Valid first-arg rules such as
`+select|0` and `+debug|object` are unaffected. Subcommand rules (`+config|get`) are not
touched.

### Test

New test `ACLs reject a first-arg with spaces or quotes` in `tests/unit/acl.tcl`. It sends
`+select|a b`, `+select|a<TAB>b`, `+select|a"b`, `+select|"0"`, `+select|it's`,
`(+select|"a b")` and `(+select|"a\"b")`, and expects `Syntax error` for each. It then checks
that `ACL GETUSER`/`ACL LIST` still work and show `-@all +select|0`.

Fail-before (fix reverted with `git stash push src/acl.c`, test kept, rebuilt):

```bash
./runtest --single unit/acl --only "/first-arg with spaces"
```

```
[err]: ACLs reject a first-arg with spaces or quotes in tests/unit/acl.tcl
Expected 'OK' to match '*Syntax error*' (context: type eval line 11 cmd {assert_match {*Syntax error*} $e} proc ::test)
!!! WARNING The following tests failed:
```

Pass-after (fix applied, rebuilt):

```bash
./runtest --single unit/acl --only "/first-arg with spaces"
```

```
[ok]: ACLs reject a first-arg with spaces or quotes (2 ms)
\o/ All tests passed without errors!
```

Regression run (needs `make -C tests/modules`):

```bash
./runtest --single unit/acl --single unit/acl-v2 --single unit/moduleapi/usercall --single unit/moduleapi/aclcheck
```

```
\o/ All tests passed without errors!   (199 [ok])
```

### Competing PR search (redis/redis, all states, `is:pr` and `is:issue`)

Queries: `ACL first-arg crash`, `ACL firstarg quote`, `"acl.c:815"`,
`ACLDescribeSelectorCommandRules`, `command_rules sdssplitargs`, `ACL SETUSER crash`,
`ACL LIST crash`, `ACL first arg space`, `allowed_firstargs`, `ACL GETUSER assertion`,
`ACL quote`, `"first-arg" ACL`, `ACLSetSelector`, `"select|" ACL crash`,
`ACL rule quotes crash`, `ACL SETUSER quote`.

None of them fixes this. The closest results:

- #16040 (open) "Fix RM_SetModuleUserACLString silently accepting malformed ACL strings". It
  handles a NULL `sdssplitargs()` result in the module API. That is a different call site, and it
  does not stop invalid first-args from being stored.
- #11650 (open, 2022) "Handle closing paranthesis in acl selectors". Unrelated.
- #7329 (closed, 2020) "Creating usernames with quotes can break ACL LOAD". Old and different
  (usernames).
- #16007 (open issue) is an unrelated RDB-save crash.

### PR title

Reject ACL first-args with spaces or quotes, which crash ACL LIST

### PR body draft

```
A rule like +select|<first-arg> is stored as is in the selector's command_rules, a space
separated string that ACLDescribeSelectorCommandRules() and
ACLRecomputeCommandBitsFromCommandRulesAllUsers() split again with sdssplitargs().
ACLSetSelector() accepts any first-arg, so:

- With a quote (+select|a"b, +select|it's) sdssplitargs() fails and ACL LIST, ACL GETUSER,
  ACL SAVE, CONFIG REWRITE or MODULE UNLOAD crash the server on 'argv != NULL'.
- With whitespace (the single argument '+select|a b', or (+select|"a b") in a selector) the
  rule is split in two and the same commands crash on 'res == C_OK'.
- With balanced quotes (+select|"0") the split drops the quotes, so after a MODULE UNLOAD the
  user is allowed SELECT 0, which the rule as written did not allow.

    > ACL SETUSER u on nopass +select|a"b
    OK
    > ACL LIST
    (server crashes: acl.c:815 'argv != NULL' is not true)

An ACL file with such a rule loads fine, since the ACL file is split on plain spaces, and the
server crashes later.

### Fix
Reject first-args that contain whitespace or a quote with the usual "Syntax error", as is
already done for key and channel patterns with spaces. First-args are a deprecated mechanism
meant for rules like +select|0 or +debug|object, so valid rules are not affected. An ACL file
holding such a rule now fails to load with an error instead of crashing the server later.

### Tests
New test "ACLs reject a first-arg with spaces or quotes" in tests/unit/acl.tcl. It fails on
unstable (the rules are accepted, and ACL LIST would then crash) and passes with the change.
unit/acl, unit/acl-v2, unit/moduleapi/usercall and unit/moduleapi/aclcheck pass.

I used Claude Code to write this change.
```

---

## Fix B: `redis-cli --quoted-json` prints invalid JSON for strings containing `"`

### How it was found

I was probing redis-cli output modes with RESP3 replies. In `jsonStringOutput()`
(redis-cli.c:2208), the `OUTPUT_QUOTED_JSON` branch takes the `sdscatrepr()` form of the string,
including its outer quotes, and doubles every backslash. `sdscatrepr()` writes an inner double
quote as `\"`, which becomes `\\"` and ends the JSON string early. This dates from #10286
(Redis 7.0), when `--quoted-json` was added. The same function also handles map keys.

### Reproduction on unstable (8e2ba8c)

```bash
./src/redis-cli -p 7777 set q 'a"b'
```

```bash
./src/redis-cli -p 7777 --quoted-json get q
```

Output:

```
"a\\"b"
```

Running that through a JSON parser:

```bash
./src/redis-cli -p 7777 --quoted-json get q | python3 -c 'import json,sys; json.loads(sys.stdin.read())'
```

```
json.decoder.JSONDecodeError: Extra data: line 1 column 6 (char 5)
```

`--quoted-json hgetall` on a hash with `"` in a value fails the same way, with `Expecting ','
delimiter`.

### Fix

Strip the outer quotes from the `sdscatrepr()` result and JSON-escape the rest with the existing
`escapeJsonString()`. The output for `a"b` becomes `"a\\\"b"`, whose JSON value is `a\"b`. That
is the same quoted form `--quoted-json` already uses elsewhere, e.g. `"\\xf0e"` in the existing
test. `sdscatrepr()` only emits printable ASCII, so output for strings without a `"` is
byte-identical to before. I checked that values containing `"`, `\"`, NUL/control bytes,
non-ASCII bytes, `/` and the empty string now parse with Python's `json`, and that un-repr'ing
the decoded value gives back the original bytes.

### Test

New test `Double quote in quoted JSON mode` in `tests/integration/redis-cli.tcl`. It covers
`a"b`, `\"`, and a hash key and value containing `"`, in both `--json` (unchanged) and
`--quoted-json`.

Fail-before (test added, unfixed redis-cli):

```bash
./runtest --single integration/redis-cli --only "/JSON mode"
```

```
[ok]: Non-interactive TTY CLI: Escape character in JSON mode (22 ms)
[err]: Non-interactive TTY CLI: Double quote in quoted JSON mode in tests/integration/redis-cli.tcl
Expected '"a\\\"b"' to be equal to '"a\\"b"' (context: type eval line 7 cmd {assert_equal {"a\\\"b"} [run_cli --quoted-json hget quote test]} proc ::test)
!!! WARNING The following tests failed:
```

Pass-after:

```bash
./runtest --single integration/redis-cli --only "/JSON mode"
```

```
[ok]: Non-interactive TTY CLI: Escape character in JSON mode (23 ms)
[ok]: Non-interactive TTY CLI: Double quote in quoted JSON mode (16 ms)
\o/ All tests passed without errors!
```

Regression run:

```bash
./runtest --single integration/redis-cli
```

```
\o/ All tests passed without errors!   (92 [ok])
```

### Competing PR search (redis/redis, all states, `is:pr` and `is:issue`)

Queries: `"quoted-json"`, `quoted-json`, `"--quoted-json" quote`, `quoted-json invalid`,
`jsonStringOutput`, `redis-cli invalid JSON`, `redis-cli json double quote`,
`redis-cli json escape`, `sdscatrepr json`, `"Unknown reply type"`.

None of them fixes this. The only open PR touching nearby code is #15764 "redis-cli: handle RESP3
big number replies in all output modes". Its diff (checked on the PR's Files tab) only adds
`REDIS_REPLY_BIGNUM` cases to the four reply formatters and does not change
`jsonStringOutput()`, so the two changes don't overlap.

### PR title

Fix redis-cli --quoted-json output for strings with double quotes

### PR body draft

```
With --quoted-json, jsonStringOutput() takes the sdscatrepr() form of a string, including its
surrounding double quotes, and doubles every backslash in it. sdscatrepr() writes a double
quote inside the string as \", which then becomes \\" and ends the JSON string early, so a
reply containing a double quote produces invalid JSON:

    $ redis-cli set k 'a"b'
    $ redis-cli --quoted-json get k
    "a\\"b"

### Fix
Strip the outer quotes from the sdscatrepr() result and JSON-escape the rest with
escapeJsonString(), which escapes the double quotes as well as the backslashes. The output is
now "a\\\"b", whose JSON value is a\"b: the same quoted form --quoted-json already uses for
other strings (e.g. \xf0e). Since sdscatrepr() only outputs printable ASCII, strings without a
double quote are printed exactly as before. Map keys go through the same function and are
fixed too.

### Tests
New test "Double quote in quoted JSON mode" in tests/integration/redis-cli.tcl. It fails on
unstable and passes with the change. integration/redis-cli passes.

I used Claude Code to write this change.
```

---

## Upstream policy notes

- **CONTRIBUTING.md**: it opens with the full "Redis Software Grant and Contributor License
  Agreement". It says that by contributing in any form you agree to it; there is no separate
  signing step. Clause 5 has you represent that the contribution is "Your original work" and
  that you are legally entitled to grant the license. Read it before submitting, since this
  change was AI-assisted. Minor fixes: "open a pull request on GitHub". Major or semantic
  changes should be discussed in an issue first, which doesn't apply here. GitHub issues are for
  bugs and detailed feature requests only.
- **CLA bot**: none seen. Merged PR #16120's page shows no CLA bot comment or check (bots
  present: chatgpt-codex-connector, codecov, Cursor Bugbot). Because no bot would reject an
  unknown co-author, I kept the `Co-Authored-By: Claude Opus 5.5` trailer.
- **AI policy / AGENTS.md / CLAUDE.md**: none in the repo. AI-assisted PRs are evidently
  accepted. The last 400 upstream commits include 13 with `Co-authored-by: Claude Opus 5.5`
  (e.g. #15998, which also carries "Generated with Claude Code") and one with
  `Co-authored-by: Claude Sonnet 5.5` (#15938). Open PRs such as #15764 disclose AI use in the
  body. Nothing requires a human to write the PR text.
- **PR template**: none (`.github/` has only `ISSUE_TEMPLATE/`). The bodies above follow the
  Problem / Fix / Tests layout of recent merged PRs.
- **Security**: SECURITY.md asks for vulnerabilities to go through the Redis Vulnerability
  Disclosure Program. Fix A crashes the server, but reaching it requires `ACL SETUSER`
  (`@admin @dangerous`) or write access to the aclfile or config. Those principals can already
  run `SHUTDOWN`/`DEBUG`, so I treated it as a robustness bug fit for a public PR. The
  `+select|"0"` → `+select|0` widening after `MODULE UNLOAD` is also admin-induced. If you
  consider that security-relevant, report Fix A through the disclosure program instead of a
  public PR.
- **Commit trailers**: the session's default attribution would also add a `Claude-Session:` URL
  trailer. I left it out because the task said the message must end with the `Co-Authored-By`
  line, and a session URL would be noise upstream.

## Other findings, not fixed (possible follow-ups)

1. **CONFIG REWRITE can make the server unstartable.** In a config without `aclfile`,
   `ACL SETUSER u2 on nopass '~a"b' +get` is accepted. `CONFIG REWRITE` then writes
   `user u2 on nopass sanitize-payload ~a"b resetchannels -@all +get` unquoted. The next start
   fails with `FATAL CONFIG FILE ERROR ... Unbalanced quotes in configuration line`, because
   redis.conf is tokenized with `sdssplitargs()` while aclfile lines are split on plain spaces.
   A fix would quote, in `rewriteConfigUserOption()`, the tokens that contain `"` or `'`
   (including usernames). No competing PR found (`"CONFIG REWRITE" user quote`,
   `rewriteConfigUserOption`, `"Unbalanced quotes" config`).
2. **Unbalanced quotes in a selector silently create an empty selector.**
   `aclCreateSelectorFromOpSet()` ignores a NULL `sdssplitargs()` result, so `(~s"el +get)`
   returns `OK` and creates a selector with no permissions. This is the selector analogue of
   #16040. Rejecting it would make some `ACL SAVE`d files fail to load, so it needs a
   maintainer's call.
3. `tests/integration/redis-cli.tcl` "Escape character in JSON mode" lines 518 and 520 call
   `assert_equal $x $x [run_cli ...]`, which compares a value with itself and so never checks
   the reverse-solidus output.
4. Candidates ruled out because competing work exists: BLESS GET/SCAN missing `READONLY`
   (#15828, closed); redis-cli exiting with `Unknown reply type: 13` on RESP3 big numbers
   (#15764, open); `SET ... EX 60 EX 3` accepted (#16122, a commenter says a fix is ready).
   Most other open `[BUG]` issues I checked already have PRs.
5. Fuzzing with no findings: random commands over the new types (set member expiry, AR*
   arrays, LMOVEM, INCREX, MSETEX, HSETEX/HGETEX, streams with IDMP, BLESS) on an ASAN build
   with `DEBUG KEYSIZES-HIST-ASSERT 1` and `DEBUG ALLOCSIZE-SLOTS-ASSERT 1`, plus a
   WATCH-invalidation oracle and a master/replica `DEBUG DIGEST` oracle. About 100k commands
   produced only timing false positives.
