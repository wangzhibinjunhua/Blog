# android P hal3 camera 插值

## 1.以前camera插值方法
- 直接在hal层config文件里配置picture size即可

## 2.P版本上由于gms限制metadata里配置的picture size不能超过sensor本身,否则会fail
## 解决方法	
- 1.不修改metadata config文件
- 2.直接在app层获取support size list 时候加入自定义的插值 feature/setting/picturesize模块下面
``` public void setCameraCharacteristics(CameraCharacteristics characteristics) {
StreamConfigurationMap s = characteristics
.get(CameraCharacteristics.SCALER_STREAM_CONFIGURATION_MAP);
List<Size> supportedSizes = getSupportedPictureSize(s, ImageFormat.JPEG);
sortSizeInDescending(supportedSizes);

List<String> supportedSizesInStr = sizeToStr(supportedSizes);

//add by wzb test
if(supportedSizesInStr.contains("4160x3120")){
supportedSizesInStr.clear();
supportedSizesInStr.add("5824x4368");
supportedSizesInStr.add("4160x3120");
supportedSizesInStr.add("4160x3120");
supportedSizesInStr.add("4096x2304");

}

mPictureSize.onValueInitialized(supportedSizesInStr);
}	
```
- 3.\frameworks\av\services\camera\libcameraservice\api2\CameraDeviceClient.cpp 中
```
binder::Status CameraDeviceClient::createSurfaceFromGbp(
OutputStreamInfo& streamInfo, bool isStreamInfoValid,
sp<Surface>& surface, const sp<IGraphicBufferProducer>& gbp) {
```
```
if (flexibleConsumer && isPublicFormat(format) &&
            !CameraDeviceClient::roundBufferDimensionNearest(width, height,
            format, dataSpace, mDevice->info(), /*out*/&width, /*out*/&height)) {
        String8 msg = String8::format("Camera %s: No supported stream configurations with "
                "format %#x defined, failed to create output stream",
                mCameraIdStr.string(), format);
        ALOGE("%s: %s", __FUNCTION__, msg.string());
        return STATUS_ERROR(CameraService::ERROR_ILLEGAL_ARGUMENT, msg.string());
    }
```    
将flexibleConsumer赋值为false.不让系统重新去计算合适的size,从而使用我们自定义的插值
- 4.此时发现选择插值时,预览界面变为黑屏,无法正常预览,分析log发现
```
mtkcam-AppStreamMgr: [0-ConfigHandler::checkStream] framework stream dataspace:0x08c20000(V0_JFIF|STANDARD_BT601_625|TRANSFER_SMPTE_170M|RANGE_FULL) {.v3_2 = {.id = 1, .streamType = OUTPUT, .width = 5824, .height = 4368, .format = BLOB, .usage = CPU_READ_NEVER | CPU_READ_RARELY | CPU_READ_OFTEN | CPU_WRITE_NEVER (0x3), .dataSpace = UNKNOWN | STANDARD_UNSPECIFIED | STANDARD_BT601_625 | TRANSFER_UNSPECIFIED | TRANSFER_LINEAR | TRANSFER_SRGB | TRANSFER_SMPTE_170M | RANGE_UNSPECIFIED | RANGE_FULL | V0_JFIF (0x8c20000), .rotation = ROTATION_0}, .physicalCameraId = "", .bufferSize = 26286080}
01-01 00:07:29.408   683   913 E mtkcam-AppStreamMgr: [0-ConfigHandler::checkStream] unsupported size 5824x4368 for format 0x21/rotation:0 - {.v3_2 = {.id = 1, .streamType = OUTPUT, .width = 5824, .height = 4368, .format = BLOB, .usage = CPU_READ_NEVER | CPU_READ_RARELY | CPU_READ_OFTEN | CPU_WRITE_NEVER (0x3), .dataSpace = UNKNOWN | STANDARD_UNSPECIFIED | STANDARD_BT601_625 | TRANSFER_UNSPECIFIED | TRANSFER_LINEAR | TRANSFER_SRGB | TRANSFER_SMPTE_170M | RANGE_UNSPECIFIED | RANGE_FULL | V0_JFIF (0x8c20000), .rotation = ROTATION_0}, .physicalCameraId = "", .bufferSize = 26286080} (checkStream){#403:vendor/mediatek/proprietary/hardware/mtkcam3/main/hal/device/3.x/app/AppStreamMgr.ConfigHandler.cpp}
01-01 00:07:29.408   683   913 E mtkcam-AppStreamMgr: [0-ConfigHandler::checkStreams] streams[id:1] has a bad status: -22(Invalid argument) (checkStreams){#436:vendor/mediatek/proprietary/hardware/mtkcam3/main/hal/device/3.x/app/AppStreamMgr.ConfigHandler.cpp}
01-01 00:07:29.408   683   913 E mtkcam-AppStreamMgr: [0-ConfigHandler::beginConfigureStreams] checkStreams failed - StreamConfiguration={.streams = [3]{{.v3_2 = {.id = 0, .streamType = OUTPUT, .width = 960, .height = 720, .format = IMPLEMENTATION_DEFINED, .usage = CPU_READ_NEVER | CPU_WRITE_NEVER | GPU_TEXTURE (0x100), .dataSpace = UNKNOWN | STANDARD_UNSPECIFIED | TRANSFER_UNSPECIFIED | RANGE_UNSPECIFIED (0), .rotation = ROTATION_0}, .physicalCameraId = "", .bufferSize = 0}, {.v3_2 = {.id = 1, .streamType = OUTPUT, .width = 5824, .height = 4368, .format = BLOB, .usage = CPU_READ_NEVER | CPU_READ_RARELY | CPU_READ_OFTEN | CPU_WRITE_NEVER (0x3), .dataSpace = UNKNOWN | STANDARD_UNSPECIFIED | STANDARD_BT601_625 | TRANSFER_UNSPECIFIED | TRANSFER_LINEAR | TRANSFER_SRGB | TRANSFER_SMPTE_170M | RANGE_UNSPECIFIED | RANGE_FULL | V0_JFIF (0x8c20000), .rotation = ROTATION_0}, .physicalCameraId = "", .bufferSize = 26286080}, {.v3_2 = {.id = 2, .streamType = OUTPUT, .width = 176, .height = 144, .format = YCBCR_420_888, .usage = CPU_READ_NEVER | CPU_READ_RARELY | CPU_READ_OF
01-01 00:07:29.409   683   913 E mtkcam-dev3: [0-session::onConfigureStreamsLocked] fail to beginConfigureStreams (onConfigureStreamsLocked){#665:vendor/mediatek/proprietary/hardware/mtkcam3/main/hal/device/3.x/device/CameraDevice3SessionImpl.cpp}
```
系统会checkStreams里会比较metadata你配置的size和传下来的size. 不在metadata列表就认为是unsupported size.return fail.导致后面无法正常预览
修改vendor/mediatek/proprietary/hardware/mtkcam3/main/hal/device/3.x/app/AppStreamMgr.ConfigHandler.cpp文件
```
auto
ThisNamespace::
checkStream(const V3_4::Stream& rStream) const -> int
```
函数中 直接return ok.或者加以判断,size是自定义的插值,return ok.

问题得以解决,可以正常预览,拍出的照片size也是正常的
