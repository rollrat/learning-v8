
#### Callables
Are like functions is js/c++ but have some additional capabilities and there
are several different types of callables:

##### macro callables
These correspond to generated CodeStubAssebler C++ that will be inlined at
the callsite.

##### builtin callables
These will become V8 builtins with info added to builtin-definitions.h (via
the include of torque-generated/builtin-definitions-tq.h). There is only one
copy of this and this will be a call instead of being inlined as is the case
with macros.

##### runtime callables

##### intrinsic callables

#### Explicit parameters
macros and builtins can have parameters. For example:
```
@export
macro HelloWorld1(msg: JSAny) {
  Print(msg);
}
```
And we can call this from another macro like this:
```
@export
macro HelloWorld() {
  HelloWorld1('Hello World');
}
```

#### Implicit parameters
In the previous section we showed explicit parameters but we can also have
implicit parameters:
```
@export
macro HelloWorld2(implicit msg: JSAny)() {
  Print(msg);
}
@export
macro HelloWorld() {
  const msg = 'Hello implicit';
  HelloWorld2();
}
```


### Troubleshooting
Compilation error when including `src/objects/objects-inl.h:
```console
/home/danielbevenius/work/google/v8_src/v8/src/objects/object-macros.h:263:14: error: no declaration matches ‘bool v8::internal::HeapObject::IsJSCollator() const’
```
Does this need i18n perhaps?
```console
$ gn args --list out/x64.release_gcc | grep i18n
v8_enable_i18n_support
```

```console
usr/bin/ld: /tmp/ccJOrUMl.o: in function `v8::internal::MaybeHandle<v8::internal::Object>::Check() const':
/home/danielbevenius/work/google/v8_src/v8/src/handles/maybe-handles.h:44: undefined reference to `V8_Fatal(char const*, ...)'
collect2: error: ld returned 1 exit status
```
V8_Fatal is referenced but not defined in v8_monolith.a:
```console
$ nm libv8_monolith.a | grep V8_Fatal | c++filt 
...
U V8_Fatal(char const*, int, char const*, ...)
```
And I thought it might be defined in libv8_libbase.a but it is the same there.
Actually, I was looking at the wrong symbol. This was not from the logging.o 
object file. If we look at it we find:
```console
v8_libbase/logging.o:
...
0000000000000000 T V8_Fatal(char const*, int, char const*, ...)
```
In out/x64.release/obj/logging.o we can find it defined:
```console
$ nm -C  libv8_libbase.a | grep -A 50 logging.o | grep V8_Fatal
0000000000000000 T V8_Fatal(char const*, int, char const*, ...)
```
`T` means that the symbol is in the text section.
So if the linker is able to find libv8_libbase.a it should be able to resolve
this.

So we need to make sure the linker can find the directory where the libraries
are located ('-Wl,-Ldir'), and also that it will include the library ('-Wl,-llibname')

With this in place I can see that the linker can open the archive:
```console
attempt to open /home/danielbevenius/work/google/v8_src/v8/out/x64.release_gcc/obj/libv8_libbase.so failed
attempt to open /home/danielbevenius/work/google/v8_src/v8/out/x64.release_gcc/obj/libv8_libbase.a succeeded
/home/danielbevenius/work/google/v8_src/v8/out/x64.release_gcc/obj/libv8_libbase.a
```
But I'm still getting the same linking error. If we look closer at the error message
we can see that it is maybe-handles.h that is complaining. Could it be that the
order is incorrect when linking. libv8_libbase.a needs to come after libv8_monolith
Something I noticed is that even though the library libv8_libbase.a is found it
does not look like the linker actually reads the object files. I can see that it
does this for libv8_monolith.a:
```console
(/home/danielbevenius/work/google/v8_src/v8/out/x64.release_gcc/obj/libv8_monolith.a)common-node-cache.o
```
Hmm, actually looking at the signature of the function it is V8_Fatal(char const*, ...)
and not char const*, int, char const*, ...)

For a debug build it will be:
```
    void V8_Fatal(const char* file, int line, const char* format, ...);
```
And else
```
    void V8_Fatal(const char* format, ...);
```
So it looks like I need to set debug to false. With this the V8_Fatal symbol
in logging.o is:
```console
$ nm -C out/x64.release_gcc/obj/v8_libbase/logging.o | grep V8_Fatal
0000000000000000 T V8_Fatal(char const*, ...)
```


### V8 Build artifacts
What is actually build when you specify 
v8_monolithic:
When this type is chosen the build cannot be a component build, there is an
assert for this. In this case a static library build:
```
if (v8_monolithic) {                                                            
  # A component build is not monolithic.                                        
  assert(!is_component_build)                                                   
                                                                                
  # Using external startup data would produce separate files.                   
  assert(!v8_use_external_startup_data)                                         
  v8_static_library("v8_monolith") {                                            
    deps = [                                                                    
      ":v8",                                                                    
      ":v8_libbase",                                                            
      ":v8_libplatform",                                                        
      ":v8_libsampler",                                                         
      "//build/win:default_exe_manifest",                                       
    ]                                                                           
                                                                                
    configs = [ ":internal_config" ]                                            
  }                                                                             
}
```
Notice that the builtin function is called `static_library` so is a template
that can be found in `gni/v8.gni` 

v8_static_library:
This will use source_set instead of creating a static library when compiling.
When set to false, the object files that would be included in the linker command.
The can speed up the build as the creation of the static libraries is skipped.
But this does not really help when linking to v8 externally as from this project.

is_component_build:
This will compile targets declared as components as shared libraries.
All the v8_components in BUILD.gn will be built as .so files in the output
director (not the obj directory which is the case for static libraries).

So the only two options are the v8_monolith or is_component_build where it
might be an advantage of being able to build a single component and not have
to rebuild the whole monolith at times.


### V8 Internal Isolate
`src/execution/isolate.h` is where you can find the v8::internal::Isolate.
```c++
class V8_EXPORT_PRIVATE Isolate final : private HiddenFactory {

```
And HiddenFactory is just to allow Isolate to inherit privately from Factory
which can be found in src/heap/factory.h.

### Startup Walk through
This section will walk through the start up on V8 by using the hello_world example
in this project:
```console
$ LD_LIBRARY_PATH=../v8_src/v8/out/x64.release_gcc/ lldb ./hello-world
(lldb) br s -n main
Breakpoint 1: where = hello-world`main + 25 at hello-world.cc:41:38, address = 0x0000000000402821
```
```console
    V8::InitializeExternalStartupData(argv[0]);
```
This call will land in `api.cc` which will just delegate the call to and internal
(internal namespace that is). If you try to step into this function you will
just land on the next line in hello_world. This is because we compiled v8 without
external start up data so this function will be empty:
```console
$ objdump -Cd out/x64.release_gcc/obj/v8_base_without_compiler/startup-data-util.o
Disassembly of section .text._ZN2v88internal37InitializeExternalStartupDataFromFileEPKc:

0000000000000000 <v8::internal::InitializeExternalStartupDataFromFile(char const*)>:
   0:	c3                   	retq
```
Next, we have:
```console
    std::unique_ptr<Platform> platform = platform::NewDefaultPlatform();
```
This will land in `src/libplatform/default-platform.cc` which will create a new
DefaultPlatform.

```c++
Isolate* isolate = Isolate::New(create_params);
```
This will call Allocate: 
```c++
Isolate* isolate = Allocate();
```
```c++
Isolate* Isolate::Allocate() {
  return reinterpret_cast<Isolate*>(i::Isolate::New());
}
```

Remember that the internal Isolate can be found in `src/execution/isolate.h`.
In `src/execution/isolate.cc` we find `Isolate::New`
```c++
Isolate* Isolate::New(IsolateAllocationMode mode) {
  std::unique_ptr<IsolateAllocator> isolate_allocator = std::make_unique<IsolateAllocator>(mode);
  void* isolate_ptr = isolate_allocator->isolate_memory();
  Isolate* isolate = new (isolate_ptr) Isolate(std::move(isolate_allocator));
```
So we first create an IsolateAllocator instance which will allocate memory for
a single Isolate instance. This is then passed into the Isolate constructor,
notice the usage of `new` here, this is just a normal heap allocation. 

The default new operator has been deleted and an override provided that takes
a void pointer, which is just returned: 
```c++
  void* operator new(size_t, void* ptr) { return ptr; }
  void* operator new(size_t) = delete;
  void operator delete(void*) = delete;
```
In this case it just returns the memory allocateed by isolate-memory().
The reason for doing this is that using the new operator not only invokes the
new operator but the compiler will also add a call the types constructor passing
in the address of the allocated memory.
```c++
Isolate::Isolate(std::unique_ptr<i::IsolateAllocator> isolate_allocator)
    : isolate_data_(this),
      isolate_allocator_(std::move(isolate_allocator)),
      id_(isolate_counter.fetch_add(1, std::memory_order_relaxed)),
      allocator_(FLAG_trace_zone_stats
                     ? new VerboseAccountingAllocator(&heap_, 256 * KB)
                     : new AccountingAllocator()),
      builtins_(this),
      rail_mode_(PERFORMANCE_ANIMATION),
      code_event_dispatcher_(new CodeEventDispatcher()),
      jitless_(FLAG_jitless),
#if V8_SFI_HAS_UNIQUE_ID
      next_unique_sfi_id_(0),
#endif
      cancelable_task_manager_(new CancelableTaskManager()) {
```
Notice that  `isolate_data_` will be populated by calling the constructor which
takes an pointer to an Isolate.
```c++
class IsolateData final {
 public:
  explicit IsolateData(Isolate* isolate) : stack_guard_(isolate) {}
```

Back in Isolate's constructor we have:
```c++
#define ISOLATE_INIT_LIST(V)                                                   \
  /* Assembler state. */                                                       \
  V(FatalErrorCallback, exception_behavior, nullptr)                           \
  ...

#define ISOLATE_INIT_EXECUTE(type, name, initial_value) \                           
  name##_ = (initial_value);                                                        
  ISOLATE_INIT_LIST(ISOLATE_INIT_EXECUTE)                                           
#undef ISOLATE_INIT_EXECUTE
```
So lets expand the first entry to understand what is going on:
```c++
   exception_behavior_ = (nullptr);
   oom_behavior_ = (nullptr);
   event_logger_ = (nullptr);
   allow_code_gen_callback_ = (nullptr);
   modify_code_gen_callback_ = (nullptr);
   allow_wasm_code_gen_callback_ = (nullptr);
   wasm_module_callback_ = (&NoExtension);
   wasm_instance_callback_ = (&NoExtension);
   wasm_streaming_callback_ = (nullptr);
   wasm_threads_enabled_callback_ = (nullptr);
   wasm_load_source_map_callback_ = (nullptr);
   relocatable_top_ = (nullptr);
   string_stream_debug_object_cache_ = (nullptr);
   string_stream_current_security_token_ = (Object());
   api_external_references_ = (nullptr);
   external_reference_map_ = (nullptr);
   root_index_map_ = (nullptr);
   default_microtask_queue_ = (nullptr);
   turbo_statistics_ = (nullptr);
   code_tracer_ = (nullptr);
   per_isolate_assert_data_ = (0xFFFFFFFFu);
   promise_reject_callback_ = (nullptr);
   snapshot_blob_ = (nullptr);
   code_and_metadata_size_ = (0);
   bytecode_and_metadata_size_ = (0);
   external_script_source_size_ = (0);
   is_profiling_ = (false);
   num_cpu_profilers_ = (0);
   formatting_stack_trace_ = (false);
   debug_execution_mode_ = (DebugInfo::kBreakpoints);
   code_coverage_mode_ = (debug::CoverageMode::kBestEffort);
   type_profile_mode_ = (debug::TypeProfileMode::kNone);
   last_stack_frame_info_id_ = (0);
   last_console_context_id_ = (0);
   inspector_ = (nullptr);
   next_v8_call_is_safe_for_termination_ = (false);
   only_terminate_in_safe_scope_ = (false);
   detailed_source_positions_for_profiling_ = (FLAG_detailed_line_info);
   embedder_wrapper_type_index_ = (-1);
   embedder_wrapper_object_index_ = (-1);
```
So all of the entries in this list will become private members of the
Isolate class after the preprocessor is finished. There will also be public
assessor to get and set these initial values values (which is the last entry
in the ISOLATE_INIT_LIST above.

Back in isolate.cc constructor we have:
```c++
#define ISOLATE_INIT_ARRAY_EXECUTE(type, name, length) \
  memset(name##_, 0, sizeof(type) * length);
  ISOLATE_INIT_ARRAY_LIST(ISOLATE_INIT_ARRAY_EXECUTE)
#undef ISOLATE_INIT_ARRAY_EXECUTE
#define ISOLATE_INIT_ARRAY_LIST(V)                                             \
  /* SerializerDeserializer state. */                                          \
  V(int32_t, jsregexp_static_offsets_vector, kJSRegexpStaticOffsetsVectorSize) \
  ...

  InitializeDefaultEmbeddedBlob();
  MicrotaskQueue::SetUpDefaultMicrotaskQueue(this);
```
After that we have created a new Isolate, we were in this function call:
```c++
  Isolate* isolate = new (isolate_ptr) Isolate(std::move(isolate_allocator));
```
After this we will be back in `api.cc`:
```c++
  Initialize(isolate, params);
```
```c++
void Isolate::Initialize(Isolate* isolate,
                         const v8::Isolate::CreateParams& params) {
```
We are not using any external snapshot data so the following will be false:
```c++
  if (params.snapshot_blob != nullptr) {
    i_isolate->set_snapshot_blob(params.snapshot_blob);
  } else {
    i_isolate->set_snapshot_blob(i::Snapshot::DefaultSnapshotBlob());
```
```console
(gdb) p snapshot_blob_
$7 = (const v8::StartupData *) 0x0
(gdb) n
(gdb) p i_isolate->snapshot_blob_
$8 = (const v8::StartupData *) 0x7ff92d7d6cf0 <v8::internal::blob>
```
`snapshot_blob_` is also one of the members that was set up with ISOLATE_INIT_LIST.
So we are setting up the Isolate instance for creation. 

```c++
Isolate::Scope isolate_scope(isolate);                                        
if (!i::Snapshot::Initialize(i_isolate)) { 
```
In `src/snapshot/snapshot-common.cc` we find 
```c++
bool Snapshot::Initialize(Isolate* isolate) {
  ...
  const v8::StartupData* blob = isolate->snapshot_blob();
  Vector<const byte> startup_data = ExtractStartupData(blob);
  Vector<const byte> read_only_data = ExtractReadOnlyData(blob);
  SnapshotData startup_snapshot_data(MaybeDecompress(startup_data));
  SnapshotData read_only_snapshot_data(MaybeDecompress(read_only_data));
  StartupDeserializer startup_deserializer(&startup_snapshot_data);
  ReadOnlyDeserializer read_only_deserializer(&read_only_snapshot_data);
  startup_deserializer.SetRehashability(ExtractRehashability(blob));
  read_only_deserializer.SetRehashability(ExtractRehashability(blob));

  bool success = isolate->InitWithSnapshot(&read_only_deserializer, &startup_deserializer);
```
So we get the blob and create deserializers for it which are then passed to
`isolate->InitWithSnapshot` which delegated to `Isolate::Init`. The blob will
have be create previously using `mksnapshot` (more on this can be found later).

This will use a `FOR_EACH_ISOLATE_ADDRESS_NAME` macro to assign to the
`isolate_addresses_` field:
```c++
isolate_addresses_[IsolateAddressId::kHandlerAddress] = reinterpret_cast<Address>(handler_address());
isolate_addresses_[IsolateAddressId::kCEntryFPAddress] = reinterpret_cast<Address>(c_entry_fp_address());
isolate_addresses_[IsolateAddressId::kCFunctionAddress] = reinterpret_cast<Address>(c_function_address());
isolate_addresses_[IsolateAddressId::kContextAddress] = reinterpret_cast<Address>(context_address());
isolate_addresses_[IsolateAddressId::kPendingExceptionAddress] = reinterpret_cast<Address>(pending_exception_address());
isolate_addresses_[IsolateAddressId::kPendingHandlerContextAddress] = reinterpret_cast<Address>(pending_handler_context_address());
 isolate_addresses_[IsolateAddressId::kPendingHandlerEntrypointAddress] = reinterpret_cast<Address>(pending_handler_entrypoint_address());
 isolate_addresses_[IsolateAddressId::kPendingHandlerConstantPoolAddress] = reinterpret_cast<Address>(pending_handler_constant_pool_address());
 isolate_addresses_[IsolateAddressId::kPendingHandlerFPAddress] = reinterpret_cast<Address>(pending_handler_fp_address());
 isolate_addresses_[IsolateAddressId::kPendingHandlerSPAddress] = reinterpret_cast<Address>(pending_handler_sp_address());
 isolate_addresses_[IsolateAddressId::kExternalCaughtExceptionAddress] = reinterpret_cast<Address>(external_caught_exception_address());
 isolate_addresses_[IsolateAddressId::kJSEntrySPAddress] = reinterpret_cast<Address>(js_entry_sp_address());
```
After this we have a number of members that are assigned to:
```c++
  compilation_cache_ = new CompilationCache(this);
  descriptor_lookup_cache_ = new DescriptorLookupCache();
  inner_pointer_to_code_cache_ = new InnerPointerToCodeCache(this);
  global_handles_ = new GlobalHandles(this);
  eternal_handles_ = new EternalHandles();
  bootstrapper_ = new Bootstrapper(this);
  handle_scope_implementer_ = new HandleScopeImplementer(this);
  load_stub_cache_ = new StubCache(this);
  store_stub_cache_ = new StubCache(this);
  materialized_object_store_ = new MaterializedObjectStore(this);
  regexp_stack_ = new RegExpStack();
  regexp_stack_->isolate_ = this;
  date_cache_ = new DateCache();
  heap_profiler_ = new HeapProfiler(heap());
  interpreter_ = new interpreter::Interpreter(this);
  compiler_dispatcher_ =
      new CompilerDispatcher(this, V8::GetCurrentPlatform(), FLAG_stack_size);
```
After this we have:
```c++
isolate_data_.external_reference_table()->Init(this);
```
This will land in `src/codegen/external-reference-table.cc` where we have:
```c++
void ExternalReferenceTable::Init(Isolate* isolate) {                              
  int index = 0;                                                                   
  Add(kNullAddress, &index);                                                       
  AddReferences(isolate, &index);                                                  
  AddBuiltins(&index);                                                             
  AddRuntimeFunctions(&index);                                                     
  AddIsolateAddresses(isolate, &index);                                            
  AddAccessors(&index);                                                            
  AddStubCache(isolate, &index);                                                   
  AddNativeCodeStatsCounters(isolate, &index);                                     
  is_initialized_ = static_cast<uint32_t>(true);                                   
                                                                                   
  CHECK_EQ(kSize, index);                                                          
}

void ExternalReferenceTable::Add(Address address, int* index) {                 
ref_addr_[(*index)++] = address;                                                
} 

Address ref_addr_[kSize];
```

Now, lets take a look at `AddReferences`: 
```c++
Add(ExternalReference::abort_with_reason().address(), index); 
```
What are ExternalReferences?   
They represent c++ addresses used in generated code.

After that we have AddBuiltins:
```c++
static const Address c_builtins[] = {                                         
      (reinterpret_cast<v8::internal::Address>(&Builtin_HandleApiCall)), 
      ...

Address Builtin_HandleApiCall(int argc, Address* args, Isolate* isolate);
```
I can see that the function declaration is in external-reference.h but the
implementation is not there. Instead this is defined in `src/builtins/builtins-api.cc`:
```c++
BUILTIN(HandleApiCall) {                                                           
(will expand to:)

V8_WARN_UNUSED_RESULT static Object Builtin_Impl_HandleApiCall(
      BuiltinArguments args, Isolate* isolate);

V8_NOINLINE static Address Builtin_Impl_Stats_HandleApiCall(
      int args_length, Address* args_object, Isolate* isolate) {
    BuiltinArguments args(args_length, args_object);
    RuntimeCallTimerScope timer(isolate,
                                RuntimeCallCounterId::kBuiltin_HandleApiCall);
    TRACE_EVENT0(TRACE_DISABLED_BY_DEFAULT("v8.runtime"), "V8.Builtin_HandleApiCall");
    return CONVERT
}
V8_WARN_UNUSED_RESULT Address Builtin_HandleApiCall(
      int args_length, Address* args_object, Isolate* isolate) {
    DCHECK(isolate->context().is_null() || isolate->context().IsContext());
    if (V8_UNLIKELY(TracingFlags::is_runtime_stats_enabled())) {
      return Builtin_Impl_Stats_HandleApiCall(args_length, args_object, isolate);
    }
    BuiltinArguments args(args_length, args_object);
    return CONVERT_OBJECT(Builtin_Impl_HandleApiCall(args, isolate));
  }

  V8_WARN_UNUSED_RESULT static Object Builtin_Impl_HandleApiCall(
      BuiltinArguments args, Isolate* isolate) {
    HandleScope scope(isolate);                                                      
    Handle<JSFunction> function = args.target();                                  
    Handle<Object> receiver = args.receiver();                                    
    Handle<HeapObject> new_target = args.new_target();                               
    Handle<FunctionTemplateInfo> fun_data(function->shared().get_api_func_data(), 
                                        isolate);                                  
    if (new_target->IsJSReceiver()) {                                                
      RETURN_RESULT_OR_FAILURE(                                                   
          isolate, HandleApiCallHelper<true>(isolate, function, new_target,          
                                             fun_data, receiver, args));             
    } else {                                                                         
      RETURN_RESULT_OR_FAILURE(                                                      
          isolate, HandleApiCallHelper<false>(isolate, function, new_target,         
                                            fun_data, receiver, args));            
    }
  }
``` 
The `BUILTIN` macro can be found in `src/builtins/builtins-utils.h`:
```c++
#define BUILTIN(name)                                                       \
  V8_WARN_UNUSED_RESULT static Object Builtin_Impl_##name(                  \
      BuiltinArguments args, Isolate* isolate);
```

```c++
  if (setup_delegate_ == nullptr) {                                                 
    setup_delegate_ = new SetupIsolateDelegate(create_heap_objects);            
  } 

  if (!setup_delegate_->SetupHeap(&heap_)) {                                    
    V8::FatalProcessOutOfMemory(this, "heap object creation");                  
    return false;                                                               
  }    
```
This does nothing in the current code path and the code comment says that the
heap will be deserialized from the snapshot and true will be returned.

```c++
InitializeThreadLocal();
startup_deserializer->DeserializeInto(this);
```
```c++
DisallowHeapAllocation no_gc;                                               
isolate->heap()->IterateSmiRoots(this);                                     
isolate->heap()->IterateStrongRoots(this, VISIT_FOR_SERIALIZATION);         
Iterate(isolate, this);                                                     
isolate->heap()->IterateWeakRoots(this, VISIT_FOR_SERIALIZATION);           
DeserializeDeferredObjects();                                               
RestoreExternalReferenceRedirectors(accessor_infos());                      
RestoreExternalReferenceRedirectors(call_handler_infos());
```
In `heap.cc` we find IterateSmiRoots` which takes a pointer to a `RootVistor`.
RootVisitor is used for visiting and modifying (optionally) the pointers contains
in roots. This is used in garbage collection and also in serializing and deserializing
snapshots.

### Roots
RootVistor:
```c++
class RootVisitor {
 public:
  virtual void VisitRootPointers(Root root, const char* description,
                                 FullObjectSlot start, FullObjectSlot end) = 0;

  virtual void VisitRootPointer(Root root, const char* description,
                                FullObjectSlot p) {
    VisitRootPointers(root, description, p, p + 1);
  }
 
  static const char* RootName(Root root);
```
Root is an enum in `src/object/visitors.h`. This enum is generated by a macro
and expands to:
```c++
enum class Root {                                                               
  kStringTable,
  kExternalStringsTable,
  kReadOnlyRootList,
  kStrongRootList,
  kSmiRootList,
  kBootstrapper,
  kTop,
  kRelocatable,
  kDebug,
  kCompilationCache,
  kHandleScope,
  kBuiltins,
  kGlobalHandles,
  kEternalHandles,
  kThreadManager,
  kStrongRoots,
  kExtensions,
  kCodeFlusher,
  kPartialSnapshotCache,
  kReadOnlyObjectCache,
  kWeakCollections,
  kWrapperTracing,
  kUnknown,
  kNumberOfRoots                                                            
}; 
```
These can be displayed using:
```console
$ ./test/roots_test --gtest_filter=RootsTest.visitor_roots
```
Just to keep things clear for myself here, these visitor roots are only used
for GC and serialization/deserialization (at least I think so) and should not
be confused with the RootIndex enum in `src/roots/roots.h`.

Lets set a break point in `mksnapshot` and see if we can find where one of the
above Root enum elements is used to make it a little more clear what these are
used for.
```console
$ lldb ../v8_src/v8/out/x64.debug/mksnapshot 
(lldb) target create "../v8_src/v8/out/x64.debug/mksnapshot"
Current executable set to '../v8_src/v8/out/x64.debug/mksnapshot' (x86_64).
(lldb) br s -n main
Breakpoint 1: where = mksnapshot`main + 42, address = 0x00000000009303ca
(lldb) r
```
What this does is that it creates an V8 environment (Platform, Isolate, Context)
 and then saves it to a file, either a binary file on disk but it can also save
it to a .cc file that can be used in programs in which case the binary is a byte array.
It does this in much the same way as the hello-world example create a platform
and then initializes it, and the creates and initalizes a new Isolate. 
After the Isolate a new Context will be create using the Isolate. If there was
an embedded-src flag passed to mksnaphot it will be run.

StartupSerializer will use the Root enum elements for example and the deserializer
will use the same enum elements.

Adding a script to a snapshot:
```
$ gdb ../v8_src/v8/out/x64.release_gcc/mksnapshot --embedded-src="$PWD/embed.js"
```

TODO: Look into CreateOffHeapTrampolines.

So the VisitRootPointers function takes one of these Root's and visits all those
roots.  In our case the first Root to be visited is Heap::IterateSmiRoots:
```c++
void Heap::IterateSmiRoots(RootVisitor* v) {                                        
  ExecutionAccess access(isolate());                                                
  v->VisitRootPointers(Root::kSmiRootList, nullptr,                                 
                       roots_table().smi_roots_begin(),                             
                       roots_table().smi_roots_end());                              
  v->Synchronize(VisitorSynchronization::kSmiRootList);                             
}
```
And here we can see that it is using `Root::kSmiRootList`, and passing nullptr
for the description argument (I wonder what this is used for?). Next, comes
the start and end arguments. 
```console
(lldb) p roots_table().smi_roots_begin()
(v8::internal::FullObjectSlot) $5 = {
  v8::internal::SlotBase<v8::internal::FullObjectSlot, unsigned long, 8> = (ptr_ = 50680614097760)
}
```
We can list all the values of roots_table using:
```console
(lldb) expr -A -- roots_table()
```
In `src/snapshot/deserializer.cc` we can find VisitRootPointers:
```c++
void Deserializer::VisitRootPointers(Root root, const char* description,
                                     FullObjectSlot start, FullObjectSlot end)
  ReadData(FullMaybeObjectSlot(start), FullMaybeObjectSlot(end),
           SnapshotSpace::kNew, kNullAddress);
```
Notice that description is never used. `ReadData`is in the same source file:

The class SnapshotByteSource has a `data` member that is initialized upon construction
from a const char* or a Vector<const byte>. Where is this done?  
This was done back in `Snapshot::Initialize`:
```c++
  const v8::StartupData* blob = isolate->snapshot_blob();                       
  Vector<const byte> startup_data = ExtractStartupData(blob);                   
  Vector<const byte> read_only_data = ExtractReadOnlyData(blob);                
  SnapshotData startup_snapshot_data(MaybeDecompress(startup_data));            
  SnapshotData read_only_snapshot_data(MaybeDecompress(read_only_data));        
  StartupDeserializer startup_deserializer(&startup_snapshot_data); 
```
```console
(lldb) expr *this
(v8::internal::SnapshotByteSource) $30 = (data_ = "`\x04", length_ = 125752, position_ = 1)
```

All the roots in a heap are declared in src/roots/roots.h. You can access the
roots using RootsTable via the Isolate using isolate_data->roots() or by using
isolate->roots_table. The roots_ field is an array of Address elements:
```c++
class RootsTable {                                                              
 public:
  static constexpr size_t kEntriesCount = static_cast<size_t>(RootIndex::kRootListLength);
  ...
 private:
  Address roots_[kEntriesCount];                                                
  static const char* root_names_[kEntriesCount]; 
```
RootIndex is generated by a macro
```c++
enum class RootIndex : uint16_t {
```
The complete enum can be displayed using:
```console
$ ./test/roots_test --gtest_filter=RootsTest.list_root_index
```

Lets take a look at an entry:
```console
(lldb) p roots_[(uint16_t)RootIndex::kError_string]
(v8::internal::Address) $1 = 42318447256121
```
Now, there are functions in factory which can be used to retrieve these addresses,
like factory->Error_string():
```console
(lldb) expr *isolate->factory()->Error_string()
(v8::internal::String) $9 = {
  v8::internal::TorqueGeneratedString<v8::internal::String, v8::internal::Name> = {
    v8::internal::Name = {
      v8::internal::TorqueGeneratedName<v8::internal::Name, v8::internal::PrimitiveHeapObject> = {
        v8::internal::PrimitiveHeapObject = {
          v8::internal::TorqueGeneratedPrimitiveHeapObject<v8::internal::PrimitiveHeapObject, v8::internal::HeapObject> = {
            v8::internal::HeapObject = {
              v8::internal::Object = {
                v8::internal::TaggedImpl<v8::internal::HeapObjectReferenceType::STRONG, unsigned long> = (ptr_ = 42318447256121)
              }
            }
          }
        }
      }
    }
  }
}
(lldb) expr $9.length()
(int32_t) $10 = 5
(lldb) expr $9.Print()
#Error
```
These accessor functions declarations are generated by the
`ROOT_LIST(ROOT_ACCESSOR))` macros:
```c++
#define ROOT_ACCESSOR(Type, name, CamelName) inline Handle<Type> name();           
  ROOT_LIST(ROOT_ACCESSOR)                                                         
#undef ROOT_ACCESSOR
```
And the definitions can be found in `src/heap/factory-inl.h` and look like this
The implementations then look like this:
```c++
String ReadOnlyRoots::Error_string() const { 
  return  String::unchecked_cast(Object(at(RootIndex::kError_string)));
} 

Handle<String> ReadOnlyRoots::Error_string_handle() const {
  return Handle<String>(&at(RootIndex::kError_string)); 
}
```
The unit test [roots_test](./test/roots_test.cc) shows and example of this.

This shows the usage of root entries but where are the roots added to this
array. `roots_` is a member of `IsolateData` in `src/execution/isolate-data.h`:
```
  RootsTable roots_;
```
We can inspect the roots_ content by using the interal Isolate:
```
(lldb) f
frame #0: 0x00007ffff6261cdf libv8.so`v8::Isolate::Initialize(isolate=0x00000eb900000000, params=0x00007fffffffd0d0) at api.cc:8269:31
   8266	void Isolate::Initialize(Isolate* isolate,
   8267	                         const v8::Isolate::CreateParams& params) {

(lldb) expr i_isolate->isolate_data_.roots_
(v8::internal::RootsTable) $5 = {
  roots_ = {
    [0] = 0
    [1] = 0
    [2] = 0
```
So we can see that the roots are intially zero:ed out. And the type of `roots_`
is an array of `Address`'s.
```console
    frame #3: 0x00007ffff6c33d58 libv8.so`v8::internal::Deserializer::VisitRootPointers(this=0x00007fffffffcce0, root=kReadOnlyRootList, description=0x0000000000000000, start=FullObjectSlot @ 0x00007fffffffc530, end=FullObjectSlot @ 0x00007fffffffc528) at deserializer.cc:94:11
    frame #4: 0x00007ffff6b6212f libv8.so`v8::internal::ReadOnlyRoots::Iterate(this=0x00007fffffffc5c8, visitor=0x00007fffffffcce0) at roots.cc:21:29
    frame #5: 0x00007ffff6c46fee libv8.so`v8::internal::ReadOnlyDeserializer::DeserializeInto(this=0x00007fffffffcce0, isolate=0x00000f7500000000) at read-only-deserializer.cc:41:18
    frame #6: 0x00007ffff66af631 libv8.so`v8::internal::ReadOnlyHeap::DeseralizeIntoIsolate(this=0x000000000049afb0, isolate=0x00000f7500000000, des=0x00007fffffffcce0) at read-only-heap.cc:85:23
    frame #7: 0x00007ffff66af5de libv8.so`v8::internal::ReadOnlyHeap::SetUp(isolate=0x00000f7500000000, des=0x00007fffffffcce0) at read-only-heap.cc:78:53
```
This will land us in `roots.cc` ReadOnlyRoots::Iterate(RootVisitor* visitor):
```c++
void ReadOnlyRoots::Iterate(RootVisitor* visitor) {                                
  visitor->VisitRootPointers(Root::kReadOnlyRootList, nullptr,                     
                             FullObjectSlot(read_only_roots_),                     
                             FullObjectSlot(&read_only_roots_[kEntriesCount])); 
  visitor->Synchronize(VisitorSynchronization::kReadOnlyRootList);                 
} 
```
Deserializer::VisitRootPointers calls `Deserializer::ReadData` and the roots_
array is still zero:ed out when we enter this function.

```c++
void Deserializer::VisitRootPointers(Root root, const char* description,
                                     FullObjectSlot start, FullObjectSlot end) {
  ReadData(FullMaybeObjectSlot(start), FullMaybeObjectSlot(end),
           SnapshotSpace::kNew, kNullAddress);
```
Notice that we called VisitRootPointer and pased in `Root:kReadOnlyRootList`, 
nullptr (the description), and start and end addresses as FullObjectSlots. The
signature of `VisitRootPointers` looks like this:
```c++
virtual void VisitRootPointers(Root root, const char* description,            
                                 FullObjectSlot start, FullObjectSlot end)
```
In our case we are using the address of `read_only_roots_` from `src/roots/roots.h`
and the end is found by using the static member of ReadOnlyRoots::kEntrysCount.

The switch statement in `ReadData` is generated by macros so lets take a look at
an expanded snippet to understand what is going on:
```c++
template <typename TSlot>
bool Deserializer::ReadData(TSlot current, TSlot limit,
                            SnapshotSpace source_space,
                            Address current_object_address) {
  Isolate* const isolate = isolate_;
  ...
  while (current < limit) {                                                     
    byte data = source_.Get();                                                  
```
So current is the start address of the read_only_list and limit the end. `source_`
is a member of `ReadOnlyDeserializer` and is of type SnapshotByteSource.

`source_` got populated back in Snapshot::Initialize(internal_isolate):
```
const v8::StartupData* blob = isolate->snapshot_blob();
Vector<const byte> read_only_data = ExtractReadOnlyData(blob);
ReadOnlyDeserializer read_only_deserializer(&read_only_snapshot_data);
```
And `ReadOnlyDeserializer` extends `Deserialier` (src/snapshot/deserializer.h)
which has a constructor that sets the source_ member to data->Payload().
So `source_` is will be pointer to an instance of `SnapshotByteSource` which
can be found in `src/snapshot-source-sink.h`:
```c++
class SnapshotByteSource final {
 public:
  SnapshotByteSource(const char* data, int length)
      : data_(reinterpret_cast<const byte*>(data)),
        length_(length),
        position_(0) {}

  byte Get() {                                                                  
    return data_[position_++];                                                  
  }
  ...
 private:
  const byte* data_;
  int length_;
  int posistion_;
```
Alright, so we are calling source_.Get() which we can see returns the current
entry from the byte array data_ and increment the position. So with that in
mind lets take closer look at the switch statment:
```c++
  while (current < limit) {                                                     
    byte data = source_.Get();                                                  
    switch (data) {                                                             
      case kNewObject + static_cast<int>(SnapshotSpace::kNew):
        current = ReadDataCase<TSlot, kNewObject, SnapshotSpace::kNew>(isolate, current, current_object_address, data, write_barrier_needed);
        break;
      case kNewObject + static_cast<int>(SnapshotSpace::kOld):
        [[clang::fallthrough]];
      case kNewObject + static_cast<int>(SnapshotSpace::kCode):
        [[clang::fallthrough]];
      case kNewObject + static_cast<int>(SnapshotSpace::kMap):
        static_assert((static_cast<int>(SnapshotSpace::kMap) & ~kSpaceMask) == 0, "(static_cast<int>(SnapshotSpace::kMap) & ~kSpaceMask) == 0");
        [[clang::fallthrough]];
      ...
```
We can see that switch statement will assign the passed-in `current` with a new
instance of `ReadDataCase`.
```c++
  current = ReadDataCase<TSlot, kNewObject, SnapshotSpace::kNew>(isolate,
      current, current_object_address, data, write_barrier_needed);
```
Notice that kNewObject is the type of SerializerDeserliazer::Bytecode that is
to be read (I think), this enum can be found in `src/snapshot/serializer-common.h`.
`TSlot` I think stands for the "Type of Slot", which in our case is a FullMaybyObjectSlot.
```c++
  HeapObject heap_object;
  if (bytecode == kNewObject) {                                                 
    heap_object = ReadObject(space);   
```
ReadObject is also in deserializer.cc :
```c++
Address address = allocator()->Allocate(space, size);
HeapObject obj = HeapObject::FromAddress(address);
isolate_->heap()->OnAllocationEvent(obj, size);

Alright, lets set a watch point on the roots_ array to see when the first entry
is populated and try to figure this out that way:
```console
(lldb) watch set variable  isolate->isolate_data_.roots_.roots_[0]
Watchpoint created: Watchpoint 5: addr = 0xf7500000080 size = 8 state = enabled type = w
    declare @ '/home/danielbevenius/work/google/v8_src/v8/src/heap/read-only-heap.cc:28'
    watchpoint spec = 'isolate->isolate_data_.roots_.roots_[0]'
    new value: 0
(lldb) r

Watchpoint 5 hit:
old value: 0
new value: 16995320070433
Process 1687448 stopped
* thread #1, name = 'hello-world', stop reason = watchpoint 5
    frame #0: 0x00007ffff664e5b1 libv8.so`v8::internal::FullMaybeObjectSlot::store(this=0x00007fffffffc3b0, value=MaybeObject @ 0x00007fffffffc370) const at slots-inl.h:74:1
   71  	
   72  	void FullMaybeObjectSlot::store(MaybeObject value) const {
   73  	  *location() = value.ptr();
-> 74  	}
   75 
```
We can verify that location actually contains the address of `roots_[0]`:
```console
(lldb) expr -f hex -- this->ptr_
(v8::internal::Address) $164 = 0x00000f7500000080
(lldb) expr -f hex -- &this->isolate_->isolate_data_.roots_.roots_[0]
(v8::internal::Address *) $171 = 0x00000f7500000080

(lldb) expr -f hex -- value.ptr()
(unsigned long) $184 = 0x00000f7508040121
(lldb) expr -f hex -- isolate_->isolate_data_.roots_.roots_[0]
(v8::internal::Address) $183 = 0x00000f7508040121
```
The first entry is free_space_map.
```console
(lldb) expr v8::internal::Map::unchecked_cast(v8::internal::Object(value->ptr()))
(v8::internal::Map) $185 = {
  v8::internal::HeapObject = {
    v8::internal::Object = {
      v8::internal::TaggedImpl<v8::internal::HeapObjectReferenceType::STRONG, unsigned long> = (ptr_ = 16995320070433)
    }
  }
```
Next, we will go through the while loop again:
```console
(lldb) expr -f hex -- isolate_->isolate_data_.roots_.roots_[1]
(v8::internal::Address) $191 = 0x0000000000000000
(lldb) expr -f hex -- &isolate_->isolate_data_.roots_.roots_[1]
(v8::internal::Address *) $192 = 0x00000f7500000088
(lldb) expr -f hex -- location()
(v8::internal::SlotBase<v8::internal::FullMaybeObjectSlot, unsigned long, 8>::TData *) $194 = 0x00000f7500000088
```
Notice that in Deserializer::Write we have:
```c++
  dest.store(value);
  return dest + 1;
```
And it's current value is:
```console
(v8::internal::Address) $197 = 0x00000f7500000088
```
Which is the same address as roots_[1] that we just wrote to.

If we know the type that an Address points to we can use the Type::cast(Object obj)
to cast it into a pointer of that type. I think this works will all types.
```console
(lldb) expr -A -f hex  -- v8::internal::Oddball::cast(v8::internal::Object(isolate_->isolate_data_.roots_.roots_[4]))
(v8::internal::Oddball) $258 = {
  v8::internal::TorqueGeneratedOddball<v8::internal::Oddball, v8::internal::PrimitiveHeapObject> = {
    v8::internal::PrimitiveHeapObject = {
      v8::internal::TorqueGeneratedPrimitiveHeapObject<v8::internal::PrimitiveHeapObject, v8::internal::HeapObject> = {
        v8::internal::HeapObject = {
          v8::internal::Object = {
            v8::internal::TaggedImpl<v8::internal::HeapObjectReferenceType::STRONG, unsigned long> = (ptr_ = 0x00000f750804030d)
          }
        }
      }
    }
  }
}
```
You can also just cast it to an object and try printing it:
```console
(lldb) expr -A -f hex  -- v8::internal::Object(isolate_->isolate_data_.roots_.roots_[4]).Print()
#undefined
```
This is actually the Oddball UndefinedValue so it makes sense in this case I think.
With this value in the roots_ array we can use the function ReadOnlyRoots::undefined_value():
```console
(lldb) expr v8::internal::ReadOnlyRoots(&isolate_->heap_).undefined_value()
(v8::internal::Oddball) $265 = {
  v8::internal::TorqueGeneratedOddball<v8::internal::Oddball, v8::internal::PrimitiveHeapObject> = {
    v8::internal::PrimitiveHeapObject = {
      v8::internal::TorqueGeneratedPrimitiveHeapObject<v8::internal::PrimitiveHeapObject, v8::internal::HeapObject> = {
        v8::internal::HeapObject = {
          v8::internal::Object = {
            v8::internal::TaggedImpl<v8::internal::HeapObjectReferenceType::STRONG, unsigned long> = (ptr_ = 16995320070925)
          }
        }
      }
    }
  }
}
```
So how are these roots used, take the above `undefined_value` for example?  
Well most things (perhaps all) that are needed go via the Factory which the
internal Isolate is a type of. In factory we can find:
```c++
Handle<Oddball> Factory::undefined_value() {
  return Handle<Oddball>(&isolate()->roots_table()[RootIndex::kUndefinedValue]);
}
```
Notice that this is basically what we did in the debugger before but here
it is wrapped in Handle so that it can be tracked by the GC.

The unit test [isolate_test](./test/isolate_test.cc) explores the internal 
isolate and has example of usages of the above mentioned methods.

InitwithSnapshot will call Isolate::Init:
```c++
bool Isolate::Init(ReadOnlyDeserializer* read_only_deserializer,
                   StartupDeserializer* startup_deserializer) {

#define ASSIGN_ELEMENT(CamelName, hacker_name)                  \
  isolate_addresses_[IsolateAddressId::k##CamelName##Address] = \
      reinterpret_cast<Address>(hacker_name##_address());
  FOR_EACH_ISOLATE_ADDRESS_NAME(ASSIGN_ELEMENT)
#undef ASSIGN_ELEMENT
```
```c++
  Address isolate_addresses_[kIsolateAddressCount + 1] = {};
```
```console
(gdb) p isolate_addresses_
$16 = {0 <repeats 13 times>}
```

Lets take a look at the expanded code in Isolate::Init:
```console
$ clang++ -I./out/x64.release/gen -I. -I./include -E src/execution/isolate.cc > output
```
```c++
isolate_addresses_[IsolateAddressId::kHandlerAddress] = reinterpret_cast<Address>(handler_address());
isolate_addresses_[IsolateAddressId::kCEntryFPAddress] = reinterpret_cast<Address>(c_entry_fp_address());
isolate_addresses_[IsolateAddressId::kCFunctionAddress] = reinterpret_cast<Address>(c_function_address());
isolate_addresses_[IsolateAddressId::kContextAddress] = reinterpret_cast<Address>(context_address());
isolate_addresses_[IsolateAddressId::kPendingExceptionAddress] = reinterpret_cast<Address>(pending_exception_address());
isolate_addresses_[IsolateAddressId::kPendingHandlerContextAddress] = reinterpret_cast<Address>(pending_handler_context_address());
isolate_addresses_[IsolateAddressId::kPendingHandlerEntrypointAddress] = reinterpret_cast<Address>(pending_handler_entrypoint_address());
isolate_addresses_[IsolateAddressId::kPendingHandlerConstantPoolAddress] = reinterpret_cast<Address>(pending_handler_constant_pool_address());
isolate_addresses_[IsolateAddressId::kPendingHandlerFPAddress] = reinterpret_cast<Address>(pending_handler_fp_address());
isolate_addresses_[IsolateAddressId::kPendingHandlerSPAddress] = reinterpret_cast<Address>(pending_handler_sp_address());
isolate_addresses_[IsolateAddressId::kExternalCaughtExceptionAddress] = reinterpret_cast<Address>(external_caught_exception_address());
isolate_addresses_[IsolateAddressId::kJSEntrySPAddress] = reinterpret_cast<Address>(js_entry_sp_address());
```
Then functions, like handler_address() are implemented as:
```c++ 
inline Address* handler_address() { return &thread_local_top()->handler_; }   
```
```console
(gdb) x/x isolate_addresses_[0]
0x1a3500003240:	0x00000000
```
At this point in the program we have only set the entries to point contain
the addresses specified in ThreadLocalTop, At the time there are initialized
the will mostly be initialized to `kNullAddress`:
```c++
static const Address kNullAddress = 0;
```
And notice that the functions above return pointers so later these pointers can
be updated to point to something. What/when does this happen?  Lets continue and
find out...

Back in Isolate::Init we have:
```c++
  compilation_cache_ = new CompilationCache(this);
  descriptor_lookup_cache_ = new DescriptorLookupCache();
  inner_pointer_to_code_cache_ = new InnerPointerToCodeCache(this);
  global_handles_ = new GlobalHandles(this);
  eternal_handles_ = new EternalHandles();
  bootstrapper_ = new Bootstrapper(this);
  handle_scope_implementer_ = new HandleScopeImplementer(this);
  load_stub_cache_ = new StubCache(this);
  store_stub_cache_ = new StubCache(this);
  materialized_object_store_ = new MaterializedObjectStore(this);
  regexp_stack_ = new RegExpStack();
  regexp_stack_->isolate_ = this;
  date_cache_ = new DateCache();
  heap_profiler_ = new HeapProfiler(heap());
  interpreter_ = new interpreter::Interpreter(this);

  compiler_dispatcher_ =
      new CompilerDispatcher(this, V8::GetCurrentPlatform(), FLAG_stack_size);

  // SetUp the object heap.
  DCHECK(!heap_.HasBeenSetUp());
  heap_.SetUp();

  ...
  InitializeThreadLocal();
```
Lets take a look at `InitializeThreadLocal`

```c++
void Isolate::InitializeThreadLocal() {
  thread_local_top()->Initialize(this);
  clear_pending_exception();
  clear_pending_message();
  clear_scheduled_exception();
}
```
```c++
void Isolate::clear_pending_exception() {
  DCHECK(!thread_local_top()->pending_exception_.IsException(this));
  thread_local_top()->pending_exception_ = ReadOnlyRoots(this).the_hole_value();
}
```
ReadOnlyRoots 
```c++
#define ROOT_ACCESSOR(Type, name, CamelName) \
  V8_INLINE class Type name() const;         \
  V8_INLINE Handle<Type> name##_handle() const;

  READ_ONLY_ROOT_LIST(ROOT_ACCESSOR)
#undef ROOT_ACCESSOR
```
This will expand to a number of function declarations that looks like this:
```console
$ clang++ -I./out/x64.release/gen -I. -I./include -E src/roots/roots.h > output
```
```c++
inline __attribute__((always_inline)) class Map free_space_map() const;
inline __attribute__((always_inline)) Handle<Map> free_space_map_handle() const;
```
The Map class is what all HeapObject use to describe their structure. Notice
that there is also a Handle<Map> declared.
These are generated by a macro in roots-inl.h:
```c++
Map ReadOnlyRoots::free_space_map() const { 
  ((void) 0);
  return Map::unchecked_cast(Object(at(RootIndex::kFreeSpaceMap)));
} 

Handle<Map> ReadOnlyRoots::free_space_map_handle() const {
  ((void) 0);
  return Handle<Map>(&at(RootIndex::kFreeSpaceMap));
}
```
Notice that this is using the RootIndex enum that was mentioned earlier:
```c++
  return Map::unchecked_cast(Object(at(RootIndex::kFreeSpaceMap)));
```
In object/map.h there is the following line:
```c++
  DECL_CAST(Map)
```
Which can be found in objects/object-macros.h:
```c++
#define DECL_CAST(Type)                                 \
  V8_INLINE static Type cast(Object object);            \
  V8_INLINE static Type unchecked_cast(Object object) { \
    return bit_cast<Type>(object);                      \
  }
```
This will expand to something like
```c++
  static Map cast(Object object);
  static Map unchecked_cast(Object object) {
    return bit_cast<Map>(object);
  }
```
And the `Object` part is the Object contructor that takes an Address: 
```c++
  explicit constexpr Object(Address ptr) : TaggedImpl(ptr) {}
```
That leaves the at function which is a private function in ReadOnlyRoots:
```c++
  V8_INLINE Address& at(RootIndex root_index) const;
```

So we are now back in Isolate::Init after the call to InitializeThreadLocal we
have:
```c++
setup_delegate_->SetupBuiltins(this);
```

In the following line in api.cc, where does `i::OBJECT_TEMPLATE_INFO_TYPE` come from:
```c++
  i::Handle<i::Struct> struct_obj = isolate->factory()->NewStruct(
      i::OBJECT_TEMPLATE_INFO_TYPE, i::AllocationType::kOld);
```

### InstanceType
The enum `InstanceType` is defined in `src/objects/instance-type.h`:
```c++
#include "torque-generated/instance-types-tq.h" 

enum InstanceType : uint16_t {
  ...   
#define MAKE_TORQUE_INSTANCE_TYPE(TYPE, value) TYPE = value,                    
  TORQUE_ASSIGNED_INSTANCE_TYPES(MAKE_TORQUE_INSTANCE_TYPE)                     
#undef MAKE_TORQUE_INSTANCE_TYPE 
  ...
};
```
And in `gen/torque-generated/instance-types-tq.h` we can find:
```c++
#define TORQUE_ASSIGNED_INSTANCE_TYPES(V) \                                     
  ...
  V(OBJECT_TEMPLATE_INFO_TYPE, 79) \                                      
  ...
```
There is list in `src/objects/objects-definitions.h`:
```c++
#define STRUCT_LIST_GENERATOR_BASE(V, _)                                      \
  ...
  V(_, OBJECT_TEMPLATE_INFO_TYPE, ObjectTemplateInfo, object_template_info)   \
  ...
```
```c++
template <typename Impl>
Handle<Struct> FactoryBase<Impl>::NewStruct(InstanceType type,
                                            AllocationType allocation) {
  Map map = Map::GetInstanceTypeMap(read_only_roots(), type);
```
If we look in `Map::GetInstanceTypeMap` in map.cc we find:
```c++
  Map map;
  switch (type) {
#define MAKE_CASE(TYPE, Name, name) \
  case TYPE:                        \
    map = roots.name##_map();       \
    break;
    STRUCT_LIST(MAKE_CASE)
#undef MAKE_CASE
```
Now, we know that our type is:
```console
(gdb) p type
$1 = v8::internal::OBJECT_TEMPLATE_INFO_TYPE
```
```c++
    map = roots.object_template_info_map();       \
```
And we can inspect the output of the preprocessor of roots.cc and find:
```c++
Map ReadOnlyRoots::object_template_info_map() const { 
  ((void) 0);
  return Map::unchecked_cast(Object(at(RootIndex::kObjectTemplateInfoMap)));
}
```
And this is something we have seen before. 

One things I ran into was wanting to print the InstanceType using the overloaded
<< operator which is defined for the InstanceType in objects.cc.
```c++
std::ostream& operator<<(std::ostream& os, InstanceType instance_type) {
  switch (instance_type) {
#define WRITE_TYPE(TYPE) \
  case TYPE:             \
    return os << #TYPE;
    INSTANCE_TYPE_LIST(WRITE_TYPE)
#undef WRITE_TYPE
  }
  UNREACHABLE();
}
```
The code I'm using is the followig:
```c++
  i::InstanceType type = map.instance_type();
  std::cout << "object_template_info_map type: " << type << '\n';
```
This will cause the `UNREACHABLE()` function to be called and a Fatal error
thrown. But note that the following line works:
```c++
  std::cout << "object_template_info_map type: " << v8::internal::OBJECT_TEMPLATE_INFO_TYPE << '\n';
```
And prints
```console
object_template_info_map type: OBJECT_TEMPLATE_INFO_TYPE
```
In the switch/case block above the case for this value is:
```c++
  case OBJECT_TEMPLATE_INFO_TYPE:
    return os << "OBJECT_TEMPLATE_INFO_TYPE"
```
When map.instance_type() is called, it returns a value of `1023` but the value
of OBJECT_TEMPLATE_INFO_TYPE is:
```c++
OBJECT_TEMPLATE_INFO_TYPE = 79
```
And we can confirm this using:
```console
  std::cout << "object_template_info_map type: " << static_cast<uint16_t>(v8::internal::OBJECT_TEMPLATE_INFO_TYPE) << '\n';
```
Which will print:
```console
object_template_info_map type: 79
```

### IsolateData



### Context creation
When we create a new context using:
```c++
  Local<ObjectTemplate> global = ObjectTemplate::New(isolate_);
  Local<Context> context = Context::New(isolate_, nullptr, global);
```
The Context class in `include/v8.h` declares New as follows:
```c++
static Local<Context> New(Isolate* isolate,
    ExtensionConfiguration* extensions = nullptr,
    MaybeLocal<ObjectTemplate> global_template = MaybeLocal<ObjectTemplate>(),
    MaybeLocal<Value> global_object = MaybeLocal<Value>(),
    DeserializeInternalFieldsCallback internal_fields_deserializer = DeserializeInternalFieldsCallback(),
    MicrotaskQueue* microtask_queue = nullptr);
```

When a step into Context::New(isolate_, nullptr, global) this will first break
in the constructor of DeserializeInternalFieldsCallback in v8.h which has default
values for the callback function and data_args (both are nullptr). After that
gdb will break in MaybeLocal<Value> and setting val_ to nullptr. Next it will
break in Local::operator* for the value of `global` which is then passed to the
MaybeLocal<v8::ObjectTemplate> constructor. After those break points the break
point will be in api.cc and v8::Context::New. New will call NewContext in api.cc.

There will be some checks and logging/tracing and then a call to CreateEnvironment:
```c++
i::Handle<i::Context> env = CreateEnvironment<i::Context>(                         
    isolate,
    extensions,
    global_template, 
    global_object,                           
    context_snapshot_index, 
    embedder_fields_deserializer, 
    microtask_queue); 
```
The first line in CreateEnironment is:
```c++
ENTER_V8_FOR_NEW_CONTEXT(isolate);
```
Which is a macro defined in api.cc
```c++
i::VMState<v8::OTHER> __state__((isolate)); \                                 
i::DisallowExceptions __no_exceptions__((isolate)) 
```
So the first break point we break on will be the execution/vm-state-inl.h and
VMState's constructor:
```c++
template <StateTag Tag>                                                         
VMState<Tag>::VMState(Isolate* isolate)                                         
    : isolate_(isolate), previous_tag_(isolate->current_vm_state()) {           
  isolate_->set_current_vm_state(Tag);                                          
} 
```
In gdb you'll see this:
```console
(gdb) s
v8::internal::VMState<(v8::StateTag)5>::VMState (isolate=0x372500000000, this=<synthetic pointer>) at ../../src/api/api.cc:6005
6005	      context_snapshot_index, embedder_fields_deserializer, microtask_queue);
(gdb) s
v8::internal::Isolate::current_vm_state (this=0x372500000000) at ../../src/execution/isolate.h:1072
1072	  THREAD_LOCAL_TOP_ACCESSOR(StateTag, current_vm_state)
```
Notice that VMState's constructor sets its `previous_tag_` to isolate->current_vm_state()
which is generated by the macro THREAD_LOCAL_TOP_ACCESSOR.
The next break point will be:
```console
#0  v8::internal::PerIsolateAssertScopeDebugOnly<(v8::internal::PerIsolateAssertType)5, false>::PerIsolateAssertScopeDebugOnly (
    isolate=0x372500000000, this=0x7ffc7b51b500) at ../../src/common/assert-scope.h:107
107	  explicit PerIsolateAssertScopeDebugOnly(Isolate* isolate)
```
We can find that `DisallowExceptions` is defined in src/common/assert-scope.h as:
```c++
using DisallowExceptions =                                                      
    PerIsolateAssertScopeDebugOnly<NO_EXCEPTION_ASSERT, false>;
```
After all that we can start to look at the code in CreateEnvironment.

```c++
    // Create the environment.                                                       
    InvokeBootstrapper<ObjectType> invoke;                                           
    result = invoke.Invoke(isolate, maybe_proxy, proxy_template, extensions,    
                           context_snapshot_index, embedder_fields_deserializer,
                           microtask_queue);  


template <typename ObjectType>                                                  
struct InvokeBootstrapper;                                                        
                                                                                     
template <>                                                                     
struct InvokeBootstrapper<i::Context> {                                         
  i::Handle<i::Context> Invoke(                                                 
      i::Isolate* isolate, i::MaybeHandle<i::JSGlobalProxy> maybe_global_proxy, 
      v8::Local<v8::ObjectTemplate> global_proxy_template,                      
      v8::ExtensionConfiguration* extensions, size_t context_snapshot_index,    
      v8::DeserializeInternalFieldsCallback embedder_fields_deserializer,       
      v8::MicrotaskQueue* microtask_queue) {                                         
    return isolate->bootstrapper()->CreateEnvironment(                               
        maybe_global_proxy, global_proxy_template, extensions,                       
        context_snapshot_index, embedder_fields_deserializer, microtask_queue); 
  }                                                                                  
};
```
Bootstrapper can be found in `src/init/bootstrapper.cc`:
```console
HandleScope scope(isolate_);                                                      
Handle<Context> env;                                                              
  {                                                                                 
    Genesis genesis(isolate_, maybe_global_proxy, global_proxy_template,            
                    context_snapshot_index, embedder_fields_deserializer,           
                    microtask_queue);                                               
    env = genesis.result();                                                         
    if (env.is_null() || !InstallExtensions(env, extensions)) {                     
      return Handle<Context>();                                                     
    }                                                                               
  }                 
```
Notice that the break point will be in the HandleScope constructor. Then a 
new instance of Genesis is created which performs some actions in its constructor.
```c++
global_proxy = isolate->factory()->NewUninitializedJSGlobalProxy(instance_size);
```

This will land in factory.cc:
```c++
Handle<Map> map = NewMap(JS_GLOBAL_PROXY_TYPE, size);
```
`size` will be 16 in this case. `NewMap` is declared in factory.h which has
default values for its parameters:
```c++
  Handle<Map> NewMap(InstanceType type, int instance_size,                      
                     ElementsKind elements_kind = TERMINAL_FAST_ELEMENTS_KIND,  
                     int inobject_properties = 0);
```

In Factory::InitializeMap we have the following check:
```c++
DCHECK_EQ(map.GetInObjectProperties(), inobject_properties);
```
Remember that I called `Context::New` with the following arguments:
```c++
  Local<ObjectTemplate> global = ObjectTemplate::New(isolate_);
  Local<Context> context = Context::New(isolate_, nullptr, global);
```


### VMState



### TaggedImpl
Has a single private member which is declared as:
```c++
StorageType ptr_;
```
An instance can be created using:
```c++
  i::TaggedImpl<i::HeapObjectReferenceType::STRONG, i::Address>  tagged{};
```
Storage type can also be `Tagged_t` which is defined in globals.h:
```c++
 using Tagged_t = uint32_t;
```
It looks like it can be a different value when using pointer compression.

### Object (internal)
This class extends TaggedImpl:
```c++
class Object : public TaggedImpl<HeapObjectReferenceType::STRONG, Address> {       
```
An Object can be created using the default constructor, or by passing in an 
Address which will delegate to TaggedImpl constructors. Object itself does
not have any members (apart from ptr_ which is inherited from TaggedImpl that is). 
So if we create an Object on the stack this is like a pointer/reference to
an object: 
```
+------+
|Object|
|------|
|ptr_  |---->
+------+
```
Now, `ptr_` is a TaggedImpl so it would be a Smi in which case it would just
contains the value directly, for example a small integer:
```
+------+
|Object|
|------|
|  18  |
+------+
```

### Handle
A Handle is similar to a Object and ObjectSlot in that it also contains
an Address member (called location_ and declared in HandleBase), but with the
difference is that Handles can be relocated by the garbage collector.

### HeapObject


### NewContext
When we create a new context using:
```c++
const v8::Local<v8::ObjectTemplate> obt = v8::Local<v8::ObjectTemplate>();
v8::Handle<v8::Context> context = v8::Context::New(isolate_, nullptr, obt);
```
The above is using the static function New declared in `include/v8.h`
```c++
static Local<Context> New(                                                    
    Isolate* isolate,
    ExtensionConfiguration* extensions = nullptr,           
    MaybeLocal<ObjectTemplate> global_template = MaybeLocal<ObjectTemplate>(),
    MaybeLocal<Value> global_object = MaybeLocal<Value>(),                    
    DeserializeInternalFieldsCallback internal_fields_deserializer = DeserializeInternalFieldsCallback(),                                  
    MicrotaskQueue* microtask_queue = nullptr);
```
The implementation for this function can be found in `src/api/api.cc`
How does a Local become a MaybeLocal in this above case?  
This is because MaybeLocal has a constructor that takes a `Local<S>` and this will
be casted into the `val_` member of the MaybeLocal instance.


### Genesis
TODO


### What is the difference between a Local and a Handle?

Currently, the torque generator will generate Print functions that look like
the following:
```c++
template <>                                                                     
void TorqueGeneratedEnumCache<EnumCache, Struct>::EnumCachePrint(std::ostream& os) {
  this->PrintHeader(os, "TorqueGeneratedEnumCache");
  os << "\n - keys: " << Brief(this->keys());
  os << "\n - indices: " << Brief(this->indices());
  os << "\n";
}
```
Notice the last line where the newline character is printed as a string. This
would just be a char instead `'\n'`.

There are a number of things that need to happen only once upon startup for
each process. These things are placed in `V8::InitializeOncePerProcessImpl` which
can be found in `src/init/v8.cc`. This is called by v8::V8::Initialize().
```c++
  CpuFeatures::Probe(false);                                                    
  ElementsAccessor::InitializeOncePerProcess();                                 
  Bootstrapper::InitializeOncePerProcess();                                     
  CallDescriptors::InitializeOncePerProcess();                                  
  wasm::WasmEngine::InitializeOncePerProcess();
```
ElementsAccessor populates the accessor_array with Elements listed in 
`ELEMENTS_LIST`. TODO: take a closer look at Elements. 

v8::Isolate::Initialize will set up the heap.
```c++
i_isolate->heap()->ConfigureHeap(params.constraints);
```

It is when we create an new Context that Genesis is created. This will call
Snapshot::NewContextFromSnapshot.
So the context is read from the StartupData* blob with ExtractContextData(blob).

What is the global proxy?

### Builtins runtime error
Builtins is a member of Isolate and an instance is created by the Isolate constructor.
We can inspect the value of `initialized_` and that it is false:
```console
(gdb) p *this->builtins()
$3 = {static kNoBuiltinId = -1, static kFirstWideBytecodeHandler = 1248, static kFirstExtraWideBytecodeHandler = 1398, 
  static kLastBytecodeHandlerPlusOne = 1548, static kAllBuiltinsAreIsolateIndependent = true, isolate_ = 0x0, initialized_ = false, 
  js_entry_handler_offset_ = 0}
```
The above is printed form Isolate's constructor and it is not changes in the
contructor.

This is very strange, while I though that the `initialized_` was being updated
it now looks like there might be two instances, one with has this value as false
and the other as true. And also one has a nullptr as the isolate and the other
as an actual value.
For example, when I run the hello-world example:
```console
$4 = (v8::internal::Builtins *) 0x33b20000a248
(gdb) p &builtins_
$5 = (v8::internal::Builtins *) 0x33b20000a248
```
Notice that these are poiting to the same location in memory.
```console
(gdb) p &builtins_
$1 = (v8::internal::Builtins *) 0x25210000a248
(gdb) p builtins()
$2 = (v8::internal::Builtins *) 0x25210000a228
```
Alright, so after looking into this closer I noticed that I was including
internal headers in the test itself.
When I include `src/builtins/builtins.h` I will get an implementation of
isolate->builtins() in the object file which is in the shared library libv8.so,
but the field is part of object file that is part of the cctest. This will be a
different method and not the method that is in libv8_v8.so shared library.

As I'm only interested in exploring v8 internals and my goal is only for each
unit test to verify my understanding I've statically linked those object files
needed, like builtins.o and code.o to the test.

```console
 Fatal error in ../../src/snapshot/read-only-deserializer.cc, line 35
# Debug check failed: !isolate->builtins()->is_initialized().
#
#
#
#FailureMessage Object: 0x7ffed92ceb20
==== C stack trace ===============================

    /home/danielbevenius/work/google/v8_src/v8/out/x64.release_gcc/libv8_libbase.so(v8::base::debug::StackTrace::StackTrace()+0x1d) [0x7fabe6c348c1]
    /home/danielbevenius/work/google/v8_src/v8/out/x64.release_gcc/libv8_libplatform.so(+0x652d9) [0x7fabe6cac2d9]
    /home/danielbevenius/work/google/v8_src/v8/out/x64.release_gcc/libv8_libbase.so(V8_Fatal(char const*, int, char const*, ...)+0x172) [0x7fabe6c2416d]
    /home/danielbevenius/work/google/v8_src/v8/out/x64.release_gcc/libv8_libbase.so(v8::base::SetPrintStackTrace(void (*)())+0) [0x7fabe6c23de0]
    /home/danielbevenius/work/google/v8_src/v8/out/x64.release_gcc/libv8_libbase.so(V8_Dcheck(char const*, int, char const*)+0x2d) [0x7fabe6c241b1]
    /home/danielbevenius/work/google/v8_src/v8/out/x64.release_gcc/libv8.so(v8::internal::ReadOnlyDeserializer::DeserializeInto(v8::internal::Isolate*)+0x192) [0x7fabe977c468]
    /home/danielbevenius/work/google/v8_src/v8/out/x64.release_gcc/libv8.so(v8::internal::ReadOnlyHeap::DeseralizeIntoIsolate(v8::internal::Isolate*, v8::internal::ReadOnlyDeserializer*)+0x4f) [0x7fabe91e5a7d]
    /home/danielbevenius/work/google/v8_src/v8/out/x64.release_gcc/libv8.so(v8::internal::ReadOnlyHeap::SetUp(v8::internal::Isolate*, v8::internal::ReadOnlyDeserializer*)+0x66) [0x7fabe91e5a2a]
    /home/danielbevenius/work/google/v8_src/v8/out/x64.release_gcc/libv8.so(v8::internal::Isolate::Init(v8::internal::ReadOnlyDeserializer*, v8::internal::StartupDeserializer*)+0x70b) [0x7fabe90633bb]
    /home/danielbevenius/work/google/v8_src/v8/out/x64.release_gcc/libv8.so(v8::internal::Isolate::InitWithSnapshot(v8::internal::ReadOnlyDeserializer*, v8::internal::StartupDeserializer*)+0x7b) [0x7fabe906299f]
    /home/danielbevenius/work/google/v8_src/v8/out/x64.release_gcc/libv8.so(v8::internal::Snapshot::Initialize(v8::internal::Isolate*)+0x1e9) [0x7fabe978d941]
    /home/danielbevenius/work/google/v8_src/v8/out/x64.release_gcc/libv8.so(v8::Isolate::Initialize(v8::Isolate*, v8::Isolate::CreateParams const&)+0x33d) [0x7fabe8d999e3]
    /home/danielbevenius/work/google/v8_src/v8/out/x64.release_gcc/libv8.so(v8::Isolate::New(v8::Isolate::CreateParams const&)+0x28) [0x7fabe8d99b66]
    ./test/builtins_test() [0x4135a2]
    ./test/builtins_test() [0x43a1b7]
    ./test/builtins_test() [0x434c99]
    ./test/builtins_test() [0x41a3a7]
    ./test/builtins_test() [0x41aafb]
    ./test/builtins_test() [0x41b085]
    ./test/builtins_test() [0x4238e0]
    ./test/builtins_test() [0x43b1aa]
    ./test/builtins_test() [0x435773]
    ./test/builtins_test() [0x422836]
    ./test/builtins_test() [0x412ea4]
    ./test/builtins_test() [0x412e3d]
    /lib64/libc.so.6(__libc_start_main+0xf3) [0x7fabe66b31a3]
    ./test/builtins_test() [0x412d5e]
Illegal instruction (core dumped)
```
The issue here is that I'm including the header in the test, which means that
code will be in the object code of the test, while the implementation part will
be in the linked dynamic library which is why these are pointing to different
areas in memory. The one retreived by the function call will use the

### Goma
I've goma referenced in a number of places so just makeing a note of what it is
here: Goma is googles internal distributed compile service.

### WebAssembly
This section is going to take a closer look at how wasm works in V8.

We can use a wasm module like this:
```js
  const buffer = fixtures.readSync('add.wasm'); 
  const module = new WebAssembly.Module(buffer);                             
  const instance = new WebAssembly.Instance(module);                        
  instance.exports.add(3, 4);
```
Where is the WebAssembly object setup?  We have sen previously that objects and
function are added in `src/init/bootstrapper.cc` and for Wasm there is a function
named Genisis::InstallSpecialObjects which calls:
```c++
  WasmJs::Install(isolate, true);
```
This call will land in `src/wasm/wasm-js.cc` where we can find:
```c++
void WasmJs::Install(Isolate* isolate, bool exposed_on_global_object) {
  ...
  Handle<String> name = v8_str(isolate, "WebAssembly")
  ...
  NewFunctionArgs args = NewFunctionArgs::ForFunctionWithoutCode(               
      name, isolate->strict_function_map(), LanguageMode::kStrict);             
  Handle<JSFunction> cons = factory->NewFunction(args);                         
  JSFunction::SetPrototype(cons, isolate->initial_object_prototype());          
  Handle<JSObject> webassembly =                                                
      factory->NewJSObject(cons, AllocationType::kOld); 
  JSObject::AddProperty(isolate, webassembly, factory->to_string_tag_symbol(),  
                        name, ro_attributes);                                   

  InstallFunc(isolate, webassembly, "compile", WebAssemblyCompile, 1);          
  InstallFunc(isolate, webassembly, "validate", WebAssemblyValidate, 1);            
  InstallFunc(isolate, webassembly, "instantiate", WebAssemblyInstantiate, 1);
  ...
  Handle<JSFunction> module_constructor =                                       
      InstallConstructorFunc(isolate, webassembly, "Module", WebAssemblyModule);
  ...
}
```
And all the rest of the functions that are available on the `WebAssembly` object
are setup in the same function.
```console
(lldb) br s -name Genesis::InstallSpecialObjects
```
Now, lets also set a break point in WebAssemblyModule:
```console
(lldb) br s -n WebAssemblyModule
(lldb) r
```
```c++
  v8::Isolate* isolate = args.GetIsolate();                                         
  i::Isolate* i_isolate = reinterpret_cast<i::Isolate*>(isolate);                   
  if (i_isolate->wasm_module_callback()(args)) return;                              
```
Notice the `wasm_module_callback()` function which is a function that is setup
on the internal Isolate in `src/execution/isolate.h`:
```c++
#define ISOLATE_INIT_LIST(V)                                                   \
  ...
  V(ExtensionCallback, wasm_module_callback, &NoExtension)                     \
  V(ExtensionCallback, wasm_instance_callback, &NoExtension)                   \
  V(WasmStreamingCallback, wasm_streaming_callback, nullptr)                   \
  V(WasmThreadsEnabledCallback, wasm_threads_enabled_callback, nullptr)        \
  V(WasmLoadSourceMapCallback, wasm_load_source_map_callback, nullptr) 

#define GLOBAL_ACCESSOR(type, name, initialvalue)                \              
  inline type name() const {                                     \              
    DCHECK(OFFSET_OF(Isolate, name##_) == name##_debug_offset_); \              
    return name##_;                                              \              
  }                                                              \              
  inline void set_##name(type value) {                           \              
    DCHECK(OFFSET_OF(Isolate, name##_) == name##_debug_offset_); \              
    name##_ = value;                                             \              
  }                                                                             
  ISOLATE_INIT_LIST(GLOBAL_ACCESSOR)                                            
#undef GLOBAL_ACCESSOR
```
So this would be expanded by the preprocessor into:
```c++
inline ExtensionCallback wasm_module_callback() const {
  ((void) 0);
  return wasm_module_callback_;
}
inline void set_wasm_module_callback(ExtensionCallback value) {
  ((void) 0);
  wasm_module_callback_ = value;
}
```
Also notice that if `wasm_module_callback()` return true the `WebAssemblyModule`
fuction will return and no further processing of the instructions in that function
will be done. `NoExtension` is a function that looks like this:
```c++
bool NoExtension(const v8::FunctionCallbackInfo<v8::Value>&) { return false; }
```
And is set as the default function for module/instance callbacks.

Looking a little further we can see checks for WASM Threads support (TODO: take
a look at this).
And then we have:
```c++
  module_obj = i_isolate->wasm_engine()->SyncCompile(                             
        i_isolate, enabled_features, &thrower, bytes);
```
`SyncCompile` can be found in `src/wasm/wasm-engine.cc` and will call
`DecodeWasmModule` which can be found in `src/wasm/module-decoder.cc`.
```c++
ModuleResult result = DecodeWasmModule(enabled, bytes.start(), bytes.end(),
                                       false, kWasmOrigin, 
                                       isolate->counters(), allocator()); 
```
```c++
ModuleResult DecodeWasmModule(const WasmFeatures& enabled,                      
                              const byte* module_start, const byte* module_end, 
                              bool verify_functions, ModuleOrigin origin,       
                              Counters* counters,                               
                              AccountingAllocator* allocator) {
  ...
  ModuleDecoderImpl decoder(enabled, module_start, module_end, origin);
  return decoder.DecodeModule(counters, allocator, verify_functions);
```
DecodeModuleHeader:
```c++
  uint32_t magic_word = consume_u32("wasm magic");
```
This will land in `src/wasm/decoder.h` consume_little_endian(name):
```c++

```
A wasm module has the following preamble:
```
magic nr: 0x6d736100 
version: 0x1
```
These can be found as a constant in `src/wasm/wasm-constants.h`:
```c++
constexpr uint32_t kWasmMagic = 0x6d736100; 
constexpr uint32_t kWasmVersion = 0x01;
```
After the DecodeModuleHeader the code will iterate of the sections (type,
import, function, table, memory, global, export, start, element, code, data,
custom).
For each section `DecodeSection` will be called:
```c++
DecodeSection(section_iter.section_code(), section_iter.payload(),
              offset, verify_functions);
```
There is an enum named `SectionCode` in `src/wasm/wasm-constants.h` which
contains the various sections which is used in switch statement in DecodeSection
. Depending on the `section_code` there are Decode<Type>Section methods that
will be called. In our case section_code is:
```console
(lldb) expr section_code
(v8::internal::wasm::SectionCode) $5 = kTypeSectionCode
```
And this will match the `kTypeSectionCode` and `DecodeTypeSection` will be
called.

ValueType can be found in `src/wasm/value-type.h` and there are types for
each of the currently supported types:
```c++
constexpr ValueType kWasmI32 = ValueType(ValueType::kI32);                      
constexpr ValueType kWasmI64 = ValueType(ValueType::kI64);                      
constexpr ValueType kWasmF32 = ValueType(ValueType::kF32);                      
constexpr ValueType kWasmF64 = ValueType(ValueType::kF64);                      
constexpr ValueType kWasmAnyRef = ValueType(ValueType::kAnyRef);                
constexpr ValueType kWasmExnRef = ValueType(ValueType::kExnRef);                
constexpr ValueType kWasmFuncRef = ValueType(ValueType::kFuncRef);              
constexpr ValueType kWasmNullRef = ValueType(ValueType::kNullRef);              
constexpr ValueType kWasmS128 = ValueType(ValueType::kS128);                    
constexpr ValueType kWasmStmt = ValueType(ValueType::kStmt);                    
constexpr ValueType kWasmBottom = ValueType(ValueType::kBottom);
```

`FunctionSig` is declared with a `using` statement in value-type.h:
```c++
using FunctionSig = Signature<ValueType>;
```
We can find `Signature` in src/codegen/signature.h:
```c++
template <typename T>
class Signature : public ZoneObject {
 public:
  constexpr Signature(size_t return_count, size_t parameter_count,
                      const T* reps)
      : return_count_(return_count),
        parameter_count_(parameter_count),
        reps_(reps) {}
```
The return count can be zero, one (or greater if multi-value return types are
enabled). The parameter count also makes sense, but reps is not clear to me what
that represents.
```console
(lldb) fr v
(v8::internal::Signature<v8::internal::wasm::ValueType> *) this = 0x0000555555583950
(size_t) return_count = 1
(size_t) parameter_count = 2
(const v8::internal::wasm::ValueType *) reps = 0x0000555555583948
```
Before the call to `Signature`s construtor we have:
```c++
    // FunctionSig stores the return types first.                               
    ValueType* buffer = zone->NewArray<ValueType>(param_count + return_count);  
    uint32_t b = 0;                                                             
    for (uint32_t i = 0; i < return_count; ++i) buffer[b++] = returns[i];           
    for (uint32_t i = 0; i < param_count; ++i) buffer[b++] = params[i];         
                                                                                
    return new (zone) FunctionSig(return_count, param_count, buffer);
```
So `reps_` contains the return (re?) and the params (ps?).


After the DecodeWasmModule has returned in SyncCompile we will have a
ModuleResult. This will be compiled to NativeModule:
```c++
ModuleResult result =                                                         
      DecodeWasmModule(enabled, bytes.start(), bytes.end(), false, kWasmOrigin, 
                       isolate->counters(), allocator());
Handle<FixedArray> export_wrappers;                                           
  std::shared_ptr<NativeModule> native_module =                                 
      CompileToNativeModule(isolate, enabled, thrower,                          
                            std::move(result).value(), bytes, &export_wrappers);
```
`CompileToNativeModule` can be found in `module-compiler.cc`

TODO: CompileNativeModule...

There is an example in [wasm_test.cc](./test/wasm_test.cc).

### ExtensionCallback
Is a typedef defined in `include/v8.h`:
```c++
typedef bool (*ExtensionCallback)(const FunctionCallbackInfo<Value>&); 
```




### JSEntry
TODO: This section should describe the functions calls below.
```console
 * frame #0: 0x00007ffff79a52e4 libv8.so`v8::(anonymous namespace)::WebAssemblyModule(v8::FunctionCallbackInfo<v8::Value> const&) [inlined] v8::FunctionCallbackInfo<v8::Value>::GetIsolate(this=0x00007fffffffc9a0) const at v8.h:11204:40
    frame #1: 0x00007ffff79a52e4 libv8.so`v8::(anonymous namespace)::WebAssemblyModule(args=0x00007fffffffc9a0) at wasm-js.cc:638
    frame #2: 0x00007ffff6fe9e92 libv8.so`v8::internal::FunctionCallbackArguments::Call(this=0x00007fffffffca40, handler=CallHandlerInfo @ 0x00007fffffffc998) at api-arguments-inl.h:158:3
    frame #3: 0x00007ffff6fe7c42 libv8.so`v8::internal::MaybeHandle<v8::internal::Object> v8::internal::(anonymous namespace)::HandleApiCallHelper<true>(isolate=<unavailable>, function=Handle<v8::internal::HeapObject> @ 0x00007fffffffca20, new_target=<unavailable>, fun_data=<unavailable>, receiver=<unavailable>, args=BuiltinArguments @ 0x00007fffffffcae0) at builtins-api.cc:111:36
    frame #4: 0x00007ffff6fe67d4 libv8.so`v8::internal::Builtin_Impl_HandleApiCall(args=BuiltinArguments @ 0x00007fffffffcb20, isolate=0x00000f8700000000) at builtins-api.cc:137:5
    frame #5: 0x00007ffff6fe6319 libv8.so`v8::internal::Builtin_HandleApiCall(args_length=6, args_object=0x00007fffffffcc10, isolate=0x00000f8700000000) at builtins-api.cc:129:1
    frame #6: 0x00007ffff6b2c23f libv8.so`Builtins_CEntry_Return1_DontSaveFPRegs_ArgvOnStack_BuiltinExit + 63
    frame #7: 0x00007ffff68fde25 libv8.so`Builtins_JSBuiltinsConstructStub + 101
    frame #8: 0x00007ffff6daf46d libv8.so`Builtins_ConstructHandler + 1485
    frame #9: 0x00007ffff690e1d5 libv8.so`Builtins_InterpreterEntryTrampoline + 213
    frame #10: 0x00007ffff6904b5a libv8.so`Builtins_JSEntryTrampoline + 90
    frame #11: 0x00007ffff6904938 libv8.so`Builtins_JSEntry + 120
    frame #12: 0x00007ffff716ba0c libv8.so`v8::internal::(anonymous namespace)::Invoke(v8::internal::Isolate*, v8::internal::(anonymous namespace)::InvokeParams const&) [inlined] v8::internal::GeneratedCode<unsigned long, unsigned long, unsigned long, unsigned long, unsigned long, long, unsigned long**>::Call(this=<unavailable>, args=17072495001600, args=<unavailable>, args=17072631376141, args=17072630006049, args=<unavailable>, args=<unavailable>) at simulator.h:142:12
    frame #13: 0x00007ffff716ba01 libv8.so`v8::internal::(anonymous namespace)::Invoke(isolate=<unavailable>, params=0x00007fffffffcf50)::InvokeParams const&) at execution.cc:367
    frame #14: 0x00007ffff716aa10 libv8.so`v8::internal::Execution::Call(isolate=0x00000f8700000000, callable=<unavailable>, receiver=<unavailable>, argc=<unavailable>, argv=<unavailable>) at execution.cc:461:10

```


### CustomArguments
Subclasses of CustomArguments, like PropertyCallbackArguments and 
FunctionCallabackArguments are used for setting up and accessing values
on the stack, and also the subclasses provide methods to call various things
like `CallNamedSetter` for PropertyCallbackArguments and `Call` for
FunctionCallbackArguments.

#### FunctionCallbackArguments
```c++
class FunctionCallbackArguments                                                 
    : public CustomArguments<FunctionCallbackInfo<Value> > {
  FunctionCallbackArguments(internal::Isolate* isolate, internal::Object data,  
                            internal::HeapObject callee,                        
                            internal::Object holder,                            
                            internal::HeapObject new_target,                    
                            internal::Address* argv, int argc);
```
This class is in the namespace v8::internal so I'm curious why the explicit
namespace is used here?

#### BuiltinArguments
This class extends `JavaScriptArguments`
```c++
class BuiltinArguments : public JavaScriptArguments {
 public:
  BuiltinArguments(int length, Address* arguments)
      : Arguments(length, arguments) {

  static constexpr int kNewTargetOffset = 0;
  static constexpr int kTargetOffset = 1;
  static constexpr int kArgcOffset = 2;
  static constexpr int kPaddingOffset = 3;
                                                                                
  static constexpr int kNumExtraArgs = 4;
  static constexpr int kNumExtraArgsWithReceiver = 5;
```
`JavaScriptArguments is declared in `src/common/global.h`:
```c++
using JavaScriptArguments = Arguments<ArgumentsType::kJS>;
```
`Arguments` can be found in `src/execution/arguments.h`and is templated with 
the a type of `ArgumentsType` (in `src/common/globals.h`):
```c++
enum class ArgumentsType {                                                          
  kRuntime,                                                                         
  kJS,                                                                              
}; 
```
An instance of Arguments only has a length which is the number of arguments,
and an Address pointer which points to the first argument. The functions it
provides allows for getting/setting specific arguments and handling various
types (like `Handle<S>`, smi, etc). It also overloads the operator[] allowing
to specify an index and getting back an Object to that argument.
In `BuiltinArguments` the constants specify the index's and provides functions
to get them:
```c++
  inline Handle<Object> receiver() const;                                       
  inline Handle<JSFunction> target() const;                                     
  inline Handle<HeapObject> new_target() const;
```

### Promises
A Promise is an object which has a state. For example, we can create a promise
using the following call:

```c++
  i::Handle<i::JSPromise> promise = factory->NewJSPromise();
  print_handle(promise);
```
This will output:
```console
0x3e46080c1cb9: [JSPromise]
 - map: 0x3e46081c0fa9 <Map(HOLEY_ELEMENTS)> [FastProperties]
 - prototype: 0x3e46082063ed <Object map = 0x3e46081c0fd1>
 - elements: 0x3e46080406e9 <FixedArray[0]> [HOLEY_ELEMENTS]
 - status: pending
 - reactions: 0
 - has_handler: 0
 - handled_hint: 0
 - properties: 0x3e46080406e9 <FixedArray[0]> {}
```
The implementation for `JSPromise` can be found in `src/objects/js-promise.h`.

Now, `factory->NewJSPromise` looks like this:
```c++
Handle<JSPromise> Factory::NewJSPromise() {
  Handle<JSPromise> promise = NewJSPromiseWithoutHook();
  isolate()->RunPromiseHook(PromiseHookType::kInit, promise, undefined_value());
  return promise;
}
Handle<JSPromise> Factory::NewJSPromiseWithoutHook() {
  Handle<JSPromise> promise =
      Handle<JSPromise>::cast(NewJSObject(isolate()->promise_function()));
  promise->set_reactions_or_result(Smi::zero());
  promise->set_flags(0);
  ZeroEmbedderFields(promise);
  DCHECK_EQ(promise->GetEmbedderFieldCount(), v8::Promise::kEmbedderFieldCount);
  return promise;
}
```
Notice `isolate->promise_function()` what does it return and where is it
defined:
```console
$ gdb ./test/promise_test
(gdb) b Factory::NewJSPromise
(gdb) r
(gdb) p isolate()->promise_function()
$1 = {<v8::internal::HandleBase> = {location_ = 0x20a7c68}, <No data fields>}
```
That did tell me much. But if we look at `src/execution/isolate-inl.h` we can
find:
```c++
#define NATIVE_CONTEXT_FIELD_ACCESSOR(index, type, name)    \
  Handle<type> Isolate::name() {                            \
    return Handle<type>(raw_native_context().name(), this); \
  }                                                         \
  bool Isolate::is_##name(type value) {                     \
    return raw_native_context().is_##name(value);           \
  }
NATIVE_CONTEXT_FIELDS(NATIVE_CONTEXT_FIELD_ACCESSOR)
#undef NATIVE_CONTEXT_FIELD_ACCESSOR
```
And `NATIVE_CONTEXT_FIELDS` can be found in "src/objects/contexts.h" which is
included by isolate.h (which isolate-inl.h includes). In contexts.h we find:
```c++
#define NATIVE_CONTEXT_FIELDS(V)
  ...
  V(PROMISE_FUNCTION_INDEX, JSFunction, promise_function)
```
So the preprocessor will generate the following functions for promise_function:
```c++
inline Handle<JSFunction> promise_function();
inline bool is_promise_function(JSFunction value);

Handle<JSFunction> Isolate::promise_function() {
  return Handle<JSFunction>(raw_native_context().promise_function(), this);
}
bool Isolate::is_promise_function(JSFunction value) {
  return raw_native_context().is_promise_function(value);
}
```
And the functions that are called from the above (in src/objects/contexts.h):
```c++
inline void set_promise_function(JSFunction value);
inline bool is_promise_function(JSFunction value) const;
inline JSFunction promise_function() const;

void Context::set_promise_function(JSFunction value) {
  ((void) 0);
  set(PROMISE_FUNCTION_INDEX, value);
}
bool Context::is_promise_function(JSFunction value) const {
  ((void) 0);
  return JSFunction::cast(get(PROMISE_FUNCTION_INDEX)) == value;
}
JSFunction Context::promise_function() const {
  ((void) 0);
  return JSFunction::cast(get(PROMISE_FUNCTION_INDEX));
}
```
So that answers where the function is declared and defined and what it returns.


We can find the torque source file in `src/builtins/promise-constructor.tq` which
has comments that refer to the emcascript spec. In our case this is
[promise-executor](https://tc39.es/ecma262/#sec-promise-executor)
```c++
  transitioning javascript builtin                                                 
  PromiseConstructor(                                                              
      js-implicit context: NativeContext, receiver: JSAny,                         
      newTarget: JSAny)(executor: JSAny): JSAny {
   // 1. If NewTarget is undefined, throw a TypeError exception.                  
   if (newTarget == Undefined) {
     ThrowTypeError(MessageTemplate::kNotAPromise, newTarget);
   }
```
And for the generated c++ code we can look in `out/x64.release_gcc/gen/torque-generated/src/builtins/promise-constructor-tq-csa.cc'.

Now, if we look at the spec and the torque source we can find the first step 
in the spec is:
```
1. If NewTarget is undefined, throw a TypeError exception.
```
And in the torque source file:
```
  if (newTarget == Undefined) {
      ThrowTypeError(MessageTemplate::kNotAPromise, newTarget);
  }
```
And in the generated CodeStubAssembler c++ source for this:
```c++
TF_BUILTIN(PromiseConstructor, CodeStubAssembler) {                             
  compiler::CodeAssemblerState* state_ = state();
  compiler::CodeAssembler ca_(state());
  TNode<Object> parameter2 = UncheckedCast<Object>(Parameter(Descriptor::kJSNewTarget));
  ...

  TNode<Oddball> tmp0;
  TNode<BoolT> tmp1;
  if (block0.is_used()) {
    ca_.Bind(&block0);
    ca_.SetSourcePosition("../../src/builtins/promise-constructor.tq", 51);
    tmp0 = Undefined_0(state_);
    tmp1 = CodeStubAssembler(state_).TaggedEqual(TNode<Object>{parameter2}, TNode<HeapObject>{tmp0});
    ca_.Branch(tmp1, &block1, std::vector<Node*>{}, &block2, std::vector<Node*>{});
  }             

  if (block1.is_used()) {
    ca_.Bind(&block1);
    ca_.SetSourcePosition("../../src/builtins/promise-constructor.tq", 52);
    CodeStubAssembler(state_).ThrowTypeError(TNode<Context>{parameter0}, MessageTemplate::kNotAPromise, TNode<Object>{parameter2});
  }
```
Now, `TF_BUILTIN` is a macro on `src/builtins/builtins-utils-gen.h`. 
```c++
#define TF_BUILTIN(Name, AssemblerBase)                                 \
  class Name##Assembler : public AssemblerBase {                        \
   public:                                                              \
    using Descriptor = Builtin_##Name##_InterfaceDescriptor;            \
                                                                        \
    explicit Name##Assembler(compiler::CodeAssemblerState* state)       \
        : AssemblerBase(state) {}                                       \
    void Generate##Name##Impl();                                        \
                                                                        \
    Node* Parameter(Descriptor::ParameterIndices index) {               \
      return CodeAssembler::Parameter(static_cast<int>(index));         \
    }                                                                   \
  };                                                                    \
  void Builtins::Generate_##Name(compiler::CodeAssemblerState* state) { \
    Name##Assembler assembler(state);                                   \
    state->SetInitialDebugInformation(#Name, __FILE__, __LINE__);       \
    if (Builtins::KindOf(Builtins::k##Name) == Builtins::TFJ) {         \
      assembler.PerformStackCheck(assembler.GetJSContextParameter());   \
    }                                                                   \
    assembler.Generate##Name##Impl();                                   \
  }                                                                     \
  void Name##Assembler::Generate##Name##Impl()
```

So the above will be expanded by the preprocessor into:
```c++
TF_BUILTIN(PromiseConstructor, CodeStubAssembler) {                             
class PromiseConstructorAssembler : public CodeStubAssembler {
 public:
  using Descriptor = Builtin_PromiseConstructor_InterfaceDescriptor;

  explicit PromiseConstructorAssembler(compiler::CodeAssemblerState* state) :
      CodeStubAssembler(state) {}
  void GeneratePromiseConstructorImpl;

  Node* Parameter(Descriptor::ParameterIndices index) {
    return CodeAssembler::Parameter(static_cast<int>(index));
  }
};

void Builtins::Generate_PromiseConstructor(compiler::CodeAssemblerState* state) {
  PromiseConstructorAssembler assembler(state);
  state->SetInitialDebugInformation(PromiseConstructor, __FILE__, __LINE__);
  if (Builtins::KindOf(Builtins::kPromiseConstructor) == Builtins::TFJ) {
    assembler.PerformStackCheck(assembler.GetJSContextParameter());
  }
  assembler.GeneratePromiseConstructorImpl();
}

void PromiseConstructorAssembler::GeneratePromiseConstructorImpl()
  compiler::CodeAssemblerState* state_ = state();
  compiler::CodeAssembler ca_(state());
  TNode<Object> parameter2 = UncheckedCast<Object>(Parameter(Descriptor::kJSNewTarget));
  ... rest of the content that we already showed above.
}
```
And if we are want to inspect the generated assembler code we can find it
in
```console
$ objdump -d ../v8_src/v8/out/x64.release_gcc/obj/torque_generated_initializers/promise-constructor-tq-csa.o | c++filt
```

And this builtin is hooked up by `src/init/boostraper.cc` in:
```c++
void Genesis::InitializeGlobal(Handle<JSGlobalObject> global_object,
                               Handle<JSFunction> empty_function) {
  ...
  {  // -- P r o m i s e                                                        
    Handle<JSFunction> promise_fun = InstallFunction(
        isolate_, global, "Promise", JS_PROMISE_TYPE,
        JSPromise::kSizeWithEmbedderFields, 0, factory->the_hole_value(),
        Builtins::kPromiseConstructor);
    InstallWithIntrinsicDefaultProto(isolate_, promise_fun, Context::PROMISE_FUNCTION_INDEX); 
```
Install function takes the following parameters:
```c++
V8_NOINLINE Handle<JSFunction> InstallFunction(
    Isolate* isolate, Handle<JSObject> target, const char* name,
    InstanceType type, int instance_size, int inobject_properties,
    Handle<HeapObject> prototype, Builtins::Name call) {
  return InstallFunction(isolate, target,
                         isolate->factory()->InternalizeUtf8String(name), type, 
                         instance_size, inobject_properties, prototype, call);
}
```
The above function will be called when creating the snapshot (if snapshots
are configured) so one way to explore this is to debug `mksnapshot`:
```console
$ cd out/x64_release_gcc/
$ gdb mksnapshot
$ Breakpoint 1 at 0x1da5cc7: file ../../src/init/bootstrapper.cc, line 1409.
(gdb) br bootstrapper.cc:2347
(gdb) r
(gdb) continue
(gdb) p JS_PROMISE_TYPE
$1 = v8::internal::JS_PROMISE_TYPE
```
Details about [InstanceType](#instancetype).

And we can see that we passing in `Builtins::kPromiseConstructor`. This is declared 
in `out/x64.release_gcc/gen/torque-generated/builtin-definitions-tq.h`:
```c++
#define BUILTIN_LIST_FROM_TORQUE(CPP, TFJ, TFC, TFS, TFH, ASM) \
...
TFJ(PromiseConstructor, 1, kReceiver, kExecutor) \ 
```
For full details see the [Torque](#torque) section. We will get the following
in `builtins.h` (after being preprocessed):
```c++
class Builtins {
  enum Name : int32_t {
    ...
    kPromiseConstructor,
  };

  static void Generate_PromiseConstructor(compiler::CodeAssemblerState* state); 

};
``` 
And and in builtins.cc (also after being preprocessed):
```c++
struct Builtin_PromiseConstructor_InterfaceDescriptor {
  enum ParameterIndices {
    kJSTarget = compiler::CodeAssembler::kTargetParameterIndex,
    kReceiver,
    kExecutor,
    kJSNewTarget,
    kJSActualArgumentsCount,
    kContext,
    kParameterCount,
  };
};
const BuiltinMetadata builtin_metadata[] = {
  ...
  {"PromiseConstructor", Builtins::TFJ, {1, 0}},
  ...
};
```
`Generate_PromiseConstructor` is declared as as a static function in Builtins
and recall that we showed above that is defined in
`out/x64.release_gcc/gen/torque-generated/src/builtins/promise-constructor-tq-csa.cc'.

So to recap a little, when we do:
```js
const p1 = new Promise(function(resolve, reject) {
  console.log('Running p1 executor function...');
  resolve("success!");
});
```
This will invoke the builtin function Builtins::kPromiseConstructor that was
installed on the Promise object, which was either done upon startup if snapshots
are disabled, or by mksnapshot if they are enabled. 

TODO: continue exploration...

[Promise Objects](https://tc39.es/ecma262/#sec-promise-objects).

There is an example in [promise_test.cc](./test/promise_test.cc)


### EcmaScript Spec
To further my understanding of the V8 code base I've found it helpful to
read the [spec](https://tc39.es/ecma262/). This can make it clear why functions
and fields are named in certain ways and also why they do certain things.

#### Record
A Record in the spec is like a struct in c where each member is called a field.

#### Completion Record
Is a Record which is used as a return value and can have one of three possible
states:
```
[[Type]] (normal, return, throw, break, or continue)
```
If the type is normal, return, or throw then the CompletionRecord can have a
[[Value]] which is what is returned/thrown.
If the type is break or continue it can optionally have a [[Target]].

```js
function something() {
  if (bla) {
    return CompletionRecord({type: "normal", value: "something"});
  } else {
    return CompletionRecord({type: "throw", value: "error"});
  }
}
const result = something();
```
So a function is the spec would return a CompletionRecord and the spec has to
describe what is done in each case. If it was an abrupt (throw) in the example
above it return the value. Instead of writing that as text the spec writers
can use:
```
ReturnIfAbrupt(result)
or
Let result ? be something()
```

```js
function something() {
  return CompletionRecord({type: "normal", value: "something"});
}
const result = something();
```
```
Let result be !something()
```
This means that something() will never return an abrupt completion.

#### Builtin Objects
In the spec one can see object referred to as `%Array%` which referrs to builtin
objects. These are listed in [well-known-intrinsic-objects](https://tc39.es/ecma262/#sec-well-known-intrinsic-objects).

#### Realms
All js must be associated with a realm which consists of the builtin/intrinsic
objects, a global environment.

`src/parsing/parser.h` we can find:
```c++
class V8_EXPORT_PRIVATE Parser : public NON_EXPORTED_BASE(ParserBase<Parser>) { 
  ...
  enum CompletionKind {                                                             
    kNormalCompletion,                                                              
    kThrowCompletion,                                                               
    kAbruptCompletion                                                               
  };
```
But I can't find any usages of this enum? 

#### Internal fields/methods
When you see something like [[Notation]] you can think of this as a field in
an object that is not exposed to JavaScript user code but internal to the JavaScript
engine. These can also be used for internal methods.
