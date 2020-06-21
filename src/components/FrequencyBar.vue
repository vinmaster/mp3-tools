<template>
  <div class="container">
    <h2>Frequency Bar</h2>
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

    this.canvasContext.clearRect(0, 0, this.$refs.canvasElement.width, this.$refs.canvasElement.height);

    this.drawCanvas();
  }

  drawCanvas() {
    if (!this.audioElement.paused) {
      this.requestId = requestAnimationFrame(this.drawCanvas.bind(this));
    }

    this.analyserNode.getByteFrequencyData(this.dataArray);

    const width2 = this.$refs.canvasElement.width;
    const height2 = this.$refs.canvasElement.height;
    this.canvasContext.fillStyle = '#222';
    this.canvasContext.fillRect(0, 0, width2, height2);

    const barWidth = (width2 / this.dataArray.length) * 2.5;
    let barHeight = 0;
    let x2 = 0;

    for (let i = 0; i < this.dataArray.length; i++) {
      barHeight = this.dataArray[i];
      this.canvasContext.fillStyle = `rgb(1,${barHeight + 100},50)`;
      this.canvasContext.fillRect(x2, height2 - barHeight / 2, barWidth, barHeight);

      x2 += barWidth + 1;
    }
  }
}
</script>

<style scoped>
.container {
  display: flex;
  flex-direction: column;
}
</style>
