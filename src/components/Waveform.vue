<template>
  <div class="container">
    <h2>Modified Waveform</h2>
    <audio style="width: 100%;" controls ref="audioElement" />
    <canvas ref="canvasElement"></canvas>
    <div id="waveform-wavesurfer"></div>
  </div>
</template>

<script lang="ts">
import { Component, Vue, Prop, Watch } from 'vue-property-decorator';

@Component({
  // props: {
  //   audioBuffer: {
  //     type: AudioBuffer,
  //     required: true,
  //   },
  // },
})
export default class Waveform extends Vue {
  @Prop({ required: true })
  audioBuffer!: AudioBuffer;

  audioContext = new AudioContext();
  sourceNode!: MediaElementAudioSourceNode;

  canvasContext!: CanvasRenderingContext2D;
  wavesurfer: any;

  $refs!: {
    audioElement: HTMLMediaElement;
    canvasElement: HTMLCanvasElement;
  };

  @Watch('audioBuffer')
  onAudioBufferChanged(value: AudioBuffer, oldValue: AudioBuffer) {
    if (value) {
      this.drawCanvas();
    }
  }

  mounted() {
    const canvasContext = this.$refs.canvasElement.getContext('2d');
    if (canvasContext) {
      this.canvasContext = canvasContext;
    }

    const WaveSurfer = window['WaveSurfer' as any] as any;
    this.wavesurfer = WaveSurfer.create({
      container: '#waveform-wavesurfer',
      backgroundColor: '#222',
      waveColor: '#01ff00',
      progressColor: '#007500',
      backend: 'MediaElement',
      mediaControls: true,
      removeMediaElementOnDestroy: false,
    });

    if (this.audioBuffer) {
      this.drawCanvas();
    }
  }

  beforeDestroy() {
    this.cleanUp();
  }

  drawCanvas() {
    // No audio buffer
    if (!this.audioBuffer) {
      return;
    }
    // console.log('audio buffer', this.audioBuffer, {
    //   duration: this.audioBuffer.duration,
    //   length: this.audioBuffer.length,
    //   numberOfChannels: this.audioBuffer.numberOfChannels,
    //   sampleRate: this.audioBuffer.sampleRate,
    //   frames: this.audioBuffer.length,
    //   duration2: this.audioBuffer.length / this.audioBuffer.sampleRate,
    // });
    const leftChannel = this.audioBuffer.getChannelData(0);

    const newBuffer = this.audioContext.createBuffer(
      this.audioBuffer.numberOfChannels,
      this.audioBuffer.length,
      this.audioBuffer.sampleRate
    );

    const newLeftChannel = newBuffer.getChannelData(0);
    for (let i = 0; i < leftChannel.length; i++) {
      if (Math.abs(leftChannel[i]) < 0.5) {
        newLeftChannel[i] = leftChannel[i];
        // newLeftChannel[i] = 0;
      } else {
        newLeftChannel[i] = 0;
      }
    }

    const width = this.$refs.canvasElement.width;
    const height = this.$refs.canvasElement.height;

    this.canvasContext.save();
    this.canvasContext.clearRect(
      0,
      0,
      this.$refs.canvasElement.width,
      this.$refs.canvasElement.height
    );
    this.canvasContext.fillStyle = '#222';
    this.canvasContext.fillRect(0, 0, width, height);
    this.canvasContext.strokeStyle = '#121';
    // this.canvasContext.strokeStyle = '#01ff00';
    this.canvasContext.globalCompositeOperation = 'lighter';
    this.canvasContext.translate(0, height / 2);
    this.canvasContext.globalAlpha = 0.06;

    for (let i = 0; i < newLeftChannel.length; i++) {
      if (i % 50 !== 0) continue; // Speed up drawing
      const x = Math.floor((width * i) / newLeftChannel.length);
      const y = (newLeftChannel[i] * height) / 2;
      this.canvasContext.beginPath();
      this.canvasContext.moveTo(x, 0);
      this.canvasContext.lineTo(x + 1, y);
      this.canvasContext.stroke();
    }
    this.canvasContext.restore();

    const blob = this.bufferToWave(newBuffer);

    const objectUrl = URL.createObjectURL(blob);
    this.$refs.audioElement.setAttribute('src', objectUrl);

    // wavesurfer.load(this.$refs.audioElement);
    this.wavesurfer.empty();
    this.wavesurfer.loadMediaElement(this.$refs.audioElement);
    // wavesurfer.loadBlob(blob);
  }

  bufferToWave(abuffer: AudioBuffer) {
    const len = abuffer.length;

    const numOfChan = abuffer.numberOfChannels,
      length = len * numOfChan * 2 + 44,
      buffer = new ArrayBuffer(length),
      view = new DataView(buffer),
      channels = [];
    let i,
      sample,
      offset = 0,
      pos = 0;

    const setUint16 = (data: any) => {
      view.setUint16(pos, data, true);
      pos += 2;
    };

    const setUint32 = (data: any) => {
      view.setUint32(pos, data, true);
      pos += 4;
    };

    // write WAVE header
    setUint32(0x46464952); // "RIFF"
    setUint32(length - 8); // file length - 8
    setUint32(0x45564157); // "WAVE"

    setUint32(0x20746d66); // "fmt " chunk
    setUint32(16); // length = 16
    setUint16(1); // PCM (uncompressed)
    setUint16(numOfChan);
    setUint32(abuffer.sampleRate);
    setUint32(abuffer.sampleRate * 2 * numOfChan); // avg. bytes/sec
    setUint16(numOfChan * 2); // block-align
    setUint16(16); // 16-bit (hardcoded in this demo)

    setUint32(0x61746164); // "data" - chunk
    setUint32(length - pos - 4); // chunk length

    // write interleaved data
    for (i = 0; i < abuffer.numberOfChannels; i++) channels.push(abuffer.getChannelData(i));

    while (pos < length) {
      for (i = 0; i < numOfChan; i++) {
        // interleave channels
        sample = Math.max(-1, Math.min(1, channels[i][offset])); // clamp
        sample = (0.5 + sample < 0 ? sample * 32768 : sample * 32767) | 0; // scale to 16-bit signed int
        view.setInt16(pos, sample, true); // write 16-bit sample
        pos += 2;
      }
      offset++; // next source sample
    }

    // create Blob
    return new Blob([buffer], { type: 'audio/wav' });
    // return new Blob([buffer], { type: 'audio/mpeg' });
  }

  cleanUp() {
    // Remove object in memory
    URL.revokeObjectURL(this.$refs.audioElement.getAttribute('src') as string);

    this.$refs.audioElement.removeAttribute('src');
  }
}
</script>

<style scoped>
.container {
  display: flex;
  flex-direction: column;
}
</style>
