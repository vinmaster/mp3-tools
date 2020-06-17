<template>
  <div class="container">
    <input type="file" accept="audio/*" @change="fileChange" multiple />
    <audio style="width: 100%;" controls ref="audioElement" />
    <button role="switch" aria-checked="false" @click="playPauseClicked">
      <span>Play/Pause</span>
    </button>
    <canvas ref="canvasElement"></canvas>
    <canvas ref="canvasElement2"></canvas>
  </div>
</template>

<script lang="ts">
import { Component, Vue } from 'vue-property-decorator';

@Component
export default class Main extends Vue {
  files: FileList | undefined;
  currentFileIndex = 0;

  audioContext = new AudioContext();
  sourceNode!: MediaElementAudioSourceNode;
  analyserNode!: AnalyserNode;

  canvasContext!: CanvasRenderingContext2D;
  canvasContext2!: CanvasRenderingContext2D;
  dataArray!: Uint8Array;
  dataArray2!: Uint8Array;

  $refs!: {
    audioElement: HTMLMediaElement;
    canvasElement: HTMLCanvasElement;
    canvasElement2: HTMLCanvasElement;
  };

  created() {
    console.log('created');
  }

  mounted() {
    console.log('mounted');
    this.$refs.audioElement.onplay = () => {
      this.drawCanvas();
    };

    const canvasContext = this.$refs.canvasElement.getContext('2d');
    const canvasContext2 = this.$refs.canvasElement2.getContext('2d');
    if (canvasContext && canvasContext2) {
      this.canvasContext = canvasContext;
      this.canvasContext2 = canvasContext2;
      this.canvasContext2.clearRect(
        0,
        0,
        this.$refs.canvasElement2.width,
        this.$refs.canvasElement2.height
      );
    }

    fetch('test.mp3')
      .then(response => response.arrayBuffer())
      .then(arrayBuffer => {
        const blob = new Blob([arrayBuffer], { type: 'audio/mp3' });
        this.loadToAudioElement(blob);

        this.setupVisual(arrayBuffer);
      });
  }

  beforeDestroy() {
    console.log('beforeDestroy');
    this.cleanUp();
  }

  setupVisual(arrayBuffer: ArrayBuffer) {
    this.audioContext.decodeAudioData(arrayBuffer).then(audioBuffer => {
      console.log('audio buffer', audioBuffer);

      this.analyserNode = this.audioContext.createAnalyser();
      this.sourceNode.connect(this.analyserNode);

      const bufferLength = this.analyserNode.frequencyBinCount;
      this.dataArray = new Uint8Array(bufferLength);
      this.dataArray2 = new Uint8Array(bufferLength);

      console.log(this.audioContext);
      console.log(this.sourceNode);
      console.log(this.analyserNode);
      this.drawCanvas();
    });
  }

  drawCanvas() {
    if (!this.$refs.audioElement.paused) {
      requestAnimationFrame(this.drawCanvas.bind(this));
    }

    // Waveform/oscilloscope
    this.analyserNode.getByteTimeDomainData(this.dataArray);

    const width = this.$refs.canvasElement.width;
    const height = this.$refs.canvasElement.height;
    this.canvasContext.fillStyle = '#222';
    this.canvasContext.fillRect(0, 0, width, height);

    this.canvasContext.lineWidth = 2;
    this.canvasContext.strokeStyle = '#01ff00';

    this.canvasContext.beginPath();

    const sliceWidth = (width * 1.0) / this.dataArray.length;
    let x = 0;

    for (let i = 0; i < this.dataArray.length; i++) {
      const v = this.dataArray[i] / 128.0;
      const y = (v * height) / 2;

      if (i === 0) {
        this.canvasContext.moveTo(x, y);
      } else {
        this.canvasContext.lineTo(x, y);
      }

      x += sliceWidth;
    }

    this.canvasContext.lineTo(this.$refs.canvasElement.width, this.$refs.canvasElement.height / 2);
    this.canvasContext.stroke();

    // Frequency bar
    this.analyserNode.getByteFrequencyData(this.dataArray2);

    const width2 = this.$refs.canvasElement2.width;
    const height2 = this.$refs.canvasElement2.height;
    this.canvasContext2.fillStyle = '#222';
    this.canvasContext2.fillRect(0, 0, width2, height2);

    const barWidth = (width2 / this.dataArray2.length) * 2.5;
    let barHeight = 0;
    let x2 = 0;

    for (let i = 0; i < this.dataArray2.length; i++) {
      barHeight = this.dataArray2[i];
      this.canvasContext2.fillStyle = `rgb(1,${barHeight + 100},50)`;
      this.canvasContext2.fillRect(x2, height2 - barHeight / 2, barWidth, barHeight);

      x2 += barWidth + 1;
    }
  }

  playPauseClicked() {
    // Autoplay policy
    if (this.audioContext.state === 'suspended') {
      this.audioContext.resume();
    }

    if (!this.hasAudioSource) return;

    if (this.$refs.audioElement.paused) {
      this.$refs.audioElement.play();
    } else {
      this.$refs.audioElement.pause();
    }
  }

  fileChange(event: { target: HTMLInputElement }) {
    const files = event.target.files;
    if (!files || files.length === 0) return;

    this.cleanUp();

    this.currentFileIndex = 0;
    this.files = files;

    const file = files[this.currentFileIndex] as any;
    file.arrayBuffer().then((arrayBuffer: ArrayBuffer) => {
      this.setupVisual(arrayBuffer);
      console.log('array buffer 1', arrayBuffer);
    });

    // Slower performance
    // const resp = new Response(files[this.currentFileIndex]);
    // resp.arrayBuffer().then(arrayBuffer => {
    //   console.log('array buffer 2', arrayBuffer);
    // });

    // this.readBlobAsArrayBuffer(files[this.currentFileIndex]).then(arrayBuffer => {
    //   if (!arrayBuffer) return;
    //   console.log('array buffer 0', arrayBuffer);
    //   this.audioContext.decodeAudioData(arrayBuffer).then(audioBuffer => {
    //     console.log('audio buffer', audioBuffer);
    //   });
    // });

    // File is a kind of a Blob
    this.loadToAudioElement(files[this.currentFileIndex]);
  }

  loadToAudioElement(blob: Blob) {
    const objectUrl = URL.createObjectURL(blob);
    this.$refs.audioElement.setAttribute('src', objectUrl);

    if (!this.sourceNode) {
      this.sourceNode = this.audioContext.createMediaElementSource(this.$refs.audioElement);
      this.sourceNode.connect(this.audioContext.destination);
    }
  }

  readBlobAsArrayBuffer(blob: Blob): Promise<ArrayBuffer | null> {
    const fileReader = new FileReader();
    return new Promise((resolve, reject) => {
      fileReader.onerror = () => {
        fileReader.abort();
        reject(fileReader.error);
      };
      fileReader.onload = event => {
        const arrayBuffer = fileReader.result as ArrayBuffer;
        resolve(arrayBuffer);
      };
      fileReader.readAsArrayBuffer(blob);
    });
  }

  cleanUp() {
    if (this.files && this.files.length > 0) {
      // Remove object in memory
      URL.revokeObjectURL(this.$refs.audioElement.getAttribute('src') as string);
      this.files = undefined;
    }

    this.$refs.audioElement.removeAttribute('src');
  }

  get hasAudioSource() {
    return this.$refs.audioElement.hasAttribute('src');
  }
}
</script>

<!-- Add "scoped" attribute to limit CSS to this component only -->
<style scoped>
.container {
  display: flex;
  flex-direction: column;
}
</style>
