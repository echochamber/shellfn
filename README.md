# shellfn

This is rust attribute-like proc macro which reduces amount of code required to call shell command and parse results.

It allows to wrap script in any language with strongly typed function. The function's arguments are set as env variables and the result of the script is parsed either as a value or as an iterator.

## Examples

### Basic

```rust
use shellfn::shellfn;

#[shell]
fn list_modified(dir: &str) -> Result<impl Iterator<Item=String>, Box<Error>> { r#"
    cd $DIR
    git status | grep '^\s*modified:' | awk '{print $2}'
"# }
```

### Different interpreter

```rust
use shellfn::shell;
use std::error::Error;

#[shell(cmd = "python -c")]
fn pretty_json(json: &str, indent: u8, sort_keys: bool) -> Result<String, Box<Error>> { r#"
import os, json

input = os.environ['JSON']
indent = int(os.environ['INDENT'])
sort_keys = os.environ['SORT_KEYS'] == 'true'
obj = json.loads(input)

print(json.dumps(obj, indent=indent, sort_keys=sort_keys))
 "# }
```

## Usage

You can use `#[shell]` attribute on functions that have:
- body containing only one expression - string literal with script to execute
- parameters that have method `.to_string()` defined
- result that is either `void`, `T`, `Result<T, E>`, `impl Iterator<Item=T>`, `Result<impl Iterator<Item=T>>` or `Result<impl Iterator<Item=Result<T, E>>>` with constrains:
```
T: FromStr,
<T as FromStr>::Err: StdError,
E: From<shellfn::Error<<T as FromStr>::Err>>,
```

- ## Details

`#[shell]` attribute does the following:

1. Sets every argument as env variable
2. Creates shell command
3. Launches the command using `std::process::Command`
4. Depending on the return type, parses the output

Most of the steps can be adjusted:
- default command is `bash -c`. You can change it with `cmd` parameter:
```rust
#[shell(cmd = "python -c")]
```
- by default, the script is added as a last argument. You can change it using special variable `PROGRAM` in the `cmd` parameter:
```rust
#[shell(cmd = "bash -c PROGRAM -i")]
```
- if the return type is not wrapping some part of the result in `Result`, you may decide to suppress panics by adding `no_panic` flag:
```rust
#[shell(no_panic)]
```

Following return types are currently recognized:

|                  return type                  |  flags   | on parse fail | on error exit code | on spawn fail | notes |
|-----------------------------------------------|----------|---------------|--------------------|---------------|-------|
|                                               |          | -             | panic              | panic         |       |
|                                               | no_panic | -             | nothing            | nothing       |       |
| T                                             |          | panic         | panic              | panic         | 2     |
| T                                             | no_panic | panic         | panic              | panic         | 1,2   |
| Result<T, E>                                  |          | error         | error              | error         | 2     |
| Result<T, E>                                  | no_panic | error         | error              | error         | 1,2   |
| impl Iterator<Item=T>                         |          | panic         | panic              | panic         |       |
| impl Iterator<Item=T>                         | no_panic | ignore errors | ignore errors      | empty iter    | 3     |
| impl Iterator<Item=Result<T, E>>              |          | item error    | panic              | panic         | 3     |
| impl Iterator<Item=Result<T, E>>              | no_panic | item error    | ignored            | empty iter    |       |
| Result<impl Iterator<Item=T>, E>              |          | panic         | ignored            | error         |       |
| Result<impl Iterator<Item=T>, E>              | no_panic | ignore errors | ignored            | error         |       |
| Result<impl Iterator<Item=Result<T, E1>>, E2> |          | item error    | ignored            | error         |       |
| Result<impl Iterator<Item=Result<T, E1>>, E2> | no_panic | item error    | ignored            | error         | 1     |

Glossary:

|     action    |                                 meaning                                  |
|---------------|--------------------------------------------------------------------------|
| panic         | panics (.expect or panic!)                                               |
| nothing       | consumes and ignores error (let _ = ...)                                 |
| error         | returns error                                                            |
| ignore errors | yields all successfuly parsed items, ignores parsing failures (flat_map) |
| empty iter    | returns empty iterator                                                   |
| item error    | when parsing fails, yields Err                                           |
| ignored       | ignores exit code, behaves in the same way for exit code 0 and != 0      |

Notes:

1. `no_panic` attribute makes no difference
2. It reads whole stdout first before any failure
3. It yields all items until error exit code occurs

# Contribution

All contributions and comments are more than welcome! Don't be afraid to open an issue or PR whenever you find a bug or have an idea to improve this crate.

# License

MIT License

Copyright (c) 2017 Marcin Sas-Szymański

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all
copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
SOFTWARE.