# Linker namespace and art VM

## classloader and namespace

```cpp (art/libnativeloader/native_loader.cpp)
jstring CreateClassLoaderNamespace(JNIEnv* env, int32_t target_sdk_version, jobject class_loader,
                                   bool is_shared, jstring dex_path, jstring library_path,
                                   jstring permitted_path) {
#if defined(__ANDROID__)
  std::lock_guard<std::mutex> guard(g_namespaces_mutex);
  auto ns = g_namespaces->Create(env, target_sdk_version, class_loader, is_shared, dex_path,
                                 library_path, permitted_path);
  if (!ns.ok()) {
    return env->NewStringUTF(ns.error().message().c_str());
  }
#else
  UNUSED(env, target_sdk_version, class_loader, is_shared, dex_path, library_path, permitted_path);
#endif
  return nullptr;
}
```