- Feature Name: (fill me in with a unique ident, `command_overhaul`)
- Start Date: (fill me in with today's date, YYYY-MM-DD)
- RFC PR: [rust-lang/rfcs#0000](https://github.com/rust-lang/rfcs/pull/0000)
- Rust Issue: [rust-lang/rust#0000](https://github.com/rust-lang/rust/issues/0000)

# Summary
[summary]: #summary

`std::process::Command` is in need of serious overhaul to improve error handling ergonomics and protect programmers from error handling mistakes.

The principal changes proposed are:

 * Add `#[must_use]` to several critical types in `std::process` [2021].
 * Provide a cooked method `Command::run()` which combines spawn, wait, and an error check.
 * `impl std::error::Error` for `process::ExitStatus` and provide `.ok()` [2021].
 * Change the return value of`Command::output` [2021] to a suitable `Result` type which can represent failure wait status, stderr output, or both.


# Motivation (overall)
[motivation]: #motivation

Currently, correct use of `Command` involves a lot of tiresome boilerplate to deal with `ExitStatua` and so on.  The compiler fails to spot many obvious programming mistakes, including failing to check the subprocess's exit status.

There are a number of distinct but interlocking problems.  It seems convenient to me to deal with them in a single RFC, but there is a separate instance of the RFC template questions for each.

Running subprocesses is complex, and provides many opportunites for errors to occur, so there are some questions about how to best represent this complexity in a convenient API.


# Diagnose failure to run, or wait, or check exit status

## Motivation

The following incorrect program fragments are all accepted today:

```
    Command::new("touch")
        .args(&["/dev/enoent/touch-1"]);
    // ^ programmer surely wanted to actually run the command

    Command::new("touch")
        .args(&["/dev/enoent/touch-2"])
        .spawn()?;
    // ^ accidentally failed to wait, if programmer wanted to make a daemon
    //   or zombie or something they should have to write let _ =.

    Command::new("touch")
        .args(&["/dev/enoent/touch-3"])
        .spawn()?
        .wait()?;
    // ^ accidentally failed to check exit status
```

## Proposed change

The compiler will report accidental failure to: actually run the command; wait for it to complete; or to check its exit status.

`std::process::Command`, `std::process::Child` and `std::process::ExitStatus` will all be marked with `#[must_use]`.

This is not a compatible change.  It will be done in the 2021 edition.

## Drawbacks

Some code that deliberately wants to leak a child process or ignore an exit status will have to use `let _ =` or some such.  This requirement is in line with general Rust practice.

## Alternatives

We could invent an alternative less error-prone API which still gives the same level of detailed control.  It would have to have a different name.

Making this change only in the 2021 edition will not help fix existing programs in the ecosystem which have these error handling bugs.  We could make the breaking change immediately.  But that doesn't seem very friendly.

## Prior art

Many existing types are `#[must_use]`.  This is generally true of builder types and of types representing possible errors.  For another example, consider futures, which only do anything if you await them.

## Unresolved questions

None


# Provide `Command::run`

## Motivation

Today simply running a subprogram in a minimally correct way looks something like this:

```
    let status = Command::new("touch")
	.args(&["/dev/enoent/touch-4"])
	.status()?;
    if !status.success() {
	Err(anyhow!("touch failed {}", status))
	    ?;
    }
```

Even this clumsy formulation has the problem that the stderr output from the command goes to our own stderr.  That is something people may want sometimes but it is not a good default.

## Guide level explanation

Provide a new method on `Command`, called `run`, which runs the command, and returns a `Result<(),_>`.  Program failures, including (by default) any stderr output, produce a suitable `Err`.

## Reference level explanation

```
    impl Command {
        /// self.stderr is piped by default
        /// If stderr piped, collects the stderr and treats any as an error.
        /// Otherwise, stderr is not collected and not treated as an error.
	///
        /// Returns `Err(io::Error)` if we failed to spawn or wait for the child, or failed to obtain its output for some reason not caused by the child.
        /// Returns `Ok(Err(CommandError))` if we obtained the child's status and (if applicable) stderr output as expected, but one of those indicated an error.
        pub fn run(&mut self) -> Result<Result<(), CommandError>, io::Error>;

    pub struct CommandError { status: ExitStatus, stderr: Vec<u8>, }
    impl CommandError {
        /// For callers who want to tolerate some exit statuses, but still fail if there was stderr output.  Eg, someone running `diff`.
        pub fn just_status(self) -> Result<ExitStatus, CommandError> {
            if self.stderr_bytes().is_mepty() { Ok(self.status) }
            else { Err(self) }
        }
        pub fn get_status(&self) -> ExitStatus { ... }
        pub fn stderr(&self) -> Option<Cow<'_, str>> { ... } // uses from_*_lossy
        pub fn stderr_bytes(&self) -> &[u8] { ... }
        pub fn into_raw_parts(self) -> (ExitStstus, Vec<u8>) { ... }
    }
```

## Drawbacks

The nested `Result` is unusual.

The `CommandError` type may seem baroque.

## Prior art

In modern Python 3 the recommended way to run a process is a `run` function which can capture the stderr (and stdout) output, but does not do so by default.

tcl's `exec` proc treats stderr output as an error, unless it is explictly dealt with another way.

Nested `Result` types occur occasionally in Rust code especially when doing complex things.

## Rationale and alternatives

`run` seems an obviously useful convenience.  It should be the usual way to run a synchronous subprocess.  The error return from `run` must contain the exit status and the sderr output.

Perhaps the `io::Error` (comms error) should be subsumed into `CommandError`.  Another way to look at that is that it would be hiding the nested `Result` inside `CommandError`.  The problem is that the accessor methods then all need to cope with the `we failed to actually figure out what the command did` situation.

`stderr` providing the result of from_*_lossy is needed because the format is not known by the programmer writing portable code.  On Unix it is UTF-8.  On Windows, it is UTF-16 at least some of the time.

The dichotomy between `just_status` and `get_status` is to make it hard to accidentally forget to check for stderr output while checking for an expectected nonzero exit status.  Perhaps this is overkill, in which case just `status` might do.

# Unresolved questions

Is it right for `run` to call stderr output an error by default?  I think this is safer, but there is room for reasonable disagreement.

Is it better to have `Result<Result<(),_>,_>` or to have accessor methods with complex interlocking conditions ?

Is it right to treat stderr as UTF-16 on Windows ?


# Enhance `ExitStatus`

## Motivation

A caller who does not want to capture or check stderr must write un-ergonomic code to handle `ExitStatus`, calling `sucess()` and converting the result to an error.

## Proposed change

```
    impl std::error::Error for ExitStatus { .... }

    impl ExitStatus {
        fn ok(self) -> Result<(), ExitStatus> {
            if self.success() { Ok(()) }
           else { Err(self) }
        }
    }
```

The new trait impl woudl have to be in the 2021 edition and that really means that the `ok` method has to as well.

## Motivation

The `ok` method for converting `ExitStatus` into a `Result` is obviously convenient.

If you have determined that things are wrong you can `fehler::throw!` (or `return Err()`) an `ExitStatus` directly.

## Drawbacks

It is unusual to have an `Error` which might represent some kind of success code.  However, this is appropriate: sometimes a program is expected to give a particular nonzero exit status and unexpectedly exits 0; in which case being able to use the `ExitStatus` as an error directly is convenient.

One obvious alternative would be to introduce a new `ExitStatusError` type which would be a newtype for `ExitStatus` (maybe only for nonzero ones).

## Prior art

Many things in Rust already have an `ok()` method.

I'm not aware of prior art for `fn ok(self: Thing) -> Result<(),Thing>`.


# Change the return value of `Command::output` and abolish `Output`

## Motivation

`std::process::Output` is an accident waiting to happen.  It is a transparent struct which combines desired output with error information in a way that requires programme/reviewer vigilance to see that all the fields are checked.

Worse, the fact that it returns a `Result` makes it seem like

Actually doing things right is quite hard work.

Today, this is accepted:

``
    let output = Command::new("ls")
        .arg("/dev/enoent/ls-1")
        .output()?;
    let stdout = String::from_utf8(output.stdout)?;
    eprintln!("got = {}", &stdout);
    // ^ two mistakes:
    //   1. failed to check exit status
    //   2. stderr is discarded
```

The minimally correct version looks something like this:

```
    let output = Command::new("ls")
	.arg("/dev/enoent/ls-2")
	.output()?;
    let stderr = String::from_utf8_lossy(&output.stderr);
    if !output.status.success() {
	Err(anyhow!("ls failed {}: {}",
		    output.status,
		    &stderr))
	    ?;
    } else if !output.stderr.is_empty() {
	Err(anyhow!("ls exited 0 but produced stderr output: {}",
		    &stderr))
	    ?;
    }
    let stdout = String::from_utf8(output.stdout)?;
    eprintln!("got = {}", &stdout);
````

## Proposed change

```
    impl Command {
        pub fn output(
