---
title: Verification
date: 2018-08-02 01:20:43
tags: 权限
categories: Android
---

Verification介绍.其代码在InstallParams的handleStartCopy中

<!--more-->

# Verification介绍

```java
......//此处已经获得了合适的安装位置
finalInstallArgs args = createInstallArgs(this);
mArgs =args;
if (ret == PackageManager.INSTALL_SUCCEEDED) {
   final int requiredUid =mRequiredVerifierPackage == null ? -1
                        :getPackageUid(mRequiredVerifierPackage);
   if (requiredUid != -1 &&isVerificationEnabled()) {
   //创建一个Intent，用于查找满足条件的广播接收者
   finalIntent verification = new
                    Intent(Intent.ACTION_PACKAGE_NEEDS_VERIFICATION);
   verification.setDataAndType(packageURI, PACKAGE_MIME_TYPE);
   verification.addFlags(Intent.FLAG_GRANT_READ_URI_PERMISSION);
   //查找满足Intent条件的广播接收者
   finalList<ResolveInfo> receivers = queryIntentReceivers(
               verification,null,PackageManager.GET_DISABLED_COMPONENTS);
   // verificationId为当前等待Verification的安装包个数
   finalint verificationId = mPendingVerificationToken++;
   //设置Intent的参数，例如要校验的包名
   verification.putExtra(PackageManager.EXTRA_VERIFICATION_ID,
                              VerificationId);
   verification.putExtra(
                            PackageManager.EXTRA_VERIFICATION_INSTALLER_PACKAGE,
                            installerPackageName);
   verification.putExtra(
                   PackageManager.EXTRA_VERIFICATION_INSTALL_FLAGS,flags);
  if(verificationURI != null) {
     verification.putExtra(PackageManager.EXTRA_VERIFICATION_URI,
                                verificationURI);
  }
  finalPackageVerificationState verificationState = new
                       PackageVerificationState(requiredUid,args);
  //将上面创建的PackageVerificationState保存到mPendingVerification中
  mPendingVerification.append(verificationId, verificationState);
  //筛选符合条件的广播接收者
  finalList<ComponentName> sufficientVerifiers =
                   matchVerifiers(pkgLite,receivers,verificationState);
  if (sufficientVerifiers != null) {
      finalint N = sufficientVerifiers.size();
      ......
      for(int i = 0; i < N; i++) {
        finalComponentName verifierComponent = sufficientVerifiers.get(i);
        final Intent sufficientIntent = newIntent(verification);
        sufficientIntent.setComponent(verifierComponent);
        //向校验包发送广播
        mContext.sendBroadcast(sufficientIntent);
       }
    }
 }
 //除此之外，如果在执行adb install的时候指定了校验包，则需要向其单独发送校验广播
 finalComponentName requiredVerifierComponent =
                        matchComponentForVerifier(mRequiredVerifierPackage,
                       receivers);
 if (ret == PackageManager.INSTALL_SUCCEEDED
        &&mRequiredVerifierPackage != null) {
      verification.setComponent(requiredVerifierComponent);
      mContext.sendOrderedBroadcast(verification,
      android.Manifest.permission.PACKAGE_VERIFICATION_AGENT,
           new BroadcastReceiver() {
           //调用sendOrderdBroadcast，并传递一个BroadcastReceiver，该对象将在
          //广播发送的最后被调用。读者可参考sendOrderdBroadcast的文档说明
           public void onReceive(Context context, Intent intent) {
            final Message msg =mHandler.obtainMessage(
                      CHECK_PENDING_VERIFICATION);
            msg.arg1 = verificationId;
            //设置一个超时执行时间，该值来自Settings数据库的secure表，默认为60秒
            mHandler.sendMessageDelayed(msg, getVerificationTimeout());
           }
         },null, 0, null, null);
           mArgs = null;
        }
    }......//不用做Verification的流程
```
PKMS的Verification工作其实就是收集安装包的信息，然后向对应的校验者发送广播。但遗憾的是，当前Android中还没有能处理Verification的组件。

另外，该组件处理完Verification后，需要调用PKMS的verifyPendingInstall函数，以通知校验结果。
