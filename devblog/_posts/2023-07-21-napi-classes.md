---
layout: post
title: "Creating a NodeJS Class, the cool way"
date: 2023-07-21 00:00:00 -0300
categories: nodejs
---

# ...or how Linus Torvalds would make a NodeJS class

## Index

- [Introduction](#introduction)
- [Node-API](#node-api)
    - [Constructor](#constructor)
    - [Creating a class](#creating-a-class)
    - [Constructor](#constructor)
    - [Getters](#getter)
    - [Setters](#setter)
    - [Finalizer](#finalizer)
- [Conclusion](#conclusion)
- [Final Code](#final-code)

## Introduction

JavaScript is a very flexible language, and it allows us to do a lot of things. One of the things that we can do is to create classes, and use them as we would in other languages. You can create a class very trivially, like this:

```javascript
class MyClass {
    constructor() {
        this.myProperty = 0;
    }
}
```

By using the keyword `class`, one may create a class with a given name (in this case, `MyClass`). The constructor is a special function that is called when an instance of the class is created. Classes are the "Blueprints" of objects, every time you need a new object, you create a new instance of the class. The constructor is the function that is called when you create a new instance of the class. In this case, the constructor sets the property `myProperty` to 0.

```javascript
let myInstance = new MyClass();
```

This is how you create a new instance of the class. You can access the property `myProperty` like this:

```javascript
console.log(myInstance.myProperty);
```

Of course, you can add all sort of methods, like getters/setters. This post is not about that, though. This common way is very boring and straightforward. What if we could do something more interesting?

## Node-API

Node-API is a project that aims to provide a stable API for native modules. It is a C API that is compatible with NodeJS, and it is used by many projects, including NodeJS itself. It is a very interesting project, and it allows us to do some cool things. One of the things that it allows us to do is to create JavaScript classes directly from C, and use them in JavaScript. This is what we are going to do.

First of all, we need to create a new NodeJS project, more specifically a node add-on, a native piece of code that can be used along normal JS code. You can do that by running `npm init` in an empty directory.

Then, we need to install the `node-addon-api` package. This package provides the Node-API myaddons for NodeJS. You can install it by running `npm install node-addon-api`. Now, we need to create a new file, called `myaddon.gyp`. This file is used by NodeJS to build native modules. This is how it looks like:

```gyp
{
  "targets": [
    {
      "target_name": "myaddon",
      "sources": [ "main.c" ]
    }
  ],
}
```

This file tells NodeJS to build a module called `myaddon`, using the file `main.c` as the source. Now, we need to create the file `main.c` and include the napi header file. This is how it looks like:

```c
#include <node/node_api.h>
// OR, check which one works for you
#include <napi.h>
```

Now, we need to create a function that will be called when the module is loaded. This function will be called `init`, and it will be exported by the module.

```c
// Module initialization logic
napi_value Init(napi_env env, napi_value exports) {
    return exports;
}
```

It doesn't do anything, but it is a good start. Now, we need to tell NodeJS to export this function.

```c
NAPI_MODULE(NODE_GYP_MODULE_NAME, Init)
```

This will export the function `Init` as the module initialization function. Now, we need to build the module. You can do that by running `node-gyp configure build`. This will create a new folder called `build`, and inside it, you will find the file `myaddon.node`. This is the file that we need to load in NodeJS. We can do that by creating a new file called `index.js` and adding the following code:

```javascript
const myaddon = require('./build/Release/myaddon.node');
```

This will load the module that we just created and store it in the variable `myaddon`. Now, we can run the file by running `node index.js`. This will print nothing, because we didn't do anything. Let's change that.

Let's make things spicier. Let's create a function that will write a message to the console, but in C. This is how it looks like:

```c
napi_value PrintHello(napi_env env, napi_callback_info info) {
    napi_status status;
    napi_value result;
    status = napi_create_string_utf8(env, "Hello World!", NAPI_AUTO_LENGTH, &result);
    if (status != napi_ok) return NULL;
    printf("Hello World!\n");
    return result;
}
```

This function will create a new string, and print it to the console. Now, we need to export this function. We can do that by adding the following code to the `Init` function:

```c
napi_value PrintHello(napi_env env, napi_callback_info info);
napi_value Init(napi_env env, napi_value exports) {
    napi_status status;
    napi_value fn;
    status = napi_create_function(env, NULL, 0, PrintHello, NULL, &fn);
    if (status != napi_ok) return NULL;
    status = napi_set_named_property(env, exports, "printHello", fn);
    if (status != napi_ok) return NULL;
    return exports;
}
```

We should place all the symbols we want to export in `exports`, so node can find them. Now, we need to build the module again, and load it in NodeJS. We can do that by running `node-gyp build` and then `node index.js`. This will print `Hello World!` to the console.

```javascript
// This will return the string "Hello World!" too!
myaddon.printHello();
```

What if I want the string to be passed from JavaScript to C? We can do that by adding a parameter to the function. This is how it looks like:

```c
napi_value PrintHello(napi_env env, napi_callback_info info) {
    napi_status status;
    napi_value result;
    size_t argc = 1;
    napi_value argv[1];
    status = napi_get_cb_info(env, info, &argc, argv, NULL, NULL);
    if (status != napi_ok) return NULL;
    char str[100];
    size_t str_len;
    status = napi_get_value_string_utf8(env, argv[0], str, 100, &str_len);
    if (status != napi_ok) return NULL;
    printf("%s\n", str);
    status = napi_create_string_utf8(env, str, str_len, &result);
    if (status != napi_ok) return NULL;
    return result;
}
```

This function will get the first argument passed to it, and print it to the console. In node all parameters are passed by the napi_env parameter, and we use the napi_get_* to access the data. This is done that way, because any ABI changes won't affect the application. Now, we need to build the module again, and load it in NodeJS. We can do that by running `node-gyp build` and then `node index.js`. This will print the string passed to the function to the console.

```javascript
// This will return the string "Hello World!" too!
myaddon.printHello("Hello World!");
```

### Creating a class

Ok, where's the class thing? We are getting there. Let's create a new file called `class.c`. This file will contain the class definition. [This is how it looks like](#final-code) (**caution** - long scary C code):

There's a lot in here, so let by parts.

```c
// Define the C class structure
typedef struct
{
    int value;
} MyClass;
```

Here, we are keeping a C representation of the class's state. I'm only storing the class property `value`, methods are not stored here.

### Constructor

```c
// Constructor for the class
napi_value Constructor(napi_env env, napi_callback_info info)
```

This is the constructor for our class, meaning that this will be called every time someone creates a new instance of the class. This is where we allocate resources, perform property initialization, and associate the C class instance with the JavaScript class instance. It
should return the newly created instance.

The first thing we need is to allocate memory for our class instance. This is done with `malloc` and `free` in the constructor and destructor respectively. Notice how we are using `malloc` to allocate memory for the class instance, and `free` to release it. This is because we are using C, and not C++. If you are using C++, you should use `new` and `delete` instead.

```c
napi_status status;

MyClass *myClass = (MyClass *)malloc(sizeof(MyClass));
```

We check if the memory allocation was successful, and if not, we throw an error and return `NULL`. We've seen `napi_throw_error` before, it throws a runtime error to the caller, that can be caught with a `try/catch` block in JavaScript.

Then we initialize the class instance properties. In this case, we are initializing the `value` property to `0`.

```
    // Initialize class instance properties here (if needed)
    myClass->value = 0;
```

Now we get the instance of our object that we are creating, this is passed as a cb_info.

```c
    napi_value thisArg;
    status = napi_get_cb_info(env, info, NULL, NULL, &thisArg, NULL);
```

Finally, associate the state we created with the JavaScript class instance. This is done with `napi_wrap`. Wrapping a state to a class means that every time a method is called, it will operate on that particular state. We malloc it, because the napi won't copy the state, it will just keep a reference to it. This means that if you use a stack-allocated value, it will be destroyed when the method returns, and the napi will be left with a dangling pointer.

```c
    // Associate the MyClass instance with the JavaScript class
    status = napi_wrap(env, thisArg, myClass, NULL, NULL, NULL);
```

If everything went well, we return the instance.

```c
    return thisArg;
```

### Setter

```c
// Method that updates the value property of the class
napi_value UpdateValue(napi_env env, napi_callback_info info)
```

This is a setter for `value`, notice how all functions receive the `env` and `info` parameters, and returns `napi_value`. This is actually a type, `napi_callback` and is called every time that v8 will call your code. You'll see this everywhere in napi.

As for this setter, it simply updates the value in our instance. **Important** this operates in a particular instance of the class, whereas the constructor is a static method that is called when the class is created (i.e: have no associated object).

```c
status = napi_get_cb_info(env, info, &argc, args, &thisArg, NULL);
if (status != napi_ok)
{
    napi_throw_error(env, NULL, "Failed to get callback info.");
    return NULL;
}
```

We use this to get the arguments passed to the function, and the `this` object. `ThisArg` is the object we are operating on. In this case, it's the instance of the class.

```c
MyClass *myClass;
status = napi_unwrap(env, thisArg, (void **)&myClass);
```
`napi_unwrap` is used to get the C instance associated with the JavaScript object. This is how we can access the class's state in a C-friendly way. It returns a pointer, so any changes we make to the instance will be reflected in the JavaScript object. You don't need
to worry about synchronization, as v8 will not run code from the same instance in parallel.

If we want to set the value in the instance, we first need the new value. We can get it like this:

```c
int value;
status = napi_get_value_int32(env, args[0], &value);
```

This will get the first argument passed to the function, and convert it to an integer. If the argument is not an integer, it will throw an error. Finally, we set the value in the instance.

```c
myClass->value = value;
```

### Getter

The third function we have here is a getter for the value property. It's very similar to the setter, but it doesn't take any arguments, and it returns the value instead of setting it. Remember: napi is supposed to hide the v8 and node specific details, so your code doesn't depend on a specific version of node or v8. This means that every value from js and to js should be wrapped in the `napi_value` opaque type. This is how we return the value:


```c
    status = napi_get_cb_info(env, info, NULL, NULL, &thisArg, NULL);
    if (status != napi_ok)
    {
        napi_throw_error(env, NULL, "Failed to get callback info.");
        return NULL;
    }
```

Get our context, as usual. Then we need a pointer to our instance's state:
```c
MyClass *myClass;
status = napi_unwrap(env, thisArg, (void **)&myClass);
```

Finally, we create the value to return from the instance's state:

```c
    status = napi_create_int32(env, myClass->value, &result);
    if (status != napi_ok)
    {
        napi_throw_error(env, NULL, "Failed to create result value.");
        return NULL;
    }

    return result;
```

### Module initialization

The initialization is straightforward to what we've seen before. We define the class, and set it as a property of the exports object.

Those are the associated methods for the class. We need to define them in the class descriptor. Remember when we defined the class state, and only stored the value property? This is where we define the methods. The descriptor is an array of `napi_property_descriptor` struct, that define the name of the method, the function that implements it, and some flags. We are only interested in the name and the function, so we set the rest to `NULL`.
```c
    napi_status status;
    napi_property_descriptor desc[] = {
        {"constructor", NULL, Constructor, NULL, NULL, NULL, napi_default, NULL},
        {"updateValue", NULL, UpdateValue, NULL, NULL, NULL, napi_default, NULL},
        {"getValue", NULL, GetValue, NULL, NULL, NULL, napi_default, NULL},
    };
```
Now we build the class itself, giving it the name, the constructor, the number of methods, the descriptor, and the resulting value is the class itself.
```c
    napi_value constructor;
    status = napi_define_class(env, "MyClass", NAPI_AUTO_LENGTH, Constructor, NULL, 3, desc, &constructor);
    if (status != napi_ok)
    {
        napi_throw_error(env, NULL, "Failed to define class.");
        return NULL;
    }
```

At this point, our class definition already exists in the v8 engine, but we need to expose it to the JavaScript side. We do this by setting it as a property of the exports object.

This behaves analogously to the `module.exports` object in JavaScript. We set the property name, and the value is the class we just created.

```c
    status = napi_set_named_property(env, exports, "MyClass", constructor);
    if (status != napi_ok)
    {
        napi_throw_error(env, NULL, "Failed to set named property.");
        return NULL;
    }

    return exports;
```

And that's it! We have a class that can be used from JavaScript, and that has a state that can be accessed from C.

### Finalizer

Our constructor `mallocated` our class, we need to free it afterwards. To append a destructor to the class, we would need to use `napi_add_finalizer`. This function takes a pointer to the class, and a function that will be called when the class is garbage collected. Notice that we don't control when the GC calls it, but when it does, we can free the memory. Here's how we would use it to our class:

```c
status = napi_add_finalizer(env, constructor, NULL, Finalize, NULL, NULL);
```

The finalizer function is a little different from our usual `napi_callback`. It takes a `napi_env`, the object that is being garbage collected, and a `void*` that we can use to pass data to the finalizer. We can use this to pass a pointer to the class state, and free it in the finalizer. Here's how we would do it:

```c
napi_value Finalize(napi_env env, void *finalize_data, void *finalize_hint)
{
    MyClass *myClass = (MyClass *)finalize_data;
    free(myClass);
    return NULL;
}
```

## Conclusion

I've decided to make this small write-up because I'm having fun with napi, and I think it's a great way to write native modules for node. I'm writing native bindings to [librustreexo](https://github.com:mit-dci/rustreexo), a rust library that implements the [Utreexo accumulator](https://github.com/utreexo/utreexo).

In general, I'm liking napi. It's a bit verbose, but it's very clear and easy to understand. I'm not sure how it will perform, but I'm not expecting it to be a bottleneck. The calling convention doesn't seen to be very costly. I'll write a follow-up post when I have something working, probably with some benchmarks.

## Final code

```c
#include <node/node_api.h>
#include <stdlib.h>

// Define the C class structure
typedef struct
{
    int value;
} MyClass;

// Constructor for the class
napi_value Constructor(napi_env env, napi_callback_info info)
{
    napi_status status;

    MyClass *myClass = (MyClass *)malloc(sizeof(MyClass));
    if (myClass == NULL)
    {
        napi_throw_error(env, NULL, "Memory allocation failed.");
        return NULL;
    }

    // Initialize class instance properties here (if needed)
    myClass->value = 0;

    napi_value thisArg;
    status = napi_get_cb_info(env, info, NULL, NULL, &thisArg, NULL);
    if (status != napi_ok)
    {
        napi_throw_error(env, NULL, "Failed to get callback info.");
        free(myClass);
        return NULL;
    }

    // Associate the MyClass instance with the JavaScript class
    status = napi_wrap(env, thisArg, myClass, NULL, NULL, NULL);
    if (status != napi_ok)
    {
        napi_throw_error(env, NULL, "Failed to wrap native instance.");
        free(myClass);
        return NULL;
    }

    return thisArg;
}

// Method that updates the value property of the class
napi_value UpdateValue(napi_env env, napi_callback_info info)
{
    napi_status status;
    size_t argc = 1;
    napi_value args[1];
    napi_value thisArg;

    status = napi_get_cb_info(env, info, &argc, args, &thisArg, NULL);
    if (status != napi_ok)
    {
        napi_throw_error(env, NULL, "Failed to get callback info.");
        return NULL;
    }

    if (argc < 1)
    {
        napi_throw_error(env, NULL, "Wrong number of arguments. Function expects one argument.");
        return NULL;
    }

    MyClass *myClass;
    status = napi_unwrap(env, thisArg, (void **)&myClass);
    if (status != napi_ok)
    {
        napi_throw_error(env, NULL, "Failed to unwrap native instance.");
        return NULL;
    }

    int value;
    status = napi_get_value_int32(env, args[0], &value);
    if (status != napi_ok)
    {
        napi_throw_error(env, NULL, "Invalid argument. Function expects an integer.");
        return NULL;
    }

    myClass->value = value;

    return NULL;
}

// Method that retrieves the value property of the class
napi_value GetValue(napi_env env, napi_callback_info info)
{
    napi_status status;
    napi_value result;
    napi_value thisArg;

    status = napi_get_cb_info(env, info, NULL, NULL, &thisArg, NULL);
    if (status != napi_ok)
    {
        napi_throw_error(env, NULL, "Failed to get callback info.");
        return NULL;
    }

    MyClass *myClass;
    status = napi_unwrap(env, thisArg, (void **)&myClass);
    if (status != napi_ok)
    {
        napi_throw_error(env, NULL, "Failed to unwrap native instance.");
        return NULL;
    }

    status = napi_create_int32(env, myClass->value, &result);
    if (status != napi_ok)
    {
        napi_throw_error(env, NULL, "Failed to create result value.");
        return NULL;
    }

    return result;
}

// Module initialization logic
napi_value Init(napi_env env, napi_value exports)
{
    napi_status status;
    napi_property_descriptor desc[] = {
        {"constructor", NULL, Constructor, NULL, NULL, NULL, napi_default, NULL},
        {"updateValue", NULL, UpdateValue, NULL, NULL, NULL, napi_default, NULL},
        {"getValue", NULL, GetValue, NULL, NULL, NULL, napi_default, NULL},
    };

    napi_value constructor;
    status = napi_define_class(env, "MyClass", NAPI_AUTO_LENGTH, Constructor, NULL, 3, desc, &constructor);
    if (status != napi_ok)
    {
        napi_throw_error(env, NULL, "Failed to define class.");
        return NULL;
    }

    status = napi_set_named_property(env, exports, "MyClass", constructor);
    if (status != napi_ok)
    {
        napi_throw_error(env, NULL, "Failed to set named property.");
        return NULL;
    }

    return exports;
}

NAPI_MODULE(NODE_GYP_MODULE_NAME, Init)
```
