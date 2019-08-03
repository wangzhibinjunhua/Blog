# [android P hal3] 添加第三方算法后,概率性拍照不保存问题.md

## 有项目在添加第三方算法比如美颜,虚化等,出现概率性拍照不保存照片的问题
### 1. 问题分析
- 查找log中关键信息
```
E MtkCam/ZslProc:
[operator()] failed
to select HBC buffer, err:-19(No such device)
(operator()){#394:vendor/mediatek/proprietary/hardware/mtkcam3/pipeline/model/zsl/ZslProcessor.cpp}
```
原因:HBC buffer由于p2s处理时间过长而用光，导致之后的capture request无法获取buffer
处理时间过长是由于三方算法耗时过长,可以查看三方算法耗时
```
07-04 14:42:47.533943 525 7237 D MtkCam/TPI_S_ARC_SB: [process] [arc_prv_kpi] singlecam bokeh process ret:0, cost time: 255ms 
07-04 14:42:47.697319 525 7237 D MtkCam/TPI_S_ARC_SB: [process] [arc_prv_kpi] singlecam bokeh process ret:0, cost time: 155ms 
07-04 14:42:47.896528 525 7237 D MtkCam/TPI_S_ARC_SB: [process] [arc_prv_kpi] singlecam bokeh process ret:0, cost time: 91ms 
07-04 14:42:47.983712 525 7237 D MtkCam/TPI_S_ARC_SB: [process] [arc_prv_kpi] singlecam bokeh process ret:0, cost time: 84ms 
07-04 14:42:48.211666 525 7237 D MtkCam/TPI_S_ARC_SB: [process] [arc_prv_kpi] singlecam bokeh process ret:0, cost time: 225ms
```

### 2.解决方法
- 由于三方算法耗时长,考虑可以减小preview size,减少算法计算量. 尝试过适当减小preview size,只是将问题出现概率降低. 如果再减小size,势必影响用户体验
- 由于是zsl拍照流程,buffer被用光,导致之后的capture request无法获取buffer.  于是关闭zsl,走normal流程就不会有问题.此问题得以解决

### 3.关闭zsl方法
- vendor/mediatek/proprietary/hardware/mtkcam3/pipeline/policy/ConfigSettingPolicyMediator.cpp
```
 auto
 145 ThisNamespace::
 146 evaluateConfiguration(
 147     ConfigurationOutputParams& out,
 148     ConfigurationInputParams const& in __unused
 149 ) -> int
 150 {
 151     //---------------------------------
 152     // 1st level
 153 
 154     featuresetting::ConfigurationInputParams featureIn;
 155     featuresetting::ConfigurationOutputParams featureOut;
 156     featureIn.pSessionParams = &mPipelineUserConfiguration->pParsedAppConfiguration->sessionParams;
 157     featureIn.isP1DirectFDYUV = mPipelineStaticInfo->isP1DirectFDYUV;
 158 
 159     // check zsl enable tag in config stage
 160     MUINT8 bConfigEnableZsl = false;
 161     if(IMetadata::getEntry<MUINT8>(featureIn.pSessionParams, MTK_CONTROL_CAPTURE_ZSL_MODE, bConfigEnableZsl))
 162     {
 163         MY_LOGI("ZSL mode in SessionParams meta (0x%x) : %d", MTK_CONTROL_CAPTURE_ZSL_MODE, bConfigEnableZsl);
 164     } else {
 165         // get default zsl mode
 166         android::sp<IMetadataProvider const>
 167         pMetadataProvider = NSMetadataProviderManager::valueForByDeviceId(mPipelineStaticInfo->openId);
 168         IMetadata::IEntry const& entry = pMetadataProvider->getMtkStaticCharacteristics()
 169                                          .entryFor(MTK_CONTROL_CAPTURE_DEFAULT_ZSL_MODE);
 170         if ( !entry.isEmpty() ) {
 171             bConfigEnableZsl = entry.itemAt(0, Type2Type<MUINT8>());
 172             MY_LOGI("default ZSL mode in static meta (0x%x) : %d", MTK_CONTROL_CAPTURE_DEFAULT_ZSL_MODE, bConfigEnableZsl);
 173         }
 174     }
 175     featureIn.isZslMode = (bConfigEnableZsl) ? true: false;
 ```
 此处可以加以判断,featureIn.isZslMode = false; 关闭zsl即可,不影响app层的菜单选项
 - 由于hal3的优化,hal3上对比hal1 关闭zsl的影响很小
