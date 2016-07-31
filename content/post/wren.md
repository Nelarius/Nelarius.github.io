
+++
title = "Embedding Wren in C++, part 1"
date = "2016-05-05"
draft = true
menu = "main"
description = "A brief look at embedding Wren in your C++ application by example"
tags = ["c++", "wren", "embedding"]


+++

Wren is a small, fast, class-based scripting language designed to be easily embeddable in a host application. As such, it fills very much the same niche as Lua. Why would you choose Wren over Lua in your application? It boils mostly down to stylistic preferences. Wren is very class oriented, and adheres to the curly-brace style tradition, in contrast to Lua's `begin...end` blocks. If one of the first things you always did in your Lua projects was design a class model, then Wren might be worth checking out. If you're in the mood for surveying new tech for your C++ application or game engine, then definitely check it out! Wren is currently under active development, so breaking changes might be occasionally introduced. The source code lives on [github](https://github.com/munificent/wren).

Even though Wren's embedding API is simple to use, it can be a lot of work to manually write all of the binding code. For that reason, I've worked on a template-based library to automatically generate the binding code for you. It's called Wren++, and is heavily inspired by libraries like Luabridge for the Lua programming language. You can find it on [github](https://github.com/Nelarius/wrenpp) as well.

This post is divided into two parts. First, I will give an introduction on using Wren's C API via a few examples. You will see how to handle method arguments and return values, as well as how binding C++ classes to Wren works. After you've seen how Wren's embedding API works, I'll show how those same examples could be done using Wren++.

A tour of the language itself is a bit beyond the scope of this article, but a very good overview of the language is presented on the Wren website: [wren.io](http://wren.io). If you are not familiar with the language, then I suggest you go there and read before proceeding with this post. You can even test Wren code in the browser over at the [wren-nest](http://ppvk.github.io/wren-nest/).

## The wren embedding API

In order to use Wren, you need to link the wren static library to your program. The header file `wren.h` also need to be included (located in `wren/src/include/`). When you include `wren.h` into your C++ source code, don't forget to put it in an `extern "C"` block to disable C++ name mangling on the C code:

```cpp
extern "C" {
#include <wren.h>
}
```

The best guide to the embedding API is the `wren.h` header file itself, at the moment. I recommend that you have it open and refer to it as needed while reading this post!

### Creating a VM

Having gotten the project set-up out of the way, let's create a new instance of the virtual machine and execute some code.

```cpp
// create the configuration that the virtual machine will use
WrenConfiguration configuration;
// fill the configuration with default values
wrenInitConfiguration(&configuration);

WrenVM* vm = wrenNewVM(&configuration);

// do something with your vm!
WrenInterpretResult result = wrenInterpret(vm, "System.print(\"Hello, world!\")");
```

If everything went okay, then result will be `WREN_RESULT_SUCCESS`, but `WREN_RESULT_COMPILE_ERROR` and `WREN_RESULT_RUNTIME_ERROR` can be returned as well. Once you are done with the VM, it can be freed by calling `wrenFreeVM(WrenVM* vm)`.

### Basic configuration

The above example doesn't actually print anything, because Wren doesn't know what to do with the `System.print` call. You have to provide that function yourself. You can do this by assigning a callback to the configuration's  `writeFn` field, which must be a function with the `void(WrenVM*, const char*)` signature.

```cpp
void write(WrenVM* vm, const char* str) {
  std::printf("%s", str);
}

// elsewhere...
WrenConfiguration config;
wrenInitConfiguration(&config);
config.writeFn = write;
```

Wren doesn't load modules either by default. Again, you have to provide that function yourself. The module loading function must have the signature `char*(WrenVM*, const char*)`.

```cpp
// in this example, module names work the same way as they do in the
// CLI. That is, the module name is specified without the .wren postfix.
char* loadModule(WrenVM* vm, const char* name) {
  std::string path( name );
  path += ".wren";
  std::ifstream fin;
  fin.open( path, std::ios::in );
  std::stringstream buffer;
  buffer << fin.rdbuf() << '\0';
  std::string source = buffer.str();
  
  char* cbuffer = (char*) malloc( source.size() );
  memcpy( cbuffer, source.c_str(), source.size() );
  return cbuffer;
}

// elsewhere...
config.loadModuleFn = loadModule;
```

Now that our VM is configured, let's bind some C++ code to it.

### Binding free functions to Wren

There are two different hooks for foreign code in Wren. C++ code can be called from foreign methods.

```dart
class Math {
  foreign static cos(num)
  foreign static sin(num)
}
```

C++ state may be stored in foreign classes.

```dart
foreign class Vec3 {
  foreign plus(rhs)
}
```

When Wren encounters a foreign method, it calls a callback function which implements the method. All foreign method callbacks have the same type, `typedef void (*WrenForeignMethodFn)(WrenVM*)`, as defined in `wren.h`. These callbacks are where your glue code or implementation lives. How does Wren find them? Wren uses another callback function, which should return the correct callback function based on the foreign method's signature. This callback finder (`WrenBindForeignMethodFn` in `wren.h`) can be assigned to the `bindForeignMethodFn` field of `WrenConfiguration`. 

One way of implementing the callback finder is to store the foreign method callback functions as values in a global data structure such as a map, and have the `bindForeignMethodFn` find the them using the foreign method signature as a key. Here's the gist of what Wren++ does:

```cpp
// a map between the foreign method signature and the callback function
std::map<std::string, WrenForeignMethodFn> boundForeignMethods{};

// this function has the signature corresponding to bindForeignMethodFn
WrenForeignMethodFn bindForeignMethod(WrenVM* vm,
                                      const char* module,
                                      const char* className,
                                      bool isStatic,
                                      const char* signature
                                      ) {
  std::string fullSignature{ module };
  fullSignature += className;
  fullSignature += signature;
  if (isStatic) {
      fullSignature += "s";
  }
  auto it = boundForeignMethods.find(fullSignature);
  if (it != boundsForeignMethods.end()) {
      return it->second;
  }
  return nullptr;
}
```

Now that we have that in place, let's implement the methods of the `Math` class from above. We are going to implement two functions to match the foreign methods in Wren: `wrenSin` and `wrenCos`.

Looking at the declaration for `WrenForeignMethodFn`, we can see that we have only the Wren VM instance to work with in our glue code. The Wren VM provides a slot API which we can use to access method arguments, as well as pass return values back to Wren.

```cpp
extern "C" {
#include <wren.h>
}
#include <cmath>

void wrenCos(WrenVM* vm) {
  // method arguments are placed in slots 1 and up
  // the receiver of the method (the object) is always in slot 0
  //
  // in Wren, the Number type is actually just a double. So when we pass 
  // a Number as a method argument in Wren, we need to get a double
  // from the slot in C++
  double x = wrenGetSlotDouble(vm, 1);

  // identical methods exist for bool, string, wren values, and foreign objects

  // the method's return value is taken from slot 0, so let's place our
  // result there
  wrenSetSlotDouble(vm, 0, std::cos(x));
}

void wrenSin(WrenVM* vm) {
  double x = wrenGetSlotDouble(vm, 1);
  wrenSetSlotDouble(vm, 0, std::sin(x));
}

std::map<std::string, WrenForeignMethodFn> boundForeignMethods{
  { "mainMathcos(_)s", wrenCos },
  { "mainMathsin(_)s", wrenSin }
};

WrenForeignMethodFn bindForeignMethod(WrenVM* vm,
                                      const char* module,
                                      const char* className,
                                      bool isStatic,
                                      const char* signature
                                      ) {
  std::string signature{ module };
  signature += className;
  signature += signature;
  if (isStatic) {
      signature += "s";
  }
  auto it = boundForeignMethods.find(signature);
  if (it != boundsForeignMethods.end()) {
      return it->second;
  }
  return nullptr;
}

int main() {
  WrenConfiguration configuration;
  wrenInitConfiguration(&configuration);
  configuration.bindForeignMethodFn = bindForeignMethod;
  configuration.writeFn= write;
  configuration.loadModuleFn = loadModule;

  WrenVM* vm = wrenNewVM(&configuration);
  // you can now use the math class
  wrenInterpret(
    vm,
    "class Math {\n"
    "  foreign static cos(num)\n"
    "  foreign static sin(num)\n"
    "}\n"
    "System.print(\"%(Math.cos(1.570796326))\")\n"
  );
  wrenFreeVM(vm);
  return 0;
}
```

Note that when we initialized `boundForeignMethods` with the two bound callbacks, the module's name was `main`. If the class whose methods you are implementing are not in a separate file (as was the case above), then the module's name is `main`. If the math class had been defined in, say, `math.wren`, then the module name would have been `math`.

We can get away with our expensive string concatenation and map-find operations within `bindForeignMethod`, because it gets called for each method only once, when the class is defined.

Also, note that we wrote the signature as `cos(_)` and `sin(_)`. You don't spell out the argument name in the signature, but instead you replace each argument with an underscore. Since the argument underscores are part of the method signature, you can bind functions to overloaded methods in Wren.

### Binding classes to Wren

Wren can store user-defined state in an instance of a foreign class. We are going to use this feature to bind our C++ types to Wren.

When Wren encounters a foreign class, it looks up the allocator and finalizer callbacks for that class via a class-finder callback, which behaves very similarly to the `bindForeignMethodFn` that we just implemented. Wren uses the allocator callback to create a byte array within the object, and initialize whatever state the user might want to reside within the byte array. The byte array can be accessed via the slot API. The finalizer can be used to call a destructor on the byte array, if necessary. That's all we need to place our C++ objects within Wren objects.

Here's a small foreign class.

```dart
foreign class Vec3 {
  // this constructor doesn't do anything in wren, so we leave it empty
  // however, we want a constructor to fire in C++!
  construct new(x, y, z) {}

  foreign norm()
  foreign dot( rhs )
  foreign cross( rhs )    // returns the result as a new vector

  // accessors
  foreign x
  foreign x=( rhs )
  foreign y
  foreign y=( rhs )
  foreign z
  foreign z=( rhs )
}
```

Let's look at binding the following C++ struct to `Vec3` in Wren.

```cpp
struct Vec3f {
  union {
    float v[3];
    struct { float x, y, z; };
  };

  Vec3( float x, float y, float z )
  : v{ x, y, z } {}

  Vec3()
  : v{ 0.f, 0.f, 0.f } {}

  float norm() const {
    return sqrt( x*x + y*y + z*z );
  }

  float dot( const Vec3& rhs ) const {
    return x*rhs.x + y*rhs.y + z*rhs.z;
  }

  Vec3 cross( const Vec3& rhs ) const {
    return Vec3 {
      y*rhs.z - z*rhs.y,
      z*rhs.x - x*rhs.z,
      x*rhs.y - y*rhs.x
    };
  }
}; 
```

The binding code is fairly straight forward. We use the slot API like we did before, but this time we also use it to access the foreign byte arrays.

```cpp
// the allocator has the same signature as any other WrenForeignMethodFn
// the allocator gets called during Vec3.new(_,_,_)
void vec3fAllocate(WrenVM* vm) {
  // this function creates a new instance of the foreign class stored 
  // the slot denoted by the third argument, and places the resulting
  // object in the slot denoted by the second argument
  // remember that the receiver (the Vec3 class) is already in slot zero
  // the fourth argument denotes the size of the byte array to be created
  // finally, the function returns the byte array that it just created
  void* bytes = wrenSetSlotNewForeign(vm, 0, 0, sizeof(Vec3f));
  new (bytes) Vec3f(
    wrenGetSlotDouble(vm, 1),
    wrenGetSlotDouble(vm, 2),
    wrenGetSlotDouble(vm, 3)
  );
}

// the finalizer function has a different signature
// no access to the VM is allowed, because when the finalzer is called,
// the VM is in the middle of a garbage collection
void vec3fFinalize(void* bytes) {
  // do nothing
}

void vec3fNorm(WrenVM* vm) {
  // we can access the byte array of the object in slot zero
  const Vec3f* v = (const Vec3f*)wrenGetSlotForeign(vm, 0);
  wrenSetSlotDouble(vm, 0, v->norm());
}

void vec3fDot(WrenVM* vm) {
  const Vec3f* lhs = (const Vec3f*)wrenGetSlotForeign(vm, 0);
  const Vec3f* rhs = (const Vec3f*)wrenGetSlotForeign(vm, 1);
  wrenSetSlotDouble(vm, 0, lhs->dot(*rhs));
}

void vec3fCross(WrenVM* vm) {
  const Vec3f* lhs = (const Vec3f*)wrenGetSlotForeign(vm, 0);
  const Vec3f* rhs = (const Vec3f*)wrenGetSlotForeign(vm, 1);
  Vec3f result = lhs->cross(*rhs);
  // we need to return a new Vec3f to wren, so we mimic
  // the constructor here
  void* bytes = wrenSetSlotNewForeign(vm, 0, 0, sizeof(Vec3f));
  new (bytes) Vec3f(result);
}

void vec3fGetX(WrenVM* vm) {
  const Vec3f* v = (const Vec3f*)wrenGetSlotForeign(vm, 0);
  wrenSetSlotDouble(vm, 0, v->x);
}

void vec3fSetX(WrenVM* vm) {
  Vec3f* v = (Vec3f*)wrenGetSlotForeign(vm, 0);
  double newx = wrenGetSlotDouble(vm, 1);
  v->x = newx;
}

// repeat for y and z
```

Next, we need to store our methods, allocators and finalizers in their respective data structures, as well as provide Wren a callback for locating the allocators and finalizers based on the module and class name. The class-finder will work much the same way as our method-finder did earlier. Here's what the rest of the program now looks like:

```cpp
// note that according to the "vector" module name, our Vec3 definition
// now lies in the file vector.wren
std::map<std::string, WrenForeignMethodFn> boundForeignMethods{
  { "vectorVec3norm()",   vec3fNorm },
  { "vectorVec3dot(_)",   vec3fDot },
  { "vectorVec3cross(_)", vec3fCross },
  { "vectorVec3x",        vec3fGetX },
  { "vectorVec3x=(_)",    vec3fSetX},
  { "vectorVec3y",        vec3fGetY },
  { "vectorVec3y=(_)",    vec3fSetY },
  { "vectorVec3z",        vec3fGetZ },
  { "vectorVec3z=(_)",    vec3fSetZ }
};

WrenForeignMethodFn bindForeignMethod(WrenVM*,
                                      const char* module,
                                      const char* className,
                                      bool isStatic,
                                      const char* signature
                                      ) {
  // as before
}

// WrenForeignClassMethods is just a struct containing a
// WrenForeignMethodFn, and a WrenFinalizerFn
std::map<std::string, WrenForeignClassMethods> boundForeignClasses{
  { "vectorVec3", { vec3fAllocate, vec3fFinalize } }
};

WrenForeignClassFn bindForeignClass(WrenVM*,
                                    const char* module,
                                    const char* className
                                    ) {
  std::string identifier{ module };
  name += className;
  auto it = boundForeignClasses.find(identifier);
  if (it != boundForeignClasses.end()) {
    return it->second;
  }
  return WrenForeignClassMethods{ nullptr, nullptr };
}

int main() {
  WrenConfiguration configuration;
  wrenInitConfiguration(&configuration);
  configuration.bindForeignMethodFn = bindForeignMethod;
  configuration.bindForeignClassFn = bindForeignClass;
  configuration.writeFn = write;
  configuration.loadModuleFn = loadModule;

  WrenVM* vm = wrenNewVM(&configuration);
  wrenInterpret(
    vm,
    "import \"vector\" for Vec3     \n"
    "var v = Vec3.new(1.0, 2.0, 3.0)\n"
    "var norm = v.norm()            \n"
  );
  wrenFreeVM(vm);
  return 0;
}
```

There! With fairly minimal amounts of boiler-plate, you've wrapped a C++ type for Wren to access. Compared to e.g. Python's embedding API, this is smooth sailing.

If you have used templates in the past, then you probably were already thinking about ways to generate the above binding functions automatically via template parameters. Especially the allocator and finalizer functions are just begging to be templatized:

```cpp
template<typename T>
void allocateType(WrenVM* vm) {
  void* bytes = wrenSetSlotNewForeign(vm, 0, 0, sizeof(T));
  new (bytes) T();
}

template<typename T>
void finalizeType(void* bytes) {
  T* type = (T*)bytes;
  type->~T();
}
```

Likewise, the foreign method wrappers can be parametrized by the class type as well as the method itself. In this way a unique wrapping function is automatically generated for each method, which we just did by hand. The tricky part is automatically handling arguments and return types. You need to deduce which Wren slot API function to call based on the type of argument -- and then generate code which calls it. It turns out that with C++14's metaprogramming features, that's possible with a surprisingly minimal amount of code. I wrote a separate [blog post](http://www.johannmuszynski.com/blog/post/modern_c_templates_are_your_friend/) about that a while back, when I was trying to figure it out for myself. The "toy problem" that the post is centered around is a template wrapper around Wren's slot API:

```cpp
// this isn't exactly the same as in the post because both Wren's C API
// and my code worked differently back when the post was written
template<typename T>
struct WrenStack;

template<>
struct WrenStack<double> {
    // use this instead of GetArgument<>
    static double get(WrenVM* vm, int slot) {
        return wrenGetSlotDouble( vm, slot);
    }

    // use this instead of Return<>
    static void set(WrenVM* vm, int slot, double val) {
        wrenSetSlotDouble(vm, slot, val);
    }
};

template<>
struct WrenStack<bool> {
    static bool get( WrenVM* vm, int slot ) {
        return wrenGetSlotBool(vm, slot);
    }

    static void set(WrenVM* vm, int slot, bool val) {
        wrenSetSlotBool(vm, slot, val);
    }
};

// etc.
```

These structs, along with the compile-time index sequence method from my other blog post, can be used to wrap any function or method with the correct wrenGetSlot/wrenSetSlot function calls.

### Calling Wren code from C++

Wren allows you to call any method from C++. Arguments are passed and return values received as usual with the slot API. Let's look at how that's done by calling some of the foreign math methods that we bound to wren earlier.

```cpp
// get the variable based on the module it is in
// and the variable name
// finally, place it in slot zero
wrenGetVariable(vm, "main", "Math", 0);
WrenValue* receiver = wrenGetSlotValue(vm, 0);
WrenValue* method = wrenMakeCallHandle(vm, "cos(_)");
```

Before we can call the method handle, we need to set up the stack via the slot API. The receiver (the object or class whose method we are calling) needs to be placed in slot zero. Any arguments need to be placed in the following slots.

```cpp
// place the receiver in slot 0
wrenSetSlotValue(vm, 0, receiver);
wrenSetSlotDouble(vm, 1, M_PI / 2.0);
// now the stack is set -- call the method!
wrenCall(vm, method);
```

The return value is now in slot zero. We know that the return value is a double in this case. But since Wren is dynamically typed, potentially anything could be returned from a method. The C API provides a function to check what type of value is in a slot. Let's make sure that it is indeed a double.

```cpp
WrenType returnType = wrenGetSlotType(vm, 0);
assert(returnType == WREN_TYPE_NUM);
double res = wrenGetSlotDouble(vm, 0);
```

`WrenType` is just an enum with the following values:

```c
typedef enum
{
  WREN_TYPE_BOOL,
  WREN_TYPE_NUM,
  WREN_TYPE_FOREIGN,
  WREN_TYPE_LIST,
  WREN_TYPE_NULL,
  WREN_TYPE_STRING,
  
  // The object is of a type that isn't accessible by the C API.
  WREN_TYPE_UNKNOWN
} WrenType;
```

It's also completely possible to wrap method calls in template magic. The technique for doing this was not something covered in the previous blog post. The basic problem is the following. We have a function which takes the method call arguments as a template parameter pack. Our receiver and method pointers are set up. We need to call the appropriate `wrenSetSlot*` functions in order per function argument.

```cpp
struct WrenMethod {
  WrenValue* receiver;
  WrenValue* method;
  WrenVM*    vm;
  
  template<typename... Arg>
  void call(Arg&&... arg) {
    // how do we write a sequence of statements containing
    // WrenStack<Arg>::set(vm, slot, std::forward<Arg>(arg))?
    // We can't just expand the parameter pack directly within
    // the function's block scope
  }
};
```

The idea is to use C++11 initializer lists, and expand our calls to `WrenStack<T>::set` within the list. We need a dummy type which can be list-initialized.

```cpp
struct ExpandType {
  template<typename... T>
  ExpandType(T&& ...) {}
};
```

Everything you need to know about expanding parameter packs within dummy initializer lists is explained beautifully in this [StackOverflow post](http://stackoverflow.com/questions/17339789/how-to-call-a-function-on-all-variadic-template-args).

Using the index-list technique from my other blog post, the automatic method calling code will look something like this.

```cpp
template<typename... Arg, std::size_t... index>
void passArgumentsToWren(WrenVM* vm, 
                         const std::tuple<Arg...>& tuple,
                         std::index_sequence< index... >) {
    using Traits = ParameterPackTraits<Arg...>;
    ExpandType{
        0,
        (WrenStack<typename Traits::template ParameterType<index>>::set(
            vm,
            index + 1,
            std::get<index>(tuple)
        ), 0)...
    };
}

struct WrenMethod {
  WrenValue* receiver;
  WrenValue* method;
  WrenVM*    vm;
  
  template<typename... Arg>
  void call(Arg&&... arg) {
    constexpr std::size_t arity = sizeof...(Arg);
    wrenSetSlotValue(vm, 0, receiver);
    std::tuple<Arg...> tuple = std::make_tuple(arg...);
    passArgumentsToWren(vm, tuple, std::make_index_sequence<arity>{});
    wrenCall(vm, method);
  }
};
```

The beauty of templates is the fact that even though the above looks like it contains lots of overhead code, it is only compile-time overhead. After being compiled, only the correct sequence of `wrenSetSlot` statements will remain.

### Next

You've now seen some of the functionality that the Wren C API offers. Not everything was covered here; the Wren slot API contains functions for storing values directly in list elements, as well as storing null in a slot. More functions will probably be added in the future -- these will perhaps be visited upon in a future post once the API evolves sufficiently.

You can read about Wren++ in part 2.
