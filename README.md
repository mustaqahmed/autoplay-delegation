This proposal is deprecated, replaced by [WICG/capability-delegation](https://github.com/WICG/capability-delegation)

---

# Dynamic Delegation of Autoplay Capability
A tentative API to experiment with dynamic overrides for "autoplay" feature
policy.

## Problem overview

### The use-case we want to test

We want to support the use-case that a subframe can't use "autoplay" by default
but the top frame can temporarily allow the subframe to autoplay whenever it
(the top frame) wants.  One particular use case we have is that the user clicks
on a "play" button on the top frame, which would notify the subframe to play a
video.  Until the top frame notifies that subframe (possibly through a
`postMessage`), the subframe won't be able to start the playback.  Additionally,
the top frame could later disallow autoplaying in the subframe after first
allowing it.

### Traditional feature policies can't help

A site can control the availability of capabilities through feature policies.
The policy can be specified either by an HTTP header or an `<iframe>` atribute.
In any case, the effective policy is determined during frame-loading, so the
feature control here is "static": for example, chaging the `<iframe>` attribute
later on has no effect on the availability of the capability in the subframe.
See related links below for feature policy details.


## A tentative API for experiments

We are proposing a simple API to test the idea: if one frame specifies a new
feature policy when sending a `postMessage` to a another frame, the new policy
becomes effective on the receiver frame upon receiving the message.  To make
this in-line with existing feature policy hierarchy, the receiver frame has to
be a child frame of the sender; the `allow` parameter would be ignored
otherwise.  (The strict parent-to-child constraint would prevent abusive
propagation of feature policies between, say, sibling frames.)

```javascript
// Script for top frame

let targetWindow = frames[0];

function playInSubframe() {
    targetWindow.postMessage("play_video", {allow: "autoplay"});
    setTimeout(1000, pauseInSubframe);
}

function pauseInSubframe() {
    targetWindow.postMessage("pause_video", {allow: ""});
}

document.getElementById("button").addEventListener("click", playInSubframe);
```

```javascript
// Script for subframe
function messageReceiver(e) {
    if (e.source !== window.parent)
        return;

    let videoElement = document.getElementById("video_elem");

    if (e.data === "play_video")
        videoElement.play();
    else if (e.data === "pause_video")
        videoElement.pause();
}

window.addEventListener("message", messageReceiver);
```

### Why just autoplay?

While the `allow` parameter can possibly contain any feature-policy name (see
Related links below), we are not pursuing this general direction now because
this seems complicated from the perspective of effective feature policies.  In
Security Considerations below, we will oned highlight one security aspect too.


## Security considerations

For autoplay, dynamic delegation seem safe since the host frame can't access any
information from a cross-origin subframe.

Howver this is not true for dynamic delegetion any _general capability_ because
it would give the host frame fine-grained control over the behavior of
cross-origin frames.  For example, it could allow an attacker to selectively
enable/disable the features controlled by feature policies and put a
cross-origin window in an inconsistent state, resulting in unexpected behavior
and potentially leaking interesting state from the subframe.


## Related links

- [Introduction to Feature
  Policy](https://developers.google.com/web/updates/2018/06/feature-policy).

- [List of feature policies in Chromium](https://cs.chromium.org/chromium/src/third_party/blink/renderer/core/feature_policy/feature_policy_features.json5?rcl=d118d0cb1f880768e0612ddf31b5036785cab614&l=28).
