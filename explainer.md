# Finding Existing Sessions in EME

## Motivation

EME currently provides an application with an API to create a new session and
generate a license request based on some initialization data.  If applications
create a new session every time they encounter init data, this can lead to many
duplicate sessions which consume limited resources in hardware DRM
implementations, and can ultimately lead to playback failures.

To avoid this, applications need to know when init data can be ignored and when
a new session needs to be created.

Parsing init data requires applications to know the details of init data formats
for each key system, which may not be available.  Binary comparison of init data
helps, but still produces duplicate sessions, as two distinct init data binaries
may still refer to the same set of keys.  The only component that knows for
certain which init data binaries map to which sessions is the CDM, so the CDM
should provide an API (through EME) to help applications understand this.


## Overview

In this proposal, we add a method to [`MediaKeys`][] which allows an application
to find an existing session (if it exists) that contains usable keys for some
init data.  This method can go beyond the binary comparison that applications
use today, and may find usable sessions even for unique init data.

We also propose a polyfill which would allow applications to adopt the new
method right away.  This polyfill uses binary comparisons of init data, and
therefore produces the same false negatives that we want to eliminate with the
new method.  While this is not perfect, it is no worse than where we started.
The polyfill will help offload the naive init data comparison logic from
applications and smooth the transition to the new method.

Currently, [`MediaKeys`][] has a `createSession()` method which allows the
creation of new sessions.  We should extend `MediaKeys` to add a method called
`findSessionByInitData`, which will find an existing session based on
initialization data.  This initialization data may or may not be
binary-identical to the init data used to create the session, but the CDM can
determine equivalence in terms of usable keys.

[`MediaKeys`]: https://www.w3.org/TR/encrypted-media/#mediakeys-interface


## Web IDL

```js
[SecureContext]
partial interface MediaKeys {
    Promise<MediaKeySession> findSessionByInitData(DOMString initDataType,
                                                   BufferSource initData);
};
```

_findSessionByInitData_

Locates a session based on the initData.  If a `MediaKeySession` exists whose
keys are usable and can decrypt media data associated with this initData, the
returned Promise will be resolved with that session.  If no such session exists,
the Promise will be resolved with null.  If the data is invalid in some way, the
Promise will be rejected with an appropriate error.

|Parameter   |Type        |Nullable|Optional|Description|
|------------|------------|--------|--------|-----------|
|initDataType|DOMString   | ✘      | ✘      |The Initialization Data Type of the initData.|
|initData    |BufferSource| ✘      | ✘      |Initialization Data|


## Application notes

Applications can use this method to determine whether or not a new session needs
to be created.  If a session exists for a given initialization data, the
application knows it does not need to create a new one.

For browsers that lack this new method, applications can create a basic polyfill
that falls back to the same binary comparisons of initData used by applications
today.


## Examples

```js
async function setupMediaKeys(mediaKeys) {
  await video.setMediaKeys(mediaKeys);

  video.addEventListener('encrypted', async (event) => {
    let session = await mediaKeys.findSessionByInitData(
        event.initDataType, event.initData);

    if (!session) {
      session = mediaKeys.createSession();
      session.addEventListener('message', handleMessageEvents);
      session.generateRequest(event.initDataType, event.initData);
    }
  });
}
```


## Backward Compatibility

Applications which don't use the new method will not be affected.  If they have
some existing way to avoid duplication sessions, either through binary
comparisons or through some application-specific out-of-band communication,
they may continue to do use it.

User agents which don't implement the new method can be polyfilled, or
applications can simply check for the existence of the new method before calling
it.  For example:

```js
if (mediaKeys.findSessionByInitData) {
  session = await mediaKeys.findSessionByInitData(
      event.initDataType, event.initData);
}
```


## Alternatives Considered

### Allow generateRequest() not to send a 'message' event

If the CDM already has a usable session for the media data associated with the
initData, `generateRequest()` could simply not send a 'message' event.  This
would avoid the application fetching duplicate licenses.

Whether or not this avoids the wasting of hardware DRM resources depends on the
user agent's implementation of `createSession()`.  At least at the JavaScript
level, the session exists, and the application may still be tracking that
session object without knowing that it is unneeded.

It seems preferable to avoid a solution which leaves the application in the
dark about what the CDM is doing.


### Change generateRequest() to resolve with a boolean

If the CDM already has a usable session for the media data associated with the
initData, `generateRequest()` could signal to the application that it does not
need this session by resolving with `false`.  This signal would allow
applications to close the session and drop references to it.

Like the previous alternative, this may or may not avoid wasting hardware DRM
resources, depending on the user agent's implementation of `createSession()`.
Also like the previous alternative, applications that were unaware of this
change would still track the session.

Applications aware of the change running in user agents that had not implemented
it would need to be very careful in checking the return value.  Older UAs would
resolve with `undefined`, which is falsey.  It would be very easy for
applications to mistake `undefined` for `false` and close a necessary session.

It seems preferable to keep the application in control and avoid a confusing
upgrade path while user agents are still being updated.


### Add a method to ask if a new session is needed

An application would be able to query if a session was needed before creating
one.  The method would return a Promise to a boolean, and would be more explicit
than the previous alternative.

This is very similar to what we chose for this proposal, except that it provides
less information to applications.  If a usable session already exists for a
given initData, an application might like to know more about it.  For example,
an application might find the expiration time useful when deciding whether or
not to create the session.  Returning the actual session gives more control
to applications.


## Sample Polyfill

This has been tested and found to work in Chrome 65 (current stable release at
the time this explainer was originally drafted).

```js
if (window.MediaKeys && window.MediaKeySession &&
    !MediaKeys.prototype.findSessionByInitData) {
  // We need to shim generateRequest() to keep track of sessions we know about.
  const originalGenerateRequest = MediaKeySession.prototype.generateRequest;
  let activeSessionList = [];

  MediaKeySession.prototype.generateRequest = function(initDataType, initData) {
    // Add this session to our list.
    activeSessionList.push({
      session: this,
      initDataType: initDataType,
      // Normalize from BufferSource to Uint8Array.
      initData: new Uint8Array(initData),
    });

    // Remove the session from our list when it closes.
    this.closed.then(() => {
      for (let i = 0; i < activeSessionList.length; ++i) {
        if (activeSessionList[i].session == this) {
          activeSessionList.splice(i, 1);
          return;
        }
      }
    });

    return originalGenerateRequest.call(this, initDataType, initData);
  };

  // A simple implementation which performs binary comparisons on the initData
  // to find a matching session in activeSessionList.  This can produce false
  // negatives, but that will be impossible to avoid in a polyfilled version.
  MediaKeys.prototype.findSessionByInitData =
      async function(initDataType, initData) {
    // Normalize from BufferSource to Uint8Array.
    const initDataU8 = new Uint8Array(initData);

    for (let i = 0; i < activeSessionList.length; ++i) {
      if (activeSessionList[i].initDataType == initDataType &&
          Uint8ArraysEqual(activeSessionList[i].initData, initDataU8)) {
        const session = activeSessionList[i].session;

        // Ignore any expired sessions.
        // NaN means "never expires", and NaN <= now() is always false.
        if (session.expiration <= Date.now()) {
          continue;
        }

        // Ignore any session whose keys are all unusable.
        let usable = false;
        session.keyStatuses.forEach((status, keyId) => {
          if (status == 'usable') {
            usable = true;
          }
        });
        if (!usable) {
          continue;
        }

        // If we reach this point, this is the session we are looking for.
        return session;
      }
    }

    // Return null to indicate that we failed to find a match.
    // A session may still exist which can decrypt the media data associated
    // with initData, but this polyfill can't determine that for certain.  Only
    // the CDM and a native implementation of this method can avoid false
    // negatives.
    return null;
  };
}
```
