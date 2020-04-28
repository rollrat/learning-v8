
### Local

```c++
Local<String> script_name = ...;
```
So what is script_name. Well it is an object reference that is managed by the v8 GC.
The GC needs to be able to move things (pointers around) and also track if
things should be GC'd. Local handles as opposed to persistent handles are light
weight and mostly used local operations. These handles are managed by
HandleScopes so you must have a handlescope on the stack and the local is only
valid as long as the handlescope is valid. This uses Resource Acquisition Is
Initialization (RAII) so when the HandleScope instance goes out of scope it
will remove all the Local instances.

The `Local` class (in `include/v8.h`) only has one member which is of type
pointer to the type `T`. So for the above example it would be:
```c++
  String* val_;
```
You can find the available operations for a Local in `include/v8.h`.

```shell
(lldb) p script_name.IsEmpty()
(bool) $12 = false
````

A Local<T> has overloaded a number of operators, for example ->:
```shell
(lldb) p script_name->Length()
(int) $14 = 7
````
Where Length is a method on the v8 String class.

The handle stack is not part of the C++ call stack, but the handle scopes are
embedded in the C++ stack. Handle scopes can only be stack-allocated, not
allocated with new.

### Persistent
https://v8.dev/docs/embed:
Persistent handles provide a reference to a heap-allocated JavaScript Object, 
just like a local handle. There are two flavors, which differ in the lifetime
management of the reference they handle. Use a persistent handle when you need
to keep a reference to an object for more than one function call, or when handle
lifetimes do not correspond to C++ scopes. Google Chrome, for example, uses
persistent handles to refer to Document Object Model (DOM) nodes.

A persistent handle can be made weak, using PersistentBase::SetWeak, to trigger
a callback from the garbage collector when the only references to an object are
from weak persistent handles.


A UniquePersistent<SomeType> handle relies on C++ constructors and destructors
to manage the lifetime of the underlying object.
A Persistent<SomeType> can be constructed with its constructor, but must be
explicitly cleared with Persistent::Reset.

So how is a persistent object created?  
Let's write a test and find out (`test/persistent-object_text.cc`):
```console
$ make test/persistent-object_test
$ ./test/persistent-object_test --gtest_filter=PersistentTest.value
```
Now, to create an instance of Persistent we need a Local<T> instance or the
Persistent instance will just be empty.
```c++
Local<Object> o = Local<Object>::New(isolate_, Object::New(isolate_));
```
`Local<Object>::New` can be found in `src/api/api.cc`:
```c++
Local<v8::Object> v8::Object::New(Isolate* isolate) {
  i::Isolate* i_isolate = reinterpret_cast<i::Isolate*>(isolate);
  LOG_API(i_isolate, Object, New);
  ENTER_V8_NO_SCRIPT_NO_EXCEPTION(i_isolate);
  i::Handle<i::JSObject> obj =
      i_isolate->factory()->NewJSObject(i_isolate->object_function());
  return Utils::ToLocal(obj);
}
```
The first thing that happens is that the public Isolate pointer is cast to an
pointer to the internal `Isolate` type.
`LOG_API` is a macro in the same source file (src/api/api.cc):
```c++
#define LOG_API(isolate, class_name, function_name)                           \
  i::RuntimeCallTimerScope _runtime_timer(                                    \
      isolate, i::RuntimeCallCounterId::kAPI_##class_name##_##function_name); \
  LOG(isolate, ApiEntryCall("v8::" #class_name "::" #function_name))
```
If our case the preprocessor would expand that to:
```c++
  i::RuntimeCallTimerScope _runtime_timer(
      isolate, i::RuntimeCallCounterId::kAPI_Object_New);
  LOG(isolate, ApiEntryCall("v8::Object::New))
```
`LOG` is a macro that can be found in `src/log.h`:
```c++
#define LOG(isolate, Call)                              \
  do {                                                  \
    v8::internal::Logger* logger = (isolate)->logger(); \
    if (logger->is_logging()) logger->Call;             \
  } while (false)
```
And this would expand to:
```c++
  v8::internal::Logger* logger = isolate->logger();
  if (logger->is_logging()) logger->ApiEntryCall("v8::Object::New");
```
So with the LOG_API macro expanded we have:
```c++
Local<v8::Object> v8::Object::New(Isolate* isolate) {
  i::Isolate* i_isolate = reinterpret_cast<i::Isolate*>(isolate);
  i::RuntimeCallTimerScope _runtime_timer( isolate, i::RuntimeCallCounterId::kAPI_Object_New);
  v8::internal::Logger* logger = isolate->logger();
  if (logger->is_logging()) logger->ApiEntryCall("v8::Object::New");

  ENTER_V8_NO_SCRIPT_NO_EXCEPTION(i_isolate);
  i::Handle<i::JSObject> obj =
      i_isolate->factory()->NewJSObject(i_isolate->object_function());
  return Utils::ToLocal(obj);
}
```
Next we have `ENTER_V8_NO_SCRIPT_NO_EXCEPTION`:
```c++
#define ENTER_V8_NO_SCRIPT_NO_EXCEPTION(isolate)                    \
  i::VMState<v8::OTHER> __state__((isolate));                       \
  i::DisallowJavascriptExecutionDebugOnly __no_script__((isolate)); \
  i::DisallowExceptions __no_exceptions__((isolate))
```
So with the macros expanded we have:
```c++
Local<v8::Object> v8::Object::New(Isolate* isolate) {
  i::Isolate* i_isolate = reinterpret_cast<i::Isolate*>(isolate);
  i::RuntimeCallTimerScope _runtime_timer( isolate, i::RuntimeCallCounterId::kAPI_Object_New);
  v8::internal::Logger* logger = isolate->logger();
  if (logger->is_logging()) logger->ApiEntryCall("v8::Object::New");

  i::VMState<v8::OTHER> __state__(i_isolate));
  i::DisallowJavascriptExecutionDebugOnly __no_script__(i_isolate);
  i::DisallowExceptions __no_exceptions__(i_isolate));

  i::Handle<i::JSObject> obj =
      i_isolate->factory()->NewJSObject(i_isolate->object_function());

  return Utils::ToLocal(obj);
}
```
TODO: Look closer at `VMState`.  

First, `i_isolate->object_function()` is called and the result passed to
`NewJSObject`. `object_function` is generated by a macro named 
`NATIVE_CONTEXT_FIELDS`:
```c++
#define NATIVE_CONTEXT_FIELD_ACCESSOR(index, type, name)     \
  Handle<type> Isolate::name() {                             \
    return Handle<type>(raw_native_context()->name(), this); \
  }                                                          \
  bool Isolate::is_##name(type* value) {                     \
    return raw_native_context()->is_##name(value);           \
  }
NATIVE_CONTEXT_FIELDS(NATIVE_CONTEXT_FIELD_ACCESSOR)
```
`NATIVE_CONTEXT_FIELDS` is a macro in `src/contexts` and it c
```c++
#define NATIVE_CONTEXT_FIELDS(V)                                               \
...                                                                            \
  V(OBJECT_FUNCTION_INDEX, JSFunction, object_function)                        \
```

```c++
  Handle<type> Isolate::object_function() {
    return Handle<JSFunction>(raw_native_context()->object_function(), this);
  }

  bool Isolate::is_object_function(JSFunction* value) {
    return raw_native_context()->is_object_function(value);
  }
```
I'm not clear on the different types of context, there is a native context, a "normal/public" context.
In `src/contexts-inl.h` we have the native_context function:
```c++
Context* Context::native_context() const {
  Object* result = get(NATIVE_CONTEXT_INDEX);
  DCHECK(IsBootstrappingOrNativeContext(this->GetIsolate(), result));
  return reinterpret_cast<Context*>(result);
}
```
`Context` extends `FixedArray` so the get function is the get function of FixedArray and `NATIVE_CONTEXT_INDEX` 
is the index into the array where the native context is stored.

Now, lets take a closer look at `NewJSObject`. If you search for NewJSObject in `src/heap/factory.cc`:
```c++
Handle<JSObject> Factory::NewJSObject(Handle<JSFunction> constructor, PretenureFlag pretenure) {
  JSFunction::EnsureHasInitialMap(constructor);
  Handle<Map> map(constructor->initial_map(), isolate());
  return NewJSObjectFromMap(map, pretenure);
}
```
`NewJSObjectFromMap` 
```c++
...
  HeapObject* obj = AllocateRawWithAllocationSite(map, pretenure, allocation_site);
```
So we have created a new map

### Map
So an HeapObject contains a pointer to a Map, or rather has a function that 
returns a pointer to Map. I can't see any member map in the HeapObject class.

Lets take a look at when a map is created.
```console
(lldb) br s -f map_test.cc -l 63
```

```c++
Handle<Map> Factory::NewMap(InstanceType type,
                            int instance_size,
                            ElementsKind elements_kind,
                            int inobject_properties) {
  HeapObject* result = isolate()->heap()->AllocateRawWithRetryOrFail(Map::kSize, MAP_SPACE);
  result->set_map_after_allocation(*meta_map(), SKIP_WRITE_BARRIER);
  return handle(InitializeMap(Map::cast(result), type, instance_size,
                              elements_kind, inobject_properties),
                isolate());
}
```
We can see that the above is calling `AllocateRawWithRetryOrFail` on the heap 
instance passing a size of `88` and specifying the `MAP_SPACE`:
```c++
HeapObject* Heap::AllocateRawWithRetryOrFail(int size, AllocationSpace space,
                                             AllocationAlignment alignment) {
  AllocationResult alloc;
  HeapObject* result = AllocateRawWithLigthRetry(size, space, alignment);
  if (result) return result;

  isolate()->counters()->gc_last_resort_from_handles()->Increment();
  CollectAllAvailableGarbage(GarbageCollectionReason::kLastResort);
  {
    AlwaysAllocateScope scope(isolate());
    alloc = AllocateRaw(size, space, alignment);
  }
  if (alloc.To(&result)) {
    DCHECK(result != exception());
    return result;
  }
  // TODO(1181417): Fix this.
  FatalProcessOutOfMemory("CALL_AND_RETRY_LAST");
  return nullptr;
}
```
The default value for `alignment` is `kWordAligned`. Reading the docs in the header it says that this function
will try to perform an allocation of size `88` in the `MAP_SPACE` and if it fails a full GC will be performed
and the allocation retried.
Lets take a look at `AllocateRawWithLigthRetry`:
```c++
  AllocationResult alloc = AllocateRaw(size, space, alignment);
```
`AllocateRaw` can be found in `src/heap/heap-inl.h`. There are different paths that will be taken depending on the
`space` parameteter. Since it is `MAP_SPACE` in our case we will focus on that path:
```c++
AllocationResult Heap::AllocateRaw(int size_in_bytes, AllocationSpace space, AllocationAlignment alignment) {
  ...
  HeapObject* object = nullptr;
  AllocationResult allocation;
  if (OLD_SPACE == space) {
  ...
  } else if (MAP_SPACE == space) {
    allocation = map_space_->AllocateRawUnaligned(size_in_bytes);
  }
  ...
}
```
`map_space_` is a private member of Heap (src/heap/heap.h):
```c++
MapSpace* map_space_;
```
`AllocateRawUnaligned` can be found in `src/heap/spaces-inl.h`:
```c++
AllocationResult PagedSpace::AllocateRawUnaligned( int size_in_bytes, UpdateSkipList update_skip_list) {
  if (!EnsureLinearAllocationArea(size_in_bytes)) {
    return AllocationResult::Retry(identity());
  }

  HeapObject* object = AllocateLinearly(size_in_bytes);
  MSAN_ALLOCATED_UNINITIALIZED_MEMORY(object->address(), size_in_bytes);
  return object;
}
```
The default value for `update_skip_list` is `UPDATE_SKIP_LIST`.
So lets take a look at `AllocateLinearly`:
```c++
HeapObject* PagedSpace::AllocateLinearly(int size_in_bytes) {
  Address current_top = allocation_info_.top();
  Address new_top = current_top + size_in_bytes;
  allocation_info_.set_top(new_top);
  return HeapObject::FromAddress(current_top);
}
```
Recall that `size_in_bytes` in our case is `88`.
```console
(lldb) expr current_top
(v8::internal::Address) $5 = 24847457492680
(lldb) expr new_top
(v8::internal::Address) $6 = 24847457492768
(lldb) expr new_top - current_top
(unsigned long) $7 = 88
```
Notice that first the top is set to the new_top and then the current_top is returned and that will be a pointer
to the start of the object in memory (which in this case is of v8::internal::Map which is also of type HeapObject).
I've been wondering why Map (and other HeapObject) don't have any member fields and only/mostly 
getters/setters for the various fields that make up an object. Well the answer is that pointers to instances of
for example Map point to the first memory location of the instance. And the getters/setter functions use indexed
to read/write to memory locations. The indexes are mostly in the form of enum fields that define the memory layout
of the type.

Next, in `AllocateRawUnaligned` we have the `MSAN_ALLOCATED_UNINITIALIZED_MEMORY` macro:
```c++
  MSAN_ALLOCATED_UNINITIALIZED_MEMORY(object->address(), size_in_bytes);
```
`MSAN_ALLOCATED_UNINITIALIZED_MEMORY` can be found in `src/msan.h` and `ms` stands for `Memory Sanitizer` and 
would only be used if `V8_US_MEMORY_SANITIZER` is defined.
The returned `object` will be used to construct an `AllocationResult` when returned.
Back in `AllocateRaw` we have:
```c++
if (allocation.To(&object)) {
    ...
    OnAllocationEvent(object, size_in_bytes);
  }

  return allocation;
```
This will return us in `AllocateRawWithLightRetry`:
```c++
AllocationResult alloc = AllocateRaw(size, space, alignment);
if (alloc.To(&result)) {
  DCHECK(result != exception());
  return result;
}
```
This will return us back in `AllocateRawWithRetryOrFail`:
```c++
  HeapObject* result = AllocateRawWithLigthRetry(size, space, alignment);
  if (result) return result;
```
And that return will return to `NewMap` in `src/heap/factory.cc`:
```c++
  result->set_map_after_allocation(*meta_map(), SKIP_WRITE_BARRIER);
  return handle(InitializeMap(Map::cast(result), type, instance_size,
                              elements_kind, inobject_properties),
                isolate());
```
`InitializeMap`:
```c++
  map->set_instance_type(type);
  map->set_prototype(*null_value(), SKIP_WRITE_BARRIER);
  map->set_constructor_or_backpointer(*null_value(), SKIP_WRITE_BARRIER);
  map->set_instance_size(instance_size);
  if (map->IsJSObjectMap()) {
    DCHECK(!isolate()->heap()->InReadOnlySpace(map));
    map->SetInObjectPropertiesStartInWords(instance_size / kPointerSize - inobject_properties);
    DCHECK_EQ(map->GetInObjectProperties(), inobject_properties);
    map->set_prototype_validity_cell(*invalid_prototype_validity_cell());
  } else {
    DCHECK_EQ(inobject_properties, 0);
    map->set_inobject_properties_start_or_constructor_function_index(0);
    map->set_prototype_validity_cell(Smi::FromInt(Map::kPrototypeChainValid));
  }
  map->set_dependent_code(DependentCode::cast(*empty_fixed_array()), SKIP_WRITE_BARRIER);
  map->set_weak_cell_cache(Smi::kZero);
  map->set_raw_transitions(MaybeObject::FromSmi(Smi::kZero));
  map->SetInObjectUnusedPropertyFields(inobject_properties);
  map->set_instance_descriptors(*empty_descriptor_array());

  map->set_visitor_id(Map::GetVisitorId(map));
  map->set_bit_field(0);
  map->set_bit_field2(Map::IsExtensibleBit::kMask);
  int bit_field3 = Map::EnumLengthBits::encode(kInvalidEnumCacheSentinel) |
                   Map::OwnsDescriptorsBit::encode(true) |
                   Map::ConstructionCounterBits::encode(Map::kNoSlackTracking);
  map->set_bit_field3(bit_field3);
  map->set_elements_kind(elements_kind); //HOLEY_ELEMENTS
  map->set_new_target_is_base(true);
  isolate()->counters()->maps_created()->Increment();
  if (FLAG_trace_maps) LOG(isolate(), MapCreate(map));
  return map;
```

Creating a new map ([map_test.cc](./test/map_test.cc):
```c++
  i::Handle<i::Map> map = i::Map::Create(asInternal(isolate_), 10);
  std::cout << map->instance_type() << '\n';
```
`Map::Create` can be found in objects.cc:
```c++
Handle<Map> Map::Create(Isolate* isolate, int inobject_properties) {
  Handle<Map> copy = Copy(handle(isolate->object_function()->initial_map()), "MapCreate");
```
So, the first thing that will happen is `isolate->object_function()` will be called. This is function
that is generated by the preprocessor.

```c++
// from src/context.h
#define NATIVE_CONTEXT_FIELDS(V)                                               \
  ...                                                                          \
  V(OBJECT_FUNCTION_INDEX, JSFunction, object_function)                        \

// from src/isolate.h
#define NATIVE_CONTEXT_FIELD_ACCESSOR(index, type, name)     \
  Handle<type> Isolate::name() {                             \
    return Handle<type>(raw_native_context()->name(), this); \
  }                                                          \
  bool Isolate::is_##name(type* value) {                     \
    return raw_native_context()->is_##name(value);           \
  }
NATIVE_CONTEXT_FIELDS(NATIVE_CONTEXT_FIELD_ACCESSOR)
```
`object_function()` will become:
```c++
  Handle<JSFunction> Isolate::object_function() {
    return Handle<JSFunction>(raw_native_context()->object_function(), this);
  }
```

Lets look closer at `JSFunction::initial_map()` in in object-inl.h:
```c++
Map* JSFunction::initial_map() {
  return Map::cast(prototype_or_initial_map());
}
```

`prototype_or_initial_map` is generated by a macro:
```c++
ACCESSORS_CHECKED(JSFunction, prototype_or_initial_map, Object,
                  kPrototypeOrInitialMapOffset, map()->has_prototype_slot())
```
`ACCESSORS_CHECKED` can be found in `src/objects/object-macros.h`:
```c++
#define ACCESSORS_CHECKED(holder, name, type, offset, condition) \
  ACCESSORS_CHECKED2(holder, name, type, offset, condition, condition)

#define ACCESSORS_CHECKED2(holder, name, type, offset, get_condition, \
                           set_condition)                             \
  type* holder::name() const {                                        \
    type* value = type::cast(READ_FIELD(this, offset));               \
    DCHECK(get_condition);                                            \
    return value;                                                     \
  }                                                                   \
  void holder::set_##name(type* value, WriteBarrierMode mode) {       \
    DCHECK(set_condition);                                            \
    WRITE_FIELD(this, offset, value);                                 \
    CONDITIONAL_WRITE_BARRIER(GetHeap(), this, offset, value, mode);  \
  }

#define FIELD_ADDR(p, offset) \
  (reinterpret_cast<Address>(p) + offset - kHeapObjectTag)

#define READ_FIELD(p, offset) \
  (*reinterpret_cast<Object* const*>(FIELD_ADDR(p, offset)))
```
The preprocessor will expand `prototype_or_initial_map` to:
```c++
  JSFunction* JSFunction::prototype_or_initial_map() const {
    JSFunction* value = JSFunction::cast(
        (*reinterpret_cast<Object* const*>(
            (reinterpret_cast<Address>(this) + kPrototypeOrInitialMapOffset - kHeapObjectTag))))
    DCHECK(map()->has_prototype_slot());
    return value;
  }
```
Notice that `map()->has_prototype_slot())` will be called first which looks like this:
```c++
Map* HeapObject::map() const {
  return map_word().ToMap();
}
```
__TODO: Add notes about MapWord__  
```c++
MapWord HeapObject::map_word() const {
  return MapWord(
      reinterpret_cast<uintptr_t>(RELAXED_READ_FIELD(this, kMapOffset)));
}
```
First thing that will happen is `RELAXED_READ_FIELD(this, kMapOffset)`
```c++
#define RELAXED_READ_FIELD(p, offset)           \
  reinterpret_cast<Object*>(base::Relaxed_Load( \
      reinterpret_cast<const base::AtomicWord*>(FIELD_ADDR(p, offset))))

#define FIELD_ADDR(p, offset) \
  (reinterpret_cast<Address>(p) + offset - kHeapObjectTag)
```
This will get expanded by the preprocessor to:
```c++
  reinterpret_cast<Object*>(base::Relaxed_Load(
      reinterpret_cast<const base::AtomicWord*>(
          (reinterpret_cast<Address>(this) + kMapOffset - kHeapObjectTag)))
```
`src/base/atomicops_internals_portable.h`:
```c++
inline Atomic8 Relaxed_Load(volatile const Atomic8* ptr) {
  return __atomic_load_n(ptr, __ATOMIC_RELAXED);
}
```
So this will do an atomoic load of the ptr with the memory order of __ATOMIC_RELELAXED.


`ACCESSORS_CHECKED` also generates a `set_prototyp_or_initial_map`:
```c++
  void JSFunction::set_prototype_or_initial_map(JSFunction* value, WriteBarrierMode mode) {
    DCHECK(map()->has_prototype_slot());
    WRITE_FIELD(this, kPrototypeOrInitialMapOffset, value);
    CONDITIONAL_WRITE_BARRIER(GetHeap(), this, kPrototypeOrInitialMapOffset, value, mode);
  }
```
What does `WRITE_FIELD` do?  
```c++
#define WRITE_FIELD(p, offset, value)                             \
  base::Relaxed_Store(                                            \
      reinterpret_cast<base::AtomicWord*>(FIELD_ADDR(p, offset)), \
      reinterpret_cast<base::AtomicWord>(value));
```
Which would expand into:
```c++
  base::Relaxed_Store(                                            \
      reinterpret_cast<base::AtomicWord*>(
          (reinterpret_cast<Address>(this) + kPrototypeOrInitialMapOffset - kHeapObjectTag)
      reinterpret_cast<base::AtomicWord>(value));
```

Lets take a look at what `instance_type` does:
```c++
InstanceType Map::instance_type() const {
  return static_cast<InstanceType>(READ_UINT16_FIELD(this, kInstanceTypeOffset));
}
```

To see what the above is doing we can do the same thing in the debugger:
Note that I got `11` below from `map->kInstanceTypeOffset - i::kHeapObjectTag`
```console
(lldb) memory read -f u -c 1 -s 8 `*map + 11`
0x6d4e6609ed4: 585472345729139745
(lldb) expr static_cast<InstanceType>(585472345729139745)
(v8::internal::InstanceType) $34 = JS_OBJECT_TYPE
```

Take `map->has_non_instance_prototype()`:
```console
(lldb) br s -n has_non_instance_prototype
(lldb) expr -i 0 -- map->has_non_instance_prototype()
```
The above command will break in `src/objects/map-inl.h`:
```c++
BIT_FIELD_ACCESSORS(Map, bit_field, has_non_instance_prototype, Map::HasNonInstancePrototypeBit)

// src/objects/object-macros.h
#define BIT_FIELD_ACCESSORS(holder, field, name, BitField)      \
  typename BitField::FieldType holder::name() const {           \
    return BitField::decode(field());                           \
  }                                                             \
  void holder::set_##name(typename BitField::FieldType value) { \
    set_##field(BitField::update(field(), value));              \
  }
```
The preprocessor will expand that to:
```c++
  typename Map::HasNonInstancePrototypeBit::FieldType Map::has_non_instance_prototype() const {
    return Map::HasNonInstancePrototypeBit::decode(bit_field());
  }                                                             \
  void holder::set_has_non_instance_prototype(typename BitField::FieldType value) { \
    set_bit_field(Map::HasNonInstancePrototypeBit::update(bit_field(), value));              \
  }
```
So where can we find `Map::HasNonInstancePrototypeBit`?  
It is generated by a macro in `src/objects/map.h`:
```c++
// Bit positions for |bit_field|.
#define MAP_BIT_FIELD_FIELDS(V, _)          \
  V(HasNonInstancePrototypeBit, bool, 1, _) \
  ...
  DEFINE_BIT_FIELDS(MAP_BIT_FIELD_FIELDS)
#undef MAP_BIT_FIELD_FIELDS

#define DEFINE_BIT_FIELDS(LIST_MACRO) \
  DEFINE_BIT_RANGES(LIST_MACRO)       \
  LIST_MACRO(DEFINE_BIT_FIELD_TYPE, LIST_MACRO##_Ranges)

#define DEFINE_BIT_RANGES(LIST_MACRO)                               \
  struct LIST_MACRO##_Ranges {                                      \
    enum { LIST_MACRO(DEFINE_BIT_FIELD_RANGE_TYPE, _) kBitsCount }; \
  };

#define DEFINE_BIT_FIELD_RANGE_TYPE(Name, Type, Size, _) \
  k##Name##Start, k##Name##End = k##Name##Start + Size - 1,
```
Alright, lets see what preprocessor expands that to:
```c++
  struct MAP_BIT_FIELD_FIELDS_Ranges {
    enum { 
      kHasNonInstancePrototypeBitStart, 
      kHasNonInstancePrototypeBitEnd = kHasNonInstancePrototypeBitStart + 1 - 1,
      ... // not showing the rest of the entries.
      kBitsCount 
    };
  };

```
So this would create a struct with an enum and it could be accessed using:
`i::Map::MAP_BIT_FIELD_FIELDS_Ranges::kHasNonInstancePrototypeBitStart`
The next part of the macro is
```c++
  LIST_MACRO(DEFINE_BIT_FIELD_TYPE, LIST_MACRO##_Ranges)

#define DEFINE_BIT_FIELD_TYPE(Name, Type, Size, RangesName) \
  typedef BitField<Type, RangesName::k##Name##Start, Size> Name;
```
Which will get expanded to:
```c++
  typedef BitField<HasNonInstancePrototypeBit, MAP_BIT_FIELD_FIELDS_Ranges::kHasNonInstancePrototypeBitStart, 1> HasNonInstancePrototypeBit;
```
So this is how `HasNonInstancePrototypeBit` is declared and notice that it is of type `BitField` which can be
found in `src/utils.h`:
```c++
template<class T, int shift, int size>
class BitField : public BitFieldBase<T, shift, size, uint32_t> { };

template<class T, int shift, int size, class U>
class BitFieldBase {
 public:
  typedef T FieldType;
```

Map::HasNonInstancePrototypeBit::decode(bit_field());
first bit_field is called:
```c++
byte Map::bit_field() const { return READ_BYTE_FIELD(this, kBitFieldOffset); }

```
And the result of that is passed to `Map::HasNonInstancePrototypeBit::decode`:

```console
(lldb) br s -n bit_field
(lldb) expr -i 0 --  map->bit_field()
```

```c++
byte Map::bit_field() const { return READ_BYTE_FIELD(this, kBitFieldOffset); }
```
So, `this` is the current Map instance, and we are going to read from.
```c++
#define READ_BYTE_FIELD(p, offset) \
  (*reinterpret_cast<const byte*>(FIELD_ADDR(p, offset)))

#define FIELD_ADDR(p, offset) \
  (reinterpret_cast<Address>(p) + offset - kHeapObjectTag)
```
Which will get expanded to:
```c++
byte Map::bit_field() const { 
  return *reinterpret_cast<const byte*>(
      reinterpret_cast<Address>(this) + kBitFieldOffset - kHeapObjectTag)
}
```

The instance_size is the instance_size_in_words << kPointerSizeLog2 (3 on my machine):
```console
(lldb) memory read -f x -s 1 -c 1 *map+8
0x24d1cd509ed1: 0x03
(lldb) expr 0x03 << 3
(int) $2 = 24
(lldb) expr map->instance_size()
(int) $3 = 24
```
`i::HeapObject::kHeaderSize` is 8 on my system  which is used in the `DEFINE_FIELD_OFFSET_CONSTANTS:
```c++
#define MAP_FIELDS(V)
V(kInstanceSizeInWordsOffset, kUInt8Size)
V(kInObjectPropertiesStartOrConstructorFunctionIndexOffset, kUInt8Size)
...
DEFINE_FIELD_OFFSET_CONSTANTS(HeapObject::kHeaderSize, MAP_FIELDS)
```

So we can use this information to read the `inobject_properties_start_or_constructor_function_index` directly from memory using:
```console
(lldb) expr map->inobject_properties_start_or_constructor_function_index()
(lldb) memory read -f x -s 1 -c 1 map+9
error: invalid start address expression.
error: address expression "map+9" evaluation failed
(lldb) memory read -f x -s 1 -c 1 *map+9
0x17b027209ed2: 0x03
```

Inspect the visitor_id (which is the last of the first byte):
```console
lldb) memory read -f x -s 1 -c 1 *map+10
0x17b027209ed3: 0x15
(lldb) expr (int) 0x15
(int) $8 = 21
(lldb) expr map->visitor_id()
(v8::internal::VisitorId) $11 = kVisitJSObjectFast
(lldb) expr (int) $11
(int) $12 = 21
```

Inspect the instance_type (which is part of the second byte):
```console
(lldb) expr map->instance_type()
(v8::internal::InstanceType) $41 = JS_OBJECT_TYPE
(lldb) expr v8::internal::InstanceType::JS_OBJECT_TYPE
(uint16_t) $35 = 1057
(lldb) memory read -f x -s 2 -c 1 *map+11
0x17b027209ed4: 0x0421
(lldb) expr (int)0x0421
(int) $40 = 1057
```
Notice that `instance_type` is a short so that will take up 2 bytes
```console
(lldb) expr map->has_non_instance_prototype()
(bool) $60 = false
(lldb) expr map->is_callable()
(bool) $46 = false
(lldb) expr map->has_named_interceptor()
(bool) $51 = false
(lldb) expr map->has_indexed_interceptor()
(bool) $55 = false
(lldb) expr map->is_undetectable()
(bool) $56 = false
(lldb) expr map->is_access_check_needed()
(bool) $57 = false
(lldb) expr map->is_constructor()
(bool) $58 = false
(lldb) expr map->has_prototype_slot()
(bool) $59 = false
```

Verify that the above is correct:
```console
(lldb) expr map->has_non_instance_prototype()
(bool) $44 = false
(lldb) memory read -f x -s 1 -c 1 *map+13
0x17b027209ed6: 0x00

(lldb) expr map->set_has_non_instance_prototype(true)
(lldb) memory read -f x -s 1 -c 1 *map+13
0x17b027209ed6: 0x01

(lldb) expr map->set_has_prototype_slot(true)
(lldb) memory read -f x -s 1 -c 1 *map+13
0x17b027209ed6: 0x81
```


Inspect second int field (bit_field2):
```console
(lldb) memory read -f x -s 1 -c 1 *map+14
0x17b027209ed7: 0x19
(lldb) expr map->is_extensible()
(bool) $78 = true
(lldb) expr -- 0x19 & (1 << 0)
(bool) $90 = 1

(lldb) expr map->is_prototype_map()
(bool) $79 = false

(lldb) expr map->is_in_retained_map_list()
(bool) $80 = false

(lldb) expr map->elements_kind()
(v8::internal::ElementsKind) $81 = HOLEY_ELEMENTS
(lldb) expr v8::internal::ElementsKind::HOLEY_ELEMENTS
(int) $133 = 3
(lldb) expr  0x19 >> 3
(int) $134 = 3
```

Inspect third int field (bit_field3):
```console
(lldb) memory read -f b -s 4 -c 1 *map+15
0x17b027209ed8: 0b00001000001000000000001111111111
(lldb) memory read -f x -s 4 -c 1 *map+15
0x17b027209ed8: 0x082003ff
```

So we know that a Map instance is a pointer allocated by the Heap and with a specific 
size. Fields are accessed using indexes (remember there are no member fields in the Map class).
We also know that all HeapObject have a Map. The Map is sometimes referred to as the HiddenClass
and somethime the shape of an object. If two objects have the same properties they would share 
the same Map. This makes sense and I've see blog post that show this but I'd like to verify
this to fully understand it.
I'm going to try to match https://v8project.blogspot.com/2017/08/fast-properties.html with 
the code.

So, lets take a look at adding a property to a JSObject. We start by creating a new Map and then 
use it to create a new JSObject:
```c++
  i::Handle<i::Map> map = factory->NewMap(i::JS_OBJECT_TYPE, 32);
  i::Handle<i::JSObject> js_object = factory->NewJSObjectFromMap(map);

  i::Handle<i::String> prop_name = factory->InternalizeUtf8String("prop_name");
  i::Handle<i::String> prop_value = factory->InternalizeUtf8String("prop_value");
  i::JSObject::AddProperty(js_object, prop_name, prop_value, i::NONE);  
```
Lets take a closer look at `AddProperty` and how it interacts with the Map. This function can be
found in `src/objects.cc`:
```c++
void JSObject::AddProperty(Handle<JSObject> object, Handle<Name> name,
                           Handle<Object> value,
                           PropertyAttributes attributes) {
  LookupIterator it(object, name, object, LookupIterator::OWN_SKIP_INTERCEPTOR);
  CHECK_NE(LookupIterator::ACCESS_CHECK, it.state());
```
First we have the LookupIterator constructor (`src/lookup.h`) but since this is a new property which 
we know does not exist it will not find any property.
```c++
CHECK(AddDataProperty(&it, value, attributes, kThrowOnError,
                        CERTAINLY_NOT_STORE_FROM_KEYED)
            .IsJust());
```
```c++
  Handle<JSReceiver> receiver = it->GetStoreTarget<JSReceiver>();
  ...
  it->UpdateProtector();
  // Migrate to the most up-to-date map that will be able to store |value|
  // under it->name() with |attributes|.
  it->PrepareTransitionToDataProperty(receiver, value, attributes, store_mode);
  DCHECK_EQ(LookupIterator::TRANSITION, it->state());
  it->ApplyTransitionToDataProperty(receiver);

  // Write the property value.
  it->WriteDataValue(value, true);
```
`PrepareTransitionToDataProperty`:
```c++
  Representation representation = value->OptimalRepresentation();
  Handle<FieldType> type = value->OptimalType(isolate, representation);
  maybe_map = Map::CopyWithField(map, name, type, attributes, constness,
  representation, flag);
```
`Map::CopyWithField`:
```c++
  Descriptor d = Descriptor::DataField(name, index, attributes, constness, representation, wrapped_type);
```
Lets take a closer look the Decriptor which can be found in `src/property.cc`:
```c++
Descriptor Descriptor::DataField(Handle<Name> key, int field_index,
                                 PropertyAttributes attributes,
                                 PropertyConstness constness,
                                 Representation representation,
                                 MaybeObjectHandle wrapped_field_type) {
  DCHECK(wrapped_field_type->IsSmi() || wrapped_field_type->IsWeakHeapObject());
  PropertyDetails details(kData, attributes, kField, constness, representation,
                          field_index);
  return Descriptor(key, wrapped_field_type, details);
}
```
`Descriptor` is declared in `src/property.h` and describes the elements in a instance-descriptor array. These
are returned when calling `map->instance_descriptors()`. Let check some of the arguments:
```console
(lldb) job *key
#prop_name
(lldb) expr attributes
(v8::internal::PropertyAttributes) $27 = NONE
(lldb) expr constness
(v8::internal::PropertyConstness) $28 = kMutable
(lldb) expr representation
(v8::internal::Representation) $29 = (kind_ = '\b')
```
The Descriptor class contains three members:
```c++
 private:
  Handle<Name> key_;
  MaybeObjectHandle value_;
  PropertyDetails details_;
```
Lets take a closer look `PropertyDetails` which only has a single member named `value_`
```c++
  uint32_t value_;
```
It also declares a number of classes the extend BitField, for example:
```c++
class KindField : public BitField<PropertyKind, 0, 1> {};
class LocationField : public BitField<PropertyLocation, KindField::kNext, 1> {};
class ConstnessField : public BitField<PropertyConstness, LocationField::kNext, 1> {};
class AttributesField : public BitField<PropertyAttributes, ConstnessField::kNext, 3> {};
class PropertyCellTypeField : public BitField<PropertyCellType, AttributesField::kNext, 2> {};
class DictionaryStorageField : public BitField<uint32_t, PropertyCellTypeField::kNext, 23> {};

// Bit fields for fast objects.
class RepresentationField : public BitField<uint32_t, AttributesField::kNext, 4> {};
class DescriptorPointer : public BitField<uint32_t, RepresentationField::kNext, kDescriptorIndexBitCount> {};
class FieldIndexField : public BitField<uint32_t, DescriptorPointer::kNext, kDescriptorIndexBitCount> {

enum PropertyKind { kData = 0, kAccessor = 1 };
enum PropertyLocation { kField = 0, kDescriptor = 1 };
enum class PropertyConstness { kMutable = 0, kConst = 1 };
enum PropertyAttributes {
  NONE = ::v8::None,
  READ_ONLY = ::v8::ReadOnly,
  DONT_ENUM = ::v8::DontEnum,
  DONT_DELETE = ::v8::DontDelete,
  ALL_ATTRIBUTES_MASK = READ_ONLY | DONT_ENUM | DONT_DELETE,
  SEALED = DONT_DELETE,
  FROZEN = SEALED | READ_ONLY,
  ABSENT = 64,  // Used in runtime to indicate a property is absent.
  // ABSENT can never be stored in or returned from a descriptor's attributes
  // bitfield.  It is only used as a return value meaning the attributes of
  // a non-existent property.
};
enum class PropertyCellType {
  // Meaningful when a property cell does not contain the hole.
  kUndefined,     // The PREMONOMORPHIC of property cells.
  kConstant,      // Cell has been assigned only once.
  kConstantType,  // Cell has been assigned only one type.
  kMutable,       // Cell will no longer be tracked as constant.
  // Meaningful when a property cell contains the hole.
  kUninitialized = kUndefined,  // Cell has never been initialized.
  kInvalidated = kConstant,     // Cell has been deleted, invalidated or never
                                // existed.
  // For dictionaries not holding cells.
  kNoCell = kMutable,
};


template<class T, int shift, int size>
class BitField : public BitFieldBase<T, shift, size, uint32_t> { };
```
The Type T of KindField will be `PropertyKind`, the `shift` will be 0 , and the `size` 1.
Notice that `LocationField` is using `KindField::kNext` as its shift. This is a static class constant
of type `uint32_t` and is defined as:
```c++
static const U kNext = kShift + kSize;
```
So `LocationField` would get the value from KindField which should be:
```c++
class LocationField : public BitField<PropertyLocation, 1, 1> {};
```

The constructor for PropertyDetails looks like this:
```c++
PropertyDetails(PropertyKind kind, PropertyAttributes attributes, PropertyCellType cell_type, int dictionary_index = 0) {
    value_ = KindField::encode(kind) | LocationField::encode(kField) |
             AttributesField::encode(attributes) |
             DictionaryStorageField::encode(dictionary_index) |
             PropertyCellTypeField::encode(cell_type);
  }
```
So what does KindField::encode(kind) actualy do then?
```console
(lldb) expr static_cast<uint32_t>(kind())
(uint32_t) $36 = 0
(lldb) expr static_cast<uint32_t>(kind()) << 0
(uint32_t) $37 = 0
```
This value is later returned by calling `kind()`:
```c++
PropertyKind kind() const { return KindField::decode(value_); }
```
So we have all this information about this property, its type (Representation), constness, if it is 
read-only, enumerable, deletable, sealed, frozen. After that little detour we are back in `Descriptor::DataField`:
```c++
  return Descriptor(key, wrapped_field_type, details);
```
Here we are using the key (name of the property), the wrapped_field_type, and PropertyDetails we created.
What is `wrapped_field_type` again?  
If we back up a few frames back into `Map::TransitionToDataProperty` we can see that the type passed in
is taken from the following code:
```c++
  Representation representation = value->OptimalRepresentation();
  Handle<FieldType> type = value->OptimalType(isolate, representation);
```
So this is only taking the type of the field:
```console
(lldb) expr representation.kind()
(v8::internal::Representation::Kind) $51 = kHeapObject
```
This makes sense as the map only deals with the shape of the propery and not the value.
Next in `Map::CopyWithField` we have:
```c++
  Handle<Map> new_map = Map::CopyAddDescriptor(map, &d, flag);
```
`CopyAddDescriptor` does:
```c++
  Handle<DescriptorArray> descriptors(map->instance_descriptors());
 
  int nof = map->NumberOfOwnDescriptors();
  Handle<DescriptorArray> new_descriptors = DescriptorArray::CopyUpTo(descriptors, nof, 1);
  new_descriptors->Append(descriptor);
  
  Handle<LayoutDescriptor> new_layout_descriptor =
      FLAG_unbox_double_fields
          ? LayoutDescriptor::New(map, new_descriptors, nof + 1)
          : handle(LayoutDescriptor::FastPointerLayout(), map->GetIsolate());

  return CopyReplaceDescriptors(map, new_descriptors, new_layout_descriptor,
                                flag, descriptor->GetKey(), "CopyAddDescriptor",
                                SIMPLE_PROPERTY_TRANSITION);
```
Lets take a closer look at `LayoutDescriptor`

```console
(lldb) expr new_layout_descriptor->Print()
Layout descriptor: <all tagged>
```
TODO: Take a closer look at LayoutDescritpor

Later when actually adding the value in `Object::AddDataProperty`:
```c++
  it->WriteDataValue(value, true);
```
This call will end up in `src/lookup.cc` and in our case the path will be the following call:
```c++
  JSObject::cast(*holder)->WriteToField(descriptor_number(), property_details_, *value);
```
TODO: Take a closer look at LookupIterator.
`WriteToField` can be found in `src/objects-inl.h`:
```c++
  FieldIndex index = FieldIndex::ForDescriptor(map(), descriptor);
```
`FieldIndex::ForDescriptor` can be found in `src/field-index-inl.h`:
```c++
inline FieldIndex FieldIndex::ForDescriptor(const Map* map, int descriptor_index) {
  PropertyDetails details = map->instance_descriptors()->GetDetails(descriptor_index);
  int field_index = details.field_index();
  return ForPropertyIndex(map, field_index, details.representation());
}
```
Notice that this is calling `instance_descriptors()` on the passed-in map. This as we recall from earlier returns
and DescriptorArray (which is a type of WeakFixedArray). A Descriptor array 

Our DecsriptorArray only has one entry:
```console
(lldb) expr map->instance_descriptors()->number_of_descriptors()
(int) $6 = 1
(lldb) expr map->instance_descriptors()->GetKey(0)->Print()
#prop_name
(lldb) expr map->instance_descriptors()->GetFieldIndex(0)
(int) $11 = 0
```
We can also use `Print` on the DescriptorArray:
```console
lldb) expr map->instance_descriptors()->Print()

  [0]: #prop_name (data field 0:h, p: 0, attrs: [WEC]) @ Any
```
In our case we are accessing the PropertyDetails and then getting the `field_index` which I think tells us
where in the object the value for this property is stored.
The last call in `ForDescriptor` is `ForProperty:
```c++
inline FieldIndex FieldIndex::ForPropertyIndex(const Map* map,
                                               int property_index,
                                               Representation representation) {
  int inobject_properties = map->GetInObjectProperties();
  bool is_inobject = property_index < inobject_properties;
  int first_inobject_offset;
  int offset;
  if (is_inobject) {
    first_inobject_offset = map->GetInObjectPropertyOffset(0);
    offset = map->GetInObjectPropertyOffset(property_index);
  } else {
    first_inobject_offset = FixedArray::kHeaderSize;
    property_index -= inobject_properties;
    offset = FixedArray::kHeaderSize + property_index * kPointerSize;
  }
  Encoding encoding = FieldEncoding(representation);
  return FieldIndex(is_inobject, offset, encoding, inobject_properties,
                    first_inobject_offset);
}
```
I was expecting `inobject_propertis` to be 1 here but it is 0:
```console
(lldb) expr inobject_properties
(int) $14 = 0
```
Why is that, what am I missing?  
These in-object properties are stored directly on the object instance and not do not use
the properties array. All get back to an example of this later to clarify this.
TODO: Add in-object properties example.

Back in `JSObject::WriteToField`:
```c++
  RawFastPropertyAtPut(index, value);
```

```c++
void JSObject::RawFastPropertyAtPut(FieldIndex index, Object* value) {
  if (index.is_inobject()) {
    int offset = index.offset();
    WRITE_FIELD(this, offset, value);
    WRITE_BARRIER(GetHeap(), this, offset, value);
  } else {
    property_array()->set(index.outobject_array_index(), value);
  }
}
```
In our case we know that the index is not inobject()
```console
(lldb) expr index.is_inobject()
(bool) $18 = false
```
So, `property_array()->set()` will be called.
```console
(lldb) expr this
(v8::internal::JSObject *) $21 = 0x00002c31c6a88b59
```
JSObject inherits from JSReceiver which is where the property_array() function is declared.
```c++
  inline PropertyArray* property_array() const;
```
```console
(lldb) expr property_array()->Print()
0x2c31c6a88bb1: [PropertyArray]
 - map: 0x2c31f5603e21 <Map>
 - length: 3
 - hash: 0
           0: 0x2c31f56025a1 <Odd Oddball: uninitialized>
         1-2: 0x2c31f56026f1 <undefined>
(lldb) expr index.outobject_array_index()
(int) $26 = 0
(lldb) expr value->Print()
#prop_value
```
Looking at the above values printed we should see the property be written to entry 0.
```console
(lldb) expr property_array()->get(0)->Print()
#uninitialized
// after call to set
(lldb) expr property_array()->get(0)->Print()
#prop_value
```

```console
(lldb) expr map->instance_descriptors()
(v8::internal::DescriptorArray *) $4 = 0x000039a927082339
```
So a map has an pointer array of instance of DescriptorArray

```console
(lldb) expr map->GetInObjectProperties()
(int) $19 = 1
```
Each Map has int that tells us the number of properties it has. This is the number specified when creating
a new Map, for example:
```console
i::Handle<i::Map> map = i::Map::Create(asInternal(isolate_), 1);
```
But at this stage we don't really have any properties. The value for a property is associated with the actual
instance of the Object. What the Map specifies is index of the value for a particualar property. 

#### Creating a Map instance
Lets take a look at when a map is created.
```console
(lldb) br s -f map_test.cc -l 63
```

```c++
Handle<Map> Factory::NewMap(InstanceType type,
                            int instance_size,
                            ElementsKind elements_kind,
                            int inobject_properties) {
  HeapObject* result = isolate()->heap()->AllocateRawWithRetryOrFail(Map::kSize, MAP_SPACE);
  result->set_map_after_allocation(*meta_map(), SKIP_WRITE_BARRIER);
  return handle(InitializeMap(Map::cast(result), type, instance_size,
                              elements_kind, inobject_properties),
                isolate());
}
```
We can see that the above is calling `AllocateRawWithRetryOrFail` on the heap instance passing a size of `88` and
specifying the `MAP_SPACE`:
```c++
HeapObject* Heap::AllocateRawWithRetryOrFail(int size, AllocationSpace space,
                                             AllocationAlignment alignment) {
  AllocationResult alloc;
  HeapObject* result = AllocateRawWithLigthRetry(size, space, alignment);
  if (result) return result;

  isolate()->counters()->gc_last_resort_from_handles()->Increment();
  CollectAllAvailableGarbage(GarbageCollectionReason::kLastResort);
  {
    AlwaysAllocateScope scope(isolate());
    alloc = AllocateRaw(size, space, alignment);
  }
  if (alloc.To(&result)) {
    DCHECK(result != exception());
    return result;
  }
  // TODO(1181417): Fix this.
  FatalProcessOutOfMemory("CALL_AND_RETRY_LAST");
  return nullptr;
}
```
The default value for `alignment` is `kWordAligned`. Reading the docs in the header it says that this function
will try to perform an allocation of size `88` in the `MAP_SPACE` and if it fails a full GC will be performed
and the allocation retried.
Lets take a look at `AllocateRawWithLigthRetry`:
```c++
  AllocationResult alloc = AllocateRaw(size, space, alignment);
```
`AllocateRaw` can be found in `src/heap/heap-inl.h`. There are different paths that will be taken depending on the
`space` parameteter. Since it is `MAP_SPACE` in our case we will focus on that path:
```c++
AllocationResult Heap::AllocateRaw(int size_in_bytes, AllocationSpace space, AllocationAlignment alignment) {
  ...
  HeapObject* object = nullptr;
  AllocationResult allocation;
  if (OLD_SPACE == space) {
  ...
  } else if (MAP_SPACE == space) {
    allocation = map_space_->AllocateRawUnaligned(size_in_bytes);
  }
  ...
}
```
`map_space_` is a private member of Heap (src/heap/heap.h):
```c++
MapSpace* map_space_;
```
`AllocateRawUnaligned` can be found in `src/heap/spaces-inl.h`:
```c++
AllocationResult PagedSpace::AllocateRawUnaligned( int size_in_bytes, UpdateSkipList update_skip_list) {
  if (!EnsureLinearAllocationArea(size_in_bytes)) {
    return AllocationResult::Retry(identity());
  }

  HeapObject* object = AllocateLinearly(size_in_bytes);
  MSAN_ALLOCATED_UNINITIALIZED_MEMORY(object->address(), size_in_bytes);
  return object;
}
```
The default value for `update_skip_list` is `UPDATE_SKIP_LIST`.
So lets take a look at `AllocateLinearly`:
```c++
HeapObject* PagedSpace::AllocateLinearly(int size_in_bytes) {
  Address current_top = allocation_info_.top();
  Address new_top = current_top + size_in_bytes;
  allocation_info_.set_top(new_top);
  return HeapObject::FromAddress(current_top);
}
```
Recall that `size_in_bytes` in our case is `88`.
```console
(lldb) expr current_top
(v8::internal::Address) $5 = 24847457492680
(lldb) expr new_top
(v8::internal::Address) $6 = 24847457492768
(lldb) expr new_top - current_top
(unsigned long) $7 = 88
```
Notice that first the top is set to the new_top and then the current_top is returned and that will be a pointer
to the start of the object in memory (which in this case is of v8::internal::Map which is also of type HeapObject).
I've been wondering why Map (and other HeapObject) don't have any member fields and only/mostly 
getters/setters for the various fields that make up an object. Well the answer is that pointers to instances of
for example Map point to the first memory location of the instance. And the getters/setter functions use indexed
to read/write to memory locations. The indexes are mostly in the form of enum fields that define the memory layout
of the type.

Next, in `AllocateRawUnaligned` we have the `MSAN_ALLOCATED_UNINITIALIZED_MEMORY` macro:
```c++
  MSAN_ALLOCATED_UNINITIALIZED_MEMORY(object->address(), size_in_bytes);
```
`MSAN_ALLOCATED_UNINITIALIZED_MEMORY` can be found in `src/msan.h` and `ms` stands for `Memory Sanitizer` and 
would only be used if `V8_US_MEMORY_SANITIZER` is defined.
The returned `object` will be used to construct an `AllocationResult` when returned.
Back in `AllocateRaw` we have:
```c++
if (allocation.To(&object)) {
    ...
    OnAllocationEvent(object, size_in_bytes);
  }

  return allocation;
```
This will return us in `AllocateRawWithLightRetry`:
```c++
AllocationResult alloc = AllocateRaw(size, space, alignment);
if (alloc.To(&result)) {
  DCHECK(result != exception());
  return result;
}
```
This will return us back in `AllocateRawWithRetryOrFail`:
```c++
  HeapObject* result = AllocateRawWithLigthRetry(size, space, alignment);
  if (result) return result;
```
And that return will return to `NewMap` in `src/heap/factory.cc`:
```c++
  result->set_map_after_allocation(*meta_map(), SKIP_WRITE_BARRIER);
  return handle(InitializeMap(Map::cast(result), type, instance_size,
                              elements_kind, inobject_properties),
                isolate());
```
`InitializeMap`:
```c++
  map->set_instance_type(type);
  map->set_prototype(*null_value(), SKIP_WRITE_BARRIER);
  map->set_constructor_or_backpointer(*null_value(), SKIP_WRITE_BARRIER);
  map->set_instance_size(instance_size);
  if (map->IsJSObjectMap()) {
    DCHECK(!isolate()->heap()->InReadOnlySpace(map));
    map->SetInObjectPropertiesStartInWords(instance_size / kPointerSize - inobject_properties);
    DCHECK_EQ(map->GetInObjectProperties(), inobject_properties);
    map->set_prototype_validity_cell(*invalid_prototype_validity_cell());
  } else {
    DCHECK_EQ(inobject_properties, 0);
    map->set_inobject_properties_start_or_constructor_function_index(0);
    map->set_prototype_validity_cell(Smi::FromInt(Map::kPrototypeChainValid));
  }
  map->set_dependent_code(DependentCode::cast(*empty_fixed_array()), SKIP_WRITE_BARRIER);
  map->set_weak_cell_cache(Smi::kZero);
  map->set_raw_transitions(MaybeObject::FromSmi(Smi::kZero));
  map->SetInObjectUnusedPropertyFields(inobject_properties);
  map->set_instance_descriptors(*empty_descriptor_array());

  map->set_visitor_id(Map::GetVisitorId(map));
  map->set_bit_field(0);
  map->set_bit_field2(Map::IsExtensibleBit::kMask);
  int bit_field3 = Map::EnumLengthBits::encode(kInvalidEnumCacheSentinel) |
                   Map::OwnsDescriptorsBit::encode(true) |
                   Map::ConstructionCounterBits::encode(Map::kNoSlackTracking);
  map->set_bit_field3(bit_field3);
  map->set_elements_kind(elements_kind); //HOLEY_ELEMENTS
  map->set_new_target_is_base(true);
  isolate()->counters()->maps_created()->Increment();
  if (FLAG_trace_maps) LOG(isolate(), MapCreate(map));
  return map;
```

### Context
Context extends `FixedArray` (`src/context.h`). So an instance of this Context is a FixedArray and we can 
use Get(index) etc to get entries in the array.

### V8_EXPORT
This can be found in quite a few places in v8 source code. For example:

    class V8_EXPORT ArrayBuffer : public Object {

What is this?  
It is a preprocessor macro which looks like this:

    #if V8_HAS_ATTRIBUTE_VISIBILITY && defined(V8_SHARED)
    # ifdef BUILDING_V8_SHARED
    #  define V8_EXPORT __attribute__ ((visibility("default")))
    # else
    #  define V8_EXPORT
    # endif
    #else
    # define V8_EXPORT
    #endif 

So we can see that if `V8_HAS_ATTRIBUTE_VISIBILITY`, and `defined(V8_SHARED)`, and also 
if `BUILDING_V8_SHARED`, `V8_EXPORT` is set to `__attribute__ ((visibility("default"))`.
But in all other cases `V8_EXPORT` is empty and the preprocessor does not insert 
anything (nothing will be there come compile time). 
But what about the `__attribute__ ((visibility("default"))` what is this?  

In the GNU compiler collection (GCC) environment, the term that is used for exporting is visibility. As it 
applies to functions and variables in a shared object, visibility refers to the ability of other shared objects 
to call a C/C++ function. Functions with default visibility have a global scope and can be called from other 
shared objects. Functions with hidden visibility have a local scope and cannot be called from other shared objects.

Visibility can be controlled by using either compiler options or visibility attributes.
In your header files, wherever you want an interface or API made public outside the current Dynamic Shared Object (DSO)
, place `__attribute__ ((visibility ("default")))` in struct, class and function declarations you wish to make public.
 With `-fvisibility=hidden`, you are telling GCC that every declaration not explicitly marked with a visibility attribute 
has a hidden visibility. There is such a flag in build/common.gypi


### ToLocalChecked()
You'll see a few of these calls in the hello_world example:

     Local<String> source = String::NewFromUtf8(isolate, js, NewStringType::kNormal).ToLocalChecked();

NewFromUtf8 actually returns a Local<String> wrapped in a MaybeLocal which forces a check to see if 
the Local<> is empty before using it. 
NewStringType is an enum which can be kNormalString (k for constant) or kInternalized.

The following is after running the preprocessor (clang -E src/api.cc):

    # 5961 "src/api.cc"
    Local<String> String::NewFromUtf8(Isolate* isolate,
                                  const char* data,
                                  NewStringType type,
                                  int length) {
      MaybeLocal<String> result; 
      if (length == 0) { 
        result = String::Empty(isolate); 
      } else if (length > i::String::kMaxLength) { 
        result = MaybeLocal<String>(); 
      } else { 
        i::Isolate* i_isolate = reinterpret_cast<internal::Isolate*>(isolate); 
        i::VMState<v8::OTHER> __state__((i_isolate)); 
        i::RuntimeCallTimerScope _runtime_timer( i_isolate, &i::RuntimeCallStats::API_String_NewFromUtf8); 
        LOG(i_isolate, ApiEntryCall("v8::" "String" "::" "NewFromUtf8")); 
        if (length < 0) length = StringLength(data); 
        i::Handle<i::String> handle_result = NewString(i_isolate->factory(), static_cast<v8::NewStringType>(type), i::Vector<const char>(data, length)) .ToHandleChecked(); 
        result = Utils::ToLocal(handle_result); 
     };
     return result.FromMaybe(Local<String>());;
    }

I was wondering where the Utils::ToLocal was defined but could not find it until I found:

    MAKE_TO_LOCAL(ToLocal, String, String)

    #define MAKE_TO_LOCAL(Name, From, To)                                       \
    Local<v8::To> Utils::Name(v8::internal::Handle<v8::internal::From> obj) {   \
      return Convert<v8::internal::From, v8::To>(obj);                          \
    }

The above can be found in src/api.h. The same goes for `Local<Object>, Local<String>` etc.


### Small Integers
Reading through v8.h I came accross `// Tag information for Smi`
Smi stands for small integers.

A pointer is really just a integer that is treated like a memory address. We can
use that memory address to get the start of the data located in that memory slot.
But we can also just store an normal value like 18 in it. There might be cases
where it does not make sense to store a small integer somewhere in the heap and
have a pointer to it, but instead store the value directly in the pointer itself.
But that only works for small integers so there needs to be away to know if the
value we want is stored in the pointer or if we should follow the value stored to
the heap to get the value.

A word on a 64 bit machine is 8 bytes (64 bits) and all of the pointers need to
be aligned to multiples of 8. So a pointer could be:
```
1000       = 8
10000      = 16
11000      = 24
100000     = 32
1000000000 = 512
```
Remember that we are talking about the pointers and not the values store at
the memory location they point to. We can see that there are always three bits
that are zero in the pointers. So we can use them for something else and just
mask them out when using them as pointers.

Tagging involves borrowing one bit of the 32-bit, making it 31-bit and having
the leftover bit represent a tag. If the tag is zero then this is a plain value,
but if tag is 1 then the pointer must be followed.
This does not only have to be for numbers it could also be used for object (I think)

Instead the small integer is represented by the 32 bits plus a pointer to the
64-bit number. V8 needs to know if a value stored in memory represents a 32-bit
integer, or if it is really a 64-bit number, in which case it has to follow the
pointer to get the complete value. This is where the concept of tagging comes in.



### Properties/Elements
Take the following object:

    { firstname: "Jon", lastname: "Doe' }

The above object has two named properties. Named properties differ from integer indexed 
which is what you have when you are working with arrays.

Memory layout of JavaScript Object:
```
Properties                  JavaScript Object               Elements
+-----------+              +-----------------+         +----------------+
|property1  |<------+      | HiddenClass     |  +----->|                |
+-----------+       |      +-----------------+  |      +----------------+
|...        |       +------| Properties      |  |      | element1       |<------+
+-----------+              +-----------------+  |      +----------------+       |
|...        |              | Elements        |--+      | ...            |       |
+-----------+              +-----------------+         +----------------+       |
|propertyN  | <---------------------+                  | elementN       |       |
+-----------+                       |                  +----------------+       |
                                    |                                           |
                                    |                                           |
                                    |                                           | 
Named properties:    { firstname: "Jon", lastname: "Doe' } Indexed Properties: {1: "Jon", 2: "Doe"}
```
We can see that properies and elements are stored in different data structures.
Elements are usually implemented as a plain array and the indexes can be used for fast access
to the elements. 
But for the properties this is not the case. Instead there is a mapping between the property names
and the index into the properties.

In `src/objects/objects.h` we can find JSObject:

    class JSObject: public JSReceiver {
    ...
    DECL_ACCESSORS(elements, FixedArrayBase)


And looking a the `DECL_ACCESSOR` macro:

    #define DECL_ACCESSORS(name, type)    \
      inline type* name() const;          \
      inline void set_##name(type* value, \
                             WriteBarrierMode mode = UPDATE_WRITE_BARRIER);

    inline FixedArrayBase* name() const;
    inline void set_elements(FixedArrayBase* value, WriteBarrierMode = UPDATE_WRITE_BARRIER)

Notice that JSObject extends JSReceiver which is extended by all types that can have properties defined on them. I think this includes all JSObjects and JSProxy. It is in JSReceiver that the we find the properties array:

    DECL_ACCESSORS(raw_properties_or_hash, Object)

Now properties (named properties not elements) can be of different kinds internally. These work just
like simple dictionaries from the outside but a dictionary is only used in certain curcumstances
at runtime.

```
Properties                  JSObject                    HiddenClass (Map)
+-----------+              +-----------------+         +----------------+
|property1  |<------+      | HiddenClass     |-------->| bit field1     |
+-----------+       |      +-----------------+         +----------------+
|...        |       +------| Properties      |         | bit field2     |
+-----------+              +-----------------+         +----------------+
|...        |              | Elements        |         | bit field3     |
+-----------+              +-----------------+         +----------------+
|propertyN  |              | property1       |         
+-----------+              +-----------------+         
                           | property2       |
                           +-----------------+
                           | ...             |
                           +-----------------+

```

#### JSObject
Each JSObject has as its first field a pointer to the generated HiddenClass. A hiddenclass contain mappings from property names to indices into the properties data type. When an instance of JSObject is created a `Map` is passed in.
As mentioned earlier JSObject inherits from JSReceiver which inherits from HeapObject

For example,in [jsobject_test.cc](./test/jsobject_test.cc) we first create a new Map using the internal Isolate Factory:

    v8::internal::Handle<v8::internal::Map> map = factory->NewMap(v8::internal::JS_OBJECT_TYPE, 24);
    v8::internal::Handle<v8::internal::JSObject> js_object = factory->NewJSObjectFromMap(map);
    EXPECT_TRUE(js_object->HasFastProperties());

When we call `js_object->HasFastProperties()` this will delegate to the map instance:

    return !map()->is_dictionary_map();

How do you add a property to a JSObject instance?
Take a look at [jsobject_test.cc](./test/jsobject_test.cc) for an example.


### Caching
Are ways to optimize polymorphic function calls in dynamic languages, for example JavaScript.

#### Lookup caches
Sending a message to a receiver requires the runtime to find the correct target method using
the runtime type of the receiver. A lookup cache maps the type of the receiver/message name
pair to methods and stores the most recently used lookup results. The cache is first consulted
and if there is a cache miss a normal lookup is performed and the result stored in the cache.

#### Inline caches
Using a lookup cache as described above still takes a considerable amount of time since the
cache must be probed for each message. It can be observed that the type of the target does often
not vary. If a call to type A is done at a particular call site it is very likely that the next
time it is called the type will also be A.
The method address looked up by the system lookup routine can be cached and the call instruction
can be overwritten. Subsequent calls for the same type can jump directly to the cached method and
completely avoid the lookup. The prolog of the called method must verify that the receivers
type has not changed and do the lookup if it has changed (the type if incorrect, no longer A for
example).

The target methods address is stored in the callers code, or "inline" with the callers code, 
hence the name "inline cache".

If V8 is able to make a good assumption about the type of object that will be passed to a method,
it can bypass the process of figuring out how to access the objects properties, and instead use the stored information from previous lookups to the objects hidden class.

#### Polymorfic Inline cache (PIC)
A polymorfic call site is one where there are many equally likely receiver types (and thus
call targets).

- Monomorfic means there is only one receiver type
- Polymorfic a few receiver types
- Megamorfic very many receiver types

This type of caching extends inline caching to not just cache the last lookup, but cache
all lookup results for a given polymorfic call site using a specially generated stub.
Lets say we have a method that iterates through a list of types and calls a method. If 
all the types are the same (monomorfic) a PIC acts just like an inline cache. The calls will
directly call the target method (with the method prolog followed by the method body).
If a different type exists in the list there will be a cache miss in the prolog and the lookup
routine called. In normal inline caching this would rebind the call, replacing the call to this
types target method. This would happen each time the type changes.

With PIC the cache miss handler will generate a small stub routine and rebinds the call to this
stub. The stub will check if the receiver is of a type that it has seen before and branch to 
the correct targets. Since the type of the target is already known at this point it can directly
branch to the target method body without the need for the prolog.
If the type has not been seen before it will be added to the stub to handle that type. Eventually
the stub will contain all types used and there will be no more cache misses/lookups.

The problem is that we don't have type information so methods cannot be called directly, but 
instead be looked up. In a static language a virtual table might have been used. In JavaScript
there is no inheritance relationship so it is not possible to know a vtable offset ahead of time.
What can be done is to observe and learn about the "types" used in the program. When an object
is seen it can be stored and the target of that method call can be stored and inlined into that
call. Bascially the type will be checked and if that particular type has been seen before the
method can just be invoked directly. But how do we check the type in a dynamic language? The
answer is hidden classes which allow the VM to quickly check an object against a hidden class.

The inline caching source are located in `src/ic`.

## --trace-ic

    $ out/x64.debug/d8 --trace-ic --trace-maps class.js

    before
    [TraceMaps: Normalize from= 0x19a314288b89 to= 0x19a31428aff9 reason= NormalizeAsPrototype ]
    [TraceMaps: ReplaceDescriptors from= 0x19a31428aff9 to= 0x19a31428b051 reason= CopyAsPrototype ]
    [TraceMaps: InitialMap map= 0x19a31428afa1 SFI= 34_Person ]

    [StoreIC in ~Person+65 at class.js:2 (0->.) map=0x19a31428afa1 0x10e68ba83361 <String[4]: name>]
    [TraceMaps: Transition from= 0x19a31428afa1 to= 0x19a31428b0a9 name= name ]
    [StoreIC in ~Person+102 at class.js:3 (0->.) map=0x19a31428b0a9 0x2beaa25abd89 <String[3]: age>]
    [TraceMaps: Transition from= 0x19a31428b0a9 to= 0x19a31428b101 name= age ]
    [TraceMaps: SlowToFast from= 0x19a31428b051 to= 0x19a31428b159 reason= OptimizeAsPrototype ]
    [StoreIC in ~Person+65 at class.js:2 (.->1) map=0x19a31428afa1 0x10e68ba83361 <String[4]: name>]
    [StoreIC in ~Person+102 at class.js:3 (.->1) map=0x19a31428b0a9 0x2beaa25abd89 <String[3]: age>]
    [LoadIC in ~+546 at class.js:9 (0->.) map=0x19a31428b101 0x10e68ba83361 <String[4]: name>]
    [CallIC in ~+571 at class.js:9 (0->1) map=0x0 0x32f481082231 <String[5]: print>]
    Daniel
    [LoadIC in ~+642 at class.js:10 (0->.) map=0x19a31428b101 0x2beaa25abd89 <String[3]: age>]
    [CallIC in ~+667 at class.js:10 (0->1) map=0x0 0x32f481082231 <String[5]: print>]
    41
    [LoadIC in ~+738 at class.js:11 (0->.) map=0x19a31428b101 0x10e68ba83361 <String[4]: name>]
    [CallIC in ~+763 at class.js:11 (0->1) map=0x0 0x32f481082231 <String[5]: print>]
    Tilda
    [LoadIC in ~+834 at class.js:12 (0->.) map=0x19a31428b101 0x2beaa25abd89 <String[3]: age>]
    [CallIC in ~+859 at class.js:12 (0->1) map=0x0 0x32f481082231 <String[5]: print>]
    2
    [CallIC in ~+927 at class.js:13 (0->1) map=0x0 0x32f481082231 <String[5]: print>]
    after

LoadIC (0->.) means that it has transitioned from unititialized state (0) to pre-monomophic state (.)
monomorphic state is specified with a `1`. These states can be found in [src/ic/ic.cc](https://github.com/v8/v8/blob/df1494d69deab472a1a709bd7e688297aa5cc655/src/ic/ic.cc#L33-L52).
What we are doing caching knowledge about the layout of the previously seen object inside the StoreIC/LoadIC calls.

    $ lldb -- out/x64.debug/d8 class.js

#### HeapObject
This class describes heap allocated objects. It is in this class we find information regarding the type of object. This 
information is contained in `v8::internal::Map`.

### v8::internal::Map
`src/objects/map.h`  
* `bit_field1`  
* `bit_field2`
* `bit field3` contains information about the number of properties that this Map has,
a pointer to an DescriptorArray. The DescriptorArray contains information like the name of the 
property, and the posistion where the value is stored in the JSObject.
I noticed that this information available in src/objects/map.h. 

#### DescriptorArray
Can be found in src/objects/descriptor-array.h. This class extends FixedArray and has the following
entries:

```
[0] the number of descriptors it contains  
[1] If uninitialized this will be Smi(0) otherwise an enum cache bridge which is a FixedArray of size 2: 
  [0] enum cache: FixedArray containing all own enumerable keys  
  [1] either Smi(0) or a pointer to a FixedArray with indices  
[2] first key (and internalized String  
[3] first descriptor  
```
### Factory
Each Internal Isolate has a Factory which is used to create instances. This is because all handles needs to be allocated
using the factory (src/heap/factory.h)


### Objects 
All objects extend the abstract class Object (src/objects/objects.h).

### Oddball
This class extends HeapObject and  describes `null`, `undefined`, `true`, and `false` objects.


#### Map
Extends HeapObject and all heap objects have a Map which describes the objects structure.
This is where you can find the size of the instance, access to the inobject_properties.

### Compiler pipeline
When a script is compiled all of the top level code is parsed. These are function declarartions (but not the function
bodies). 

    function f1() {       <- top level code
      console.log('f1');  <- non top level
    }

    function f2() {       <- top level code
      f1();               <- non top level
      console.logg('f2'); <- non top level
    }

    f2();                 <- top level code
    var i = 10;           <- top level code

The non top level code must be pre-parsed to check for syntax errors.
The top level code is parsed and compiles by the full-codegen compiler. This compiler does not perform any optimizations and
it's only task is to generate machine code as quickly as possible (this is pre turbofan)

    Source ------> Parser  --------> Full-codegen ---------> Unoptimized Machine Code

So the whole script is parsed even though we only generated code for the top-level code. The pre-parse (the syntax checking)
was not stored in any way. The functions are lazy stubs that when/if the function gets called the function get compiled. This
means that the function has to be parsed (again, the first time was the pre-parse remember).

If a function is determined to be hot it will be optimized by one of the two optimizing compilers crankshaft for older parts of JavaScript or Turbofan for Web Assembly (WASM) and some of the newer es6 features.

The first time V8 sees a function it will parse it into an AST but not do any further processing of that tree
until that function is used. 

                         +-----> Full-codegen -----> Unoptimized code
                        /                               \/ /\       \
    Parser  ------> AST -------> Cranshaft    -----> Optimized code  |
                        \                                           /
                         +-----> Turbofan     -----> Optimized code

Inline Cachine (IC) is done here which also help to gather type information.
V8 also has a profiler thread which monitors which functions are hot and should be optimized. This profiling
also allows V8 to find out information about types using IC. This type information can then be fed to Crankshaft/Turbofan.
The type information is stored as a 8 bit value. 

When a function is optimized the unoptimized code cannot be thrown away as it might be needed since JavaScript is highly
dynamic the optimzed function migth change and the in that case we fallback to the unoptimzed code. This takes up
alot of memory which may be important for low end devices. Also the time spent in parsing (twice) takes time.

The idea with Ignition is to be an bytecode interpreter and to reduce memory consumption, the bytecode is very consice
compared to native code which can vary depending on the target platform.
The whole source can be parsed and compiled, compared to the current pipeline the has the pre-parse and parse stages mentioned above. So even unused functions will get compiled.
The bytecode becomes the source of truth instead of as before the AST.

    Source ------> Parser  --------> Ignition-codegen ---------> Bytecode ---------> Turbofan ----> Optimized Code ---+
                                                                  /\                                                  |
                                                                   +--------------------------------------------------+

    function bajja(a, b, c) {
      var d = c - 100;
      return a + d * b;
    }

    var result = bajja(2, 2, 150);
    print(result); 

    $ ./d8 test.js --ignition  --print_bytecode

    [generating bytecode for function: bajja]
    Parameter count 4
    Frame size 8
     14 E> 0x2eef8d9b103e @    0 : 7f                StackCheck
     38 S> 0x2eef8d9b103f @    1 : 03 64             LdaSmi [100]   // load 100
     38 E> 0x2eef8d9b1041 @    3 : 2b 02 02          Sub a2, [2]    // a2 is the third argument. a2 is an argument register
           0x2eef8d9b1044 @    6 : 1f fa             Star r0        // r0 is a register for local variables. We only have one which is d
     47 S> 0x2eef8d9b1046 @    8 : 1e 03             Ldar a1        // LoaD accumulator from Register argument a1 which is b
     60 E> 0x2eef8d9b1048 @   10 : 2c fa 03          Mul r0, [3]    // multiply that is our local variable in r0
     56 E> 0x2eef8d9b104b @   13 : 2a 04 04          Add a0, [4]    // add that to our argument register 0 which is a 
     65 S> 0x2eef8d9b104e @   16 : 83                Return         // return the value in the accumulator?


### Abstract Syntax Tree (AST)
In src/ast/ast.h. You can print the ast using the `--print-ast` option for d8.

Lets take the following javascript and look at the ast:

    const msg = 'testing';
    console.log(msg);

```
$ d8 --print-ast simple.js
[generating interpreter code for user-defined function: ]
--- AST ---
FUNC at 0
. KIND 0
. SUSPEND COUNT 0
. NAME ""
. INFERRED NAME ""
. DECLS
. . VARIABLE (0x7ffe5285b0f8) (mode = CONST) "msg"
. BLOCK NOCOMPLETIONS at -1
. . EXPRESSION STATEMENT at 12
. . . INIT at 12
. . . . VAR PROXY context[4] (0x7ffe5285b0f8) (mode = CONST) "msg"
. . . . LITERAL "testing"
. EXPRESSION STATEMENT at 23
. . ASSIGN at -1
. . . VAR PROXY local[0] (0x7ffe5285b330) (mode = TEMPORARY) ".result"
. . . CALL Slot(0)
. . . . PROPERTY Slot(4) at 31
. . . . . VAR PROXY Slot(2) unallocated (0x7ffe5285b3d8) (mode = DYNAMIC_GLOBAL) "console"
. . . . . NAME log
. . . . VAR PROXY context[4] (0x7ffe5285b0f8) (mode = CONST) "msg"
. RETURN at -1
. . VAR PROXY local[0] (0x7ffe5285b330) (mode = TEMPORARY) ".result"
```
You can find the declaration of EXPRESSION in ast.h.

### Bytecode
Can be found in src/interpreter/bytecodes.h

* StackCheck checks that stack limits are not exceeded to guard against overflow.
* `Star` Store content in accumulator regiser in register (the operand).
* Ldar   LoaD accumulator from Register argument a1 which is b

The registers are not machine registers, apart from the accumlator as I understand it, but would instead be stack allocated.


#### Parsing
Parsing is the parsing of the JavaScript and the generation of the abstract syntax tree. That tree is then visited and 
bytecode generated from it. This section tries to figure out where in the code these operations are performed.

For example, take the script example.

    $ make run-script
    $ lldb -- run-script
    (lldb) br s -n main
    (lldb) r

Lets take a look at the following line:

    Local<Script> script = Script::Compile(context, source).ToLocalChecked();

This will land us in `api.cc`

    ScriptCompiler::Source script_source(source);
    return ScriptCompiler::Compile(context, &script_source);

    MaybeLocal<Script> ScriptCompiler::Compile(Local<Context> context, Source* source, CompileOptions options) {
    ...
    auto isolate = context->GetIsolate();
    auto maybe = CompileUnboundInternal(isolate, source, options);

`CompileUnboundInternal` will call `GetSharedFunctionInfoForScript` (in src/compiler.cc):

    result = i::Compiler::GetSharedFunctionInfoForScript(
          str, name_obj, line_offset, column_offset, source->resource_options,
          source_map_url, isolate->native_context(), NULL, &script_data, options,
          i::NOT_NATIVES_CODE);

    (lldb) br s -f compiler.cc -l 1259

    LanguageMode language_mode = construct_language_mode(FLAG_use_strict);
    (lldb) p language_mode
    (v8::internal::LanguageMode) $10 = SLOPPY

`LanguageMode` can be found in src/globals.h and it is an enum with three values:

    enum LanguageMode : uint32_t { SLOPPY, STRICT, LANGUAGE_END };

`SLOPPY` mode, I assume, is the mode when there is no `"use strict";`. Remember that this can go inside a function and does not
have to be at the top level of the file.

    ParseInfo parse_info(script);

There is a [unit test](./test/ast_test.cc) that shows how a ParseInfo instance can be created
and inspected.

This will call ParseInfo's constructor (in src/parsing/parse-info.cc), and which will call `ParseInfo::InitFromIsolate`:

    DCHECK_NOT_NULL(isolate);
    set_hash_seed(isolate->heap()->HashSeed());
    set_stack_limit(isolate->stack_guard()->real_climit());
    set_unicode_cache(isolate->unicode_cache());
    set_runtime_call_stats(isolate->counters()->runtime_call_stats());
    set_ast_string_constants(isolate->ast_string_constants());

I was curious about these ast_string_constants:

    (lldb) p *ast_string_constants_
    (const v8::internal::AstStringConstants) $58 = {
      zone_ = {
        allocation_size_ = 1312
        segment_bytes_allocated_ = 8192
        position_ = 0x0000000105052538 <no value available>
        limit_ = 0x0000000105054000 <no value available>
        allocator_ = 0x0000000103e00080
        segment_head_ = 0x0000000105052000
        name_ = 0x0000000101623a70 "../../src/ast/ast-value-factory.h:365"
        sealed_ = false
      }
      string_table_ = {
        v8::base::TemplateHashMapImpl<void *, void *, v8::base::HashEqualityThenKeyMatcher<void *, bool (*)(void *, void *)>, v8::base::DefaultAllocationPolicy> = {
          map_ = 0x0000000105054000
          capacity_ = 64
          occupancy_ = 41
          match_ = {
            match_ = 0x000000010014b260 (libv8.dylib`v8::internal::AstRawString::Compare(void*, void*) at ast-value-factory.cc:122)
          }
        }
      }
      hash_seed_ = 500815076
      anonymous_function_string_ = 0x0000000105052018
      arguments_string_ = 0x0000000105052038
      async_string_ = 0x0000000105052058
      await_string_ = 0x0000000105052078
      boolean_string_ = 0x0000000105052098
      constructor_string_ = 0x00000001050520b8
      default_string_ = 0x00000001050520d8
      done_string_ = 0x00000001050520f8
      dot_string_ = 0x0000000105052118
      dot_for_string_ = 0x0000000105052138
      dot_generator_object_string_ = 0x0000000105052158
      dot_iterator_string_ = 0x0000000105052178
      dot_result_string_ = 0x0000000105052198
      dot_switch_tag_string_ = 0x00000001050521b8
      dot_catch_string_ = 0x00000001050521d8
      empty_string_ = 0x00000001050521f8
      eval_string_ = 0x0000000105052218
      function_string_ = 0x0000000105052238
      get_space_string_ = 0x0000000105052258
      length_string_ = 0x0000000105052278
      let_string_ = 0x0000000105052298
      name_string_ = 0x00000001050522b8
      native_string_ = 0x00000001050522d8
      new_target_string_ = 0x00000001050522f8
      next_string_ = 0x0000000105052318
      number_string_ = 0x0000000105052338
      object_string_ = 0x0000000105052358
      proto_string_ = 0x0000000105052378
      prototype_string_ = 0x0000000105052398
      return_string_ = 0x00000001050523b8
      set_space_string_ = 0x00000001050523d8
      star_default_star_string_ = 0x00000001050523f8
      string_string_ = 0x0000000105052418
      symbol_string_ = 0x0000000105052438
      this_string_ = 0x0000000105052458
      this_function_string_ = 0x0000000105052478
      throw_string_ = 0x0000000105052498
      undefined_string_ = 0x00000001050524b8
      use_asm_string_ = 0x00000001050524d8
      use_strict_string_ = 0x00000001050524f8
      value_string_ = 0x0000000105052518
    } 

So these are constants that are set on the new ParseInfo instance using the values from the isolate. Not exactly sure what I 
want with this but I might come back to it later.
So, we are back in ParseInfo's constructor:

    set_allow_lazy_parsing();
    set_toplevel();
    set_script(script);

Script is of type v8::internal::Script which can be found in src/object/script.h

Back now in compiler.cc and the GetSharedFunctionInfoForScript function:

    Zone compile_zone(isolate->allocator(), ZONE_NAME);

    ...
    if (parse_info->literal() == nullptr && !parsing::ParseProgram(parse_info, isolate))

`ParseProgram`:

    Parser parser(info);
    ...
    FunctionLiteral* result = nullptr;
    result = parser.ParseProgram(isolate, info);

`parser.ParseProgram`: 

    Handle<String> source(String::cast(info->script()->source()));


    (lldb) job *source
    "var user1 = new Person('Fletch');\x0avar user2 = new Person('Dr.Rosen');\x0aprint("user1 = " + user1.name);\x0aprint("user2 = " + user2.name);\x0a\x0a"

So here we can see our JavaScript as a String.

    std::unique_ptr<Utf16CharacterStream> stream(ScannerStream::For(source));
    scanner_.Initialize(stream.get(), info->is_module());
    result = DoParseProgram(info);

`DoParseProgram`:

    (lldb) br s -f parser.cc -l 639
    ...

    this->scope()->SetLanguageMode(info->language_mode());
    ParseStatementList(body, Token::EOS, &ok);

This call will land in parser-base.h and its `ParseStatementList` function.

    (lldb) br s -f parser-base.h -l 4695

    StatementT stat = ParseStatementListItem(CHECK_OK_CUSTOM(Return, kLazyParsingComplete));

    result = CompileToplevel(&parse_info, isolate, Handle<SharedFunctionInfo>::null());

This will land in `CompileTopelevel` (in the same file which is src/compiler.cc):

    // Compile the code.
    result = CompileUnoptimizedCode(parse_info, shared_info, isolate);

This will land in `CompileUnoptimizedCode` (in the same file which is src/compiler.cc):

    // Prepare and execute compilation of the outer-most function.
    std::unique_ptr<CompilationJob> outer_job(
       PrepareAndExecuteUnoptimizedCompileJob(parse_info, parse_info->literal(),
                                              shared_info, isolate));


    std::unique_ptr<CompilationJob> job(
        interpreter::Interpreter::NewCompilationJob(parse_info, literal, isolate));
    if (job->PrepareJob() == CompilationJob::SUCCEEDED &&
        job->ExecuteJob() == CompilationJob::SUCCEEDED) {
      return job;
    }

PrepareJobImpl:

    CodeGenerator::MakeCodePrologue(parse_info(), compilation_info(),
                                    "interpreter");
    return SUCCEEDED;

codegen.cc `MakeCodePrologue`:

interpreter.cc ExecuteJobImpl:

    generator()->GenerateBytecode(stack_limit());    

src/interpreter/bytecode-generator.cc

     RegisterAllocationScope register_scope(this);

The bytecode is register based (if that is the correct term) and we had an example previously. I'm guessing 
that this is what this call is about.

VisitDeclarations will iterate over all the declarations in the file which in our case are:

    var user1 = new Person('Fletch');
    var user2 = new Person('Dr.Rosen');

    (lldb) p *variable->raw_name()
    (const v8::internal::AstRawString) $33 = {
       = {
        next_ = 0x000000010600a280
        string_ = 0x000000010600a280
      }
      literal_bytes_ = (start_ = "user1", length_ = 5)
      hash_field_ = 1303438034
      is_one_byte_ = true
      has_string_ = false
    }

    // Perform a stack-check before the body.
    builder()->StackCheck(info()->literal()->start_position());

So that call will output a stackcheck instruction, like in the example above:

    14 E> 0x2eef8d9b103e @    0 : 7f                StackCheck

### Performance
Say you have the expression x + y the full-codegen compiler might produce:

    movq rax, x
    movq rbx, y
    callq RuntimeAdd

If x and y are integers just using the `add` operation would be much quicker:

    movq rax, x
    movq rbx, y
    add rax, rbx


Recall that functions are optimized so if the compiler has to bail out and unoptimize 
part of a function then the whole functions will be affected and it will go back to 
the unoptimized version.

## Bytecode
This section will examine the bytecode for the following JavaScript:

    function beve() {
      const p = new Promise((resolve, reject) => {
        resolve('ok');
      });

      p.then(msg => {
        console.log(msg);
      });
    }

    beve(); 

    $ d8 --print-bytecode promise.js

First have the main function which does not have a name:

    [generating bytecode for function: ]
    (The code that generated this can be found in src/objects.cc BytecodeArray::Dissassemble)
    Parameter count 1
    Frame size 32
           // load what ever the FixedArray[4] is in the constant pool into the accumulator.
           0x34423e7ac19e @    0 : 09 00             LdaConstant [0] 
           // store the FixedArray[4] in register r1
           0x34423e7ac1a0 @    2 : 1e f9             Star r1
           // store zero into the accumulator.
           0x34423e7ac1a2 @    4 : 02                LdaZero
           // store zero (the contents of the accumulator) into register r2.
           0x34423e7ac1a3 @    5 : 1e f8             Star r2
           // 
           0x34423e7ac1a5 @    7 : 1f fe f7          Mov <closure>, r3
           0x34423e7ac1a8 @   10 : 53 96 01 f9 03    CallRuntime [DeclareGlobalsForInterpreter], r1-r3
      0 E> 0x34423e7ac1ad @   15 : 90                StackCheck
    141 S> 0x34423e7ac1ae @   16 : 0a 01 00          LdaGlobal [1], [0]
           0x34423e7ac1b1 @   19 : 1e f9             Star r1
    141 E> 0x34423e7ac1b3 @   21 : 4f f9 03          CallUndefinedReceiver0 r1, [3]
           0x34423e7ac1b6 @   24 : 1e fa             Star r0
    148 S> 0x34423e7ac1b8 @   26 : 94                Return

    Constant pool (size = 2)
    0x34423e7ac149: [FixedArray] in OldSpace
     - map = 0x344252182309 <Map(HOLEY_ELEMENTS)>
     - length: 2
           0: 0x34423e7ac069 <FixedArray[4]>
           1: 0x34423e7abf59 <String[4]: beve>

    Handler Table (size = 16) Load the global with name in constant pool entry <name_index> into the
    // accumulator using FeedBackVector slot <slot> outside of a typeof

* LdaConstant <idx> 
Load the constant at index from the constant pool into the accumulator.  
* Star <dst>
Store the contents of the accumulator register in dst.  
* Ldar <src>
Load accumulator with value from register src.  
* LdaGlobal <idx> <slot>
Load the global with name in constant pool entry idx into the accumulator using FeedBackVector slot  outside of a typeof.
* Mov <closure>, <r3>
Store the value of register  

You can find the declarations for the these instructions in `src/interpreter/interpreter-generator.cc`.


## Unified code generation architecture

## FeedbackVector
Is attached to every function and is responsible for recording and managing all execution feedback, which is information about types enabling. 
You can find the declaration for this class in `src/feedback-vector.h`


## BytecodeGenerator
Is currently the only part of V8 that cares about the AST.

## BytecodeGraphBuilder
Produces high-level IR graph based on interpreter bytecodes.


## TurboFan
Is a compiler backend that gets fed a control flow graph and then does instruction selection, register allocation and code generation. The code generation generates 


### Execution/Runtime
I'm not sure if V8 follows this exactly but I've heard and read that when the engine comes 
across a function declaration it only parses and verifies the syntax and saves a ref
to the function name. The statements inside the function are not checked at this stage
only the syntax of the function declaration (parenthesis, arguments, brackets etc). 



