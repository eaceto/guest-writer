---
layout: post
title: "Securing credentials in iOS"
description: "Behind the scenes of how your credentials are secured using iOS local authentication and biometry."
date: "2019-11-10 20:30"
author:
  name: "Ezequiel Aceto"
  url: "eaceto_pub"
  mail: "eaceto@pm.me"
  avatar: "https://twitter.com/eaceto_pub/profile_image?size=original"
tags:
- ios
- swift
- local authentication
- keychain
- security
---

**TL;DR:** Securing your credentials and providing an easy way to use them may seem like a hard-to-find trade-off. When authenticating using Auth0 you get a bunch of benefits.  The implementation is easy to perform; the user experience is smooth across different screen sizes devices, and platforms; and you have a seamless and secure authentication method in your app. 
As you will see in this article, getting top-class security and the best user experience is not only possible, but it is already implemented in the Auth0 iOS SDK transparently.

## Securing credentials

The Auth0 SDK for iOS provides an easy way to add an authentication flow to your app. As you may have seen in the [iOS SDK Documentation](https://auth0.com/docs/quickstart/native/ios-swift/) the integration is as easy as:

* Installing the dependency, using Cocoapods for example, as described [here](https://auth0.com/docs/quickstart/native/ios-swift/00-login#install-dependencies)
  
``` swift
  use_frameworks!
  pod 'Auth0', '~> 1.18'
```

* Configuring your application in the Dashboard and adding the callback URLs in your iOS app. You can learn more about configuring you app in the dasboard [here](https://auth0.com/docs/quickstart/native/ios-swift/00-login#configure-auth0), and about adding callbacks in the following [section](https://auth0.com/docs/quickstart/native/ios-swift/00-login#add-the-callback).

* Starting the authentication process by calling **start** as described below

``` swift
Auth0
    .webAuth()
    .scope("openid profile offline_access")
    .audience("https://YOUR_DOMAIN/userinfo")
    .start {
        switch $0 {
        case .failure(let error):
            // Handle the error
        case .success(let credentials):
            // Handle success
        }
}
```

It is possible to go a little further and store the credentials securely, where only the device owner can access them to access their application. Only a few lines of code are needed to achieve this.

``` swift

// Create a credentials manager, an object that is part of Auth0 SDK
let credentialsManager = CredentialsManager(authentication: Auth0.authentication())

// Enable Biometrics to store / retrieve credentials.
credentialsManager.enableBiometrics(withTitle: "Touch ID / Face ID Login")

Auth0
    .webAuth()
    .scope("openid profile offline_access")
    .audience("https://YOUR_DOMAIN/userinfo")
    .start {
        switch $0 {
        case .failure(let error):
            // Handle the error
        case .success(let credentials):
            // Handle success
            // Use the credentials manager to store this new credentials
            credentialsManager.store(credentials: credentials)
        }
}
```

Do not forget that using Face ID in an iOS app requires a special declaration in your app's Info.plist, where you explain your users the reason why you are requesting access to Face ID.

``` xml
<key>NSFaceIDUsageDescription</key>
<string>Using Face ID to unlock credentials</string>
```

Retriving those stored credentials any time you need them, only requires accessing **credentials** from the credentials manager obect, as shown in the following snippet

``` swift
credentialsManager.credentials { error, credentials in
    guard error == nil, let credentials = credentials else {
        // No credentials present, or not able to access them. Ask the user to login again
        return
    }
    // User in authenticated!
}
```

## Behind the scenes

This feature relies on two key technologies: Keychain and Local Authentication.

### Keychain

Every platform must provide a user of a secure vault where sensitive information can (and must) be stored. When browsing the web and telling the browser to remember your passwords and credit card information, you are relying on one of many implementations of a secure storage. On the iOS and macOS ecosystem, this storage is called Keychain. 

The Keychain in iOS (iOS and macOS have the same behaviour nowadays) is protected when the device is locked, and unprotected when the device is unlocked. And apps cannot access entries created by other apps. They have their own sandbox where to play. 

Moreover, the Keychain is strongly tied to the hardware on Apple A7 processor and newer (and MacBook Pro with TouchId), it's implementation is what we called hardware-backed keychain. There is a specific hardware, the Secure Enclave, dedicated to generate, store, retrieve and perform operations with entries in the Keychain. 

![Keychain and Local Authentication working togheter](https://docs-assets.developer.apple.com/published/4f475da36d/c97c30fd-5aee-4cd5-b30e-c39aec1a9d55.png)

### Local Authentication

Local Authentication is a framework introduced on iOS 8 and macOS 10.10 that allow developers to ask app users to introduce their passcode or biometry data (Face ID / Touch ID) to unlock some features or gain access to the app. It is a fully transparent framework, where neither the application nor the developer can access your passcode of biometry information. Only the app knows if the challenge was overcome or not.

![Local Authentication](https://docs-assets.developer.apple.com/published/08a1846d5e/b32218fc-f538-412c-80d7-183c920d9429.png)

### How things work

How all this software and hardware technology improves the security of your credentials when using Auth0? The CredentialsManager let’s you store your credentials in the Keychain, the safest place in the platform, and can only allows you to retrieve them by unlocking it using Touch ID or Face ID (if present in the device), or passcode when no biometry is present.

Under the hoods, Auth0's CredentialsManager uses an Open Source wrapper around Keychain (crafted by Auth0) called [SimpleKeychain](https://github.com/auth0/SimpleKeychain). The API of SimpleKeychain is quite simple, and it can be used in your app alongside CredentialsManager without interference.


## References
* [Local Authentication](https://developer.apple.com/documentation/localauthentication)
* [Keychain Services](https://developer.apple.com/documentation/security/keychain_services)
* [Accessing Keychain Items with Face ID or Touch ID](https://developer.apple.com/documentation/localauthentication/accessing_keychain_items_with_face_id_or_touch_id)
