
### Function methods
The declaration of Function can be found in `include/v8.h` (just noting this as I've looked for it several times)

### Symbol
The declarations for the Symbol class can be found in `v8.h` and the internal
implementation in `src/api/api.cc`.

The well known Symbols are generated using macros so you won't find the just
by searching using the static function names like 'GetToPrimitive`.
```c++
#define WELL_KNOWN_SYMBOLS(V)                 \
  V(AsyncIterator, async_iterator)            \
  V(HasInstance, has_instance)                \
  V(IsConcatSpreadable, is_concat_spreadable) \
  V(Iterator, iterator)                       \
  V(Match, match)                             \
  V(Replace, replace)                         \
  V(Search, search)                           \
  V(Split, split)                             \
  V(ToPrimitive, to_primitive)                \
  V(ToStringTag, to_string_tag)               \
  V(Unscopables, unscopables)

#define SYMBOL_GETTER(Name, name)                                   \
  Local<Symbol> v8::Symbol::Get##Name(Isolate* isolate) {           \
    i::Isolate* i_isolate = reinterpret_cast<i::Isolate*>(isolate); \
    return Utils::ToLocal(i_isolate->factory()->name##_symbol());   \
  }
```
So GetToPrimitive would become:
```c++
Local<Symbol> v8::Symbol::GeToPrimitive(Isolate* isolate) {
  i::Isolate* i_isolate = reinterpret_cast<i::Isolate*>(isolate);
  return Utils::ToLocal(i_isolate->factory()->to_primitive_symbol());
}

```



There is an example in [symbol-test.cc](./test/symbol-test.cc).

### String types
There are a number of different String types in V8 which are optimized for various situations.
If we look in src/objects/objects.h we can see the object hierarchy:
```
    Object
      SMI
      HeapObject    // superclass for every object instans allocated on the heap.
        ...
        Name
          String
            SeqString
              SeqOneByteString
              SeqTwoByteString
            SlicedString
            ConsString
            ThinString
            ExternalString
              ExternalOneByteString
              ExternalTwoByteString
            InternalizedString
              SeqInternalizedString
                SeqOneByteInternalizedString
                SeqTwoByteInternalizedString
              ConsInternalizedString
              ExternalInternalizedString
                ExternalOneByteInternalizedString
                ExternalTwoByteInternalizedString
```

Note that v8::String is declared in `include/v8.h`.

`Name` as can be seen extends HeapObject and anything that can be used as a
property name should extend Name.

Looking at the declaration in include/v8.h we find the following:
```c++
    int GetIdentityHash();
    static Name* Cast(Value* obj)
```

#### String
A String extends `Name` and has a length and content. The content can be made
up of 1 or 2 byte characters.
Looking at the declaration in include/v8.h we find the following:
```c++
    enum Encoding {
      UNKNOWN_ENCODING = 0x1,
      TWO_BYTE_ENCODING = 0x0,
      ONE_BYTE_ENCODING = 0x8
    };

    int Length() const;
    int Uft8Length const;
    bool IsOneByte() const;
```
Example usages can be found in [test/string_test.cc](./test/string_test.cc).
Looking at the functions I've seen one that returns the actual bytes
from the String. You can get at the in utf8 format using:
```c++
    String::Utf8Value print_value(joined);
    std::cout << *print_value << '\n';
```
So that is the only string class in include/v8.h, but there are a lot more
implementations that we've seen above. There are used for various cases, for
example for indexing, concatenation, and slicing).

#### SeqString
Represents a sequence of charaters which (the characters) are either one or two
bytes in length.

#### ConsString
These are string that are built using:

    const str = "one" + "two";

This would be represented as:
```
         +--------------+
         |              | 
   [str|one|two]     [one|...]   [two|...]
             |                       |
             +-----------------------+
```
So we can see that one and two in str are pointers to existing strings. 


#### ExternalString
These strings are located on the native heap. The ExternalString structure has a
pointer to this external location and the usual length field for all Strings.

## Builtins
Are JavaScript functions/objects that are provided by V8. These are built using a
C++ DSL and are passed through:

    CodeStubAssembler -> CodeAssembler -> RawMachineAssembler.

Builtins need to have bytecode generated for them so that they can be run in TurboFan.

`src/code-stub-assembler.h`

All the builtins are declared in `src/builtins/builtins-definitions.h` by the
`BUILTIN_LIST_BASE` macro. 
There are different type of builtins (TF = Turbo Fan):
* TFJ 
JavaScript linkage which means it is callable as a JavaScript function  
* TFS
CodeStub linkage. A builtin with stub linkage can be used to extract common code into a separate code object which can
then be used by multiple callers. These is useful because builtins are generated at compile time and
included in the V8 snapshot. This means that they are part of every isolate that is created. Being 
able to share common code for multiple builtins will save space.

* TFC 
CodeStub linkage with custom descriptor

To see how this works in action we first need to disable snapshots. If we don't, we won't be able to
set breakpoints as the the heap will be serialized at compile time and deserialized upon startup of v8.

To find the option to disable snapshots use:

    $ gn args --list out.gn/learning --short | more
    ...
    v8_use_snapshot=true
    $ gn args out.gn/learning
    v8_use_snapshot=false
    $ gn -C out.gn/learning

After building we should be able to set a break point in bootstrapper.cc and its function 
`Genesis::InitializeGlobal`:

    (lldb) br s -f bootstrapper.cc -l 2684

Lets take a look at how the `JSON` object is setup:

    Handle<String> name = factory->InternalizeUtf8String("JSON");
    Handle<JSObject> json_object = factory->NewJSObject(isolate->object_function(), TENURED);

`TENURED` means that this object should be allocated directly in the old generation.

    JSObject::AddProperty(global, name, json_object, DONT_ENUM);

`DONT_ENUM` is checked by some builtin functions and if set this object will be ignored by those
functions.

    SimpleInstallFunction(json_object, "parse", Builtins::kJsonParse, 2, false);

Here we can see that we are installing a function named `parse`, which takes 2 parameters. You can
find the definition in src/builtins/builtins-json.cc.
What does the `SimpleInstallFunction` do?

Lets take `console` as an example which was created using:

    Handle<JSObject> console = factory->NewJSObject(cons, TENURED);
    JSObject::AddProperty(global, name, console, DONT_ENUM);
    SimpleInstallFunction(console, "debug", Builtins::kConsoleDebug, 1, false,
                          NONE);

    V8_NOINLINE Handle<JSFunction> SimpleInstallFunction(
      Handle<JSObject> base, 
      const char* name, 
      Builtins::Name call, 
      int len,
      bool adapt, 
      PropertyAttributes attrs = DONT_ENUM,
      BuiltinFunctionId id = kInvalidBuiltinFunctionId) {

So we can see that base is our Handle to a JSObject, and name is "debug".
Builtins::Name is Builtins:kConsoleDebug. Where is this defined?  
You can find a macro named `CPP` in `src/builtins/builtins-definitions.h`:

   CPP(ConsoleDebug)

What does this macro expand to?  
It is part of the `BUILTIN_LIST_BASE` macro in builtin-definitions.h
We have to look at where BUILTIN_LIST is used which we can find in builtins.cc.
In `builtins.cc` we have an array of `BuiltinMetadata` which is declared as:

    const BuiltinMetadata builtin_metadata[] = {
      BUILTIN_LIST(DECL_CPP, DECL_API, DECL_TFJ, DECL_TFC, DECL_TFS, DECL_TFH, DECL_ASM)
    };

    #define DECL_CPP(Name, ...) { #Name, Builtins::CPP, \
                                { FUNCTION_ADDR(Builtin_##Name) }},

Which will expand to the creation of a BuiltinMetadata struct entry in the array. The
BuildintMetadata struct looks like this which might help understand what is going on:

    struct BuiltinMetadata {
      const char* name;
      Builtins::Kind kind;
      union {
        Address cpp_entry;       // For CPP and API builtins.
        int8_t parameter_count;  // For TFJ builtins.
      } kind_specific_data;
    };

So the `CPP(ConsoleDebug)` will expand to an entry in the array which would look something like
this:

    { ConsoleDebug, 
      Builtins::CPP, 
      {
        reinterpret_cast<v8::internal::Address>(reinterpret_cast<intptr_t>(Builtin_ConsoleDebug))
      }
    },

The third paramter is the creation on the union which might not be obvious.

Back to the question I'm trying to answer which is:  
"Buildtins::Name is is Builtins:kConsoleDebug. Where is this defined?"  
For this we have to look at `builtins.h` and the enum Name:

    enum Name : int32_t {
    #define DEF_ENUM(Name, ...) k##Name,
        BUILTIN_LIST_ALL(DEF_ENUM)
    #undef DEF_ENUM
        builtin_count
     };

This will expand to the complete list of builtins in builtin-definitions.h using the DEF_ENUM
macro. So the expansion for ConsoleDebug will look like:

    enum Name: int32_t {
      ...
      kDebugConsole,
      ...
    };

So backing up to looking at the arguments to SimpleInstallFunction which are:

    SimpleInstallFunction(console, "debug", Builtins::kConsoleDebug, 1, false,
                          NONE);

    V8_NOINLINE Handle<JSFunction> SimpleInstallFunction(
      Handle<JSObject> base, 
      const char* name, 
      Builtins::Name call, 
      int len,
      bool adapt, 
      PropertyAttributes attrs = DONT_ENUM,
      BuiltinFunctionId id = kInvalidBuiltinFunctionId) {

We know about `Builtins::Name`, so lets look at len which is one, what is this?  
SimpleInstallFunction will call:

    Handle<JSFunction> fun =
      SimpleCreateFunction(base->GetIsolate(), function_name, call, len, adapt);

`len` would be used if adapt was true but it is false in our case. This is what it would 
be used for if adapt was true:

    fun->shared()->set_internal_formal_parameter_count(len);

I'm not exactly sure what adapt is referring to here.

PropertyAttributes is not specified so it will get the default value of `DONT_ENUM`.
The last parameter which is of type BuiltinFunctionId is not specified either so the
default value of `kInvalidBuiltinFunctionId` will be used. This is an enum defined in 
`src/objects/objects.h`.


This [blog](https://v8project.blogspot.se/2017/11/csa.html) provides an example of adding
a function to the String object. 

    $ out.gn/learning/mksnapshot --print-code > output

You can then see the generated code from this. This will produce a code stub that can 
be called through C++. Lets update this to have it be called from JavaScript:

Update builtins/builtins-string-get.cc :

    TF_BUILTIN(GetStringLength, StringBuiltinsAssembler) {
      Node* const str = Parameter(Descriptor::kReceiver);
      Return(LoadStringLength(str));
    }

We also have to update builtins/builtins-definitions.h:

    TFJ(GetStringLength, 0)

And bootstrapper.cc:

    SimpleInstallFunction(prototype, "len", Builtins::kGetStringLength, 0, true);

If you now build using 'ninja -C out.gn/learning' you should be able to run d8 and try this out:

    d8> const s = 'testing'
    undefined
    d8> s.len()
    7

Now lets take a closer look at the code that is generated for this:

    $ out.gn/learning/mksnapshot --print-code > output

Looking at the output generated I was surprised to see two entries for GetStringLength (I changed the name
just to make sure there was not something else generating the second one). Why two?

The following uses Intel Assembly syntax which means that no register/immediate prefixes and the first operand is the 
destination and the second operand the source.
```
--- Code ---
kind = BUILTIN
name = BeveStringLength
compiler = turbofan
Instructions (size = 136)
0x1fafde09b3a0     0  55             push rbp
0x1fafde09b3a1     1  4889e5         REX.W movq rbp,rsp                  // movq rsp into rbp

0x1fafde09b3a4     4  56             push rsi                            // push the value of rsi (first parameter) onto the stack 
0x1fafde09b3a5     5  57             push rdi                            // push the value of rdi (second parameter) onto the stack
0x1fafde09b3a6     6  50             push rax                            // push the value of rax (accumulator) onto the stack

0x1fafde09b3a7     7  4883ec08       REX.W subq rsp,0x8                  // make room for a 8 byte value on the stack
0x1fafde09b3ab     b  488b4510       REX.W movq rax,[rbp+0x10]           // move the value rpm + 10 to rax
0x1fafde09b3af     f  488b58ff       REX.W movq rbx,[rax-0x1]
0x1fafde09b3b3    13  807b0b80       cmpb [rbx+0xb],0x80                // IsString(object). compare byte to zero
0x1fafde09b3b7    17  0f8350000000   jnc 0x1fafde09b40d  <+0x6d>        // jump it carry flag was not set

0x1fafde09b3bd    1d  488b400f       REX.W movq rax,[rax+0xf]
0x1fafde09b3c1    21  4989e2         REX.W movq r10,rsp
0x1fafde09b3c4    24  4883ec08       REX.W subq rsp,0x8
0x1fafde09b3c8    28  4883e4f0       REX.W andq rsp,0xf0
0x1fafde09b3cc    2c  4c891424       REX.W movq [rsp],r10
0x1fafde09b3d0    30  488945e0       REX.W movq [rbp-0x20],rax
0x1fafde09b3d4    34  48be0000000001000000 REX.W movq rsi,0x100000000
0x1fafde09b3de    3e  48bad9c228dfa8090000 REX.W movq rdx,0x9a8df28c2d9    ;; object: 0x9a8df28c2d9 <String[101]: CAST(LoadObjectField(object, offset, MachineTypeOf<T>::value)) at ../../src/code-stub-assembler.h:432>
0x1fafde09b3e8    48  488bf8         REX.W movq rdi,rax
0x1fafde09b3eb    4b  48b830726d0a01000000 REX.W movq rax,0x10a6d7230    ;; external reference (check_object_type)
0x1fafde09b3f5    55  40f6c40f       testb rsp,0xf
0x1fafde09b3f9    59  7401           jz 0x1fafde09b3fc  <+0x5c>
0x1fafde09b3fb    5b  cc             int3l
0x1fafde09b3fc    5c  ffd0           call rax
0x1fafde09b3fe    5e  488b2424       REX.W movq rsp,[rsp]
0x1fafde09b402    62  488b45e0       REX.W movq rax,[rbp-0x20]
0x1fafde09b406    66  488be5         REX.W movq rsp,rbp
0x1fafde09b409    69  5d             pop rbp
0x1fafde09b40a    6a  c20800         ret 0x8

// this is where we jump to if IsString failed
0x1fafde09b40d    6d  48ba71c228dfa8090000 REX.W movq rdx,0x9a8df28c271    ;; object: 0x9a8df28c271 <String[76]\: CSA_ASSERT failed: IsString(object) [../../src/code-stub-assembler.cc:1498]\n>
0x1fafde09b417    77  e8e4d1feff     call 0x1fafde088600     ;; code: BUILTIN
0x1fafde09b41c    7c  cc             int3l
0x1fafde09b41d    7d  cc             int3l
0x1fafde09b41e    7e  90             nop
0x1fafde09b41f    7f  90             nop


Safepoints (size = 8)

RelocInfo (size = 7)
0x1fafde09b3e0  embedded object  (0x9a8df28c2d9 <String[101]: CAST(LoadObjectField(object, offset, MachineTypeOf<T>::value)) at ../../src/code-stub-assembler.h:432>)
0x1fafde09b3ed  external reference (check_object_type)  (0x10a6d7230)
0x1fafde09b40f  embedded object  (0x9a8df28c271 <String[76]\: CSA_ASSERT failed: IsString(object) [../../src/code-stub-assembler.cc:1498]\n>)
0x1fafde09b418  code target (BUILTIN)  (0x1fafde088600)

--- End code --- 
```


### TF_BUILTIN macro
Is a macro to defining Turbofan (TF) builtins and can be found in `builtins/builtins-utils-gen.h`

If we take a look at the file src/builtins/builtins-bigint-gen.cc and the following
function:
```c++
TF_BUILTIN(BigIntToI64, CodeStubAssembler) {                                       
  if (!Is64()) {                                                                   
    Unreachable();                                                                 
    return;                                                                        
  }                                                                                
                                                                                   
  TNode<Object> value = CAST(Parameter(Descriptor::kArgument));                    
  TNode<Context> context = CAST(Parameter(Descriptor::kContext));                  
  TNode<BigInt> n = ToBigInt(context, value);                                      
                                                                                   
  TVARIABLE(UintPtrT, var_low);                                                    
  TVARIABLE(UintPtrT, var_high);                                                   
                                                                                   
  BigIntToRawBytes(n, &var_low, &var_high);                                        
  Return(var_low.value());                                                         
}
```
Let's take our GetStringLength example from above and see what this will be expanded to after
processing this macro:
```console
$ clang++ --sysroot=build/linux/debian_sid_amd64-sysroot -isystem=./buildtools/third_party/libc++/trunk/include -isystem=buildtools/third_party/libc++/trunk/include -I. -E src/builtins/builtins-bigint-gen.cc > builtins-bigint-gen.cc.pp
```
```c++
static void Generate_BigIntToI64(compiler::CodeAssemblerState* state);

class BigIntToI64Assembler : public CodeStubAssembler { 
 public:
  using Descriptor = Builtin_BigIntToI64_InterfaceDescriptor; 
  explicit BigIntToI64Assembler(compiler::CodeAssemblerState* state) : CodeStubAssembler(state) {} 
  void GenerateBigIntToI64Impl(); 
  Node* Parameter(Descriptor::ParameterIndices index) {
    return CodeAssembler::Parameter(static_cast<int>(index));
  }
}; 

void Builtins::Generate_BigIntToI64(compiler::CodeAssemblerState* state) {
  BigIntToI64Assembler assembler(state);
  state->SetInitialDebugInformation("BigIntToI64", "src/builtins/builtins-bigint-gen.cc", 14);
  if (Builtins::KindOf(Builtins::kBigIntToI64) == Builtins::TFJ) {
    assembler.PerformStackCheck(assembler.GetJSContextParameter());
  }
  assembler.GenerateBigIntToI64Impl();
} 
void BigIntToI64Assembler::GenerateBigIntToI64Impl() {
 if (!Is64()) {                                                                
   Unreachable();                                                              
   return;                                                                     
 }                                                                             
                                                                                
 TNode<Object> value = Cast(Parameter(Descriptor::kArgument));                 
 TNode<Context> context = Cast(Parameter(Descriptor::kContext));                
 TNode<BigInt> n = ToBigInt(context, value);                                   
                                                                               
 TVariable<UintPtrT> var_low(this);                                            
 TVariable<UintPtrT> var_high(this);                                           
                                                                                
 BigIntToRawBytes(n, &var_low, &var_high);                                     
 Return(var_low.value());                                                      
} 
```

From the resulting class you can see how `Parameter` can be used from within `TF_BUILTIN` macro.

## Building V8
You'll need to have checked out the Google V8 sources to you local file system and build it by following 
the instructions found [here](https://developers.google.com/v8/build).

### [gclient](https://www.chromium.org/developers/how-tos/depottools) sync
```console
$ gclient sync
```

### [GN](https://chromium.googlesource.com/chromium/src/+/master/tools/gn/docs/quick_start.md)
GN stands for Generate Ninja.
```console
$ tools/dev/v8gen.py --help
```

Show all executables
```console
$ gn ls out/x64.release_gcc --type=executable
```
If you want details of a specific target you can use `desc`:
```console
$ gn desc out/x64.release_gcc/ //:mksnapshot
``` 
`//:mksnapshot` is a label where `//` indicates the root directory and then the
name of the label. If the label was in a subdirectory that subdirectory would
come before the `:`. 
This command is useful to see which files are included in an executable.

When gn starts it will search for a .gn file in the current directory. The one
in specifies the following:
```
import("//build/dotfile_settings.gni")                                          
```
This includes a bunch of gni files from the build directory
(the dotfile_settings.gni) that is.

```console
buildconfig = "//build/config/BUILDCONFIG.gn"                                   
```
This is the master GN build configuration file.


Show all shared_libraries:
```console
$ gn ls out/x64.release_gcc --type=shared_libraries
```

Edit build arguments:
```console
$ vi out/x64.release_gcc/args.gn
$ gn args out/x64.release_gcc
```

This will open an editor where you can set configuration options. I've been using the following:
```console
v8_monolithic = true
use_custom_libcxx = false
v8_use_external_startup_data = false
is_debug = true
target_cpu = "x64"
v8_enable_backtrace = true
v8_enable_slow_dchecks = true
v8_optimized_debug = false
```

List avaiable build arguments:
```console
$ gn args --list out/x64.release
```

List all available targets:
```console
$ ninja -C out/x64.release/ -t targets all
```

Building:
```console
$ env CPATH=/usr/include ninja -C out/x64.release_gcc/
```

Running quickchecks:
```
$ ./tools/run-tests.py --outdir=out/x64.release --quickcheck
```

You can use `./tools-run-tests.py -h` to list all the opitions that can be passed
to run-tests.

#### Troubleshooting build:
```console
/v8_src/v8/out/x64.release/obj/libv8_monolith.a(eh-frame.o):eh-frame.cc:function v8::internal::EhFrameWriter::WriteEmptyEhFrame(std::__1::basic_ostream<char, std::__1::char_traits<char> >&): error: undefined reference to 'std::__1::basic_ostream<char, std::__1::char_traits<char> >::write(char const*, long)'
clang: error: linker command failed with exit code 1 (use -v to see invocation)
```
`-stdlib=libc++` is llvm's C++ runtime. This runtime has a `__1` namespace.
I looks like the static library above was compiled with clangs/llvm's `libc++`
as we are seeing the `__1` namespace.

-stdlib=libstdc++ is GNU's C++ runtime

So we can see that the namespace `std::__1` is used which we now
know is the namespace that libc++ which is clangs libc++ library.
I guess we could go about this in two ways, either we can change v8 build of
to use glibc++ when compiling so that the symbols are correct when we want to
link against it, or we can update our linker (ld) to use libc++.

We need to include the correct libraries to link with during linking, 
which means specifying:
```
-stdlib=libc++ -Wl,-L$(v8_build_dir)
```
If we look in $(v8_build_dir) we find `libc++.so`. We also need to this library
to be found at runtime by the dynamic linker using `LD_LIBRARY_PATH`:
```console
$ LD_LIBRARY_PATH=../v8_src/v8/out/x64.release/ ./hello-world
```
Notice that this is using `ld` from our path. We can tell clang to use a different
search path with the `-B` option:
```console
$ clang++ --help | grep -- '-B'
  -B <dir>                Add <dir> to search path for binaries and object files used implicitly
```

`libgcc_s` is GCC low level runtime library. I've been confusing this with
glibc++ libraries for some reason but they are not the same.

Running cctest:
```console
$ out.gn/learning/cctest test-heap-profiler/HeapSnapshotRetainedObjectInfo
```
To get a list of the available tests:
```console
$ out.gn/learning/cctest --list
```

Checking formating/linting:
```
$ git cl format
```
You can then `git diff` and see the changes.

Running pre-submit checks:
```console
$ git cl presubmit
```

Then upload using:
```console
$ git cl upload
```

#### Build details
So when we run gn it will generate Ninja build file. GN itself is written in 
C++ but has a python wrapper around it. 

A group in gn is just a collection of other targets which enables them to have
a name.

So when we run gn there will be a number of .ninja files generated. If we look
in the root of the output directory we find two .ninja files:
```console
build.ninja  toolchain.ninja
```
By default ninja will look for `build.ninja' and when we run ninja we usually
specify the `-C out/dir`. If no targets are specified on the command line ninja
will execute all outputs unless there is one specified as default. V8 has the 
following default target:
```
default all

build all: phony $
    ./bytecode_builtins_list_generator $                                        
    ./d8 $                                                                      
    obj/fuzzer_support.stamp $                                                  
    ./gen-regexp-special-case $                                                 
    obj/generate_bytecode_builtins_list.stamp $                                 
    obj/gn_all.stamp $                                                          
    obj/json_fuzzer.stamp $                                                     
    obj/lib_wasm_fuzzer_common.stamp $                                          
    ./mksnapshot $                                                              
    obj/multi_return_fuzzer.stamp $                                             
    obj/parser_fuzzer.stamp $                                                   
    obj/postmortem-metadata.stamp $                                             
    obj/regexp_builtins_fuzzer.stamp $                                          
    obj/regexp_fuzzer.stamp $                                                   
    obj/run_gen-regexp-special-case.stamp $                                     
    obj/run_mksnapshot_default.stamp $                                          
    obj/run_torque.stamp $                                                      
    ./torque $                                                                  
    ./torque-language-server $                                                  
    obj/torque_base.stamp $                                                     
    obj/torque_generated_definitions.stamp $                                    
    obj/torque_generated_initializers.stamp $                                   
    obj/torque_ls_base.stamp $                                                  
    ./libv8.so.TOC $                                                            
    obj/v8_archive.stamp $
    ...
```
A `phony` rule can be used to create an alias for other targets. 
The `$` in ninja is an escape character so in the case of the all target it
escapes the new line, like using \ in a shell script.

Lets take a look at `bytecode_builtins_list_generator`: 
```
build $:bytecode_builtins_list_generator: phony ./bytecode_builtins_list_generator
```
The format of the ninja build statement is:
```
build outputs: rulename inputs
```
We are again seeing the `$` ninja escape character but this time it is escaping
the colon which would otherwise be interpreted as separating file names. The output
in this case is bytecode_builtins_list_generator. And I'm guessing, as I can't
find a connection between `./bytecode_builtins_list_generator` and 

The default `target_out_dir` in this case is //out/x64.release_gcc/obj.
The executable in BUILD.gn which generates this does not specify any output
directory so I'm assuming that it the generated .ninja file is place in the 
target_out_dir in this case where we can find `bytecode_builtins_list_generator.ninja` 
This file has a label named:
```
label_name = bytecode_builtins_list_generator                                   
```
Hmm, notice that in build.ninja there is the following command:
```
subninja toolchain.ninja
```
And in `toolchain.ninja` we have:
```
subninja obj/bytecode_builtins_list_generator.ninja
```
This is what is making `./bytecode_builtins_list_generator` available.

```console
$ ninja -C out/x64.release_gcc/ -t targets all  | grep bytecode_builtins_list_generator
$ rm out/x64.release_gcc/bytecode_builtins_list_generator 
$ ninja -C out/x64.release_gcc/ bytecode_builtins_list_generator
ninja: Entering directory `out/x64.release_gcc/'
[1/1] LINK ./bytecode_builtins_list_generator
```

Alright, so I'd like to understand when in the process torque is run to
generate classes like TorqueGeneratedStruct:
```c++
class Struct : public TorqueGeneratedStruct<Struct, HeapObject> {
```
```
./torque $                                                                  
./torque-language-server $                                                  
obj/torque_base.stamp $                                                     
obj/torque_generated_definitions.stamp $                                    
obj/torque_generated_initializers.stamp $                                   
obj/torque_ls_base.stamp $  
```
Like before we can find that obj/torque.ninja in included by the subninja command
in toolchain.ninja:
```
subninja obj/torque.ninja
```
So this is building the executable `torque`, but it has not been run yet.
```console
$ gn ls out/x64.release_gcc/ --type=action
//:generate_bytecode_builtins_list
//:postmortem-metadata
//:run_gen-regexp-special-case
//:run_mksnapshot_default
//:run_torque
//:v8_dump_build_config
//src/inspector:protocol_compatibility
//src/inspector:protocol_generated_sources
//tools/debug_helper:gen_heap_constants
//tools/debug_helper:run_mkgrokdump
```
Notice the `run_torque` target
```console
$ gn desc out/x64.release_gcc/ //:run_torque
```
If we look in toolchain.ninja we have a rule named `___run_torque___build_toolchain_linux_x64__rule`
```console
command = python ../../tools/run.py ./torque -o gen/torque-generated -v8-root ../.. 
  src/builtins/array-copywithin.tq
  src/builtins/array-every.tq
  src/builtins/array-filter.tq
  src/builtins/array-find.tq
  ...
```
And there is a build that specifies the .h and cc files in gen/torque-generated
which has this rule in it if they change.


## Building chromium
When making changes to V8 you might need to verify that your changes have not broken anything in Chromium. 

Generate Your Project (gpy) :
You'll have to run this once before building:

    $ gclient sync
    $ gclient runhooks

#### Update the code base

    $ git fetch origin master
    $ git co master
    $ git merge origin/master

### Building using GN

    $ gn args out.gn/learning

### Building using Ninja

    $ ninja -C out.gn/learning 

Building the tests:

    $ ninja -C out.gn/learning chrome/test:unit_tests

An error I got when building the first time:

    traceback (most recent call last):
    File "./gyp-mac-tool", line 713, in <module>
      sys.exit(main(sys.argv[1:]))
    File "./gyp-mac-tool", line 29, in main
      exit_code = executor.Dispatch(args)
    File "./gyp-mac-tool", line 44, in Dispatch
      return getattr(self, method)(*args[1:])
    File "./gyp-mac-tool", line 68, in ExecCopyBundleResource
      self._CopyStringsFile(source, dest)
    File "./gyp-mac-tool", line 134, in _CopyStringsFile
      import CoreFoundation
    ImportError: No module named CoreFoundation
    [6642/20987] CXX obj/base/debug/base.task_annotator.o
    [6644/20987] ACTION base_nacl: build newlib plib_9b4f41e4158ebb93a5d28e6734a13e85
    ninja: build stopped: subcommand failed.

I was able to get around this by:

    $ pip install -U pyobjc

#### Using a specific version of V8
The instructions below work but it is also possible to create a soft link from chromium/src/v8
to local v8 repository and the build/test. 

So, we want to include our updated version of V8 so that we can verify that it builds correctly with our change to V8.
While I'm not sure this is the proper way to do it, I was able to update DEPS in src (chromium) and set
the v8 entry to git@github.com:danbev/v8.git@064718a8921608eaf9b5eadbb7d734ec04068a87:

    "git@github.com:danbev/v8.git@064718a8921608eaf9b5eadbb7d734ec04068a87"

You'll have to run `gclient sync` after this. 

Another way is to not updated the `DEPS` file, which is a version controlled file, but instead update
`.gclientrc` and add a `custom_deps` entry:

    solutions = [{u'managed': False, u'name': u'src', u'url': u'https://chromium.googlesource.com/chromium/src.git', 
    u'custom_deps': {
      "src/v8": "git@github.com:danbev/v8.git@27a666f9be7ca3959c7372bdeeee14aef2a4b7ba"
    }, u'deps_file': u'.DEPS.git', u'safesync_url': u''}]

## Buiding pdfium
You may have to compile this project (in addition to chromium to verify that changes in v8 are not breaking
code in pdfium.

### Create/clone the project

     $ mkdir pdfuim_reop
     $ gclient config --unmanaged https://pdfium.googlesource.com/pdfium.git
     $ gclient sync
     $ cd pdfium

### Building

    $ ninja -C out/Default

#### Using a branch of v8
You should be able to update the .gclient file adding a custom_deps entry:

    solutions = [
    {
      "name"        : "pdfium",
      "url"         : "https://pdfium.googlesource.com/pdfium.git",
      "deps_file"   : "DEPS",
      "managed"     : False,
      "custom_deps" : {
        "v8": "git@github.com:danbev/v8.git@064718a8921608eaf9b5eadbb7d734ec04068a87"
      },
    },
   ]
   cache_dir = None
You'll have to run `gclient sync` after this too.




## Code in this repo

#### hello-world
[hello-world](./hello-world.cc) is heavily commented and show the usage of a static int being exposed and
accessed from JavaScript.

#### instances
[instances](./instances.cc) shows the usage of creating new instances of a C++ class from JavaScript.

#### run-script
[run-script](./run-script.cc) is basically the same as instance but reads an external file, [script.js](./script.js)
and run the script.

#### tests
The test directory contains unit tests for individual classes/concepts in V8 to help understand them.

## Building this projects code

    $ make

## Running

    $ ./hello-world

## Cleaning

    $ make clean

## Contributing a change to V8
1) Create a working branch using `git new-branch name`
2) git cl upload  

See Googles [contributing-code](https://www.chromium.org/developers/contributing-code) for more details.

### Find the current issue number

    $ git cl issue

## Debugging

    $ lldb hello-world
    (lldb) br s -f hello-world.cc -l 27

There are a number of useful functions in `src/objects-printer.cc` which can also be used in lldb.

#### Print value of a Local object

    (lldb) print _v8_internal_Print_Object(*(v8::internal::Object**)(*init_fn))

#### Print stacktrace

    (lldb) p _v8_internal_Print_StackTrace()

#### Creating command aliases in lldb
Create a file named [.lldbinit](./.lldbinit) (in your project director or home directory). This file can now be found in v8's tools directory.



### Using d8
This is the source used for the following examples:

    $ cat class.js
    function Person(name, age) {
      this.name = name;
      this.age = age;
    }

    print("before");
    const p = new Person("Daniel", 41);
    print(p.name);
    print(p.age);
    print("after"); 


### V8_shell startup
What happens when the v8_shell is run?   

    $ lldb -- out/x64.debug/d8 --enable-inspector class.js
    (lldb) breakpoint set --file d8.cc --line 2662
    Breakpoint 1: where = d8`v8::Shell::Main(int, char**) + 96 at d8.cc:2662, address = 0x0000000100015150

First v8::base::debug::EnableInProcessStackDumping() is called followed by some windows specific code guarded
by macros. Next is all the options are set using `v8::Shell::SetOptions`

SetOptions will call `v8::V8::SetFlagsFromCommandLine` which is found in src/api.cc:

    i::FlagList::SetFlagsFromCommandLine(argc, argv, remove_flags);

This function can be found in src/flags.cc. The flags themselves are defined in src/flag-definitions.h

Next a new SourceGroup array is create:
    
    options.isolate_sources = new SourceGroup[options.num_isolates];
    SourceGroup* current = options.isolate_sources;
    current->Begin(argv, 1);
    for (int i = 1; i < argc; i++) {
      const char* str = argv[i];

    (lldb) p str
    (const char *) $6 = 0x00007fff5fbfed4d "manual.js"

There are then checks performed to see if the args is `--isolate` or `--module`, or `-e` and if not (like in our case)

    } else if (strncmp(str, "-", 1) != 0) {
      // Not a flag, so it must be a script to execute.
      options.script_executed = true;

TODO: I'm not exactly sure what SourceGroups are about but just noting this and will revisit later.

This will take us back `int Shell::Main` in src/d8.cc

    ::V8::InitializeICUDefaultLocation(argv[0], options.icu_data_file);

    (lldb) p argv[0]
    (char *) $8 = 0x00007fff5fbfed48 "./d8"

See [ICU](international-component-for-unicode) a little more details.

Next the default V8 platform is initialized:

    g_platform = i::FLAG_verify_predictable ? new PredictablePlatform() : v8::platform::CreateDefaultPlatform();

v8::platform::CreateDefaultPlatform() will be called in our case.

We are then back in Main and have the following lines:

    2685 v8::V8::InitializePlatform(g_platform);
    2686 v8::V8::Initialize();

This is very similar to what I've seen in the [Node.js startup process](https://github.com/danbev/learning-nodejs#startint-argc-char-argv).

We did not specify any natives_blob or snapshot_blob as an option on the command line so the defaults 
will be used:

    v8::V8::InitializeExternalStartupData(argv[0]);

back in src/d8.cc line 2918:

    Isolate* isolate = Isolate::New(create_params);

this call will bring us into api.cc line 8185:

     i::Isolate* isolate = new i::Isolate(false);
So, we are invoking the Isolate constructor (in src/isolate.cc).

    isolate->set_snapshot_blob(i::Snapshot::DefaultSnapshotBlob());

api.cc:

    isolate->Init(NULL);
    
    compilation_cache_ = new CompilationCache(this);
    context_slot_cache_ = new ContextSlotCache();
    descriptor_lookup_cache_ = new DescriptorLookupCache();
    unicode_cache_ = new UnicodeCache();
    inner_pointer_to_code_cache_ = new InnerPointerToCodeCache(this);
    global_handles_ = new GlobalHandles(this);
    eternal_handles_ = new EternalHandles();
    bootstrapper_ = new Bootstrapper(this);
    handle_scope_implementer_ = new HandleScopeImplementer(this);
    load_stub_cache_ = new StubCache(this, Code::LOAD_IC);
    store_stub_cache_ = new StubCache(this, Code::STORE_IC);
    materialized_object_store_ = new MaterializedObjectStore(this);
    regexp_stack_ = new RegExpStack();
    regexp_stack_->isolate_ = this;
    date_cache_ = new DateCache();
    call_descriptor_data_ =
      new CallInterfaceDescriptorData[CallDescriptors::NUMBER_OF_DESCRIPTORS];
    access_compiler_data_ = new AccessCompilerData();
    cpu_profiler_ = new CpuProfiler(this);
    heap_profiler_ = new HeapProfiler(heap());
    interpreter_ = new interpreter::Interpreter(this);
    compiler_dispatcher_ =
      new CompilerDispatcher(this, V8::GetCurrentPlatform(), FLAG_stack_size);


src/builtins/builtins.cc, this is where the builtins are defined.
TODO: sort out what these macros do.

In src/v8.cc we have a couple of checks for if the options passed are for a stress_run but since we 
did not pass in any such flags this code path will be followed which will call RunMain:

    result = RunMain(isolate, argc, argv, last_run);

this will end up calling:

    options.isolate_sources[0].Execute(isolate);

Which will call SourceGroup::Execute(Isolate* isolate)

    // Use all other arguments as names of files to load and run.
    HandleScope handle_scope(isolate);
    Local<String> file_name = String::NewFromUtf8(isolate, arg, NewStringType::kNormal).ToLocalChecked();
    Local<String> source = ReadFile(isolate, arg);
    if (source.IsEmpty()) {
      printf("Error reading '%s'\n", arg);
      Shell::Exit(1);
    }
    Shell::options.script_executed = true;
    if (!Shell::ExecuteString(isolate, source, file_name, false, true)) {
      exception_was_thrown = true;
      break;
    }

    ScriptOrigin origin(name);
    if (compile_options == ScriptCompiler::kNoCompileOptions) {
      ScriptCompiler::Source script_source(source, origin);
      return ScriptCompiler::Compile(context, &script_source, compile_options);
    }

Which will delegate to ScriptCompiler(Local<Context>, Source* source, CompileOptions options):

    auto maybe = CompileUnboundInternal(isolate, source, options);

CompileUnboundInternal

    result = i::Compiler::GetSharedFunctionInfoForScript(
        str, name_obj, line_offset, column_offset, source->resource_options,
        source_map_url, isolate->native_context(), NULL, &script_data, options,
        i::NOT_NATIVES_CODE);

src/compiler.cc

    // Compile the function and add it to the cache.
    ParseInfo parse_info(script);
    Zone compile_zone(isolate->allocator(), ZONE_NAME);
    CompilationInfo info(&compile_zone, &parse_info, Handle<JSFunction>::null());


Back in src/compiler.cc-info.cc:

    result = CompileToplevel(&info);

    (lldb) job *result
    0x17df0df309f1: [SharedFunctionInfo]
     - name = 0x1a7f12d82471 <String[0]: >
     - formal_parameter_count = 0
     - expected_nof_properties = 10
     - ast_node_count = 23
     - instance class name = #Object

     - code = 0x1d8484d3661 <Code: BUILTIN>
     - source code = function bajja(a, b, c) {
      var d = c - 100;
      return a + d * b;
    }

    var result = bajja(2, 2, 150);
    print(result);

     - anonymous expression
     - function token position = -1
     - start position = 0
     - end position = 114
     - no debug info
     - length = 0
     - optimized_code_map = 0x1a7f12d82241 <FixedArray[0]>
     - feedback_metadata = 0x17df0df30d09: [FeedbackMetadata]
     - length: 3
     - slot_count: 11
     Slot #0 LOAD_GLOBAL_NOT_INSIDE_TYPEOF_IC
     Slot #2 kCreateClosure
     Slot #3 LOAD_GLOBAL_NOT_INSIDE_TYPEOF_IC
     Slot #5 CALL_IC
     Slot #7 CALL_IC
     Slot #9 LOAD_GLOBAL_NOT_INSIDE_TYPEOF_IC

     - bytecode_array = 0x17df0df30c61


Back in d8.cc:

    maybe_result = script->Run(realm);


src/api.cc

    auto fun = i::Handle<i::JSFunction>::cast(Utils::OpenHandle(this));

    (lldb) job *fun
    0x17df0df30e01: [Function]
     - map = 0x19cfe0003859 [FastProperties]
     - prototype = 0x17df0df043b1
     - elements = 0x1a7f12d82241 <FixedArray[0]> [FAST_HOLEY_ELEMENTS]
     - initial_map =
     - shared_info = 0x17df0df309f1 <SharedFunctionInfo>
     - name = 0x1a7f12d82471 <String[0]: >
     - formal_parameter_count = 0
     - context = 0x17df0df03bf9 <FixedArray[245]>
     - feedback vector cell = 0x17df0df30ed1 Cell for 0x17df0df30e49 <FixedArray[13]>
     - code = 0x1d8484d3661 <Code: BUILTIN>
     - properties = 0x1a7f12d82241 <FixedArray[0]> {
        #length: 0x2c35a5718089 <AccessorInfo> (const accessor descriptor)
        #name: 0x2c35a57180f9 <AccessorInfo> (const accessor descriptor)
        #arguments: 0x2c35a5718169 <AccessorInfo> (const accessor descriptor)
        #caller: 0x2c35a57181d9 <AccessorInfo> (const accessor descriptor)
        #prototype: 0x2c35a5718249 <AccessorInfo> (const accessor descriptor)

      }

    i::Handle<i::Object> receiver = isolate->global_proxy();
    Local<Value> result;
    has_pending_exception = !ToLocal<Value>(i::Execution::Call(isolate, fun, receiver, 0, nullptr), &result);

src/execution.cc

### Zone
Taken directly from src/zone/zone.h:
```
// The Zone supports very fast allocation of small chunks of
// memory. The chunks cannot be deallocated individually, but instead
// the Zone supports deallocating all chunks in one fast
// operation. The Zone is used to hold temporary data structures like
// the abstract syntax tree, which is deallocated after compilation.
```



### V8 flags

    $ ./d8 --help

### d8

    (lldb) br s -f d8.cc -l 2935

    return v8::Shell::Main(argc, argv);

    api.cc:6112
    i::ReadNatives();
    natives-external.cc

### v8::String::NewFromOneByte
So I was a little confused when I first read this function name and thought it
had something to do with the length of the string. But the byte is the type
of the chars that make up the string.
For example, a one byte char would be reinterpreted as uint8_t:

    const char* data

    reinterpret_cast<const uint8_t*>(data)


#### Tasks
* gdbinit has been updated. Check if there is something that should be ported to lldbinit


### Invocation walkthrough 
This section will go through calling a Script to understand what happens in V8.

I'll be using [run-scripts.cc](./run-scripts.cc) as the example for this.

    $ lldb -- ./run-scripts
    (lldb) br s -n main

I'll step through until the following call:

    script->Run(context).ToLocalChecked();

So, Script::Run is defined in api.cc
First things that happens in this function is a macro:

    PREPARE_FOR_EXECUTION_WITH_CONTEXT_IN_RUNTIME_CALL_STATS_SCOPE(
         "v8", 
         "V8.Execute", 
         context, 
         Script, 
         Run, 
         MaybeLocal<Value>(),
         InternalEscapableScope, 
    true);
    TRACE_EVENT_CALL_STATS_SCOPED(isolate, category, name);
    PREPARE_FOR_EXECUTION_GENERIC(isolate, context, class_name, function_name, \
        bailout_value, HandleScopeClass, do_callback);

So, what does the preprocessor replace this with then:

    auto isolate = context.IsEmpty() ? i::Isolate::Current()                               : reinterpret_cast<i::Isolate*>(context->GetIsolate());

I'm skipping TRACE_EVENT_CALL_STATS_SCOPED for now.
`PREPARE_FOR_EXECUTION_GENERIC` will be replaced with:

    if (IsExecutionTerminatingCheck(isolate)) {                        \
      return bailout_value;                                            \
    }                                                                  \
    HandleScopeClass handle_scope(isolate);                            \
    CallDepthScope<do_callback> call_depth_scope(isolate, context);    \
    LOG_API(isolate, class_name, function_name);                       \
    ENTER_V8_DO_NOT_USE(isolate);                                      \
    bool has_pending_exception = false

 


    auto fun = i::Handle<i::JSFunction>::cast(Utils::OpenHandle(this));

    (lldb) job *fun
    0x33826912c021: [Function]
     - map = 0x1d0656c03599 [FastProperties]
     - prototype = 0x338269102e69
     - elements = 0x35190d902241 <FixedArray[0]> [FAST_HOLEY_ELEMENTS]
     - initial_map =
     - shared_info = 0x33826912bc11 <SharedFunctionInfo>
     - name = 0x35190d902471 <String[0]: >
     - formal_parameter_count = 0
     - context = 0x338269102611 <FixedArray[265]>
     - feedback vector cell = 0x33826912c139 <Cell value= 0x33826912c069 <FixedArray[24]>>
     - code = 0x1319e25fcf21 <Code BUILTIN>
     - properties = 0x35190d902241 <FixedArray[0]> {
        #length: 0x2e9d97ce68b1 <AccessorInfo> (const accessor descriptor)
        #name: 0x2e9d97ce6921 <AccessorInfo> (const accessor descriptor)
        #arguments: 0x2e9d97ce6991 <AccessorInfo> (const accessor descriptor)
        #caller: 0x2e9d97ce6a01 <AccessorInfo> (const accessor descriptor)
        #prototype: 0x2e9d97ce6a71 <AccessorInfo> (const accessor descriptor)
     }

The code for i::JSFunction is generated in src/api.h. Lets take a closer look at this.

    #define DECLARE_OPEN_HANDLE(From, To) \
      static inline v8::internal::Handle<v8::internal::To> \
      OpenHandle(const From* that, bool allow_empty_handle = false);

    OPEN_HANDLE_LIST(DECLARE_OPEN_HANDLE)

OPEN_HANDLE_LIST looks like this:

    #define OPEN_HANDLE_LIST(V)                    \
    ....
    V(Script, JSFunction)                        \ 

So lets expand this for JSFunction and it should become:

      static inline v8::internal::Handle<v8::internal::JSFunction> \
        OpenHandle(const Script* that, bool allow_empty_handle = false);

So there will be an function named OpenHandle that will take a const pointer to Script.

A little further down in src/api.h there is another macro which looks like this:

    OPEN_HANDLE_LIST(MAKE_OPEN_HANDLE)

MAKE_OPEN_HANDLE:
```c++
    #define MAKE_OPEN_HANDLE(From, To)
      v8::internal::Handle<v8::internal::To> Utils::OpenHandle( 
      const v8::From* that, bool allow_empty_handle) {         
      return v8::internal::Handle<v8::internal::To>(                         
        reinterpret_cast<v8::internal::Address*>(const_cast<v8::From*>(that))); 
      }
```
And remember that JSFunction is included in the `OPEN_HANDLE_LIST` so there will
be the following in the source after the preprocessor has processed this header:
A concrete example would look like this:
```c++
v8::internal::Handle<v8::internal::JSFunction> Utils::OpenHandle(
    const v8::Script* that, bool allow_empty_handle) {
  return v8::internal::Handle<v8::internal::JSFunction>(
      reinterpret_cast<v8::internal::Address*>(const_cast<v8::Script*>(that))); }
```

You can inspect the output of the preprocessor using:
```console
$ clang++ -I./out/x64.release/gen -I. -I./include -E src/api/api-inl.h > api-inl.output
```


So where is JSFunction declared? 
It is defined in objects.h





## Ignition interpreter
User JavaScript also needs to have bytecode generated for them and they also use
the C++ DLS and use the CodeStubAssembler -> CodeAssembler -> RawMachineAssembler
just like builtins.

## C++ Domain Specific Language (DLS)


#### Build failure
After rebasing I've seen the following issue:

    $ ninja -C out/Debug chrome
    ninja: Entering directory `out/Debug'
    ninja: error: '../../chrome/renderer/resources/plugins/plugin_delay.html', needed by 'gen/chrome/grit/renderer_resources.h', missing and no known rule to make it

The "solution" was to remove the out directory and rebuild.

### Tasks
To find suitable task you can use `label:HelpWanted` at [bugs.chromium.org](https://bugs.chromium.org/p/v8/issues/list?can=2&q=label%3AHelpWanted+&colspec=ID+Type+Status+Priority+Owner+Summary+HW+OS+Component+Stars&x=priority&y=owner&cells=ids).


### OpenHandle
What does this call do: 

    Utils::OpenHandle(*(source->source_string));

    OPEN_HANDLE_LIST(MAKE_OPEN_HANDLE)

Which is a macro defined in src/api.h:

    #define MAKE_OPEN_HANDLE(From, To)                                             \
      v8::internal::Handle<v8::internal::To> Utils::OpenHandle(                    \
          const v8::From* that, bool allow_empty_handle) {                         \
      DCHECK(allow_empty_handle || that != NULL);                                \
      DCHECK(that == NULL ||                                                     \
           (*reinterpret_cast<v8::internal::Object* const*>(that))->Is##To()); \
      return v8::internal::Handle<v8::internal::To>(                             \
          reinterpret_cast<v8::internal::To**>(const_cast<v8::From*>(that)));    \
    }

    OPEN_HANDLE_LIST(MAKE_OPEN_HANDLE)

If we take a closer look at the macro is should expand to something like this in our case:

     v8::internal::Handle<v8::internal::To> Utils::OpenHandle(const v8:String* that, false) {
       DCHECK(allow_empty_handle || that != NULL);                                \
       DCHECK(that == NULL ||                                                     \
           (*reinterpret_cast<v8::internal::Object* const*>(that))->IsString()); \
       return v8::internal::Handle<v8::internal::String>(                             \
          reinterpret_cast<v8::internal::String**>(const_cast<v8::String*>(that)));    \
     }

So this is returning a new v8::internal::Handle, the constructor is defined in src/handles.h:95.
     
src/objects.cc
Handle<WeakFixedArray> WeakFixedArray::Add(Handle<Object> maybe_array,
10167                                            Handle<HeapObject> value,
10168                                            int* assigned_index) {
Notice the name of the first parameter `maybe_array` but it is not of type maybe?

### Context
JavaScript provides a set of builtin functions and objects. These functions and objects can be changed by user code. Each context
is separate collection of these objects and functions.

And internal::Context is declared in `deps/v8/src/contexts.h` and extends FixedArray
```console
class Context: public FixedArray {
```

A Context can be create by calling:
```console
const v8::HandleScope handle_scope(isolate_);
Handle<Context> context = Context::New(isolate_,
                                       nullptr,
                                       v8::Local<v8::ObjectTemplate>());
```
`Context::New` can be found in `src/api.cc:6405`:
```c++
Local<Context> v8::Context::New(
    v8::Isolate* external_isolate, v8::ExtensionConfiguration* extensions,
    v8::MaybeLocal<ObjectTemplate> global_template,
    v8::MaybeLocal<Value> global_object,
    DeserializeInternalFieldsCallback internal_fields_deserializer) {
  return NewContext(external_isolate, extensions, global_template,
                    global_object, 0, internal_fields_deserializer);
}
```
The declaration of this function can be found in `include/v8.h`:
```c++
static Local<Context> New(
      Isolate* isolate, ExtensionConfiguration* extensions = NULL,
      MaybeLocal<ObjectTemplate> global_template = MaybeLocal<ObjectTemplate>(),
      MaybeLocal<Value> global_object = MaybeLocal<Value>(),
      DeserializeInternalFieldsCallback internal_fields_deserializer =
          DeserializeInternalFieldsCallback());
```
So we can see the reason why we did not have to specify `internal_fields_deserialize`.
What is `ExtensionConfiguration`?  
This class can be found in `include/v8.h` and only has two members, a count of the extension names 
and an array with the names.

If specified these will be installed by `Boostrapper::InstallExtensions` which will delegate to 
`Genesis::InstallExtensions`, both can be found in `src/boostrapper.cc`.
Where are extensions registered?   
This is done once per process and called from `V8::Initialize()`:
```c++
void Bootstrapper::InitializeOncePerProcess() {
  free_buffer_extension_ = new FreeBufferExtension;
  v8::RegisterExtension(free_buffer_extension_);
  gc_extension_ = new GCExtension(GCFunctionName());
  v8::RegisterExtension(gc_extension_);
  externalize_string_extension_ = new ExternalizeStringExtension;
  v8::RegisterExtension(externalize_string_extension_);
  statistics_extension_ = new StatisticsExtension;
  v8::RegisterExtension(statistics_extension_);
  trigger_failure_extension_ = new TriggerFailureExtension;
  v8::RegisterExtension(trigger_failure_extension_);
  ignition_statistics_extension_ = new IgnitionStatisticsExtension;
  v8::RegisterExtension(ignition_statistics_extension_);
}
```
The extensions can be found in `src/extensions`. You register your own extensions and an example of this
can be found in [test/context_test.cc](./test/context_test.cc).


```console
(lldb) br s -f node.cc -l 4439
(lldb) expr context->length()
(int) $522 = 281
```
This output was taken

Creating a new Context is done by `v8::CreateEnvironment`
```console
(lldb) br s -f api.cc -l 6565
```
```c++
InvokeBootstrapper<ObjectType> invoke;
   6635    result =
-> 6636        invoke.Invoke(isolate, maybe_proxy, proxy_template, extensions,
   6637                      context_snapshot_index, embedder_fields_deserializer);
```
This will later end up in `Snapshot::NewContextFromSnapshot`:
```c++
Vector<const byte> context_data =
      ExtractContextData(blob, static_cast<uint32_t>(context_index));
  SnapshotData snapshot_data(context_data);

  MaybeHandle<Context> maybe_result = PartialDeserializer::DeserializeContext(
      isolate, &snapshot_data, can_rehash, global_proxy,
      embedder_fields_deserializer);
```
So we can see here that the Context is deserialized from the snapshot. What does the Context contain at this stage:
```console
(lldb) expr result->length()
(int) $650 = 281
(lldb) expr result->Print()
// not inlcuding the complete output
```
Lets take a look at an entry:
```console
(lldb) expr result->get(0)->Print()
0xc201584331: [Function] in OldSpace
 - map = 0xc24c002251 [FastProperties]
 - prototype = 0xc201584371
 - elements = 0xc2b2882251 <FixedArray[0]> [HOLEY_ELEMENTS]
 - initial_map =
 - shared_info = 0xc2b2887521 <SharedFunctionInfo>
 - name = 0xc2b2882441 <String[0]: >
 - formal_parameter_count = -1
 - kind = [ NormalFunction ]
 - context = 0xc201583a59 <FixedArray[281]>
 - code = 0x2df1f9865a61 <Code BUILTIN>
 - source code = () {}
 - properties = 0xc2b2882251 <FixedArray[0]> {
    #length: 0xc2cca83729 <AccessorInfo> (const accessor descriptor)
    #name: 0xc2cca83799 <AccessorInfo> (const accessor descriptor)
    #arguments: 0xc201587fd1 <AccessorPair> (const accessor descriptor)
    #caller: 0xc201587fd1 <AccessorPair> (const accessor descriptor)
    #constructor: 0xc201584c29 <JSFunction Function (sfi = 0xc2b28a6fb1)> (const data descriptor)
    #apply: 0xc201588079 <JSFunction apply (sfi = 0xc2b28a7051)> (const data descriptor)
    #bind: 0xc2015880b9 <JSFunction bind (sfi = 0xc2b28a70f1)> (const data descriptor)
    #call: 0xc2015880f9 <JSFunction call (sfi = 0xc2b28a7191)> (const data descriptor)
    #toString: 0xc201588139 <JSFunction toString (sfi = 0xc2b28a7231)> (const data descriptor)
    0xc2b28bc669 <Symbol: Symbol.hasInstance>: 0xc201588179 <JSFunction [Symbol.hasInstance] (sfi = 0xc2b28a72d1)> (const data descriptor)
 }

 - feedback vector: not available
```
So we can see that this is of type `[Function]` which we can cast using:
```
(lldb) expr JSFunction::cast(result->get(0))->code()->Print()
0x2df1f9865a61: [Code]
kind = BUILTIN
name = EmptyFunction
```

```console
(lldb) expr JSFunction::cast(result->closure())->Print()
0xc201584331: [Function] in OldSpace
 - map = 0xc24c002251 [FastProperties]
 - prototype = 0xc201584371
 - elements = 0xc2b2882251 <FixedArray[0]> [HOLEY_ELEMENTS]
 - initial_map =
 - shared_info = 0xc2b2887521 <SharedFunctionInfo>
 - name = 0xc2b2882441 <String[0]: >
 - formal_parameter_count = -1
 - kind = [ NormalFunction ]
 - context = 0xc201583a59 <FixedArray[281]>
 - code = 0x2df1f9865a61 <Code BUILTIN>
 - source code = () {}
 - properties = 0xc2b2882251 <FixedArray[0]> {
    #length: 0xc2cca83729 <AccessorInfo> (const accessor descriptor)
    #name: 0xc2cca83799 <AccessorInfo> (const accessor descriptor)
    #arguments: 0xc201587fd1 <AccessorPair> (const accessor descriptor)
    #caller: 0xc201587fd1 <AccessorPair> (const accessor descriptor)
    #constructor: 0xc201584c29 <JSFunction Function (sfi = 0xc2b28a6fb1)> (const data descriptor)
    #apply: 0xc201588079 <JSFunction apply (sfi = 0xc2b28a7051)> (const data descriptor)
    #bind: 0xc2015880b9 <JSFunction bind (sfi = 0xc2b28a70f1)> (const data descriptor)
    #call: 0xc2015880f9 <JSFunction call (sfi = 0xc2b28a7191)> (const data descriptor)
    #toString: 0xc201588139 <JSFunction toString (sfi = 0xc2b28a7231)> (const data descriptor)
    0xc2b28bc669 <Symbol: Symbol.hasInstance>: 0xc201588179 <JSFunction [Symbol.hasInstance] (sfi = 0xc2b28a72d1)> (const data descriptor)
 }

 - feedback vector: not available
```
So this is the JSFunction associated with the deserialized context. Not sure what this is about as looking at the source code it looks like
an empty function. A function can also be set on the context so I'm guessing that this give access to the function of a context once set.
Where is function set, well it is probably deserialized but we can see it be used in `deps/v8/src/bootstrapper.cc`:
```c++
{
  Handle<JSFunction> function = SimpleCreateFunction(isolate, factory->empty_string(), Builtins::kAsyncFunctionAwaitCaught, 2, false);
  native_context->set_async_function_await_caught(*function);
}
​```console
(lldb) expr isolate()->builtins()->builtin_handle(Builtins::Name::kAsyncFunctionAwaitCaught)->Print()
```

`Context::Scope` is a RAII class used to Enter/Exit a context. Lets take a closer look at `Enter`:
```c++
void Context::Enter() {
  i::Handle<i::Context> env = Utils::OpenHandle(this);
  i::Isolate* isolate = env->GetIsolate();
  ENTER_V8_NO_SCRIPT_NO_EXCEPTION(isolate);
  i::HandleScopeImplementer* impl = isolate->handle_scope_implementer();
  impl->EnterContext(env);
  impl->SaveContext(isolate->context());
  isolate->set_context(*env);
}
```
So the current context is saved and then the this context `env` is set as the current on the isolate.
`EnterContext` will push the passed-in context (deps/v8/src/api.cc):
```c++
void HandleScopeImplementer::EnterContext(Handle<Context> context) {
  entered_contexts_.push_back(*context);
}
...
DetachableVector<Context*> entered_contexts_;
```
```c++
DetachableVector is a delegate/adaptor with some additonaly features on a std::vector.
Handle<Context> context1 = NewContext(isolate);
Handle<Context> context2 = NewContext(isolate);
Context::Scope context_scope1(context1);        // entered_contexts_ [context1], saved_contexts_[isolateContext]
Context::Scope context_scope2(context2);        // entered_contexts_ [context1, context2], saved_contexts[isolateContext, context1]
```

Now, `SaveContext` is using the current context, not `this` context (`env`) and pushing that to the end of the saved_contexts_ vector.
We can look at this as we entered context_scope2 from context_scope1:


And `Exit` looks like:
```c++
void Context::Exit() {
  i::Handle<i::Context> env = Utils::OpenHandle(this);
  i::Isolate* isolate = env->GetIsolate();
  ENTER_V8_NO_SCRIPT_NO_EXCEPTION(isolate);
  i::HandleScopeImplementer* impl = isolate->handle_scope_implementer();
  if (!Utils::ApiCheck(impl->LastEnteredContextWas(env),
                       "v8::Context::Exit()",
                       "Cannot exit non-entered context")) {
    return;
  }
  impl->LeaveContext();
  isolate->set_context(impl->RestoreContext());
}
```


#### EmbedderData
A context can have embedder data set on it. Like decsribed above a Context is
internally A FixedArray. `SetEmbedderData` in Context is implemented in `src/api.cc`:
```c++
const char* location = "v8::Context::SetEmbedderData()";
i::Handle<i::FixedArray> data = EmbedderDataFor(this, index, true, location);
i::Handle<i::FixedArray> data(env->embedder_data());
```
`location` is only used for logging and we can ignore it for now.
`EmbedderDataFor`:
```c++
i::Handle<i::Context> env = Utils::OpenHandle(context);
...
i::Handle<i::FixedArray> data(env->embedder_data());
```
We can find `embedder_data` in `src/contexts-inl.h`

```c++
#define NATIVE_CONTEXT_FIELD_ACCESSORS(index, type, name) \
  inline void set_##name(type* value);                    \
  inline bool is_##name(type* value) const;               \
  inline type* name() const;
  NATIVE_CONTEXT_FIELDS(NATIVE_CONTEXT_FIELD_ACCESSORS)
```
And `NATIVE_CONTEXT_FIELDS` in context.h:
```c++
#define NATIVE_CONTEXT_FIELDS(V)                                               \
  V(GLOBAL_PROXY_INDEX, JSObject, global_proxy_object)                         \
  V(EMBEDDER_DATA_INDEX, FixedArray, embedder_data)                            \
...

#define NATIVE_CONTEXT_FIELD_ACCESSORS(index, type, name) \
  void Context::set_##name(type* value) {                 \
    DCHECK(IsNativeContext());                            \
    set(index, value);                                    \
  }                                                       \
  bool Context::is_##name(type* value) const {            \
    DCHECK(IsNativeContext());                            \
    return type::cast(get(index)) == value;               \
  }                                                       \
  type* Context::name() const {                           \
    DCHECK(IsNativeContext());                            \
    return type::cast(get(index));                        \
  }
NATIVE_CONTEXT_FIELDS(NATIVE_CONTEXT_FIELD_ACCESSORS)
#undef NATIVE_CONTEXT_FIELD_ACCESSORS
```
So the preprocessor would expand this to:
```c++
FixedArray embedder_data() const;

void Context::set_embedder_data(FixedArray value) {
  DCHECK(IsNativeContext());
  set(EMBEDDER_DATA_INDEX, value);
}

bool Context::is_embedder_data(FixedArray value) const {
  DCHECK(IsNativeContext());
  return FixedArray::cast(get(EMBEDDER_DATA_INDEX)) == value;
}

FixedArray Context::embedder_data() const {
  DCHECK(IsNativeContext());
  return FixedArray::cast(get(EMBEDDER_DATA_INDEX));
}
```
We can take a look at the initial data:
```console
lldb) expr data->Print()
0x2fac3e896439: [FixedArray] in OldSpace
 - map = 0x2fac9de82341 <Map(HOLEY_ELEMENTS)>
 - length: 3
         0-2: 0x2fac1cb822e1 <undefined>
(lldb) expr data->length()
(int) $5 = 3
```
And after setting:
```console
(lldb) expr data->Print()
0x2fac3e896439: [FixedArray] in OldSpace
 - map = 0x2fac9de82341 <Map(HOLEY_ELEMENTS)>
 - length: 3
           0: 0x2fac20c866e1 <String[7]: embdata>
         1-2: 0x2fac1cb822e1 <undefined>

(lldb) expr v8::internal::String::cast(data->get(0))->Print()
"embdata"
```
This was taken while debugging [ContextTest::EmbedderData](./test/context_test.cc).

### ENTER_V8_FOR_NEW_CONTEXT
This macro is used in `CreateEnvironment` (src/api.cc) and the call in this function looks like this:
```c++
ENTER_V8_FOR_NEW_CONTEXT(isolate);
```


### Factory::NewMap
This section will take a look at the following call:
```c++
i::Handle<i::Map> map = factory->NewMap(i::JS_OBJECT_TYPE, 24);
```

Lets take a closer look at this function which can be found in `src/factory.cc`:
```
Handle<Map> Factory::NewMap(InstanceType type, int instance_size,
                            ElementsKind elements_kind,
                            int inobject_properties) {
  CALL_HEAP_FUNCTION(
      isolate(),
      isolate()->heap()->AllocateMap(type, instance_size, elements_kind,
                                     inobject_properties),
      Map);
}

```
If we take a look at factory.h we can see the default values for elements_kind and inobject_properties:
```c++
Handle<Map> NewMap(InstanceType type, int instance_size,
                     ElementsKind elements_kind = TERMINAL_FAST_ELEMENTS_KIND,
                     int inobject_properties = 0);
```
If we expand the CALL_HEAP_FUNCTION macro we will get:
```c++
    AllocationResult __allocation__ = isolate()->heap()->AllocateMap(type,
                                                                     instance_size,
                                                                     elements_kind,
                                                                     inobject_properties),
    Object* __object__ = nullptr;
    RETURN_OBJECT_UNLESS_RETRY(isolate(), Map)
    /* Two GCs before panicking.  In newspace will almost always succeed. */
    for (int __i__ = 0; __i__ < 2; __i__++) {
      (isolate())->heap()->CollectGarbage(
          __allocation__.RetrySpace(),
          GarbageCollectionReason::kAllocationFailure);
      __allocation__ = FUNCTION_CALL;
      RETURN_OBJECT_UNLESS_RETRY(isolate, Map)
    }
    (isolate())->counters()->gc_last_resort_from_handles()->Increment();
    (isolate())->heap()->CollectAllAvailableGarbage(
        GarbageCollectionReason::kLastResort);
    {
      AlwaysAllocateScope __scope__(isolate());
    t __allocation__ = isolate()->heap()->AllocateMap(type,
                                                      instance_size,
                                                      elements_kind,
                                                      inobject_properties),
    }
    RETURN_OBJECT_UNLESS_RETRY(isolate, Map)
    /* TODO(1181417): Fix this. */
    v8::internal::Heap::FatalProcessOutOfMemory("CALL_AND_RETRY_LAST", true);
    return Handle<Map>();
```
So, lets take a look at `isolate()->heap()->AllocateMap` in 'src/heap/heap.cc':
```c++
  HeapObject* result = nullptr;
  AllocationResult allocation = AllocateRaw(Map::kSize, MAP_SPACE);
```
`AllocateRaw` can be found in src/heap/heap-inl.h:
```c++
  bool large_object = size_in_bytes > kMaxRegularHeapObjectSize;
  HeapObject* object = nullptr;
  AllocationResult allocation;
  if (NEW_SPACE == space) {
    if (large_object) {
      space = LO_SPACE;
    } else {
      allocation = new_space_->AllocateRaw(size_in_bytes, alignment);
      if (allocation.To(&object)) {
        OnAllocationEvent(object, size_in_bytes);
      }
      return allocation;
    }
  }
 } else if (MAP_SPACE == space) {
    allocation = map_space_->AllocateRawUnaligned(size_in_bytes);
 }

```
```console
(lldb) expr large_object
(bool) $3 = false
(lldb) expr size_in_bytes
(int) $5 = 80
(lldb) expr map_space_
(v8::internal::MapSpace *) $6 = 0x0000000104700f60
```
`AllocateRawUnaligned` can be found in `src/heap/spaces-inl.h`
```c++
  HeapObject* object = AllocateLinearly(size_in_bytes);
```

### v8::internal::Object
Is an abstract super class for all classes in the object hierarch and both Smi and HeapObject
are subclasses of Object so there are no data members in object only functions.
For example:
```
  bool IsObject() const { return true; }
  INLINE(bool IsSmi() const
  INLINE(bool IsLayoutDescriptor() const
  INLINE(bool IsHeapObject() const
  INLINE(bool IsPrimitive() const
  INLINE(bool IsNumber() const
  INLINE(bool IsNumeric() const
  INLINE(bool IsAbstractCode() const
  INLINE(bool IsAccessCheckNeeded() const
  INLINE(bool IsArrayList() const
  INLINE(bool IsBigInt() const
  INLINE(bool IsUndefined() const
  INLINE(bool IsNull() const
  INLINE(bool IsTheHole() const
  INLINE(bool IsException() const
  INLINE(bool IsUninitialized() const
  INLINE(bool IsTrue() const
  INLINE(bool IsFalse() const
  ...
```

### v8::internal::Smi
Extends v8::internal::Object and are not allocated on the heap. There are no members as the
pointer itself is used to store the information.

### v8::internal::HeapObject
Is just a pointer type with functions but no members.
I've yet to find any types that have members. How are things stored?
For example, the Map* in HeapObject must be stored somewhere right?
```c++
inline Map* map() const;
inline void set_map(Map* value);
```
Lets take a look at the definition of `map` in objects-inl.h:
```c++
Map* HeapObject::map() const {
  return map_word().ToMap();
}

MapWord HeapObject::map_word() const {
  return MapWord(
      reinterpret_cast<uintptr_t>(RELAXED_READ_FIELD(this, kMapOffset)));
}
```

`RELAXED_READ_FIELD` will expand to:
```c++
  reinterpret_cast<Object*>(base::Relaxed_Load(
      reinterpret_cast<const base::AtomicWord*>(
          (reinterpret_cast<const byte*>(this) + offset - kHeapObjectTag)))

```

### Heap
Lets take a look when the heap is constructed by using test/heap_test and setting a break
point in Heap's constructor. 
```console
$ make test/heap_test
(lldb) br s -f v8_test_fixture.h -l 30
(lldb) r
```
So when is a Heap created?
`Heap heap_;` is a private member of Isolate so it's constructor will be called 
when a new Isolate is created:
```console
(lldb) bt
* thread #1: tid = 0xc5c373, 0x0000000100c70a65 libv8.dylib`v8::internal::Heap::Heap(this=0x0000000104001c20) + 21 at heap.cc:247, queue = 'com.apple.main-thread', stop reason = step over
  * frame #0: 0x0000000100c70a65 libv8.dylib`v8::internal::Heap::Heap(this=0x0000000104001c20) + 21 at heap.cc:247
    frame #1: 0x0000000100ecb542 libv8.dylib`v8::internal::Isolate::Isolate(this=0x0000000104001c00) + 82 at isolate.cc:2482
    frame #2: 0x0000000100ecc875 libv8.dylib`v8::internal::Isolate::Isolate(this=0x0000000104001c00) + 21 at isolate.cc:2544
    frame #3: 0x000000010011f110 libv8.dylib`v8::Isolate::Allocate() + 32 at api.cc:8204
    frame #4: 0x000000010011f6e1 libv8.dylib`v8::Isolate::New(params=0x0000000100076768) + 17 at api.cc:8275
    frame #5: 0x0000000100066233 heap_test`V8TestFixture::SetUp(this=0x0000000104901650) + 35 at v8_test_fixture.h:30
```
The constructor for Heap can be found in `src/heap/heap.cc`. Lets take a look at the fields of
a Heap instance (can be found in src/heap/heap.h);
```c++
Object* roots_[kRootListLength];
```
kRootListLength is an entry in the RootListIndex enum:
```console
(lldb) expr ::RootListIndex::kRootListLength
(int) $8 = 529
```
Notice that the types in this array is `Object*`. The `RootListIndex` enum is populated using a macro.
You can call the `root` function on a heap instace to get the object at that index:
```c++
Object* root(RootListIndex index) { 
  return roots_[index]; 
}
```

These are only the indexes into the array, the array itself has not been populated yet.
The array is created in Heap's constructor:
```c++
memset(roots_, 0, sizeof(roots_[0]) * kRootListLength);
```
And as we can see it is initially empty:
```console
(lldb) expr roots_
(v8::internal::Object *[529]) $10 = {
  [0] = 0x0000000000000000
  [1] = 0x0000000000000000
  [2] = 0x0000000000000000
  ...
  [529] = 0x0000000000000000
```
After returning from Heap's constructor we are back in Isolate's constructor.
An Isolate has a private member `ThreadLocalTop thread_local_top_;` which calls
`InitializeInternal` when constructed:
```c++
void ThreadLocalTop::InitializeInternal() {
  c_entry_fp_ = 0;
  c_function_ = 0;
  handler_ = 0;
#ifdef USE_SIMULATOR
  simulator_ = nullptr;
#endif
  js_entry_sp_ = kNullAddress;
  external_callback_scope_ = nullptr;
  current_vm_state_ = EXTERNAL;
  try_catch_handler_ = nullptr;
  context_ = nullptr;
  thread_id_ = ThreadId::Invalid();
  external_caught_exception_ = false;
  failed_access_check_callback_ = nullptr;
  save_context_ = nullptr;
  promise_on_stack_ = nullptr;

  // These members are re-initialized later after deserialization
  // is complete.
  pending_exception_ = nullptr;
  wasm_caught_exception_ = nullptr;
  rethrowing_message_ = false;
  pending_message_obj_ = nullptr;
  scheduled_exception_ = nullptr;
}
```
TODO: link these fields with entry code when v8 is about to call a builtin or javascript function.
Most of the rest of the initialisers set the members to null or equivalent. But lets take a look
at what the constructor does:
```c++
id_ = base::Relaxed_AtomicIncrement(&isolate_counter_, 1);
```
```console
(lldb) expr id_
(v8::base::Atomic32) $13 = 1
```
```c++
memset(isolate_addresses_, 0, sizeof(isolate_addresses_[0]) * (kIsolateAddressCount + 1));
```
What is `isolate_addresses_`?
```c++
Address isolate_addresses_[kIsolateAddressCount + 1];
```
`Address` can be found in `src/globals.h`:

```c++
typedef uintptr_t Address;
```

Also in `src/globals.h` we find:

```c++
#define FOR_EACH_ISOLATE_ADDRESS_NAME(C)                \
  C(Handler, handler)                                   \
  C(CEntryFP, c_entry_fp)                               \
  C(CFunction, c_function)                              \
  C(Context, context)                                   \
  C(PendingException, pending_exception)                \
  C(PendingHandlerContext, pending_handler_context)     \
  C(PendingHandlerCode, pending_handler_code)           \
  C(PendingHandlerOffset, pending_handler_offset)       \
  C(PendingHandlerFP, pending_handler_fp)               \
  C(PendingHandlerSP, pending_handler_sp)               \
  C(ExternalCaughtException, external_caught_exception) \
  C(JSEntrySP, js_entry_sp)

enum IsolateAddressId {
#define DECLARE_ENUM(CamelName, hacker_name) k##CamelName##Address,
  FOR_EACH_ISOLATE_ADDRESS_NAME(DECLARE_ENUM)
#undef DECLARE_ENUM
      kIsolateAddressCount
};
```
Will expand to:
```c++
enum IsolateAddressId {
  kHandlerAddress,
  kCEntryAddress,
  kCFunctionAddress,
  kContextAddress,
  kPendingExceptionAddress,
  kPendingHandlerAddress,
  kPendingHandlerCodeAddress,
  kPendingHandlerOffsetAddress,
  kPendingHandlerFPAddress,
  kPendingHandlerSPAddress,
  kExternalCaughtExceptionAddress,
  kJSEntrySPAddress,
  kIsolateAddressCount
};
```
Alright, so we know where the `kIsolateAddressCount` comes from and that memory is allocated
and set to zero for this in the Isolate constructor. Where then are these entries filled?
In isolate.cc when an Isolate is initialized by `bool Isolate::Init(StartupDeserializer* des)`:

```c++
#define ASSIGN_ELEMENT(CamelName, hacker_name)                  \
  isolate_addresses_[IsolateAddressId::k##CamelName##Address] = \
      reinterpret_cast<Address>(hacker_name##_address());
  FOR_EACH_ISOLATE_ADDRESS_NAME(ASSIGN_ELEMENT)
#undef ASSIGN_ELEMENT
```
```c++
  isolate_addressess_[IsolateAddressId::kHandlerAddress] = reinterpret_cast<Address>(handler_address());
  isolate_addressess_[IsolateAddressId::kCEntryAddress] = reinterpret_cast<Address>(c_entry_fp_address());
  isolate_addressess_[IsolateAddressId::kFunctionAddress] = reinterpret_cast<Address>(c_function_address());
  isolate_addressess_[IsolateAddressId::kContextAddress] = reinterpret_cast<Address>(context_address());
  isolate_addressess_[IsolateAddressId::kPendingExceptionAddress] = reinterpret_cast<Address>(pending_exception_address());
  isolate_addressess_[IsolateAddressId::kPendingHandlerContextAddress] = reinterpret_cast<Address>(pending_handler_context_address());
  isolate_addressess_[IsolateAddressId::kPendingHandlerCodeAddress] = reinterpret_cast<Address>(pending_handler_code_address());
  isolate_addressess_[IsolateAddressId::kPendingHandlerOffsetAddress] = reinterpret_cast<Address>(pending_handler_offset_address());
  isolate_addressess_[IsolateAddressId::kPendingHandlerFPAddress] = reinterpret_cast<Address>(pending_handler_fp_address());
  isolate_addressess_[IsolateAddressId::kPendingHandlerSPAddress] = reinterpret_cast<Address>(pending_handler_sp_address());
  isolate_addressess_[IsolateAddressId::kExternalCaughtExceptionAddress] = reinterpret_cast<Address>(external_caught_exception_address());
  isolate_addressess_[IsolateAddressId::kJSEntrySPAddress] = reinterpret_cast<Address>(js_entry_sp);
```

So where does `handler_address()` and the rest of those functions come from?
This is defined in `isolate.h`:
```c++
inline Address* handler_address() { return &thread_local_top_.handler_; }
inline Address* c_entry_fp_address() { return &thread_local_top_.c_entry_fp_; }
inline Address* c_function_address() { return &thread_local_top_.c_function_; }
Context** context_address() { return &thread_local_top_.context_; }
inline Address* js_entry_sp_address() { return &thread_local_top_.js_entry_sp_; }
Address pending_message_obj_address() { return reinterpret_cast<Address>(&thread_local_top_.pending_message_obj_); }

THREAD_LOCAL_TOP_ADDRESS(Context*, pending_handler_context)
THREAD_LOCAL_TOP_ADDRESS(Code*, pending_handler_code)
THREAD_LOCAL_TOP_ADDRESS(intptr_t, pending_handler_offset)
THREAD_LOCAL_TOP_ADDRESS(Address, pending_handler_fp)
THREAD_LOCAL_TOP_ADDRESS(Address, pending_handler_sp)


#define THREAD_LOCAL_TOP_ADDRESS(type, name) \
  type* name##_address() { return &thread_local_top_.name##_; }

Context* pending_handler_context_address() { return &thread_local_top_.pending_handler_context_;
```

When is `Isolate::Init` called?
```console
// v8_test_fixture.h
virtual void SetUp() {
  isolate_ = v8::Isolate::New(create_params_);
}

Isolate* Isolate::New(const Isolate::CreateParams& params) {
  Isolate* isolate = Allocate();
  Initialize(isolate, params);
  return isolate;
}
```
Lets take a closer look at `Initialize`. First the array buffer allocator is set on the 
internal isolate which is just a setter and not that interesting.

Next the snapshot blob is set (just showing the path we are taking):
```c++
i_isolate->set_snapshot_blob(i::Snapshot::DefaultSnapshotBlob());
```
Now, `i_isolate->set_snapshot_blob` is generated by a macro. The macro can be found in
`src/isolate.h`:

```c++
  // Accessors.
#define GLOBAL_ACCESSOR(type, name, initialvalue)                       \
  inline type name() const {                                            \
    DCHECK(OFFSET_OF(Isolate, name##_) == name##_debug_offset_);        \
    return name##_;                                                     \
  }                                                                     \
  inline void set_##name(type value) {                                  \
    DCHECK(OFFSET_OF(Isolate, name##_) == name##_debug_offset_);        \
    name##_ = value;                                                    \
  }
  ISOLATE_INIT_LIST(GLOBAL_ACCESSOR)
#undef GLOBAL_ACCESSOR
```
In this case the entry in ISOLATE_INIT_LIST is:
```c++
V(const v8::StartupData*, snapshot_blob, nullptr)
```
So this will generate two functions:
```c++
  inline type snapshot_blob() const {
    DCHECK(OFFSET_OF(Isolate, snapshot_blob_) == snapshot_blob_debug_offset_);
    return snapshot_blob_;
  }

  inline void set_snapshot_blob(const v8::StartupData* value) {
    DCHECK(OFFSET_OF(Isolate, snapshot_blob_) == snapshot_blob_debug_offset_);
    snapshot_blob_ = value;
  }
```
Next we have:
```c++
if (params.entry_hook || !i::Snapshot::Initialize(i_isolate)) {
...
```
```c++
bool Snapshot::Initialize(Isolate* isolate) {
  if (!isolate->snapshot_available()) return false;
  base::ElapsedTimer timer;
  if (FLAG_profile_deserialization) timer.Start();

  const v8::StartupData* blob = isolate->snapshot_blob();
  CheckVersion(blob);
  Vector<const byte> startup_data = ExtractStartupData(blob);
  SnapshotData startup_snapshot_data(startup_data);
  Vector<const byte> builtin_data = ExtractBuiltinData(blob);
  BuiltinSnapshotData builtin_snapshot_data(builtin_data);
  StartupDeserializer deserializer(&startup_snapshot_data,
                                   &builtin_snapshot_data);
  deserializer.SetRehashability(ExtractRehashability(blob));
  bool success = isolate->Init(&deserializer);
  if (FLAG_profile_deserialization) {
    double ms = timer.Elapsed().InMillisecondsF();
    int bytes = startup_data.length();
    PrintF("[Deserializing isolate (%d bytes) took %0.3f ms]\n", bytes, ms);
  }
  return success;
}
```
Lets take a closer look at `bool success = isolate->Init(&deserializer);`:
```c++
#define ASSIGN_ELEMENT(CamelName, hacker_name)                  \
  isolate_addresses_[IsolateAddressId::k##CamelName##Address] = \
      reinterpret_cast<>(hacker_name##_address());
  FOR_EACH_ISOLATE_ADDRESS_NAME(ASSIGN_ELEMENT)
#undef ASSIGN_ELEMENT
```
Like mentioned before we have the isolate_address_ array that gets populated with 
pointers.
```c++
global_handles_ = new GlobalHandles(this);
```
```c++
GlobalHandles::GlobalHandles(Isolate* isolate)
    : isolate_(isolate),
      number_of_global_handles_(0),
      first_block_(nullptr),
      first_used_block_(nullptr),
      first_free_(nullptr),
      post_gc_processing_count_(0),
      number_of_phantom_handle_resets_(0) {}
```
TODO: take a closer look at the GlobalHandles class.
```c++
  global_handles_ = new GlobalHandles(this);
  eternal_handles_ = new EternalHandles();
  bootstrapper_ = new Bootstrapper(this);
  handle_scope_implementer_ = new HandleScopeImplementer(this);
  load_stub_cache_ = new StubCache(this);
  store_stub_cache_ = new StubCache(this);
...
  call_descriptor_data_ =
      new CallInterfaceDescriptorData[CallDescriptors::NUMBER_OF_DESCRIPTORS];
...
  if (!heap_.SetUp()) {
    V8::FatalProcessOutOfMemory(this, "heap setup");
    return false;
  }
```
Now, lets look at `head_.SetUp()`:
```c++
  if (!configured_) {
    if (!ConfigureHeapDefault()) return false;
  }
```
```c++
bool Heap::ConfigureHeapDefault() { return ConfigureHeap(0, 0, 0); }
```
Looking at the code this sets up the generation in the heap.

```c++
// Initialize the interface descriptors ahead of time.
#define INTERFACE_DESCRIPTOR(Name, ...) \
  { Name##Descriptor(this); }
  INTERFACE_DESCRIPTOR_LIST(INTERFACE_DESCRIPTOR)
#undef INTERFACE_DESCRIPTOR
```
`INTERFACE_DESCRIPTOR_LIST` can be found in `src/interface-descriptors.h`
```c++
#define INTERFACE_DESCRIPTOR_LIST(V)  \
  V(Void)                             \
  V(ContextOnly)                      \
  V(Load)                             \
  ...
```
So the first entry `Void` will expand to:
```c++
  { VoidDescriptor(this); }
```
```c++
class V8_EXPORT_PRIVATE VoidDescriptor : public CallInterfaceDescriptor {
 public:
  DECLARE_DESCRIPTOR(VoidDescriptor, CallInterfaceDescriptor)
};
```

```c++
InitializeThreadLocal();
bootstrapper_->Initialize(create_heap_objects);
setup_delegate_->SetupBuiltins(this);


isolate->heap()->IterateSmiRoots(this);

#define SMI_ROOT_LIST(V)                                                       \
  V(Smi, stack_limit, StackLimit)                                              \
  V(Smi, real_stack_limit, RealStackLimit)                                     \
  V(Smi, last_script_id, LastScriptId)                                         \
  V(Smi, last_debugging_id, LastDebuggingId)                                   \
  V(Smi, hash_seed, HashSeed)                                                  \
  V(Smi, next_template_serial_number, NextTemplateSerialNumber)                \
  V(Smi, arguments_adaptor_deopt_pc_offset, ArgumentsAdaptorDeoptPCOffset)     \
  V(Smi, construct_stub_create_deopt_pc_offset,                                \
    ConstructStubCreateDeoptPCOffset)                                          \
  V(Smi, construct_stub_invoke_deopt_pc_offset,                                \
    ConstructStubInvokeDeoptPCOffset)                                          \
  V(Smi, interpreter_entry_return_pc_offset, InterpreterEntryReturnPCOffset)
```

The Isolate class extends HiddenFactory (src/execution/isolate.h):
```c++
// HiddenFactory exists so Isolate can privately inherit from it without making
// Factory's members available to Isolate directly.
class V8_EXPORT_PRIVATE HiddenFactory : private Factory {};

class Isolate : private HiddenFactory {
  ...
}
```
So `isolate->factory()` looks like this:
```c++
v8::internal::Factory* factory() {
  // Upcast to the privately inherited base-class using c-style casts to avoid
  // undefined behavior (as static_cast cannot cast across private bases).
  return (v8::internal::Factory*)this;  // NOLINT(readability/casting)
}
```
The factory class can be found in `src/heap/factory`.

Next, lets take a closer look at NewMap:
```c++
i::Handle<i::Map> map = factory->NewMap(i::JS_OBJECT_TYPE, 24);
```
What we are doing is creating a new Map, or hidden class. Every HeapObject has a Map
which can be set/get. The header for map can be found in `src/objects/map.h`.
The map contains information about its size and how to iterate over it which is required
for GC.

```c++
  HeapObject* result = isolate()->heap()->AllocateRawWithRetryOrFail(Map::kSize, MAP_SPACE);

  allocation = map_space_->AllocateRawUnaligned(size_in_bytes);

```
In `src/heap/spaces-inl.h` we find `AllocateLinearly`:
```c++
  HeapObject* object = AllocateLinearly(size_in_bytes);
```
`AllocateLinearly` in `spaces-inl.h`:
```c++
  Address current_top = allocation_info_.top();

```
```console
(lldb) expr allocation_info_
(v8::internal::LinearAllocationArea) $13 = (top_ = 16313499525632, limit_ = 16313500041216)
```
`LinearAllocationArea` can also be found in src/heap/spaces.h and is pretty simple with only
two member fields:
```c++
Address top_;
Address limit_;
```
Back to `AllocateLinearly`:
```c++
HeapObject* PagedSpace::AllocateLinearly(int size_in_bytes) {
  Address current_top = allocation_info_.top();
  Address new_top = current_top + size_in_bytes;
  DCHECK_LE(new_top, allocation_info_.limit());
  allocation_info_.set_top(new_top);
  return HeapObject::FromAddress(current_top);
}
```
We can see that this function takes the current allocation top and adds the
size to that to get a new allaction top which is then set. A pointer to a heap
object is returned which points to the old top which makes sense, as it has to point
to the beginning of the instance in memory. 

```c++
  // Converts an address to a HeapObject pointer.
  static inline HeapObject* FromAddress(Address address) {
    DCHECK_TAG_ALIGNED(address);
    return reinterpret_cast<HeapObject*>(address + kHeapObjectTag);
  }
```
`kHeapObjectTag` can be found in `include/v8.h`:
```c++
const int kHeapObjectTag = 1;
const int kWeakHeapObjectTag = 3;
const int kHeapObjectTagSize = 2;
const intptr_t kHeapObjectTagMask = (1 << kHeapObjectTagSize) - 1;
```
```console
(lldb) expr ::kHeapObjectTag
(const int) $17 = 1
```
So in:
```c++
return reinterpret_cast<HeapObject*>(address + kHeapObjectTag);
```
`address` is the current_top and we are adding 1 to it before casting this memory location to be used as the pointer to a HeapObject.
To recap, at this stage we are creating a Map instance which is a type of HeapObject:
```c++
class Map : public HeapObject {
  ...
}
```
But we have not created an instance yet, instead only reserved memory by moving the current top allowing room for the new Map.
```console
(lldb) expr address
(v8::internal::Address) $18 = 16313499525632
(lldb) expr address + ::kHeapObjectTag
(unsigned long) $19 = 16313499525633
```

So back in `Factory::NewMap` we have:
```c++
  HeapObject* result = isolate()->heap()->AllocateRawWithRetryOrFail(Map::kSize, MAP_SPACE);
  result->set_map_after_allocation(*meta_map(), SKIP_WRITE_BARRIER);
```

TODO: split `InitializeMap` so it is a little more readable here.
```c++
  return handle(InitializeMap(Map::cast(result), type, instance_size,
                              elements_kind, inobject_properties),
```


### Spaces
From `src/heap/spaces.h`

```console
Young Generation                  Old Generation
+-------------+------------+  +-----------+-------------+
| scavenger   |            |  | map space | old object  |
+-------------+------------+  +-----------+-------------+
```
The `map space` contains only map objects.
The old and map spaces consist of list of pages. 
The Page class can be found in `src/heap/spaces.h`
```c++
class Page : public MemoryChunk {
}
```


In our case the calling v8::Isolate::New which is done by the test fixture: 
```c++
virtual void SetUp() {
  isolate_ = v8::Isolate::New(create_params_);
}
```
This will call: 
```c++
Isolate* Isolate::New(const Isolate::CreateParams& params) {
  Isolate* isolate = Allocate();
  Initialize(isolate, params);
  return isolate;
}
```
In `Isolate::Initialize` we'll call `i::Snapshot::Initialize(i_isolate)`:
```c++
if (params.entry_hook || !i::Snapshot::Initialize(i_isolate)) {
  ...
```
Which will call:
```c++
bool success = isolate->Init(&deserializer);
```
Before this call all the roots are uninitialized. Reading this [blog](https://v8project.blogspot.com/) it says that
the Isolate class contains a roots table. It looks to me that the Heap contains this data structure but perhaps that
is what they meant. 
```console
(lldb) bt 3
* thread #1, queue = 'com.apple.main-thread', stop reason = step over
  * frame #0: 0x0000000101584f43 libv8.dylib`v8::internal::StartupDeserializer::DeserializeInto(this=0x00007ffeefbfe200, isolate=0x000000010481cc00) at startup-deserializer.cc:39
    frame #1: 0x0000000101028bb6 libv8.dylib`v8::internal::Isolate::Init(this=0x000000010481cc00, des=0x00007ffeefbfe200) at isolate.cc:3036
    frame #2: 0x000000010157c682 libv8.dylib`v8::internal::Snapshot::Initialize(isolate=0x000000010481cc00) at snapshot-common.cc:54
```
In `startup-deserializer.cc` we can find `StartupDeserializer::DeserializeInto`:
```c++
  DisallowHeapAllocation no_gc;
  isolate->heap()->IterateSmiRoots(this);
  isolate->heap()->IterateStrongRoots(this, VISIT_ONLY_STRONG);
```
After 
If we take a look in `src/roots.h` we can find the read-only roots in Heap. If we take the 10 value, which is:
```c++
V(String, empty_string, empty_string)                                        \
```
we can then inspect this value:
```console
(lldb) expr roots_[9]
(v8::internal::Object *) $32 = 0x0000152d30b82851
(lldb) expr roots_[9]->IsString()
(bool) $30 = true
(lldb) expr roots_[9]->Print()
#
```
So this entry is a pointer to objects on the managed heap which have been deserialized from the snapshot.

The heap class has a lot of members that are initialized during construction by the body of the constructor looks like this:
```c++
{
  // Ensure old_generation_size_ is a multiple of kPageSize.
  DCHECK_EQ(0, max_old_generation_size_ & (Page::kPageSize - 1));

  memset(roots_, 0, sizeof(roots_[0]) * kRootListLength);
  set_native_contexts_list(nullptr);
  set_allocation_sites_list(Smi::kZero);
  set_encountered_weak_collections(Smi::kZero);
  // Put a dummy entry in the remembered pages so we can find the list the
  // minidump even if there are no real unmapped pages.
  RememberUnmappedPage(nullptr, false);
}
```
We can see that roots_ is filled with 0 values. We can inspect `roots_` using:
```console
(lldb) expr roots_
(lldb) expr RootListIndex::kRootListLength
(int) $16 = 509
```
Now they are all 0 at this stage, so when will this array get populated?  
These will happen in `Isolate::Init`:
```c++
  heap_.SetUp()
  if (!create_heap_objects) des->DeserializeInto(this);

void StartupDeserializer::DeserializeInto(Isolate* isolate) {
-> 17    Initialize(isolate);
startup-deserializer.cc:37

isolate->heap()->IterateSmiRoots(this);
```

This will delegate to `ConfigureHeapDefaults()` which will call Heap::ConfigureHeap:
```c++
enum RootListIndex {
  kFreeSpaceMapRootIndex,
  kOnePointerFillerMapRootIndex,
  ...
}
```

```console
(lldb) expr heap->RootListIndex::kFreeSpaceMapRootIndex
(int) $3 = 0
(lldb) expr heap->RootListIndex::kOnePointerFillerMapRootIndex
(int) $4 = 1
```

### MemoryChunk
Found in `src/heap/spaces.h` an instace of a MemoryChunk represents a region in memory that is 
owned by a specific space.


### Embedded builtins
In the [blog post](https://v8project.blogspot.com/) explains how the builtins are embedded into the executable
in to the .TEXT section which is readonly and therefore can be shared amoung multiple processes. We know that
builtins are compiled and stored in the snapshot but now it seems that the are instead placed in to `out.gn/learning/gen/embedded.cc` and the combined with the object files from the compile to produce the libv8.dylib.
V8 has a configuration option named `v8_enable_embedded_builtins` which which case `embedded.cc` will be added 
to the list of sources. This is done in `BUILD.gn` and the `v8_snapshot` target. If `v8_enable_embedded_builtins` is false then `src/snapshot/embedded-empty.cc` will be included instead. Both of these files have the following functions:
```c++
const uint8_t* DefaultEmbeddedBlob()
uint32_t DefaultEmbeddedBlobSize()

#ifdef V8_MULTI_SNAPSHOTS
const uint8_t* TrustedEmbeddedBlob()
uint32_t TrustedEmbeddedBlobSize()
#endif
```
These functions are used by `isolate.cc` and declared `extern`:
```c++
extern const uint8_t* DefaultEmbeddedBlob();
extern uint32_t DefaultEmbeddedBlobSize();
```
And the usage of `DefaultEmbeddedBlob` can be see in Isolate::Isolate where is sets the embedded blob:
```c++
SetEmbeddedBlob(DefaultEmbeddedBlob(), DefaultEmbeddedBlobSize());
```
Lets set a break point there and see if this is empty of not.
```console
(lldb) expr v8_embedded_blob_size_
(uint32_t) $0 = 4021088
```
So we can see that we are not using the empty one. Isolate::SetEmbeddedBlob

We can see in `src/snapshot/deserializer.cc` (line 552) we have a check for the embedded_blob():
```c++
  CHECK_NOT_NULL(isolate->embedded_blob());
  EmbeddedData d = EmbeddedData::FromBlob();
  Address address = d.InstructionStartOfBuiltin(builtin_index);
```
`EmbeddedData can be found in `src/snapshot/snapshot.h` and the implementation can be found in snapshot-common.cc.
```c++
Address EmbeddedData::InstructionStartOfBuiltin(int i) const {
  const struct Metadata* metadata = Metadata();
  const uint8_t* result = RawData() + metadata[i].instructions_offset;
  return reinterpret_cast<Address>(result);
}
```
```console
(lldb) expr *metadata
(const v8::internal::EmbeddedData::Metadata) $7 = (instructions_offset = 0, instructions_length = 1464)
```
```c++
  struct Metadata {
    // Blob layout information.
    uint32_t instructions_offset;
    uint32_t instructions_length;
  };
```
```console
(lldb) expr *this
(v8::internal::EmbeddedData) $10 = (data_ = "\xffffffdc\xffffffc0\xffffff88'"y[\xffffffd6", size_ = 4021088)
(lldb) expr metadata[i]
(const v8::internal::EmbeddedData::Metadata) $8 = (instructions_offset = 0, instructions_length = 1464)
```
So, is it possible for us to verify that this information is in the .text section?
```console
(lldb) expr result
(const uint8_t *) $13 = 0x0000000101b14ee0 "UH\x89�jH\x83�(H\x89U�H�\x16H\x89}�H�u�H�E�H\x89U�H\x83�
(lldb) image lookup --address 0x0000000101b14ee0 --verbose
      Address: libv8.dylib[0x00000000019cdee0] (libv8.dylib.__TEXT.__text + 27054464)
      Summary: libv8.dylib`v8_Default_embedded_blob_ + 7072
       Module: file = "/Users/danielbevenius/work/google/javascript/v8/out.gn/learning/libv8.dylib", arch = "x86_64"
       Symbol: id = {0x0004b596}, range = [0x0000000101b13340-0x0000000101ee8ea0), name="v8_Default_embedded_blob_"
```
So what we have is a pointer to the .text segment which is returned:
```console
(lldb) memory read -f x -s 1 -c 13 0x0000000101b14ee0
0x101b14ee0: 0x55 0x48 0x89 0xe5 0x6a 0x18 0x48 0x83
0x101b14ee8: 0xec 0x28 0x48 0x89 0x55
```
And we can compare this with `out.gn/learning/gen/embedded.cc`:
```c++
V8_EMBEDDED_TEXT_HEADER(v8_Default_embedded_blob_)
__asm__(
  ...
  ".byte 0x55,0x48,0x89,0xe5,0x6a,0x18,0x48,0x83,0xec,0x28,0x48,0x89,0x55\n"
  ...
);
```
The macro `V8_EMBEDDED_TEXT_HEADER` can be found `src/snapshot/macros.h`:
```c++
#define V8_EMBEDDED_TEXT_HEADER(LABEL)         \
  __asm__(V8_ASM_DECLARE(#LABEL)               \
          ".csect " #LABEL "[DS]\n"            \
          #LABEL ":\n"                         \
          ".llong ." #LABEL ", TOC[tc0], 0\n"  \
          V8_ASM_TEXT_SECTION                  \
          "." #LABEL ":\n");

define V8_ASM_DECLARE(NAME) ".private_extern " V8_ASM_MANGLE_LABEL NAME "\n"
#define V8_ASM_MANGLE_LABEL "_"
#define V8_ASM_TEXT_SECTION ".csect .text[PR]\n"
```
And would be expanded by the preprocessor into:
```c++
  __asm__(".private_extern " _ v8_Default_embedded_blob_ "\n"
          ".csect " v8_Default_embedded_blob_ "[DS]\n"
          v8_Default_embedded_blob_ ":\n"
          ".llong ." v8_Default_embedded_blob_ ", TOC[tc0], 0\n"
          ".csect .text[PR]\n"
          "." v8_Default_embedded_blob_ ":\n");
  __asm__(
    ...
    ".byte 0x55,0x48,0x89,0xe5,0x6a,0x18,0x48,0x83,0xec,0x28,0x48,0x89,0x55\n"
    ...
  );

```

Back in `src/snapshot/deserialzer.cc` we are on this line:
```c++
  Address address = d.InstructionStartOfBuiltin(builtin_index);
  CHECK_NE(kNullAddress, address);
  if (RelocInfo::OffHeapTargetIsCodedSpecially()) {
    // is false in our case so skipping the code here
  } else {
    MaybeObject* o = reinterpret_cast<MaybeObject*>(address);
    UnalignedCopy(current, &o);
    current++;
  }
  break;
```

### print-code
```console
$ ./d8 -print-bytecode  -print-code sample.js 
[generated bytecode for function:  (0x2a180824ffbd <SharedFunctionInfo>)]
Parameter count 1
Register count 5
Frame size 40
         0x2a1808250066 @    0 : 12 00             LdaConstant [0]
         0x2a1808250068 @    2 : 26 f9             Star r2
         0x2a180825006a @    4 : 27 fe f8          Mov <closure>, r3
         0x2a180825006d @    7 : 61 32 01 f9 02    CallRuntime [DeclareGlobals], r2-r3
         0x2a1808250072 @   12 : 0b                LdaZero 
         0x2a1808250073 @   13 : 26 fa             Star r1
         0x2a1808250075 @   15 : 0d                LdaUndefined 
         0x2a1808250076 @   16 : 26 fb             Star r0
         0x2a1808250078 @   18 : 00 0c 10 27       LdaSmi.Wide [10000]
         0x2a180825007c @   22 : 69 fa 00          TestLessThan r1, [0]
         0x2a180825007f @   25 : 9a 1c             JumpIfFalse [28] (0x2a180825009b @ 53)
         0x2a1808250081 @   27 : a7                StackCheck 
         0x2a1808250082 @   28 : 13 01 01          LdaGlobal [1], [1]
         0x2a1808250085 @   31 : 26 f9             Star r2
         0x2a1808250087 @   33 : 0c 02             LdaSmi [2]
         0x2a1808250089 @   35 : 26 f7             Star r4
         0x2a180825008b @   37 : 5e f9 fa f7 03    CallUndefinedReceiver2 r2, r1, r4, [3]
         0x2a1808250090 @   42 : 26 fb             Star r0
         0x2a1808250092 @   44 : 25 fa             Ldar r1
         0x2a1808250094 @   46 : 4c 05             Inc [5]
         0x2a1808250096 @   48 : 26 fa             Star r1
         0x2a1808250098 @   50 : 8a 20 00          JumpLoop [32], [0] (0x2a1808250078 @ 18)
         0x2a180825009b @   53 : 25 fb             Ldar r0
         0x2a180825009d @   55 : ab                Return 
Constant pool (size = 2)
0x2a1808250035: [FixedArray] in OldSpace
 - map: 0x2a18080404b1 <Map>
 - length: 2
           0: 0x2a180824ffe5 <FixedArray[2]>
           1: 0x2a180824ff61 <String[#9]: something>
Handler Table (size = 0)
Source Position Table (size = 0)
[generated bytecode for function: something (0x2a180824fff5 <SharedFunctionInfo something>)]
Parameter count 3
Register count 0
Frame size 0
         0x2a18082501ba @    0 : 25 02             Ldar a1
         0x2a18082501bc @    2 : 34 03 00          Add a0, [0]
         0x2a18082501bf @    5 : ab                Return 
Constant pool (size = 0)
Handler Table (size = 0)
Source Position Table (size = 0)
--- Raw source ---
function something(x, y) {
  return x + y
}
for (let i = 0; i < 10000; i++) {
  something(i, 2);
}


--- Optimized code ---
optimization_id = 0
source_position = 0
kind = OPTIMIZED_FUNCTION
stack_slots = 14
compiler = turbofan
address = 0x108400082ae1

Instructions (size = 536)
0x108400082b20     0  488d1df9ffffff REX.W leaq rbx,[rip+0xfffffff9]
0x108400082b27     7  483bd9         REX.W cmpq rbx,rcx
0x108400082b2a     a  7418           jz 0x108400082b44  <+0x24>
0x108400082b2c     c  48ba6800000000000000 REX.W movq rdx,0x68
0x108400082b36    16  49bae0938c724b560000 REX.W movq r10,0x564b728c93e0  (Abort)    ;; off heap target
0x108400082b40    20  41ffd2         call r10
0x108400082b43    23  cc             int3l
0x108400082b44    24  8b59d0         movl rbx,[rcx-0x30]
0x108400082b47    27  4903dd         REX.W addq rbx,r13
0x108400082b4a    2a  f6430701       testb [rbx+0x7],0x1
0x108400082b4e    2e  740d           jz 0x108400082b5d  <+0x3d>
0x108400082b50    30  49bae0f781724b560000 REX.W movq r10,0x564b7281f7e0  (CompileLazyDeoptimizedCode)    ;; off heap target
0x108400082b5a    3a  41ffe2         jmp r10
0x108400082b5d    3d  55             push rbp
0x108400082b5e    3e  4889e5         REX.W movq rbp,rsp
0x108400082b61    41  56             push rsi
0x108400082b62    42  57             push rdi
0x108400082b63    43  48ba4200000000000000 REX.W movq rdx,0x42
0x108400082b6d    4d  4c8b15c4ffffff REX.W movq r10,[rip+0xffffffc4]
0x108400082b74    54  41ffd2         call r10
0x108400082b77    57  cc             int3l
0x108400082b78    58  4883ec18       REX.W subq rsp,0x18
0x108400082b7c    5c  488975a0       REX.W movq [rbp-0x60],rsi
0x108400082b80    60  488b4dd0       REX.W movq rcx,[rbp-0x30]
0x108400082b84    64  f6c101         testb rcx,0x1
0x108400082b87    67  0f8557010000   jnz 0x108400082ce4  <+0x1c4>
0x108400082b8d    6d  81f9204e0000   cmpl rcx,0x4e20
0x108400082b93    73  0f8c0b000000   jl 0x108400082ba4  <+0x84>
0x108400082b99    79  488b45d8       REX.W movq rax,[rbp-0x28]
0x108400082b9d    7d  488be5         REX.W movq rsp,rbp
0x108400082ba0    80  5d             pop rbp
0x108400082ba1    81  c20800         ret 0x8
0x108400082ba4    84  493b6560       REX.W cmpq rsp,[r13+0x60] (external value (StackGuard::address_of_jslimit()))
0x108400082ba8    88  0f8669000000   jna 0x108400082c17  <+0xf7>
0x108400082bae    8e  488bf9         REX.W movq rdi,rcx
0x108400082bb1    91  d1ff           sarl rdi, 1
0x108400082bb3    93  4c8bc7         REX.W movq r8,rdi
0x108400082bb6    96  4183c002       addl r8,0x2
0x108400082bba    9a  0f8030010000   jo 0x108400082cf0  <+0x1d0>
0x108400082bc0    a0  83c701         addl rdi,0x1
0x108400082bc3    a3  0f8033010000   jo 0x108400082cfc  <+0x1dc>
0x108400082bc9    a9  e921000000     jmp 0x108400082bef  <+0xcf>
0x108400082bce    ae  6690           nop
0x108400082bd0    b0  488bcf         REX.W movq rcx,rdi
0x108400082bd3    b3  83c102         addl rcx,0x2
0x108400082bd6    b6  0f802c010000   jo 0x108400082d08  <+0x1e8>
0x108400082bdc    bc  4c8bc7         REX.W movq r8,rdi
0x108400082bdf    bf  4183c001       addl r8,0x1
0x108400082be3    c3  0f802b010000   jo 0x108400082d14  <+0x1f4>
0x108400082be9    c9  498bf8         REX.W movq rdi,r8
0x108400082bec    cc  4c8bc1         REX.W movq r8,rcx
0x108400082bef    cf  81ff10270000   cmpl rdi,0x2710
0x108400082bf5    d5  0f8d0b000000   jge 0x108400082c06  <+0xe6>
0x108400082bfb    db  493b6560       REX.W cmpq rsp,[r13+0x60] (external value (StackGuard::address_of_jslimit()))
0x108400082bff    df  77cf           ja 0x108400082bd0  <+0xb0>
0x108400082c01    e1  e943000000     jmp 0x108400082c49  <+0x129>
0x108400082c06    e6  498bc8         REX.W movq rcx,r8
0x108400082c09    e9  4103c8         addl rcx,r8
0x108400082c0c    ec  0f8061000000   jo 0x108400082c73  <+0x153>
0x108400082c12    f2  488bc1         REX.W movq rax,rcx
0x108400082c15    f5  eb86           jmp 0x108400082b9d  <+0x7d>
0x108400082c17    f7  33c0           xorl rax,rax
0x108400082c19    f9  48bef50c240884100000 REX.W movq rsi,0x108408240cf5    ;; object: 0x108408240cf5 <NativeContext[261]>
0x108400082c23   103  48bb101206724b560000 REX.W movq rbx,0x564b72061210    ;; external reference (Runtime::StackGuard)
0x108400082c2d   10d  488bf8         REX.W movq rdi,rax
0x108400082c30   110  4c8bc6         REX.W movq r8,rsi
0x108400082c33   113  49ba2089a3724b560000 REX.W movq r10,0x564b72a38920  (CEntry_Return1_DontSaveFPRegs_ArgvOnStack_NoBuiltinExit)    ;; off heap target
0x108400082c3d   11d  41ffd2         call r10
0x108400082c40   120  488b4dd0       REX.W movq rcx,[rbp-0x30]
0x108400082c44   124  e965ffffff     jmp 0x108400082bae  <+0x8e>
0x108400082c49   129  48897da8       REX.W movq [rbp-0x58],rdi
0x108400082c4d   12d  488b1dd1ffffff REX.W movq rbx,[rip+0xffffffd1]
0x108400082c54   134  33c0           xorl rax,rax
0x108400082c56   136  48bef50c240884100000 REX.W movq rsi,0x108408240cf5    ;; object: 0x108408240cf5 <NativeContext[261]>
0x108400082c60   140  4c8b15ceffffff REX.W movq r10,[rip+0xffffffce]
0x108400082c67   147  41ffd2         call r10
0x108400082c6a   14a  488b7da8       REX.W movq rdi,[rbp-0x58]
0x108400082c6e   14e  e95dffffff     jmp 0x108400082bd0  <+0xb0>
0x108400082c73   153  48b968ea2f744b560000 REX.W movq rcx,0x564b742fea68    ;; external reference (Heap::NewSpaceAllocationTopAddress())
0x108400082c7d   15d  488b39         REX.W movq rdi,[rcx]
0x108400082c80   160  4c8d4f0c       REX.W leaq r9,[rdi+0xc]
0x108400082c84   164  4c8945b0       REX.W movq [rbp-0x50],r8
0x108400082c88   168  49bb70ea2f744b560000 REX.W movq r11,0x564b742fea70    ;; external reference (Heap::NewSpaceAllocationLimitAddress())
0x108400082c92   172  4d390b         REX.W cmpq [r11],r9
0x108400082c95   175  0f8721000000   ja 0x108400082cbc  <+0x19c>
0x108400082c9b   17b  ba0c000000     movl rdx,0xc
0x108400082ca0   180  49ba200282724b560000 REX.W movq r10,0x564b72820220  (AllocateRegularInYoungGeneration)    ;; off heap target
0x108400082caa   18a  41ffd2         call r10
0x108400082cad   18d  488d78ff       REX.W leaq rdi,[rax-0x1]
0x108400082cb1   191  488b0dbdffffff REX.W movq rcx,[rip+0xffffffbd]
0x108400082cb8   198  4c8b45b0       REX.W movq r8,[rbp-0x50]
0x108400082cbc   19c  4c8d4f0c       REX.W leaq r9,[rdi+0xc]
0x108400082cc0   1a0  4c8909         REX.W movq [rcx],r9
0x108400082cc3   1a3  488d4f01       REX.W leaq rcx,[rdi+0x1]
0x108400082cc7   1a7  498bbd40010000 REX.W movq rdi,[r13+0x140] (root (heap_number_map))
0x108400082cce   1ae  8979ff         movl [rcx-0x1],rdi
0x108400082cd1   1b1  c4c1032ac0     vcvtlsi2sd xmm0,xmm15,r8
0x108400082cd6   1b6  c5fb114103     vmovsd [rcx+0x3],xmm0
0x108400082cdb   1bb  488bc1         REX.W movq rax,rcx
0x108400082cde   1be  e9bafeffff     jmp 0x108400082b9d  <+0x7d>
0x108400082ce3   1c3  90             nop
0x108400082ce4   1c4  49c7c500000000 REX.W movq r13,0x0
0x108400082ceb   1cb  e850f30300     call 0x1084000c2040     ;; eager deoptimization bailout
0x108400082cf0   1d0  49c7c501000000 REX.W movq r13,0x1
0x108400082cf7   1d7  e844f30300     call 0x1084000c2040     ;; eager deoptimization bailout
0x108400082cfc   1dc  49c7c502000000 REX.W movq r13,0x2
0x108400082d03   1e3  e838f30300     call 0x1084000c2040     ;; eager deoptimization bailout
0x108400082d08   1e8  49c7c503000000 REX.W movq r13,0x3
0x108400082d0f   1ef  e82cf30300     call 0x1084000c2040     ;; eager deoptimization bailout
0x108400082d14   1f4  49c7c504000000 REX.W movq r13,0x4
0x108400082d1b   1fb  e820f30300     call 0x1084000c2040     ;; eager deoptimization bailout
0x108400082d20   200  49c7c505000000 REX.W movq r13,0x5
0x108400082d27   207  e814f30700     call 0x108400102040     ;; lazy deoptimization bailout
0x108400082d2c   20c  49c7c506000000 REX.W movq r13,0x6
0x108400082d33   213  e808f30700     call 0x108400102040     ;; lazy deoptimization bailout

Source positions:
 pc offset  position
        f7         0

Inlined functions (count = 1)
 0x10840824fff5 <SharedFunctionInfo something>

Deoptimization Input Data (deopt points = 7)
 index  bytecode-offset    pc
     0               22    NA 
     1                2    NA 
     2               46    NA 
     3                2    NA 
     4               46    NA 
     5               27   120 
     6               27   14a 

Safepoints (size = 50)
0x108400082c40     120   200  10000010000000 (sp -> fp)       5
0x108400082c6a     14a   20c  10000000000000 (sp -> fp)       6
0x108400082cad     18d    NA  00000000000000 (sp -> fp)  <none>

RelocInfo (size = 34)
0x108400082b38  off heap target
0x108400082b52  off heap target
0x108400082c1b  full embedded object  (0x108408240cf5 <NativeContext[261]>)
0x108400082c25  external reference (Runtime::StackGuard)  (0x564b72061210)
0x108400082c35  off heap target
0x108400082c58  full embedded object  (0x108408240cf5 <NativeContext[261]>)
0x108400082c75  external reference (Heap::NewSpaceAllocationTopAddress())  (0x564b742fea68)
0x108400082c8a  external reference (Heap::NewSpaceAllocationLimitAddress())  (0x564b742fea70)
0x108400082ca2  off heap target
0x108400082cec  runtime entry  (eager deoptimization bailout)
0x108400082cf8  runtime entry  (eager deoptimization bailout)
0x108400082d04  runtime entry  (eager deoptimization bailout)
0x108400082d10  runtime entry  (eager deoptimization bailout)
0x108400082d1c  runtime entry  (eager deoptimization bailout)
0x108400082d28  runtime entry  (lazy deoptimization bailout)
0x108400082d34  runtime entry  (lazy deoptimization bailout)

--- End code ---
$ 

```

### Building Google Test
```console
$ mkdir lib
$ mkdir deps ; cd deps
$ git clone git@github.com:google/googletest.git
$ cd googletest/googletest
$ /usr/bin/clang++ --std=c++14 -Iinclude -I. -pthread -c src/gtest-all.cc
$ ar -rv libgtest-linux.a gtest-all.o 
$ cp libgtest-linux.a ../../../../lib/gtest
```

Linking issue:
```console
./lib/gtest/libgtest-linux.a(gtest-all.o):gtest-all.cc:function testing::internal::BoolFromGTestEnv(char const*, bool): error: undefined reference to 'std::__cxx11::basic_string<char, std::char_traits<char>, std::allocator<char> >::c_str() const'
```
```
$ nm lib/gtest/libgtest-linux.a | grep basic_string | c++filt 
....
```
There are a lot of symbols listed above but the point is that in the object
file of `libgtest-linux.a` these symbols were compiled in. Now, when we compile
v8 and the tests we are using `-std=c++14` and we have to use the same when compiling
gtest. Lets try that. Just adding that does not help in this case. We need to
check which c++ headers are being used:
```console
$ /usr/bin/clang++ -print-search-dirs
programs: =/usr/bin:/usr/bin/../lib/gcc/x86_64-redhat-linux/9/../../../../x86_64-redhat-linux/bin
libraries: =/usr/lib64/clang/9.0.0:
            /usr/bin/../lib/gcc/x86_64-redhat-linux/9:
            /usr/bin/../lib/gcc/x86_64-redhat-linux/9/../../../../lib64:
            /usr/bin/../lib64:
            /lib/../lib64:
            /usr/lib/../lib64:
            /usr/bin/../lib/gcc/x86_64-redhat-linux/9/../../..:
            /usr/bin/../lib:
            /lib:/usr/lib
$ 
```
Lets search for the `string` header and inspect the namespace in that header:
```console
$ find /usr/ -name string
/usr/include/c++/9/debug/string
/usr/include/c++/9/experimental/string
/usr/include/c++/9/string
/usr/src/debug/gcc-9.2.1-1.fc31.x86_64/obj-x86_64-redhat-linux/x86_64-redhat-linux/libstdc++-v3/include/string
```
```console
$ vi /usr/include/c++/9/string
```
So this looks alright and thinking about this a little more I've been bitten
by the linking with different libc++ symbols issue (again). When we compile using Make we
are using the c++ headers that are shipped with v8 (clang libc++). Take the
string header for example in v8/buildtools/third_party/libc++/trunk/include/string
which is from clang's c++ library which does not use namespaces (__11 or __14 etc).

But when I compiled gtest did not specify the istystem include path and the
default would be used adding symbols with __11 into them. When the linker tries
to find these symbols it fails as it does not have any such symbols in the libraries
that it searches.

Create a simple test linking with the standard build of gtest to see if that
compiles and runs:
```console
$ /usr/bin/clang++ -std=c++14 -I./deps/googletest/googletest/include  -L$PWD/lib -g -O0 -o test/simple_test test/main.cc test/simple.cc lib/libgtest.a -lpthread
```
That worked and does not segfault. 

But when I run the version that is built using the makefile I get:
```console
lldb) target create "./test/persistent-object_test"
Current executable set to './test/persistent-object_test' (x86_64).
(lldb) r
Process 1024232 launched: '/home/danielbevenius/work/google/learning-v8/test/persistent-object_test' (x86_64)
warning: (x86_64) /lib64/libgcc_s.so.1 unsupported DW_FORM values: 0x1f20 0x1f21

[ FATAL ] Process 1024232 stopped
* thread #1, name = 'persistent-obje', stop reason = signal SIGSEGV: invalid address (fault address: 0x33363658)
    frame #0: 0x00007ffff7c0a7b0 libc.so.6`__GI___libc_free + 32
libc.so.6`__GI___libc_free:
->  0x7ffff7c0a7b0 <+32>: mov    rax, qword ptr [rdi - 0x8]
    0x7ffff7c0a7b4 <+36>: lea    rsi, [rdi - 0x10]
    0x7ffff7c0a7b8 <+40>: test   al, 0x2
    0x7ffff7c0a7ba <+42>: jne    0x7ffff7c0a7f0            ; <+96>
(lldb) bt
* thread #1, name = 'persistent-obje', stop reason = signal SIGSEGV: invalid address (fault address: 0x33363658)
  * frame #0: 0x00007ffff7c0a7b0 libc.so.6`__GI___libc_free + 32
    frame #1: 0x000000000042bb58 persistent-object_test`std::__1::basic_stringbuf<char, std::__1::char_traits<char>, std::__1::allocator<char> >::~basic_stringbuf(this=0x000000000046e908) at iosfwd:130:32
    frame #2: 0x000000000042ba4f persistent-object_test`std::__1::basic_stringstream<char, std::__1::char_traits<char>, std::__1::allocator<char> >::~basic_stringstream(this=0x000000000046e8f0, vtt=0x000000000044db28) at iosfwd:139:32
    frame #3: 0x0000000000420176 persistent-object_test`std::__1::basic_stringstream<char, std::__1::char_traits<char>, std::__1::allocator<char> >::~basic_stringstream(this=0x000000000046e8f0) at iosfwd:139:32
    frame #4: 0x000000000042bacc persistent-object_test`std::__1::basic_stringstream<char, std::__1::char_traits<char>, std::__1::allocator<char> >::~basic_stringstream(this=0x000000000046e8f0) at iosfwd:139:32
    frame #5: 0x0000000000427f4e persistent-object_test`testing::internal::scoped_ptr<std::__1::basic_stringstream<char, std::__1::char_traits<char>, std::__1::allocator<char> > >::reset(this=0x00007fffffffcee8, p=0x0000000000000000) at gtest-port.h:1216:9
    frame #6: 0x0000000000427ee9 persistent-object_test`testing::internal::scoped_ptr<std::__1::basic_stringstream<char, std::__1::char_traits<char>, std::__1::allocator<char> > >::~scoped_ptr(this=0x00007fffffffcee8) at gtest-port.h:1201:19
    frame #7: 0x000000000041f265 persistent-object_test`testing::Message::~Message(this=0x00007fffffffcee8) at gtest-message.h:89:18
    frame #8: 0x00000000004235ec persistent-object_test`std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> > testing::internal::StreamableToString<int>(streamable=0x00007fffffffcf9c) at gtest-message.h:247:3
    frame #9: 0x000000000040d2bd persistent-object_test`testing::internal::FormatFileLocation(file="/home/danielbevenius/work/google/learning-v8/deps/googletest/googletest/src/gtest-internal-inl.h", line=663) at gtest-port.cc:946:28
    frame #10: 0x000000000041b7e2 persistent-object_test`testing::internal::GTestLog::GTestLog(this=0x00007fffffffd060, severity=GTEST_FATAL, file="/home/danielbevenius/work/google/learning-v8/deps/googletest/googletest/src/gtest-internal-inl.h", line=663) at gtest-port.cc:972:18
    frame #11: 0x000000000042242c persistent-object_test`testing::internal::UnitTestImpl::AddTestInfo(this=0x000000000046e480, set_up_tc=(persistent-object_test`testing::Test::SetUpTestCase() at gtest.h:427), tear_down_tc=(persistent-object_test`testing::Test::TearDownTestCase() at gtest.h:435), test_info=0x000000000046e320)(), void (*)(), testing::TestInfo*) at gtest-internal-inl.h:663:7
    frame #12: 0x000000000040d04f persistent-object_test`testing::internal::MakeAndRegisterTestInfo(test_case_name="Persistent", name="object", type_param=0x0000000000000000, value_param=0x0000000000000000, code_location=<unavailable>, fixture_class_id=0x000000000046d748, set_up_tc=(persistent-object_test`testing::Test::SetUpTestCase() at gtest.h:427), tear_down_tc=(persistent-object_test`testing::Test::TearDownTestCase() at gtest.h:435), factory=0x000000000046e300)(), void (*)(), testing::internal::TestFactoryBase*) at gtest.cc:2599:22
    frame #13: 0x00000000004048b8 persistent-object_test`::__cxx_global_var_init() at persistent-object_test.cc:5:1
    frame #14: 0x00000000004048e9 persistent-object_test`_GLOBAL__sub_I_persistent_object_test.cc at persistent-object_test.cc:0
    frame #15: 0x00000000004497a5 persistent-object_test`__libc_csu_init + 69
    frame #16: 0x00007ffff7ba512e libc.so.6`__libc_start_main + 126
    frame #17: 0x0000000000404eba persistent-object_test`_start + 42
```

### Google test (gtest) linking issue
This issue came up when linking a unit test with gtest:
```console
/usr/bin/ld: ./lib/gtest/libgtest-linux.a(gtest-all.o): in function `testing::internal::BoolFromGTestEnv(char const*, bool)':
/home/danielbevenius/work/google/learning-v8/deps/googletest/googletest/src/gtest-port.cc:1259: undefined reference to `std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> >::~basic_string()'
```
So this indicated that the object files in `libgtest-linux.a` where infact using
headers from libc++ and not libstc++. This was a really stupig mistake on my
part, I'd not specified the output file explicitly (-o) so this was getting
added into the current working directory, but the file included in the archive
was taken from within deps/googltest/googletest/ directory which was old and
compiled using libc++.

### Peristent cast-function-type
This issue was seen in Node.js when compiling with GCC. It can also been see
if building V8 using GCC and also enabling `-Wcast-function-type` in BUILD.gn:
```
      "-Wcast-function-type",
```
There are unit tests in V8 that also produce this warning, for example
`test/cctest/test-global-handles.cc`:
Original:
```console
g++ -MMD -MF obj/test/cctest/cctest_sources/test-global-handles.o.d -DV8_INTL_SUPPORT -DUSE_UDEV -DUSE_AURA=1 -DUSE_GLIB=1 -DUSE_NSS_CERTS=1 -DUSE_X11=1 -D_FILE_OFFSET_BITS=64 -D_LARGEFILE_SOURCE -D_LARGEFILE64_SOURCE -D__STDC_CONSTANT_MACROS -D__STDC_FORMAT_MACROS -DCR_SYSROOT_HASH=9c905c99558f10e19cc878b5dca1d4bd58c607ae -D_DEBUG -DDYNAMIC_ANNOTATIONS_ENABLED=1 -DENABLE_DISASSEMBLER -DV8_TYPED_ARRAY_MAX_SIZE_IN_HEAP=64 -DENABLE_GDB_JIT_INTERFACE -DENABLE_MINOR_MC -DOBJECT_PRINT -DV8_TRACE_MAPS -DV8_ENABLE_ALLOCATION_TIMEOUT -DV8_ENABLE_FORCE_SLOW_PATH -DV8_ENABLE_DOUBLE_CONST_STORE_CHECK -DV8_INTL_SUPPORT -DENABLE_HANDLE_ZAPPING -DV8_SNAPSHOT_NATIVE_CODE_COUNTERS -DV8_CONCURRENT_MARKING -DV8_ENABLE_LAZY_SOURCE_POSITIONS -DV8_CHECK_MICROTASKS_SCOPES_CONSISTENCY -DV8_EMBEDDED_BUILTINS -DV8_WIN64_UNWINDING_INFO -DV8_ENABLE_REGEXP_INTERPRETER_THREADED_DISPATCH -DV8_SNAPSHOT_COMPRESSION -DV8_ENABLE_CHECKS -DV8_COMPRESS_POINTERS -DV8_31BIT_SMIS_ON_64BIT_ARCH -DV8_DEPRECATION_WARNINGS -DV8_IMMINENT_DEPRECATION_WARNINGS -DV8_TARGET_ARCH_X64 -DV8_HAVE_TARGET_OS -DV8_TARGET_OS_LINUX -DDEBUG -DDISABLE_UNTRUSTED_CODE_MITIGATIONS -DV8_ENABLE_CHECKS -DV8_COMPRESS_POINTERS -DV8_31BIT_SMIS_ON_64BIT_ARCH -DV8_DEPRECATION_WARNINGS -DV8_IMMINENT_DEPRECATION_WARNINGS -DU_USING_ICU_NAMESPACE=0 -DU_ENABLE_DYLOAD=0 -DUSE_CHROMIUM_ICU=1 -DU_STATIC_IMPLEMENTATION -DICU_UTIL_DATA_IMPL=ICU_UTIL_DATA_FILE -DUCHAR_TYPE=uint16_t -I../.. -Igen -I../../include -Igen/include -I../.. -Igen -I../../third_party/icu/source/common -I../../third_party/icu/source/i18n -I../../include -I../../tools/debug_helper -fno-strict-aliasing --param=ssp-buffer-size=4 -fstack-protector -funwind-tables -fPIC -pipe -B../../third_party/binutils/Linux_x64/Release/bin -pthread -m64 -march=x86-64 -Wno-builtin-macro-redefined -D__DATE__= -D__TIME__= -D__TIMESTAMP__= -Wall -Wno-unused-local-typedefs -Wno-maybe-uninitialized -Wno-deprecated-declarations -Wno-comments -Wno-packed-not-aligned -Wno-missing-field-initializers -Wno-unused-parameter -fno-omit-frame-pointer -g2 -Wno-strict-overflow -Wno-return-type -Wcast-function-type -O3 -fno-ident -fdata-sections -ffunction-sections -fvisibility=default -std=gnu++14 -Wno-narrowing -Wno-class-memaccess -fno-exceptions -fno-rtti --sysroot=../../build/linux/debian_sid_amd64-sysroot -c ../../test/cctest/test-global-handles.cc -o obj/test/cctest/cctest_sources/test-global-handles.o
In file included from ../../include/v8-inspector.h:14,
                 from ../../src/execution/isolate.h:15,
                 from ../../src/api/api.h:10,
                 from ../../src/api/api-inl.h:8,
                 from ../../test/cctest/test-global-handles.cc:28:
../../include/v8.h: In instantiation of ‘void v8::PersistentBase<T>::SetWeak(P*, typename v8::WeakCallbackInfo<P>::Callback, v8::WeakCallbackType) [with P = v8::Global<v8::Object>; T = v8::Object; typename v8::WeakCallbackInfo<P>::Callback = void (*)(const v8::WeakCallbackInfo<v8::Global<v8::Object> >&)]’:
../../test/cctest/test-global-handles.cc:292:47:   required from here
../../include/v8.h:10750:16: warning: cast between incompatible function types from ‘v8::WeakCallbackInfo<v8::Global<v8::Object> >::Callback’ {aka ‘void (*)(const v8::WeakCallbackInfo<v8::Global<v8::Object> >&)’} to ‘Callback’ {aka ‘void (*)(const v8::WeakCallbackInfo<void>&)’} [-Wcast-function-type]
10750 |                reinterpret_cast<Callback>(callback), type);
      |                ^~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
../../include/v8.h: In instantiation of ‘void v8::PersistentBase<T>::SetWeak(P*, typename v8::WeakCallbackInfo<P>::Callback, v8::WeakCallbackType) [with P = v8::internal::{anonymous}::FlagAndGlobal; T = v8::Object; typename v8::WeakCallbackInfo<P>::Callback = void (*)(const v8::WeakCallbackInfo<v8::internal::{anonymous}::FlagAndGlobal>&)]’:
../../test/cctest/test-global-handles.cc:493:53:   required from here
../../include/v8.h:10750:16: warning: cast between incompatible function types from ‘v8::WeakCallbackInfo<v8::internal::{anonymous}::FlagAndGlobal>::Callback’ {aka ‘void (*)(const v8::WeakCallbackInfo<v8::internal::{anonymous}::FlagAndGlobal>&)’} to ‘Callback’ {aka ‘void (*)(const v8::WeakCallbackInfo<void>&)’} [-Wcast-function-type]
```
Formatted for git commit message:
```console
g++ -MMD -MF obj/test/cctest/cctest_sources/test-global-handles.o.d 
...
In file included from ../../include/v8-inspector.h:14,
                 from ../../src/execution/isolate.h:15,
                 from ../../src/api/api.h:10,
                 from ../../src/api/api-inl.h:8,
                 from ../../test/cctest/test-global-handles.cc:28:
../../include/v8.h:
In instantiation of ‘void v8::PersistentBase<T>::SetWeak(
    P*,
    typename v8::WeakCallbackInfo<P>::Callback,
    v8::WeakCallbackType)
[with 
  P = v8::Global<v8::Object>; 
  T = v8::Object;
  typename v8::WeakCallbackInfo<P>::Callback =
  void (*)(const v8::WeakCallbackInfo<v8::Global<v8::Object> >&)
]’:
../../test/cctest/test-global-handles.cc:292:47:   required from here
../../include/v8.h:10750:16: warning:
cast between incompatible function types from
‘v8::WeakCallbackInfo<v8::Global<v8::Object> >::Callback’ {aka
‘void (*)(const v8::WeakCallbackInfo<v8::Global<v8::Object> >&)’} to 
‘Callback’ {aka ‘void (*)(const v8::WeakCallbackInfo<void>&)’}
[-Wcast-function-type]
10750 |                reinterpret_cast<Callback>(callback), type);
      |                ^~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
```
This commit suggests adding a pragma specifically for GCC to suppress
this warning. The motivation for this is that there were quite a few
of these warnings in the Node.js build, but these have been suppressed
by adding a similar pragma but around the include of v8.h [1].

[1] https://github.com/nodejs/node/blob/331d63624007be4bf49d6d161bdef2b5e540affa/src/node.h#L63-L70

```console
$ 
In file included from persistent-obj.cc:8:
/home/danielbevenius/work/google/v8_src/v8/include/v8.h: In instantiation of ‘void v8::PersistentBase<T>::SetWeak(P*, typename v8::WeakCallbackInfo<P>::Callback, v8::WeakCallbackType) [with P = Something; T = v8::Object; typename v8::WeakCallbackInfo<P>::Callback = void (*)(const v8::WeakCallbackInfo<Something>&)]’:

persistent-obj.cc:57:38:   required from here
/home/danielbevenius/work/google/v8_src/v8/include/v8.h:10750:16: warning: cast between incompatible function types from ‘v8::WeakCallbackInfo<Something>::Callback’ {aka ‘void (*)(const v8::WeakCallbackInfo<Something>&)’} to ‘Callback’ {aka ‘void (*)(const v8::WeakCallbackInfo<void>&)’} [-Wcast-function-type]
10750 |                reinterpret_cast<Callback>(callback), type);
      |                ^~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
```
Currently, we have added a pragma to avoid this warning in node.js but we'd like
to add this in v8 and closer to the actual code that is causing it. In node we
have to set the praga on the header.
```c++
template <class T>
template <typename P>
V8_INLINE void PersistentBase<T>::SetWeak(
    P* parameter,
    typename WeakCallbackInfo<P>::Callback callback,
    WeakCallbackType type) {
  typedef typename WeakCallbackInfo<void>::Callback Callback;
  V8::MakeWeak(reinterpret_cast<internal::Address*>(this->val_), parameter,
               reinterpret_cast<Callback>(callback), type);
}
```
Notice the second parameter is `typename WeakCallbackInfo<P>::Callback` which is
a typedef:
```c++
  typedef void (*Callback)(const WeakCallbackInfo<T>& data);
```
This is a function declaration for `Callback` which is a function that takes
a reference to a const WeakCallbackInfo<T> and returns void. So we could define
it like this:
```c++
void WeakCallback(const v8::WeakCallbackInfo<Something>& data) {
  Something* obj = data.GetParameter();
  std::cout << "in make weak callback..." << '\n';
}
```
And the trying to cast it into:
```c++
  typedef typename v8::WeakCallbackInfo<void>::Callback Callback;
  Callback cb = reinterpret_cast<Callback>(WeakCallback);
```
This is done as V8::MakeWeak has the following signature:
```c++
void V8::MakeWeak(i::Address* location, void* parameter,
                  WeakCallbackInfo<void>::Callback weak_callback,
                  WeakCallbackType type) {
  i::GlobalHandles::MakeWeak(location, parameter, weak_callback, type);
}
```


### gdb warnings
```console
warning: Could not find DWO CU obj/v8_compiler/common-node-cache.dwo(0x42b8adb87d74d56b) referenced by CU at offset 0x206f7 [in module /home/danielbevenius/work/google/learning-v8/hello-world]
```
This can be worked around by specifying the `--cd` argument to gdb:
```console
$ gdb --cd=/home/danielbevenius/work/google/v8_src/v8/out/x64.release --args /home/danielbevenius/work/google/learning-v8/hello-world
```

### Building with g++
Update args.gn to include:
```
is_clang = false
```
Next I got the following error when trying to compile:
```console
$ ninja -v -C out/x64.release/ obj/test/cctest/cctest_sources/test-global-handles.o
ux/debian_sid_amd64-sysroot -fexceptions -frtti -c ../../src/torque/instance-type-generator.cc -o obj/torque_base/instance-type-generator.o
In file included from /usr/include/c++/9/bits/stl_algobase.h:59,
                 from /usr/include/c++/9/memory:62,
                 from ../../src/torque/implementation-visitor.h:8,
                 from ../../src/torque/instance-type-generator.cc:5:
/usr/include/c++/9/x86_64-redhat-linux/bits/c++config.h:3:10: fatal error: bits/wordsize.h: No such file or directory
    3 | #include <bits/wordsize.h>
      |          ^~~~~~~~~~~~~~~~~
compilation terminated.
ninja: build stopped: subcommand failed.
```
```console
$ export CPATH=/usr/include
```

```console
third_party/binutils/Linux_x64/Release/bin/ld.gold: error: cannot open /usr/lib64/libatomic.so.1.2.0: No such file or directory
```
```console
$ sudo dnf install -y libatomic
```
I still got an error because of a warning but I'm trying to build using:
```
treat_warnings_as_errors = false
```
Lets see how that works out. I also had to use gnus linker by disableing 
gold:
```console
use_gold = false
```

### CodeStubAssembler
This history of this is that JavaScript builtins used be written in assembly
which gave very good performance but made porting V8 to different architectures
more difficult as these builtins had to have specific implementations for each
supported architecture, so it dit not scale very well. With the addition of features
to the JavaScript specifications having to support new features meant having to
implement them for all platforms which made it difficult to keep up and deliver
these new features.

The goal is to have the perfomance of handcoded assembly but not have to write
it for every platform. So a portable assembly language was build on top of
Tubofans backend. This is an API that generates Turbofan's machine-level IR.
This IR can be used by Turbofan to produce very good machine code on all platforms.
So one "only" has to implement one component/function/feature (not sure what to
call this) and then it can be made available to all platforms. They no longer
have to maintain all that handwritten assembly.

Just to be clear CSA is a C++ API that is used to generate IR which is then
compiled in to machine code for the target instruction set architectur.

### Torque
[Torque](https://v8.dev/docs/torque) is a DLS language to avoid having to use
the CodeStubAssembler directly (it is still used behind the scene). This language
is statically typed, garbage collected, and compatible with JavaScript.

The JavaScript standard library was implemented in V8 previously using hand
written assembly. But as we mentioned in the previous section this did not scale.

It could have been written in JavaScript too, and I think this was done in the
past but this has some issues as builtins would need warmup time to become
optimized, there were also issues with monkey-patching and exposing VM internals
unintentionally.

Is torque run a build time, I'm thinking yes as it would have to generate the
c++ code.

There is a main function in torque.cc which will be built into an executable
```console
$ ./out/x64.release_gcc/torque --help
Unexpected command-line argument "--help", expected a .tq file.
```

The files that are processed by torque are defined in BUILD.gc in the 
`torque_files` section. There is also a template named `run_torque`.
I've noticed that this template and others in GN use the script  `tools/run.py`.
This is apperently because GN can only execute scripts at the moment and what this
script does is use python to create a subprocess with the passed in argument:
```console
$ gn help action
```
And a template is way to reuse code in GN.


There is a make target that shows what is generated by torque:
```console
$ make torque-example
```
This will create a directory in the current directory named `gen/torque-generated`.
Notice that this directory contains c++ headers and sources.

It take [torque-example.tq](./torque-example.tq) as input. For this file the
following header will be generated:
```c++
#ifndef V8_GEN_TORQUE_GENERATED_TORQUE_EXAMPLE_TQ_H_                            
#define V8_GEN_TORQUE_GENERATED_TORQUE_EXAMPLE_TQ_H_                            
                                                                                
#include "src/builtins/builtins-promise.h"                                      
#include "src/compiler/code-assembler.h"                                        
#include "src/codegen/code-stub-assembler.h"                                    
#include "src/utils/utils.h"                                                    
#include "torque-generated/field-offsets-tq.h"                                  
#include "torque-generated/csa-types-tq.h"                                      
                                                                                
namespace v8 {                                                                  
namespace internal {                                                            
                                                                                
void HelloWorld_0(compiler::CodeAssemblerState* state_);                        

}  // namespace internal                                                        
}  // namespace v8                                                              
                                                                                
#endif  // V8_GEN_TORQUE_GENERATED_TORQUE_EXAMPLE_TQ_H_

```
This is only to show the generated files and make it clear that torque will
generate these file which will then be compiled during the v8 build. So, lets
try copying `example-torque.tq` to v8/src/builtins directory.
```console
$ cp torque-example.tq ../v8_src/v8/src/builtins/
```
This is not enough to get it included in the build, we have to update BUILD.gn
and add this file to the `torque_files` list. After running the build we can
see that there is a file named `src/builtins/torque-example-tq-csa.h` generated
along with a .cc.

To understand how this works I'm going to use https://v8.dev/docs/torque-builtins
as a starting point:
```
  transitioning javascript builtin                                              
  MathIs42(js-implicit context: NativeContext, receiver: JSAny)(x: JSAny): Boolean {
    const number: Number = ToNumber_Inline(x);                                  
    typeswitch (number) {                                                       
      case (smi: Smi): {                                                        
        return smi == 42 ? True : False;                                        
      }                                                                         
      case (heapNumber: HeapNumber): {                                          
        return Convert<float64>(heapNumber) == 42 ? True : False;               
      }                                                                         
    }                                                                           
  }                   
```
This has been updated to work with the latest V8 version.

Next, we need to update `src/init/bootstrappers.cc` to add/install this function
on the math object:
```c++
  SimpleInstallFunction(isolate_, math, "is42", Builtins::kMathIs42, 1, true);
```
After this we need to rebuild v8:
```console
$ env CPATH=/usr/include ninja -v -C out/x64.release_gcc
```
```console
$ d8
d8> Math.is42(42)
true
d8> Math.is42(2)
false
```

If we look at the generated code that Torque has produced in 
`out/x64.release_gcc/gen/torque-generated/src/builtins/math-tq-csa.cc` (we can
run it through the preprocessor using):
```console
$ clang++ --sysroot=build/linux/debian_sid_amd64-sysroot -isystem=./buildtools/third_party/libc++/trunk/include -isystem=buildtools/third_party/libc++/trunk/include -I. -E out/x64.release_gcc/gen/torque-generated/src/builtins/math-tq-csa.cc > math.cc.pp
```
If we open math.cc.pp and search for `Is42` we can find:
```c++
class MathIs42Assembler : public CodeStubAssembler {                            
 public:                                                                        
  using Descriptor = Builtin_MathIs42_InterfaceDescriptor;                      
  explicit MathIs42Assembler(compiler::CodeAssemblerState* state) : CodeStubAssembler(state) {}
  void GenerateMathIs42Impl();                                                  
  Node* Parameter(Descriptor::ParameterIndices index) {                         
    return CodeAssembler::Parameter(static_cast<int>(index));                   
  }                                                                             
};                                                                              
                                                                                
void Builtins::Generate_MathIs42(compiler::CodeAssemblerState* state) {         
  MathIs42Assembler assembler(state);                                           
  state->SetInitialDebugInformation("MathIs42", "out/x64.release_gcc/gen/torque-generated/src/builtins/math-tq-csa.cc", 2121);
  if (Builtins::KindOf(Builtins::kMathIs42) == Builtins::TFJ) {                 
    assembler.PerformStackCheck(assembler.GetJSContextParameter());             
  }                                                                             
  assembler.GenerateMathIs42Impl();                                             
}                                                                               
                                                                                
void MathIs42Assembler::GenerateMathIs42Impl() {     
  ...
```
So this is what gets generated by the Torque compiler and what we see
above is CodeStubAssemble class. 

If we take a look in out/x64.release_gcc/gen/torque-generated/builtin-definitions-tq.h
we can find the following line that has been generated:
```c++
TFJ(MathIs42, 1, kReceiver, kX) \                                               
```
Now, there is a section about the [TF_BUILTIN](#tf_builtin) macro, and it will
create function declarations, and function and class definitions:

Now, in src/builtins/builtins.h we have the following macros:
```c++
class Builtins {
 public:

  enum Name : int32_t {
#define DEF_ENUM(Name, ...) k##Name,                                            
    BUILTIN_LIST(DEF_ENUM, DEF_ENUM, DEF_ENUM, DEF_ENUM, DEF_ENUM, DEF_ENUM,    
                 DEF_ENUM)                                                      
#undef DEF_ENUM 
    ...
  }

#define DECLARE_TF(Name, ...) \                                                 
  static void Generate_##Name(compiler::CodeAssemblerState* state);             
                                                                                
  BUILTIN_LIST(IGNORE_BUILTIN, DECLARE_TF, DECLARE_TF, DECLARE_TF, DECLARE_TF,  
               IGNORE_BUILTIN, DECLARE_ASM)
```
And `BUILTINS_LIST` is declared in src/builtins/builtins-definitions.h and this
file includes:
```c++
#include "torque-generated/builtin-definitions-tq.h"

#define BUILTIN_LIST(CPP, TFJ, TFC, TFS, TFH, BCH, ASM)  \                          
  BUILTIN_LIST_BASE(CPP, TFJ, TFC, TFS, TFH, ASM)        \                          
  BUILTIN_LIST_FROM_TORQUE(CPP, TFJ, TFC, TFS, TFH, ASM) \                          
  BUILTIN_LIST_INTL(CPP, TFJ, TFS)                       \                          
  BUILTIN_LIST_BYTECODE_HANDLERS(BCH)     
```
Notice `BUILTIN_LIST_FROM_TORQUE`, this is how our MathIs42 gets included from
builtin-definitions-tq.h. This is in turn included by builtins.h.

If we take a look at the this header after it has gone through the preprocessor
we can see what has been generated for MathIs42:
```console
$ clang++ --sysroot=build/linux/debian_sid_amd64-sysroot -isystem=./buildtools/third_party/libc++/trunk/include -isystem=buildtools/third_party/libc++/trunk/include -I. -I./out/x64.release_gcc/gen/ -E src/builtins/builtins.h > builtins.h.pp
```
First MathIs42 will be come a member in the Name enum of the Builtins class:
```c++
class Builtins {
 public:

  enum Name : int32_t { 
    ...
    kMathIs42,
  };

  static void Generate_MathIs42(compiler::CodeAssemblerState* state); 
```
We should also take a look in `src/builtins/builtins-descriptors.h` as the BUILTIN_LIST
is used there two and specifically to our current example there is a
`DEFINE_TFJ_INTERFACE_DESCRIPTOR` macro used:
```c++
BUILTIN_LIST(IGNORE_BUILTIN, DEFINE_TFJ_INTERFACE_DESCRIPTOR,
             DEFINE_TFC_INTERFACE_DESCRIPTOR, DEFINE_TFS_INTERFACE_DESCRIPTOR,
             DEFINE_TFH_INTERFACE_DESCRIPTOR, IGNORE_BUILTIN,
             DEFINE_ASM_INTERFACE_DESCRIPTOR)

#define DEFINE_TFJ_INTERFACE_DESCRIPTOR(Name, Argc, ...)                \
  struct Builtin_##Name##_InterfaceDescriptor {                         \
    enum ParameterIndices {                                             \
      kJSTarget = compiler::CodeAssembler::kTargetParameterIndex,       \
      ##__VA_ARGS__,                                                    \
      kJSNewTarget,                                                     \
      kJSActualArgumentsCount,                                          \
      kContext,                                                         \
      kParameterCount,                                                  \
    };                                                                  \
  }; 
```
So the above will generate the following code but this time for builtins.cc:
```console
$ clang++ --sysroot=build/linux/debian_sid_amd64-sysroot -isystem=./buildtools/third_party/libc++/trunk/include -isystem=buildtools/third_party/libc++/trunk/include -I. -I./out/x64.release_gcc/gen/ -E src/builtins/builtins.cc > builtins.cc.pp
```

```c++
struct Builtin_MathIs42_InterfaceDescriptor { 
  enum ParameterIndices { 
    kJSTarget = compiler::CodeAssembler::kTargetParameterIndex,
    kReceiver,
    kX,
    kJSNewTarget,
    kJSActualArgumentsCount,
    kContext,
    kParameterCount,
  };

const BuiltinMetadata builtin_metadata[] = {
  ...
  {"MathIs42", Builtins::TFJ, {1, 0}}
  ...
};
```
BuiltinMetadata is a struct defined in builtins.cc and in our case the name 
is passed, then the type, and the last struct is specifying the number of parameters
and the last 0 is unused as far as I can tell and only there make it different
from the constructor that takes an Address parameter.

So, where is `Generate_MathIs42` used:
```c++
void SetupIsolateDelegate::SetupBuiltinsInternal(Isolate* isolate) {
  Code code;
  ...
  code = BuildWithCodeStubAssemblerJS(isolate, index, &Builtins::Generate_MathIs42, 1, "MathIs42");
  AddBuiltin(builtins, index++, code);
  ...
```
`BuildWithCodeStubAssemblerJS` can be found in `src/builtins/setup-builtins-internal.cc`
```c++
Code BuildWithCodeStubAssemblerJS(Isolate* isolate, int32_t builtin_index,
                                  CodeAssemblerGenerator generator, int argc,
                                  const char* name) {
  Zone zone(isolate->allocator(), ZONE_NAME);
  const int argc_with_recv = (argc == kDontAdaptArgumentsSentinel) ? 0 : argc + 1;
  compiler::CodeAssemblerState state(
      isolate, &zone, argc_with_recv, Code::BUILTIN, name,
      PoisoningMitigationLevel::kDontPoison, builtin_index);
  generator(&state);
  Handle<Code> code = compiler::CodeAssembler::GenerateCode(
      &state, BuiltinAssemblerOptions(isolate, builtin_index));
  return *code;
```
Lets add a conditional break point so that we can stop in this function when
`MathIs42` is passed in:
```console
(gdb) br setup-builtins-internal.cc:161
(gdb) cond 1 ((int)strcmp(name, "MathIs42")) == 0
```
We can see that we first create a new `CodeAssemblerState`, which we say previously
was that type that the `Generate_MathIs42` function takes. TODO: look into this class
a litte more.
After this `generator` will be called with the newly created state passed in:
```console 
(gdb) p generator
$8 = (v8::internal::(anonymous namespace)::CodeAssemblerGenerator) 0x5619fd61b66e <v8::internal::Builtins::Generate_MathIs42(v8::internal::compiler::CodeAssemblerState*)>
```
TODO: Take a closer look at generate and how that code works. 
After generate returns we will have the following call:
```c++
  generator(&state);                                                               
  Handle<Code> code = compiler::CodeAssembler::GenerateCode(                       
      &state, BuiltinAssemblerOptions(isolate, builtin_index));                    
  return *code;
```
Then next thing that will happen is the code returned will be added to the builtins
by calling `SetupIsolateDelegate::AddBuiltin`:
```c++
void SetupIsolateDelegate::AddBuiltin(Builtins* builtins, int index, Code code) {
  builtins->set_builtin(index, code);                                           
} 
```
`set_builtins` can be found in src/builtins/builtins.cc` and looks like this:
```c++
void Builtins::set_builtin(int index, Code builtin) {                           
  isolate_->heap()->set_builtin(index, builtin);                                
}
```
And Heap::set_builtin does:
```c++
 void Heap::set_builtin(int index, Code builtin) {
  isolate()->builtins_table()[index] = builtin.ptr();
}
```
So this is how the builtins_table is populated.
 
And when is `SetupBuiltinsInternal` called?  
It is called from `SetupIsolateDelegat::SetupBuiltins` which is called from Isolate::Init.

Just to recap before I loose track of what is going on...We have math.tq, which
is the torque source file. This is parsed by the torque compiler/parser and it
will generate c++ headers and source files, one of which will be a
CodeStubAssembler class for our MathI42 function. It will also generate the 
"torque-generated/builtin-definitions-tq.h. 
After this has happened the sources need to be compiled into object files. After
that if a snapshot is configured to be created, mksnapshot will create a new
Isolate and in that process the MathIs42 builtin will get added. Then a context will
be created and saved. The snapshot can then be deserialized into an Isoalte as
some later point.

Alright, so we have seen what gets generated for the function MathIs42 but how
does this get "hooked" but to enable us to call `Math.is42(11)`?  

In bootstrapper.cc we can see a number of lines:
```c++
 SimpleInstallFunction(isolate_, math, "trunc", Builtins::kMathTrunc, 1, true); 
```
And we are going to add a line like the following:
```c++
 SimpleInstallFunction(isolate_, math, "is42", Builtins::kMathIs42, 1, true);
```
The signature for `SimpleInstallFunction` looks like this
```c++
V8_NOINLINE Handle<JSFunction> SimpleInstallFunction(
    Isolate* isolate, Handle<JSObject> base, const char* name,
    Builtins::Name call, int len, bool adapt,
    PropertyAttributes attrs = DONT_ENUM) {
  Handle<String> internalized_name = isolate->factory()->InternalizeUtf8String(name);
  Handle<JSFunction> fun = SimpleCreateFunction(isolate, internalized_name, call, len, adapt);       
  JSObject::AddProperty(isolate, base, internalized_name, fun, attrs);          
  return fun;                                                                   
} 
```
So we see that the function is added as a property to the Math object.
Notice that we also have to add `kMathIs42` to the Builtins class which is now
part of the builtins_table_ array which we went through above.

#### Transitioning/Transient
In torgue source files we can sometimes see types declared as `transient`, and
functions that have a `transitioning` specifier. In V8 HeapObjects can change
at runtime (I think an example of this would be deleting an element in an array
which would transition it to a different type of array HoleyElementArray or
something like that. TODO: verify and explain this). And a function that calls
JavaScript which cause such a transition is marked with transitioning.

