---
name: frida-hooks-javascript
description: Create and debug Frida hook scripts for dynamic instrumentation with JavaScript
keywords: [frida, instrumentation, reverse-engineering, hooks, javascript]
version: 1.0.0
---

# Frida Hooks JavaScript Skill

This skill helps create, structure, and debug Frida hook scripts written in JavaScript. Frida is a dynamic instrumentation toolkit that allows you to inject code into live processes to observe and modify behavior at runtime.

## What is Frida?

Frida is a dynamic instrumentation framework that enables you to:
- Hook functions and methods in running processes
- Intercept and modify function arguments and return values
- Trace execution flow and inspect memory
- Test security properties and reverse-engineer binaries
- Monitor API calls and system behavior

## When to Use This Skill

Use this skill when you need to:
- Create JavaScript-based Frida hook scripts
- Hook native functions or JavaScript methods
- Intercept API calls and modify behavior
- Build custom instrumentation for testing or analysis
- Debug complex application behavior at runtime

## Frida Hook Script Structure

### Basic Template

All Frida JavaScript hook scripts follow this structure:

```javascript
// 1. Define target to instrument
const moduleName = "targetModule";

// 2. Define hook callbacks
const hooks = {
  functionName: {
    onEnter: function(args) {
      // Called when function is entered
      // Log arguments, inspect memory, etc.
      console.log("Arguments:", args);
    },
    onLeave: function(retval) {
      // Called when function returns
      // Inspect return value, modify behavior, etc.
      console.log("Return value:", retval);
      return retval;
    }
  }
};

// 3. Attach hooks to target functions
Object.keys(hooks).forEach(function(funcName) {
  const address = Module.findExportByName(moduleName, funcName);
  if (address) {
    Interceptor.attach(address, hooks[funcName]);
  } else {
    console.log("Function not found:", funcName);
  }
});
```

## Core Concepts

### 1. Finding Functions

**Export-based lookup** (simplest):
```javascript
const addr = Module.findExportByName("libc.so", "malloc");
```

**Pattern matching** (for internal functions):
```javascript
const pattern = "48 8d 15 ? ? ? ? 48 39 c2"; // x86-64 pattern
const matches = Memory.scanSync(module.base, module.size, pattern);
matches.forEach(function(match) {
  console.log("Found at:", match.address);
});
```

**Symbol-based** (when symbols available):
```javascript
const symbols = DebugSymbol.findFunctionsMatching("*crypto*");
symbols.forEach(function(sym) {
  console.log("Symbol:", sym.name, "at", sym.address);
});
```

### 2. Intercepting Calls

**Basic interception**:
```javascript
const target = Module.findExportByName("libc.so", "strlen");
Interceptor.attach(target, {
  onEnter: function(args) {
    const str = args[0].readCString();
    console.log("strlen called with:", str);
  },
  onLeave: function(retval) {
    console.log("strlen returns:", retval);
  }
});
```

**Modifying arguments**:
```javascript
Interceptor.attach(target, {
  onEnter: function(args) {
    // Replace first argument
    args[0] = ptr("0x12345678");
  }
});
```

**Replacing return values**:
```javascript
Interceptor.attach(target, {
  onLeave: function(retval) {
    // Force return value
    return ptr("1");
  }
});
```

### 3. Working with Native Data Types

**Reading memory**:
```javascript
const ptr = args[0]; // pointer argument

// Read as string
const str = ptr.readCString();

// Read as struct
const data = ptr.readByteArray(64);

// Read as integer
const value = ptr.readInt();

// Read as UTF-16
const wide = ptr.readUtf16String();
```

**Writing memory**:
```javascript
const target = Memory.alloc(256);
target.writeUtf8String("Hello");
target.add(32).writeInt(42);
```

### 4. Calling Native Functions

**Calling existing functions**:
```javascript
const libc = Module.findBaseAddress("libc.so");
const sprintf = new NativeFunction(
  Module.findExportByName("libc.so", "sprintf"),
  "int",
  ["pointer", "pointer", "int"]
);

const buffer = Memory.alloc(256);
sprintf(buffer, ptr(0x1234), 42);
const result = buffer.readCString();
```

**Creating callbacks**:
```javascript
const callback = new NativeCallback(function(x, y) {
  return x + y;
}, "int", ["int", "int"]);
```

## Common Patterns

### Pattern 1: Tracing Function Calls

```javascript
const Module = Process.enumerateModules()[0];
const target = Module.findExportByName("mylib.so", "processData");

Interceptor.attach(target, {
  onEnter: function(args) {
    console.log("[+] processData called");
    console.log("    arg0 (buffer):", args[0]);
    console.log("    arg1 (size):", args[1]);
    
    const data = args[0].readByteArray(parseInt(args[1]));
    console.log("    data:", data);
  },
  onLeave: function(retval) {
    console.log("[-] processData returns:", retval);
  }
});
```

### Pattern 2: Method Hooking (JavaScript Context)

```javascript
// Hook Java methods (Android)
Java.perform(function() {
  const ActivityClass = Java.use("android.app.Activity");
  ActivityClass.onCreate.implementation = function(savedInstanceState) {
    console.log("Activity.onCreate called");
    this.onCreate(savedInstanceState);
  };
});

// Hook ObjC methods (iOS)
ObjC.chooseSync(ObjC.classes.NSURLRequest).forEach(function(instance) {
  console.log("NSURLRequest:", instance.URL().toString());
});
```

### Pattern 3: Monitoring System Calls

```javascript
const open = Module.findExportByName("libc.so", "open");
Interceptor.attach(open, {
  onEnter: function(args) {
    const filename = args[0].readCString();
    const flags = args[1].toInt32();
    console.log("open():", filename, "flags:", flags);
  }
});
```

### Pattern 4: Credential/Secret Interception

```javascript
const authenticate = Module.findExportByName("myapp.so", "authenticate");
Interceptor.attach(authenticate, {
  onEnter: function(args) {
    const username = args[0].readCString();
    const password = args[1].readCString();
    console.log("[!] Credentials:", username, password);
  }
});
```

### Pattern 5: Bypassing Checks

```javascript
const validateLicense = Module.findExportByName("myapp.so", "isValidLicense");
Interceptor.replace(validateLicense, new NativeCallback(function() {
  console.log("[!] License check bypassed");
  return 1; // Always return true
}, "int", []));
```

## Best Practices

### 1. Error Handling

```javascript
try {
  const target = Module.findExportByName("mylib.so", "criticalFunction");
  if (!target) {
    throw new Error("Function not found");
  }
  Interceptor.attach(target, hooks);
} catch (e) {
  console.error("[!] Error:", e.message);
}
```

### 2. Memory Safety

```javascript
// Always validate pointers before reading
function safeReadString(ptr, maxLength) {
  if (!ptr || ptr.isNull()) {
    return "[null]";
  }
  try {
    return ptr.readCString(maxLength || 256);
  } catch (e) {
    return "[invalid pointer]";
  }
}
```

### 3. Performance Considerations

```javascript
// Use callbacks sparingly for high-frequency functions
let callCount = 0;
const SAMPLE_RATE = 1000; // Log every 1000th call

Interceptor.attach(target, {
  onEnter: function(args) {
    if (callCount++ % SAMPLE_RATE === 0) {
      console.log("Sample:", args[0]);
    }
  }
});
```

### 4. Organizing Complex Scripts

```javascript
// Group related hooks by module
const hooks = {
  cryptography: {
    "crypto_hash": { /* hook */ },
    "crypto_verify": { /* hook */ }
  },
  networking: {
    "socket": { /* hook */ },
    "send": { /* hook */ }
  }
};

// Attach all hooks
Object.keys(hooks).forEach(function(category) {
  Object.keys(hooks[category]).forEach(function(funcName) {
    const addr = Module.findExportByName("mylib.so", funcName);
    if (addr) {
      Interceptor.attach(addr, hooks[category][funcName]);
    }
  });
});
```

## Running Frida Scripts

### Using Frida CLI

```bash
# Attach to running process
frida -n "processName" -l script.js

# Attach to specific PID
frida -p 1234 -l script.js

# Start process and inject
frida -f com.example.app -l script.js

# Python script wrapper
frida-python -f "binary" -l script.js
```

### Using Frida Python

```python
import frida

# Load script
with open("script.js") as f:
    script_code = f.read()

# Attach to process
device = frida.get_local_device()
process = device.attach("processName")
script = process.create_script(script_code)

# Handle messages from script
def on_message(message, data):
    if message['type'] == 'send':
        print(message['payload'])

script.on('message', on_message)
script.load()
input("Press Enter to exit...")
```

## Debugging Tips

1. **Test in isolation**: Create minimal test scripts first
2. **Log extensively**: Use console.log() for debugging
3. **Validate assumptions**: Check that functions exist before hooking
4. **Monitor resources**: High-frequency hooks can impact performance
5. **Use Frida console**: `frida -n processName` for interactive debugging
6. **Check module layout**: Use `frida-trace` to list functions before writing hooks

## References

- [Frida Documentation](https://frida.re/docs/home/)
- [Frida JavaScript API](https://frida.re/docs/javascript-api/)
- [Frida Examples](https://github.com/frida/frida/tree/main/tests/data)
- [OWASP Mobile Security Testing Guide](https://owasp.org/www-project-mobile-security-testing-guide/)
