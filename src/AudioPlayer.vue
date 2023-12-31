<template>
  <div id="audio-player">
    <audio-player-controller
      v-bind:duration-sec="audioBuffer.duration"
      v-bind:play-time-sec="$data.$_playTimeSec"
      v-bind:is-playing="$data.$_isPlaying"
      v-bind:is-seeking="$data.$_isSeeking"
      v-bind:is-loop-enabled="isLoopEnabled"
      v-bind:loop-definition="$_loopDefinition"
      v-on:seek-start="$_seekStart"
      v-on:seek-in-sec="$_seekInSec"
      v-on:seek-instantly-in-sec="$_seekInstantlyInSec"
      v-on:seek-end="$_seekEnd"
      v-on:play="$_play"
      v-on:pause="$_pause"
      v-on:enable-loop="$_enableLoopPlayback"
      v-on:disable-loop="$_disableLoopPlayback"
    >
    </audio-player-controller>
    <audio-player-seek-bar
      id="audio-player-seek-bar"
      ref="audioPlayerSeekBar"
      v-bind:current-time-sec="$data.$_playTimeSec"
      v-bind:duration-sec="audioBuffer.duration"
      v-bind:is-loop-enabled="isLoopEnabled"
      v-bind:is-seeking="$data.$_isSeeking"
      v-bind:loop-definition="$_loopDefinition"
      v-on:seek="$_seekInSec"
      v-on:seek-start="$_seekStart"
      v-on:seek-end="$_seekEnd"
    >
    </audio-player-seek-bar>
  </div>
</template>

<style scoped>
#audio-player {
  display: flex;
  flex-direction: column;
}

#audio-player-seek-bar {
  margin-top: 30px;
}
</style>

<script lang="ts">
import { defineComponent } from 'vue';
import AudioPlayerController from './AudioPlayer/AudioPlayerController.vue';
import AudioPlaybackLoopDefinition from './AudioPlayer/modules/AudioPlaybackLoopDefinition';
import AudioPlayerSeekBar from './AudioPlayer/AudioPlayerSeekBar.vue';

type AudioBufferSourceNodePool = { [audioBufferSourceNodeId: string]: AudioBufferSourceNode };

export default defineComponent({
  emits: {
    'update:isLoopEnabled': (isLoopEnabled: boolean) => true,
    'updatePlayTime': (playTimeSec: number) => true,
  },

  components: {
    AudioPlayerController,
    AudioPlayerSeekBar,
  },

  watch: {
    async '$data.$_audioBufferSourceNodePool'(audioBufferSourceNodePool: AudioBufferSourceNodePool) {
      if (Object.keys(audioBufferSourceNodePool).length === 0) {
        await this.$_suspend();
        this.$_seekInSec(0);
      }
    },

    '$data.$_audioBufferSourceNodesWaitingForStop'(audioBufferSourceNodesWaitingForStop: Array<AudioBufferSourceNode>) {
      if (this.$data.$_isPlaying) {
        if (audioBufferSourceNodesWaitingForStop.length === 0) return;
        let audioBufferSourceNodeWaitingForStop = this.$data.$_audioBufferSourceNodesWaitingForStop.pop();
        audioBufferSourceNodeWaitingForStop?.stop();
      }
    },

    '$data.$_isPlaying'(isPLaying: boolean) {
      if (isPLaying) {
        if (this.$data.$_audioBufferSourceNodesWaitingForStop.length === 0) return;
        let audioBufferSourceNodeWaitingForStop = this.$data.$_audioBufferSourceNodesWaitingForStop.pop();
        audioBufferSourceNodeWaitingForStop?.stop();
      }
    },

    isLoopEnabled(isLoopEnabled: boolean) {
      if (this.$_latestAudioBufferSourceNode === undefined) return;
      if (!isLoopEnabled) {
        this.$_seekInSec(this.$_getPlayTimeSec());
      } else {
        this.$_seekInstantlyInSec(this.$_loopDefinition.beginTimeSec);
      }
      this.$_latestAudioBufferSourceNode.loop = isLoopEnabled;
    },

    $_loopDefinition(loopDefinition) {
      if (this.isLoopEnabled) {
        this.$_seekInstantlyInSec(loopDefinition.beginTimeSec);
      }
    },
  },

  props: {
    audioBuffer:    { type: AudioBuffer, required: true },
    audioContext:   { type: AudioContext, required: true },
    loopDefinition: { type: AudioPlaybackLoopDefinition, required: true },
    isLoopEnabled:  { type: Boolean, required: true },
  },

  data(): {
      $_previousTimeSec: number,
      $_playTimeSec: number,
      $_isPlaying: boolean,
      $_isSeeking: boolean,
      $_wasPlayingOnSeek: boolean | undefined,

      $_audioBufferSourceNodePool: AudioBufferSourceNodePool,
      $_audioBufferSourceNodesWaitingForStop: Array<AudioBufferSourceNode>,
      $_audioBufferSourceNodeStartOffsetSec: number,
      $_latestAudioBufferSourceNodeId: string | undefined,
      $_originOfCurrentTime: number,
      $_animtionFrameRequestId: number | undefined,
  } {
    return {
      $_previousTimeSec: 0,
      $_playTimeSec: 0,
      $_isPlaying: false,
      $_isSeeking: false,
      $_wasPlayingOnSeek: undefined,

      $_audioBufferSourceNodePool: {},
      $_audioBufferSourceNodesWaitingForStop: [],
      $_audioBufferSourceNodeStartOffsetSec: 0,
      $_latestAudioBufferSourceNodeId: undefined,
      $_originOfCurrentTime: 0,
      $_animtionFrameRequestId: undefined,
    };
  },

  computed: {
    $_latestAudioBufferSourceNode(): AudioBufferSourceNode | undefined {
      if (this.$data.$_latestAudioBufferSourceNodeId === undefined) return undefined;
      if (!Object.keys(this.$data.$_audioBufferSourceNodePool).includes(this.$data.$_latestAudioBufferSourceNodeId)) return undefined;
      return this.$data.$_audioBufferSourceNodePool[this.$data.$_latestAudioBufferSourceNodeId];
    },

    $_loopDefinition(): AudioPlaybackLoopDefinition {
      if (this.$_isApplicableLoopDefinition) {
        return this.loopDefinition;
      } else {
        return new AudioPlaybackLoopDefinition(0, this.audioBuffer.duration);
      }
    },

    $_isApplicableLoopDefinition(): boolean {
      if (this.loopDefinition === undefined) return false;
      if (this.loopDefinition.beginTimeSec < this.loopDefinition.endTimeSec) return true;
      return false;
    },
  },

  mounted() {
    this.$data.$_animtionFrameRequestId = window.requestAnimationFrame(this.$_onRequestAnimationFrame);
    window.addEventListener('keydown',   this.$_onKeydown);
    window.addEventListener('mousemove', this.$_onMousemove);
    window.addEventListener('mouseup',   this.$_onMouseup);
  },

  async beforeUnmount() {
    window.removeEventListener('mouseup',   this.$_onMouseup);
    window.removeEventListener('mousemove', this.$_onMousemove);
    window.removeEventListener('keydown',   this.$_onKeydown);
    if (this.$data.$_animtionFrameRequestId !== undefined) {
      window.cancelAnimationFrame(this.$data.$_animtionFrameRequestId);
    }
    if (this.$_latestAudioBufferSourceNode !== undefined) {
      this.$_requestAudioBufferSourceNodeToStop();
      await this.$_resume();
    }
    if (this.audioContext.state !== 'closed') await this.audioContext.close();
  },

  methods: {
    $_onKeydown(keyboardEvent: KeyboardEvent) {
      switch (keyboardEvent.code) {
        case 'Space':
          (this.$data.$_isPlaying)? this.$_pause() : this.$_play();
          return true;
        default:
          return false;
      }
    },

    $_onMousemove(mouseEvent: MouseEvent) {
      (this.$refs.audioPlayerSeekBar as InstanceType<typeof AudioPlayerSeekBar>).onMousemove(mouseEvent);
    },

    $_onMouseup(mouseEvent: MouseEvent) {
      (this.$refs.audioPlayerSeekBar as InstanceType<typeof AudioPlayerSeekBar>).onMouseup(mouseEvent);
    },

    /* private */
    $_onRequestAnimationFrame() {
      this.$_update();
      this.$data.$_animtionFrameRequestId = window.requestAnimationFrame(this.$_onRequestAnimationFrame);
    },

    async $_suspend() {
      switch (this.audioContext.state) {
        case 'running':
          await this.audioContext.suspend();
          break;
      }
      this.$data.$_isPlaying = false;
    },

    async $_resume() {
      if (this.$_latestAudioBufferSourceNode === undefined) {
        this.$_seekInSec(0);
        this.$_createNewAudioBufferSourceNode();
      }
      switch (this.audioContext.state) {
        case 'suspended':
          await this.audioContext.resume();
          break;
      }
      this.$data.$_isPlaying = true;
    },

    async $_seekInstantlyInSec(seekTimeSec: number) {
      await this.$_seekStart();
      this.$_seekInSec(seekTimeSec);
      await this.$_seekEnd();
    },

    $_seekInSec(seekTimeSec: number) {
      this.$data.$_audioBufferSourceNodeStartOffsetSec = seekTimeSec;
      this.$data.$_originOfCurrentTime = this.audioContext.currentTime - seekTimeSec;
    },

    async $_seekStart() {
      this.$data.$_isSeeking = true;
      this.$data.$_wasPlayingOnSeek = this.$data.$_isPlaying;
      await this.$_suspend();
    },

    async $_seekEnd() {
      this.$_requestAudioBufferSourceNodeToStop();
      this.$_createNewAudioBufferSourceNode();
      if (this.$data.$_wasPlayingOnSeek) {
        await this.$_resume();
      }
      this.$data.$_wasPlayingOnSeek = undefined;
      this.$data.$_isSeeking = false;
    },

    $_createNewAudioBufferSourceNode() {
      let newAudioBufferSourceNodeId = String(new Date().getTime());
      let newAudioBufferSourceNode = this.audioContext.createBufferSource();
      this.$data.$_latestAudioBufferSourceNodeId = newAudioBufferSourceNodeId;
      this.$data.$_audioBufferSourceNodePool[newAudioBufferSourceNodeId] = newAudioBufferSourceNode;
      newAudioBufferSourceNode.buffer = this.audioBuffer;
      newAudioBufferSourceNode.connect(this.audioContext.destination);
      newAudioBufferSourceNode.onended = () => {
        delete this.$data.$_audioBufferSourceNodePool[newAudioBufferSourceNodeId];
        this.$_suspend();
      };
      newAudioBufferSourceNode.loop = this.isLoopEnabled;
      newAudioBufferSourceNode.loopStart = this.$_loopDefinition.beginTimeSec;
      newAudioBufferSourceNode.loopEnd = this.$_loopDefinition.endTimeSec;
      newAudioBufferSourceNode.start(0, this.$data.$_audioBufferSourceNodeStartOffsetSec);
    },

    $_getPlayTimeSec() {
      let offsetTime = this.audioContext.currentTime - this.$data.$_originOfCurrentTime;
      if (this.$_latestAudioBufferSourceNode === undefined) return offsetTime;
      if (this.$_latestAudioBufferSourceNode.loop) {
        if (offsetTime > this.$_latestAudioBufferSourceNode.loopStart) {
          let loopDuration = this.$_latestAudioBufferSourceNode.loopEnd - this.$_latestAudioBufferSourceNode.loopStart;
          let offsetFromLoopStart = (offsetTime - this.$_latestAudioBufferSourceNode.loopStart) % loopDuration;
          offsetTime = this.$_latestAudioBufferSourceNode.loopStart + offsetFromLoopStart;
        }
      }
      return offsetTime;
    },

    async $_play() {
      await this.$_resume();
    },

    async $_pause() {
      await this.$_suspend();
    },

    $_update() {
      let playTimeSec = this.$_getPlayTimeSec();
      this.$data.$_previousTimeSec = this.$data.$_playTimeSec;
      this.$data.$_playTimeSec = playTimeSec;
      this.$emit('updatePlayTime', playTimeSec);
    },

    $_requestAudioBufferSourceNodeToStop() {
      for (let [ audioBufferSourceNodeId, audioBufferSourceNode ] of Object.entries(this.$data.$_audioBufferSourceNodePool)) {
        delete this.$data.$_audioBufferSourceNodePool[audioBufferSourceNodeId];
        this.$data.$_audioBufferSourceNodesWaitingForStop.push(audioBufferSourceNode);
        audioBufferSourceNode.onended = null;
      }
    },

    $_enableLoopPlayback() {
      this.$emit('update:isLoopEnabled', true);
    },

    $_disableLoopPlayback() {
      this.$emit('update:isLoopEnabled', false);
    },
  },
});
</script>