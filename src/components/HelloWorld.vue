<template>
  <div class="hello">
    <h3>vue ncnn webassembly nanodet demo</h3>
    <button @click="open" style="height: 30px">Start</button>
    <button @click="close" style="height: 30px">End</button>
    <div>
      <canvas id="canvas" width="640"></canvas>
    </div>
    <video id="video" playsinline autoplay></video>
  </div>
</template>

<script>
import { simd, threads } from "./../../public/wasmFeatureDetect.js";

export default {
  name: "HandModel",
  data: function () {
    return {
      ctx: null,
      video: null,
      isopen: true,
    };
  },
  mounted() {
    this.checkWasmSupportAndLoad();
  },
  methods: {
    async checkWasmSupportAndLoad() {
      var w = 420;
      var h = 320;
      let img = new Image();
      img.src = require("./../assets/logo.png");
      img.onload = () => {
        var canvas = document.getElementById("canvas");
        canvas.setAttribute("width", w);
        canvas.setAttribute("height", h);
        var ctx = canvas.getContext("2d");
        ctx.drawImage(img, 0, 0, w, h);
      };
      var has_simd;
      var has_threads;
      simd().then((simdSupported) => {
        has_simd = simdSupported;
        threads().then((threadsSupported) => {
          has_threads = threadsSupported;
          if (has_simd) {
            if (has_threads) {
              var nanodet_module_name = "nanodet-simd-threads";
            } else {
              nanodet_module_name = "nanodet-simd";
            }
          } else {
            if (has_threads) {
              nanodet_module_name = "nanodet-threads";
            } else {
              nanodet_module_name = "nanodet-basic";
            }
          }
          console.log("load " + nanodet_module_name);
          var nanodetwasm = nanodet_module_name + ".wasm";
          var nanodetjs = nanodet_module_name + ".js";
          fetch(nanodetwasm)
            .then((response) => response.arrayBuffer())
            .then((buffer) => {
              Module.wasmBinary = buffer;
              var script = document.createElement("script");
              script.src = nanodetjs;
              script.onload = () => {
                console.log("Emscripten  loaded.");
              };
              document.body.appendChild(script);
            });
          console.log(Module);
        });
      });
    },
    async open() {
      var wasmModuleLoadedCallbacks = [];
      Module.onRuntimeInitialized = function () {
        wasmModuleLoaded = true;
        for (var i = 0; i < wasmModuleLoadedCallbacks.length; i++) {
          wasmModuleLoadedCallbacks[i]();
        }
      };
      // console.log(Module);
      var w = 640;
      var h = 480;
      var wasmModuleLoaded = true;
      var dst = null;
      var isStreaming = false;
      var video = document.getElementById("video");
      var canvas = document.getElementById("canvas");
      var ctx = canvas.getContext("2d");

      // Wait until the video stream canvas play
      video.addEventListener(
        "canplay",
        function (e) {
          if (!isStreaming) {
            // videoWidth isn't always set correctly in all browsers
            if (video.videoWidth > 0)
              h = video.videoHeight / (video.videoWidth / w);
            canvas.setAttribute("width", w);
            canvas.setAttribute("height", h);
            isStreaming = true;
          }
        },
        false
      );
      // Wait for the video to start to play
      video.addEventListener("play", function () {
        //Setup image memory
        var id = ctx.getImageData(0, 0, canvas.width, canvas.height);
        var d = id.data;
        if (wasmModuleLoaded) {
          mallocAndCallSFilter();
        } else {
          wasmModuleLoadedCallbacks.push(mallocAndCallSFilter);
        }
        function mallocAndCallSFilter() {
          if (dst != null) {
            _free(dst);
            dst = null;
          }
          dst = _malloc(d.length);
          //console.log("What " + d.length);
          sFilter();
        }
      });
      this.capture();

      function ncnn_nanodet() {
        var canvas = document.getElementById("canvas");
        var ctx = canvas.getContext("2d");
        var imageData = ctx.getImageData(0, 0, canvas.width, canvas.height);
        var data = imageData.data;
        HEAPU8.set(data, dst);
        _nanodet_ncnn(dst, canvas.width, canvas.height);
        var result = HEAPU8.subarray(dst, dst + data.length);
        imageData.data.set(result);
        ctx.putImageData(imageData, 0, 0);
      }
      //Request Animation Frame function
      var sFilter = function () {
        console.log(video.ended, video.paused);
        if (video.paused || video.ended) return;
        ctx.fillRect(0, 0, w, h);
        ctx.drawImage(video, 0, 0, w, h);
        ncnn_nanodet();
        window.requestAnimationFrame(sFilter);
      };
    },
    capture() {
      this.video = document.querySelector("video");
      var constraints = {
        audio: false,
        video: {
          width: 640,
          height: 480,
          facingMode: "user",
        },
      };
      navigator.mediaDevices
        .getUserMedia(constraints)
        .then(function (mediaStream) {
          var stream = mediaStream;
          video.srcObject = mediaStream;
          video.onloadedmetadata = function (e) {
            video.play();
          };
        })
        .catch(function (err) {
          console.log(err.message);
        });
    },
    close() {
      this.video.srcObject.getTracks().forEach((track) => {
        track.stop();
      });
      this.video.srcObject = null;
      this.checkWasmSupportAndLoad();
    },
  },
};
</script>
