# PEKO
**PE**rmissions with **KO**tlin

[![Build Status](https://travis-ci.org/deva666/Peko.svg?branch=master)](https://travis-ci.org/deva666/Peko) [![Android Arsenal](https://img.shields.io/badge/Android%20Arsenal-Peko-blue.svg?style=flat)](https://android-arsenal.com/details/1/6861) [![License](https://img.shields.io/badge/License-Apache%202.0-blue.svg)](https://opensource.org/licenses/Apache-2.0)
---
### Android Permissions with Kotlin Coroutines
No more callbacks, builders, listeners or verbose code for requesting Android permissions.  
Get Permission Request Result asynchronously with one function call.  
Thanks to [Kotlin Coroutines](https://github.com/Kotlin/kotlinx.coroutines), permissions requests are async and lightweight (no new threads are used/created).

***

### Installation

Add `jcenter` repository

```
compile 'com.markodevcic.peko:peko:1.0.1'
```

### Example 
In an Activity or a Fragment that implements `CoroutineScope` interface:
```kotlin
launch {
    val (grantedPermissions) = Peko.requestPermissionsAsync(this, Manifest.permission.BLUETOOTH) 
    
    if (Manifest.permission.BLUETOOTH in grantedPermissions) {
        //we have Bluetooth permission
    } else {
        
    }
}
```

Or use one of the extension functions on Activity or a Fragment:
```kotlin
launch {
    val (grantedPermissions) = requestPermissionsAsync(Manifest.permission.BLUETOOTH) 
    
    if (Manifest.permission.BLUETOOTH in grantedPermissions) {
        //we have Bluetooth permission
    } else {
        
    }
}
```

### Screen rotations
Library has support for screen rotations. 
To avoid memory leaks, all Coroutines that have not completed yet, should be cancelled in the `onDestroy` function.
When you detect a orientation change, cancel the `Job` of a `CoroutineScope` with an instance of `ActivityRotatingException`. Internally, this will retain the current request that is in progress. The request is then resumed with calling `resumeRequest` method.

Example:

First:
```kotlin

//job that will be cancelled in onDestroy
private val job = CompletableDeferred<Any>()

private fun requestPermission(vararg permissions: String) {
    launch { 
        val (grantedPermissions) = Peko.requestPermissionsAsync(this@MainActivity, *permissions)
        //check granted permissions
    }
}
```

Then in `onDestroy` of an Activity:
```kotlin
if (isChangingConfigurations) {
    job.completeExceptionally(ActivityRotatingException()) //screen rotation, retain the results
} else { 
    job.cancel() //no rotation, just cancel the Coroutine
}
``` 

And when this Activity gets recreated in one of the Activity lifecycle functions, e.g.`onCreate`:
```kotlin

//check if we have a request already (or some other way you detect screen orientation)
if (Peko.isRequestInProgress()) {
    launch {
        //get the existing request and await the result
        val (grantedPermissions) = Peko.resumeRequest() 
        //check granted permissions
    }
}
```

### Permission Rationales
If you want to show a permission rationale to the user, you can use the built in `AlertDialogPermissionRationale`. This will show an Alert Dialog with your message and title, explaining to user why this rationale is needed. It will be shown only once and only if user denies the permission for the first time.

```kotlin
val rationale = AlertDialogPermissionRationale(this@MainActivity) {
    this.setTitle("Need permissions")
    this.setMessage("Please give permissions to use this feature")	
}

launch {
    val permissionResult = Peko.requestPermissionsAsync(this@MainActivity, Manifest.permission.BLUETOOTH, rationale = rationale)
}
```

There is also a `SnackBarRationale` class that shows a SnackBar when permission rationale is required.

```kotlin
val snackBar = Snackbar.make(rootView, "Permissions needed to continue", Snackbar.LENGTH_LONG)
val snackBarRationale = SnackBarRationale(snackBar, actionTitle = "Request again")

launch {
    val permissionResult = Peko.requestPermissionsAsync(this@MainActivity, Manifest.permission.BLUETOOTH, rationale = snackBarRationale)
}
```

You can also show your own implementation of Permission Rationale to the user. This can be your Dialog, a fragment, or any other UI component. Just implement the interface `PermissionRationale`. If `true` is returned from suspend function `shouldRequestAfterRationaleShownAsync`, permissions will be asked for again, otherwise the request completes and returns the current permission result.

Here is how a rationale with a Fragment can be implemented

```kotlin
class FragmentRationale : Fragment(), PermissionRationale {

    var startCallback: (() -> Unit)? = null // callback in parent activity to trigger replacing fragments in fragment transaction 
    private lateinit var continuation: CancellableContinuation<Boolean>

    override fun onCreateView(inflater: LayoutInflater, container: ViewGroup?, savedInstanceState: Bundle?): View? {
        return inflater.inflate(R.layout.fragment_rationale, container, false)
    }

    override fun onActivityCreated(savedInstanceState: Bundle?) {
        super.onActivityCreated(savedInstanceState)
        btnRequestAgain.setOnClickListener {
            continuation.resume(true)
            activity?.supportFragmentManager?.popBackStack()
        }
        btnCancel.setOnClickListener {
            continuation.resume(false)
            activity?.supportFragmentManager?.popBackStack()
        }
    }

    override suspend fun shouldRequestAfterRationaleShownAsync(): Boolean {
        return suspendCancellableCoroutine { c ->
            startCallback?.invoke()
            continuation = c
        }
    }
}
```


## License
```text
Copyright 2019 Marko Devcic

Licensed under the Apache License, Version 2.0 (the "License");
you may not use this file except in compliance with the License.
You may obtain a copy of the License at

   http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS,
WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
See the License for the specific language governing permissions and
limitations under the License.
```