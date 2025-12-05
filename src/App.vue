<template>
  <div id="app" :class="isMobile && 'mobile'">
    <div class="camera">
      <img src="@/assets/logo.png"
        alt="logo"
        class="logo"
        v-show="!isCameraOpened"
      />
      <idlive-document-capture
        @error="error"
        @close="close"
        @detection="detection"
        @open="open"
        @initialize="initialize"
        @beforeCapture="beforeCapture"
        @capture="capture"
        @beforeOpen="beforeOpen"
        :auto_capture_disabled="false"
        debug
        class="id-camera"
        ref="camera"
        v-show="isCameraOpened"
      ></idlive-document-capture>
    </div>
    <div id="error">{{ errorText }}</div>
    <div class="control">
      <button @click="takePhoto()" v-if="isCameraReady" :disabled="isCapturing">Capture</button>
      <button @click="openCamera()" v-if="!isCameraReady" :disabled="!isCameraLoaded || isCameraOpening">Open camera</button>
      <button @click="closeCamera()" v-if="isCameraReady">Close camera</button>
    </div>
    <div v-if="isMobile && isCameraReady">
      <button class="close-camera_mobile" @click="closeCamera()">close</button>
      <button class="capture-camera_mobile" @click="takePhoto()" :disabled="isCapturing"></button>
    </div>
    <div class="detection" v-if="detectionErrors.length > 0">
      <b>Detection feedback:</b>
      <ul class="detection-errors">
        <li v-for="(error, index) in detectionErrors" :key="index">{{ error }}</li>
      </ul>
    </div>
    <div class="detection-success" v-else-if="isCameraReady && !isCapturing">
      <b>✓ Document detected - Ready to capture</b>
    </div>
    <div class="result">
      <div class="result__label">
        Results:
        <span id="result-loading" v-if="isResultsLoading"></span>
      </div>
      <ul>
        <li v-for="item in results" :key="item + Math.random()"><pre v-html="item"></pre></li>
      </ul>
    </div>
    <div class="module-version">
      Module version: {{ moduleVersion }}
    </div>
    <div class="idld-server-info">
      IDLive Doc Server{{ info }}
    </div>
    <div class="fps">
      FPS: {{ fps }}
    </div>
  </div>
</template>

<script lang="ts">
import { defineComponent } from 'vue'
import pkg from 'idlive-document-capture-web/package.json';
import type { CaptureEvent, ErrorEvent, DetectionEvent } from 'idlive-document-capture-web';
import { EventFPSCounter } from '@/FpsCounter';

export default defineComponent({
  name: 'app',
  data: function () {
    return {
      fpsCounter: new EventFPSCounter(),
      fps: 0,
      results: [] as any[],
      errorText: '',
      detectionText: '',
      detectionErrors: [] as string[],
      isResultsLoading: false,
      isCameraLoaded: false,
      isCameraOpened: false,
      isCameraOpening: false,
      isCameraReady: false,
      isCapturing: false,
      info: '', // Add this line
      pipelines: '', // Add this line
      moduleVersion: pkg.version,
      isMobile: /Android|webOS|iPhone|iPad|iPod|BlackBerry/i.test(navigator.userAgent),
    }
  },
  mounted() {
    // https://github.com/vuejs/core/issues/8334
    this.camera.addEventListener('capture', this.capture);
    this.camera.addEventListener('beforeCapture', this.beforeCapture);
  },
  computed: {
    camera() {
      return this.$refs.camera as HTMLIdLiveDocumentCaptureElement;
    },
  },
  methods: {
    getApiVersion() {
        return fetch('/idld_api/api_version')
        .then(response => {
            if (!response.ok) {
              throw new Error(`HTTP error! status: ${response.status}`);
            }
            return response.json();
        })
        .then(data => {
            console.log('API Version:', data.version);
            // Filter and store the pipelines
            this.pipelines = data.availablePipelines.filter((pipeline: string) => !pipeline.startsWith('default-')).join(',');
            this.info = `(${data.version}): ${this.pipelines}`
            return data.version;
        })
        .catch(error => {
            console.error('Failed to fetch API version:', error);
            throw error;
        });
    },
    capture(event: CaptureEvent) {
      console.log('capture', event.detail[0]);

      const { photo, encryptedFile } = event.detail[0];
      if (!photo) {
        return;
      }
      const photoData = new FormData();
      photoData.append('file', photo);

      // Append pipelines parameter to the URL
      const idldUrlWithParams = `/idld_api/check_liveness_file?pipelines=${encodeURIComponent(this.pipelines)}`;
      const iadUrlWithParams = `/iad_api/check_capture_liveness`;

      this.isCapturing = false;
      this.isResultsLoading = true;

      const idldRequest = fetch(idldUrlWithParams, {
        method: "post",
        body: photoData
      }).then(response => response.json());

      const iadRequest = fetch(iadUrlWithParams, {
        method: "post",
        headers: { 'Content-Type': 'application/octet-stream' },
        body: encryptedFile
      }).then(response => response.json());

      Promise.all([idldRequest, iadRequest])
        .then(([idldResult, iadResult]) => {
          this.results.push(`IAD Result:\n ${JSON.stringify(iadResult, null, 2)}`); // Push the formatted IAD result to the results array
          this.results.push(`IDLD Result:\n ${JSON.stringify(idldResult, null, 2)}`); // Push the formatted IDLD result to the results array
        })
        .catch(e => {
          console.error(e);
          this.results.push(e.toString());
        })
        .finally(() => this.isResultsLoading = false);
    },
    async initialize() {
      console.log('initialize');

      // Try to load the new license file first, fallback to the old one
      let detectorLicense: string;
      
      try {
        const response = await fetch('/idlive-doc-detector-v2.4.4.eval-license.txt');
        if (response.ok) {
          const licenseText = await response.text();
          // Try the license as-is (base64 format)
          detectorLicense = licenseText.trim();
          console.log('Using new license file (base64 format)');
        } else {
          throw new Error('License file not found');
        }
      } catch (error) {
        console.warn('Could not load new license file, using fallback license:', error);
        // Fallback to the original PGP signature format
        detectorLicense = `-----BEGIN PGP SIGNATURE-----

iHUEABYIAB0WIQREm3xOf5iQkiWFr/oM5DV3qXaw5gUCaO4zfwAKCRAM5DV3qXaw
5jvhAQCVbhu5lchr+ITXghkcMXxCOxsJUfSTrsHpOBj86JM+OwEA+kZjiqHnBVYy
7b7VrwO44HRHXq5wgziFtBTXBBlzRAM=
=kHYS
-----END PGP SIGNATURE-----`;
      }
      
      this.camera.setLicense(detectorLicense, 'documentDetector');

      this.isCameraLoaded = true;
      this.fpsCounter.addEventListener('fps', this.fpsUpdated);
      this.getApiVersion()
        .then(version => {
          console.log('Fetched API Version:', version);
        })
        .catch(error => {
          console.error('Error fetching API version:', error);
        });
    },
    fpsUpdated(event: Event) {
      const customEvent = event as CustomEvent<number>;
      console.log(`fps: ${customEvent.detail}`);

      this.fps = customEvent.detail;
    },
    detection(event: DetectionEvent) {
      const detectionResult = event.detail[0];
      const errors = detectionResult.errors || [];
      
      // Convert error codes to user-friendly messages
      this.detectionErrors = errors.map((error: string) => {
        return this.getDetectionErrorMessage(error);
      });
      
      // Keep the first error for backward compatibility
      this.detectionText = errors.length > 0 ? errors[0] : '';
      
      this.fpsCounter.onEvent();
    },
    getDetectionErrorMessage(error: string): string {
      // Use string literals matching the DetectionError enum values
      const errorMessages: Record<string, string> = {
        'DOCUMENT_NOT_FOUND': '📄 Document not found - Please position the document in the frame',
        'DOCUMENT_SIZE_LOWER_THAN_10_PERCENT': '📏 Document too small - Move closer to the document',
        'DOCUMENT_BORDERS_OUTSIDE_OF_FRAME': '🔲 Document edges outside frame - Ensure the entire document is visible',
        'MULTIPLE_DOCUMENTS_IN_FRAME': '📚 Multiple documents detected - Please show only one document at a time',
        'DOCUMENT_TOO_CLOSE_TO_BORDER': '⚠️ Document too close to edge - Center the document in the frame',
        'LICENSE_NOT_INSTALLED': '🔑 License error - Please check license configuration',
      };
      
      return errorMessages[error] || `⚠️ ${error}`;
    },
    error(event: ErrorEvent) {
      const errorMessage = event.detail[0].message;
      console.error('error', event.detail[0]);

      // Suppress "More than one feature" error - this is expected for multi-feature licenses
      // and doesn't prevent functionality since we're setting the license with documentDetector type
      if (errorMessage && errorMessage.includes('More than one feature')) {
        console.warn('License has multiple features - this is expected. License should still work for document detection.');
        // Don't set error text as this is expected behavior and doesn't affect functionality
        return;
      }

      // Only show other errors
      this.errorText = errorMessage;
    },
    open() {
      console.log('open');

      this.isCameraReady = true;
      this.isCameraOpening = false;
    },
    beforeCapture() {
      console.log('beforeCapture');

      this.isCapturing = true;
    },
    beforeOpen() {
      console.log('beforeOpen');

      this.isCameraOpening = true;
      this.isCameraOpened = true;
      this.errorText = '';
    },
    close() {
      console.log('close');

      this.detectionText = '';
      this.detectionErrors = [];
      this.isCameraReady = false;
      this.isCameraOpened = false;
      this.isCameraOpening = false;
      this.isCapturing = false;
      this.fps = 0;
    },
    openCamera() {
      this.camera.openCamera();
    },
    closeCamera() {
      this.camera.closeCamera();
    },
    takePhoto() {
      this.camera.takePhoto();
    },
  }
})
</script>

<style>
body {
    background: whitesmoke;
}

#app {
    font-family: 'Avenir', Helvetica, Arial, sans-serif;
    -webkit-font-smoothing: antialiased;
    -moz-osx-font-smoothing: grayscale;
    text-align: center;
    color: #2c3e50;
    max-width: 800px;
    margin: auto;
}

.id-camera {
    width: 100%;
    height: 100%;
}

.control button {
    margin: 20px;
    padding: 10px;
    border: none;
    background: #4ca585;
    border-radius: 5px;
    color: white;
    font-size: 15px;
}

.control button:disabled,
.capture-camera_mobile:disabled  {
    background: #aaa;
}

#result-loading:after,
button:disabled:after  {
    content: "☼";
    display: inline-block;
    margin-left: 8px;
    animation: lds-dual-ring 2.5s linear infinite;
}

.capture-camera_mobile:disabled:after  {
    margin-left: 0;
    color: #fff;
    font-size: 2rem;
}

@keyframes lds-dual-ring {
    0% {
        transform: rotate(0deg);
    }
    100% {
        transform: rotate(360deg);
    }
}

.logo {
    max-width: 600px;
    width: 100%;
}

.camera {
    height: 300px;
    display: flex;
    align-items: center;
    justify-content: center;
}

.control {
    display: flex;
    justify-content: center;
}

.close-camera_mobile {
    position: fixed;
    right: 6vw;
    top: 2vh;
    text-shadow: 0 0 4px black;
    color: white;
    font-size: 20px;
    user-select: none;
    background: none;
    border: none;
    z-index: 104;
}

.capture-camera_mobile {
    width: 50px;
    height: 50px;
    border-radius: 25px;
    border: 2px solid white;
    outline: none;
    background-color: red;
    position: fixed;
    left: calc(50% - 25px);
    bottom: 10px;
    z-index: 104;
}

@media screen and (orientation: landscape) and (min-width: 360px) {
    .capture-camera_mobile {
        bottom: calc(50% - 25px);
        left: auto;
        right: 5vw;
    }
}

.result {
    padding-bottom: 40px;
}

.result li {
    text-align: left;
}

.result__label {
    font-weight: bold;
    margin-top: 40px;
    font-size: 22px;
}

.detection {
    text-align: left;
    color: #a40e26;
    margin: 10px 0;
}

.detection-errors {
    margin: 5px 0;
    padding-left: 20px;
    list-style-type: none;
}

.detection-errors li {
    margin: 5px 0;
    padding: 5px;
    background-color: #ffe6e6;
    border-left: 3px solid #a40e26;
    border-radius: 3px;
}

.detection-success {
    text-align: left;
    color: #2d8659;
    margin: 10px 0;
    padding: 5px;
    background-color: #e6f7ed;
    border-left: 3px solid #2d8659;
    border-radius: 3px;
}

#error {
    color: red;
}

.module-version {
    position: absolute;
    left: 10px;
    top: 10px;
}

.idld-server-info {
    position: absolute;
    left: 10px;
    top: 50px;
}

.fps {
    position: absolute;
    left: 10px;
    top: 30px;
    color: red;
}

.mobile .fps {
    z-index: 1000;
}
.mobile .detection {
    z-index: 1000;
    position: absolute;
    left: 10px;
    top: 50px;
}
</style>
