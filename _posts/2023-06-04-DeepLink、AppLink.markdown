---
layout: post
title:  "DeepLink、AppLink"
date:   2023-06-04
toc:  true
tags: [Android源码解读]
---
Deeplink深层链接

## 定义

### DeepLink

DeepLink是指将用户直接转到应用app中的特定内容的网址。在Android中，可以通过添加`intent-filter`来设置DeepLink，以便将用户吸引到正确的`Activity`。不过，如果用户设备上安装的其他应用可以处理相同的`intent`，则用户可能无法直接进入您的应用。例如，点击银行发来的电子邮件中的网址可能会显示一个对话框，询问用户是使用浏览器还是银行自己的应用打开此链接。

- eg.`snssdk1233://aweme/detail/12345678`

### AppLink

AppLink是一种特殊类型的DeepLink，在Android 6.0及更高版本中可以使用。AppLink能使应用将自己指定为给定类型链接的默认处理程序，从而绕过应用选择对话框。如果用户不想使用某个应用作为默认处理程序，则可以从设备的系统设置中替换此行为。

AppLink可以带来以下好处：

- **安全且具体**：AppLink使用链接到自己网站网域的 HTTP 网址，因此其他应用都无法使用这个链接。AppLink的要求之一，就是要通过对网域所有权的验证。
- **顺畅的用户体验**：由于AppLink对网站和应用中的相同内容使用相同HTTP 网址，因此未安装应用的用户会直接转到网站（而不是应用），不会显示404，也不会出现错误。
- **Android 免安装应用支持**：借助Android免安装应用，用户无需安装即可运行Android应用。
- **通过 Google 搜索吸引用户**：用户可以通过在移动浏览器、Google搜索应用、Android中的屏幕搜索或通过 Google 助理点击来自Google的网址，直接打开应用中的特定内容。

## 使用方式

### 为DeepLink添加`intent-filter`

如需创建指向应用的DeepLink，须在清单中添加一个包含以下元素和属性值的`intent-filter`：

- `<action>`
  - 指定 `ACTION_VIEW`intent操作，以便能够从Google搜索中访问此`intent-filter`。
- `<data>`
  - 添加一个或多个`<data>`标记，每个标记都代表一个解析为`Activity`的URI格式。`<data>` 标记必须至少包含`android:scheme`属性。
- `<category>`
  - 包含`BROWSABLE`类别。如果要从网络浏览器中访问`intent-filter`，就必须提供该类别。否则，在浏览器中点击链接便无法解析为你的应用。
  - 此外，还要包含`DEFAULT`类别。这样您的应用才可以响应隐式`intent`。否则，只有在`intent`指定您的应用组件名称时，Activity才能启动。

以下 XML 代码段展示了如何在清单中为DeepLink指定`intent-filter`。URI `example://gizmos` 和 `http://www.example.com/gizmos` 都会解析到此`Activity`。

```xml
<activity
    android:name="com.example.android.GizmosActivity"
    android:label="@string/title_gizmos" >
    
    <intent-filter android:label="@string/filter_view_http_gizmos">
        <action android:name="android.intent.action.VIEW" />
        <category android:name="android.intent.category.DEFAULT" />
        <category android:name="android.intent.category.BROWSABLE" />
        <!-- Accepts URIs that begin with "http://www.example.com/gizmos” -->
        <data android:scheme="http"
              android:host="www.example.com"
              android:pathPrefix="/gizmos" />
        <!-- note that the leading "/" is required for pathPrefix-->
    </intent-filter>
    
    <intent-filter android:label="@string/filter_view_example_gizmos">
        <action android:name="android.intent.action.VIEW" />
        <category android:name="android.intent.category.DEFAULT" />
        <category android:name="android.intent.category.BROWSABLE" />
        <!-- Accepts URIs that begin with "example://gizmos” -->
        <data android:scheme="example"
              android:host="gizmos" />
    </intent-filter>
    
</activity>
```

注意，`<data>`元素是这两个`intent-filter`的唯一区别。虽然同一过滤器可以包含多个 `<data>` 元素，但如果想要声明唯一网址（例如特定的 `scheme` 和 `host` 组合），则创建单独的过滤器很重要，因为同一`intent-filter`中的多个`<data>`元素实际上会合并在一起以涵盖合并后属性的所有变体。

```xml
    <intent-filter>
      ...
      <data android:scheme="https" android:host="www.example.com" />
      <data android:scheme="app" android:host="open.my.app" />
    </intent-filter>
```

看起来这似乎仅支持`https://www.example.com`和`app://open.my.app`。但是，实际上除了这两种之外，它还支持`app://www.example.com`和`https://open.my.app`。

当向应用清单添加包含`Activity`内容URI的`intent-filter`后，Android可以在运行时将所有包含匹配URI的 `Intent` 转发到应用。

### 验证 AppLink

如需向应用添加AppLink，在定义使用HTTP网址打开应用的`intent-filter`之后，还需要验证你是否为相关应用和网站网址的所有者。如果系统成功验证你是网址所有者，则会自动将这些网址`Intent`路由到你的应用。

1. 在清单中请求自动验证应用链接
   
   在应用清单中的任一网址`intent-filter`中设置 `android:autoVerify="true"`。这样即可向Android系统说明其应该验证你的应用是否属于`intent-filter`中使用的网址网域。
   
   ```xml
   <activity ...>
        <intent-filter android:autoVerify="true">
            <action android:name="android.intent.action.VIEW" />
            <category android:name="android.intent.category.DEFAULT" />
            <category android:name="android.intent.category.BROWSABLE" />
            <data android:scheme="http" android:host="www.example.com" />
            <data android:scheme="https" />
        </intent-filter>
    </activity>
   ```

   当任一`intent-filter`上存在 `android:autoVerify="true"` 时，在搭载Android 6.0及更高版本的设备上安装应用会使系统尝试验证与应用的所有`intent-filter`中的网址关联的所有主机。验证涉及以下方面：
   
   1. 系统会检查所有包含以下各项的`intent-filter`：
       1. `action`：`android.intent.action.VIEW`
       2. `category`：`android.intent.category.BROWSABLE` 和 `android.intent.category.DEFAULT`
       3. `scheme`：`http` 或 `https`
   2. 对于在上述`intent-filter`中找到的每个唯一主机名，Android会在 `https://hostname/.well-known/assetlinks.json` 查询Digital Asset Links文件的相应网站。
   3. 仅当系统为清单中的所有主机找到匹配的Digital Asset Links文件后，才会将你的应用确立为处理指定网址格式的默认处理程序。
   
2. 声明网站关联性
   1. 你必须在`https://hostname/.well-known/assetlinks.json`发布[Digital Asset Links](https://developers.google.com/digital-asset-links/v1/getting-started?hl=zh-cn) JSON文件，以指示与网站相关联的Android应用并验证应用的网址`Intent`。
      1.   JSON文件使用下列字段标识关联的应用：

      2. `package_name`：在应用的`build.gradle`文件中声明的应用 ID。
      
      3. `sha256_cert_fingerprints`：应用的签名证书的SHA256指纹。可以利用 Java 密钥工具，通过以下命令生成该指纹：
      
           ```
           $ keytool -list -v -keystore my-release-key.keystore
           ```
      
      4. 以下`assetlinks.json`示例文件可为`com.example`应用授予链接打开权限：
      
           ```
           [{
             "relation": ["delegate_permission/common.handle_all_urls"],
             "target": {
               "namespace": "android_app",
               "package_name": "com.example",
               "sha256_cert_fingerprints":
               ["14:6D:E9:83:C5:73:06:50:D8:EE:B9:95:2F:34:FC:64:16:A0:83:42:E6:1D:BE:A8:8A:04:96:B2:3F:CF:44:E5"]
             }
           }]
           ```

### DeepLink跳转

```kotlin
val intent = Intent(Intent.ACTION_VIEW)
intent.addCategory(Intent.CATEGORY_BROWSABLE)
intent.addCategory(Intent.CATEGORY_DEFAULT)
intent.data = Uri.parse("https://www.example.com")
startActivity(intent)
```

### 测试

```
$ adb shell am start
        -W -a android.intent.action.VIEW
        -d <URI>
```

## 实现原理

从`Activity.startActivity`方法开始看起

```java
// Activity
@Override
public void startActivity(Intent intent) {
    this.startActivity(intent, null);
}

@Override
public void startActivity(Intent intent, @Nullable Bundle options) {
    if (options != null) {
        startActivityForResult(intent, -1, options);
    } else {
        startActivityForResult(intent, -1);
    }
}

public void startActivityForResult(@RequiresPermission Intent intent, int requestCode) {
    startActivityForResult(intent, requestCode, null);
}

public void startActivityForResult(@RequiresPermission Intent intent, int requestCode,
                                   @Nullable Bundle options) {
    if (mParent == null) {
        Instrumentation.ActivityResult ar =
                mInstrumentation.execStartActivity(
                        this, mMainThread.getApplicationThread(), mToken, this,
                        intent, requestCode, options);
        if (ar != null) {
            mMainThread.sendActivityResult(
                    mToken, mEmbeddedID, requestCode, ar.getResultCode(),
                    ar.getResultData());
        }
        ...
    }
    ...
}
```

走进`Instrumentation.execStartActivity`方法

```java
// Instrumentation
@UnsupportedAppUsage
public ActivityResult execStartActivity(
        Context who, IBinder contextThread, IBinder token, Activity target,
        Intent intent, int requestCode, Bundle options) {
    ...
    try {
        int result = ActivityTaskManager.getService()
                .startActivity(whoThread, who.getBasePackageName(), intent,
                        intent.resolveTypeIfNeeded(who.getContentResolver()),
                        token, target != null ? target.mEmbeddedID : null,
                        requestCode, 0, null, options);
    } catch (RemoteException e) {
        throw new RuntimeException("Failure from system", e);
    }
    return null;
}
```

继续走进`ActivityTaskManagerService.startActivity`方法

```java
// ActivityTaskManagerService
@Override
public final int startActivity(IApplicationThread caller, String callingPackage,
        Intent intent, String resolvedType, IBinder resultTo, String resultWho, int requestCode,
        int startFlags, ProfilerInfo profilerInfo, Bundle bOptions) {
    return startActivityAsUser(caller, callingPackage, intent, resolvedType, resultTo,
            resultWho, requestCode, startFlags, profilerInfo, bOptions,
            UserHandle.getCallingUserId());
}

@Override
public int startActivityAsUser(IApplicationThread caller, String callingPackage,
        Intent intent, String resolvedType, IBinder resultTo, String resultWho, int requestCode,
        int startFlags, ProfilerInfo profilerInfo, Bundle bOptions, int userId) {
    return startActivityAsUser(caller, callingPackage, intent, resolvedType, resultTo,
            resultWho, requestCode, startFlags, profilerInfo, bOptions, userId,
            true /*validateIncomingUser*/);
}

int startActivityAsUser(IApplicationThread caller, String callingPackage,
        Intent intent, String resolvedType, IBinder resultTo, String resultWho, int requestCode,
        int startFlags, ProfilerInfo profilerInfo, Bundle bOptions, int userId,
        boolean validateIncomingUser) {
    ...
    return getActivityStartController().obtainStarter(intent, "startActivityAsUser")
            .setCaller(caller)
            .setCallingPackage(callingPackage)
            .setResolvedType(resolvedType)
            .setResultTo(resultTo)
            .setResultWho(resultWho)
            .setRequestCode(requestCode)
            .setStartFlags(startFlags)
            .setProfilerInfo(profilerInfo)
            .setActivityOptions(bOptions)
            .setMayWait(userId)
            .execute();

}
```

继续走到`ActivityStarter.execute`方法

```java
// ActivityStarter
int execute() {
    ...
    if (mRequest.mayWait) {
        return startActivityMayWait(mRequest.caller, mRequest.callingUid,
                mRequest.callingPackage, mRequest.realCallingPid, mRequest.realCallingUid,
                mRequest.intent, mRequest.resolvedType,
                mRequest.voiceSession, mRequest.voiceInteractor, mRequest.resultTo,
                mRequest.resultWho, mRequest.requestCode, mRequest.startFlags,
                mRequest.profilerInfo, mRequest.waitResult, mRequest.globalConfig,
                mRequest.activityOptions, mRequest.ignoreTargetSecurity, mRequest.userId,
                mRequest.inTask, mRequest.reason,
                mRequest.allowPendingRemoteAnimationRegistryLookup,
                mRequest.originatingPendingIntent, mRequest.allowBackgroundActivityStart);
    }
    ...
}

private int startActivityMayWait(IApplicationThread caller, int callingUid,
        String callingPackage, int requestRealCallingPid, int requestRealCallingUid,
        Intent intent, String resolvedType, IVoiceInteractionSession voiceSession,
        IVoiceInteractor voiceInteractor, IBinder resultTo, String resultWho, int requestCode,
        int startFlags, ProfilerInfo profilerInfo, WaitResult outResult,
        Configuration globalConfig, SafeActivityOptions options, boolean ignoreTargetSecurity,
        int userId, TaskRecord inTask, String reason,
        boolean allowPendingRemoteAnimationRegistryLookup,
        PendingIntentRecord originatingPendingIntent, boolean allowBackgroundActivityStart) {

    ...
    ResolveInfo rInfo = mSupervisor.resolveIntent(intent, resolvedType, userId,
        0 /* matchFlags */,
                computeResolveFilterUid(
                        callingUid, realCallingUid, mRequest.filterCallingUid));
    ...
    final ActivityRecord[] outRecord = new ActivityRecord[1];
    int res = startActivity(caller, intent, ephemeralIntent, resolvedType, aInfo, rInfo,
            voiceSession, voiceInteractor, resultTo, resultWho, requestCode, callingPid,
            callingUid, callingPackage, realCallingPid, realCallingUid, startFlags, options,
            ignoreTargetSecurity, componentSpecified, outRecord, inTask, reason,
            allowPendingRemoteAnimationRegistryLookup, originatingPendingIntent,
            allowBackgroundActivityStart);
    ...
    return res;
}
```

核心逻辑在`ActivityStackSupervisor.resolveIntent`方法里

```java
// ActivityStackSupervisor
ResolveInfo resolveIntent(Intent intent, String resolvedType, int userId, int flags,
        int filterCallingUid) {
    ...
    final long token = Binder.clearCallingIdentity();
    try {
        return mService.getPackageManagerInternalLocked().resolveIntent(
                intent, resolvedType, modifiedFlags, userId, true, filterCallingUid);
    } finally {
        Binder.restoreCallingIdentity(token);
    }
}
```

继续走进`PackageManagerService.resolveIntent`方法

```java
// PackageManagerService
@Override
public ResolveInfo resolveIntent(Intent intent, String resolvedType,
        int flags, int userId, boolean resolveForStart, int filterCallingUid) {
    return resolveIntentInternal(
            intent, resolvedType, flags, userId, resolveForStart, filterCallingUid);
}

private ResolveInfo resolveIntentInternal(Intent intent, String resolvedType,
        int flags, int userId, boolean resolveForStart, int filterCallingUid) {
   ...
    final List<ResolveInfo> query = queryIntentActivitiesInternal(intent, resolvedType,
            flags, filterCallingUid, userId, resolveForStart, true /*allowDynamicSplits*/);
    final ResolveInfo bestChoice =
            chooseBestActivity(intent, resolvedType, flags, query, userId);
    return bestChoice;
}
```

`PackageManagerService`里做了两步操作：

1. 通过`queryIntentActivitiesInternal`方法找到所有适合的`Activity`
2. 再通过`chooseBestActivity` 方法选择出最终的`Activity`

### `PackageManagerService.queryIntentActivitiesInternal`

```java
// PackageManagerService
private @NonNull List<ResolveInfo> queryIntentActivitiesInternal(Intent intent,
        String resolvedType, int flags, int filterCallingUid, int userId,
        boolean resolveForStart, boolean allowDynamicSplits) {
    ...
    final String pkgName = intent.getPackage();
    ComponentName comp = intent.getComponent();
    if (comp == null) {
        if (intent.getSelector() != null) {
            intent = intent.getSelector();
            comp = intent.getComponent();
        }
    }
    if (comp != null) {
        final List<ResolveInfo> list = new ArrayList<>(1);
        final ActivityInfo ai = getActivityInfo(comp, flags, userId);
        if (ai != null) {
            final boolean blockResolution = !isTargetSameInstantApp
                && ((!matchInstantApp && !isCallerInstantApp && isTargetInstantApp)
                        || (matchVisibleToInstantAppOnly && isCallerInstantApp
                                && isTargetHiddenFromInstantApp));
            if (!blockResolution) {
                final ResolveInfo ri = new ResolveInfo();
                ri.activityInfo = ai;
                list.add(ri);
            }
        }
        return applyPostResolutionFilter(
            list, instantAppPkgName, allowDynamicSplits, filterCallingUid, resolveForStart,
            userId, intent);
    }
    ...
    List<ResolveInfo> result;
    if (pkgName == null) {
        result = filterIfNotSystemUser(mComponentResolver.queryActivities(
            intent, resolvedType, flags, userId), userId);
    } else {
        final PackageParser.Package pkg = mPackages.get(pkgName);
        result = null;
        if (pkg != null) {
            result = filterIfNotSystemUser(mComponentResolver.queryActivities(
            intent, resolvedType, flags, pkg.activities, userId), userId);
        }
    }
}
```

`queryIntentActivitiesInternal`方法分为三种情况：

1. 如果`intent.getComponent() != null`，即为显示`Intent`，则通过`getActivityInfo`方法就可以找到符合要求的`Activity`
2. 如果没有设置`Component`，即为隐式`Intent`。并且`intent.getPackage() == null`，则遍历查找所有应用程序的`Activity`
3. 如果没有设置`Component`，但是设置了`Package`，就去指定包下的`Activity`列表中查找

后面两种情况Deeplink都有可能走到，我们依次来看。先看没有设置包名的情况

#### `intent.getPackage() == null`

```java
// ComponentResolver
List<ResolveInfo> queryActivities(Intent intent, String resolvedType, int flags, int userId) {
    synchronized (mLock) {
        return mActivities.queryIntent(intent, resolvedType, flags, userId);
    }
}

private static final class ActivityIntentResolver
        extends IntentResolver<PackageParser.ActivityIntentInfo, ResolveInfo> {
        
    List<ResolveInfo> queryIntent(Intent intent, String resolvedType, int flags,
            int userId) {
        ...
        return super.queryIntent(intent, resolvedType,
                (flags & PackageManager.MATCH_DEFAULT_ONLY) != 0,
                userId);
    }
}

// IntentResolver
public List<R> queryIntent(Intent intent, String resolvedType, boolean defaultOnly,
        int userId) {
    String scheme = intent.getScheme();
    ArrayList<R> finalList = new ArrayList<R>();
    F[] firstTypeCut = null;
    F[] secondTypeCut = null;
    F[] thirdTypeCut = null;
    F[] schemeCut = null;
    
    // If the intent includes a MIME type, then we want to collect all of
    // the filters that match that MIME type.
    if (resolvedType != null) {
        int slashpos = resolvedType.indexOf('/');
        if (slashpos > 0) {
            final String baseType = resolvedType.substring(0, slashpos);
            if (!baseType.equals("*")) {
                if (resolvedType.length() != slashpos+2
                        || resolvedType.charAt(slashpos+1) != '*') {
                    // Not a wild card, so we can just look for all filters that
                    // completely match or wildcards whose base type matches.
                    firstTypeCut = mTypeToFilter.get(resolvedType);
                    secondTypeCut = mWildTypeToFilter.get(baseType);
                } else {
                    // We can match anything with our base type.
                    firstTypeCut = mBaseTypeToFilter.get(baseType);
                    secondTypeCut = mWildTypeToFilter.get(baseType);
                }
                // Any */* types always apply, but we only need to do this
                // if the intent type was not already */*.
                thirdTypeCut = mWildTypeToFilter.get("*");
            } else if (intent.getAction() != null) {
                // The intent specified any type ({@literal *}/*).  This
                // can be a whole heck of a lot of things, so as a first
                // cut let's use the action instead.
                firstTypeCut = mTypedActionToFilter.get(intent.getAction());
            }
        }
    }
    if (scheme != null) {
        schemeCut = mSchemeToFilter.get(scheme);
    }
    if (resolvedType == null && scheme == null && intent.getAction() != null) {
        firstTypeCut = mActionToFilter.get(intent.getAction());
    }

    FastImmutableArraySet<String> categories = getFastIntentCategories(intent);
    if (firstTypeCut != null) {
        buildResolveList(intent, categories, debug, defaultOnly, resolvedType,
                scheme, firstTypeCut, finalList, userId);
    }
    if (secondTypeCut != null) {
        buildResolveList(intent, categories, debug, defaultOnly, resolvedType,
                scheme, secondTypeCut, finalList, userId);
    }
    if (thirdTypeCut != null) {
        buildResolveList(intent, categories, debug, defaultOnly, resolvedType,
                scheme, thirdTypeCut, finalList, userId);
    }
    if (schemeCut != null) {
        buildResolveList(intent, categories, debug, defaultOnly, resolvedType,
                scheme, schemeCut, finalList, userId);
    }
    sortResults(finalList);
    ...
    return finalList;
}
```

```java
// IntentResolver
/**
 * All filters that have been registered.
 */
private final ArraySet<F> mFilters = new ArraySet<F>();

/**
 * All of the MIME types that have been registered, such as "image/jpeg",
 * "image/*", or "{@literal *}/*".
 */
private final ArrayMap<String, F[]> mTypeToFilter = new ArrayMap<String, F[]>();

/**
 * The base names of all of all fully qualified MIME types that have been
 * registered, such as "image" or "*".  Wild card MIME types such as
 * "image/*" will not be here.
 */
private final ArrayMap<String, F[]> mBaseTypeToFilter = new ArrayMap<String, F[]>();

/**
 * The base names of all of the MIME types with a sub-type wildcard that
 * have been registered.  For example, a filter with "image/*" will be
 * included here as "image" but one with "image/jpeg" will not be
 * included here.  This also includes the "*" for the "{@literal *}/*"
 * MIME type.
 */
private final ArrayMap<String, F[]> mWildTypeToFilter = new ArrayMap<String, F[]>();

/**
 * All of the URI schemes (such as http) that have been registered.
 */
private final ArrayMap<String, F[]> mSchemeToFilter = new ArrayMap<String, F[]>();

/**
 * All of the actions that have been registered, but only those that did
 * not specify data.
 */
private final ArrayMap<String, F[]> mActionToFilter = new ArrayMap<String, F[]>();

/**
 * All of the actions that have been registered and specified a MIME type.
 */
private final ArrayMap<String, F[]> mTypedActionToFilter = new ArrayMap<String, F[]>();
```

`queryIntent`方法会根据`Intent`的`resolvedType`、`scheme`、`action`等属性进行匹配，最终存储到`finalList`中再进行一轮排序后返回给调用方所有满足条件的`Activity`。匹配规则如下：

1. 如果`Intent`的`resolvedType != null`（`Intent`设置了`android:mimeType`）
   1. 如果`resolvedType`"/"后面不是通配符"*"，且"/"前面也不是通配符"*"，以"text/plain"为例，优先筛选出设置了`android:mimeType="text/plain"`的`Activity`数组`firstTypeCut`，其次筛选出设置了`android:mimeType="text/*"`的`Activity`数组`secondTypeCut`，最后筛选出设置了`android:mimeType="*/*"`的`Activity`数组`thirdTypeCut`
   2. 如果`resolvedType`"/"后面是"*"，但"/"前面不是"*"，以"text/*"为例，优先筛选出设置了`android:mimeType="text/plain"`的`Activity`数组`firstTypeCut`，其次筛选出设置了`android:mimeType="text/*"`的`Activity`数组`secondTypeCut`，最后筛选出设置了`android:mimeType="*/*"`的`Activity`数组`thirdTypeCut`
   3. 如果`resolvedType`是"/"前面是"*"，筛选出满足相同`Action`条件的`Activity`数组`firstTypeCut`
2. 如果`intent.getScheme() != null`（`Intent`设置了`android:scheme`）
   1. 筛选出满足相同scheme的`Activity`数组中`schemeCut`
3. 如果`resolvedType==null && intent.getScheme() == null`，但是`intent.getAction() != null`
   1. 筛选出满足相同`Action`条件的`Activity`数组`firstTypeCut`
4. 如果`resolvedType==null`且`intent.getScheme() == null`且`intent.getAction() == null`
   1. 返回空列表

到此得到了最多四个数组`firstTypeCut`、`secondTypeCut`、`thirdTypeCut`、`schemeCut`，这四个数组还需进行一轮匹配，具体匹配规则在`buildResolveList`方法中，我们继续往下看

```java
// IntentResolver
private void buildResolveList(Intent intent, FastImmutableArraySet<String> categories,
        boolean debug, boolean defaultOnly, String resolvedType, String scheme,
        F[] src, List<R> dest, int userId) {
    final String action = intent.getAction();
    final Uri data = intent.getData();
    final String packageName = intent.getPackage();   
    ...
    final int N = src != null ? src.length : 0; 
    for (i=0; i<N && (filter=src[i]) != null; i++) {
        int match;
        ... 
        match = filter.match(action, resolvedType, scheme, data, categories, TAG);
        if (match >= 0) {
            if (!defaultOnly || filter.hasCategory(Intent.CATEGORY_DEFAULT)) {
                final R oneResult = newResult(filter, match, userId);
                if (oneResult != null) {
                    dest.add(oneResult);
                }
            } else {
                hasNonDefaults = true;
            }
        } 
        ...
    }   
    ...
}

// IntentFilter
public final int match(String action, String type, String scheme,
        Uri data, Set<String> categories, String logTag) {
    if (action != null && !matchAction(action)) {
        return NO_MATCH_ACTION;
    }

    int dataMatch = matchData(type, scheme, data);
    if (dataMatch < 0) {
        return dataMatch;
    }

    String categoryMismatch = matchCategories(categories);
    if (categoryMismatch != null) {
        return NO_MATCH_CATEGORY;
    }
    return dataMatch;
}
```

```java
// IntentFilter
public final boolean matchAction(String action) {
    return hasAction(action);
}

public final boolean hasAction(String action) {
    return action != null && mActions.contains(action);
}

public final int matchData(String type, String scheme, Uri data) {
    final ArrayList<String> types = mDataTypes;
    final ArrayList<String> schemes = mDataSchemes;

    int match = MATCH_CATEGORY_EMPTY;

    if (types == null && schemes == null) {
        return ((type == null && data == null) ? (MATCH_CATEGORY_EMPTY+MATCH_ADJUSTMENT_NORMAL) : NO_MATCH_DATA);
    }

    if (schemes != null) {
        if (schemes.contains(scheme != null ? scheme : "")) {
            match = MATCH_CATEGORY_SCHEME;
        } else {
            return NO_MATCH_DATA;
        }

        final ArrayList<PatternMatcher> schemeSpecificParts = mDataSchemeSpecificParts;
        if (schemeSpecificParts != null && data != null) {
            match = hasDataSchemeSpecificPart(data.getSchemeSpecificPart())
                    ? MATCH_CATEGORY_SCHEME_SPECIFIC_PART : NO_MATCH_DATA;
        }
        ...
        if (match == NO_MATCH_DATA) {
            return NO_MATCH_DATA;
        }
    } else {
        if (scheme != null && !"".equals(scheme)
                && !"content".equals(scheme)
                && !"file".equals(scheme)) {
            return NO_MATCH_DATA;
        }
    }

    if (types != null) {
        if (findMimeType(type)) {
            match = MATCH_CATEGORY_TYPE;
        } else {
            return NO_MATCH_TYPE;
        }
    } else {
        if (type != null) {
            return NO_MATCH_TYPE;
        }
    }

    return match + MATCH_ADJUSTMENT_NORMAL;
}

public final String matchCategories(Set<String> categories) {
    if (categories == null) {
        return null;
    }

    Iterator<String> it = categories.iterator();

    if (mCategories == null) {
        return it.hasNext() ? it.next() : null;
    }

    while (it.hasNext()) {
        final String category = it.next();
        if (!mCategories.contains(category)) {
            return category;
        }
    }

    return null;
}
```

必须满足`matchAction`、`matchData`、`matchCategories`全部三个方法且`Activity`声明了`Intent.CATEGORY_DEFAULT`才会将对应`Activity`放进`finalList`中。

1. `matchAction`方法需要`Intent`的`Action`和`Activity`的`Action`相一致
2. `matchData`方法便是找出能够与`intent.getData`相匹配的`Activity`，会根据`Activity`的`android:scheme`、`android:host`、`android:path`等属性进行匹配
3. `matchCategories`方法需要`Activity`的`category`列表包含`Intent`所有`category`

到此，便找到了所有满足条件的`Activity`，根据优先级排个序后便返回结果。

```java
// IntentResolver
protected void sortResults(List<R> results) {
    Collections.sort(results, mResolvePrioritySorter);
}

private static final Comparator mResolvePrioritySorter = new Comparator() {
    public int compare(Object o1, Object o2) {
        final int q1 = ((IntentFilter) o1).getPriority();
        final int q2 = ((IntentFilter) o2).getPriority();
        return (q1 > q2) ? -1 : ((q1 < q2) ? 1 : 0);
    }
};
```

#### `intent.getPackage() != null`

```java
// ComponentResolver
List<ResolveInfo> queryIntentForPackage(Intent intent, String resolvedType,
        int flags, List<PackageParser.Activity> packageActivities, int userId) {
    ...
    final int activitiesSize = packageActivities.size();
    ArrayList<PackageParser.ActivityIntentInfo[]> listCut = new ArrayList<>(activitiesSize);

    ArrayList<PackageParser.ActivityIntentInfo> intentFilters;
    for (int i = 0; i < activitiesSize; ++i) {
        intentFilters = packageActivities.get(i).intents;
        if (intentFilters != null && intentFilters.size() > 0) {
            PackageParser.ActivityIntentInfo[] array =
                    new PackageParser.ActivityIntentInfo[intentFilters.size()];
            intentFilters.toArray(array);
            listCut.add(array);
        }
    }
    return super.queryIntentFromList(intent, resolvedType, defaultOnly, listCut, userId);
}

// IntentResolver
public List<R> queryIntentFromList(Intent intent, String resolvedType, boolean defaultOnly,
        ArrayList<F[]> listCut, int userId) {
    ArrayList<R> resultList = new ArrayList<R>();

    FastImmutableArraySet<String> categories = getFastIntentCategories(intent);
    final String scheme = intent.getScheme();
    int N = listCut.size();
    for (int i = 0; i < N; ++i) {
        buildResolveList(intent, categories, debug, defaultOnly, resolvedType, scheme,
                listCut.get(i), resultList, userId);
    }
    filterResults(resultList);
    sortResults(resultList);
    return resultList;
}
```

逻辑很类似，最后依然是调用`buildResolveList`找出匹配的`Activity`。但是是从指定包下的`Activity`列表中查找，相比没有设置包名的情况，大大节省了开销。

### `PackageManagerService.chooseBestActivity`

```java
private ResolveInfo chooseBestActivity(Intent intent, String resolvedType,
        int flags, List<ResolveInfo> query, int userId) {
    if (query != null) {
        final int N = query.size();
        if (N == 1) {
            return query.get(0);
        } else if (N > 1) {
            // If there is more than one activity with the same priority,
            // then let the user decide between them.
            ResolveInfo r0 = query.get(0);
            ResolveInfo r1 = query.get(1);
            // If the first activity has a higher priority, or a different
            // default, then it is always desirable to pick it.
            if (r0.priority != r1.priority
                    || r0.preferredOrder != r1.preferredOrder
                    || r0.isDefault != r1.isDefault) {
                return query.get(0);
            }
            ResolveInfo ri = findPreferredActivityNotLocked(intent, resolvedType,
                    flags, query, r0.priority, true, false, debug, userId);
            if (ri != null) {
                return ri;
            }
            ri = new ResolveInfo(mResolveInfo);
            ri.activityInfo = new ActivityInfo(ri.activityInfo);
            ri.activityInfo.labelRes = ResolverActivity.getLabelRes(intent.getAction());
            final String intentPackage = intent.getPackage();
            if (!TextUtils.isEmpty(intentPackage) && allHavePackage(query, intentPackage)) {
                final ApplicationInfo appi = query.get(0).activityInfo.applicationInfo;
                ri.resolvePackageName = intentPackage;
                if (userNeedsBadging(userId)) {
                    ri.noResourceId = true;
                } else {
                    ri.icon = appi.icon;
                }
                ri.iconResourceId = appi.icon;
                ri.labelRes = appi.labelRes;
            }
            ri.activityInfo.applicationInfo = new ApplicationInfo(
                    ri.activityInfo.applicationInfo);
            return ri;
        }
    }
    return null
}
```

`chooseBestActivity`分为四种情况：

1.  如果只有唯一一个的`Activity`，则直接返回该`Activity`
2. 如果有多个`Activity`，但是首个优先级更高，则选择首个
3. 如果有多个`Activity`，优先级也相同，则根据`findPreferredActivityNotLocked`方法找到用户有保存处理该`Intent`的`Activity`，如果有则直接返回
4. 如果有多个`Activity`，优先级相同，且用户没有保存处理该`Intent`的`Activity`时，选择`ResolverActivity`显示所有结果供用户选择

由于`findPreferredActivityNotLocked`方法源码较为复杂，感兴趣的同学可以自行研究。

参考

1. https://developer.android.com/training/app-links?hl=zh-cn

2. https://juejin.cn/post/6854573214819958791

3. https://www.cnblogs.com/cainyu/p/15045890.html

4. https://blog.csdn.net/qinjuning/article/details/7384906

注

1. 以上代码均基于API 30