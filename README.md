# Fumadocs Setup Skills

Step-by-step guides for setting up a complete Fumadocs documentation site.

## 📖 Files

- **`[File 1] Docs.md`** — Core documentation setup: sidebar tabs, section colors, navbar icon
- **`[File 2] Home.md`** — Landing page with hero, features, CTAs
- **`[File 3] Search.md`** — Orama search integration with custom UI
- **`[File 4] Banner.md`** — Announcement banner with gradient text
- **`[File 5] Blog.md`** — Full blog system with tags, authors, TOC
- **`[File 6] Layout.md`** — Multiple layout presets (Docs, Spacious, Glass, Notebook) with unified configuration

## ⚠️ Warning

The following files require manual terminal command entry:

- **`[File 2] Home.md`** — Step 1 requires running `pnpm dlx @fumadocs/cli@latest customize`
- **`[File 5] Blog.md`** — Step 1 requires running `pnpm dlx @fumadocs/cli@latest add blog`
- **`[File 6] Layout.md`** — Step 1 requires running `pnpm dlx @fumadocs/cli@latest customize`

**When using with AI agents:** The AI will prompt you to run these commands manually. Confirm completion before the agent proceeds with subsequent steps.

**Note:** `[File 5] Blog.md` is particularly long and may require extra attention when running with an agent, as the agent can sometimes get confused with the number of steps involved.

## 🚀 Usage

Feed these files to an AI agent like Claude using:

```
Read `<skill>`, and apply it to the `<dir>` in order and in full without improvising. By saying "in order", it means you MUST apply the skill strictly step-by-step. By saying "without improvising", it means you should only copy and paste the code from the skill markdown file without thinking and modifying. You may need to read the target file (if any) first to find the difference between the target file and the code provided by the skill markdown file so that you can make the changes more accurately.
```

**Example:**
```
Read "/path/to/skill/[File 1] Docs.md", and apply it to the "template" folder in order and in full without improvising. By saying "in order", it means you MUST apply the skill strictly step-by-step. By saying "without improvising", it means you should only copy and paste the code from the skill file without thinking and modifying. You may need to read the target file (if any) first to find the difference between the target file and the code provided by the skill markdown file so that you can make the changes more accurately.
```

Each file contains sequential tasks that build upon previous ones. Apply them in numeric order for a complete setup.
