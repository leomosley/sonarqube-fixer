# Common SonarQube Rule Fixes

**Code Smells:**
- `S1135` (TODO comment) — Remove or resolve the TODO
- `S3776` (cognitive complexity) — Extract inner logic into named helper functions
- `S107` (too many parameters) — Introduce an options/config object
- `S1066` (collapsible if) — Merge nested `if` into a single condition with `&&`
- `S1854` (useless assignment) — Remove the dead assignment or use the value
- `S2814` (variable re-declaration) — Replace `var` with `const`/`let`
- `S1192` (duplicated string literal) — Extract to a named constant

**Bugs:**
- `S2201` (return value ignored) — Use or explicitly discard the return value
- `S905` (non-boolean in boolean context) — Add an explicit boolean comparison
- `S3516` (function always returns same value) — Fix the logic or simplify the function

**Security / Vulnerabilities:**
- `S2068` (hardcoded credential) — Move value to an environment variable
- `S5042` (zip slip) — Validate that extracted paths stay within the target directory
- `S6096` (path traversal) — Sanitize and validate user-supplied file paths
- `S4790` (weak hash) — Replace MD5/SHA-1 with SHA-256 or stronger

**Shell scripts:**
- `S2086` / `SC2086` — Quote variables: `"$var"` not `$var`
- `S2155` / `SC2155` — Separate `declare`/`local` from assignment to capture exit codes
- `SC2034` — Remove unused variables
- `SC2046` — Quote command substitutions: `"$(cmd)"` not `$(cmd)`
- Use `[[ ]]` instead of `[ ]` for bash conditionals

**Java / Kotlin:**
- `S2095` (resource leak) — Use try-with-resources
- `S1612` (lambda can be method ref) — Replace `x -> foo(x)` with `Foo::foo`

**Python:**
- `S1481` (unused variable) — Remove or prefix with `_`
- `S5754` (bare except) — Catch specific exception types
