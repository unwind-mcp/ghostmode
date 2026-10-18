# Ghost Mode

**See what your AI agent would do, without letting it do anything.**

> Standalone package extracted from the [UNWIND monorepo](https://github.com/unwind-mcp/unwind) (250+ commits, 1,859 tests). Full development history lives there.

Ghost Mode is a safe first step before giving an AI agent real authority.

It's a zero-config dry-run layer for AI agents. It intercepts every write and lets reads through. Think of it as a flight simulator for AI agents — nothing gets modified, but you see everything the agent tried to do.

Requires Python 3.9+.

## Quick Start

### MCP Client Users (Claude Desktop, Cursor, etc.)

**1. Install it.**

```bash
pipx install ghostmode
```

No pipx? Use a small virtual environment instead. On macOS with Homebrew Python, a plain `pip install` is refused with "externally-managed-environment".

```bash
python3 -m venv ~/.ghostmode && ~/.ghostmode/bin/pip install ghostmode
```

**2. Find its full path** with `which ghostmode`. If you used the virtual environment, it's `~/.ghostmode/bin/ghostmode`.

**3. Put Ghost Mode in front of the server in your client's config.** In Claude Desktop, go to Settings → Developer → Edit Config. In Cursor, edit `~/.cursor/mcp.json`. Use full paths, because these apps don't see your shell's PATH:

```json
{
  "mcpServers": {
    "documents-ghost": {
      "command": "/full/path/to/ghostmode",
      "args": ["--", "npx", "-y", "@modelcontextprotocol/server-filesystem", "/Users/you/Documents"]
    }
  }
}
```

Restart the app and give your agent a task. When the session ends, a summary is written to `ghostmode-summary.txt` in your working directory (or your home directory if that isn't writable) and printed to stderr.

To try it in a terminal first, run `ghostmode -v -- npx -y @modelcontextprotocol/server-filesystem ~/Documents`. It waits for an MCP client on stdin; press Ctrl+C to stop and see the summary.

### OpenClaw Users

There is no separate Ghost Mode plugin for OpenClaw. Ghost Mode for OpenClaw comes with UNWIND's experimental OpenClaw adapter:

```bash
pip install unwind-mcp
git clone https://github.com/unwind-mcp/unwind.git
openclaw plugins install ./unwind/openclaw-adapter
# Restart the OpenClaw gateway, then:
unwind ghost on
```

## What it does

Ghost Mode intercepts state-modifying tool calls — file writes, deletions, emails, API posts, shell commands — and returns fake success responses. The agent thinks it completed the action. Your filesystem, email, and APIs are untouched.

A shadow virtual filesystem keeps the agent consistent: if it "writes" a file and then reads it back, Ghost Mode serves the written content from memory. The agent doesn't know it's in a sandbox.

## Example output

```
=======================================================
  👻 GHOST MODE SESSION SUMMARY
=======================================================
  Duration:     112.7s
  Total events: 10
  Intercepted:  4 writes blocked
  Passed:       2 reads forwarded
  Shadow reads: 4 served from VFS

  Files the agent tried to write:
    📝 /home/user/Documents/notes.txt
    📝 /home/user/Documents/archive/draft.txt
    📝 /home/user/Documents/summary.md

  Directories the agent tried to create:
    📁 /home/user/Documents/archive

  Files the agent tried to delete or move away:
    🗑️  /home/user/Documents/draft.txt

  🚫 4 write(s) were blocked. Nothing was modified.

  Full timeline:
  ✅ 18:53:06  [PASSED]  list_directory → /home/user/Documents
  ✅ 18:53:07  [PASSED]  read_text_file → /home/user/Documents/notes.txt
  🚫 18:54:03  [BLOCKED]  edit_file → /home/user/Documents/notes.txt
  🚫 18:54:03  [BLOCKED]  create_directory → /home/user/Documents/archive
  🚫 18:54:08  [BLOCKED]  move_file → /home/user/Documents/draft.txt
  👻 18:54:13  [SHADOW]  directory_tree → /home/user/Documents
  🚫 18:54:26  [BLOCKED]  write_file → /home/user/Documents/summary.md
  👻 18:54:27  [SHADOW]  read_text_file → /home/user/Documents/notes.txt
  👻 18:54:34  [SHADOW]  read_text_file → /home/user/Documents/summary.md
  👻 18:54:34  [SHADOW]  directory_tree → /home/user/Documents
=======================================================
```

This is a real session: an agent fixed a typo, archived a draft and wrote a summary. It saw every change it made, but nothing on disk changed.

## Use cases

**Testing untrusted tools.** Install a new MCP server and see what it actually does before giving it real access.

**Reviewing agent behaviour.** Let your agent plan and "execute" a complex task, then review the timeline before running it for real.

**Demos and training.** Show stakeholders what an AI agent workflow looks like without risking real data.

**Compliance dry runs.** Verify that an agent workflow stays within policy boundaries before going live.

## What Ghost Mode is not

Ghost Mode does not enforce policy or provide rollback. It is a visibility tool, not a security system. If you need enforcement, audit trails, or recovery, that's what [UNWIND](https://github.com/unwind-mcp/unwind) is for.

## CLI options (MCP proxy mode)

```
ghostmode -- <upstream command>        Basic usage
ghostmode -v -- <command>              Verbose: log every action to stderr
ghostmode --export log.json -- <cmd>   Export session log on exit
ghostmode --also-block my_tool -- ...  Add custom tool names to intercept
ghostmode --allow my_tool -- ...       Pass an unknown read-only tool through
```

## How it knows what to block

Ghost Mode recognises 50+ common MCP tool names as state-modifying (filesystem writes, email sends, API posts, shell commands, database mutations, git operations). It also uses prefix heuristics: any tool starting with `create_`, `delete_`, `send_`, `write_`, `execute_`, etc. gets intercepted. Anything else is only passed through if it is known to be read-only: either the server marks it with the MCP `readOnlyHint` annotation, or its name starts with `read_`, `get_`, `list_`, `search_`, `fetch_` and similar. **Unknown tools are blocked by default**, and listed in the session summary.

- If a read-only tool is being blocked, use `--allow tool_name` to pass it through.
- If a tool slips through, use `--also-block tool_name` to add it.

## Keeping the agent consistent

For filesystem servers (including `@modelcontextprotocol/server-filesystem`), the shadow VFS keeps everything the agent sees in line with what it "did": edited files read back with the edit applied, moved files appear at their new path and are gone from the old one, and new files and directories show up in `list_directory`, `directory_tree`, `search_files`, `get_file_info` and `read_multiple_files`. Real file contents are fetched through the upstream server, so its own access rules still apply.

## Zero dependencies

Ghost Mode has no external dependencies. It uses only Python 3.9+ stdlib (asyncio, json, pathlib). It installs in under a second.

## Known limitations

**Shell commands can't see ghost-written files.** If the agent writes a file through Ghost Mode and then tries to run it via `bash_exec` or `shell_exec`, the shell will fail because the file only exists in the in-memory shadow VFS, not on disk. Ghost Mode works best for tool-call-level dry runs, not multi-step build/execute workflows.

**Moved directories appear empty.** Moving a real directory shows it at the new path, but its contents are not mirrored there.

**Session summary in background clients.** Claude Desktop and Cursor run MCP servers in the background and may not surface stderr output. Ghost Mode automatically writes a `ghostmode-summary.txt` file to your working directory on exit (falling back to your home directory), whether the client disconnects or stops the server with a signal. You can also use `--export log.json` for a machine-readable session log.

## Want more?

Ghost Mode shows you what happened. **UNWIND** lets you control it:

- 15-stage enforcement pipeline (path jail, SSRF shield, DLP, canary honeypots)
- Smart snapshots with one-command rollback
- CRAFT tamper-evident audit chain
- Real-time trust light dashboard
- Detects encoded/obfuscated data exfiltration attempts

```bash
pip install unwind-mcp
```

[github.com/unwind-mcp/unwind](https://github.com/unwind-mcp/unwind)

## License

MIT
