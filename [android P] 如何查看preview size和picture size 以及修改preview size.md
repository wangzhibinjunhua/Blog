# [android P] 如何查看preview size和picture size 以及修改preview size

## 1.如何查看preview size和picture size
- adb logcat 抓取一份log,打开camera 预览并拍照
- 搜索log中如下log
```
 MtkCam/ppl_context: [dump]     [APP     ] 00  960x720  OUT t:0/r:0 maxBuffers:11 d/s:0x00000000(UNKNOWN) Hal-Client-usage:0x100(0|HW_TEXTURE) Real:0x32315659(YV12) Request:0x22(IMPLEMENTATION_DEFINED) Override:0x32315659(YV12) Hal-usage:0x20033(0|SW_READ_OFTEN|SW_WRITE_OFTEN|HW_CAMERA_WRITE) HalStream::(consumer/producer)Usage:0/0x20033 bufOffset:  0 rowStrideInBytes/sizeInBytes: 960/691200 480/172800 480/197280 s0:d0:App:YV12:0|HW_TEXTURE 0xa7c9b500
08-02 01:46:51.221   584   967 I MtkCam/ppl_context: [dump]     [APP     ] 01 4160x3120 OUT t:0/r:0 maxBuffers:11 d/s:0x08c20000(V0_JFIF|STANDARD_BT601_625|TRANSFER_SMPTE_170M|RANGE_FULL) Hal-Client-usage:0x3(0|SW_READ_OFTEN) Real:0x21() Request:0x21(BLOB) Override:0x21(BLOB) Hal-usage:0x20033(0|SW_READ_OFTEN|SW_WRITE_OFTEN|HW_CAMERA_WRITE) HalStream::(consumer/producer)Usage:0/0x20033 bufOffset:  0 rowStrideInBytes/sizeInBytes: 16916480/16916480 s1:d0:App:BLOB:0|SW_READ_OFTEN 0xa7c9bc00
08-02 01:46:51.221   584   967 I MtkCam/ppl_context: [dump]     [APP     ] 02  192x144  OUT t:0/r:0 maxBuffers:11 d/s:0x08c20000(V0_JFIF|STANDARD_BT601_625|TRANSFER_SMPTE_170M|RANGE_FULL) Hal-Client-usage:0x3(0|SW_READ_OFTEN) Real:0x32315659(YV12) Request:0x23(YCbCr_420_888) Override:0x23(YCbCr_420_888) Hal-usage:0x20033(0|SW_READ_OFTEN|SW_WRITE_OFTEN|HW_CAMERA_WRITE) HalStream::(consumer/producer)Usage:0/0x20033 bufOffset:  0 rowStrideInBytes/sizeInBytes: 192/27648 96/6912 96/11808 s2:d0:App:YV12:0|SW_READ_OFTEN 0xa7c9bd00
```
[dump]     [APP     ] 00  960x720 为preview size

[dump]     [APP     ] 01 4160x3120 为picture size


## 2.修改preview size
- vendor\mediatek\proprietary\custom\mt6761\hal\imgsensor_metadata\gc5025_mipi_raw\config_static_metadata_scaler.h
修改HAL_PIXEL_FORMAT_YCbCr_420_888
