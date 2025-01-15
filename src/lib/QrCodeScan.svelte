<script lang="ts">
    import { createEventDispatcher } from "svelte";
    import { onMount, onDestroy } from "svelte";
    import jsQR from "jsqr";
    
    let videoElement: HTMLVideoElement;
    let qrCodeText: string = "";
    let scanning: boolean = false;
    const dispatch = createEventDispatcher();  // Create event dispatcher
  
    onMount(() => {
      async function startCamera() {
        try {
          const stream = await navigator.mediaDevices.getUserMedia({
            video: { facingMode: "environment" },
          });
          videoElement.srcObject = stream;
          videoElement.play();
          scanning = true;
          requestAnimationFrame(scanQRCode);
        } catch (err) {
          console.error("Error accessing camera:", err);
          scanning = false;
        }
      }
  
      function scanQRCode() {
        if (!scanning) return;
        if (videoElement.paused || videoElement.ended) {
          return requestAnimationFrame(scanQRCode);
        }
  
        const videoWidth = videoElement.videoWidth;
        const videoHeight = videoElement.videoHeight;
        if (videoWidth === 0 || videoHeight === 0) {
          return requestAnimationFrame(scanQRCode);
        }
  
        const canvas = document.createElement("canvas");
        const context = canvas.getContext("2d");
  
        if (context) {
          canvas.width = videoWidth;
          canvas.height = videoHeight;
          context.drawImage(videoElement, 0, 0, canvas.width, canvas.height);
  
          const imageData = context.getImageData(0, 0, canvas.width, canvas.height);
          const code = jsQR(imageData.data, canvas.width, canvas.height, {
            inversionAttempts: "dontInvert",
          });
  
          if (code) {
            qrCodeText = code.data;
            dispatch("qrScanned", qrCodeText);  // Dispatch QR code text
          }
        }
        
        requestAnimationFrame(scanQRCode);
      }
  
      startCamera();
    });
  
    onDestroy(() => {
      scanning = false;
    });
  </script>
  
  <video bind:this={videoElement}></video>
  
  <!-- {#if qrCodeText}
    <div class="qr-text">
      Scanned QR Code: {qrCodeText}
    </div>
  {/if} -->
  