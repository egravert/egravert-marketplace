# Agentic Practices Installer

Install agentic engineering practices into the current project's CLAUDE.md as permanent coding standards.

## Procedure

Follow these steps exactly when this skill is invoked:

### Step 1: Identify available languages

List the files in this skill's `references/` directory matching the pattern `agentic-*-practices.md`. Extract the language name from each filename (e.g., `agentic-go-practices.md` → "Go"). These are the supported languages.

### Step 2: Ask the user which language to install

Present the available languages and ask the user to choose one. If only one language is available, still confirm it with the user. Do not auto-detect — the user decides.

If the user requests a language that has no practices file, tell them which languages are currently supported and stop.

### Step 3: Read the project's CLAUDE.md

Look for `CLAUDE.md` in the project root. If it does not exist, create a new one.

### Step 4: Check for existing installation

Search the CLAUDE.md for the marker:
```
<!-- BEGIN agentic-practices vX.Y.Z -->
```

- **If the marker exists and the version matches `1.0.0`:** Tell the user the practices are already installed and up to date. Stop.
- **If the marker exists but the version is older:** Replace everything between the `BEGIN` and `END` markers (inclusive) with the new version. Tell the user the practices were updated from vX.Y.Z to v1.0.0.
- **If no marker exists:** Proceed to installation.

### Step 5: Install the practices

Read the contents of `references/agentic-<language>-practices.md` for the chosen language.

Append the following block to the end of CLAUDE.md:

```
<!-- BEGIN agentic-practices v1.0.0 (<language>) -->

[contents of the language-specific practices file]

<!-- END agentic-practices -->
```

Replace `<language>` with the chosen language name (e.g., "go", "python").

### Step 6: Confirm

Tell the user:
- Which language practices were installed
- The version installed (1.0.0)
- That the practices will apply to all future Claude Code sessions in this project
- That they can re-run this skill to update if a newer version is available
