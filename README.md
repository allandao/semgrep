
# Semgrep Contributions

With motivatation from the DevOps course at KTH. First non-trivial proposed contribution to a major project I've used in the past.

Main Repo:
https://github.com/returntocorp/semgrep/
Forked Repo:
https://github.com/allandao/semgrep
Pull Request (username allandao):
https://github.com/returntocorp/semgrep/pull/7850

Other (small contributions) from open source exploration while investigating potential issues to fix:
https://github.com/actions/runner-images/pull/7438 - merged PR on major project (for a typo)
https://github.com/Azure/azure-powershell/issues/20817 - clear out issue backlog

## Context

I wanted to contribute to Semgrep by tackling these issues to gain a better understanding of the codebase:

- Semgrep didn't scan and returns zero exit code if you use space character in config -
https://github.com/returntocorp/semgrep/issues/7630
- --error should explain why the exit status is nonzero - https://github.com/returntocorp/semgrep/issues/7661

These issues were related to the semgrep ci command, which adds a layer of complexity and is unrelated to my goal of contributing to the basic semgrep cli commands.

While investigating these issues and the codebase to better understand Sempgrep, I started to notice that some summary messages had "files" consistently plural, while others did not. I wanted to investigate the reason why.

## Examples

### Correct
```
┌─────────────┐
│ Scan Status │
└─────────────┘
  Scanning 1 file tracked by git with 1073 Code rules:
┌────────────────┐
│ 1 Code Finding │
└────────────────┘
┌─────────────────┐
│ 2 Code Findings │
└─────────────────┘
Ran 1073 rules on 1 file: 0 findings.

```

### Incorrect
```
┌──────────────┐
│ Scan Summary │
└──────────────┘
Some files were skipped or only partially analyzed.
  Partially scanned: 1 files only partially analyzed due to parsing or internal Semgrep errors
  Scan skipped: 1 files matching .semgrepignore patterns

```

-----

## Investigation
With this comparison, I could focus on the differences on "Scan Status" and "Code Finding(s)" outputs compared to "Scan Summary" outputs. 

Searching for Partially scanned: and Scan skipped: revealed that this output is directly from OCaml code. The file of interest is [Summary_report.ml](https://github.com/returntocorp/semgrep/blob/d1103fe8aa9dfee140bb4214b3f9ebab1d2a4d4c/src/osemgrep/reporting/Summary_report.ml#L62) and the relevant code: 

```ocaml
let opt_msg msg = function
    | [] -> None
    | xs -> Some (string_of_int (List.length xs) ^ " " ^ msg)
  in
  let out_skipped =
    let mb = string_of_int Stdlib.(max_target_bytes / 1000 / 1000) in
    Common.map_filter Fun.id
      [
        opt_msg "files not matching --include patterns" include_ignored;
        opt_msg "files matching --exclude patterns" exclude_ignored;
        opt_msg ("files larger than " ^ mb ^ " MB") file_size_ignored;
        opt_msg "files matching .semgrepignore patterns" semgrep_ignored;
        (if legacy then None else opt_msg "other files ignored" other_ignored);
      ]
  in
  let out_partial =
    opt_msg
      "files only partially analyzed due to a parsing or internal Semgrep error"
      errors
  in
  match (out_skipped, out_partial) with
  | [], None -> ()
  | xs, parts ->
      (* TODO if limited_fragments:
              for fragment in limited_fragments:
                  message += f"\n  {fragment}" *)
      Fmt.pf ppf "Some files were skipped or only partially analyzed.@.";
      Option.iter (fun txt -> Fmt.pf ppf "  Partially scanned: %s@." txt) parts;
      (match xs with
      | [] -> ()
      | xs ->
          Fmt.pf ppf "  Scan skipped: %s@." (String.concat ", " xs);
          Fmt.pf ppf
            "  For a full list of skipped files, run semgrep with the \
             --verbose flag.@.");
      Fmt.pf ppf "@."
```

Great, we found the location of the OCaml code responsible for the output. So why do we have passing examples? I took "tracked by git" as inspiration for correct output, revealing the following items of interest:
- [sempgrep-main.py](https://github.com/returntocorp/semgrep/blob/d1103fe8aa9dfee140bb4214b3f9ebab1d2a4d4c/cli/src/semgrep/semgrep_main.py#L156)
- [Status_report.ml](https://github.com/returntocorp/semgrep/blob/d1103fe8aa9dfee140bb4214b3f9ebab1d2a4d4c/src/osemgrep/reporting/Status_report.ml#L15)

Here, there is a connection between the Python wrapper and Semgrep core. In particular, notice
```py
def print_summary_line(
    # ... 
    # From sepgrep-main.py
    # ...
    # Aha, noticed unit_str() call!
    # from semgrep.util import unit_str
    summary_line = f"Scanning {unit_str(file_count, 'file')}"
```

Looking into [util.py](https://github.com/returntocorp/semgrep/blob/d1103fe8aa9dfee140bb4214b3f9ebab1d2a4d4c/cli/src/semgrep/util.py#L182), we find that there unit_str is already conveniently created to handle plurality for words such as file in output messages.
```py
def unit_str(count: int, unit: str, pad: bool = False) -> str:
    if count != 1:
        unit += "s"
    elif pad:
        unit += " "

    return f"{count} {unit}"
```

Back to [Summary_report.ml](https://github.com/returntocorp/semgrep/blob/d1103fe8aa9dfee140bb4214b3f9ebab1d2a4d4c/src/osemgrep/reporting/Summary_report.ml#L62). It seems that the output messages from this code is what is shown the in the terminal. Hence, changes should appear here.

-----

## Changes

[Summary_report.ml](https://github.com/returntocorp/semgrep/blob/d1103fe8aa9dfee140bb4214b3f9ebab1d2a4d4c/src/osemgrep/reporting/Summary_report.ml#L62)

```ocaml
(* Code changes in Summary_report.ml*)
(* Change the structure of output messages to consider file plurality *)
| xs ->
      let file_count = List.length xs in
      let file_str = if file_count = 1 then "file" else "files" in
      (* Notice spacing handled here already! *)
      Some (string_of_int file_count ^ " " ^ file_str ^ " " ^ msg)

(* ... *)

let out_partial =
    (* Remove preceding word "files" *)
    opt_msg
      "only partially analyzed due to a parsing or internal Semgrep error"
      errors
  in 
  (* ... *)
```

```ocaml
(* Tricky change, matter of wording. Original... *)
(* 1 other files ignored *)
(if legacy then None else opt_msg "other files ignored" other_ignored);

(* Change *)
(* 1 file also ignored but not already mentioned *)
(if legacy then None else opt_msg "also ignored but not already mentioned" other_ignored);
```

See the [original code with notes](notes-original-Summary_report.ml) and [new code with the edits above](new.ml).

----- 

## Concerns

1. Automatic scanning/parsing may break in some instances

Users or particular features that scan for the phrase "files ... " in output messages may now fail in cases of an example output message such as "1 file ignored not matching --include patterns" as we now present both "file" and "files" as things to search for. However, given that the rest of the summary already had correct plurality, scanning code should have already been robust enough the handle the edge case of 1 file instances.

2. Significant amounts of snapshots become out of date

Search queries such as this one will reveal a large sample of snapshots:
https://github.com/search?q=repo%3Areturntocorp%2Fsemgrep+Partially+scanned%3A&type=code

This change may necessitate that these snapshots change given that the plurality of the word "files" is incorrect.

-----

### Next Steps
I want to revisit the issues posted by other users in [Context](#context).

### Links and Notes
https://semgrep.dev/docs/contributing/contributing-code/
https://semgrep.dev/docs/cli-reference/#exit-codes
https://semgrep.dev/docs/contributing/semgrep-core-contributing/
https://semgrep.dev/docs/contributing/semgrep-contributing/


Exit Codes Match - semgrep-core
- https://semgrep.dev/docs/cli-reference/#exit-codes
- https://github.com/returntocorp/semgrep/blob/d1103fe8aa9dfee140bb4214b3f9ebab1d2a4d4c/src/osemgrep/core/Exit_code.ml#L4

## Further Contributions
Semgrep PR - Semgrep CI Command Run Error Behavior

Two branches of notice on allandao/devops-course:
- default branch 2023 - essay branch
- madao - open source contribution branch

Pull Request Template
https://github.com/returntocorp/semgrep/blob/a2fba21f8fda9f24c9473640a2880c2ad607aeaa/.github/pull_request_template.md

### CLI Investigation
https://github.com/returntocorp/semgrep/blob/develop/cli/src/semgrep/commands/
ci.py
``` -- config ```
https://github.com/returntocorp/semgrep/blob/34cd3b73e2d17ed5a312b7f524c084e333555193/src/osemgrep/cli_scan/Rule_fetching.ml
https://github.com/returntocorp/semgrep/blob/fd5a9e9ec6754fc197e66d7a3e56c65fba83b8d8/cli/src/semgrep/config_resolver.py
```ocaml
| Error msg ->
                   (* was raise Semgrep_error, but equivalent to abort now *)
                   Error.abort
                     (spf "Failed to download config from %s: %s"
                        (Uri.to_string url) msg)
```
```py
__main__.py > 
from semgrep.cli import cli
def main() -> int:
    cli()  # thus we look into cli.lpy
    return 0

cli.py
from semgrep.commands.ci import ci
cli.add_command(ci)

commands/ci.py 
To run multiple rule files simultaneously, use --config before every YAML, URL, or Semgrep registry entry name. For example `semgrep --config p/python --config myrules/myrule.yaml`
@click.option(
    "--config",
    "-c",
    "-f",
    multiple=True,
    envvar="SEMGREP_RULES",
Ideas: 
- SEMGREP_RULES
- @handle_command_errors
def ci(
configs=config
autofix=scan_handler.autofix if scan_handler else False,

except SemgrepError as e:
        output_handler.handle_semgrep_errors([e])
        output_handler.output({}, all_targets=set(), filtered_rules=[])
        logger.info(f"Encountered error when running rules: {e}")
        if isinstance(e, SemgrepError):
            exit_code = e.code
        else:
            exit_code = FATAL_EXIT_CODE
        if scan_handler:
            scan_handler.report_failure(exit_code)
        sys.exit(exit_code)
```
Flag --no-suppress-errors, a red herring:
By default, Semgrep may suppress certain types of errors to avoid overwhelming the user with excessive output. For verbose output, not for program stalling.

Semgrep_error vs Error.Abort

### Understanding the Issue

#### Command Permutations - all exit code 0 despite failure
Show error output in terminal
```
semgrep ci --config="p/default" --config=" p/javascript" --sarif --output="output.sarif" --metrics off --force-color 
semgrep --config=" p/auto" "test.js"
semgrep ci --config p/default
semgrep ci --config "p/default"
semgrep ci --config="p/default"
```
Send error output to file
```
semgrep ci --config="p/default" --config=" p/python"  --sarif --output="output.sarif" --metrics off --force-color
semgrep ci --config="p/default, p/python"  --sarif --output="output.sarif" --metrics off --force-color 
semgrep ci --config="p/default" "p/python"  --sarif --output="output.sarif" --metrics off --force-color
semgrep ci --config="p/default p/python"  --sarif --output="output.sarif" --metrics off --force-color
```
-----

Error Message Appears
```
Any of these permutations:
semgrep ci --config="p/default" --config=" p/javascript"
semgrep ci --force-color --config="p/default"  --config=" p/javascript"
semgrep ci --force-color --metrics off --config="p/default"  --config=" p/javascript"
semgrep ci --config="p/default"  --config=" p/javascript" --force-color --metrics off

  SCAN ENVIRONMENT
  versions    - semgrep 1.22.0 on python 3.8.10                        
  environment - running in environment git, triggering event is unknown
[ERROR] WARNING: unable to find a config; path ` p/javascript` does not exist
[ERROR] invalid configuration file found (1 configs were invalid)
```

Better understanding of the issue - the error is in the exit code returned, not that there is no error output. It is correctly sent to file output (rather than error message in the terminal) with the flags given by the issue proposer.
```
echo $? 
returns 7 for semgrep --config=" p/auto" "test.js"
returns 0 for all semgrep ci runs
```
#### Next Steps
- Requires investigation into semgrep CI
- Error code returns are in semgrep-core, not Python wrapper

