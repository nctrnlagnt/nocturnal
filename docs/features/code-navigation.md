# Code Navigation and Refactoring

## What

Syntax-aware code navigation and refactoring that understands code structure,
not just text patterns. Parses files into an abstract syntax tree first, then
operates on the actual code structure — function boundaries, symbol scopes, call
relationships.

Supported languages: C, C++, Python, JavaScript, TypeScript, Rust, Go, Zig,
Perl, Bash, Ruby, PHP, Common Lisp, Emacs Lisp.

## How it works

Standard find-and-replace operates on raw text, which leads to false matches
(the same word in a comment, string, or different scope) and broken code
(indentation mismatches, partial replacements). Code navigation parses files
into a structured representation first, then operates on
the actual code structure — function boundaries, symbol scopes, call
relationships.

This means:

- Renaming a function only renames the actual function and its call sites, not
  comments that happen to mention it.
- Moving a function between files preserves its exact signature and body.
- Finding references finds actual usages, not string matches.

## Capabilities

| Operation | Description |
|-----------|-------------|
| List definitions | Enumerate functions, methods, classes with signatures and doc comments |
| Show definitions | Get full source code for named definitions |
| Find references | Find all usages of a symbol across files |
| Rename symbols | Rename variables, functions, classes across multiple files |
| Move functions | Transfer functions between files while preserving structure |
| Replace functions | Replace entire function definitions (signature + body + docs) |
| Delete functions | Remove functions from a file cleanly |
| Insert functions | Add new function definitions at a specific position |
| Build call trees | Analyze function call relationships |

## Usage

The agent invokes these operations automatically when working with source code.
You don't need to call them directly — just ask the agent to perform structural
changes.

Examples of what you can ask:

> "Move the `parse_config` function from `src/main.rs` to `src/config.rs`."

> "Rename `process_request` to `handle_request` across the entire codebase."

> "Show me all functions defined in the `src/api/` directory."

> "Find every place `connect_timeout` is used."

> "Replace the `validate_input` function with this new version..."

The agent will use syntax-aware operations by default, falling back to text
editing only when structural tools can't handle the case.
