# 打开linker日志

## Enable logging of dlopen/dlsym and library loading errors for apps (Available in Android O)

Starting with Android O it is possible to enable logging of dynamic
linker activity for debuggable apps by setting a property corresponding
to the fully-qualified name of the specific app:
针对单个应用

```shell
adb shell setprop debug.ld.app.com.example.myapp dlerror,dlopen,dlsym
adb logcat
```

Any combination of `dlerror`, `dlopen`, and `dlsym` can be used. There's
no separate `dlclose` option: `dlopen` covers both loading and unloading
of libraries. Note also that `dlerror` doesn't correspond to actual
calls of dlerror(3) but to any time the dynamic linker writes to its
internal error buffer, so you'll see any errors the dynamic linker would
have reported, even if the code you're debugging doesn't actually call
dlerror(3) itself.

On userdebug and eng builds it is possible to enable tracing for the
whole system by using the `debug.ld.all` system property instead of
app-specific one. For example, to enable logging of all dlopen(3)
(and thus dclose(3)) calls, and all failures, but not dlsym(3) calls:

全局生效

```shell
adb shell setprop debug.ld.all dlerror,dlopen
```

## 重新编译ROM

``init.environ.rc.in``添加``export LD_DEBUG 3``，使得代码``bionic/linker/linker_debug.h``中的``g_ld_debug_verbosity``值变化，从而输出日志。