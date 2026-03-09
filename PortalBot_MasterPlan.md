# PortalBot
## Master Plan · Master Prompt · Full Code Review

> Discord Bot for the Portal Speedrunning Community  
> Author: @valoix · March 2026 · Python 3.10+ / discord.py 2.x

---

## Table of Contents

1. [Project Overview](#1-project-overview)
2. [Architecture — How the Files Connect](#2-architecture--how-the-files-connect)
3. [Master Plan](#3-master-plan)
4. [Master Prompt](#4-master-prompt)
5. [Full Code Review](#5-full-code-review)
6. [Bug Tracker](#6-bug-tracker)
7. [Quick-Start Cheatsheet](#7-quick-start-cheatsheet)

---

# 1. Project Overview

PortalBot is a custom Discord bot built specifically for the Portal speedrunning community (P1SR). It handles real-time community tools — like tracking world records, converting speedrun times/ticks, managing community pins, and auto-moderating bad actors — all from inside Discord chat.

Think of it as your server's full-time staff member that never sleeps: it announces WRs, converts times, pins funny messages, nukes spam, and converts game recordings for sharing. It's written in Python using the discord.py library, runs on any machine with Python 3.10 or newer, and only needs a single bot token to get going.

---

## What Is the Stack?

| Layer | Technology | What It Does |
|-------|-----------|--------------|
| Bot Framework | discord.py >= 2.0 | Talks to Discord's API |
| Language | Python 3.10+ | Core logic of every feature |
| Config | python-dotenv | Keeps secrets out of code |
| Video | moviepy >= 2.0 + ffmpeg.exe | MKV → MP4 conversion |
| Data | JSON (wr_archive.json) | World record database |
| State | Plain .txt files | Cube counter, pin IDs |

---

## Feature Summary

| Command / Feature | What It Does | Who Can Use It |
|-------------------|-------------|----------------|
| `help++` | Lists all commands | Everyone |
| `cube++` / `cube--` | Increments/decrements the community cube counter | Community Contributor+ |
| `wr++ search name/category/year` | Searches the WR archive and DMs results | Everyone |
| `wr++ <post>` | Posts a new WR announcement and archives it | Privileged Roles |
| `time2tick++ <time>` | Converts run time to tick count, validates it | Everyone |
| `tick2time++ <ticks>` | Converts tick count to real time | Everyone |
| `emergencyexit++` | Hard shuts down the bot process | Moderators (P1SR server only) |
| Pinnerino (📌 reaction) | Community pins messages when 10 people react 📌 | Everyone (reaction threshold) |
| MKV → MP4 auto-convert | Any .mkv attachment gets converted and resized | Automatic on upload |
| Dox Protection | Auto-bans users whose message/username contains protected terms | Automatic |
| Timeout Terms | 24-hour timeout for spam patterns (invite links, etc.) | Automatic |

---

# 2. Architecture — How the Files Connect

PortalBot is split across six Python files, each with a clear job. Here is how they fit together:

| File | Role | Analogy |
|------|------|---------|
| `main.py` | Entry point | The ON switch — it just calls `bot.run_discord_bot()` |
| `bot.py` | Brain / event router | The front desk — receives every message and reaction, decides what to do |
| `responses.py` | Cube counter + utilities | The toolbox — shared helpers that bot.py and others call |
| `ticks.py` | Speedrun time math | The calculator — pure math, no Discord code in here at all |
| `demoparser.py` | Binary .dem file reader | The forensics lab — reads raw bytes from game demo files |
| `wr_archive.json` | WR database | The filing cabinet — a list of every recorded WR |

---

## Data Flow Diagram

When a message arrives, this is the chain of calls:

```
Discord → on_message() in bot.py
          ├── Security checks (dox, timeout terms)  → responses.py
          ├── help++           → inline in bot.py
          ├── cube++ / cube--  → responses.handle_response()
          ├── time2tick++      → ticks.time_to_tick()
          ├── tick2time++      → ticks.tick_to_time()
          ├── wr++             → handle_wr_command() → bot.py WR functions
          └── .mkv attachment  → handle_mkv_conversion() → moviepy

Discord → on_raw_reaction_add() in bot.py
          └── 📌 reaction      → handle_pin_reaction() → responses.generate_message_link()
```

---

# 3. Master Plan

This section is a complete roadmap: where the project is today, what should be tackled first, and what the long-term vision looks like. Use this as your north star when deciding what to work on next.

---

## 3.1 Current State (What Works Right Now)

- Commands: `help++`, `cube++`/`cube--`, `wr++ search/post`, `time2tick++`, `tick2time++`, `emergencyexit++`
- Pinnerino: community-driven message pinning via 📌 reaction threshold
- Auto-moderation: dox ban, timeout terms, ESC sequence filtering
- MKV → MP4 video conversion with adaptive resolution (240p / 480p / 720p by file size)
- WR archive backed by `wr_archive.json` — searchable by name, category, or year
- Demo parser: reads HL2DEMO binary format, extracts packet ticks and header metadata
- Easter eggs: 4104 reaction trigger, 1163 heart reaction in tick2time

---

## 3.2 Immediate Fixes (Do These First)

These are bugs or risks that exist in the current code. Fix them before adding anything new.

1. **Fix the MKV converter** — it renames the file from `.mkv` to `.mp4` without actually re-encoding it. This creates a broken file for most players. Use moviepy's `write_videofile()` on the original path instead of `os.rename()`.

2. **Fix the file handle leak in `demoparser.py`** — the code opens a file inside the `while` loop but only closes it after the loop, and only if parsing succeeded. Wrap the file open in a `with` statement.

3. **Add a `downloads/` directory creation check** — the bot crashes on first run if `downloads/` does not exist. Add `os.makedirs(DOWNLOADS_PATH, exist_ok=True)` in `run_discord_bot()`.

4. **Add a `pinnerino/` directory creation check** — same issue, same fix pattern.

5. **Guard the cube counter** — if `cube_count.txt` is missing or empty, `get_cube_count()` throws an exception. Initialize it to `0` if not found.

6. **Replace the bare `except: pass` blocks** — they swallow real errors silently. At minimum, log the exception with `print_colour('R', str(e))`.

---

## 3.3 Short-Term Roadmap (Next 1–3 Months)

### Persistence and Reliability

- Move `cube_count.txt`, pinnerino IDs, and `demo_info` to a SQLite database — flat files get corrupted if the bot crashes mid-write.
- Add a proper logging module (Python's built-in `logging`) instead of `print()` calls, so log files are written to disk.
- Wrap `client.run()` in a reconnect loop so the bot restarts itself after a network drop.

### Demo Parser Integration

- The demo parser (`demoparser.py`) is built but **never called** from `bot.py`. Wire it up: detect `.dem` file attachments in `on_message()` and call `Parser.parse_demo()` → `generate_embed()` to post parsed info.
- Fix `generate_embed()` — it re-reads `archive_demo_info.txt` for `total_ticks` but the file accumulates data from all demos. It should only count ticks from the current demo.

### WR Archive Improvements

- Add `wr++ add` as a proper explicit sub-command instead of falling through to the else branch.
- Add `wr++ delete` and `wr++ edit` for fixing mistakes.
- Add date sorting so search results come back newest-first.
- Validate the `time` field format on insert (must match `MM:SS.sss` or `H:MM:SS.sss`).

### MKV Converter

- Fix the container re-encoding (see Immediate Fixes above).
- Add progress feedback — long videos time out Discord's 3-second acknowledgement window. Send a "Working on it..." message immediately.
- Add a file size limit warning before attempting conversion.

---

## 3.4 Medium-Term Roadmap (3–6 Months)

- **Slash Commands** — migrate from text prefix commands (`wr++`, `help++`) to Discord's native slash command system. This gives users autocomplete, typed parameters, and a much better experience.
- **Leaderboard embeds** — instead of DM'ing raw text, build rich Discord embeds with medals (🥇🥈🥉) for the top 3 in each category.
- **Speedrun.com API integration** — auto-pull live WR data from the SRC API rather than relying on manual `wr++` posts.
- **Role-based access via config file** — right now `PRIVILEGED_ROLES` is hardcoded in two places. Move it to a `config.json` or `.env` so it can be changed without a code edit.
- **Unit tests for `ticks.py`** — the tick math is the most used feature and currently has zero tests. Add pytest test cases for known good/bad values.

---

## 3.5 Long-Term Vision (6+ Months)

- **Multi-server support** — most of the code assumes `P1SR_SERVER_ID`. Refactor to support multiple servers with per-server config.
- **Web dashboard** — a simple Flask/FastAPI page where admins can view the WR archive, edit records, and see moderation logs without needing Discord access.
- **Automated demo submission workflow** — runners submit a `.dem` file, the bot parses it, extracts the tick count, converts it to time, and posts a formatted run submission embed for verifiers to approve.
- **Clip auto-tagging** — use ffmpeg metadata to detect game chapter/map name from MKV files and auto-tag the converted clip.

---

# 4. Master Prompt

Copy this entire block and paste it at the start of any AI-assisted coding session on PortalBot. It gives the AI full context so you don't have to explain things from scratch each time.

---

```
You are a Python expert helping a hobbyist maintain and extend PortalBot, a Discord bot for the
Portal speedrunning community (P1SR). Here is everything you need to know about the project:

PROJECT SUMMARY
───────────────
PortalBot is a Python 3.10+ Discord bot using discord.py 2.x. It runs as a single process on a
host machine (Windows or Linux). It is NOT hosted on a cloud platform — it runs locally. Config
comes from a .env file loaded by python-dotenv.

FILE MAP
────────
main.py        — entry point. Calls bot.run_discord_bot(). Nothing else.
bot.py         — ALL Discord event handlers: on_ready, on_message, on_raw_reaction_add.
               — All command routing, WR functions, pin handler, MKV converter.
responses.py   — cube counter read/write, dox detection, utility helpers, colored console output.
ticks.py       — pure math: time_to_tick() and tick_to_time(). No Discord imports.
demoparser.py  — binary HL2DEMO file reader: Reader, Demo, Parser classes.
wr_archive.json — flat JSON list of WR records: {name, category, time, date, link}.
cube_count.txt — single integer, the current cube count.
pinnerino/     — pinnerino_message_ids.txt stores Discord message IDs already pinned.
downloads/     — temp folder for in-progress video conversions.

KEY CONSTANTS (bot.py)
──────────────────────
PIN_CHANNEL_ID      = 1192784040634351757   # where pinned messages go
WR_CHANNEL_ID       = 1174320230386901023   # where WR announcements go
MOD_LOG_CHANNEL_ID  = 778521955435413514
P1SR_SERVER_ID      = '305456639530500096'  # only server where most commands work
REACTION_PIN_THRESHOLD = 10                 # 📌 reactions needed to pin
PRIVILEGED_ROLES    = ['Community contributor', 'SRC verifier', 'Moderation Team', 'Admin']

WR CATEGORY CODES
─────────────────
g  = Glitchless
i  = Inbounds
o  = Out of Bounds
nl = NoSLA Legacy
nu = NoSLA Unrestricted

TICK MATH
─────────
Portal runs at 66.667 ticks/second, meaning 1 tick = 0.015 seconds exactly (1/66.667).
A valid run time must convert to a whole number of ticks. ticks.py validates this.

SECURITY MODEL
──────────────
DOX_TERMS and TIMEOUT_TERMS come from the .env file — NEVER hardcode them.
Dox terms trigger an immediate ban.
Timeout terms trigger a 24-hour timeout and log to MOD_LOG_CHANNEL_ID.
Both also delete the triggering message.

KNOWN BUGS TO WATCH OUT FOR
────────────────────────────
1. MKV converter uses os.rename() instead of actual re-encoding — broken output.
2. demoparser.py has a file handle issue inside the while loop.
3. downloads/ and pinnerino/ directories must exist before first run.
4. cube_count.txt must exist and contain an integer.
5. Bare 'except: pass' blocks in several places hide real errors.

CODING STYLE RULES FOR THIS PROJECT
─────────────────────────────────────
- Async all the way for Discord operations — never use blocking I/O inside an async function
  without offloading it.
- Keep ticks.py pure — no Discord imports, no file I/O. It's a math module only.
- All file paths go through BASE_PATH — never hardcode absolute paths.
- Secrets (tokens, terms) belong in .env only — not in any Python file.
- Use responses.print_colour() for ALL console output so the log is color-coded.
- When adding commands, add them to the help++ output in on_message() too.

WHAT I WILL ASK YOU TO DO
──────────────────────────
I may ask you to: fix bugs, add new commands, refactor existing code, write tests, explain what
something does, or help me migrate to slash commands. Always give me complete, runnable code.
Do not truncate. If a file is being changed, show me the full updated file, not just the diff.
```

---

# 5. Full Code Review

Every file, explained clearly. If something is clever, this section explains the trick. If something is broken or risky, it is flagged clearly.

---

## 5.1 `main.py` — The On Switch

This is intentionally tiny. Its only job is to be the file you run.

```python
import bot

# -----------------------------
#   PORTAL BOT MAIN BRANCH
# -----------------------------

if __name__ == "__main__":
    bot.run_discord_bot()
```

The `if __name__ == '__main__':` guard is Python's way of saying "only run this when you execute the file directly — not when another file imports it." This means if someone does `import main` somewhere, it won't accidentally start the bot. This is a clean, correct pattern.

**Rating: ✅ Clean. No issues.**

---

## 5.2 `bot.py` — The Brain

This is the largest file and the one you will edit most often. It has four logical sections: configuration, WR archive functions, helper functions, and the main bot event handlers.

### Configuration Block

```python
BASE_PATH = os.getenv("BASE_PATH", os.path.dirname(os.path.abspath(__file__)))
```

This is smart — it lets you override where files are stored by setting `BASE_PATH` in `.env`. If you don't set it, it defaults to the folder where `bot.py` lives. Good portability.

```python
DOX_TERMS = os.getenv("DOX_TERMS", "").split(",") if os.getenv("DOX_TERMS") else []
```

This reads a comma-separated list from `.env` and splits it into a Python list. The conditional guard prevents a `['']` list when the env var is empty — a subtle but important detail.

### WR Archive Functions

`load_wr_archive()` and `save_wr_archive()` are the read/write pair for `wr_archive.json`. They open, parse, and write a JSON list of record dictionaries. Clean and straightforward.

`add_wr_record()` normalizes the category string through `CATEGORY_ALIASES` before saving — so whether someone types `'inbounds'`, `'inb'`, or `'i'`, it always gets stored as `'i'`. Good defensive design.

`search_wr_by_year()` uses a string tail match:

```python
r['date'].endswith(year) or r['date'][-4:] == year[-4:]
```

This works but is fragile — it depends on the date being formatted as `DD/MM/YYYY`. If any record has a different format, the search silently misses it. A better approach: parse the date with `datetime.strptime()` and compare the `.year` attribute.

### `handle_pin_reaction()` — The Pinnerino System

When someone adds a 📌 emoji to a message, Discord fires `on_raw_reaction_add`. The bot checks if the reaction count has exactly hit 10 (`REACTION_PIN_THRESHOLD`). If yes, it checks a text file to make sure the message hasn't already been pinned, then formats and sends it to the pin channel.

The reaction count check uses `==` not `>=`. This means if the bot is offline when the 10th reaction happens, it will miss it entirely and never catch up. A safer approach is `>=` rather than `==`.

`sanitize_mentions()` replaces `@` with `@ ` (with a space) to prevent the bot from accidentally pinging everyone or a specific user when echoing pinned message content. Good safety habit.

### `handle_mkv_conversion()` — The Video Converter

This is the most complex function and also has the most significant bug.

> **🔴 THE BUG:** `os.rename(input_path, output_path)` just renames `input_video.mkv` to `output_video.mp4`. It does **NOT** transcode the video. It changes the filename but not the container or codec. Most video players will refuse to play or will render it incorrectly because the internal data is still MKV format.

**The Fix:** Remove the `os.rename()` line. Instead, pass the original `.mkv` path directly into `VideoFileClip()`:

```python
clip = VideoFileClip(input_path)        # open the .mkv directly
resized_clip.write_videofile(resized_path)  # this DOES the actual encoding
```

The adaptive resolution logic IS correct — it picks 240p for files over 30MB, 480p for 20–30MB, and 720p for anything smaller. This keeps Discord's 8MB upload limit in mind.

### `on_message()` — The Main Event Handler

This function runs every time anyone posts anything in any channel on the server. It is the core of the bot. It runs in this order every time:

1. Self-check: ignore messages from the bot itself.
2. ESC sequence check: block terminal escape codes from being processed.
3. Dox check on message content → auto-ban if hit.
4. Dox check on username → auto-ban if hit.
5. Timeout terms check → 24hr mute + mod log if hit.
6. Command routing: `help++`, `cube++`, tick conversions, `wr++`, `dm++`, etc.
7. Attachment handling: MKV conversion.
8. Log the message to console.

The `dm++` command is restricted to a specific Discord username (`'valoix'`) with a plain string comparison. Discord usernames are not globally unique in the new username system — this could be spoofed. A more reliable check would be comparing the author's numeric user ID (`message.author.id == 123456789`) instead of the display name.

The **4104 easter egg**: if a message contains `'4104'`, there is a 1-in-10 random chance the bot reacts with a custom emoji. `1163` in the tick converter gets a ❤️. These are community in-jokes baked into the code.

---

## 5.3 `responses.py` — The Toolbox

This module provides helper utilities. It is imported by `bot.py` and relies on the same `.env` file.

### Cube Counter

`get_cube_count()` opens `cube_count.txt`, reads a number, and returns it. `set_cube_count()` writes it back. Simple. The problem: if the file doesn't exist, the function throws a `FileNotFoundError` and the bot crashes. Add a `try/except` that returns `0` and creates the file if it's missing:

```python
def get_cube_count():
    try:
        with open(CUBE_COUNT_PATH, "r") as f:
            return int(f.read().strip())
    except (FileNotFoundError, ValueError):
        set_cube_count(0)
        return 0
```

`handle_response()` processes `cube++` and `cube--` commands. It checks `P1SR_SERVER_ID` and `PRIVILEGED_ROLES` first so the command only works in the right server and for the right users. Good gating.

### Dox Detection

`text_dox_blox()` and `name_dox_blox()` both loop through `DOX_TERMS` and check if any term appears in the message or username. They also check `if term and ...` to skip empty strings that can appear in the list if the `.env` value ends with a comma. Good defensive check.

Note: these checks are **case-sensitive**. If a `DOX_TERM` is `'BadWord'` and someone posts `'badword'`, it won't catch it. Consider doing the comparison on both `.lower()` strings:

```python
if term and term.lower() in message.lower():
    return True
```

### `print_colour()` and `print_error()`

These functions use ANSI escape codes to colorize the terminal output — Red for errors, Green for successes, Blue for info. This makes the console log much easier to read at a glance when the bot is running.

```python
colours = {
    "R": "\033[31m",  # Red
    "G": "\033[32m",  # Green
    "B": "\033[34m",  # Blue
}
```

The `\033[` is the escape character — a standard terminal color protocol that works on Linux/macOS. On Windows it works in Windows Terminal and VSCode but may not work in the old Command Prompt without enabling virtual terminal processing first.

`print_error()` maps numeric error codes to human-readable messages. Error `303` (Dox in image) is defined in the error table but never triggered anywhere in the code — image content scanning is not yet implemented.

---

## 5.4 `ticks.py` — The Calculator

This is the cleanest, most self-contained file in the project. It is pure math with zero external dependencies beyond Python's built-in `math` module.

### `minute_checker(time)`

Takes a time string like `'1:23.450'` and converts it to a total tick count.

```python
time = time.split(":")
timecount = len(time) - 1
for i in time:
    convertedSeconds += float(i) * (pow(60, timecount))
    timecount -= 1
```

This is an elegant positional sum. For `'1:23.450'`, it splits into `['1', '23.450']`, then computes `1×(60¹) + 23.450×(60⁰) = 60 + 23.450 = 83.450` seconds. Then divides by `0.015` to get ticks.

The 86,400-second cap (24 hours) prevents absurd input. Returns `False` if exceeded, which the caller interprets as invalid.

### `time_to_tick(time)`

Calls `minute_checker()` to get a (possibly fractional) tick count, then checks if it's a whole number using `.is_integer()`. If it IS a whole number, the time is valid — it lands exactly on a tick boundary. If it is NOT, it suggests the nearest floor and ceiling tick values. This is the validator the community uses to verify run legitimacy.

### `tick_to_time(ticks)`

Multiplies the tick count by `0.015` to get seconds, then calls `second_to_minute()` to format it. Has two easter eggs baked in: tick `1163` gets a ❤️, tick `4104` gets the custom server emoji.

### `second_to_minute(time)`

Formats a decimal number of seconds into `MM:SS.sss` or `H:MM:SS.sss` strings. The zero-padding logic is a bit verbose — it uses `while` loops to pad the decimal places to a fixed length. Functional, but could be replaced with Python's f-string formatting:

```python
f'{seconds:06.3f}'  # gives '05.250' for 5.25
```

---

## 5.5 `demoparser.py` — The Forensics Lab

This is the most technically sophisticated file. It reads binary `.dem` files — the raw game replay format used by Portal (Source Engine). Most Discord bots never touch binary data at all, so this is genuinely advanced work.

### The `Reader` Class

`Reader` wraps a `bytes` object (the raw file content) with a cursor (`self.index`) and methods to pull structured data from it:

- **`read_bytes(n)`** — Returns the next `n` bytes and advances the cursor. All other methods build on this.
- **`skip(n)`** — Advances the cursor without reading — used to jump over data sections the bot doesn't need.
- **`read_string_nulled()`** — Reads a C-style null-terminated string (ends at `0x00` byte). This is how the Source Engine stores names in headers.
- **`read_int(n, signed)`** — Reads a little-endian signed integer. "Little-endian" means the least significant byte comes first — this is how x86 CPUs store numbers and how Valve's engine writes them.
- **`read_float(n)`** — Uses `struct.unpack('<f', ...)` to decode a 4-byte IEEE 754 float. The `'<'` means little-endian, `'f'` means 32-bit float.

### The `Demo` Class

A plain data container with no methods — it's just a structured bundle of fields that get filled by the `Parser`. This is a clean Data Transfer Object pattern.

### The `Parser` Class — `parse_demo()`

This method implements the HL2DEMO binary format specification:

1. Reads the 8-byte magic string `'HL2DEMO\0'` to verify the file is a real demo.
2. Reads the 1416-byte header (protocol versions, server name, client name, map name, directory, playback stats).
3. Loops through packets. Each packet starts with a 1-byte type and a 4-byte tick number.

The packet types handled in the `match/case` block:

| Type | Name | Handling |
|------|------|----------|
| 7 | STOP | End of demo — break out of the loop |
| 1/2 | SIGNON/DATA | Skip 84 bytes of header data, then skip variable-length payload |
| 3 | SYNCTICK | No data, just move on |
| 4 | CONSOLECMD | Skip a variable-length string payload |
| 5 | USERCMD | Skip 4-byte cmd ID plus variable payload |
| 6 | DATATABLES | Skip variable-length payload |
| 8 | STRINGTABLES | Skip variable-length payload |

### The File Handle Issue — Detailed

The `with open()` statement inside the loop is actually fine — it auto-closes after each iteration. But the `file.close()` at the bottom of the function is redundant and confusing: it references the variable from the last loop iteration and will fail if the loop never ran. Remove it:

```python
# Inside the while loop — this is fine, auto-closes each time:
with open("./demo_info/archive_demo_info.txt", "a") as file:
    file.write(f"{tick}\n")

# After the loop — REMOVE this line, it's redundant:
file.close()  # ← delete this
```

### `generate_embed()`

This builds a Discord Embed (a formatted card) with all the parsed demo info. The design issue: it counts ALL lines in `archive_demo_info.txt`, not just the ticks from the current demo. Every demo adds to that running total, so the "Measured Ticks" field will be wildly wrong after the first demo is processed.

The fix: use `len(self.demo.ticks)` instead of reading the archive file — the ticks were already collected into the `Demo` object during `parse_demo()`:

```python
# Instead of:
with open("./demo_info/archive_demo_info.txt", "r") as file:
    for line in file.readlines():
        total_ticks += 1

# Use:
ticks_len = len(self.demo.ticks)
```

---

## 5.6 `wr_archive.json` — The Filing Cabinet

A flat JSON array of world record objects. Currently contains community WRs dating back to 2015. Example entry:

```json
{
    "name": "Msushi",
    "category": "o",
    "time": "7:31.800",
    "date": "05/07/2017",
    "link": "https://discord.com/channels/305456639530500096/..."
}
```

The archive contains records with `"link": "N/A"` for older records where the original Discord message is lost. This is fine — it's honest about data availability.

Dates are stored as `DD/MM/YYYY` strings. This is fine for display but makes date-range queries harder. If you ever need to search by date range or sort by date, consider adding a `date_epoch` field or parsing with `datetime`.

---

# 6. Bug Tracker

A consolidated list of every issue found during code review, with severity ratings and suggested fixes.

| Severity | File / Location | Issue | Fix |
|----------|----------------|-------|-----|
| 🔴 HIGH | `bot.py: handle_mkv_conversion()` | `os.rename()` creates a broken `.mp4` (not re-encoded) | Remove rename; use `VideoFileClip(input_path)` directly |
| 🔴 HIGH | `bot.py: run_discord_bot()` | `downloads/` dir missing on first run → crash | `os.makedirs(DOWNLOADS_PATH, exist_ok=True)` |
| 🟠 MED | `demoparser.py: parse_demo()` | Redundant `file.close()` after `with` block; confusing code | Remove the `file.close()` at bottom of while loop |
| 🟠 MED | `demoparser.py: generate_embed()` | Counts ALL archive ticks, not just current demo's | Use `len(self.demo.ticks)` instead of reading archive file |
| 🟠 MED | `responses.py: get_cube_count()` | `FileNotFoundError` if `cube_count.txt` missing | `try/except` that initializes to `0` and creates file |
| 🟠 MED | `bot.py: pinnerino path` | `pinnerino/` dir missing on first run → crash | `os.makedirs(PINNERINO_PATH, exist_ok=True)` |
| 🟡 LOW | `bot.py: handle_pin_reaction()` | Uses `== 10` check — misses pin if bot was offline at 10th reaction | Change to `>= REACTION_PIN_THRESHOLD` |
| 🟡 LOW | `bot.py: dm++ command` | Compares username string instead of user ID — spoofable | Compare `message.author.id == <numeric ID>` |
| 🟡 LOW | `responses.py: dox checks` | Case-sensitive: `'badword'` won't match `'BadWord'` | Lowercase both: `term.lower() in message.lower()` |
| 🟡 LOW | `bot.py: search_wr_by_year()` | String tail match is fragile for non-standard date formats | Parse with `datetime.strptime`, compare `.year` |
| ℹ️ INFO | `demoparser.py (all)` | Parser is never called from `bot.py` — built but not wired up | Detect `.dem` attachments in `on_message()` and call `Parser` |
| ℹ️ INFO | `responses.py: Error 303` | "Dox in image" error code defined but image scanning not implemented | Future feature — image content scanning |

---

# 7. Quick-Start Cheatsheet

Everything you need to get from zero to running bot in one read.

---

## First-Time Setup

1. Install Python 3.10+ from [python.org](https://python.org)
2. Clone or unzip the project into a folder.
3. Open a terminal in that folder and run:
   ```bash
   pip install -r requirements.txt
   ```
4. Copy `.env-example` to `.env` and fill in your Discord bot token.
5. Create the missing directories:
   ```bash
   mkdir downloads
   mkdir pinnerino
   ```
6. Make sure `cube_count.txt` exists with a single number:
   ```bash
   echo 0 > cube_count.txt
   ```
7. Place `ffmpeg.exe` in the project folder (Windows) or install ffmpeg via your package manager (Linux/Mac).
8. Run the bot:
   ```bash
   python main.py
   ```

---

## Environment Variables (`.env`)

| Variable | Example | Required? |
|----------|---------|-----------|
| `DISCORD_TOKEN` | `MTk4NjIy...` | **Yes** |
| `DOX_TERMS` | `term1,term2,term3` | No — but highly recommended |
| `TIMEOUT_TERMS` | `discord.gg/,@everyone` | No — but highly recommended |
| `BASE_PATH` | `/home/user/portalbot` | No — defaults to script directory |

---

## Console Color Key

| Color | Meaning | When You See It |
|-------|---------|-----------------|
| 🔴 Red | Error — something broke | Failed operations, exceptions, security events |
| 🟢 Green | Success — operation completed | WR added, video converted, DM sent |
| 🔵 Blue | Info — process is running | Download in progress, conversion starting, event fired |

---

## WR Category Codes

| Code | Full Name | Aliases Accepted |
|------|-----------|-----------------|
| `g` | Glitchless | `glitchless`, `gless` |
| `i` | Inbounds | `inbounds`, `inb` |
| `o` | Out of Bounds | `oob` |
| `nl` | NoSLA Legacy | `noslal` |
| `nu` | NoSLA Unrestricted | `noslau` |

---

*PortalBot — built for the Portal speedrunning community. If something breaks, check the bug tracker above first.*
