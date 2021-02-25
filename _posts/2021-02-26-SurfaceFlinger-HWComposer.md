---
title: SurfaceFlinger 之 HWComposer
author: DongXiaoFat
date: 2021-02-26 12:45:00 +0800
categories: [Android, Framework]
tags: [Framework]
excerpt: SurfaceFlinger 之 HWComposer.
math: true
mermaid: true
thumbnail: /assets/img/favicons/av.jpg
---
# SurfaceFlinger 之 HWComposer

> HWC（hwcomposer）是Android中进行窗口（Layer）合成和显示的HAL层模块，其实现是特定于设备的，而且通常由显示设备制造商 (OEM)完成，为SurfaceFlinger服务提供硬件支持。

> SurfaceFlinger可以使用OpenGL ES合成Layer，这需要占用并消耗GPU资源。大多数GPU都没有针对图层合成进行优化，当SurfaceFlinger通过GPU合成图层时，应用程序无法使用GPU进行自己的渲染。而HWC通过硬件设备进行图层合成，可以减轻GPU的合成压力。

## HWComposer 与 SurfaceFlinger的联系

1. 在 SurfaceFlinger 进行init的时候会去获取HWComposer实例，而这份实例是由SurfaceFlingerFactory 根据servicename去创建，而这个servicename 是被保存在了SurfaceFlingerBE中，在SurfaceFlingerBE初始化的时候去获取"debug.sf.hwc_service_name"属性对应的值，如果没有配置该键，就有用默认值"default"

```cpp
// frameworks/native/services/surfaceflinger/SurfaceFlinger.cpp
void SufaceFlinger::init() {
    ....
    mCompositionEngine->setHwComposer(getFactory().createHWComposer(getBE().mHwcServiceName))①
    ....
}

SurfaceFlingerBE::SurfaceFlingerBE() : mHwcServiceName(getServiceName()){}

srd::String getHwcServiceName() {
    char value[PROPERTY_VALUE_MAX] = {};
    property_get("debug.sf.hwc_service_name", value, "default")
}
// frameworks/native/services/surfaceflinger/SurfaceFlinger.h
    SurfaceFlingerBE& getBE() { return mBE; }
    const SurfaceFlingerBE& getBE() const { return mBE; }
    SurfaceFlingerBE mBE;

```
2. HWComposer 与 SurfaceFlinger 的联系

SF 通过层层封装，将 Composer 封装在功能模块

![](res/picture/hwcomposer.jpg)
```cpp
// frameworks/native/services/surfaceflinger/SurfaceFlingerFactory.cpp
std::unique_ptr<HWComposer> createHWComposer(const std::string& serviceName)  override {
    return strd::make_unique<android::impl::HWComposer>(
        std::make_unique<Hwc2::impl::Composer>(serviceName)
    );
}

// frameworks/native/services/surfaceflinger/DisplayHardware/HWComposer.h
namespace android {
    ...
class HWComposer {
    ....//很多纯虚功能函数
}
namespace impl {
    class HWComposer final:public android::HWComposer {
        public:
            explicit HWComposer(std::unique_ptr<Hwc2::Composer> composer);
    }
}
    ...
}
// frameworks/native/services/surfaceflinger/DisplayHardware/ComposerHal.h
namespace android {
    namespace Hwc2 {
        ....
        class Composer {
            //很多纯虚功能函数
        } 
        namespace impl {
            class CommandReader:public CommandReaderBase {

            }
            class Composer final : public Hwc2::Composer {
                ....
                sp<V2_1::IComposer> mComposer;
                sp<V2_1::IComposerClient> mClient;
                sp<V2_2::IComposerClient> mClient_2_2;
                sp<IComposerClient> mClient_2_3;
                CommandWriter mWriter;
                CommandReader mReader;
            }
        }
    }
}

//frameworks/native/services/surfaceflinger/DisplayHardware/ComposerHal.cpp
namespace android {
    namespace Hwc2 {
        ...
        namespace impl {
            Composer::Composer(const std::string& servicename):mWrite(kWriterInitialSize), mIsUsingVrComposer(serviceName == std::string("vr")){
                //通过HIDL通信获取 HAL service。
                mComposer = V2_1::IComposer::getService(serviceName);
                if (sp<IComposer> composer_2_3 = IComposer::castFrom(mComposer)) {
                    composer_2_3->createClient_2_3([&](cost auto& tmpError,const auto& tmpClient){
                        ....
                        mClient = tmpClient;
                        mClient_2_2 = tmpClient;
                        mClient_3_3 = tmpClient;
                    });
                }else{
                    mComposer->createClient_2_3([&](cost auto& tmpError,const auto& tmpClient){
                        ...
                        mClient = tmpClient;
                    });
                }
            }
        }
    }
}

// framework/native/services/surfaceflinger/DisplayHardware/HWComposer.cpp
namespace android {

    namespace impl {
        HWComposer::HWComposer(std::unique_ptr<Hwc2::Composer> composer)
        : mHwcDevice(std::make_unique<HWC2::Device>(std::move(composer))){}
    }
}

// framework/native/services/surfaceflinger/DisplayHardware/HWC2.h
namespace HWC2 {
    class Display;
    class Display{
        //很多display相关的纯虚函数
    }
    class Layer;
        class Layer{
        //很多Layer相关的纯虚函数
    }
    class ComposerCallback{}
    class Device{
        ...
        std::unique_ptr<android::Hwc2::Composer> mComposer;
    }
    namespace impl{
        class Display:public HWC2::Display{
            //很多Display相关的虚函数
            ...
            android::Hwc2::Composer& mComposer;
        }

        class Layer:public HWC2::Layer{
            //很多Layer相关的虚函数
            ...
            android::Hwc2::Composer& mComposer;
        }
    }
}

// framework/native/services/surfaceflinger/DisplayHardware/HWC2.cpp
namespace HWC2 {
    Device::Device(std::unique_ptr<android::Hwc2::Composer> composer):mComposer(std::move(composer)) {
        loadCapabilities();
    }
}
```
## 探究Composer库

在surfaceflinger的mk文件中我们搜索的composer的库为
- android.hardware.graphics.composer@2.1
- android.hardware.graphics.composer@2.2
- android.hardware.graphics.composer@2.3

```xml
<!--frameworks/native/services/surfaceflinger/Android.bp-->
cc_defaults {
    name: "libsurfaceflinger_defaults",
    defaults:["surfaceflinger_defaults"],
    shared_libs: [
        "android.frameworks.vr.composer@1.0",
        "android.hardware.graphics.composer@2.1",
        "android.hardware.graphics.composer@2.2",
        "android.hardware.graphics.composer@2.3",
    ]
}

```
通过搜索代码可以发现android.hardware.graphics.composer@2.1这个库在"hardware/interfaces/graphics/composer/2.1",2.2和2.3与2.1只是版本迭代原理相同。

> 由于这个compsoer@2.1 是一个HIDL接口库,其实现都放在了default文件夹，在这个文件夹下我们可以看到有关键的四分文件

- default
    - Android.bp
    - android.hardware.graphics.composer@2.1-service.rc
    - passthrough.cpp
    - service.cpp

Android.bp 文件会编译2个包，一个是以passthrough.cpp为资源文件的动态库，另一个是以service.cpp为资源的可执行库，而上面的rc文件就是启动这个执行库的入口。

```xml
cc_library_shared {
    name: "android.hardware.graphics.composer@2.1-impl",
    srcs:["passthrough.cpp"],
    ...
}

cc_binary {
    name: "android.hardware.graphics.composer@2.1-service",
    srcs: ["service.cpp"],
    ...
}
```
首先看一下service.cpp文件
```cpp
// hardware/interfaces/graphics/composer/2.1/default/service.cpp
int main() {
    android::ProcessState::initWithDriver("dev/vndbinder");
    android::ProcessState::self()->setThreadPoolMaxThreadCount(4);
    android::ProcessState::self()->startThreadPool();
    ...
    return defaultPassthroughServiceImplementation<IComposer>(4);
}

// hardware/interfaces/graphics/composer/2.1/default/passthrough.cpp
using android::hardware::graphics::composer::V2_1::passthrough::HwcLoader;
extern "C" IComposer* HIDL_FETCH_IComposer(const char* /*name*/) {
    return HwcLoader::load();
}
```
Android的HIDL原理这里就不介绍了，”defaultPassthroughServiceImplementation“方法最终会去调用直通模式的入库函数HIDL_FETCH_IComposer，而该入口只干一件事”load()"。
接下来就让我们看看这个HwcLoader::load()。

## HwcLoader
HwcLoader 顾名思义是用来加载Hwc的，至于加载的是谁，通过阅读代码可知是加载时composer驱动，这里存在Android加载硬件驱动程序的固有套路就不详细介绍了。
这里load主要分为三个部分：
1. loadModule:加载驱动模块
2. createHalWithAdapter:创建ComposerHal实例
3. createComposer:构建Composer实例

```cpp
namespace hardware {
namespace graphics {
namespace composer {
namespace V2_1 {
namespace passthrough {

class HwcLoader {
   public:
    static IComposer* load() {
        const hw_module_t* module = loadModule();
        if (!module) {
            return nullptr;
        }

        auto hal = createHalWithAdapter(module);
        if (!hal) {
            return nullptr;
        }

        return createComposer(std::move(hal));
    }
```

### Composer 驱动模块的加载
通过hw_get_module接口根据HWC_HARDWARE_MODULE_ID也就是“hwcomposer”去找到相应的库，首先会去根据property属性去查找，然后再根据一些配置信息去查找，最后再使用“hwcomposer.default.so”，通过查找代码可以找的该默认库的编译脚本的路径为：hardware\libhardware\modules\hwcomposer\Android.bp,而当前我的Android 10的模拟器使用的是“vendor/lib/hw/hwcomposer.ranchu.so”,而默认库与非默认库存在的区别在于默认库没有什么扩展功能，这一点在接下来的“hwc2_device_t”结构体就能够看出来。

```cpp
    // load hwcomposer2 module
    static const hw_module_t* loadModule() {
        const hw_module_t* module;
        int error = hw_get_module(HWC_HARDWARE_MODULE_ID, &module);
        if (error) {
            ALOGI("falling back to gralloc module");
            error = hw_get_module(GRALLOC_HARDWARE_MODULE_ID, &module);
        }

        if (error) {
            ALOGE("failed to get hwcomposer or gralloc module");
            return nullptr;
        }

        return module;
    }
// hardware\libhardware\include\hardware\hwcomposer_defs.h
#define HWC_HARDWARE_MODULE_ID "hwcomposer"
// hardware\libhardware\include\hardware\hwcomposer.h
typedef struct hwc_composer_device_1 {
    struct hw_device_t common;
    int (*prepare)(struct hwc_composer_device_1 *dev,
                    size_t numDisplays, hwc_display_contents_1_t** displays);
    int (*set)(struct hwc_composer_device_1 *dev,
                size_t numDisplays, hwc_display_contents_1_t** displays);
    int (*eventControl)(struct hwc_composer_device_1* dev, int disp,
            int event, int enabled);
    ...
    int (*getDisplayConfigs)(struct hwc_composer_device_1* dev, int disp,
            uint32_t* configs, size_t* numConfigs);
    ...
} hwc_composer_device_1_t
typedef struct hwc_module {
    /**
     * Common methods of the hardware composer module.  This *must* be the first member of
     * hwc_module as users of this structure will cast a hw_module_t to
     * hwc_module pointer in contexts where it's known the hw_module_t references a
     * hwc_module.
     */
    struct hw_module_t common;
} hwc_module_t;

// hardware\libhardware\modules\hwcomposer\hwcomposer.cpp
hwc_module_t HAL_MODULE_INFO_SYM = {
    .common = {
        .tag = HARDWARE_MODULE_TAG,
        .version_major = 1,
        .version_minor = 0,
        .id = HWC_HARDWARE_MODULE_ID,
        .name = "Sample hwcomposer module",
        .author = "The Android Open Source Project",
        .methods = &hwc_module_methods,
    }
};

struct hwc_context_t {
    hwc_composer_device_1_t device;
    /* our private state goes below here */
};
static int hwc_device_open(const struct hw_module_t* module, const char* name, struct hw_device_t** device);

static struct hw_module_methods_t hwc_module_methods = {
    .open = hwc_device_open//@_1
};
```
至此上述说的第一步“loadModule:加载驱动模块” 就完成了，接下来就会根据第一步获取的 hw_module_t 来进行第二步。

### 创建ComposerHal实例

```cpp
    // create a ComposerHal instance, insert an adapter if necessary
    static std::unique_ptr<hal::ComposerHal> createHalWithAdapter(const hw_module_t* module) {
        bool adapted;
        hwc2_device_t* device = openDeviceWithAdapter(module, &adapted);
        if (!device) {
            return nullptr;
        }
        auto hal = std::make_unique<HwcHal>();
        return hal->initWithDevice(std::move(device), !adapted) ? std::move(hal) : nullptr;//@_2
    }

    // open hwcomposer2 device, install an adapter if necessary
    static hwc2_device_t* openDeviceWithAdapter(const hw_module_t* module, bool* outAdapted) {
        if (module->id && std::string(module->id) == GRALLOC_HARDWARE_MODULE_ID) {
            *outAdapted = true;
            return adaptGrallocModule(module);
        }

        hw_device_t* device;
        //这里的open就是前面@_1处的hwc_device_open
        int error = module->methods->open(module, HWC_HARDWARE_COMPOSER, &device);
        if (error) {
            ALOGE("failed to open hwcomposer device: %s", strerror(-error));
            return nullptr;
        }

        int major = (device->version >> 24) & 0xf;
        if (major != 2) {
            *outAdapted = true;
            return adaptHwc1Device(std::move(reinterpret_cast<hwc_composer_device_1*>(device)));
        }

        *outAdapted = false;
        return reinterpret_cast<hwc2_device_t*>(device);
    }
```
上面提到的open其实就是@_1处的hwc_device_open,而这个函数里其实就是给我们hw_device_t结构体赋值，包括指定其close函数指定为“hwc_device_close”，openDeviceWithAdapter将hw_device_t强转为hwc2_device，是因为hwc2_device的首地址就是指向其首成员hw_device_t common；

```cpp
typedef struct hwc2_device {
    /* Must be the first member of this struct, since a pointer to this struct
     * will be generated by casting from a hw_device_t* */
    struct hw_device_t common;

    /* getCapabilities(..., outCount, outCapabilities)
     *
     * Provides a list of capabilities (described in the definition of
     * hwc2_capability_t above) supported by this device. This list must
     * not change after the device has been loaded.
     */
    void (*getCapabilities)(struct hwc2_device* device, uint32_t* outCount,
            int32_t* /*hwc2_capability_t*/ outCapabilities);

    /* getFunction(..., descriptor)
     *
     * Returns a function pointer which implements the requested description.
     *
     * Parameters:
     *   descriptor - the function to return
     *
     * Returns either a function pointer implementing the requested descriptor
     *   or NULL if the described function is not supported by this device.
     */
    hwc2_function_pointer_t (*getFunction)(struct hwc2_device* device,
            int32_t /*hwc2_function_descriptor_t*/ descriptor);
} hwc2_device_t;
```
有了hwc2_device_t后我们再看一下@_2处，该环节是通过hwc2_device_t里的getCapabilities跟getFunction初始化hwcomposer库里的功能函数。这里说明一下如果是默认库，也即"hwcomposer.default.so",由于没有给getCapabilities跟getFunction赋值所以这里就是空执行。同期可以参考一下高通的hwcomposer库。
```cpp
// hardware\qcom\display\msm8909\sdm\libs\hwc2\hwcomposer.cpp
hwc2_device_t dummy_device = {
    .common = {.tag = HARDWARE_DEVICE_TAG,
               .version = HWC_DEVICE_API_VERSION_2_0,
               .module = &HAL_MODULE_INFO_SYM,
               .close = Hwc2DeviceClose},
    .getCapabilities = Hwc2GetCapabilities,
    .getFunction = Hwc2GetFunction,
};

int Hwc2DeviceOpen(const struct hw_module_t* /*module*/, const char* name,
                   struct hw_device_t** device) {
  if (strcmp(name, HWC_HARDWARE_COMPOSER)) {
    return -EINVAL;
  }

  if (physical_display == nullptr) {
    hwc2_display_t physical_dummy;

    Hwc2ImplCreateVirtualDisplay(&dummy_device, kPhysicalDummyWidth,
                                 kPhysicalDummyHeight, nullptr,
                                 &physical_dummy);

    displays.find(physical_dummy)->second.MakePhysical();

    Hwc2ImplSetActiveConfig(&dummy_device, physical_dummy, kDummyConfig);
  }

  *device = &dummy_device.common;
  return 0;
}
```

接下来就让我们看看这个设备功能初始化——initWithDevice
```cpp
// hardware\interfaces\graphics\composer\2.1\utils\passthrough\include\composer-passthrough\2.1\HwcHal.h

using HwcHal = detail::HwcHalImpl<hal::ComposerHal>;
template <typename Hal>
class HwcHalImpl : public Hal {
...
    bool initWithDevice(hwc2_device_t* device, bool requireReliablePresentFence) {
        // we own the device from this point on
        mDevice = device;
        initCapabilities();//该函数较为简单就是获取驱动设备的功能个数。
        if (requireReliablePresentFence &&
            hasCapability(HWC2_CAPABILITY_PRESENT_FENCE_IS_NOT_RELIABLE)) {
            ALOGE("present fence must be reliable");
            mDevice->common.close(&mDevice->common);
            mDevice = nullptr;
            return false;
        }

        if (!initDispatch()) {
            mDevice->common.close(&mDevice->common);
            mDevice = nullptr;
            return false;
        }

        return true;
    }
...
}
```
其中initCapabilities较为简单就是获取驱动设备的功能个数，而initDispatch则是将本地的功能函数与驱动设备的做映射。
```cpp
/* Function descriptors for use with getFunction */
typedef enum {
    HWC2_FUNCTION_INVALID = 0,
    HWC2_FUNCTION_ACCEPT_DISPLAY_CHANGES,
    HWC2_FUNCTION_CREATE_LAYER,
    HWC2_FUNCTION_CREATE_VIRTUAL_DISPLAY,
    HWC2_FUNCTION_DESTROY_LAYER,
    HWC2_FUNCTION_DESTROY_VIRTUAL_DISPLAY,
    HWC2_FUNCTION_DUMP,
    ...//太多就省略
    HWC2_FUNCTION_SET_DISPLAY_BRIGHTNESS,
} hwc2_function_descriptor_t;

    template <typename T>
    bool initDispatch(hwc2_function_descriptor_t desc, T* outPfn) {
        auto pfn = mDevice->getFunction(mDevice, desc);
        if (pfn) {
            *outPfn = reinterpret_cast<T>(pfn);
            return true;
        } else {
            ALOGE("failed to get hwcomposer2 function %d", desc);
            return false;
        }
    }

    virtual bool initDispatch() {
        if (!initDispatch(HWC2_FUNCTION_ACCEPT_DISPLAY_CHANGES, &mDispatch.acceptDisplayChanges) ||
            !initDispatch(HWC2_FUNCTION_CREATE_LAYER, &mDispatch.createLayer) ||
            !initDispatch(HWC2_FUNCTION_CREATE_VIRTUAL_DISPLAY, &mDispatch.createVirtualDisplay) ||
            !initDispatch(HWC2_FUNCTION_DESTROY_LAYER, &mDispatch.destroyLayer) ||
            .....
    }
```
mDispatch 是一个宏定义结构体，通过getFunction会从设备那边返回一个函数指针给到mDispatch里对应的成员，这样一来就可以通过HAL接口的mDispatch里的函数去访问硬件厂商的hwcomposer库里的函数了。
```cpp
typedef int32_t /*hwc2_error_t*/ (*HWC2_PFN_ACCEPT_DISPLAY_CHANGES)(
        hwc2_device_t* device, hwc2_display_t display);
 struct {
        HWC2_PFN_ACCEPT_DISPLAY_CHANGES acceptDisplayChanges;
        HWC2_PFN_CREATE_LAYER createLayer;
        HWC2_PFN_CREATE_VIRTUAL_DISPLAY createVirtualDisplay;
        HWC2_PFN_DESTROY_LAYER destroyLayer;
        HWC2_PFN_DESTROY_VIRTUAL_DISPLAY destroyVirtualDisplay;
        HWC2_PFN_DUMP dump;
        ....
 } mDispatch={}
```
hardware\qcom\display\msm8909\sdm\libs\hwc2\hwcomposer.cpp
```cpp
int32_t Hwc2ImplAcceptDisplayChanges(hwc2_device_t* /*device*/,
                                     hwc2_display_t display) {
  if (displays.find(display) == displays.end()) {
    return HWC2_ERROR_BAD_DISPLAY;
  }

  return HWC2_ERROR_NONE;
}

hwc2_function_pointer_t Hwc2GetFunction(struct hwc2_device* /*device*/,
                                        int32_t descriptor) {
  switch (descriptor) {
    case HWC2_FUNCTION_ACCEPT_DISPLAY_CHANGES:
      return reinterpret_cast<hwc2_function_pointer_t>(
          Hwc2ImplAcceptDisplayChanges);
    case HWC2_FUNCTION_CREATE_LAYER:
      return reinterpret_cast<hwc2_function_pointer_t>(Hwc2ImplCreateLayer);
      ...
  }
```