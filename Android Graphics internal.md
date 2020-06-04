# Android Graphics internal

## 1. Allocation

### 1. GraphicBuffer

> frameworks/native/libs/ui/GraphicBuffer.cpp

* function

```C++
GraphicBuffer::GraphicBuffer(uint32_t inWidth, uint32_t inHeight, PixelFormat inFormat,
                             uint32_t inLayerCount, uint64_t inUsage,
                             std::string requestorName): GraphicBuffer() {
    mInitCheck = initWithSize(inWidth, inHeight, inFormat, 
                              inLayerCount, inUsage, std::move(requestorName));
}
```

```C++
status_t GraphicBuffer::initWithSize(uint32_t inWidth, uint32_t inHeight,
        PixelFormat inFormat, uint32_t inLayerCount, uint64_t inUsage,
        std::string requestorName)
{
    GraphicBufferAllocator& allocator = GraphicBufferAllocator::get();
    status_t err = allocator.allocate(inWidth, inHeight, inFormat, inLayerCount,
            inUsage, &handle, &outStride, mId,
            std::move(requestorName));
    ...
}
```



> frameworks/native/libs/ui/Gralloc2.cpp

```C++
status_t Gralloc2Allocator::allocate(std::string /*requestorName*/, uint32_t width, 
                                     uint32_t height, PixelFormat format,
                                     uint32_t layerCount, uint64_t usage,
                                     uint32_t bufferCount, uint32_t* outStride,
                                     buffer_handle_t* outBufferHandles,
                                     bool importBuffers) const {
    IMapper::BufferDescriptorInfo descriptorInfo = {};
    descriptorInfo.width = width;
    descriptorInfo.height = height;
    descriptorInfo.layerCount = layerCount;
    descriptorInfo.format = static_cast<hardware::graphics::common::V1_1::PixelFormat>(format);
    descriptorInfo.usage = usage;

    BufferDescriptor descriptor;
    status_t error = mMapper.createDescriptor(static_cast<void*>(&descriptorInfo),
                                              static_cast<void*>(&descriptor));
    if (error != NO_ERROR) {
        return error;
    }

    auto ret = mAllocator->allocate(descriptor, bufferCount,
                                    [&](const auto& tmpError, const auto& tmpStride,
                                        const auto& tmpBuffers) {
                                        error = static_cast<status_t>(tmpError);
```



```C++
status_t Gralloc2Mapper::createDescriptor(void* bufferDescriptorInfo,
                                          void* outBufferDescriptor) const {
    IMapper::BufferDescriptorInfo* descriptorInfo =
            static_cast<IMapper::BufferDescriptorInfo*>(bufferDescriptorInfo);
    BufferDescriptor* outDescriptor = static_cast<BufferDescriptor*>(outBufferDescriptor);

    status_t status = validateBufferDescriptorInfo(descriptorInfo);
    if (status != NO_ERROR) {
        return status;
    }

    Error error;
    auto hidl_cb = [&](const auto& tmpError, const auto& tmpDescriptor)
                   {
                       error = tmpError;
                       if (error != Error::NONE) {
                           return;
                       }

                       *outDescriptor = tmpDescriptor;
                   };

    hardware::Return<void> ret;
    if (mMapperV2_1 != nullptr) {
        ret = mMapperV2_1->createDescriptor_2_1(*descriptorInfo, hidl_cb);
    } else {
```



## 2. Lock

### 1. GraphicBuffer

> frameworks/native/libs/ui/include/ui/GraphicBuffer.h

* function

```C++
status_t GraphicBuffer::lock(uint32_t inUsage, 
                             void** vaddr, 
                             int32_t* outBytesPerPixel,
                             int32_t* outBytesPerStride)
```



* flow to GraphicBufferMapper

```C++
status_t res = getBufferMapper().lock(handle, inUsage, rect, vaddr, outBytesPerPixel, outBytesPerStride);
```



### 2. GraphicBufferMapper

> frameworks/native/libs/ui/include/ui/GraphicBufferMapper.h

* Constructor

```C++
GraphicBufferMapper::GraphicBufferMapper() {
    mMapper = std::make_unique<const Gralloc4Mapper>();    
    if (mMapper->isLoaded()) {
        mMapperVersion = Version::GRALLOC_4;
        return;
    }
    mMapper = std::make_unique<const Gralloc3Mapper>();
    if (mMapper->isLoaded()) {
        mMapperVersion = Version::GRALLOC_3;
        return;
    }
    mMapper = std::make_unique<const Gralloc2Mapper>();
    if (mMapper->isLoaded()) {
        mMapperVersion = Version::GRALLOC_2;
        return;
    }

    LOG_ALWAYS_FATAL("gralloc-mapper is missing");
}
```



* function

```C++
status_t GraphicBufferMapper::lock(buffer_handle_t handle, 
                                   uint32_t usage, 
                                   const Rect& bounds,
                                   void** vaddr, 
                                   int32_t* outBytesPerPixel,
                                   int32_t* outBytesPerStride)
```



* flow to Gralloc3

```C++
return mMapper->lock(handle, usage, bounds, fenceFd, vaddr, outBytesPerPixel, outBytesPerStride);
```



### 3. Gralloc3 (HIDL Client)

> frameworks/native/libs/ui/include/ui/Gralloc3.h

* Constructor

```C++
using android::hardware::graphics::mapper::V3_0::BufferDescriptor;
using android::hardware::graphics::mapper::V3_0::Error;
using android::hardware::graphics::mapper::V3_0::IMapper;
using android::hardware::graphics::mapper::V3_0::YCbCrLayout

Gralloc3Mapper::Gralloc3Mapper() {
    mMapper = IMapper::getService();
    if (mMapper == nullptr) {
        ALOGW("mapper 3.x is not supported");        return;
    }
    if (mMapper->isRemote()) {
        LOG_ALWAYS_FATAL("gralloc-mapper must be in passthrough mode");
    }
}
```



* function

```C++
status_t lock(buffer_handle_t bufferHandle, 
              uint64_t usage, 
              const Rect& bounds, 
              int acquireFence,
              void** outData, 
              int32_t* outBytesPerPixel,                  
              int32_t* outBytesPerStride) const override;
```



* flow to vendor Gralloc (through HIDL)

```C++
#include <android/hardware/graphics/mapper/3.0/IMapper.h>

auto ret = mMapper->lock(buffer, usage, accessRegion, acquireFenceHandle,
                             [&](const auto& tmpError, const auto& tmpData,
                                 const auto& tmpBytesPerPixel, 
                                 const auto& tmpBytesPerStride) {
                                 error = tmpError;
                                 if (error != Error::NONE) {
                                     return;
                                 }
                                 *outData = tmpData;
                                 if (outBytesPerPixel) {
                                     *outBytesPerPixel = tmpBytesPerPixel;
                                 }
                                 if (outBytesPerStride) {
                                     *outBytesPerStride = tmpBytesPerStride;
                                 }
                             });
```



### IMapper interface (Client <-> Server) 

> hardware/interfaces/graphics/mapper/3.0/IMapper.hal [(링크)](https://android.googlesource.com/platform/hardware/interfaces/+/refs/heads/android10-gsi/graphics/mapper/3.0/IMapper.hal)
>

* hidl-gen compiler 가 .hal interface 를 .h 및 .cpp 로 컴파일함

![hidl-gen](https://source.android.com/devices/architecture/images/treble_cpp_compiler_generated_files.png)



### 4. Vendor Gralloc (HIDL Server)

여기부터 vendor측 구현사항으로 ARM 에서 배포되는 opensource gralloc 기반으로 작성 [(링크)](https://developer.arm.com/tools-and-software/graphics-and-gaming/mali-drivers/android-gralloc-module)



> driver/product/android/gralloc/src/GrallocMapper.h

```C++
#include <android/hardware/graphics/mapper/2.1/IMapper.h>
```



* function

```C++
Return<void> lock(void* buffer, uint64_t cpuUsage,
                  const IMapper::Rect& accessRegion,
                  const hidl_handle& acquireFence,
                  lock_cb hidl_cb) override;
```



* flow to mali gralloc

```C++
if (mali_gralloc_lock_async(&privateModule, bufferHandle, cpuUsage,
                            accessRegion.left, accessRegion.top,
                            accessRegion.width, accessRegion.height,
                           &data, fenceFd) < 0)
```









___

# Appendix

## A. HIDL (HAL Interface Definition Language)

* https://source.android.com/devices/architecture/hidl-cpp/interfaces
* server 는 interface 의 함수를 구현
* client 는 interface 의 함수를 호출



## B. GraphicBuffer usage example

```C++
sp<GraphicBuffer> buffer = new GraphicBuffer(1024, 1024
                                HAL_PIXEL_FORMAT_RGB_8888
                                GralhicBuffer::USAGE_SW_WRITE_OFTEN |
                                GraphicBuffer::USAGE_HW_TEXTURE )
unsigned char* data = nullptr;
buffer->lock(GraphicBuffer::USAGE_SW_WRITE_OFTEN, (void**)&data);

//render to buffer with SW API
    
buffer->unlock();

EGLint eglImgAttr[] = {
    EGL_IMAGE_PRESERVED_KHR,
    EGL_TRUE,
    EGL_NONE,
    EGL_NONE
};
EGLImageKHR img = eglCreateImage(eglGetDisplay(EGL_DEFAULT_DISPLAY),
                                EGL_NO_CONTEXT,
                                EGL_NATIVE_BUFFER_ANDROID,
                                buffer->getNativeBuffer(), eglImgAttr);

int texture;
glGenTexture(1, &texture);
glBindTexture(texture);
glEGLImageTargetTexture2DOES(GL_TEXTURE_2D, img);
```

