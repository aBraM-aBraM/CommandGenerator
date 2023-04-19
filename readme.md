# Command Generator

This is a specification for a method of generating integration between server and client.

The purpose is to generate an rpyc-like / ssh-like behavior between two communicating computers
with one computer running compiled code and exporting features using a simple python api.

|          | Client | Server   |
|----------|--------|----------|
| Software | Python | Compiled |

>I use the name Server to describe a machine controlled by a client this specification doesn't
> require the Server to bind to a port.

## Assumptions

I assume you compile your server with a language that has
some `binutils` support. You should be able to receive a list
of all demangled functions symbols from the binary.

## Communication

The client sends commands to the server.

For example this what a `dirlist` / `ls` would look like

```c++
// server

std::array<std::string> dirlist(std::string path)
{
    ...
}
```

```python
// client
def dirlist(path: str) -> List[str]:
    pass
```

> I use c++ and python pseudocode for the server and client respectively

We'd expect to have something similar to this pseudo tlv packet:

| command number | arg count |
|----------------|-----------|
| size0          | arg0      |
| size1          | arg1      | 
| .              | .         |
| ..             | ..        |
| sizeN          | argN      |

And we'd expect the server's code to look something like:

```c++
// server

while(1)
{
    Command command = recv_command();
    send(command.method(command.args));
}
```

## Code Generation

### Mapping between command_number to function

In order to allow commands to be generated there's a need for some 
smart macros.

The main challenge in generating code is
avoiding the creation of a manual `map` / `dictionary` mapping
between `command_number` -> `function`.

This could be addressed in the following manner.

We would have all exported functions be wrapped in a macro
changing their section to their name with some common prefix.

```c++
#define EXPORT(func_name) __attribute__((section("ex_ # func_name"))) func_name

std::array<std::string> EXPORT(dirlist)(std::string path) {
    ...
}
```

And have them centralized in one section using a linker script.

```ld
SECTIONS {
    .command_table : {
        command_table_start = .;
        *(ex_*);
    }
}
```

Now we'd be able to have their relative address to `command_table_start`
calculated by parsing the binary with binutils' `nm` as a command_number;

This would allow us to have the server's code changed to this:
```c++
// server

extern char command_table_start;

while(1)
{
    Command command = recv_command();
    auto function_addr = &command_table_start + command.command_number; 
    send((FuncType)(function_addr)(command.args));
}
```

Now we can also demangle each function name and generate a python
function from the compiled signature. This requires a translation from
the compiled language types to python types.

This will require more work if your selected compiled language doesn't
add types into symbols (c++ adds symbols).

The output of `nm -C <elf>` looks like the following line:
```
ADDRESS       TYPE SIGNATURE
0000000001129 T    my_function(int)
```


```python
# api_generator.py

def generate_api(address, signature):
    print(
        f"""
        def {signature.name}({signature.args}):
            return communication.send({address}, {signature.args})
        """
    )

def translate_signature(nm_signature: str):
    return Signature(name=..., args=...)

def main():
    for sym in sys.stdin.read():
        sym_addr, _, sym_signature = sym.split()
        if sym_signature.startswith("ex_"):
            generate_api(sym_addr, translate_signature(sym_signature))
```

Usage:

```sh
nm -C <my_elf> | api_generator.py > commands.py
```