<template>
  <audio-player
    ref="audioPlayer"
    v-if="audioBuffer && audioContext"
    v-bind:audio-buffer="audioBuffer"
    v-bind:audio-context="audioContext"
    v-bind:loop-definition="loopDefinition"
    v-model:is-loop-enabled="isLoopEnabled"
  >
  </audio-player>
</template>

<script lang="ts">
import { defineComponent } from 'vue';
import AudioPlayer from './AudioPlayer.vue';
import AudioPlaybackLoopDefinition from './AudioPlayer/modules/AudioPlaybackLoopDefinition';

export default defineComponent({
  name: 'App',

  components: { AudioPlayer },

  data(): {
    audioContext: AudioContext | undefined,
    audioBuffer: AudioBuffer | undefined,
    loopDefinition: AudioPlaybackLoopDefinition,
    isLoopEnabled: boolean,
  } {
    return {
      audioContext: undefined,
      audioBuffer: undefined,
      loopDefinition: new AudioPlaybackLoopDefinition(0, 0),
      isLoopEnabled: false,
    };
  },

  computed: {
    audioLoaded() {
      if (this.audioContext === undefined) return false;
      if (this.audioBuffer === undefined) return false;
      return true;
    },
  },

  mounted() {
    fetch('sample.mp3')
      .then((response: Response) => response.arrayBuffer())
      .then((audioData: ArrayBuffer) => {
        this.audioContext = new AudioContext();
        return this.audioContext.decodeAudioData(audioData.slice(0));
      })
      .then((audioBuffer: AudioBuffer) => {
        this.loopDefinition.beginTimeSec = 0;
        this.loopDefinition.endTimeSec = audioBuffer.duration;
        this.audioBuffer = audioBuffer;
      });
  },
});
</script>
