# FaceMesh Extraction

## Introduction
FaceMesh Extraction is the use of face detection models (including Augmented Reality (AR) APIs) to determine which portions of an image may contain faces, and to provide information about those faces to the application. There is a spectrum of information that can be provided, ranging from simple bounding boxes to a full face mesh. This document/effort will be focused on what is necessary to allow rendering to a full face mesh.

## Use Cases
Some example use cases include, but are not limited to:
* Replacing the user’s face with another face
* Adding decorating elements on/attached to the user’s face
* Virtual “try-on” of products (e.g. Makeup, Hats, Glasses, Jewelry)
* Hints for makeup tutorials (e.g. highlighting the area to apply the makeup to)

Some of these use cases would likely be integrated by pages which already have existing camera-based experiences, while some are new scenarios that will need to be created from the ground up. When analyzing different requirements between these types of scenarios, the former will be referred to as “VC Scenarios”, since voice conferencing services are a key example of such items, while the latter will be referred to as “e-commerce scenarios”.

## Non-Goals
This document is focused on exposing full face meshes. It is not attempting to build on top of, or replace, the face detection portions of the [Shape Detection API](https://wicg.github.io/shape-detection-api/#face-detection-api), which is focused on providing bounding boxes of all faces in a provided image.

## Proposed API Shape
> TODO: This API builds upon a `ProcessingMediaStreamTrack` interface being worked on by @alvestrand, which is not yet publicly available. Update this section and link to that work when it is available.

The `ProcessingMediaStreamTrack` wraps an existing `MediaStreamTrack` and essentially generates streams of “raw” or unencoded video frame data, similar to the [InsertableStreams API](https://github.com/w3c/webrtc-insertable-streams/blob/master/explainer.md). This API is an extension that allows configuring any additional requested metadata to be returned alongside the raw frame. 

### Usage
The `ProcessingMediaStreamTrack` is constructed by wrapping an existing track. Other than exposing the streams to manipulate the data, this track functions the same as an ordinary track of the type it wraps. This constructor will be extended to take a dictionary which allows for requesting metadata, or providing data for other transforms to be conducted on the data before it is handed to the `ProcessingMediaStreamTrack` (though the latter is out of scope for this work).

```js
// Create/get a standard Video Track
let stream = await navigator.getUserMedia({
  video: {
    // In order to receive faceMesh data in the extracted streams, it needs to be
    // requested to ensure that the provided track can support that mode.
    faceMeshExtraction: true,
  }
});

let [track] = stream.getTracks();

// Create the ProcessingMediaStreamTrack with an additional passed in configuration
let processingTrack = new ProcessingMediaStreamTrack(track, { 
  faceMeshExtraction: true,
});
```

Data from the streams can then be manipulated by:

```js
let rawTransform = new TransformStream({
    start() {
    // Called on startup.
    },

    async transform(rawFrame, controller) {
    // Manipulate the raw frame, including using any present metadata.

    // Send it (or a new ProcessingVideoFrame) to the output stream.
    controller.enqueue(rawFrame);
    },

    flush() {
    // Called when the stream is about to be closed.
    }
});

processingTrack.framesSource
  .pipeThrough(rawTransform)
  .pipeTo(processingTrack.framesSink);
```

Any transforms specified in `ProcessingMediaStreamInit` will be applied to the `ProcessingVideoFrame`s received in the `framesSource` stream.

### API
The following IDL is a rough sketch of the `ProcessingMediaStreamTrack` IDL, that will eventually be fully proposed elsewhere. It is included here for context while that work is not available yet.

> TODO: Remove this section and link to that explainer when it is published.

```webidl
// We inherit from VideoFrame such that other modules can modify this interface
// rather than pushing dependencies/additional metadata onto the more-widely 
// used VideoFrame type.
interface ProcessingVideoFrame : VideoFrame {}

interface ProcessingMediaStreamTrack : MediaStreamTrack {
  constructor ProcessingMediaStreamTrack(MediaStreamTrack baseTrack);
  [SameObject] readonly attribute ReadableStream<ProcessingVideoFrame> framesSource;
  [SameObject] readonly attribute WriteableStream<ProcessingVideoFrame> framesSink;
}
```

The following IDL is proposed to allow for extending `ProcessingMediaStreamTrack`:

```webidl
// This dictionary is a stub that should be extended by any additional "metadata"
// modules.
dictionary ProcessingMediaStreamInit {}

// Note that the inheritance from MediaStreamTrack is existing.
// Some properties from |init| may require (a) corresponding MediaTrackSetting(s)
// to be enabled on |baseTrack|, and the constructor will throw if they are not.
partial interface ProcessingMediaStreamTrack : MediaStreamTrack {
  constructor ProcessingMediaStreamTrack(MediaStreamTrack baseTrack,
                                         optional ProcessingMediaStreamInit init);
}

// This dictionary is a stub that should contain any requested and supported 
// metadata from the ProcessingMediaStreamTrack.
dictionary VideoFrameMetadata {}

// Extend the UnencodedFrame type to allow requesting metadata
partial interface ProcessingVideoFrame : VideoFrame {
        VideoFrameMetadata getRequestedMetadata();
}
```

FaceMesh extraction would then build on top of the above extension to the `ProcessingMediaStreamTrack`.

Note that the face mesh’s UVs/Indices should follow a mapping to be specced out, similar to ARCore's Augmented Face's [canonical face mesh](https://developers.google.com/ar/develop/developer-guides/creating-assets-for-augmented-faces).

```webidl
// One new Constraint is added for MediaStreamTracks. This Constraint is valid
// only on Video tracks, and is used to indicate that the underlying device must
// support FaceMeshExtraction.
partial dictionary MediaTrackSupportedConstraints {
  boolean faceMeshExtraction = true;
}

partial dictionary MediaTrackCapabilities {
  // Similar to echoCancellation, the sequence is used to indicate whether or not
  // the script can control if FaceMesh Extraction is running, (and if the script
  // cannot control that behavior, it will report whether or not it is running).
  sequence<boolean> faceMeshExtraction;
}

partial dictionary MediaTrackConstraintSet {
  ConstrainBoolean faceMeshExtraction;
}

partial dictionary MediaTrackSettings {
  boolean faceMeshExtraction;
}

interface FaceMesh {
  // This ID is unique for the *session* and not for the face. E.g. Person A will
  // always have the same ID throughout the current stream/session, but this id
  // MUST NOT be a global identifier for Person A
  [SameObject] readonly attribute int id;
  [SameObject] readonly attribute Float32Array meshVertices;
  [SameObject] readonly attribute Float32Array meshNormals;
  [SameObject] readonly attribute DOMPointReadOnly position;
  [SameObject] readonly attribute DOMPointReadOnly orientation;

  // Note that because the mapping of the UVs and indices are static, the page
  // can reliably detect landmarks by using well-defined groups of indices.
  // Because of this, "landmarks" are not explicitly exposed by the API.

  // These fields will be specced out, however due to the size and the fact that
  // this data will be needed for rendering, they will be statically supplied by
  // the UA, so that developers don't have to hard-code in a massive blob of data,
  // which could be error-prone.
  static readonly attribute Uint16Array indices;
  static readonly attribute Float32Array uvs;
}

// Note that if faceMesh is requested but the track that this init is used with
// does not support the faceMeshExtraction constraint, then the constructor
// will fail.
partial dictionary ProcessingMediaStreamInit {
        boolean faceMeshExtraction = false;
}

partial dictionary VideoFrameMetadata {
        sequence<FaceMesh> meshes;
}
```

### Background/Rationale
It was decided to directly integrate with the Media Capture APIs given that simply getting the face mesh from a camera frame is something that can be polyfilled via WASM today, and to ease the barrier to adoption. This provides the lowest barrier to entry for VC type scenarios (who are already using Media Capture APIs), and provides potential performance improvements over some of the other possible approaches for e-commerce type scenarios (who would likely need to start from scratch regardless of the selected approach).

This decision does not preclude future work to integrate these detected faces into a WebXR session/render loop (particularly as face tracking becomes more robust and possible on non-front facing cameras). Nor does it preclude the possibility of enabling UA-Side compositing of textures onto the face mesh as an additive feature (which has the potential to be more performant). However, at the moment there do not appear to be clear benefits to these approaches. As other use cases become apparent, or as the API evolves, this decision may be revisited.

#### Integration with getUserMedia
One major drawback of this approach is that it may preclude incorporating WebXR concepts for positioning or to integrate with other WebXR features or a virtual environment. However, currently on at least one major native implementation it would not be possible to integrate with these features anyway. Since most face positioning will be relative to the passed image, and not really “full” spatial tracking, it seems unlikely that some of the sophisticated tracking scenarios described in the WebXR specs would be needed. One potential downside is that any pages wishing to use this feature would need to request full access to the camera (rather than some of the more limited AR permissions which would only expose the face mesh). However, VC type scenarios already need this full camera access for their basic operations. Pages consuming the feature receive some benefits in not needing to worry about synchronization issues or “mode switching” to start/stop detecting faces.

#### Exposing Face Mesh to the page
If the UA simply provides the page with the face mesh, the page is empowered to take more control over rendering decisions. This could include incorporating data from different APIs, rendering (or duplicating) the image in some place/way that is *not* directly over the users face. There are also more options for how this data is exposed to the page, which may provide more flexibility than other potential approaches. 

However, consuming the face mesh in this situation likely becomes more expensive for a given page to adapt, as they would need to do some additional rendering themselves (or incorporate a library to do this rendering). It may require re-working parts of their image pipeline. However, such an approach would likely be more conducive to being implemented by a polyfill. If pages have access to the face mesh directly, they may use this data for new and innovative scenarios not considered here.

## Appendix A: Alternative API Shapes considered
The following alternatives were also considered and rejected for the reasons documented below. While these were currently rejected, nothing precludes future work from adopting these as alternative means of accessing FaceMesh data.

### How Pages Request/Access Detected Faces

#### Integration with WebXR
There are incubation efforts within the WebXR device API, which are currently working towards exposing the raw camera image at the same time as any detected/tracked metadata (such as faces). This would provide a synced up camera frame with the relevant detected face mesh. Integrating with WebXR would allow easier re-use of various WebXR concepts such as XRSpace to track where various faces are. However, most current implementations only work on the front-facing camera and in the “viewer” reference space, where such concepts provide more flexibility than is actually needed; although the ability to easily “parent” something to the detected face would be an added bonus from a developer’s perspective. Additionally, the restriction to use front facing cameras also limits the amount of other AR features that can be used (often most other AR features are not compatible with the front facing camera). If integration with the WebXR API is pursued, a new mode to enable front facing sessions would be added (as this provides a clean, expected way to indicate features that cannot be supported in this mode).

The libraries and capabilities that enable facial-mesh extraction are often tied very closely to any other AR platform support, and thus integrating with WebXR may seem like the most logical place for developers interested in the features to look. The pattern of exposing the face mesh and forcing the page to perform it’s own compositing also fits the existing uses of the WebXR API. Even if the initial implementation was integrated with Media Capture, it may still be possible to expose data in XRFrames/XRSessions, as there is already some integration precedent with other specs (e.g. Gamepad).

Integrating solely with WebXr would come with some fairly significant downsides for VC type scenarios, who already have camera access and would then likely need to perform some form of “mode-switch” in javascript to switch from the standard mode to the webxr mode, given the various restrictions of webxr rendering (often full-screen and taking over/pausing the window’s animation frame loop). E-Commerce scenarios likely would be able to follow either approach; however, a webxr integration may provide easier integration with a framework such as ModelViewer, and/or would allow for potentially more limited access requests from the page, if the site itself didn’t need the raw camera frames and the webxr api could handle that final compositing.

#### As a standalone pipeline
This would mostly be following the [current face detection logic](https://wicg.github.io/shape-detection-api/#face-detection-api) specified in the shape detection API, though expanding it to also expose the facial mesh. This approach could be modified to perform the compositing by passing either a target for the detector object to render to, or simply having the detector provide a bitmap in return. Due to the additional logic that would need to run, there may be UA constraints that would cause this to be the slowest option available. This would almost certainly require/introduce more texture copies than either existing algorithm. However, this exposes the ability for sites to perform facial recognition on images that don’t have to be live (and could be supplied to them as a picture or video the user has already taken). This could also be an object which is generated and allows querying for the latest detected set of face meshes if it is associated with a data stream. Likely a timestamp would need to be exposed, which could be correlated with the data from either a webxr session or from user media. This introduces the highest potential for desynchronization between audio, video, and the detected faces.

### Returned Data Structure

#### UA-Compositing

In the UA Compositing approach, the site would supply the UA with a texture which would be mapped onto a mesh with well-defined UVs. 

If the UA provides the compositing, then the configuration for the page becomes much simpler. The page simply decides which (if any) faces should have textures overlaid and passes this information to the UA. The UA would ensure that this texture can stay in sync with the audio and video, and has a much greater ability to make optimizations to the compositing pipeline. This approach would also likely result in fewer texture copies (depending on the UA implementation). Depending on the algorithms run by UAs, they have more opportunity for differentiation by pulling in various other signals from the image (e.g. lighting/shadowing/other algorithms they may wish to apply), to create a more realistic render with data that may either not be feasible to expose to or be generated by the page (due to the way it is calculated in Machine-Learned models).

However, pages would lose full control over which effects were applied, as only a limited number of “knobs'' could realistically be exposed (and indeed some “knobs” may be UA specific). The page also wouldn’t be able to have any part of the texture extend beyond the face, unless additional support for that (via Regions or something similar) was added. Additionally, the page may have additional information which can’t easily be fed into such a system (e.g. any lighting modifications that the page may wish to make for its own virtual background). It also wouldn’t easily allow for duplicating and/or drawing the face anywhere other than where the face was detected (e.g. to replicate a user's face onto an avatar at a different position in the scene). The polyfill for such an approach would likely be both more complicated and less performant.

#### Hybrid Approach
It may be possible to perform a hybrid approach; whereupon a page may indicate their preference for having the UA conduct the rendering or conducting their own rendering. In such a case the page should always receive the face mesh, but also has the opportunity to provide the UA with a texture to pre-composite on the image. The page would of course be free to perform any additional compositing/overlays after the initial composite performed by the UA.
