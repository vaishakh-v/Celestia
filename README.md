## Welcome to GitHub Pages

You can use the [editor on GitHub](https://github.com/vaishakh-v/Celestia/edit/main/README.md) to maintain and preview the content for your website in Markdown files.

Whenever you commit to this repository, GitHub Pages will run [Jekyll](https://jekyllrb.com/) to rebuild the pages in your site, from the content in your Markdown files.

### Markdown

Markdown is a lightweight and easy-to-use syntax for styling your writing. It includes conventions for

```markdown
Syntax highlighted code block

# Header 1
## Header 2
### Header 3

- Bulleted
- List

1. Numbered
2. List

**Bold** and _Italic_ and `Code` text

[Link](url) and ![Image](src)
```

For more details see [GitHub Flavored Markdown](https://guides.github.com/features/mastering-markdown/).

### Jekyll Themes

Your Pages site will use the layout and styles from the Jekyll theme you have selected in your [repository settings](https://github.com/vaishakh-v/Celestia/settings/pages). The name of this theme is saved in the Jekyll `_config.yml` configuration file.

<model-viewer src="Upload model...">

    <div class="progress-bar hide" slot="progress-bar">
        <div class="update-bar"></div>
    </div>
    <button slot="ar-button" id="ar-button">
        View in your space
    </button>
    <div id="ar-prompt">
        <img src="https://modelviewer.dev/shared-assets/icons/hand.png">
    </div>
</model-viewer>


<!doctype html>
<!--
Copyright 2020 The Immersive Web Community Group
Permission is hereby granted, free of charge, to any person obtaining a copy of
this software and associated documentation files (the "Software"), to deal in
the Software without restriction, including without limitation the rights to
use, copy, modify, merge, publish, distribute, sublicense, and/or sell copies of
the Software, and to permit persons to whom the Software is furnished to do so,
subject to the following conditions:
The above copyright notice and this permission notice shall be included in all
copies or substantial portions of the Software.
THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY, FITNESS
FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE AUTHORS OR
COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER LIABILITY, WHETHER
IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM, OUT OF OR IN
CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE SOFTWARE.
-->
<html>
  <head>
    <meta charset='utf-8'>
    <meta name='viewport' content='width=device-width, initial-scale=1, user-scalable=no'>
    <meta name='mobile-web-app-capable' content='yes'>
    <meta name='apple-mobile-web-app-capable' content='yes'>
    <link rel='icon' type='image/png' sizes='32x32' href='favicon-32x32.png'>
    <link rel='icon' type='image/png' sizes='96x96' href='favicon-96x96.png'>
    <link rel='stylesheet' href='css/common.css'>

    <title>Barebones AR</title>
  </head>
  <body>
    <div id="overlay">
      <header>
        <details open>
          <summary>Barebones WebXR DOM Overlay</summary>
          <p>
            This sample demonstrates extremely simple use of an "immersive-ar"
            session with no library dependencies, with an optional DOM overlay.
            It doesn't render anything exciting, just draws a rectangle with a
            slowly changing color to prove it's working.
            <a class="back" href="./index.html">Back</a>
          </p>
          <div id="session-info"></div>
          <div id="warning-zone"></div>
          <button id="xr-button" class="barebones-button" disabled>XR not found</button>
        </details>
      </header>
    </div>
    <main style='text-align: center;'>
      <p>Click 'Enter AR' to see content</p>
    </main>
    <script type="module">
      // XR globals.
      let xrButton = document.getElementById('xr-button');
      let xrSession = null;
      let xrRefSpace = null;

      // WebGL scene globals.
      let gl = null;

      function checkSupportedState() {
        navigator.xr.isSessionSupported('immersive-ar').then((supported) => {
          if (supported) {
            xrButton.innerHTML = 'Enter AR';
          } else {
            xrButton.innerHTML = 'AR not found';
          }

          xrButton.disabled = !supported;
        });
      }

      function initXR() {
        if (!window.isSecureContext) {
          let message = "WebXR unavailable due to insecure context";
          document.getElementById("warning-zone").innerText = message;
        }

        if (navigator.xr) {
          xrButton.addEventListener('click', onButtonClicked);
          navigator.xr.addEventListener('devicechange', checkSupportedState);
          checkSupportedState();
        }
      }

      function onButtonClicked() {
        if (!xrSession) {
            // Ask for an optional DOM Overlay, see https://immersive-web.github.io/dom-overlays/
            navigator.xr.requestSession('immersive-ar', {
                optionalFeatures: ['dom-overlay'],
                domOverlay: {root: document.getElementById('overlay')}
            }).then(onSessionStarted, onRequestSessionError);
        } else {
          xrSession.end();
        }
      }

      function onSessionStarted(session) {
        xrSession = session;
        xrButton.innerHTML = 'Exit AR';

        // Show which type of DOM Overlay got enabled (if any)
        if (session.domOverlayState) {
          document.getElementById('session-info').innerHTML = 'DOM Overlay type: ' + session.domOverlayState.type;
        }

        session.addEventListener('end', onSessionEnded);
        let canvas = document.createElement('canvas');
        gl = canvas.getContext('webgl', {
          xrCompatible: true
        });
        session.updateRenderState({ baseLayer: new XRWebGLLayer(session, gl) });
        session.requestReferenceSpace('local').then((refSpace) => {
          xrRefSpace = refSpace;
          session.requestAnimationFrame(onXRFrame);
        });
      }

      function onRequestSessionError(ex) {
        alert("Failed to start immersive AR session.");
        console.error(ex.message);
      }

      function onEndSession(session) {
        session.end();
      }

      function onSessionEnded(event) {
        xrSession = null;
        xrButton.innerHTML = 'Enter AR';
        document.getElementById('session-info').innerHTML = '';
        gl = null;
      }

      function onXRFrame(t, frame) {
        let session = frame.session;
        session.requestAnimationFrame(onXRFrame);
        let pose = frame.getViewerPose(xrRefSpace);

        if (pose) {
          gl.bindFramebuffer(gl.FRAMEBUFFER, session.renderState.baseLayer.framebuffer);

          // Update the clear color so that we can observe the color in the
          // headset changing over time. Use a scissor rectangle to keep the AR
          // scene visible.
          const width = session.renderState.baseLayer.framebufferWidth;
          const height = session.renderState.baseLayer.framebufferHeight;
          gl.enable(gl.SCISSOR_TEST);
          gl.scissor(width / 4, height / 4, width / 2, height / 2);
          let time = Date.now();
          gl.clearColor(Math.cos(time / 2000), Math.cos(time / 4000), Math.cos(time / 6000), 0.5);
          gl.clear(gl.COLOR_BUFFER_BIT | gl.DEPTH_BUFFER_BIT);
        }
      }

      initXR();
    </script>
  </body>
</html>


