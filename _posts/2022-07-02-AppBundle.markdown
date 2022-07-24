---
layout: post
title:  "AppBundle"
date:   2022-07-02
toc:  true
tags: [Android开发框架]
---

## `App Bundle`简介

`Android App Bundle`是一种发布格式，其中包含应用所有经过编译的代码和资源，它会将`Apk`生成及签名交由`Google Play`来完成。

当我们将`App Bundle`上传至`Google Play`后，`Google Play`会将`App Bundle`在语言、屏幕密度和abi维度进行拆分。如果你的手机是一个x86，xhdpi的手机，你在`Google Play`应用市场下载`Apk`时，`Google Play`会获取手机的信息，然后帮你拼装好一个`Apk`，这个`Apk`的资源只有xhdpi的、so库只有x86的，其他无关的都会剔除，从而减少应用大小。   

![AppBundle流程图1](../assets/img/AppBundle1.png)

```HTML
<video src="../assets/img/AppBundle2.mp4" controls="controls" width="500" height="300"></video>
```

另外，`Google Play`还提供了`Play Feature Delivery`（动态交付）的高级能力。

`Play Feature Delivery`的独特优势在于，能够自定义如何以及何时将应用的不同功能下载到搭载 `Android 5.0`（API 级别 21）或更高版本的设备上。例如，为了减小应用的初始下载大小，可以将某些功能模块配置为按需下载，或者只能由支持特定功能（比如拍照或AR）的设备下载。

`Play Feature Delivery`支持的下载策略有：

- 安装时下载

- 按需下载

- 按条件下载

![AppBundle结构图](../assets/img/AppBundle3.png)

- `Base Apk`：包含了所有其他拆分`Apk`都可以访问的代码和资源，并提供应用的基本功能。当用户请求下载应用时，会首先下载并安装该`Apk`。

- `Configuration Apks`：每个`Configuration Apk`都包含针对特定语言、屏幕密度和abi的native libraries 和资源。当用户下载应用时，他们的设备只会下载并安装该设备对应的`Configuration Apk` 。每个`Configuration Apk`都是`Base Apk` 或`Dynamic Feature Apk`的依赖项，也就是说，`Configuration Apk`会随它们为之提供代码和资源的 `Apk` 一起下载和安装。

- `Dynamic Feature Apks`：每个`Dynamic Feature Apk`都包含进行了模块化处理的某项应用功能的代码和资源，开发者可以自定义如何以及何时将该功能下载到设备上。

到此我们简单做个小结：

`AppBundle + GooglePlay`提供了两种功能：

1. 将资源在语言、屏幕密度和abi维度进行拆分，用户下载时根据设备下载对应资源，剔除无关资源，从而减少应用大小。
2. 能够自定义如何以及何时将应用的部分不同功能模块下载到设备上 ，从而减少应用的初始下载大小。

## `App Bundle`使用方式

### `Configuration Apks`

默认情况支持为每一组语言、屏幕密度和abi库生成`Configuration Apks`，可以通过在`build.gradle`中添加如下方式关闭。

```Groovy
android {
    bundle {
        language {
            enableSplit = false
        }
        density {
            enableSplit = false
        }
        abi {
            enableSplit = false
        }
    }
}
```

### `Dynamic Feature Module`

#### [创建`Dynamic Feature Module`](https://developer.android.com/guide/app-bundle/play-feature-delivery?hl=zh-cn)

1. 在`Create New Module`对话框中，选择`Dynamic Feature Module`，然后点击`Next`

   ![Create New Module1](../assets/img/AppBundle4.png)

2. 点击`Next`后，有三项参数需要自己配置：

   1. `Module title`：动态下载安装时显示给用户的名字
   2. `Install-time inclusion`
      1. `Include module at install-time`：应用安装时下载
      2. `Do not include module at install-time（on-demand only）`： 按需下载
      3. `Only include module at app install for devices with specified features`：设备满足指定条件时下载
   3. `Fusing`：对于不支持按需下载的设备(`Android 5.0`以下的系统)，是否直接融合安装

   ![Create New Module2](../assets/img/AppBundle5.png)

3. 点击`Finish`后

   1. `app/build.gradle`文件会新增如下代码
      1. ```Groovy
         android {
             dynamicFeatures = [':DFMoudle']
         }
         ```
   2. `Dynamic Feature Module`的`AndroidManifest`文件会新增如下代码
      1. ```xml
         <?xml version="1.0" encoding="utf-8"?>
         <manifest xmlns:android="http://schemas.android.com/apk/res/android"
             xmlns:dist="http://schemas.android.com/apk/distribution"
             package="com.bytedance.dfmoudle1">
         
             <dist:module
                 // 支持免安装体验
                 dist:instant="false"
                 dist:title="@string/title_dfmoudle1">
                 // 1.安装时下载
                 <dist:delivery>
                     <dist:install-time />
                 </dist:delivery>
                 // 2.按需下载
                 <dist:delivery>
                     <dist:on-demand />
                 </dist:delivery>
                 // 3.按条件下载
                 <dist:delivery>
                     <dist:install-time>
                         <dist:conditions>
                             <dist:user-countries dist:include="true">
                                 <dist:country dist:code="BR" />
                                 <dist:country dist:code="ID" />
                             </dist:user-countries>
                         </dist:conditions>
                     </dist:install-time>
                  </dist:delivery>
                  // Android 5.0以下融合安装
                  <dist:fusing dist:include="true" />
             </dist:module>
             
         </manifest>
         ```

#### [下载按需模块/语言资源](https://developer.android.com/guide/playcore/play-feature-delivery?hl=zh-cn#access_downloaded_modules)

按需下载的模块需要手动触发下载。

1. `app/build.gradle`添加`Play Core`库
   1. ```groovy
      implementation "com.google.android.play:core:${versions}"
      ```
2. 请求按需模块/语言资源

1. 1. ```Kotlin
      // 请求按需模块
      val splitInstallManager = SplitInstallManagerFactory.create(context)
      val request = SplitInstallRequest
              .newBuilder()
              .addModule("pictureMessages")
              .build()
      
      splitInstallManager
          .startInstall(request)
          .addOnSuccessListener { sessionId -> ... }
          .addOnFailureListener { exception ->  ... }
          
      // 请求语言资源
      val request = SplitInstallRequest
              .newBuilder()
              .addLanguage(Locale.forLanguageTag(lang))
              .build()
      
      splitInstallManager
          .startInstall(request)
          .addOnSuccessListener { sessionId -> ... }
          .addOnFailureListener { exception ->  ... }
      ```

#### [访问`Dynamic Feature Mudule`](https://developer.android.com/guide/playcore/play-feature-delivery?hl=zh-cn#access_downloaded_modules)

如需在下载后从已下载的模块访问代码和资源，应用需要为应用和应用下载的功能模块中的每个 `Activity` 启用 [`SplitCompat`库](https://developer.android.com/reference/com/google/android/play/core/splitcompat/SplitCompat?hl=zh-cn)。

1. 启用`SplitCompat`

   方法一：清单文件中声明`SplitCompatApplication`

   ```xml
   <application
       ...
       android:name="com.google.android.play.core.splitcompat.SplitCompatApplication">
   </application>
   ```

   方法二：运行时调用`SplitCompat.install`，或者自定义`Application`继承于`SplitCompatApplication`

   ```kotlin
   class MyApplication : SplitCompatApplication() {
       ...
   }
   ```

   ```kotlin
   override fun attachBaseContext(base: Context) {
       super.attachBaseContext(base)
       SplitCompat.install(this)
   }
   ```

2. 为模块`Activity`启用`SplitCompat`

   为应用启用`SplitCompat`后，还需要为应用在功能模块中下载的每个`Activity`启用`SplitCompat`。

   ```kotlin
   override fun attachBaseContext(base: Context) {
       super.attachBaseContext(base)
       SplitCompat.installActivity(this)
   }
   ```

### 本地测试

默认情况从`Android Studio`将应用部署到连接的设备时，`IDE`会针对目标设备配置构建和部署`Apk`。这是因为，针对特定设备配置构建 `Apk`比针对应用支持的所有设备配置构建`App Bundle`速度更快。

1. 下载[`bundletool`](https://developer.android.com/studio/command-line/bundletool?hl=zh-cn#deploy_with_bundletool)工具

   `bundletool` 是一种底层工具，可供`Android Studio`、`AGP插件`和`Google Play`用于构建`App Bundle`文件并将`App Bundle`转换为部署到设备的各种`Apk`。

2. `Android Studio`构建 `App Bundle`

   `Build` -> `Build Bundle(s) / APK(s)` -> `Build Bundle(s)`

3. 根据`App Bundle`生成一组`Apk`

   当`bundletool` 从`App Bundle`生成`Apk`时，它会将这些`Apk`纳入到一个名为`Apk Set Archive`的容器中，该容器以`.apks`作为文件扩展名。

   ```kotlin
   bundletool build-apks --bundle=/MyApp/my_app.aab --output=/MyApp/my_app.apks
   ```

4. 将`Apks`部署到连接的设备

   生成`Apks`文件后，`bundletool`可以将其中适当的`Apk`组合部署到已连接的设备。

   ```kotlin
   bundletool install-apks --apks=/MyApp/my_app.apks
   ```

## 实现原理

### `AAB`构建

#### `Android`构建流程

![Android构建流程](../assets/img/AppBundle6.png)

典型`Android`应用模块的构建流程按照以下步骤执行：

1. 编译器将您的源代码转换成`DEX`文件（`Dalvik`可执行文件，其中包括在`Android`设备上运行的字节码），并将其他所有内容转换成编译后的资源。
2. 打包器将`DEX`文件和编译后的资源组合成`Apk`或`AAB`（具体取决于所选的`build`目标）。 
3. 打包器使用调试或发布密钥库为`Apk`或`AAB`签名。
4. 在生成最终`Apk`之前，打包器会使用 [zipalign](https://developer.android.com/studio/command-line/zipalign) 工具对应用进行优化，以减少其在设备上运行时所占用的内存。

#### 查看AGP源码

在`app/build.gradle`中添加下图代码

![AGP源码1](../assets/img/AppBundle7.png)

`Sync`后就可以找到`Android Gradle Plugin`源码

![AGP源码2](../assets/img/AppBundle8.png)

#### `AAB`构建

在`ApplicationTaskManager`中可以找到与`AAB`相关任务。

```Kotlin
// ApplicationTaskManager
private fun createDynamicBundleTask(variantInfo: ComponentInfo<ApplicationVariantBuilderImpl, ApplicationVariantImpl>) {
    val variant = variantInfo.variant
    // 1.将文件压缩成base.zip
    taskFactory.register(PerModuleBundleTask.CreationAction(variant))
    ...
    if (variant.variantType.isBaseModule) {
        taskFactory.register(ParseIntegrityConfigTask.CreationAction(variant))
        // 2.生成aab
        taskFactory.register(PackageBundleTask.CreationAction(variant))
        
        taskFactory.register(FinalizeBundleTask.CreationAction(variant))
        taskFactory.register(BundleIdeModelProducerTask.CreationAction(variant))
        // 3.根据aab文件生成apks文件
        taskFactory.register(BundleToApkTask.CreationAction(variant))
        taskFactory.register(BundleToStandaloneApkTask.CreationAction(variant))
        taskFactory.register(ExtractApksTask.CreationAction(variant))

        taskFactory.register(MergeNativeDebugMetadataTask.CreationAction(variant))
        variant.taskContainer.assembleTask.configure { task ->
            task.dependsOn(variant.artifacts.get(InternalArtifactType.MERGED_NATIVE_DEBUG_METADATA))
        }
    }
}
```

主要任务有三个：

1. `PerModuleBundleTask`（将各个模块压缩成`zip`包）
2. `PackageBundleTask`（将各个`zip`包组合生成`AAB`）

1. `BundleToApkTask`（根据`AAB`生成`Apks`）

##### `PerModuleBundleTask`

```Kotlin
abstract class PerModuleBundleTask @Inject constructor(objects: ObjectFactory) :
    NonIncrementalTask() {
    
    class CreationAction(
        creationConfig: ApkCreationConfig
    ) : VariantTaskCreationAction<PerModuleBundleTask, ApkCreationConfig>(
        creationConfig
    ) {
        override fun configure(
            task: PerModuleBundleTask
        ) {
            super.configure(task)
            val artifacts = creationConfig.artifacts
            // 文件名
            if (creationConfig is DynamicFeatureCreationConfig) {
                task.fileName.set(creationConfig.featureName.map { "$it.zip" })
            } else {
                task.fileName.set("base.zip")
            }
            ...
        }
    }
    
    public override fun doTaskAction() {
        // 创建JarCreator对象
        val jarCreator =
            JarCreatorFactory.make(
                jarFile = File(outputDir.get().asFile, fileName.get()).toPath(),
                type = jarCreatorType.get()
            )
        ...
        // 将各个文件压入
        jarCreator.use {
            // assetsFiles
            it.addDirectory(
                assetsFiles.get().asFile.toPath(),
                null,
                null,
                Relocator(FD_ASSETS)
            )
            // resFiles
            it.addJar(resFiles.get().asFile.toPath(), excludeJarManifest, ResRelocator())
            // dex files
            val dexFilesSet = if (hasFeatureDexFiles()) featureDexFiles.files else dexFiles.files
            if (dexFilesSet.size == 1) {
                addHybridFolder(it, dexFilesSet.sortedBy { it.name }, Relocator(FD_DEX), excludeJarManifest)
            } else {
                addHybridFolder(it, dexFilesSet.sortedBy { it.name }, DexRelocator(FD_DEX), excludeJarManifest)
            }
            // javaResFiles
            val javaResFilesSet =
                if (hasFeatureDexFiles()) featureJavaResFiles.files else javaResFiles.files
            addHybridFolder(it, javaResFilesSet, Relocator("root"), JarMerger.EXCLUDE_CLASSES)
    
            addHybridFolder(it, nativeLibsFiles.files, fileFilter = abiFilter)
        }
    }    
}
```

`PerModuleBundleTask`任务将需要的文件压缩成一个`base.zip`文件，压缩规则如下：

1. 将`assets`文件压入`assets`目录 
2. 将`res`文件压入`res`目录，其中`AndroidManifest.xml`、`resources.pb`文件会被压入`manifest`目录
3. 将`dex`文件压入`dex`目录
4.  将`java resources`文件压入`root`目录中

`base.zip`目录结构如图所示，文件位于`/app/build/intermediates/module_bundle/debug/base.zip`。

![base.zip目录结构](../assets/img/AppBundle9.png)

##### `PackageBundleTask`

```Kotlin
abstract class PackageBundleTask : NonIncrementalTask() {
    
    override fun doTaskAction() {
        workerExecutor.noIsolation().submit(BundleToolWorkAction::class.java) {
            it.initializeFromAndroidVariantTask(this)
            it.bundleType.set(bundleType)
            // 填充进BundleToolWorkAction
            if (baseModuleZip.isPresent) {
                it.baseModuleFile.set(baseModuleZip.get().asFileTree.singleFile)
            }
            it.featureFiles.from(featureZips)
            if (assetPackZips.isPresent) {
                it.assetPackFiles.from(assetPackZips.get().asFileTree.files)
            }
            it.mainDexList.set(mainDexList)
            ...
        }
    }
    
    abstract class BundleToolWorkAction : ProfileAwareWorkAction<Params>() {
        override fun run() {
            val builder = ImmutableList.builder<Path>()
            // 将各模块zip文件封装成builder
            if (parameters.baseModuleFile.isPresent) {
                builder.add(parameters.baseModuleFile.asFile.get().toPath())
            }
            parameters.featureFiles.forEach { builder.add(getBundlePath(it)) }
            parameters.assetPackFiles.files.forEach { builder.add(getBundlePath(it.parentFile)) }
            
            // 将三个spilt维度封装成一个SplitsConfig
            val splitsConfig = Config.SplitsConfig.newBuilder()
                .splitBy(Config.SplitDimension.Value.ABI, parameters.bundleOptions.get().enableAbi)
                .splitBy(
                    Config.SplitDimension.Value.SCREEN_DENSITY,
                    parameters.bundleOptions.get().enableDensity
                )
                .splitBy(Config.SplitDimension.Value.LANGUAGE, parameters.bundleOptions.get().enableLanguage)
                
            val bundleOptimizations = Config.Optimizations.newBuilder()
                .setSplitsConfig(splitsConfig)
                .setUncompressNativeLibraries(uncompressNativeLibrariesConfig)
            
            // 将bundle打包的一些配置封装成bundleConfig
            val bundleConfig =
                Config.BundleConfig.newBuilder()
                    .setType(parameters.bundleType.get())
                    .setCompression(
                        Config.Compression.newBuilder()
                            .addAllUncompressedGlob(noCompressGlobsForBundle)
                    )
                    .setOptimizations(bundleOptimizations)
                    
            // 调用BuildBundleCommand生成aab文件
            val command = BuildBundleCommand.builder()
                .setUncompressedBundle(true)
                .setBundleConfig(bundleConfig.build())
                .setOutputPath(bundleFile.toPath())
                .setModulesPaths(builder.build())
                
            command.build().execute()
        }
    }
}
```

`PackageBundleTask`任务，将各模块`zip`文件封装成`builder`，将三个`spilt`维度以及一些配置信息封装成`bundleConfig`，最后全部委托给`bundletool`工具中的[`BuildBundleCommand`](https://github.com/google/bundletool/blob/master/src/main/java/com/android/tools/build/bundletool/commands/BuildApksCommand.java)类去生成`AAB`文件。

[`BuildBundleCommand`](https://github.com/google/bundletool/blob/master/src/main/java/com/android/tools/build/bundletool/commands/BuildApksCommand.java)任务比较简单，主要是对`zip`包文件的拷贝、生成配置文件等，这里不多做介绍，感兴趣的同学可以自行了解。

##### `AAB`文件结构

最终生成的`AAB`文件结构如下

![AAB目录结构](../assets/img/AppBundle10.png)

可以看到，`AAB`文件格式下各个模块的代码和资源的组织方式都与 `APK` 中的相似，之所以如此，是因为每个模块都可以作为单独的`Apk` 生成。

然后，`Google Play`会使用`AAB`生成向用户提供的各种 `Apks`，如`Base Apk`、`Feature Apks`，`Configuration Apks`。以蓝色标识的目录（如 `drawable/`、`values/` 和 `lib/` 目录）表示`Google Play`用来为每个模块创建`Configuration Apk`的代码和资源。

`Base Moudle`（`base/`） 对比`base.zip`文件

![base.zip目录结构](../assets/img/AppBundle9.png)

和`base.zip`文件的区别仅仅在于：多了一个`native.pb`和`assets.pb`文件

模块协议缓冲区 (`*.pb`) 文件：这些文件提供了一些元数据，有助于向各个应用商店（如 `Google Play`）说明每个应用模块的内容。例如，`BundleConfig.pb`提供了有关`bundle`本身的信息（如用于构建`App Bundle`的构建工具版本），`native.pb`和`resources.pb`说明了每个模块中的代码和资源，这在`Google Play`针对不同的设备配置优化`Apk`时非常有用。

**到此，可以发现`AAB`本身并不支持动态分发，只是动态分发的一个载体文件，真正实现动态分发逻辑的其实是`Google Play`/ `bundletool`。**

### `AAB` -> `ApkS`

#### `BundleToApkTask`

然后来看下`Google Play`是怎么根据`AAB`文件生成`Apks`文件的，`AGP`里模拟生成`Apks`的任务是`makeApkFromBundle`，其实现类为`BundleToApkTask`。当然，也可以通过`bundletool`生成。

```kotlin
bundletool build-apks --bundle=/MyApp/my_app.aab --output=/MyApp/my_app.apks
```

```Kotlin
abstract class BundleToApkTask : NonIncrementalTask() {
    override fun doTaskAction() {
    workerExecutor.noIsolation().submit(BundleToolRunnable::class.java) {
        it.bundleFile.set(bundle)
        ...
    }
    
    abstract class BundleToolRunnable : ProfileAwareWorkAction<Params>() {
        override fun run() {
            val command = BuildApksCommand
                .builder()
                .setExecutorService(MoreExecutors.listeningDecorator(ForkJoinPool.commonPool()))
                .setBundlePath(parameters.bundleFile.asFile.get().toPath())
                ...
    
            command.build().execute()
        }
    }
}
}
```

`BundleToApkTask`任务，最后也会委托给`bundleTool`中的[`BuildApksCommand`](https://github.com/google/bundletool/blob/master/src/main/java/com/android/tools/build/bundletool/commands/BuildApksCommand.java)类。

#### `bundletool.BuildApksCommand`

该类会根据分包规则，生成一系列的`Apk`文件，包含完整`Apk`，同时也包含一些不同维度的`spilt`包。

![BuildApksCommand流程](../assets/img/AppBundle11.png)

拆分`AAB`的核心代码如下

```Kotlin
// ModuleSplitter
private ImmutableList<ModuleSplit> splitModuleInternal() {
  	// 走进runSplitters方法
    Stream var10000 = this.runSplitters().stream();
    ...
		return this.stampSource.isPresent() ? (ImmutableList)moduleSplits.stream().map((moduleSplit) -> {
        return moduleSplit.writeSourceStampInManifest((String)this.stampSource.get(), this.stampType);
    }).collect(ImmutableList.toImmutableList()) : moduleSplits;
}

private ImmutableList<ModuleSplit> runSplitters() {
  	...
    Builder<ModuleSplit> splits = ImmutableList.builder();
    SplittingPipeline resourcesPipeline = this.createResourcesSplittingPipeline();
    splits.addAll(resourcesPipeline.split(ModuleSplit.forResources(this.module, this.variantTargeting)));
    SplittingPipeline nativePipeline = this.createNativeLibrariesSplittingPipeline();
    splits.addAll(nativePipeline.split(ModuleSplit.forNativeLibraries(this.module, this.variantTargeting)));
    SplittingPipeline assetsPipeline = this.createAssetsSplittingPipeline();
    splits.addAll(assetsPipeline.split(ModuleSplit.forAssets(this.module, this.variantTargeting)));
    SplittingPipeline dexPipeline = this.createDexSplittingPipeline();
    splits.addAll(dexPipeline.split(ModuleSplit.forDex(this.module, this.variantTargeting)));
    splits.add(ModuleSplit.forRoot(this.module, this.variantTargeting));
    // 走进merge方法
    ImmutableList<ModuleSplit> mergedSplits = (new SameTargetingMerger()).merge(applyMasterManifestMutators(splits.build()));
    ImmutableList<ModuleSplit> defaultTargetingSplits = (ImmutableList)mergedSplits.stream().filter((split) -> {
      return split.getApkTargeting().equals(ApkTargeting.getDefaultInstance());
    }).collect(ImmutableList.toImmutableList());
    return mergedSplits;
}

// SameTargetingMerger
public ImmutableList<ModuleSplit> merge(ImmutableCollection<ModuleSplit> moduleSplits) {
    Builder<ModuleSplit> result = ImmutableList.builder();
  	// 根据ModuleSplit::getApkTargeting对所有资源进行拆分
    ImmutableListMultimap<ApkTargeting, ModuleSplit> splitsByTargeting = Multimaps.index(moduleSplits, ModuleSplit::getApkTargeting);
    UnmodifiableIterator var4 = splitsByTargeting.keySet().iterator();

    while(var4.hasNext()) {
        ApkTargeting targeting = (ApkTargeting)var4.next();
        result.add(this.mergeSplits(splitsByTargeting.get(targeting)));
    }

    return result.build();
}
```

拆分完成后会开始生成`Apks`文件。具体生成`Apks`的步骤就是一个反序列化的步骤，需要`aapt2`的参与(即`Android Asset Packaging Tool`，是`Android`中的资源打包工具)，也就是调用`aapt2`的`convert`命令，将`pb`格式的资源文件，生成`Android`可以使用的最终的二进制资源文件。

```kotlin
public interface Aapt2Command {
    static Aapt2Command createFromExecutablePath(final Path aapt2Path) {
        return new Aapt2Command() {
            public void convertApkProtoToBinary(Path protoApk, Path binaryApk) {
                ImmutableList<String> convertCommand = ImmutableList.of(aapt2Path.toString(), "convert", "--output-format", "binary", "-o", binaryApk.toString(), protoApk.toString());
                (new DefaultCommandExecutor()).execute(convertCommand, CommandOptions.builder().setTimeout(this.timeoutMillis).build());
            }

            public void optimizeToSparseResourceTables(Path originalApk, Path outputApk) {
                ImmutableList<String> convertCommand = ImmutableList.of(aapt2Path.toString(), "optimize", "--enable-sparse-encoding", "-o", outputApk.toString(), originalApk.toString());
                (new DefaultCommandExecutor()).execute(convertCommand, CommandOptions.builder().setTimeout(this.timeoutMillis).build());
            }
            ...
        };
    }
}
```

#### `Apks`文件结构

最后会生成`bundle.apks`文件，文件位于`/app/build/intermediates/apks_from_bundle/debug/bundle.apks`。

![Apks文件目录结构](../assets/img/AppBundle12.png)

`base-master.apk`和`df1-master.apk`为完整`Apk`，其他均为各个维度的`Split Apk`。

`toc.pb`：包含`Apk`集合的相关描述。

![base module目录结构](../assets/img/AppBundle13.png)

![split apk目录结构](../assets/img/AppBundle14.png)

### `APKS` -> `APK`

```Kotlin
bundletool install-apks --apks=/MyApp/my_app.apks
```

#### `bundletool.InstallApksCommand`

![InstallApksCommand](../assets/img/AppBundle15.png)

```Java
// InstallApksCommand
public void execute() {
    BuildApksResult toc = this.readBuildApksResult();
    this.validateInput(toc);
    AdbServer adbServer = this.getAdbServer();
    adbServer.init(this.getAdbPath());
    TempDirectory tempDirectory = new TempDirectory();
    Throwable var4 = null;

    try {
        DeviceSpec deviceSpec = (new DeviceAnalyzer(adbServer)).getDeviceSpec(this.getDeviceId());
        if (this.getDeviceTier().isPresent()) {
            deviceSpec = deviceSpec.toBuilder().setDeviceTier(Int32Value.of((Integer)this.getDeviceTier().get())).build();
        }

        if (this.getDeviceGroups().isPresent()) {
            deviceSpec = deviceSpec.toBuilder().addAllDeviceGroups((Iterable)this.getDeviceGroups().get()).build();
        }
				// 挑选与设备匹配的Split Apk
        ImmutableList<Path> apksToInstall = this.getApksToInstall(toc, deviceSpec, tempDirectory.getPath());
        ImmutableList<Path> filesToPush = ImmutableList.builder().addAll(this.getApksToPushToStorage(toc, deviceSpec, tempDirectory.getPath())).addAll((Iterable)this.getAdditionalLocalTestingFiles().orElse(ImmutableList.of())).build();
        AdbRunner adbRunner = new AdbRunner(adbServer);
        InstallOptions installOptions = InstallOptions.builder().setAllowDowngrade(this.getAllowDowngrade()).setAllowTestOnly(this.getAllowTestOnly()).setTimeout(this.getTimeout()).build();
        if (this.getDeviceId().isPresent()) {
            adbRunner.run((device) -> {
              	// 安装选中的Apk
                device.installApks(apksToInstall, installOptions);
            }, (String)this.getDeviceId().get());
        } else {
            adbRunner.run((device) -> {
                device.installApks(apksToInstall, installOptions);
            });
        }

        if (!filesToPush.isEmpty()) {
            this.pushFiles(filesToPush, toc, adbRunner);
        }

        if (toc.getLocalTestingInfo().getEnabled()) {
            this.cleanUpEmulatedSplits(adbRunner, toc);
        }
    } catch (Throwable var17) {
        ...
    }
}

private ImmutableList<Path> getApksToInstall(BuildApksResult toc, DeviceSpec deviceSpec, Path output) {
    com.android.tools.build.bundletool.commands.ExtractApksCommand.Builder extractApksCommand = ExtractApksCommand.builder().setApksArchivePath(this.getApksArchivePath()).setDeviceSpec(deviceSpec);
    ...
    // 通过ExtractApksCommand类挑选与设备匹配的Split Apk
    return extractApksCommand.build().execute();
}
```

主要介绍下：

1. 如何挑选与设备匹配的`Split Apk`（`ExtractApksCommand`）
2. 安装选中的`Apk`（s）（`Device.installApks`）

##### `bundletool.ExtractApksCommand`

![ExtractApksCommand](../assets/img/AppBundle16.png)

```Java
// ExtractApksCommand
ImmutableList<Path> execute(PrintStream output) {
  this.validateInput();
  BuildApksResult toc = ResultUtils.readTableOfContents(this.getApksArchivePath());
  Optional<ImmutableSet<String>> requestedModuleNames = this.getModules().map((modules) -> {
    return resolveRequestedModules(modules, toc);
  });
  DeviceSpec deviceSpec = applyDefaultsToDeviceSpec(this.getDeviceSpec(), toc);
  ApkMatcher apkMatcher = new ApkMatcher(deviceSpec, requestedModuleNames, this.getInstant(), true);
  // 进入getMatchingApks
  ImmutableList<GeneratedApk> generatedApks = apkMatcher.getMatchingApks(toc);
  if (generatedApks.isEmpty()) {
    throw (IncompatibleDeviceException)IncompatibleDeviceException.builder().withUserMessage("No compatible APKs found for the device.").build();
  } else {
    return Files.isDirectory(this.getApksArchivePath(), new LinkOption[0]) ? (ImmutableList)generatedApks.stream().map((matchedApk) -> {
      return this.getApksArchivePath().resolve(matchedApk.getPath().toString());
    }).collect(ImmutableList.toImmutableList()) : this.extractMatchedApksFromApksArchive(generatedApks, toc);
  }
}

// ApkMatcher
public ImmutableList<ApkMatcher.GeneratedApk> getMatchingApks(BuildApksResult buildApksResult) {
  Optional<Variant> matchingVariant = this.variantMatcher.getMatchingVariant(buildApksResult);
  matchingVariant.ifPresent((variant) -> {
    this.validateVariant(variant, buildApksResult);
  });
  ImmutableList<ApkMatcher.GeneratedApk> variantApks = matchingVariant.isPresent() ? this.getMatchingApksFromVariant((Variant)matchingVariant.get(), Version.of(buildApksResult.getBundletool().getVersion())) : ImmutableList.of();
  // 进入getMatchingApksFromAssetModules
  ImmutableList<ApkMatcher.GeneratedApk> assetModuleApks = this.getMatchingApksFromAssetModules(buildApksResult.getAssetSliceSetList());
  return ImmutableList.builder().addAll(variantApks).addAll(assetModuleApks).build();
}

public ImmutableList<ApkMatcher.GeneratedApk> getMatchingApksFromAssetModules(Collection<AssetSliceSet> assetModules) {
  ImmutableSet<String> assetModulesToMatch = (ImmutableSet)this.requestedModuleNames.orElseGet(() -> {
    return getUpfrontAssetModules(assetModules);
  });
  return (ImmutableList)assetModules.stream().filter((assetModule) -> {
    return assetModulesToMatch.contains(assetModule.getAssetModuleMetadata().getName());
  }).flatMap((assetModule) -> {
    // 进入matchesApkTargeting
    return assetModule.getApkDescriptionList().stream().filter((apkDescription) -> {
      return this.matchesApkTargeting(apkDescription.getTargeting());
    }).map((apkDescription) -> {
      return ApkMatcher.GeneratedApk.create(ZipPath.create(apkDescription.getPath()), assetModule.getAssetModuleMetadata().getName(), assetModule.getAssetModuleMetadata().getDeliveryType());
    });
  }).collect(ImmutableList.toImmutableList());
}

private boolean matchesApkTargeting(ApkTargeting apkTargeting) {
  return this.apkMatchers.stream().allMatch((matcher) -> {
    // 还是根据apkTargeting匹配
    return matcher.getApkTargetingPredicate().test(apkTargeting);
  });
}
```

##### `bundletool.Device.installApks`

```Java
// DdmlibDevice
public void installApks(ImmutableList<Path> apks, InstallOptions installOptions) {
    ...
    try {
        if (this.getVersion().isGreaterOrEqualThan(AndroidVersion.ALLOW_SPLIT_APK_INSTALLATION.getApiLevel())) {
          	// 进入installPackages方法
            this.device.installPackages(apkFiles, installOptions.getAllowReinstall(), extraArgs.build(), installOptions.getTimeout().toMillis(), TimeUnit.MILLISECONDS);
        } else {
            this.device.installPackage(((File)Iterables.getOnlyElement(apkFiles)).toString(), installOptions.getAllowReinstall(), (String[])extraArgs.build().toArray(new String[0]));
        }
       ...
    }
}

// DeviceImpl
public void installPackages(List<File> apks, boolean reinstall, List<String> installOptions, long timeout, TimeUnit timeoutUnit) throws InstallException {
    try {
        this.lastInstallMetrics = SplitApkInstaller.create(this, apks, reinstall, installOptions).install(timeout, timeoutUnit);
    }
    
}
// SplitApkInstaller
public InstallMetrics install(long timeout, TimeUnit unit) throws InstallException {
    try {
        long totalFileSize = 0L;

        File apkFile;
        for(Iterator var6 = this.mApks.iterator(); var6.hasNext(); totalFileSize += apkFile.length()) {
            apkFile = (File)var6.next();
        }

        String option = String.format("-S %d", totalFileSize);
        if (this.getOptions() != null) {
            option = this.getOptions() + " " + option;
        }
        // 1.创建一个安装session，传入所有Apk的总大小
        String sessionId = this.createMultiInstallSession(option, timeout, unit);
        int index = 0;
        boolean allUploadSucceeded = true;

        long uploadStartNs;
        // 2.依次安装Apk
        for(
            uploadStartNs = System.nanoTime(); 
            allUploadSucceeded && index < this.mApks.size(); 
            allUploadSucceeded = this.uploadApk(
                sessionId, 
                (File)this.mApks.get(index),
                index++,
                timeout,
                unit
            )
        ) {}

        long uploadFinishNs = System.nanoTime();
        // 3.如果所有Apk都安装成功，则执行commit，否则放弃安装并抛错
        if (!allUploadSucceeded) {
            this.installAbandon(sessionId, timeout, unit);
            throw new InstallException("Failed to install-write all apks");
        } else {
            this.installCommit(sessionId, timeout, unit);
            Log.d("SplitApkInstaller", "Successfully install apks: " + this.mApks.toString());
            return new InstallMetrics(uploadStartNs, uploadFinishNs, uploadFinishNs, System.nanoTime());
        }
    } catch (InstallException var14) {
        throw var14;
    } catch (Exception var15) {
        throw new InstallException(var15);
    }
}
```

安装选中的`Apk`（s）主要分三步：

1. 创建一个安装`session`，传入所有`Apk`的总大小

1. 依次安装各个`Apk`

1. 如果所有`Apk`都安装成功，则执行`commit`，否则放弃安装并抛错

### `Dynamic Feature`

![Dynamic Feature](../assets/img/AppBundle17.png)

整体流程如下：

1. 创建一个`Task`，通过`Binder`向`GP`的`SplitInstallService`发送Dynamic Feature模块的下发请求（`IPC`）

2. `GP`在接收到请求后开始下载任务。通过`broadcast`执行`onStateUpdate`，通知客户端下载的实时状态。

   1. `receiver`注册时机
      - `SplitInstallManager.registerListener`
      - `SplitCompat.install`

   1. 在`onReceive`中会分成两种情况进行分别处理：
      - `status==3`(DOWNLOAD)即下载成功，下载成功之后主要进行插件安装
      - `status!=3`的情况，这里直接转发至`StateUpdateListener`，进而通知在业务中注册的`listeners`
      - 这里需要注意一个特殊场景：`status==8`时，需要弹出授权弹窗，由用户授权继续下载，这里在业务开发时需要特别注意。触发该限制主要有两种场景：
        - 单个插件大小超过10M
        - 15min时间窗口内，静默下载（非弹窗授权下载）插件总体积之和超过10M

3. `GP`下载的`Dynamic Feature Apk`在`GP`应用文件目录`/files/dynamicsplits`下，当插件下载完成后，`GP`将下载的文件`uri`通过`broadcast`传递给`client`。客户端拿到`Apk`后会做一些解析，然后将`Apk`拷贝到自己应用内`files/splitcompat/$versioncode/unverified-splits`目录下。

4. 之后有个校验签名是否与`base.apk`一致的过程，如果符合，`unverified-splits`文件夹会被重命名为`verified-splits`。

5. 删除之前版本的`splitcompat`目录，这里主要适配版本升级情况。

6. 动态加载插件`so`资源：`makePathElements`

7. 动态加载`dex`资源：`add dex elements`

8. 动态加载`asset`资源：`add assetPath`

### 加载多`Apk`

#### `Split Apks`（静态）

使用`AAB`安装应用时，会根据设备配置安装多个`Apk`，首先安装Base Apk，然后安装`Split Apks`。

`Split Apks`是`Android 5.0`开始提供的多`Apk`构建机制。下面来看一下它的实现原理。

首先，在`ApplicationInfo`中，增加了Split Apk相关字段。

```Java
// ApplicationInfo

/**
 * The names of all installed split APKs, ordered lexicographically.
 */
public String[] splitNames;

/**
 * Full paths to zero or more split APKs, indexed by the same order as {@link #splitNames}.
 */
public String[] splitSourceDirs;

/**
 * Full path to the publicly available parts of {@link #splitSourceDirs},
 * including resources and manifest. This may be different from
 * {@link #splitSourceDirs} if an application is forward locked.
 *
 * @see #splitSourceDirs
 */
public String[] splitPublicSourceDirs;
```

然后，在创建`PathClassLoader`时，会将`splitSourceDirs`传入类加载器。

```Java
// LoadedApk
public ClassLoader getClassLoader() {
    synchronized (this) {
        if (mClassLoader == null) {
            createOrUpdateClassLoaderLocked(null /*addedPaths*/);
        }
        return mClassLoader;
    }
}

private void createOrUpdateClassLoaderLocked(List<String> addedPaths) {
    ...
    final List<String> zipPaths = new ArrayList<>(10);
    final List<String> libPaths = new ArrayList<>(10);
    ...
    makePaths(mActivityThread, isBundledApp, mApplicationInfo, zipPaths, libPaths);
    ...
    final String zip = (zipPaths.size() == 1) ? zipPaths.get(0) :
            TextUtils.join(File.pathSeparator, zipPaths);

    if (mDefaultClassLoader == null) {
        ...
        // 将zipPaths传入类加载器
        mDefaultClassLoader = ApplicationLoaders.getDefault().getClassLoaderWithSharedLibraries(
                zip, mApplicationInfo.targetSdkVersion, isBundledApp, librarySearchPath,
                libraryPermittedPath, mBaseClassLoader,
                mApplicationInfo.classLoaderName, sharedLibraries);
        mAppComponentFactory = createAppFactory(mApplicationInfo, mDefaultClassLoader);
        ...
    }
    ...
}

public static void makePaths(ActivityThread activityThread,
                             boolean isBundledApp,
                             ApplicationInfo aInfo,
                             List<String> outZipPaths,
                             List<String> outLibPaths) {
    ...
    if (aInfo.splitSourceDirs != null && !aInfo.requestsIsolatedSplitLoading()) {
      	// 将splitSourceDirs传入zipPaths
        Collections.addAll(outZipPaths, aInfo.splitSourceDirs);
    }
    ...
 }
```

最后，在创建`Resources`时，也会将`splitSourceDirs`传进去。

```Java
// LoadedApk
public Resources getResources() {
    if (mResources == null) {
        final String[] splitPaths;
        try {
            splitPaths = getSplitPaths(null);
        } catch (NameNotFoundException e) {
            ...
        }
				// 将splitPaths传入Resources
        mResources = ResourcesManager.getInstance().getResources(null, mResDir,
                splitPaths, mOverlayDirs, mApplicationInfo.sharedLibraryFiles,
                Display.DEFAULT_DISPLAY, null, getCompatibilityInfo(),
                getClassLoader(), null);
    }
    return mResources;
}

String[] getSplitPaths(String splitName) throws NameNotFoundException {
    if (mSplitLoader == null) {
        return mSplitResDirs;
    }
    return mSplitLoader.getSplitPathsForSplit(splitName);
}

private void setApplicationInfo(ApplicationInfo aInfo) {
    ...
    mSplitNames = aInfo.splitNames;
    mSplitAppDirs = aInfo.splitSourceDirs;
    mSplitResDirs = aInfo.uid == myUid ? aInfo.splitSourceDirs : aInfo.splitPublicSourceDirs;
    mSplitClassLoaderNames = aInfo.splitClassLoaderNames;
    ...
}
```

以上便是`Split Apks`加载过程，包括`code`和`resources`加载。

`Split Apks`并不支持动态加载，即`Base Apk`和`Split Apks`在`App`安装时，必须全部安装。但通过`Split Apks`工作原理，可以发现其是能够支持按需加载。

#### 动态加载

`SplitInstallHelper.updateAppInfo`方法

```Java
// SplitInstallHelper
public static void updateAppInfo(Context var0) {
    if (VERSION.SDK_INT > 25 && VERSION.SDK_INT < 28) {
        Class var1 = Class.forName("android.app.ActivityThread");
        Method var2 = var1.getMethod("currentActivityThread");
        var2.setAccessible(true);
        Object[] var3 = new Object[0];
        Object var8 = var2.invoke((Object)null, var3);
        Field var6 = var1.getDeclaredField("mAppThread");
        var6.setAccessible(true);
        Object var7 = var6.get(var8);
        Class var9 = var7.getClass();
        Class[] var10 = new Class[]{Integer.TYPE, String[].class};
        var2 = var9.getMethod("dispatchPackageBroadcast", var10);
        var3 = new Object[]{3, null};
        String[] var4 = new String[]{var0.getPackageName()};
        var3[1] = var4;
        var2.invoke(var7, var3);
        ...
    }
}
```

方法会通过反射调用`ActivityThread.AplicationThread.dispatchPackageBroadcast`方法，传入参数为3。

```Java
// ActivityThread
final void handleDispatchPackageBroadcast(int cmd, String[] packages) {
    switch (cmd) {
        // ApplicationThreadConstants.PACKAGE_REPLACED = 3
        case ApplicationThreadConstants.PACKAGE_REPLACED:{
            ...
            final ArrayList<String> oldPaths = new ArrayList<>();
            LoadedApk.makePaths(this, pkgInfo.getApplicationInfo(), oldPaths);
            pkgInfo.updateApplicationInfo(aInfo, oldPaths);
        }
    }
}

// LoadedApk
public void updateApplicationInfo(@NonNull ApplicationInfo aInfo,
        @Nullable List<String> oldPaths) {
    final List<String> newPaths = new ArrayList<>();
    makePaths(mActivityThread, aInfo, newPaths);
    ...
    synchronized (this) {
        createOrUpdateClassLoaderLocked(addedPaths);
        if (mResources != null) {
            final String[] splitPaths;
            try {
                splitPaths = getSplitPaths(null);
            } catch (NameNotFoundException e) {
                // This should NEVER fail.
                throw new AssertionError("null split not found");
            }

            mResources = ResourcesManager.getInstance().getResources(null, mResDir,
                    splitPaths, mOverlayDirs, mApplicationInfo.sharedLibraryFiles,
                    Display.DEFAULT_DISPLAY, null, getCompatibilityInfo(),
                    getClassLoader(), mApplication == null ? null
                            : mApplication.getResources().getLoaders());
        }
    }
}
```

可以看到，最终会调用`LoadedApk的updateApplicationInfo`方法，还是会走`Split Apks`那套逻辑。