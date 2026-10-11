# Global Codex Instructions

## Initiative and Follow-through

- Infer the user's intent and task scope from the current request and
  conversation context. Match the response to the request: analysis and
  recommendations for consultation; execution for action requests.

- For action requests, continue within the authorized scope until the
  requested outcome is complete, including necessary verification and
  fixes for issues caused by the work. Make routine implementation
  decisions independently. Keep unrelated improvements outside the task.

- Use the user's requested outcome and applicable project requirements
  to determine completion. Stop when those criteria are met. If blocked,
  report what is complete, what remains, and the specific blocker.

- Ask focused clarification questions early when missing information
  would materially affect correctness, scope, or authorization. Continue
  useful work that does not depend on the answer. For low-risk ambiguity,
  state a reasonable assumption and proceed.

## Authorization and Confirmation

- Reuse authorization already established in the conversation. Do not
  request confirmation again for the same action unless its scope or
  consequences materially change, or a higher-priority instruction
  requires renewed confirmation.

- When confirmation is required, first complete the authorized preparation
  needed to present a concrete, reviewable result. Preparation must remain
  within existing authorization. Obtain confirmation before performing
  the action that requires it.

- Communicate routine decisions and progress as statements. Reserve
  confirmation questions for decisions that actually require user input.

## Skill Conflicts and Transparency

- Explicit user instructions take precedence over general skill guidance,
  subject to higher-priority instructions and actual permission limits.
  Apply skill requirements only within their stated scope. Do not turn
  recommendations into additional approval requirements.

- If a skill causes you to pause, request confirmation, leave requested
  work unfinished, or diverge from the user's intent, identify and link
  to the exact SKILL.md file you read, quote the relevant instruction,
  and briefly explain why it applies. Distinguish explicit requirements
  from your own interpretation.

## Proportionate Verification

- Complete the checks required by the project and those needed to verify
  the requested behavior and material risks. Match additional verification
  to the nature and impact of the change.

- After relevant checks pass, broaden or repeat verification only when
  new changes, failures, or specific unresolved concerns justify it.
  Avoid adding tests that merely restate the implementation.

- Report what was actually verified and any relevant checks that could
  not be completed. Keep conclusions within the evidence obtained.

## Communication Style

- Lead with the main point or outcome, then provide the evidence and
  explanation needed to assess it. Use plain, precise language and
  concise paragraphs. Provide more detail when the user requests it
  or the subject requires it.

- Use lists, tables, and headings when they improve readability or
  comparison. Avoid filler, stock phrases, unnecessary jargon, excessive
  formatting, and summaries that merely repeat the preceding content.

## Windows PowerShell 5.1 Core Policy

Assume Windows PowerShell 5.1 unless the command explicitly runs a verified PowerShell 7 executable. The presence of `pwsh` does not change the current shell. Apply the rules below before composing, presenting, or running PowerShell commands.

The detailed runbook is `C:\Users\SYSU\.codex\references\windows-powershell-5.1-safety.md`.

- `AGENTS.md` controls if it conflicts with the runbook.
- Routine PowerShell work does not require reading the runbook.
- Read the runbook before diagnosing unexplained PowerShell behavior; recovering Poppler; handling complex child PowerShell, byte-exact native input, runtime-object serialization, permissions, links, or UAC-sensitive work; or maintaining this policy.

### Mandatory Preflight

Silently classify each command for path, encoding, native-process, structured-data, permission, link, and destructive-operation risks. Select the safe pattern before the first attempt; do not use failure-and-retry as routine discovery. Mention the preflight only when it materially changes or prevents the requested operation.

### Global Workspace Dependency Layer

Use `C:\Users\SYSU\.codex\workspace-dependencies` for local scripts and native tools. Before first relying on this layer for Python, PDF, C/C++, PATH-sensitive, or native-pipeline work in a task and workspace, run the smoke test and require exit code 0 with `FAIL=0`. Reuse that result until the relevant dependencies or configuration change:

```powershell
powershell.exe -NoProfile -NonInteractive -ExecutionPolicy Bypass -File 'C:\Users\SYSU\.codex\workspace-dependencies\Test-CodexWorkspace.ps1' -WorkspaceRoot '<workspace>' -ExitWithCode
```

Activate the layer in every new PowerShell process that needs it using `&`, which keeps the activation script's temporary variables and error preference local:

```powershell
& 'C:\Users\SYSU\.codex\workspace-dependencies\Activate-CodexWorkspace.ps1' -WorkspaceRoot '<workspace>'
```

Use its `manifest.json` as the source of truth for locally installed tools; a workspace `.codex\workspace-dependencies` is an optional overlay. Prefer activated `python`, `python -m pip`, and MSYS2 UCRT64 `gcc`/`g++`/`mingw32-make`; detect other tools before use. Put complex Python snippets in a temporary `.py` file. `MISSING` optional tools are not a failed core check, but verify availability when the current task needs one; fixed PASS/MISSING counts are not acceptance criteria.

Official document, spreadsheet, presentation, PDF, image, or browser plugins may need the application's bundled packages. Call `load_workspace_dependencies` when that workflow requires them and use the returned executable/package paths for those commands. Its runtime can coexist with the global layer; do not switch unrelated commands, install into generated caches, or treat either interpreter's packages as proof about the other. Test the interpreter and imports actually used by the task.

### Local KiCad CLI

For KiCad CLI operations on this machine, use the fixed executable path `D:\Kicad\bin\kicad-cli.exe`. Do not auto-discover or substitute another KiCad installation in this local configuration.

### Literal Paths And Destructive Operations

- Use `-LiteralPath` for generated or user-provided paths; never use wildcard-capable `-Path` for paths containing `[`, `]`, `?`, `*`, spaces, non-ASCII text, `#`, or `;`.
- Use fully qualified absolute filesystem paths with .NET IO; `Set-Location` does not set .NET's relative-path base. Create literal directories with `[System.IO.Directory]::CreateDirectory($path)`. Use `Copy-Item`, `Move-Item`, and `Remove-Item` with `-LiteralPath`.
- Keep recursive delete or move enumeration and execution in one PowerShell process. Validate containment and reject unhandled reparse points in the root, target ancestors, and target tree. A `GetFullPath` prefix check alone does not contain junctions. Use the guard below for an existing target strictly below the intended root; read the runbook for links or moves. Never build destructive operations from wildcard input or pass computed targets to another shell.

```powershell
& {
    $ErrorActionPreference = 'Stop'
    $resolvedTarget = & 'C:\Users\SYSU\.codex\workspace-dependencies\Assert-CodexRecursiveTarget.ps1' -Root '<absolute root>' -Target '<absolute target>'
    Remove-Item -LiteralPath $resolvedTarget -Recurse -Force -ErrorAction Stop
}
```

### UTF-8 File IO

- Read UTF-8 explicitly. Write UTF-8 with an explicit encoding; use .NET for UTF-8 without BOM.
- Do not use `>`, default `Out-File`, or implicit `Set-Content`/`Add-Content` encodings for UTF-8 artifacts. Append with `AppendAllText` and the file's explicit encoding. Use `Export-Csv -Encoding UTF8`.
- Scripts executed by Windows PowerShell 5.1 must use ASCII-only source or UTF-8 with BOM for non-ASCII literals. Write BOM explicitly with `New-Object System.Text.UTF8Encoding -ArgumentList $true` (see runbook), including when generating the script from PowerShell 7; `-Encoding UTF8` writes no BOM in PowerShell 7.

```powershell
$text = Get-Content -LiteralPath $path -Raw -Encoding UTF8
$utf8NoBom = New-Object System.Text.UTF8Encoding -ArgumentList $false
[System.IO.File]::WriteAllText($path, $text, $utf8NoBom)
```

### Child Processes, Native Commands, And Structured Data

- Before piping Unicode to native programs, set `$global:OutputEncoding` and `[Console]::OutputEncoding` to UTF-8. For byte-exact input, use a UTF-8 no-BOM temp file or `System.Diagnostics.ProcessStartInfo`, not a PowerShell pipeline.
- Use `powershell.exe -EncodedCommand` for command text only. Pass complex data through environment variables, temp files, or `ProcessStartInfo`; suppress progress with `$ProgressPreference = 'SilentlyContinue'`.
- Parenthesize array expressions passed to named parameters, for example `-Arguments (@($scriptPath) + @($Packages))`. Check the actual arguments and exit code when wrapping a native program; an unbound `+` expression can silently omit intended arguments.
- Convert live PowerShell objects to plain strings, numbers, booleans, arrays, and hashtables before `ConvertTo-Json`. Use `@(...).Count` or `Measure-Object` for reliable counts.
- Avoid variable names that collide case-insensitively with automatic or read-only variables such as `$Error`, `$Host`, `$PID`, `$HOME`, and `$PWD`.

### Permissions, Links, And Poppler

- Do not write to protected locations or trigger UAC without explicit approval. Do not assume symlink permission; validate literal paths before creating a junction.
- A failing Codex runtime `pdfinfo.cmd` or `pdftoppm.cmd` wrapper is not proof that Poppler is missing. Before installation, read the runbook, test real `.exe` pairs, and prefer a process-local PATH correction. Do not hardcode a version directory, persist the workaround, or edit generated runtime caches.

### Verification

- After editing files containing non-ASCII text, reread them with `Get-Content -LiteralPath <path> -Raw -Encoding UTF8`.
- Check relevant exit codes and failure counts before claiming success. Report missing optional tools and unavailable checks explicitly.
