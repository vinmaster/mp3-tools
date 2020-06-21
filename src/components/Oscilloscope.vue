<template>
  <div class="container">
    <h2>Oscilloscope</h2>
    <canvas ref="canvasElement"></canvas>
  </div>
</template>

<script lang="ts">
import { Component, Vue, Prop, Watch } from 'vue-property-decorator';

@Component
export default class Waveform extends Vue {
  @Prop({ required: true })
  analyserNode!: AnalyserNode;
  
  @Prop()
  audioElement!: HTMLMediaElement;

  requestId!: number;
  canvasContext!: CanvasRenderingContext2D;
  dataArray!: Uint8Array;

  $refs!: {
    canvasElement: HTMLCanvasElement;
  };

  @Watch('analyserNode')
  onAnalyserNodeChanged(value: AnalyserNode, oldValue: AnalyserNode) {
    this.setupVisual()
  }

  mounted() {
    const canvasContext = this.$refs.canvasElement.getContext('2d');
    if (canvasContext) {
      this.canvasContext = canvasContext;
    }
    this.setupVisual();
  }

  beforeDestroy() {
    cancelAnimationFrame(this.requestId);
  }

  setupVisual() {
    if (!this.analyserNode) return;

    const bufferLength = this.analyserNode.frequencyBinCount;
    this.dataArray = new Uint8Array(bufferLength);

    this.drawCanvas();
  }

  drawCanvas() {
    if (!this.audioElement.paused) {
      this.requestId = requestAnimationFrame(this.drawCanvas.bind(this));
    }

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
  }
}
</script>

<style scoped>
.container {
  display: flex;
  flex-direction: column;
}
</style>
