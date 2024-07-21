<template>
    <div>
        <div v-if=!isStarted>
            <button @click="startAudio" :disabled="isStarted">Start Audio</button>
        </div>
      <div v-if="isStarted">
        <button v-for="n in 4" :key="n-1" @click="selectChord(n-1)">
          Chord {{ n-1 }}
        </button>
      </div>
      <div ref="patcherContainer"></div>
    </div>
  </template>
  
  <script>
  import { ref, onMounted } from 'vue';
  import { createDevice, MessageEvent, TimeNow } from '@rnbo/js';
  
  export default {
    name: 'RNBOPatcher',
    setup() {
      const patcherContainer = ref(null);
      const isStarted = ref(false);
      let device;
      let context;
  
      const startAudio = async () => {
        if (isStarted.value) return;
  
        try {
          // Create an AudioContext
          const WAContext = window.AudioContext || window.webkitAudioContext;
          context = new WAContext();
  
          // Load your RNBO patch
          const response = await fetch('/patch.export.json');
          const patcher = await response.json();
  
          // Create the RNBO device
          device = await createDevice({ context, patcher });
  
          // Connect the device to the audio output
          device.node.connect(context.destination);
  
          isStarted.value = true;
          console.log('Audio started and RNBO patch loaded');
        } catch (error) {
          console.error('Error starting audio and loading RNBO patcher:', error);
        }
      };
  
      const selectChord = (chordNumber) => {
        if (device && isStarted.value) {
            console.log(chordNumber)
          device.scheduleEvent(new MessageEvent(TimeNow, 'selectchord', [chordNumber]));
          console.log(`Sent chord selection: ${chordNumber}`);
        }
      };
  
      onMounted(() => {
        // Any setup that doesn't require user interaction can go here
      });
  
      return {
        patcherContainer,
        startAudio,
        isStarted,
        selectChord
      };
    }
  }
  </script>