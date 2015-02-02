title: Android Authenticator
tags:
  - Android
date: 2015-02-02 13:22:52
---

对用Android的Authenticator，官方的API Guides是似乎没有很好的介绍文档，Reference中也只是简单地介绍了一下。虽然[sync-adapters][1]中对于Authenticator有提及，但是对于其用法也没有一个完整的认识，所以对其进行一定的学习并且记录了下来。

为什么需要Account Manager
========================
我们完全可以自己自己的账户管理机制，提交一个登录表单到服务器然后返回一个认证token。但是通常覆盖不到一些细节，如果用户在另外一客户端修改了密码怎么办；认证token的过期处理；单点登录的实现。这些东西实现起来还是挺麻烦的，但是如果Android的认证框架已经实现了这些功能，并且额外提供了用户数据同步等额外的功能，那还有理由不使用它吗？Stop Trying to Reinvent the Wheel.

<!--more-->

实现类
=====
Account相关的类都位于`android.accounts`包下，以下是其中几个主要的类。

- AccountManager
    提供访问所有用户账户的注册中心。	不同的在线服务有不同的处理账户和认证的方式，所以AccountManager对不同的`account types`使用可插拔的`authenticator`的模块，由第三方提供的`Authenticators`处理实际的账户认证细节。

    AccountManager还可以为应用生成`auth token`，所以应用不必直接处理密码。`Auth tokens`通常被`AccountManager`缓存并重复利用。

- AccountAuthenticator
    上面提到的可插拔的`authenticator`模块，处理特定的`account types`。`AccountManager`通过查找到合适的AccountAuthenticator，与其进行通信来实现对特定的`account types`的所有操作。

- AccountAuthenticatorActivity
    当需要验证用户的时候，由`authenticator`调用，来进行“登录/创建 account”的Activity的基类。

调用流程
=======
以添加账户为例

关键代码
-------
```
public AccountManagerFuture<Bundle> addAccount(final String accountType,
        final String authTokenType, final String[] requiredFeatures,
        final Bundle addAccountOptions,
        final Activity activity, AccountManagerCallback<Bundle> callback, Handler handler) {
    if (accountType == null) throw new IllegalArgumentException("accountType is null");
    final Bundle optionsIn = new Bundle();
    if (addAccountOptions != null) {
        optionsIn.putAll(addAccountOptions);
    }
    optionsIn.putString(KEY_ANDROID_PACKAGE_NAME, mContext.getPackageName());

    return new AmsTask(activity, handler, callback) {
        ②public void doWork() throws RemoteException {
            mService.addAccount(mResponse, accountType, authTokenType,
                    requiredFeatures, activity != null, optionsIn);
        }
    }.start();
}
```

```
public void addAccount(IAccountAuthenticatorResponse response, String accountType,
        String authTokenType, String[] features, Bundle options)
        throws RemoteException {
    if (Log.isLoggable(TAG, Log.VERBOSE)) {
        Log.v(TAG, "addAccount: accountType " + accountType
                + ", authTokenType " + authTokenType
                + ", features " + (features == null ? "[]" : Arrays.toString(features)));
    }
    checkBinderPermission();
    try {
        final Bundle result = ③AbstractAccountAuthenticator.this.addAccount(
            new AccountAuthenticatorResponse(response),
                accountType, authTokenType, features, options);
        if (Log.isLoggable(TAG, Log.VERBOSE)) {
            result.keySet(); // force it to be unparcelled
            Log.v(TAG, "addAccount: result " + AccountManager.sanitizeResult(result));
        }
        if (result != null) {
            ④response.onResult(result);
        }
    } catch (Exception e) {
        handleException(response, "addAccount", accountType, e);
    }
}
```

```
private class Response extends IAccountManagerResponse.Stub {
    public void onResult(Bundle bundle) {
        Intent intent = bundle.getParcelable(KEY_INTENT);
        if (intent != null && mActivity != null) {
            // since the user provided an Activity we will silently start intents
            // that we see
            ⑤mActivity.startActivity(intent);
            // leave the Future running to wait for the real response to this request
        } else if (bundle.getBoolean("retry")) {
            try {
                doWork();
            } catch (RemoteException e) {
                // this will only happen if the system process is dead, which means
                // we will be dying ourselves
            }
        } else {
            ⑨set(bundle);
        }
    }
```

```
public void ⑥finish() {
    if (mAccountAuthenticatorResponse != null) {
        // send the result bundle back if set, otherwise send an error.
        if (mResultBundle != null) {
            ⑦mAccountAuthenticatorResponse.onResult(mResultBundle);
        } else {
            mAccountAuthenticatorResponse.onError(AccountManager.ERROR_CODE_CANCELED,
                    "canceled");
        }
        mAccountAuthenticatorResponse = null;
    }
    super.finish();
}
```
```
public void onResult(Bundle result) {
    if (Log.isLoggable(TAG, Log.VERBOSE)) {
        result.keySet(); // force it to be unparcelled
        Log.v(TAG, "AccountAuthenticatorResponse.onResult: "
                + AccountManager.sanitizeResult(result));
    }
    try {
        ⑧mAccountAuthenticatorResponse.onResult(result);
    } catch (RemoteException e) {
        // this should never happen
    }
}
```

时序图如下
![android_autheticator_flow](/img/android_autheticator_flow.png)

1. 客户端调用`AccountManager#addAccount`方法添加账户。
2. `AccountManager#addAccount`方法中会调用`AmsTask#start`并返回结果，`AmsTask#start`的主要内容是调用`doWork`方法。
3. `AmsTask#doWork`方法以AIDL方式调用`AbstractAccountAuthenticator#addAccount`方法。`AbstractAccountAuthenticator#addAccount`方法需要用户实现其逻辑。实现逻辑中需要告诉框架到哪个Activity进行具体的账户添加动作，因此需要返回一个Bundle包含以`AccountManager.KEY_INTENT`为key的Intent，Intent中需包含以`AccountManager.KEY_ACCOUNT_AUTHENTICATOR_RESPONSE`为key的`AccountAuthenticatorResponse`。
4. `AccountManager#addAccount`方法中后记调用`AmsTask#Response#onResult`方法
5. `AmsTask#Response#onResult`逻辑中会判断`AbstractAccountAuthenticator#addAccount`返回的Bundle中后否包含`AccountManager.KEY_INTENT`，如果包含则启动对应的Activity，所以#3中的实现方法必须设置`AccountManager.KEY_INTENT`。
6. 跳转的Activity需要继承`AccountAuthenticatorActivity`，该类的实现逻辑中可以调用`AccountManager#addAccountExplicitly`添加账户，`AccountManager#setPassword`更新密码，`AccountManager#setAuthToken`设置token等等。最终需要调用`AccountAuthenticatorActivity#setAccountAuthenticatorResult`方法设置返回结果并调用`AccountAuthenticatorActivity#finish`结束当前的Activity
7. `AccountAuthenticatorActivity#finish`方法会调用`AccountAuthenticatorResponse#onResult`并传递上部设置的返回结果。此处的`AccountAuthenticatorResponse`对象正式之前#3设置的`AccountManager.KEY_ACCOUNT_AUTHENTICATOR_RESPONSE`。`AccountAuthenticatorActivity`会在初始化的时候自动获取
	    protected void onCreate(Bundle icicle) {
	        super.onCreate(icicle);

	        mAccountAuthenticatorResponse =
	                getIntent().getParcelableExtra(AccountManager.KEY_ACCOUNT_AUTHENTICATOR_RESPONSE);

	        if (mAccountAuthenticatorResponse != null) {
	            mAccountAuthenticatorResponse.onRequestContinued();
	        }
	    }
8. `AccountAuthenticatorResponse`是之前`AmsTask#Response`的包装类，其`onResult`实现会调用`AmsTask#Response#onResult`方法
9. 由于这次Activity返回的Bundle并没有包含`AccountManager.KEY_INTENT`，所以会调用`AmsTask#set`方法
10. 客户端在#1调用了`AccountManager#addAccount`会返回实现了`AccountManagerFuture`接口的`AmsTask`对象，调用getResult方法可以获取结果，但是线程会阻塞轮询，直至#9完成为止。

当然并不是所有的Account操作都需要启动Activiy进行辅助的。有些逻辑可以直接在`AbstractAccountAuthenticator`子类方法中直接完成，然后返回一个结果。这些实现类的返回结果方式都是通过Bundle中设置`AccountManager`中定义的特殊Key来是实现的，例如`KEY_ACCOUNTS`，`KEY_BOOLEAN_RESULT`等等。这些方法不启动的Activiy，所以AccountManager的实现中是调用的`Future2Task`而不是`AmsTask`。

实现
====
Authenticator
-------------
Authenticator需要继承`AbstractAccountAuthenticator`。
###其中几个主要的方法

- addAccount
    添加账号，需要返回Intent跳转到AccountAuthenticatorActivity进行实际的登录

- confirmCredentials
    确认账号信息，也需要Activity的辅助，可以在一些账户切换的场合使用

- updateCredentials
    更新账号信息，也需要Activity的辅助，可以在密码变更，token失效的时候调用

- getAuthToken
    获取用户token，实现逻辑可以参考官方文档对`AccountManager#getAuthToken`的[描述][2]：如果之前已经生成了token并缓存，可以通过`AccountManager#peekAuthToken`返回之前缓存的token，否则，如果通过`AccountManager#getPassword`能够获取的用户密码，则直接申请token，否则，跳转到登陆页面。

一些上述方法中使用到的参数：
- accountType
    用于标示账户的类型，通常一种类型绑定到一个AccountAuthenticator。可以多个app使用同一个accountType实现单点登陆。
- authTokenType
    用于标示authToken的类型，同一账户在不同的app中获取的authToken可能拥有不同的scope，从而可以用authTokenType来标示该authToken的类型。


AccountAuthenticatorService
---------------------------
Authenticator的实现类最后会由`AccountManager`以AIDL的方式调用。所以实现需要注册响应的Service。`AbstractAccountAuthenticator`已经已经实现了`getIBinder`方法，所以只需在`onBind`调用该方法即可。
```
public class AccountAuthenticatorService extends Service {

    private static AccountAuthenticator AUTHENTICATOR;

    public IBinder onBind(Intent intent) {
        return intent.getAction().equals(ACTION_AUTHENTICATOR_INTENT) ? getAuthenticator()
                .getIBinder() : null;
    }

    private AccountAuthenticator getAuthenticator() {
        if (AUTHENTICATOR == null)
            AUTHENTICATOR = new AccountAuthenticator(this);
        return AUTHENTICATOR;
    }
}
```

同时还需要在Manifest中注册：
```
<service android:name=".account.AccountAuthenticatorService">
	<intent-filter>
		<action android:name="android.accounts.AccountAuthenticator"/>
	</intent-filter>
	<meta-data
		android:name="android.accounts.AccountAuthenticator"
		android:resource="@xml/authenticator"/>
</service>
```

其中meta-data引用的resource决定显示在Setting & Account中的内容
```
<account-authenticator xmlns:android="http://schemas.android.com/apk/res/android"
    android:accountType="com.jumper"
    android:icon="@drawable/app_icon"
    android:label="@string/app_name"
    android:smallIcon="@drawable/app_icon"
    android:accountPreferences="@xml/account_preferences"
/>
```
android:icon，android:label，android:smallIcon决定在Account中的显示
android:accountType是传入AccountAuthorization#addAccount的值

android:accountPreferences可以定义Setting & Account中偏好界面，从而对账户进行一些偏好设置
```
<PreferenceScreen xmlns:android="http://schemas.android.com/apk/res/android">
    <PreferenceCategory android:title="@string/title_fmt" />
    <PreferenceScreen
         android:key="key1"
         android:title="@string/key1_action"
         android:summary="@string/key1_summary">
         <intent
             android:action="key1.ACTION"
             android:targetPackage="key1.package"
             android:targetClass="key1.class" />
     </PreferenceScreen>
 </PreferenceScreen>
 ```




[1]: https://udinic.wordpress.com/2013/04/24/write-your-own-android-authenticator/
[2]: http://developer.android.com/reference/android/accounts/AccountManager.html#getAuthToken