# Libcore-Syscall / Libcore-ElfLoader

Libcore-Syscall is a Java library for Android that allows you to make any Linux system calls and/or load any ELF shared objects directly from Java code.

## Features

- Support Android 5.0 - 15
- Support any system calls (as long as they are permitted by the seccomp filter)
- Support loading any ELF shared objects (lib*.so) directly from memory
- Implemented in 100% pure Java 1.8
- No shared libraries (lib*.so) are shipped with the library
- No `System.loadLibrary` or `System.load` is used
- No temporary files are created on the disk (does not require a writable path/mount point)
- Small, no dependencies

## Usage

The library provides the following classes:

- MemoryAccess/MemoryAllocator: Allocate and read/write native memory.
- NativeAccess: Register JNI methods, or call native functions (such as `dlopen`, `dlsym`, etc.) directly.
- Syscall: Make any Linux system calls.
- DlExtLibraryLoader: Load any ELF shared objects (lib*.so) directly from memory.

## Examples

Here are some examples of possible use cases.

### Load ELF Shared Object from Memory

Here is an example of how to load an ELF shared object directly from memory.
It loads the `libmmkv.so` shared object and calls the `MMKV.initialize` method.

See [TestNativeLoader.java](demo-app/src/main/java/com/example/test/app/TestNativeLoader.java) for the complete example.

```java
import com.tencent.mmkv.MMKV;

import dev.tmpfs.libcoresyscall.core.NativeAccess;
import dev.tmpfs.libcoresyscall.elfloader.DlExtLibraryLoader;

public static long initializeMMKV(@NonNull Context ctx) {
    String soname = "libmmkv.so";
    // get the ELF data from somewhere
    byte[] elfData = getElfData(soname);

    // load the ELF shared object from byte array
    // if it fails, it throws an UnsatisfiedLinkError
    long sHandle = DlExtLibraryLoader.dlopenExtFromMemory(elfData, soname, DlExtLibraryLoader.RTLD_NOW, 0, 0);

    // since dlopen from memory is not a standard function, ART does not know it
    // we need to call JNI_OnLoad manually, as if the shared object is loaded by System.loadLibrary
    long jniOnLoad = DlExtLibraryLoader.dlsym(sHandle, "JNI_OnLoad");
    if (jniOnLoad != 0) {
        long javaVm = NativeAccess.getJavaVM();
        long ret = NativeAccess.callPointerFunction(jniOnLoad, javaVm, 0);
        if (ret < 0) {
            throw new RuntimeException("JNI_OnLoad failed: " + ret);
        }
    } else {
        // should not happen, MMKV uses JNI_OnLoad to register native methods
        throw new IllegalStateException("JNI_OnLoad not found");
    }
    // initialize MMKV, since we have already loaded the libmmkv.so from memory
    // MMKV does not need to load the libmmkv.so shared object again
    MMKV.initialize(ctx, libName -> {
        // no-op
    });
    return sHandle;
}
```

### Make System Calls

Here is an example of how to make syscalls with the library. It calls the `uname` system call to get the system information.

See [TestMainActivity.java](demo-app/src/main/java/com/example/test/app/TestMainActivity.java) for the complete example.

```java
import dev.tmpfs.libcoresyscall.core.IAllocatedMemory;
import dev.tmpfs.libcoresyscall.core.MemoryAccess;
import dev.tmpfs.libcoresyscall.core.MemoryAllocator;
import dev.tmpfs.libcoresyscall.core.NativeHelper;
import dev.tmpfs.libcoresyscall.core.Syscall;

public String unameDemo() {
    StringBuilder sb = new StringBuilder();
    int __NR_uname;
    switch (NativeHelper.getCurrentRuntimeIsa()) {
        case NativeHelper.ISA_X86_64:
            __NR_uname = 63;
            break;
        case NativeHelper.ISA_ARM64:
            __NR_uname = 160;
            break;
        // add other architectures here ...
    }
    // The struct of utsname can be found in <sys/utsname.h> in the NDK.
    // ...
    int releaseOffset = 65 * 2;
    // ...
    int utsSize = 65 * 6;
    try (IAllocatedMemory uts = MemoryAllocator.allocate(utsSize, true)) {
        long utsAddress = uts.getAddress();
        Syscall.syscall(__NR_uname, utsAddress);
        // ...
        sb.append("release = ").append(MemoryAccess.peekCString(utsAddress + releaseOffset));
        // ...
        sb.append("\n");
    } catch (ErrnoException e) {
        sb.append("ErrnoException: ").append(e.getMessage());
    }
    return sb.toString();
}
```

## The Tricks

- The Android-specific `libcore.io.Memory` and the evil `sun.misc.Unsafe` are used to access the native memory.
- Anonymous executable pages are allocated using the `android.system.Os.mmap` method.
- Native methods are registered with direct access to the `art::ArtMethod::entry_point_from_jni_` field.
- Explanations of the shellcode details can be found in the [shellcode.md](attic/shellcode.md).

## Notice

- This library is not intended to be used in production code. You use it once and it may crash everywhere. It is only for a Proof of Concept.
- This library can only work on ART, not on OpenJDK HotSpot / OpenJ9 / GraalVM.
- The `execmem` SELinux permission is required to allocate anonymous executable memory. Fortunately, this permission is granted to all app domain processes.
- The `system_server` does not have the `execmem` permission. However, this is not true if you have a system-wide Xposed framework installed.

## Build

To build the library:

```shell
./gradlew :core-syscall:assembleDebug
```

To build the demo app:

```shell
./gradlew :demo-app:assembleDebug
```

## Credits

- [pine](https://github.com/canyie/pine)

## License

The library is licensed under the [Apache License, Version 2.0](http://www.apache.org/licenses/LICENSE-2.0).
