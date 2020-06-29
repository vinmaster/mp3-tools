<template>
  <div class="container">
    <h1>MP3 Tools</h1>
    <span>Sample Rate: {{ sampleRate }}</span>
    <input type="file" accept="audio/*" @change="fileChanged" multiple />
    <button role="switch" aria-checked="false" @click="playPauseClicked">
      <span>Play/Pause</span>
    </button>
    <audio style="width: 100%;" controls ref="audioElement" />
    <h2>Waveform</h2>
    <div id="main-wavesurfer"></div>
    <Waveform :audioBuffer="audioBuffer"></Waveform>
    <Oscilloscope
      ref="oscilloscope"
      :audioElement="$refs.audioElement"
      :analyserNode="analyserNode"
    ></Oscilloscope>
    <FrequencyBar
      ref="frequencyBar"
      :audioElement="$refs.audioElement"
      :analyserNode="analyserNode"
    ></FrequencyBar>
  </div>
</template>

<script lang="ts">
import { Component, Vue } from 'vue-property-decorator';
import Oscilloscope from './Oscilloscope.vue';
import FrequencyBar from './FrequencyBar.vue';
import Waveform from './Waveform.vue';
import { Config } from '../config';

@Component({
  components: { Oscilloscope, FrequencyBar, Waveform },
})
export default class Main extends Vue {
  sampleRate = Config.sampleRate;
  files: FileList | undefined;
  currentFileIndex = 0;

  audioContext = new AudioContext({ sampleRate: Config.sampleRate });
  analyserNode: AnalyserNode = null as any;
  sourceNode!: MediaElementAudioSourceNode;
  audioBuffer: AudioBuffer = null as any;

  wavesurfer: any;

  $refs!: {
    audioElement: HTMLMediaElement;
    oscilloscope: Oscilloscope;
    frequencyBar: FrequencyBar;
  };

  mounted() {
    this.$refs.audioElement.onplay = () => {
      this.$refs.oscilloscope.drawCanvas();
      this.$refs.frequencyBar.drawCanvas();
    };

    this.$refs.audioElement.onloadeddata = () => {
      this.$refs.oscilloscope.setupVisual();
      this.$refs.frequencyBar.setupVisual();
    };

    const WaveSurfer = window['WaveSurfer' as any] as any;
    this.wavesurfer = WaveSurfer.create({
      container: '#main-wavesurfer',
      backgroundColor: '#222',
      waveColor: '#01ff00',
      progressColor: '#007500',
      backend: 'MediaElement',
      mediaControls: true,
      removeMediaElementOnDestroy: false,
    });

    this.loadStatic();
  }

  beforeDestroy() {
    this.cleanUp();
  }

  loadStatic() {
    fetch('test.mp3')
      .then(response => response.arrayBuffer())
      .then(arrayBuffer => {
        const blob = new Blob([arrayBuffer], { type: 'audio/mp3' });
        this.loadToAudioElement(blob);

        return this.audioContext.decodeAudioData(arrayBuffer);
      })
      .then(audioBuffer => {
        this.audioBuffer = audioBuffer;
      });
  }

  fileChanged(event: { target: HTMLInputElement }) {
    const files = event.target.files;
    if (!files || files.length === 0) return;

    this.cleanUp();

    this.currentFileIndex = 0;
    this.files = files;

    const file = files[this.currentFileIndex] as any;
    file
      .arrayBuffer()
      .then((arrayBuffer: ArrayBuffer) => {
        return this.audioContext.decodeAudioData(arrayBuffer);
      })
      .then((audioBuffer: AudioBuffer) => {
        this.audioBuffer = audioBuffer;
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

    this.wavesurfer.empty();
    this.wavesurfer.loadMediaElement(this.$refs.audioElement);

    if (!this.sourceNode) {
      this.sourceNode = this.audioContext.createMediaElementSource(this.$refs.audioElement);
      this.sourceNode.connect(this.audioContext.destination);

      this.analyserNode = this.audioContext.createAnalyser();
      this.sourceNode.connect(this.analyserNode);
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

<style scoped>
.container {
  display: flex;
  flex-direction: column;
}
</style>
