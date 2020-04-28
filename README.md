### Learning Google V8
The sole purpose of this project is to aid me in leaning Google's V8 JavaScript engine.


## Contents
1. [Introduction](#introduction)
1. [Address](#address)
1. [TaggedImpl](#taggedimpl)
1. [Object](#object)
1. [Handle](#handle)
1. [FunctionTemplate](#functiontemplate)
1. [ObjectTemplate](#objecttemplate)
1. [Small Integers](#small-integers)
1. [String types](#string-types)
1. [Roots](#roots)
1. [Builtins](#builtins)
1. [Compiler pipeline](#compiler-pipeline)
1. [CodeStubAssembler](#codestubassembler)
1. [Torque](#torque)
1. [WebAssembly](#webassembly)
1. [Promises](#promises)
1. [V8 Build artifacts](#v8-build-artifacts)
1. [V8 Startup walkthrough](#startup-walk-through)
1. [Building V8](#building-v8)
1. [Contributing a change](#contributing-a-change)
1. [Debugging](#debugging)
1. [Building chromium](#building-chromium)
1. [Goma chromium](#goma)

## Introduction
V8 is bascially consists of the memory management of the heap and the execution stack (very simplified but helps
make my point). Things like the callback queue, the event loop and other things like the WebAPIs (DOM, ajax, 
setTimeout etc) are found inside Chrome or in the case of Node the APIs are Node.js APIs:
```
    +------------------------------------------------------------------------------------------+
    | Google Chrome                                                                            |
    |                                                                                          |
    | +----------------------------------------+          +------------------------------+     |
    | | Google V8                              |          |            WebAPIs           |     |
    | | +-------------+ +---------------+      |          |                              |     |
    | | |    Heap     | |     Stack     |      |          |                              |     |
    | | |             | |               |      |          |                              |     |
    | | |             | |               |      |          |                              |     |
    | | |             | |               |      |          |                              |     |
    | | |             | |               |      |          |                              |     |
    | | |             | |               |      |          |                              |     |
    | | +-------------+ +---------------+      |          |                              |     |
    | |                                        |          |                              |     |
    | +----------------------------------------+          +------------------------------+     |
    |                                                                                          |
    |                                                                                          |
    | +---------------------+     +---------------------------------------+                    |
    | |     Event loop      |     |          Callback queue               |                    |
    | |                     |     |                                       |                    |
    | +---------------------+     +---------------------------------------+                    |
    |                                                                                          |
    |                                                                                          |
    +------------------------------------------------------------------------------------------+
```
The execution stack is a stack of frame pointers. For each function called that function will be pushed onto 
the stack. When that function returns it will be removed. If that function calls other functions
they will be pushed onto the stack. When they have all returned execution can proceed from the returned to 
point. If one of the functions performs an operation that takes time progress will not be made until it 
completes as the only way to complete is that the function returns and is popped off the stack. This is 
what happens when you have a single threaded programming language.

So that describes synchronous functions, what about asynchronous functions?  
Lets take for example that you call setTimeout, the setTimeout function will be
pushed onto the call stack and executed. This is where the callback queue comes into play and the event loop. The setTimeout function can add functions to the callback queue. This queue will be processed by the event loop when the call stack is empty.

TODO: Add mirco task queue

### Isolate
An Isolate is an independant copy of the V8 runtime which includes its own heap.
Two different Isolates can run in parallel and can be seen as entirely different
sandboxed instances of a V8 runtime.

### Context
To allow separate JavaScript applications to run in the same isolate a context
must be specified for each one.  This is to avoid them interfering with each
other, for example by changing the builtin objects provided.

### ObjectTemplate
These allow you to create JavaScript objects without a dedicated constructor.
This would be something like:
```js
const obj = {};
```
This class is declared in include/v8.h and extends Template:
```c++
class V8_EXPORT ObjectTemplate : public Template { 
  ...
}
class V8_EXPORT Template : public Data {
  ...
}
class V8_EXPORT Data {
 private:                                                                       
  Data();                                                                       
};
```
Template does not have any members/fields, it only declared functions.

We create an instance of ObjectTemplate and we can add properties to it that
all instance created using this ObjectTemplate instance will have. This is done
by calling Set which is member of the Template class. You specify a Local<Name>
for the property. Name is a superclass for Symbols and Strings which can be both
be used as names for a property.

The implementation for `Set` can be found in `src/api/api.cc`:
```c++
void Template::Set<v8::Local<Name> name, v8::Local<Data> value, v8::PropertyAttribute attribute) {
  ...

  i::ApiNatives::AddDataProperty(isolate, templ, Utils::OpenHandle(*name),           
                                 value_obj,                                     
                                 static_cast<i::PropertyAttributes>(attribute));
}
```

There is an example in [objecttemplate_test.cc](./test/objecttemplate_test.cc)

### FunctionTemplate
Is a template that is used to create functions.

There is an example in [functionttemplate_test.cc](./test/functiontemplate_test.cc)

An instance of a function template can be created using:
```c++
  Local<FunctionTemplate> ft = FunctionTemplate::New(isolate_, function_callback, data);
  Local<Function> function = ft->GetFunction(context).ToLocalChecked();
```
And the function can be called using:
```c++
  MaybeLocal<Value> ret = function->Call(context, recv, 0, nullptr);
```
Function::Call can be found in `src/api/api.cc`: 
```c++
  bool has_pending_exception = false;
  auto self = Utils::OpenHandle(this);                                               
  i::Handle<i::Object> recv_obj = Utils::OpenHandle(*recv);                          
  i::Handle<i::Object>* args = reinterpret_cast<i::Handle<i::Object>*>(argv);   
  Local<Value> result;                                                               
  has_pending_exception = !ToLocal<Value>(                                           
      i::Execution::Call(isolate, self, recv_obj, argc, args), &result);
```
Notice that the result of `Call` which is a `MaybeHandle<Object>` will be
passed to ToLocal<Value> which is defined in `api.h`:
```c++
template <class T>                                                              
inline bool ToLocal(v8::internal::MaybeHandle<v8::internal::Object> maybe,      
                    Local<T>* local) {                                          
  v8::internal::Handle<v8::internal::Object> handle;                            
  if (maybe.ToHandle(&handle)) {                                                   
    *local = Utils::Convert<v8::internal::Object, T>(handle);                   
    return true;                                                                
  }                                                                                
  return false;                                                                 
```
So lets take a look at `Execution::Call` which can be found in `execution/execution.cc`
and it calls:
```c++
return Invoke(isolate, InvokeParams::SetUpForCall(isolate, callable, receiver, argc, argv));
```
`SetUpForCall` will return an `InvokeParams`. TODO: Take a closer look at InvokeParams.
```c++
V8_WARN_UNUSED_RESULT MaybeHandle<Object> Invoke(Isolate* isolate,              
                                                 const InvokeParams& params) {
```
```c++
Handle<Object> receiver = params.is_construct                             
                                    ? isolate->factory()->the_hole_value()         
                                    : params.receiver; 
```
In our case `is_construct` is false as we are not using `new` and the receiver,
the `this` in the function should be set to the receiver that we passed in. After
that we have `Builtins::InvokeApiFunction` 
```c++
auto value = Builtins::InvokeApiFunction(                                 
          isolate, params.is_construct, function, receiver, params.argc,        
          params.argv, Handle<HeapObject>::cast(params.new_target)); 
```

```c++
result = HandleApiCallHelper<false>(isolate, function, new_target,        
                                    fun_data, receiver, arguments);
```

api-arguments-inl.h has 
```c++
FunctionCallbackArguments::Call(CallHandlerInfo handler) {
  ...
  ExternalCallbackScope call_scope(isolate, FUNCTION_ADDR(f));                  
  FunctionCallbackInfo<v8::Value> info(values_, argv_, argc_);                  
  f(info);
  return GetReturnValue<Object>(isolate);
}
```
The call to f(info) is what invokes the callback, which is just a normal
function call. 

Back in `HandleApiCallHelper` we have:
```c++
Handle<Object> result = custom.Call(call_data);                             
                                                                                
RETURN_EXCEPTION_IF_SCHEDULED_EXCEPTION(isolate, Object);
```
`RETURN_EXCEPTION_IF_SCHEDULED_EXCEPTION` expands to:
```c++
Handle<Object> result = custom.Call(call_data);                             
do { 
  Isolate* __isolate__ = (isolate); 
  ((void) 0); 
  if (__isolate__->has_scheduled_exception()) { 
    __isolate__->PromoteScheduledException(); 
    return MaybeHandle<Object>(); 
  }
} while (false);
```
Notice that if there was an exception an empty object is returned.
Later in `Invoke` execution.cc 
```c++
  auto value = Builtins::InvokeApiFunction(                                 
          isolate, params.is_construct, function, receiver, params.argc,        
          params.argv, Handle<HeapObject>::cast(params.new_target));            
  bool has_exception = value.is_null();                                     
  if (has_exception) {                                                      
    if (params.message_handling == Execution::MessageHandling::kReport) {   
      isolate->ReportPendingMessages();                                     
    }                                                                       
    return MaybeHandle<Object>();                                           
  } else {                                                                  
    isolate->clear_pending_message();                                       
  }                                                                         
  return value;                         
```
Looking at this is looks like passing back an empty object will cause an 
exception to be triggered?


### Exceptions
When a function is called how is an exception reported/triggered in the call
chain?  

```c++
  Local<FunctionTemplate> ft = FunctionTemplate::New(isolate_, function_callback);
  Local<Function> function = ft->GetFunction(context).ToLocalChecked();
  Local<Object> recv = Object::New(isolate_);
  MaybeLocal<Value> ret = function->Call(context, recv, 0, nullptr);
```
`function->Call` will end up in `src/api/api.cc` `Function::Call` and will
in turn call `v8::internal::Execution::Call`:
```c++
  has_pending_exception = !ToLocal<Value>(                                           
      i::Execution::Call(isolate, self, recv_obj, argc, args), &result);             
  RETURN_ON_FAILED_EXECUTION(Value);                                                 
  RETURN_ESCAPED(result);            
```
Notice that the result of `Call` which is a `MaybeHandle<Object>` will be
passed to ToLocal<Value> which is defined in `api.h`:
```c++
template <class T>                                                              
inline bool ToLocal(v8::internal::MaybeHandle<v8::internal::Object> maybe,      
                    Local<T>* local) {                                          
  v8::internal::Handle<v8::internal::Object> handle;                            
  if (maybe.ToHandle(&handle)) {                                                   
    *local = Utils::Convert<v8::internal::Object, T>(handle);                   
    return true;                                                                
  }                                                                                
  return false;                                                                 
```
So lets take a look at `Execution::Call` which can be found in `execution/execution.cc`
and it calls:
```c++
return Invoke(isolate, InvokeParams::SetUpForCall(isolate, callable, receiver, argc, argv));
```
`InvokeParams` is a struct in execution.cc which has a few static functions
(one being `SetupForCall`) and also the following fields:
```
Handle<Object> target;                                                        
  Handle<Object> receiver;                                                      
  int argc;                                                                     
  Handle<Object>* argv;                                                         
  Handle<Object> new_target;                                                    
  MicrotaskQueue* microtask_queue;                                              
  Execution::MessageHandling message_handling;                                  
  MaybeHandle<Object>* exception_out;                                           
  bool is_construct;                                                            
  Execution::Target execution_target;                                           
  bool reschedule_terminate;               
```
`SetupUpForNew` will set up defaults in addition to set the values for the
constructor, new_target, argc, and argv.
Related to exception handling is:
```c++
params.message_handling = Execution::MessageHandling::kReport;
```
So with that out of the way lets focos on the `Invoke` function in execution.cc
around line 240 at the time of this writing.
Now, if the target which is our case is the `Function` is a JSFunction there is
a path which will be true in our case:
```c++
Handle<JSFunction> function = Handle<JSFunction>::cast(params.target);
```
Next `SaveAndSwitchContext` is called:
```c++
SaveAndSwitchContext save(isolate, function->context());
```
Which will call isolate->set_context(function_context).
Next, we have:
```c++
Handle<Object> receiver = params.is_construct                             
                                    ? isolate->factory()->the_hole_value()         
                                    : params.receiver;     
```
And in our case `params.is_construct` is false:
```c++
(gdb) p params.is_construct
$1 = false
```
So the receiver will just be set to the reciever set on the param which is the
receiver we passed in. Next, we have the call to `Builtins::InvokeApiFunction`
which can be found in `builtins/builtins-api.cc:
```c++
Handle<FunctionTemplateInfo> fun_data = function->IsFunctionTemplateInfo()
          ? Handle<FunctionTemplateInfo>::cast(function)
          : handle(JSFunction::cast(*function).shared().get_api_func_data(),
                   isolate);
```
```console
(gdb) p function->IsFunctionTemplateInfo()
$2 = false
```
So in our case the following will be executed:
```c++
          : handle(JSFunction::cast(*function).shared().get_api_func_data(),
                   isolate);
```
TODO: Look into JSFunction and SharedFunctionInfo.
```c++
Address small_argv[kBufferSize];
Address* argv;                                                                
const int frame_argc = argc + BuiltinArguments::kNumExtraArgsWithReceiver;
```
In our case we did not pass any arguments so argc is 0. 
```console
(gdb) p kBufferSize
$7 = 32
(gdb) p argc
$8 = 0
(gdb) p BuiltinArguments::kNumExtraArgsWithReceiver
$9 = 5
```
```c++
if (frame_argc <= kBufferSize) {
    argv = small_argv;
```
So in our case we will be using `small_argv` which is an Array of Address:es.
Next, this array will be populated:
```c++
int cursor = frame_argc - 1;                                                  
argv[cursor--] = receiver->ptr();
```
So we are setting the argv[4] to the receiver. The next argument set is:
```c++
argv[BuiltinArguments::kPaddingOffset] = ReadOnlyRoots(isolate).the_hole_value().ptr();  
argv[BuiltinArguments::kArgcOffset] = Smi::FromInt(frame_argc).ptr();         
argv[BuiltinArguments::kTargetOffset] = function->ptr();             
argv[BuiltinArguments::kNewTargetOffset] = new_target->ptr()

RelocatableArguments arguments(isolate, frame_argc, &argv[frame_argc - 1]);
result = HandleApiCallHelper<false>(isolate, function, new_target,        
                                    fun_data, receiver, arguments); 
```
So we can see that we are passing the function, new_target, `HandleApiCallHelper` 

```c++
Object raw_call_data = fun_data->call_code(); 
CallHandlerInfo call_data = CallHandlerInfo::cast(raw_call_data);
Object data_obj = call_data.data();
Handle<Object> result = custom.Call(call_data);
```
`Call` will land in `src/api/api-arguments-inl.h`:
```c++
Handle<Object> FunctionCallbackArguments::Call(CallHandlerInfo handler) {
  ...
  v8::FunctionCallback f = v8::ToCData<v8::FunctionCallback>(handler.callback()); 

}
```
This is the function callback in our example which is named `function_callback`:
```console
(gdb) p f
$14 = (v8::FunctionCallback) 0x413ec6 <function_callback(v8::FunctionCallbackInfo<v8::Value> const&)>
```
```c++
  VMState<EXTERNAL> state(isolate);                                             
  ExternalCallbackScope call_scope(isolate, FUNCTION_ADDR(f));                  
  FunctionCallbackInfo<v8::Value> info(values_, argv_, argc_); 
  f(info);                                                                      
  return GetReturnValue<Object>(isolate);                                       
}
```
This return will return to:
```c++
  RETURN_EXCEPTION_IF_SCHEDULED_EXCEPTION(isolate, Object);
```
Which will be expanded by the preprocessor into:
```c++
  do {
    Isolate* __isolate__ = (isolate);
    ((void) 0);
    if (__isolate__->has_scheduled_exception()) {
      __isolate__->PromoteScheduledException();
      return MaybeHandle<Object>();
    }
  } while (false);
```
In our case there was not exception so the following line will be reached:
```c++
if (result.is_null()) { 
  return isolate->factory()->undefined_value();
}
```
So, we returned an empty/null value from our function and the leads to the
undefined value to be returned. Just noting this as I'm not sure if it is
important of not but CustomArguments has a destructor that will be called
when the `FunctionCallbackArguments` instance goes out of scope:
```
template <typename T>                                                           
CustomArguments<T>::~CustomArguments() {                                        
  slot_at(kReturnValueOffset).store(Object(kHandleZapValue));                   
```
Back now in `Builtins::InvokeApiFunction`:
```c++
  MaybeHandle<Object> result;
  ... // HandleApiCallHelper was shown above 
  if (argv != small_argv) delete[] argv;                                        
  return result;
```
So after this return we will be back in `Invoke` in execution.cc:
```c++
  auto value = Builtins::InvokeApiFunction(
      isolate, params.is_construct, function, receiver, params.argc,
      params.argv, Handle<HeapObject>::cast(params.new_target));
  bool has_exception = value.is_null();
```
Now, even though are callback returned nothing/null, that was checked for and
instead undefined was returned. 

After this we will return to `Execution::Call` which will just return to
v8::ToLocal which will turn the Handle into a MaybeHandle.
This will then return us to Function::Call:
```c++
RETURN_ON_FAILED_EXECUTION(Value);
RETURN_ESCAPED(result);
```
`RETURN_ON_FAILED_EXECUTION` will expand to:
```c++
 do {
  if (has_pending_exception) {
    call_depth_scope.Escape();
    return MaybeLocal<Value>();
  }
} while (false);
```
And RETURN_ESCAPED:
```c++
return handle_scope.Escape(result);;
```
I wonder why this last line is a macro when it is just one line.
Finally we will be back in our test 
```c++
  MaybeLocal<Value> ret = function->Call(context, recv, 0, nullptr);
```



```console
(gdb) p result.is_null()
$15 = true
```

### Exception
When calling a Function one can throw an exception using:
```c++
  isolate->ThrowException(String::NewFromUtf8(isolate, "some error").ToLocalChecked());
```
`ThrowException` can be found in `src/api/api.cc` and what it does is:
```c++
  ENTER_V8_DO_NOT_USE(isolate);                                                 
  // If we're passed an empty handle, we throw an undefined exception           
  // to deal more gracefully with out of memory situations.                     
  if (value.IsEmpty()) {                                                        
    isolate->ScheduleThrow(i::ReadOnlyRoots(isolate).undefined_value());             
  } else {                                                                           
    isolate->ScheduleThrow(*Utils::OpenHandle(*value));                              
  }                                                                                  
  return v8::Undefined(reinterpret_cast<v8::Isolate*>(isolate));      
```
Lets take a closer look at `ScheduleThrow`. 
```c++
void Isolate::ScheduleThrow(Object exception) {                                     
  Throw(exception);                                                                 
  PropagatePendingExceptionToExternalTryCatch();                                    
  if (has_pending_exception()) {                                                    
    thread_local_top()->scheduled_exception_ = pending_exception();                 
    thread_local_top()->external_caught_exception_ = false;                         
    clear_pending_exception();                                                      
  }                                                                                 
}
```
`Throw` will end by setting the pending_exception to the exception passed in.
Next, `PropagatePendingExceptionToExternalTryCatch` will be called. This is
where a `TryCatch` handler comes into play. If one had been registered, which as
I'm writing this I did not have one (but will add one to try it out and verify).
The code for this part looks like this:
```c++
  v8::TryCatch* handler = try_catch_handler();
  handler->can_continue_ = true;
  handler->has_terminated_ = false;
  handler->exception_ = reinterpret_cast<void*>(pending_exception().ptr());
  // Propagate to the external try-catch only if we got an actual message.
  if (thread_local_top()->pending_message_obj_.IsTheHole(this)) return true;
  handler->message_obj_ = reinterpret_cast<void*>(thread_local_top()->pending_message_obj_.ptr());
```  
When a TryCatch is created its constructor will call `RegisterTryCatchHandler`
which will set the thread_local_top try_catch_handler which is retrieved above.

Prior to this there will be a call to `IsJavaScriptHandlerOnTop`:
```c++
// For uncatchable exceptions, the JavaScript handler cannot be on top.           
if (!is_catchable_by_javascript(exception)) return false;
```
```c++
  return exception != ReadOnlyRoots(heap()).termination_exception();               
}
```
I really need to understand this better and the various ways to catch/handle
exceptions (from C++ and JavaScript). 
Next (in PropagatePendingExceptionToExternalTryCatch) we have:
```c++
  // Get the top-most JS_ENTRY handler, cannot be on top if it doesn't exist.   
  Address entry_handler = Isolate::handler(thread_local_top());                 
  if (entry_handler == kNullAddress) return false;
```
Next, he have the following:
```c++
  if (!IsExternalHandlerOnTop(exception)) {                                     
    thread_local_top()->external_caught_exception_ = false;                     
    return true;                                                                
  }
```
I'm really confused at the moment with these different handler, we have one
for. 


Now, for a javascript function that is executed using `Run` what would be
used in execution.cc Execution::Call would be:
```c++
Handle<Code> code = JSEntry(isolate, params.execution_target, params.is_construct);
```
```c++
Handle<Code> JSEntry(Isolate* isolate, Execution::Target execution_target, bool is_construct) {
  if (is_construct) {
    return BUILTIN_CODE(isolate, JSConstructEntry);
  } else if (execution_target == Execution::Target::kCallable) {
    return BUILTIN_CODE(isolate, JSEntry);
    isolate->builtins()->builtin_handle(Builtins::kJSEntry)
  } else if (execution_target == Execution::Target::kRunMicrotasks) {
    return BUILTIN_CODE(isolate, JSRunMicrotasksEntry);
  }
  UNREACHABLE();
}
```

```c++
if (params.execution_target == Execution::Target::kCallable) {
  // clang-format off
  // {new_target}, {target}, {receiver}, return value: tagged pointers
  // {argv}: pointer to array of tagged pointers
  using JSEntryFunction = GeneratedCode<Address(
      Address root_register_value, Address new_target, Address target,
      Address receiver, intptr_t argc, Address** argv)>;
  JSEntryFunction stub_entry =
      JSEntryFunction::FromAddress(isolate, code->InstructionStart());
  Address orig_func = params.new_target->ptr();
  Address func = params.target->ptr();
  Address recv = params.receiver->ptr();
  Address** argv = reinterpret_cast<Address**>(params.argv);

  RuntimeCallTimerScope timer(isolate, RuntimeCallCounterId::kJS_Execution);

  value = Object(stub_entry.Call(isolate->isolate_data()->isolate_root(),
                                 orig_func, func, recv, params.argc, argv));
```

### Address
`Address` can be found in `include/v8-internal.h`:

```c++
typedef uintptr_t Address;
```
`uintptr_t` is an optional type specified in cstdint and is capable of storing
a data pointer. It is an unsigned integer type that any valid pointer to void
can be converted to this type (and back).

### TaggedImpl
This class is declared in `src/objects/tagged-impl.h and has a single private
member which is declared as:
```c++
 public
  constexpr StorageType ptr() const { return ptr_; }
 private:
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

See [tagged_test.cc](./test/tagged_test.cc) for an example.

### Object
This class extends TaggedImpl:
```c++
class Object : public TaggedImpl<HeapObjectReferenceType::STRONG, Address> {       
```
An Object can be created using the default constructor, or by passing in an 
Address which will delegate to TaggedImpl constructors. Object itself does
not have any members (apart from `ptr_` which is inherited from TaggedImpl that is). 
So if we create an Object on the stack this is like a pointer/reference to
an object: 
```
+------+
|Object|
|------|
|ptr_  |---->
+------+
```
Now, `ptr_` is a StorageType so it could be a Smi in which case it would just
contains the value directly, for example a small integer:
```
+------+
|Object|
|------|
|  18  |
+------+
```
See [object_test.cc](./test/object_test.cc) for an example.

### ObjectSlot
```c++
  i::Object obj{18};
  i::FullObjectSlot slot{&obj};
```

```
+----------+      +---------+
|ObjectSlot|      | Object  |
|----------|      |---------|
| address  | ---> |   18    |
+----------+      +---------+
```
See [objectslot_test.cc](./test/objectslot_test.cc) for an example.

### Maybe
A Maybe is like an optional which can either hold a value or nothing.
```c++
template <class T>                                                              
class Maybe {
 public:
  V8_INLINE bool IsNothing() const { return !has_value_; }                      
  V8_INLINE bool IsJust() const { return has_value_; }
  ...

 private:
  bool has_value_;                                                              
  T value_; 
}
```
I first thought that name `Just` was a little confusing but if you read this
like:
```c++
  bool cond = true;
  Maybe<int> maybe = cond ? Just<int>(10) : Nothing<int>();
```
I think it makes more sense. There are functions that check if the Maybe is
nothing and crash the process if so. You can also check and return the value
by using `FromJust`. 

The usage of Maybe is where api calls can fail and returning Nothing is a way
of signaling this.

See [maybe_test.cc](./test/maybe_test.cc) for an example.

### MaybeLocal
```c++
template <class T>                                                              
class MaybeLocal {
 public:                                                                        
  V8_INLINE MaybeLocal() : val_(nullptr) {} 
  V8_INLINE Local<T> ToLocalChecked();
  V8_INLINE bool IsEmpty() const { return val_ == nullptr; }
  template <class S>                                                            
  V8_WARN_UNUSED_RESULT V8_INLINE bool ToLocal(Local<S>* out) const {           
    out->val_ = IsEmpty() ? nullptr : this->val_;                               
    return !IsEmpty();                                                          
  }    

 private:
  T* val_;
```
`ToLocalChecked` will crash the process if `val_` is a nullptr. If you want to
avoid a crash one can use `ToLocal`.

See [maybelocal_test.cc](./test/maybelocal_test.cc) for an example.

### Data
Is the super class of all objects that can exist the V8 heap:
```c++
class V8_EXPORT Data {                                                          
 private:                                                                       
  Data();                                                                       
};
```

### Value
Value extends Data and adds a number of methods that check if a Value
is of a certain type, like `IsUndefined()`, `IsNull`, `IsNumber` etc.
It also has useful methods to convert to a Local<T>, for example:
```c++
V8_WARN_UNUSED_RESULT MaybeLocal<Number> ToNumber(Local<Context> context) const;
V8_WARN_UNUSED_RESULT MaybeLocal<String> ToNumber(Local<String> context) const;
...
```


### Handle
A Handle is similar to a Object and ObjectSlot in that it also contains
an Address member (called `location_` and declared in `HandleBase`), but with the
difference is that Handles acts as a layer of abstraction and can be relocated
by the garbage collector.
Can be found in `src/handles/handles.h`.

```c++
class HandleBase {  
 ...
 protected:
  Address* location_; 
}
template <typename T>                                                           
class Handle final : public HandleBase {
  ...
}
```

```
+----------+                  +--------+         +---------+
|  Handle  |                  | Object |         |   int   |
|----------|      +-----+     |--------|         |---------|
|*location_| ---> |&ptr_| --> | ptr_   | ----->  |     5   |
+----------+      +-----+     +--------+         +---------+
```
```console
(gdb) p handle
$8 = {<v8::internal::HandleBase> = {location_ = 0x7ffdf81d60c0}, <No data fields>}
```
Notice that `location_` contains a pointer:
```console
(gdb) p /x *(int*)0x7ffdf81d60c0
$9 = 0xa9d330
```
And this is the same as the value in obj:
```console
(gdb) p /x obj.ptr_
$14 = 0xa9d330
```
And we can access the int using any of the pointers:
```console
(gdb) p /x *value
$16 = 0x5
(gdb) p /x *obj.ptr_
$17 = 0x5
(gdb) p /x *(int*)0x7ffdf81d60c0
$18 = 0xa9d330
(gdb) p /x *(*(int*)0x7ffdf81d60c0)
$19 = 0x5
```

See [handle_test.cc](./test/handle_test.cc) for an example.

### HandleScope
A HandleScope only has three members:
```c++
  internal::Isolate* isolate_;
  internal::Address* prev_next_;
  internal::Address* prev_limit_;
```

Lets take a closer look at what happens when we construct a HandleScope:
```c++
  v8::HandleScope handle_scope{isolate_};
```
The constructor call will end up in `src/api/api.cc` and the constructor simply delegates to
`Initialize`:
```c++
HandleScope::HandleScope(Isolate* isolate) { Initialize(isolate); }

void HandleScope::Initialize(Isolate* isolate) {
  i::Isolate* internal_isolate = reinterpret_cast<i::Isolate*>(isolate);
  ...
  i::HandleScopeData* current = internal_isolate->handle_scope_data();
  isolate_ = internal_isolate;
  prev_next_ = current->next;
  prev_limit_ = current->limit;
  current->level++;
}
```
Every v8::internal::Isolate has member of type HandleScopeData:
```c++
HandleScopeData handle_scope_data_;
HandleScopeData* handle_scope_data() { return &handle_scope_data_; }
```
HandleScopeData is a struct defined in `src/handles/handles.h`:
```c++
struct HandleScopeData final {
  Address* next;
  Address* limit;
  int level;
  int sealed_level;
  CanonicalHandleScope* canonical_scope;

  void Initialize() {
    next = limit = nullptr;
    sealed_level = level = 0;
    canonical_scope = nullptr;
  }
};
```
Notice that there are two pointers (Address*) to next and a limit. When a 
HandleScope is Initialized the current handle_scope_data will be retrieved 
from the internal isolate. The HandleScope instance that is getting created
stores the next/limit pointers of the current isolate so that they can be restored
when this HandleScope is closed (see CloseScope).

So with a HandleScope created, how does a Local<T> interact with this instance?  
HandleScope:CreateHandle will get the handle_scope_data from the isolate:
```c++
Address* HandleScope::CreateHandle(Isolate* isolate, Address value) {
  HandleScopeData* data = isolate->handle_scope_data();
  if (result == data->limit) {
    result = Extend(isolate);
  }
  // Update the current next field, set the value in the created handle,        
  // and return the result.
  data->next = reinterpret_cast<Address*>(reinterpret_cast<Address>(result) + sizeof(Address));
  *result = value;
  return result;
}                         
```

When a Local<T> is created this will/might go through FactoryBase::NewStruct
which will allocate a new Map and then create a Handle for the InstanceType
being created:
```c++
Handle<Struct> str = handle(Struct::cast(result), isolate()); 
```
This will land in the constructor Handle<T>src/handles/handles-inl.h
```c++
template <typename T>                                                           
Handle<T>::Handle(T object, Isolate* isolate): HandleBase(object.ptr(), isolate) {}

HandleBase::HandleBase(Address object, Isolate* isolate)                        
    : location_(HandleScope::GetHandle(isolate, object)) {}
```
Notice that `object.ptr()` is used to pass the Address to HandleBase.
And also notice that HandleBase sets its location_ to the result of HandleScope::GetHandle.

```c++
Address* HandleScope::GetHandle(Isolate* isolate, Address value) {              
  DCHECK(AllowHandleAllocation::IsAllowed());                                   
  HandleScopeData* data = isolate->handle_scope_data();                         
  CanonicalHandleScope* canonical = data->canonical_scope;                      
  return canonical ? canonical->Lookup(value) : CreateHandle(isolate, value);   
}
```
Which will call CreateHandle in this case and this function will retrieve the
current isolate's handle_scope_data:
```c++
  HandleScopeData* data = isolate->handle_scope_data();                         
  Address* result = data->next;                                                 
  if (result == data->limit) {                                                  
    result = Extend(isolate);                                                   
  }     
```
In this case both next and limit will be 0x0 so Extend will be called.
Extend will also get the isolates handle_scope_data and check the current level
and after that get the isolates HandleScopeImplementer:
```c++
  HandleScopeImplementer* impl = isolate->handle_scope_implementer();           
```
`HandleScopeImplementer` is declared in `src/api/api.h`


The destructor for HandleScope will call CloseScope.
See [handlescope_test.cc](./test/handlescope_test.cc) for an example.

### HeapObject
TODO:

### Local
Has a single member `val_` which is of type pointer to `T`:
```c++
template <class T> class Local { 
...
 private:
  T* val_
}
```
Notice that this is a pointer to T. We could create a local using:
```c++
  v8::Local<v8::Value> empty_value;
```

So a Local contains a pointer to type T. We can access this pointer using
`operator->` and `operator*`.

We can cast from a subtype to a supertype using Local::Cast:
```c++
v8::Local<v8::Number> nr = v8::Local<v8::Number>(v8::Number::New(isolate_, 12));
v8::Local<v8::Value> val = v8::Local<v8::Value>::Cast(nr);
```
And there is also the 
```c++
v8::Local<v8::Value> val2 = nr.As<v8::Value>();
```

See [local_test.cc](./test/local_test.cc) for an example.

### MaybeLocal


### PrintObject
Using _v8_internal_Print_Object from c++:
```console
$ nm -C libv8_monolith.a | grep Print_Object
0000000000000000 T _v8_internal_Print_Object(void*)
```
Notice that this function does not have a namespace.
We can use this as:
```c++
extern void _v8_internal_Print_Object(void* object);

_v8_internal_Print_Object(*((v8::internal::Object**)(*global)));
```
Lets take a closer look at the above:
```c++
  v8::internal::Object** gl = ((v8::internal::Object**)(*global));
```
We use the dereference operator to get the value of a Local (*global), which is
just of type `T*`, a pointer to the type the Local. We are then casting that to
be of type pointer-to-pointer to Object.
```
  gl         Object*         Object
+-----+      +------+      +-------+
|     |----->|      |----->|       |
+-----+      +------+      +-------+
```
An instance of v8::internal::Object only has a single data member which is a
field named `ptr_` of type `Address`:

`src/objects/objects.h`:
```c++
class Object : public TaggedImpl<HeapObjectReferenceType::STRONG, Address> {
 public:
  constexpr Object() : TaggedImpl(kNullAddress) {}
  explicit constexpr Object(Address ptr) : TaggedImpl(ptr) {}

#define IS_TYPE_FUNCTION_DECL(Type) \
  V8_INLINE bool Is##Type() const;  \
  V8_INLINE bool Is##Type(const Isolate* isolate) const;
  OBJECT_TYPE_LIST(IS_TYPE_FUNCTION_DECL)
  HEAP_OBJECT_TYPE_LIST(IS_TYPE_FUNCTION_DECL)
  IS_TYPE_FUNCTION_DECL(HashTableBase)
  IS_TYPE_FUNCTION_DECL(SmallOrderedHashTable)
#undef IS_TYPE_FUNCTION_DECL
  V8_INLINE bool IsNumber(ReadOnlyRoots roots) const;
}
```
Lets take a look at one of these functions and see how it is implemented. For
example in the OBJECT_TYPE_LIST we have:
```c++
#define OBJECT_TYPE_LIST(V) \
  V(LayoutDescriptor)       \
  V(Primitive)              \
  V(Number)                 \
  V(Numeric)
```
So the object class will have a function that looks like:
```c++
inline bool IsNumber() const;
inline bool IsNumber(const Isolate* isolate) const;
```
And in src/objects/objects-inl.h we will have the implementations:
```c++
bool Object::IsNumber() const {
  return IsHeapObject() && HeapObject::cast(*this).IsNumber();
}
```
`IsHeapObject` is defined in TaggedImpl:
```c++
  constexpr inline bool IsHeapObject() const { return IsStrong(); }

  constexpr inline bool IsStrong() const {
#if V8_HAS_CXX14_CONSTEXPR
    DCHECK_IMPLIES(!kCanBeWeak, !IsSmi() == HAS_STRONG_HEAP_OBJECT_TAG(ptr_));
#endif
    return kCanBeWeak ? HAS_STRONG_HEAP_OBJECT_TAG(ptr_) : !IsSmi();
  }
```

The macro can be found in src/common/globals.h:
```c++
#define HAS_STRONG_HEAP_OBJECT_TAG(value)                          \
  (((static_cast<i::Tagged_t>(value) & ::i::kHeapObjectTagMask) == \
    ::i::kHeapObjectTag))
```
So we are casting `ptr_` which is of type Address into type `Tagged_t` which
is defined in src/common/global.h and can be different depending on if compressed
pointers are used or not. If they are  not supported it is the same as Address:
``` 
using Tagged_t = Address;
```

`src/objects/tagged-impl.h`:
```c++
template <HeapObjectReferenceType kRefType, typename StorageType>
class TaggedImpl {

  StorageType ptr_;
}
```
The HeapObjectReferenceType can be either WEAK or STRONG. And the storage type
is `Address` in this case. So Object itself only has one member that is inherited
from its only super class and this is `ptr_`.

So the following is telling the compiler to treat the value of our Local,
`*global`, as a pointer (which it already is) to a pointer that points to
a memory location that confirms to the layout of an v8::internal::Object type,
which we know now has a `prt_` member. And we want to dereference it and pass
it into the function.
```c++
_v8_internal_Print_Object(*((v8::internal::Object**)(*global)));
```

But I'm still missing the connection between ObjectTemplate and object.
When we create it we use:
```c++
Local<ObjectTemplate> global = ObjectTemplate::New(isolate);
```
In `src/api/api.cc` we have:
```c++
static Local<ObjectTemplate> ObjectTemplateNew(
    i::Isolate* isolate, v8::Local<FunctionTemplate> constructor,
    bool do_not_cache) {
  i::Handle<i::Struct> struct_obj = isolate->factory()->NewStruct(
      i::OBJECT_TEMPLATE_INFO_TYPE, i::AllocationType::kOld);
  i::Handle<i::ObjectTemplateInfo> obj = i::Handle<i::ObjectTemplateInfo>::cast(struct_obj);
  InitializeTemplate(obj, Consts::OBJECT_TEMPLATE);
  int next_serial_number = 0;
  if (!constructor.IsEmpty())
    obj->set_constructor(*Utils::OpenHandle(*constructor));
  obj->set_data(i::Smi::zero());
  return Utils::ToLocal(obj);
}
```
What is a `Struct` in this context?  
`src/objects/struct.h`
```c++
#include "torque-generated/class-definitions-tq.h"

class Struct : public TorqueGeneratedStruct<Struct, HeapObject> {
 public:
  inline void InitializeBody(int object_size);
  void BriefPrintDetails(std::ostream& os);
  TQ_OBJECT_CONSTRUCTORS(Struct)
```
Notice that the include is specifying `torque-generated` include which can be
found `out/x64.release_gcc/gen/torque-generated/class-definitions-tq`. So, somewhere
there must be an call to the `torque` executable which generates the Code Stub
Assembler C++ headers and sources before compiling the main source files. There is
and there is a section about this in `Building V8`.
The macro `TQ_OBJECT_CONSTRUCTORS` can be found in `src/objects/object-macros.h`
and expands to:
```c++
  constexpr Struct() = default;

 protected:
  template <typename TFieldType, int kFieldOffset>
  friend class TaggedField;

  inline explicit Struct(Address ptr);
```

So what does the TorqueGeneratedStruct look like?
```
template <class D, class P>
class TorqueGeneratedStruct : public P {
 public:
```
Where D is Struct and P is HeapObject in this case. But the above is the declartion
of the type but what we have in the .h file is what was generated. 

This type is defined in `src/objects/struct.tq`:
```
@abstract                                                                       
@generatePrint                                                                  
@generateCppClass                                                               
extern class Struct extends HeapObject {                                        
} 
```

`NewStruct` can be found in `src/heap/factory-base.cc`
```c++
template <typename Impl>
HandleFor<Impl, Struct> FactoryBase<Impl>::NewStruct(
    InstanceType type, AllocationType allocation) {
  Map map = Map::GetStructMap(read_only_roots(), type);
  int size = map.instance_size();
  HeapObject result = AllocateRawWithImmortalMap(size, allocation, map);
  HandleFor<Impl, Struct> str = handle(Struct::cast(result), isolate());
  str->InitializeBody(size);
  return str;
}
```
Every object that is stored on the v8 heap has a Map (`src/objects/map.h`) that
describes the structure of the object being stored.
```c++
class Map : public HeapObject {
```

```console
1725	  return Utils::ToLocal(obj);
(gdb) p obj
$6 = {<v8::internal::HandleBase> = {location_ = 0x30b5160}, <No data fields>}
```
So this is the connection, what we see as a Local<ObjectTemplate> is a HandleBase.
TODO: dig into this some more when I have time.


```console
(lldb) expr gl
(v8::internal::Object **) $0 = 0x00000000020ee160
(lldb) memory read -f x -s 8 -c 1 gl
0x020ee160: 0x00000aee081c0121

(lldb) memory read -f x -s 8 -c 1 *gl
0xaee081c0121: 0x0200000002080433
```


You can reload `.lldbinit` using the following command:
```console
(lldb) command source ~/.lldbinit
```
This can be useful when debugging a lldb command. You can set a breakpoint
and break at that location and make updates to the command and reload without
having to restart lldb.

Currently, the lldb-commands.py that ships with v8 contains an extra operation
of the parameter pased to `ptr_arg_cmd`:
```python
def ptr_arg_cmd(debugger, name, param, cmd):                                    
  if not param:                                                                 
    print("'{}' requires an argument".format(name))                             
    return                                                                      
  param = '(void*)({})'.format(param)                                           
  no_arg_cmd(debugger, cmd.format(param)) 
```
Notice that `param` is the object that we want to print, for example lets say
it is a local named obj:
```
param = "(void*)(obj)"
```
This will then be "passed"/formatted into the command string:
```
"_v8_internal_Print_Object(*(v8::internal::Object**)(*(void*)(obj))")
```

#### Threads
V8 is single threaded (the execution of the functions of the stack) but there
are supporting threads used for garbage collection, profiling (IC, and perhaps
other things) (I think).
Lets see what threads there are:

    $ LD_LIBRARY_PATH=../v8_src/v8/out/x64.release_gcc/ lldb ./hello-world 
    (lldb) br s -n main
    (lldb) r
    (lldb) thread list
    thread #1: tid = 0x2efca6, 0x0000000100001e16 hello-world`main(argc=1, argv=0x00007fff5fbfee98) + 38 at hello-world.cc:40, queue = 'com.apple.main-thread', stop reason = breakpoint 1.1

So at startup there is only one thread which is what we expected. Lets skip ahead to where we create the platform:

    Platform* platform = platform::CreateDefaultPlatform();
    ...
    DefaultPlatform* platform = new DefaultPlatform(idle_task_support, tracing_controller);
    platform->SetThreadPoolSize(thread_pool_size);

    (lldb) fr v thread_pool_size
    (int) thread_pool_size = 0

Next there is a check for 0 and the number of processors -1 is used as the size of the thread pool:

    (lldb) fr v thread_pool_size
    (int) thread_pool_size = 7

This is all that `SetThreadPoolSize` does. After this we have:

    platform->EnsureInitialized();

    for (int i = 0; i < thread_pool_size_; ++i)
      thread_pool_.push_back(new WorkerThread(&queue_));

`new WorkerThread` will create a new pthread (on my system which is MacOSX):

    result = pthread_create(&data_->thread_, &attr, ThreadEntry, this);

ThreadEntry can be found in src/base/platform/platform-posix.


### International Component for Unicode (ICU)
International Components for Unicode (ICU) deals with internationalization (i18n).
ICU provides support locale-sensitve string comparisons, date/time/number/currency formatting
etc. 

There is an optional API called ECMAScript 402 which V8 suppports and which is enabled by
default. [i18n-support](https://github.com/v8/v8/wiki/i18n-support) says that even if your application does 
not use ICU you still need to call InitializeICU :

    V8::InitializeICU();

### Snapshot
JavaScript specifies a lot of built-in functionality which every V8 context must provide.
For example, you can run Math.PI and that will work in a JavaScript console/repl.
The global object and all the built-in functionality must be setup and initialized
into the V8 heap. This can be time consuming and affect runtime performance if
this has to be done every time. 

Now this is where the file `snapshot_blob.bin` comes into play. 
But what are this bin file?  
The blobs above are prepared snapshots that get directly deserialized into the
heap to provide an initilized context.

There is an executable named `mksnapshot` which is defined in 
`src/snapshot/mksnapshot.c`. 

When V8 is built with `v8_use_external_startup_data` the build process will
create a snapshot_blob.bin file using a template in BUILD.gn named `run_mksnapshot`
, but if false it will generate a file named snapshot.cc. 


