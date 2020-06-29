<template>
  <div class="container">
    <h2>Modified Waveform (Removing quiet parts)</h2>
    <audio style="width: 100%;" controls ref="audioElement" />
    <canvas ref="canvasElement"></canvas>
    <div id="waveform-wavesurfer"></div>
    <a class="button" ref="downloadElement">Download</a>
  </div>
</template>

<script lang="ts">
import { Component, Vue, Prop, Watch } from 'vue-property-decorator';
import { Config } from '../config';

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

  audioContext = new AudioContext({ sampleRate: Config.sampleRate });
  sourceNode!: MediaElementAudioSourceNode;

  canvasContext!: CanvasRenderingContext2D;
  wavesurfer: any;

  $refs!: {
    audioElement: HTMLMediaElement;
    canvasElement: HTMLCanvasElement;
    downloadElement: HTMLLinkElement;
  };

  @Watch('audioBuffer')
  onAudioBufferChanged(value: AudioBuffer, oldValue: AudioBuffer) {
    if (value) {
      const newBuffer = this.createNewAudioBuffer();
      this.setupView(newBuffer);
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

    // this.wavesurfer.on('ready', async () => {
    //   console.log('wavesurfer', this.wavesurfer);
    //   console.log('pcm', (await this.wavesurfer.exportPCM(5762821)));
    //   setTimeout(() => {
    //     // console.log('mergedPeaks', this.wavesurfer.backend.mergedPeaks);
    //   }, 100);
    // });

    if (this.audioBuffer) {
      const newBuffer = this.createNewAudioBuffer();
      this.setupView(newBuffer);
    }
  }

  beforeDestroy() {
    this.cleanUp();
  }

  setupView(newBuffer: AudioBuffer | null) {
    if (newBuffer) {
      this.drawCanvas(newBuffer);

      const blob = this.bufferToWave(newBuffer);

      // Load blob into audio element
      const objectUrl = URL.createObjectURL(blob);
      this.$refs.audioElement.setAttribute('src', objectUrl);

      // Load audio with wavesurfer
      // wavesurfer.load(this.$refs.audioElement);
      this.wavesurfer.empty();
      this.wavesurfer.loadMediaElement(this.$refs.audioElement);
      // wavesurfer.loadBlob(blob);

      // Enable download of new audio
      this.$refs.downloadElement.href = objectUrl;
      this.$refs.downloadElement.setAttribute('download', 'audio.wav');
    }

    // const scriptProcessor = this.audioContext.createScriptProcessor(this.audioBuffer.length, 1, 1);
    // const analyser = this.audioContext.createAnalyser();
    // const bufferLength = analyser.frequencyBinCount;
    // const frequencyArray = new Uint8Array(bufferLength);
    // const timeDomainArray = new Uint8Array(bufferLength);
    // analyser.connect(scriptProcessor)
    // scriptProcessor.onaudioprocess = () => {
    //   analyser.getByteFrequencyData(frequencyArray);
    //   analyser.getByteTimeDomainData(timeDomainArray);
    //   console.log('frequencyArray', frequencyArray);
    // }
  }

  createNewAudioBuffer(): AudioBuffer | null {
    // No audio buffer
    if (!this.audioBuffer) {
      return null;
    }

    const leftChannel = this.audioBuffer.getChannelData(0);
    const rightChannel = this.audioBuffer.getChannelData(1);

    // let newLeftChannel = newBuffer.getChannelData(0);
    let newLeftChannel = new Float32Array(leftChannel.length);
    let removedData = new Float32Array(leftChannel.length);
    let removedDataLength = 0;
    const minValue = 0.015;
    const minSeconds = 0.25;
    const silences = [];

    const filled = [];
    // console.log('my peaks', peaks);
    console.log(this.audioContext);
    // const sampleSize = this.audioContext.sampleRate;
    // Sample size length of 100ms. Sample Rate is number per second
    const numPerSecond = 2;
    const sampleSize = this.audioContext.sampleRate / numPerSecond;

    console.log('channel length', leftChannel.length);
    console.log('sample size', sampleSize);
    console.log('channel / sample', leftChannel.length / this.audioContext.sampleRate);
    let total = 0;
    for (let i = 0; i < leftChannel.length; i++) {
      // if (Math.abs(leftChannel[i]) < 0.055) continue;
      if (i % sampleSize === 0) {
        // const rms = Math.sqrt(total / (sampleSize / 2));
        // let dB = 20 * Math.log10(rms);
        // // let dB = Math.round((Math.round(20 * (0.43429 * Math.log(rms)) * 100) / 100) * 100) / 100;
        // if (dB === -Infinity) {
        //   dB = 0;
        // }
        // filled.push(dB);

        const slice = leftChannel.slice(i, i + sampleSize);
        total = slice.reduce((acc, val) => acc + val);
        filled.push(Math.abs(total));
        for (let j = i; j < i + sampleSize; j++) {
          if (Math.abs(total) < 0.5) {
            removedData[removedDataLength++] = leftChannel[j];
            newLeftChannel[j] = 0;
          } else {
            newLeftChannel[j] = leftChannel[j];
          }
        }
      }
    }
    // console.log('filled', filled);

    // Remove quiet parts
    // TODO: Need to do this for both channels
    console.log('before', newLeftChannel.length);
    newLeftChannel = newLeftChannel.filter(n => n !== 0);
    console.log('after', newLeftChannel.length);

    console.log('removed', removedData.length);
    removedData = removedData.filter(n => n !== 0);

    // const newBuffer = this.audioContext.createBuffer(
    //   this.audioBuffer.numberOfChannels,
    //   this.audioBuffer.length,
    //   this.audioBuffer.sampleRate
    // );
    const newBuffer = this.audioContext.createBuffer(
      1,
      newLeftChannel.length,
      // removedData.length,
      this.audioBuffer.sampleRate
    );
    newBuffer.copyToChannel(newLeftChannel, 0);

    return newBuffer;
  }

  drawCanvas(newBuffer: AudioBuffer) {
    const newLeftChannel = newBuffer.getChannelData(0);

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
    // this.canvasContext.strokeStyle = '#121';
    this.canvasContext.strokeStyle = '#01ff00';
    // this.canvasContext.globalCompositeOperation = 'lighter';
    this.canvasContext.translate(0, height / 2);
    // this.canvasContext.globalAlpha = 0.06;

    // Trying to improve render using channel data
    // let count = 0;
    // const step = Math.floor(newLeftChannel.length / this.wavesurfer.drawer.getWidth());
    // for (let i = 0; i < newLeftChannel.length; i += step) {
    //   count++;
    //   // if (i % width !== 0) continue; // Speed up drawing
    //   const x = Math.floor((width * i) / newLeftChannel.length);
    //   const slice = newLeftChannel.slice(i, i + step);
    //   const min = Math.min(...slice);
    //   const max = Math.max(...slice);
    //   const y = (max * height) / 2;
    //   // const y = (newLeftChannel[i] * height) / 2;
    //   this.canvasContext.beginPath();
    //   // this.canvasContext.moveTo(x, 0);
    //   this.canvasContext.moveTo(x, (min * height) / 2);
    //   this.canvasContext.lineTo(x + 1, y);
    //   this.canvasContext.stroke();
    // }
    // console.log('count', count, width, step);

    // Original render using channel data
    for (let i = 0; i < newLeftChannel.length; i++) {
      if (i % 500 !== 0) continue; // Speed up drawing
      const x = Math.floor((width * i) / newLeftChannel.length);
      const y = (newLeftChannel[i] * height) / 2;
      this.canvasContext.beginPath();
      this.canvasContext.moveTo(x, 0);
      this.canvasContext.lineTo(x + 1, y);
      this.canvasContext.stroke();
    }

    // Render using peaks algorithm
    // const leftChannel = this.audioBuffer.getChannelData(0);
    // const rightChannel = this.audioBuffer.getChannelData(1);
    // const peaks = this.getPeaks([leftChannel, rightChannel], 0, this.wavesurfer.drawer.getWidth());
    // console.log('width', this.wavesurfer.drawer.getWidth(), width);
    // console.log('peaks length', peaks.length);
    // // console.log(peaks);
    // for (let i = 0; i < peaks.length; i++) {
    //   const x = Math.floor(i / 5);
    //   const y = (peaks[i] * height) / 2;
    //   this.canvasContext.beginPath();
    //   this.canvasContext.moveTo(x, 0);
    //   this.canvasContext.lineTo(x, y);
    //   this.canvasContext.stroke();
    // }

    this.canvasContext.restore();
  }

  getPeaks(channels: Float32Array[], first: number, last: number): number[] {
    const mergedPeaks: number[] = [];
    mergedPeaks[2 * (last - 1)] = 0;
    mergedPeaks[2 * (last - 1) + 1] = 0;
    const sampleSize = this.audioBuffer.length / last;
    const sampleStep = ~~(sampleSize / 10) || 1;
    let c;

    for (c = 0; c < channels.length; c++) {
      const channel = channels[c];

      let i;

      for (i = first; i <= last; i++) {
        const start = ~~(i * sampleSize);
        const end = ~~(start + sampleSize);

        let min = channel[start];
        let max = min;
        let j;

        for (j = start; j < end; j += sampleStep) {
          const value = channel[j];

          if (value > max) max = value;
          if (value < min) min = value;
        }

        mergedPeaks[2 * i] = max;
        mergedPeaks[2 * i + 1] = min;
      }
    }

    return mergedPeaks;
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

    this.$refs.downloadElement.href = '';
    this.$refs.downloadElement.removeAttribute('download');
  }
}
</script>

<style scoped>
.container {
  display: flex;
  flex-direction: column;
}
.button {
  display: inline-block;
  padding: 0.35em 1.2em;
  border: 0.1em solid white;
  margin: 10px 0.3em 0.3em 0;
  border-radius: 0.12em;
  box-sizing: border-box;
  text-decoration: none;
  font-weight: 300;
  text-align: center;
  transition: all 0.2s;
  color: black;
  background-color: white;
  outline: 2px solid black;
}
.button:hover {
  color: white;
  background: black;
}
</style>
