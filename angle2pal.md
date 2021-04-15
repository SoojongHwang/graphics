# eglCreateImageKHR

```c
EGLImage eglCreateImage(
    EGLDisplay display,
    EGLContext context,
    EGLenum target,
    EGLClientBuffer buffer,
    const EGLAttrib *attrib_list
 );
```

* 외부로부터의 resource 를 client API 들간에 공유되기 위한 용도로 주로 사용됨
  * CPU 에서는 buffer_handle_t lock/unlock 을 통한 memory manipulation
  * OpenGL에서는 texture 로서 bind 하여 사용가능
  * OpenCL 에서도 clImportMemory 등을 통하여 buffer 접근 가능
* 때문에 extenal buffer 에 대한 memory 를 allocate 하는곳이 있을거라 생각하고 따라가봄

## Path

* (entrypoints) libGLESv2
  * 시작 (https://cs.android.com/android/platform/superproject/+/master:external/angle/src/libGLESv2/entry_points_egl_autogen.cpp;l=630?hl=ko)
  * 끝 (https://cs.android.com/android/platform/superproject/+/master:external/angle/src/libGLESv2/egl_stubs.cpp;l=149;drc=master?hl=ko)
* libAngle
  * 시작 (https://cs.android.com/android/platform/superproject/+/master:external/angle/src/libANGLE/Display.cpp;drc=master;l=1108?hl=ko)
  * 끝(https://cs.android.com/android/platform/superproject/+/master:external/angle/src/libANGLE/Image.cpp;l=170;drc=master?hl=ko)
* (backend) vulkan
  * 시작 (https://cs.android.com/android/platform/superproject/+/master:external/angle/src/libANGLE/renderer/vulkan/ImageVk.cpp;l=53?hl=ko)
  * 끝 (https://cs.android.com/android/platform/superproject/+/master:external/angle/src/libANGLE/renderer/vulkan/ImageVk.cpp;l=89?hl=ko)
* (android backend)
  * 시작 (https://cs.android.com/android/platform/superproject/+/master:external/angle/src/libANGLE/renderer/vulkan/android/HardwareBufferImageSiblingVkAndroid.cpp;l=139?hl=ko)

# eglSwapBuffers
