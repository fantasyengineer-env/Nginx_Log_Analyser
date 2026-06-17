# log-analyser

A Bash script that parses an Nginx access log (combined format) and reports the top 5 IP addresses, requested paths, response status codes, and user agents.

## Log format

The script expects the standard Nginx "combined" log format:

```
IP - - [date:time +offset] "METHOD /path HTTP/x.x" status size "referrer" "user-agent"
```

## Usage

```bash
chmod +x log-analyser
./log-analyser /path/to/nginx-access.log
```

If no path is given, or the given path doesn't exist, the script prints a usage message and exits with status `1`.

## What it reports

1. **Top 5 IP addresses** by request count
2. **Top 5 requested paths**
3. **Top 5 response status codes**
4. **Top 5 user agents**

Each section is capped by the `TOP_N` variable (default `5`), so the limit can be changed in one place without touching the rest of the script.

## How it works

### Safety flags

```bash
set -euo pipefail
```

This stops the script early instead of letting it continue after an error.

- `-e` exits immediately if any command returns a non-zero status.
- `-u` treats use of an undefined variable as an error, which catches typos.
- `-o pipefail` makes the whole pipeline fail if any command in it fails.

### Argument handling

```bash
LOGFILE="${1:-}"
```

`$1` is the first command-line argument. The `${1:-}` form falls back to an empty string instead of erroring if `$1` is unset, which matters because `-u` would otherwise stop the script immediately, before a proper usage message can be printed.

The script then checks that a path was actually provided and that it exists, exiting with status `1` and an error message otherwise. An empty or missing file means there's no data to analyze, so there's no point continuing.

### Top IP addresses

Takes the first field of each log line (the IP address) and counts occurrences:

```bash
awk '{print $1}' "$LOGFILE" \
    | sort \
    | uniq -c \
    | sort -rn \
    | head -n "$TOP_N" \
    | awk '{printf "%s - %s requests\n", $2, $1}'
```

### Top requested paths

```bash
grep -oP '"\S+ \S+ HTTP/[0-9.]+"' "$LOGFILE" \
    | awk -F'"' '{print $2}' \
    | awk '{print $2}' \
    | sort \
    | uniq -c \
    | sort -rn \
    | head -n "$TOP_N" \
    | awk '{printf "%s - %s requests\n", $2, $1}' || true
```

The `grep` pattern is anchored with literal `"` characters to match the exact boundaries of the request field, for example `"GET /api/v1/users HTTP/1.1"`. Breaking down the pattern:

- `-o` outputs only the matched portion of each line.
- `-P` enables Perl-compatible regex.
- `\S+` greedily matches one or more non-whitespace characters.
- `HTTP/` is matched literally.
- `[0-9.]+` is a character class matching any of those characters, in any order, any number of times. (A specific count, minimum, or range can be enforced with `{n}`, `{n,}`, or `{n,}` respectively.)

The first `awk -F'"'` call uses `"` as the field delimiter to pull out the content between the quotes as a single field. The second `awk` call then splits on default whitespace to extract just the path.

### Top status codes

```bash
grep -oP '"\s*\K\d{3}(?=\s+\d+\s+")' "$LOGFILE" \
    | sort \
    | uniq -c \
    | sort -rn \
    | head -n "$TOP_N" \
    | awk '{printf "%s - %s requests\n", $2, $1}' || true
```

Breaking down the pattern:

- `\s*` matches any whitespace before the status code.
- `\K` resets the match start, so everything before it is excluded from the output.
- `\d{3}` matches exactly three digits, the status code itself.
- `(?=\s+\d+\s+")` is a lookahead: it requires whitespace, then digits (the response size), then whitespace and a closing quote, but doesn't include any of that in the match.

The lookahead exists to make sure the three digits captured are the status code and not the response size, since the size could also happen to be three digits.

### Top user agents

```bash
grep -oP '"\K[^"]+(?="$)' "$LOGFILE" \
    | sort \
    | uniq -c \
    | sort -rn \
    | head -n "$TOP_N" \
    | awk '{count=$1; $1=""; sub(/^ /, ""); printf "%s - %s requests\n", $0, count}' || true
```

Breaking down the pattern:

- `"` matches the literal opening quote.
- `\K` resets the match start so the quote itself isn't included in the output.
- `[^"]+` greedily matches one or more characters that are *not* a double quote.
- `(?="$)` is a lookahead requiring a closing quote at the end of the line, without including it in the match.

Because user agent strings contain spaces, the final `awk` call needs extra handling: `$1` (the count from `uniq -c`) is saved, then cleared from the line, and `sub(/^ /, "")` strips the leading blank space that clearing `$1` leaves behind. This avoids the count getting tangled up with the first word of the user agent string.

### Why `|| true`

`grep` exits with status `1` when it finds no matches at all in a file, even though that's a perfectly normal outcome rather than an actual error. Under `set -o pipefail`, that non-zero exit would otherwise propagate and silently kill the rest of the script. Appending `|| true` to the **end of the whole pipeline** (after the final `awk` call) tells Bash to treat "no matches found" as success for that section, so the script keeps running and reports an empty result instead of crashing.

This only works when placed at the very end. Bash's `||` operator binds at the level of the entire pipeline it sits inside, not just the command beside it, so `grep ... || true | sort | ...` is actually parsed as `(grep ...) || (true | sort | ...)` rather than `(grep ... || true) | sort | ...`. If `grep` succeeds, which it normally does, the second half never executes and `grep`'s raw matches get printed unsorted and uncounted. Placing `|| true` after the last command in the pipeline avoids this entirely.

## Notes

- Requires GNU `grep` with PCRE support (`-P` flag) and standard GNU `awk`, `sort`, `uniq`, and `head`.
- Tested with `bash`. Running with a non-Bash POSIX `sh` may fail on the `pipefail` option, which isn't supported by all shells.
