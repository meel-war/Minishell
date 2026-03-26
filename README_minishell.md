# minishell

This project has been created as part of the 42 curriculum by [pribolzi] and [meel-war].

## Description

minishell is a fully functional UNIX shell written in C, modeled after **bash**. The project reimplements from scratch the core mechanisms of a shell: input reading with GNU Readline, lexical tokenization, quote and variable expansion, syntax validation, piping, redirections, heredocs, and a complete set of built-in commands.

The shell runs an interactive loop that displays a prompt based on the current working directory, reads user input, processes it through a multi-stage parsing pipeline, and executes the resulting commands — all while handling signals correctly in every execution context.

## Key Features

- Interactive prompt displaying the current working directory
- Full tokenization and multi-stage parsing pipeline (lexer → quote handler → expander → syntax checker)
- Single and double quote handling with correct expansion rules (`'...'` suppresses expansion, `"..."` allows `$VAR`)
- Environment variable expansion including `$?` for the last exit status
- Syntax error detection matching bash error messages (`unexpected token '|'`, `newline`, etc.)
- Pipelines of arbitrary length (`cmd1 | cmd2 | cmd3 | ...`)
- Input/output redirections: `<`, `>`, `>>`
- Heredoc (`<<`) with expansion support and correct `Ctrl+C` interruption
- Complete set of built-in commands
- Signal handling with distinct behavior for interactive mode, child processes, and heredoc input
- Command history via GNU Readline (`add_history`)
- `SHLVL` tracking across nested shell instances
- Proper exit status propagation through `$?`

## Instructions

### Prerequisites

- GCC or Clang compiler
- Make
- GNU Readline library (`libreadline`, `libncurses`)
- A UNIX-like operating system (Linux, macOS)

### Compilation

```
make
```

This will compile libft, then produce an executable named `minishell`.

### Execution

```
./minishell
```

No arguments are required. The shell starts an interactive session immediately.

## Built-in Commands

| Command | Description |
|---|---|
| `echo [-n]` | Print text to stdout; `-n` suppresses the trailing newline |
| `cd [path]` | Change directory; supports `~`, `-`, and no-argument (go to `$HOME`) |
| `pwd` | Print the current working directory |
| `export [VAR=value]` | Set or display environment variables; supports `+=` appending |
| `unset <VAR>` | Remove an environment variable |
| `env` | Print all exported environment variables |
| `history` | Print the command history for the current session |
| `exit [code]` | Exit the shell with an optional exit code |

## Project Structure

```
minishell/
├── includes/
│   └── minishell.h             # All structs, token types, and function prototypes
├── src/
│   ├── main.c                  # Entry point: init → env → shell loop → free
│   ├── minishell.c             # Shell loop, parsing hub, per-line lifecycle
│   ├── signals.c               # SIGINT/SIGQUIT handlers for each context
│   ├── signals2.c              # Child and command signal setup
│   └── sigint_status.c         # Syncs g_exit_status after SIGINT
├── parsing/
│   ├── parser.c                # Splits raw input into word/operator tokens
│   ├── parser_utils.c          # Word boundary and delimiter helpers
│   ├── token.c                 # Token type assignment (CMD, PIPE, REDIR, etc.)
│   ├── quotes.c                # Quote detection and token splitting
│   ├── quotes_utils.c          # Quote boundary helpers
│   ├── join_quotes.c           # Merges adjacent tokens separated by quotes
│   ├── join_quotes_utils.c     # Space tracking for join logic
│   ├── expand.c                # $VAR and $? expansion
│   ├── expand_utils.c          # Character-level expansion helpers
│   └── syntax.c                # Syntax validation (pipe, redir, heredoc rules)
├── exec/
│   ├── exec_hub.c              # Top-level execution: fork, heredoc, pipe dispatch
│   ├── exec.c                  # Command lookup (PATH resolution) and execve
│   ├── exec_fd.c               # File descriptor setup for redirections
│   ├── exec_utils.c            # Command extraction and process counting
│   ├── pipe.c                  # Pipeline forking and command sequencing
│   ├── pipe_utils.c            # Pipe fd management and wait logic
│   ├── heredoc.c               # Heredoc pipe creation and fd lifecycle
│   ├── heredoc_initiate.c      # Heredoc struct initialization per process
│   ├── heredoc_process.c       # Heredoc input reading loop
│   ├── heredoc_utils.c         # Delimiter matching and expansion in heredoc
│   └── error_message.c         # Access and directory error formatting
├── builtin/
│   ├── builtin.c               # Built-in dispatch (simple vs. piped context)
│   ├── builtin_utils.c         # Shared helpers for built-ins
│   ├── cd.c / cd2.c            # cd implementation with ~, -, and path handling
│   ├── echo.c                  # echo with -n flag
│   ├── env.c                   # env display and environment copy/init
│   ├── export.c / export2.c    # export with sorted output and += support
│   ├── export_utils.c          # Identifier validation and var search
│   ├── unset.c                 # unset with multi-arg support
│   ├── pwd.c                   # pwd using stored cur_dir
│   ├── ft_exit.c               # exit with numeric argument validation
│   └── history.c               # Session history list management
├── utils/
│   ├── free_all.c              # Full shell struct cleanup and exit
│   ├── ft_add_env_var.c        # Dynamic environment array expansion
│   ├── ft_find_env_var.c       # Variable index search in env array
│   ├── ft_get_env.c            # Value retrieval by variable name
│   └── ft_update_env_var.c     # In-place environment variable update
├── libft/                      # Custom C standard library
└── Makefile
```

## Technical Choices

### Multi-Stage Parsing Pipeline

Rather than parsing input in a single pass, the shell processes each line through a sequential pipeline of transformations. Raw input is first split into tokens by the parser, then quotes are identified and their boundaries resolved, adjacent tokens produced by quote splitting are merged, variable expansion is applied, token types are assigned (CMD, PIPE, REDIR_IN, HEREDOC, etc.), and finally the syntax validator walks the token list to catch illegal operator sequences. This staged design keeps each transformation focused and makes the parsing logic straightforward to debug and extend.

### Quote Handling

Single and double quotes are resolved at the token level before expansion. A token containing a quoted segment is split into up to three sub-tokens (before, inside, after). Single-quoted tokens are typed `S_QUOTE` and skipped entirely during expansion. Double-quoted tokens are typed `D_QUOTE` and expanded normally. After expansion, the `join_quote` pass re-merges adjacent tokens that were originally part of the same word, using the `last_space` flag on each token to determine whether a space was present between them in the original input.

### Heredoc Implementation

Each heredoc is pre-processed before execution begins. A linked list of `t_heredoc` structs is built by walking the token list and counting `<<` operators per process. For each heredoc, a pipe is created and a child process reads from `readline` until the delimiter line is matched. The read end of the pipe is then wired as `stdin` for the target command at execution time. SIGINT during heredoc input is handled with a dedicated signal configuration that prints `^C` and sets the exit status to 1 without affecting the parent's readline state.

### Signal Handling

Three distinct signal configurations are applied depending on the execution context. In interactive mode, SIGINT reprints a new prompt by calling `rl_replace_line` and `rl_redisplay`, while SIGQUIT is ignored. In child processes executing external commands, both signals are reset to their default handlers so that `Ctrl+C` and `Ctrl+\` terminate the child normally. During heredoc input, SIGINT prints `^C` and interrupts the read loop without corrupting the parent shell's prompt.

### Built-in Execution Context

Built-in commands that modify shell state (`cd`, `export`, `unset`, `exit`, `history`) must run in the parent process, not in a fork. The dispatcher checks whether the current command list contains exactly one command with no pipes before executing it directly in the parent. When a built-in appears inside a pipeline, it is forked like any external command, since its state changes are scoped to that child and discarded after `waitpid`.

## Resources

- [GNU Bash Reference Manual](https://www.gnu.org/software/bash/manual/bash.html) — Reference for bash-compatible behavior
- [GNU Readline Library](https://tiswww.case.edu/php/chet/readline/rltop.html) — API for interactive line editing and history
- [Advanced Programming in the UNIX Environment](https://apuebook.com/) — Reference for fork, exec, pipes, and signal handling
- [sigaction man page](https://man7.org/linux/man-pages/man2/sigaction.2.html) — Portable signal handler registration

## Authors

- **pribolzi** and **meel-war**
