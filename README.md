<a href="https://c3-lang.org">
  <img src="https://c3-lang.org/assets/logo.svg" align="right" height="100" />
</a>

*A getopt-like parser for C3.*

# C3 Argparse

This library implements an argument parser that maps command-line inputs directly to variables using a compile-time macro interface.


## Specifications

- **Parser Compliance**: Supports short flag stacking (`-vh`), compact short option values (`-p8080`), long option equal-sign assignments (`--port=8080`), negative numbers, and the `--` end-of-options marker.
- **API Signatures**: Accepts options defined as pairs (`long, &dest`), triplets (`short, long, &dest` or `long, desc, &dest`), or quartets (`short, long, desc, &dest`).
- **Option Modifiers**:
  - `*` (Help): Registers a flag that formats and outputs the option documentation.
  - `?` (Optional Value):
    - For booleans: permits explicit override assignment (e.g., `--flag=false`).
    - For other types: prevents parsing failure if the value is omitted, leaving the target variable at its default value.
  - `+` (Incremental Counter): Increments integer targets on each occurrence (e.g., `-vvv` increments the counter by 3).
- **Environment Fallbacks**: Binds long options to environment variables (e.g., `"port=env:PORT"`). If the option is omitted from the command line, the parser attempts to read the specified environment variable.
- **Type Conversions**: Automatically converts inputs into basic types, enums (case-insensitive mapping), and fixed-size arrays.
- **Validation**: Supports custom validation using `fn void? (String)` callbacks.
- **Subcommands**: Stops parsing at the first positional argument and returns its index for subcommand routing.

## Usage Example

```c3
module main;

import std::io;
import std::os::argparse;

struct CLIOptions
{
	bool need_help;
	int verbose;
	int port;
	String host;
	bool quiet;
	bool dry_run;
}

CLIOptions cli_options = { .port = 8080, .host = "127.0.0.1" };

fn int main(String[] args)
{
	String[16] positional_files;

	sz? arg_last_parsed = argparse::@parse(
		args,
		"h*", "help*",          "Print help manual.",                       &cli_options.need_help,
		"v+", "verbose+",       "Increment verbosity level.",               &cli_options.verbose,
		"p",  "port=env:PORT",  "Server port (falls back to $PORT).",       &cli_options.port,
		      "host",           "Server host address.",                     &cli_options.host,
		"q?", "quiet?",         "Suppress output (accepts boolean value).", &cli_options.quiet,
		      "dry-run",                                                    &cli_options.dry_run,
		&positional_files
	);

	if (catch err = arg_last_parsed)
	{
		switch (err)
		{
			case argparse::HELP_REQUESTED:
				return 0;
			case argparse::MISSING_ARGUMENT:
				io::eprintfn("Error: Option '%s%s' requires an argument.", argparse::last_opt.len > 1 ? "--" : "-", argparse::last_opt);
				return 1;
			case argparse::ILLEGAL_OPTION:
				io::eprintfn("Error: Unrecognized option '%s%s'.", argparse::last_opt.len > 1 ? "--" : "-", argparse::last_opt);
				return 1;
			default:
				return 1;
		}
	}

	io::printfn("Starting server on %s:%d (verbosity: %d)", cli_options.host, cli_options.port, cli_options.verbose);
	return 0;
}
```

---

## Subcommand Parsing

To handle subcommands, parse global options without passing a positional array. The parser will automatically halt at the first positional argument (the subcommand name) and return its index, which you can use to route the remaining slice to another parser.

```c3
module main;
import std::io, std::os::argparse;

fn void handle_run(String[] sub_args)
{
	bool help, quiet;
	if (catch err = argparse::@parse(sub_args, 
		"h*", "help*",  "Show run help.",   &help,
		"q",  "quiet",  "Suppress output.", &quiet
	)) return;

	io::printfn("Subcommand 'run' executed (quiet: %b)", quiet);
}

fn void handle_build(String[] sub_args) { io::print("Subcommand 'build' executed.\n"); }

fn int main(String[] args)
{
	bool help;
	sz? root_result = argparse::@parse(args, "h*", "help*", "Show help.", &help);
	if (catch err = root_result) return err == argparse::HELP_REQUESTED ? 0 : 1;

	if (root_result >= args.len)
	{
		print_usage();
		return 0;
	}

	switch (args[root_result])
	{
		case "run":   handle_run(args[root_result..]);
		case "build": handle_build(args[root_result..]);
		default:      io::printfn("Unknown subcommand: '%s'", args[root_result]);
	}
	return 0;
}

fn void print_usage()
{
	io::print(
`Usage: my_app [global-options] <subcommand> [options]

Global Options:
  -h, --help    Show general help.

Subcommands:
  run           Run the server.
  build         Build the project.
`);
}
```

---

## Parsing Rules

**Value Assignment**: An option's value is resolved using the first applicable rule:
1. An inline assignment using `=` on long options: `--port=8080`
2. The remaining characters within the token for short options: `-p8080`
3. The next command-line argument, provided it does not begin with `-` (excluding negative numeric literals).

**Boolean Options**:
- **Without `?`**: Standard flags (such as `-q` or `--quiet`) assign `true` to the target and do not consume a subsequent argument.
- **With `?`**: The parser consumes the next argument if available. Values other than `"true"` or `"1"` evaluate to `false`.

**Non-Boolean Options**:
- **Without `?`**: Failure to provide a value returns a `MISSING_ARGUMENT` error.
- **With `?`**: Failure to provide a value skips assignment, preserving the variable's default value.

**Short Flag Stacking**:
- `-vvv` is processed as `-v -v -v`.
- Options requiring an argument must reside at the end of a stack (e.g., `-qp8080` maps to `-q` and `-p 8080`, whereas `-pq` maps to `-p "q"`).

**End of Options (`--`)**:
- Halts option processing. All subsequent tokens are handled as positional arguments.

## Installation

Copy `argparse.c3` into your project directory or standard library search path.
