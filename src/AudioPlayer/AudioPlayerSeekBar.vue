<template>
  <div
    id="seek-bar-container"
    v-bind="$attrs"
  >
    <div
      id="seek-bar-base"
      ref="seekBarBase"
    >
      <div
        id="seek-bar-played"
        ref="seekBarPlayed"
      >
        <div
          id="seek-bar-handle"
          ref="seekBarHandle"
          v-on:mousedown.stop="$_seekStart"
        >
        </div>
      </div>

      <div
        id="seek-bar-loop"
        class="loop"
        ref="seekBarLoop"
        v-show="isLoopEnabled"
      >
      </div>

      <div
        id="seek-bar-clickable-area"
        ref="seekBarClickableArea"
        v-on:mousedown.stop="$_seekStart"
      >
      </div>
    </div>
  </div>
</template>

<style scoped>
div#seek-bar-container {
  flex-grow: 1;
  display: flex;
  flex-direction: column;
  padding: 0 10px;
  user-select: none;
}

div#seek-bar-base {
  position: relative;
  flex-grow: 1;
  height: 3px;
  background-color: #dddddd;
}

div#seek-bar-played {
  height: 100%;
  position: absolute;
  top: 0;
  left: 0;
  background-color: #666666;
  display: flex;
  justify-content: flex-end;
  align-items: center;
}

div#seek-bar-handle {
  flex-shrink: 0;
  background-color: #666666;
  height: 15px;
  width: 15px;
  margin-right: -8px;
  border-radius: 50%;
}

div#seek-bar-loop {
  position: absolute;
  top: 0;
  height: 100%;
  opacity: 0.5;
}

div#seek-bar-clickable-area {
  position: relative;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  margin-top: -10px;
  padding: 10px 0;
}

div.loop {
  background-color: #16f43f;
}
</style>

<script lang="ts">
import { defineComponent } from 'vue';
import AudioPlaybackLoopDefinition from './modules/AudioPlaybackLoopDefinition';

function clamp(x: number, min: number, max: number): number {
  x = (x > max)? max : x;
  x = (x < min)? min : x;
  return x;
}

export default defineComponent({
  emits: {
    seek: (seekTimeSec: number) => true,
    seekStart: () => true,
    seekEnd: () => true,
  },

  watch: {
    currentTimeSec(newCurrentTimeSec: number) {
      this.$_updateSeekBarPosition(newCurrentTimeSec, this.$data.$_seekBarScale);
    },

    durationSec(newDurationSec: number) {
      this.$data.$_seekBarScale = newDurationSec / this.$data.$_seekBarBaseWidthPx;
      this.$_updateSeekBarPosition(this.currentTimeSec, this.$data.$_seekBarScale);
    },

    loopDefinition(loopDefinition: AudioPlaybackLoopDefinition) {
      this.$_updateLoopRange(loopDefinition, this.$data.$_seekBarScale);
    },

    '$data.$_seekBarBaseWidthPx'(seekBarBaseWidthPx: number) {
      this.$data.$_seekBarScale = this.durationSec / seekBarBaseWidthPx;
    },

    '$data.$_seekBarScale'(seekBarScale: number) {
      this.$_updateSeekBarPosition(this.currentTimeSec, seekBarScale);
      this.$_updateLoopRange(this.loopDefinition, seekBarScale);
    },
  },

  mounted() {
    this.$data.$_seekBarBaseResizeObserver = new ResizeObserver(
      resizeObserverEntries => {
        let resizeObserverEntry = resizeObserverEntries[0];
        this.$data.$_seekBarBaseWidthPx = resizeObserverEntry.contentRect.width;
      },
    );
    this.$data.$_seekBarBaseResizeObserver.observe(this.$refs.seekBarBase as HTMLElement);
    this.$data.$_seekBarBaseWidthPx = (this.$refs.seekBarBase as HTMLElement).clientWidth;
  },

  unmounted() {
    this.$data.$_seekBarBaseResizeObserver?.disconnect();
  },

  props: {
    currentTimeSec: { type: Number, required: true },
    durationSec: { type: Number, required: true },
    isLoopEnabled: { type: Boolean, required: true },
    isSeeking: { type: Boolean, required: true },
    loopDefinition: { type: AudioPlaybackLoopDefinition, required: true },
  },

  data():{
    $_seekBarBaseWidthPx: number,
    $_seekBarBaseResizeObserver: ResizeObserver | undefined,
    $_seekBarScale: number,
    $_isSeekingInternal: boolean,
  } {
    return {
      $_seekBarBaseWidthPx: 0,
      $_seekBarBaseResizeObserver: undefined,
      $_seekBarScale: 1,
      $_isSeekingInternal: false,
    };
  },

  methods: {
    $_updateSeekBarPosition(currentTimeSec: number, seekBarScale: number) {
      (this.$refs.seekBarPlayed as InstanceType<typeof HTMLElement>).style.width = String(currentTimeSec / seekBarScale) + 'px';
    }, 

    $_updateLoopRange(loopDefinition: AudioPlaybackLoopDefinition, seekBarScale: number) {
      (this.$refs.seekBarLoop as InstanceType<typeof HTMLElement>).style.left = String(loopDefinition.beginTimeSec / seekBarScale) + 'px';
      (this.$refs.seekBarLoop as InstanceType<typeof HTMLElement>).style.width = String(loopDefinition.loopDurationSec / seekBarScale) + 'px';
    },

    $_getSeekTimeSecByMouseEvent(mouseEvent: MouseEvent) {
      let seekBarBaseClientRect = (this.$refs.seekBarBase as InstanceType<typeof HTMLElement>).getBoundingClientRect();
      let seekBarOffsetPx = mouseEvent.clientX - seekBarBaseClientRect.x;
      return seekBarOffsetPx * this.$data.$_seekBarScale;
    },

    $_seek(mouseEvent: MouseEvent) {
      if (!this.isSeeking) return;
      let seekTimeSec = this.$_getSeekTimeSecByMouseEvent(mouseEvent);
      seekTimeSec = clamp(seekTimeSec, 0, this.durationSec);
      this.$emit('seek', seekTimeSec);
    },

    $_seekStart(mouseEvent: MouseEvent) {
      if (this.isSeeking) return;
      this.$data.$_isSeekingInternal = true;
      this.$emit('seekStart');
      this.$_seek(mouseEvent);
    },

    $_seekEnd(mouseEvent: MouseEvent) {
      this.$_seek(mouseEvent);
      this.$emit('seekEnd');
      this.$data.$_isSeekingInternal = false;
    },

    onMousemove(mouseEvent: MouseEvent) {
      if (this.$data.$_isSeekingInternal) this.$_seek(mouseEvent);
    },

    onMouseup(mouseEvent: MouseEvent) {
      if (this.$data.$_isSeekingInternal) this.$_seekEnd(mouseEvent);
    },
  },
});
</script>