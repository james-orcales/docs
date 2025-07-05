# Bash Scripting Best Practices

This document consolidates best practices from four key sources, with a focus solely on **writing reliable and safe Bash scripts**. It does not prescribe a full style guide—you're encouraged to adopt your own formatting preferences. However, any rules or recommendations stated here **override those in the original sources** where conflicts arise. 

*Sections marked with 💡 reflect original insight or deliberate divergence from the referenced guidelines.*

## Resources

- [Google Style Guide](https://google.github.io/styleguide/shellguide.html)  
- [shellharden - Safe Bash Guide](https://github.com/anordal/shellharden/blob/master/how_to_do_things_safely_in_bash.md)  
- [Security Hardening for Github Actions](https://docs.github.com/en/actions/security-for-github-actions/security-guides/security-hardening-for-github-actions?learn=getting_started&learnProduct=actions)  
- [TigerBeetle’s Tigerstyle](https://github.com/tigerbeetle/tigerbeetle/blob/main/docs/TIGER_STYLE.md)

## Table of Contents

- [Bash Scripting Best Practices](#bash-scripting-best-practices)
   * [Resources](#resources)
   * [💡 Template](#-template)
- [Security Guidelines](#security-guidelines)
   * [💡 Bash Setup](#-bash-setup)
   * [SUID/SGID](#suidsgid)
   * [Error Messages](#error-messages)
   * [Comments](#comments)
      + [💡 Function Doc Comments](#-function-doc-comments)
   * [Formatting](#formatting)
      + [💡 Indentation and Line Length](#-indentation-and-line-length)
      + [💡 Variable Expansion and Command Substitution](#-variable-expansion-and-command-substitution)
   * [Features and Bugs](#features-and-bugs)
      + [ShellCheck](#shellcheck)
      + [💡 Test, `[ … ]`, and `[[ … ]]`](#-test-and-)
      + [Testing Strings](#testing-strings)
      + [Wildcard Expansion of Filenames](#wildcard-expansion-of-filenames)
      + [💡 Eval](#-eval)
      + [Arrays](#arrays)
      + [Pipes to While](#pipes-to-while)
      + [Arithmetic](#arithmetic)
      + [Aliases](#aliases)
   * [Naming Conventions](#naming-conventions)
      + [💡 Variables Names](#-variables-names)
      + [💡 Declare, Readonly, and Local Variables](#-declare-readonly-and-local-variables)
   * [Calling Commands](#calling-commands)
      + [Builtin Commands vs. External Commands {#builtin-commands-vs.-external-commands}](#builtin-commands-vs-external-commands-builtin-commands-vs-external-commands)
      + [How to end a bash script](#how-to-end-a-bash-script)
- [Github Actions](#github-actions)
   * [Mitigating Script Injection in GitHub Actions](#mitigating-script-injection-in-github-actions)
      + [Use an action instead of inline scripts](#use-an-action-instead-of-inline-scripts)
      + [💡 Use an intermediate environment variable  ](#-use-an-intermediate-environment-variable)
      + [💡 Global Helper Functions Hack](#-global-helper-functions-hack)
      + [💡 Inline Insertions and GitHub Expression Defaults](#-inline-insertions-and-github-expression-defaults)

## 💡 Template

```bash  
.bashrc

#!/usr/bin/env bash

# WARNING: do not eval this in .bashrc itself
export __bash_setup__=$(
  cat <<EOF
set +e
set -uo pipefail
shopt -s nullglob globstar extglob
function println() {
  if [ "$#" -gt 0 ]; then
    printf "$@"
  fi
  printf '\n'
}
function warn() {  
  println "⚠️ WARN: $@" >&2  
  return 0  
}
function err() {
  println “ERROR: ❌ $@" >&2
  return 1
}
function ignore_failure() {
  "$@" || warn "⚠️ ignored failure: $*\n"
  return 0
}
EOF
)

# separate_script.bash
eval “$__bash_setup__”
```

# Security Guidelines

## 💡 Bash Setup

Use the latest version of bash available.  

- On Windows, enable [WSL2](https://learn.microsoft.com/en-us/windows/wsl/) and use Bash from a proper Linux distro.  
- On macOS, be cautious: many GNU utilities have subtle differences on Darwin vs Linux (e.g., `sed -E`). See [Builtin Commands vs External Commands.](#builtin-commands-vs.-external-commands)

Use this shebang to invoke Bash reliably: `#!/usr/bin/env bash`

💡 Set your shell environment explicitly at the top of every script:

```bash  
set +eu  
set -o pipefail  
shopt -s nullglob globstar extglob  
```

* **💡 Disable `errexit` (+e)**:  
   `set -e` is too unreliable — it silently fails in common cases like:  
* Inside `if` or `while`  
* In pipelines (`cmd1 | cmd2`)  
* Inside command substitution (`$(...)`)

Disabling it encourages **explicit and intentional error handling**.

* **💡 Disable `nounset` (+u)**:  
   Treat unset and empty variables the same. Makes scripts less brittle and easier to write defaults for. Refer to [Function Doc Comments](#💡-function-doc-comments)  
* **Enable `pipefail`**:  
   Ensures pipeline failures aren't silently ignored. The pipeline fails if any command fails — not just the last.  
    
  ```bash  
  if ! aws ecs do_something --json | jq '{ .[:-5] | .some_typo }' | tee foobar.log; then  
    # This fails if any command in the pipeline fails, thanks to pipefail.  
    # But ideally, each command’s failure should be checked and handled explicitly.  
    err "Failed to query ECS information"  
    exit 1  
  fi  
    
  # Instead of relying on `pipefail` to catch any error in the pipeline,  
  # handle each command’s failure explicitly. This provides better control and clearer error messages.  
  json=”$(aws ecs do_something --json) || {  
    err "AWS call failed"  
    exit 1  
  })”  
    
  filtered=”$(jq '{ .[:-5] | .some_typo }' <<< "$json") || {  
    err "jq failed"  
    exit 1  
  })”  
    
  println "$filtered" | tee foobar.log  
  ```  
* **Enable `nullglob`, `globstar`, `extglob`**

  * `nullglob`: globs expand to nothing instead of themselves when no matches are found.

  * `globstar`: enables `**` to match files recursively.

  * `extglob`: enables extended pattern matching (`+()`, `@()`, etc.).

## SUID/SGID

SUID and SGID are *forbidden* on shell scripts.

There are too many security issues with shell that make it nearly impossible to secure sufficiently to allow SUID/SGID. While bash does make it difficult to run SUID, it’s still possible on some platforms which is why we’re being explicit about banning it.

Use `sudo` to provide elevated access if you need it.

[https://google.github.io/styleguide/shellguide.html#suidsgid](https://google.github.io/styleguide/shellguide.html#suidsgid)

## Error Messages

All error messages should go to `STDERR`. This makes it easier to separate normal status from actual issues.

```bash

`function err() {`

  `println "ERROR: ❌ $*" >&2`

  `return 1`

`}`

`if ! do_something; then`

  `err "i’m warning you"`

  `exit 1`

`fi`

```

[https://google.github.io/styleguide/shellguide.html#stdout-vs-stderr](https://google.github.io/styleguide/shellguide.html#stdout-vs-stderr)

## Comments

### 💡 Function Doc Comments

**💡 Code is truth** — not comments. Unlike doc comments, which can drift or lie, **assertions and defaults are always enforced** at runtime. Make your functions self-documenting by validating inputs and environment variables directly in code.

```bash

`# Set defaults and assert required arguments at the top of your function.`

`# This is clearer and more reliable than relying on comments alone.`

`function deploy_service() {`

  `local -i expected_args=2`

  `if [ $# -ne $expected_args ]; then`

    `err "Expected $expected_args arguments, got $#"`

    `exit 1`

  `fi`

  `local version="${1:-v2}"`

  `local service_name="${2:?💥 'service_name' is required. No fallback makes sense here. Print a usage message instead.}"`

  `foobarbaz "${service_name:?💥}" "${version:?💥}"`

`}`

```

[https://google.github.io/styleguide/shellguide.html#function-comments](https://google.github.io/styleguide/shellguide.html#function-comments)

## Formatting

### 💡 Indentation and Line Length

**💡 Use 2 spaces for indentation — never tabs.** This ensures code looks consistent and predictable across all editors. If someone uses 8-space tabs and you open their script in a 2-space environment, it’ll appear barely indented. Likewise, your neatly indented code might look sparse in theirs. Standardizing on spaces (specifically 2) ensures structure is preserved, regardless of editor settings.

💡 This is **especially important in indentation-sensitive scripting languages** like Bash, where misaligned blocks can introduce subtle, hard-to-detect logic errors.

**💡 Stick to a soft line length limit of 120 characters** to keep lines readable without excessive wrapping.

**Exception**: The only valid use of tabs is for indenting the body of `<<-` here-documents.

[https://google.github.io/styleguide/shellguide.html#s5.1-indentation](https://google.github.io/styleguide/shellguide.html#s5.1-indentation)  
[https://google.github.io/styleguide/shellguide.html#s5.2-line-length-and-long-strings](https://google.github.io/styleguide/shellguide.html#s5.2-line-length-and-long-strings)

### 💡 Variable Expansion and Command Substitution

**Always quote** variables and command substitutions. Never use backticks.

💡 For variables, either assert their presence or provide a fallback:

* Use an inline assertion like `${foo:?💥}` to fail fast. Include a 🔥 or 💥 emoji to make errors pop in CI logs.  
* Use a fallback like `${foo:-'ARBITRARY'}` to signal that an empty or default value is acceptable.

💡 Sprinkling inline assertions (especially in GitHub Actions where env variables are overused) creates airtight null checks, forcing you to decide between failing loudly or handling defaults gracefully. As a result, unguarded expansions like `$foo` or `${foo}` become a clear code smell — they convey neither intent nor safety.

[https://github.com/anordal/shellharden/blob/master/how_to_do_things_safely_in_bash.md#the-first-thing-to-know-about-bash-coding](https://github.com/anordal/shellharden/blob/master/how_to_do_things_safely_in_bash.md#the-first-thing-to-know-about-bash-coding)  
[https://google.github.io/styleguide/shellguide.html#s5.6-variable-expansion](https://google.github.io/styleguide/shellguide.html#s5.6-variable-expansion)  
[https://google.github.io/styleguide/shellguide.html#s5.7-quoting](https://google.github.io/styleguide/shellguide.html#s5.7-quoting)

## Features and Bugs

### ShellCheck

The [ShellCheck project](https://www.shellcheck.net/) identifies common bugs and warnings for your shell scripts. It is recommended for all scripts, large or small.

[https://google.github.io/styleguide/shellguide.html#s6.1-shellcheck](https://google.github.io/styleguide/shellguide.html#s6.1-shellcheck)

### 💡 Test, `[ … ]`, and `[[ … ]]`

Prefer single-bracket conditionals for if statements. Everywhere else, use `test`. 

Double-bracket conditions have more features. But they have good POSIX substitutes for the most part:

- **Pattern matching ([[ $path == *.png || $path == *.gif ]]):** This is what `case` is for. 💡 But there are some patterns that are more straightforward to express in regex so do what’s best for your use case.  
- **Logical operators:** The usual suspects && and || work just fine – outside commands – and can be grouped with group commands: `if { true || false; } && true; then echo 1; else echo 0; fi`.  
- **Checking if a variable exists** `[[ -v varname ]]`: This is one of the good uses of double bracket conditions, 💡 but our `”${foo:-💥}”` is a good alternative here.

[https://google.github.io/styleguide/shellguide.html#s6.3-tests](https://google.github.io/styleguide/shellguide.html#s6.3-tests)  
[https://github.com/anordal/shellharden/blob/master/how_to_do_things_safely_in_bash.md#should-i-use-double-bracket-conditions](https://github.com/anordal/shellharden/blob/master/how_to_do_things_safely_in_bash.md#should-i-use-double-bracket-conditions)

### Testing Strings

Always prefer the `=` operator over flags like `test -z`.

```bash
if [ “${foo:-’’}” = “” ]; then
  do_something
fi

```  
[https://github.com/anordal/shellharden/blob/master/how_to_do_things_safely_in_bash.md#are-empty-string-comparisons-any-special](https://github.com/anordal/shellharden/blob/master/how_to_do_things_safely_in_bash.md#are-empty-string-comparisons-any-special)  
[https://google.github.io/styleguide/shellguide.html#s6.4-testing-strings](https://google.github.io/styleguide/shellguide.html#s6.4-testing-strings)

### Wildcard Expansion of Filenames

Use an explicit path when doing wildcard expansion of filenames.

As filenames can begin with a `-`, it’s a lot safer to expand wildcards with `./*` instead of `*`.

```shell
# Here's the contents of the directory:

# -f  -r  somedir  somefile



# Incorrectly deletes almost everything in the directory by force

psa@bilby$ rm -v *

removed directory: somedir'

removed somefile'

# As opposed to:

psa@bilby$ rm -v ./*

removed ./-f'

removed ./-r'

rm: cannot remove ./somedir': Is a directory

removed ./somefile'

```  
[https://google.github.io/styleguide/shellguide.html#s6.5-wildcard-expansion-of-filenames](https://google.github.io/styleguide/shellguide.html#s6.5-wildcard-expansion-of-filenames)

### 💡 Eval

Google prohibits this but as long as you follow this style guide, it shouldn’t be an issue. For most cases, there’s always a simpler and usually more verbose alternative to `eval`. One special case where eval is highly valuable is setting up a bash environment such as the `shopt` and helper functions that help enforce this style guide. In Github Actions, you set a multiline env variable containing a bash script and then all run blocks simply have to do `eval “$__bash_setup__”`. 

[https://google.github.io/styleguide/shellguide.html#s6.6-eval](https://google.github.io/styleguide/shellguide.html#s6.6-eval)

### Arrays

You are encouraged to use arrays.

[https://github.com/anordal/shellharden/blob/master/how_to_do_things_safely_in_bash.md#use-arrays-ftw](https://github.com/anordal/shellharden/blob/master/how_to_do_things_safely_in_bash.md#use-arrays-ftw)  
[https://google.github.io/styleguide/shellguide.html#arrays](https://google.github.io/styleguide/shellguide.html#arrays)

### Pipes to While

Use process substitution or the `readarray` builtin (bash4+) in preference to piping to `while`. Pipes create a subshell, so any variables modified within a pipeline do not propagate to the parent shell.

[https://google.github.io/styleguide/shellguide.html#pipes-to-while](https://google.github.io/styleguide/shellguide.html#pipes-to-while)

### Arithmetic

Always use `(( … ))` or `$(( … ))` rather than `let` or `$[ … ]` or `expr`..

[https://google.github.io/styleguide/shellguide.html#arithmetic](https://google.github.io/styleguide/shellguide.html#arithmetic)

### Aliases

Although commonly seen in `.bashrc` files, aliases are prohibited in scripts. As the [Bash manual](https://www.gnu.org/software/bash/manual/html_node/Aliases.html) notes:

For almost every purpose, shell functions are preferred over aliases.

[https://google.github.io/styleguide/shellguide.html#aliases](https://google.github.io/styleguide/shellguide.html#aliases)

## Naming Conventions

### 💡 Variables Names

**Environment and exported variables** must use `SCREAMING_SNAKE_CASE`. Everything else must be `snake_case`. This helps separate script variables from `SYSTEM_ENV_VARIABLE` that actually affect the behavior of an external application.

[https://google.github.io/styleguide/shellguide.html#s7-naming-conventions](https://google.github.io/styleguide/shellguide.html#s7-naming-conventions)

💡 Source Filenames

Lowercase, with underscores to separate words. 💡 Whitespaces are prohibited.

[https://google.github.io/styleguide/shellguide.html#s7.4-source-filenames](https://google.github.io/styleguide/shellguide.html#s7.4-source-filenames)

### 💡 Declare, Readonly, and Local Variables

`💡 readonly` and `declare` are prohibited. This style favors clarity and flexibility over immutability paranoia. 

Declare function-specific variables with `local`. This avoids polluting the global namespace and inadvertently setting variables that may have significance outside the function.

[https://google.github.io/styleguide/shellguide.html#constants-environment-variables-and-readonly-variables](https://google.github.io/styleguide/shellguide.html#constants-environment-variables-and-readonly-variables)

[https://google.github.io/styleguide/shellguide.html#use-local-variables](https://google.github.io/styleguide/shellguide.html#use-local-variables)

## Calling Commands

### Builtin Commands vs. External Commands {#builtin-commands-vs.-external-commands}

Given the choice between invoking a shell builtin and invoking a separate process, choose the builtin.

Prefer the use of builtins such as the [*Parameter Expansion*](https://www.gnu.org/software/bash/manual/html_node/Shell-Parameter-Expansion.html) functionality provided by `bash` as it’s more efficient, robust, and portable (especially when compared to things like `sed`).

```bash

# Prefer this:

addition="$(( X + Y ))"

substitution="${string/#foo/bar}"



shopt -s extglob

case “${string}” in

 foo:+([[:digit:]])) extractions=”${BASH_REMATCH[1]}” ;;

esac



# Instead of this:

addition="$(expr "${X}" + "${Y}")"

substitution="$(echo "${string}" | sed -e 's/^foo/bar/')"

extraction="$(echo "${string}" | sed -e 's/foo:([0-9])/1/')"


```

[https://google.github.io/styleguide/shellguide.html#s8.2-builtin-commands-vs-external-commands](https://google.github.io/styleguide/shellguide.html#s8.2-builtin-commands-vs-external-commands)

### How to end a bash script

Goal: The script's exit status should convey its overall success or failure.

Reality: The script's exit status is that of the last command executed.

There is a wrong way to end a bash script: Letting a command used as a condition be the last command executed, so that the script "fails" if the last condition is false. While that might happen to be correct for a script, it is a way to encode the exit status that looks accidental and is easily broken by adding or removing code to the end.

[https://github.com/anordal/shellharden/blob/master/how_to_do_things_safely_in_bash.md#how-to-end-a-bash-script](https://github.com/anordal/shellharden/blob/master/how_to_do_things_safely_in_bash.md#how-to-end-a-bash-script)

# 

# Github Actions

## Mitigating Script Injection in GitHub Actions

### Use an action instead of inline scripts

```yaml
uses: fakeaction/checktitle@v3
with:
  title: ${{ github.event.pull_request.title }}
  
```
https://docs.github.com/en/actions/security-for-github-actions/security-guides/security-hardening-for-github-actions?learn=getting_started&learnProduct=actions#using-an-action-instead-of-an-inline-script-recommended

### 💡 Use an intermediate environment variable  

```yaml
env:
  title:              ${{ github.event.pull_request.title }}
  RANDOM_API_KEY:     ${{ secrets.RANDOM_API_KEY          }}
  AWS_DEFAULT_REGION: ‘ap-southeast-1’
run: |  
  if [[ "$title" =~ ^octocat ]]; then …

```

1. Prevents input from being parsed as shell code

2. Keeps the value in memory, not embedded in the script

3. 💡 Use `snake_case` unless it’s sensitive data or an environment variable that affects an external command. In the latter’s case, use `SCREAMING_SNAKE_CASE.`

https://docs.github.com/en/actions/security-for-github-actions/security-guides/security-hardening-for-github-actions?learn=getting_started&learnProduct=actions#using-an-intermediate-environment-variable

### 💡 Global Helper Functions Hack

This pattern enables defining and injecting reusable Bash functions into multiple script steps in CI/CD workflows (e.g., GitHub Actions, GitLab CI, etc.).

**Benefits**

* Centralized, reusable functions (`println`, `warn`, `err`, `ignore_failure`)  
* Cleaner logs with consistent emoji-based output  
* Avoids managing external helper files

```yaml  
env:  
  __bash_setup__: |  
    set +e  
    set -uo pipefail  
    shopt -s nullglob globstar extglob  
      
    function println() {  
      if [ "$#" -gt 0 ]; then  
        printf "$@"  
      fi  
      printf '\n'  
    }

    function warn() {  
      println "⚠️ warn: $@" >&2  
      return 0  
    }  
      
    function err() {  
      println "❌ error: $@" >&2  
      return 1  
    }

    function ignore_failure() {  
      "$@" || warn "⚠️ ignored failure: $*\n"  
      return 0  
    }

# ...

      - name: Audit Dependencies  
        run: |  
          eval "$__bash_setup__"  
          ignore_failure npm audit --audit-level=high --json > package_error.json
          count=$(jq '[.metadata.vulnerabilities.high, .metadata.vulnerabilities.critical] | add' package_error.json)
          if [ "${count:-0}" -gt 0 ]; then  
              err "High or Critical vulnerabilities found: ${count:?💥}"  
              cat package_error.json | tee package_error.log  
              exit 1  
          fi  
          println "✅ No high or critical vulnerabilities found."  
          exit 0

```

### 💡 Inline Insertions and GitHub Expression Defaults

Generally, inline insertions should be placed inside `run` blocks. This ensures that environment variables are resolved correctly at runtime:

```yaml
env:
  foo: 'barbaz'
run: |
  do_something "${foo:?💥}"
```

However, when using **GitHub Actions `uses:` steps**, you cannot interpolate shell variables. In these cases, use GitHub expressions instead, and default to the 💥 emoji as a visual indicator of a missing or unset value:

```yaml
name: Do something
uses: Something/something@v1
with:
  foo: ${{ secrets.MY_SECRET || '💥' }}
```

This approach helps catch missing configuration early and keeps failure signals consistent across contexts.
