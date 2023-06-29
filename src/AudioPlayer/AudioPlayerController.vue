<template>
  <div id="audio-player-controller">
    <button v-on:mousedown.stop="$_seekToHead">
      Seek To Head
    </button>

    <button
      v-on:mousedown.stop="$_seekBackwardStart"
      v-on:mouseup.stop="$_seekBackwardEnd"
      v-on:mouseout.stop="$_seekBackwardEnd"
    >
      Seek Backward
    </button>

    <button v-if="isPlaying" v-on:mousedown.stop="$_pause">Pause</button>
    <button v-else           v-on:mousedown.stop="$_play">Play</button>

    <button
      v-on:mousedown.stop="$_seekForwardStart"
      v-on:mouseup.stop="$_seekForwardEnd"
      v-on:mouseout.stop="$_seekForwardEnd"
    >
      Seek Forward
    </button>

    <button v-on:mousedown.stop="$_seekToTail">
      Seek To Tail
    </button>

    <button
      class="loop-enabled"
      v-if="isLoopEnabled"
      v-on:mousedown.stop="$_disableLoop"
    >
      Disable Loop
    </button>
    <button
      v-else
      v-on:mousedown.stop="$_enableLoop"
    >
      Enable Loop
    </button>
  </div>
</template>

<style scoped>
#audio-player-controller {
  width: 100%;
  display: flex;
  flex-direction: row;
  align-items: center;
}

.loop-enabled {
  color: #16f43f !important;
}
</style>

<script lang="ts">
import { defineComponent } from 'vue';
import AudioPlaybackLoopDefinition from './modules/AudioPlaybackLoopDefinition';

const seekIntervalTimeSec = 0.1;
const seekTimeStepSec = 0.1;

export default defineComponent({
  props: {
    durationSec: { type: Number, default: 0 },
    playTimeSec: { type: Number, default: 0 },
    isPlaying: { type: Boolean, default: false },
    isSeeking: { type: Boolean, default: false },
    isLoopEnabled: { type: Boolean, required: true },
    loopDefinition: { type: AudioPlaybackLoopDefinition, required: true },
  },

  data(): {
    $_seekTargetTimeSec: number,
    $_intervalId: number | undefined,
  } {
    return {
      $_seekTargetTimeSec: 0,
      $_intervalId: undefined,
    };
  },

  methods: {
    $_seekToHead() { this.$emit('seek-instantly-in-sec', 0) },

    $_seekToTail() { this.$emit('seek-instantly-in-sec', this.durationSec) },

    $_enableLoop() { this.$emit('enable-loop') },

    $_disableLoop() { this.$emit('disable-loop') },

    $_play() { this.$emit('play') },

    $_pause() { this.$emit('pause') },

    $_seekBackwardStart() {
      if (this.isSeeking) return;
      this.$emit('seek-start');
      this.$data.$_seekTargetTimeSec = this.playTimeSec;
      this.$_updateSeekBackwardTargetTime();
      this.$_setInterval(this.$_updateSeekBackwardTargetTime, seekIntervalTimeSec);
    },

    $_updateSeekBackwardTargetTime() {
      let nextSeekTargetTimeSec = this.$data.$_seekTargetTimeSec - seekTimeStepSec;
      if (nextSeekTargetTimeSec < 0) nextSeekTargetTimeSec = 0;
      this.$data.$_seekTargetTimeSec = nextSeekTargetTimeSec;
      this.$emit('seek-in-sec', nextSeekTargetTimeSec)
    },

    $_seekBackwardEnd() {
      if (!this.isSeeking) return;
      this.$_clearInterval();
      this.$emit('seek-end');
    },

    $_seekForwardStart() {
      if (this.isSeeking) return;
      this.$emit('seek-start');
      this.$data.$_seekTargetTimeSec = this.playTimeSec;
      this.$_updateSeekForwardTargetTime();
      this.$_clearInterval();
      this.$_setInterval(this.$_updateSeekForwardTargetTime, seekIntervalTimeSec);
    },

    $_updateSeekForwardTargetTime() {
      let nextSeekTargetTimeSec = this.$data.$_seekTargetTimeSec + seekTimeStepSec;
      if (nextSeekTargetTimeSec > this.durationSec) nextSeekTargetTimeSec = this.durationSec;
      this.$data.$_seekTargetTimeSec = nextSeekTargetTimeSec;
      this.$emit('seek-in-sec', nextSeekTargetTimeSec)
    },

    $_seekForwardEnd() {
      if (!this.isSeeking) return;
      this.$_clearInterval();
      this.$emit('seek-end');
    },

    $_clearInterval() {
      if (this.$data.$_intervalId !== undefined) window.clearInterval(this.$data.$_intervalId);
      this.$data.$_intervalId = undefined;
    },

    $_setInterval(callback: () => void, intervalSec: number) {
      this.$_clearInterval();
      this.$data.$_intervalId = window.setInterval(callback, intervalSec);
    },
  },
});
</script>