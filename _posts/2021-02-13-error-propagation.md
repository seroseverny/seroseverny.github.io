---
layout: post
title: Comments about rewriting Status/StatusOr errors
---

The most well-known error propagation mechanism in C++ is "result"-or-"error".
One very common example of this pattern is abseil's [`Status/StatusOr`](https://abseil.io/docs/cpp/guides/status).
Tensorflow has an almost identical [`Status`](https://github.com/tensorflow/tensorflow/blob/master/tensorflow/core/platform/status.h) object too, together with convenient macros like [`TF_RETURN_IF_ERROR`](https://github.com/tensorflow/tensorflow/blob/21f5c12e3d9c5b0c2f4c45c70a3da08b4edf212d/tensorflow/core/platform/errors.h#L72).

Even though these libraries make error propagation very straightforward, it's
still really common to see code that unnecessarily "rewrites" error messages --
typically clients rewrite errors coming from library functions.

Consider the following example:

```cpp
// Some library function.
absl::Status SetUpCameraHardware(/*args*/) {
  if (/* something failed */) {
    return absl::NotFoundError("Did not find ABC; was XYZ done?");
  }
  if (/* something else failed */) {
    return absl::UnavailableError(
          "Could not connect because ABC reasons; consider doing XYZ");
  }
  /* ... */
  return absl::OkStatus();
}
```

```cpp
// Client of the library.
absl::Status DoClientStuff() {
  if (!SetUpCameraHardware().ok()) {
    return absl::InternalError("setup failed!");
  }
  if (!DoMoreCameraStuff().ok()) {
    return absl::InternalError("do more camera stuff failed!");
  }
  /* do even more stuff */
  return absl::OkStatus();
}
```

Now once `SetUpCameraHardware()` inevitably fails,
instead of getting details about the failure (which the library function is well positioned to provide), you'll get a pretty generic error
message; details of the original error message are thrown out.

There are indeed situations where error details should _not_ be forwarded (e.g. for security reasons), but my guess is that they are very rare in robotics.
Nevertheless, this pattern of rewriting error messages is surprisingly widespread and very disappointing. If your mechanism for debugging involves looking at logs (glogs, say), if error messages are rewritten in the fashion above, you'll likely have a harder time getting to the root of the issue (that is, if these errors ever reach a `LOG(INFO)` at all).

Instead of re-creating a `Status` object, it's better just to return the original
one; to follow this pattern:

```cpp
if (!status.ok()) { return status; } 
```

This is also an essence of `TF_RETURN_IF_ERROR` macro, which increduously hasn't made it into absl, but is [widely used](https://github.com/search?q=%22RETURN_IF_ERROR%22+absl%3A%3AStatus&type=code) nonetheless.
To summarize, the following is nicer:

```cpp
absl::Status DoClientStuff() {
  auto status = SetUpCameraHardware();
  if (!status.ok()) { return status; }
  status = DoMoreCameraStuff();
  if (!status.ok()) { return status; }
  /* do even more stuff */
  return absl::OkStatus();
}


```

and is ever nicer if you have access to a `RETURN_IF_ERROR`-like macro:

```cpp
absl::Status DoClientStuff() {
  RETURN_IF_ERROR(SetUpCameraHardware());
  RETURN_IF_ERROR(DoMoreCameraStuff());
  /* do even more stuff */
  return absl::OkStatus();
}
```

That said, the stack trace is gone, so we don't know that the error occurred
while in `DoClientStuff()`'s context. That could be problematic -- consider
if the function returning a failure has a huge number of callsites, and if it's
failing due to an error in arguments, and if the reason for failure cannot be
determined from reading the message... Ughh...

One possible verbose solution is to append extra information to a status:

```cpp
absl::Status DoClientStuff() {
  auto status = SetUpCameraHardware();
  if (!status.ok()) {
    return absl::Status(status.code(), 
        "Call to SetUpCameraHardware() failed; reason: " + 
        status.message());
  }
  /* ... */
  return absl::OkStatus();
}
```

there are some ideas around this; for example, see deepmind's [`StatusBuilder`](https://github.com/deepmind/launchpad/blob/master/courier/platform/default/status_builder.h) class.

Other approaches may try to reconstruct the entire stacktrace and dump it into
the status.

In summary, there are rarely reasons to rewrite error messages;
and instead, there are plenty of approaches that propagate error messages downstream -- more often than not making the code prettier and logs more informative.
