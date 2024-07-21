<template>
    <div>
        <div v-if="!isStarted">
            <button @click="startAudio" :disabled="isStarted">Start Audio</button>
        </div>
        <div v-if="isStarted">
            <div class="chord-grid-container">
                <div class="chord-grid">
                    <button v-for="(chord, index) in chords" :key="index" @click="selectChord(index)"
                        class="chord-button">
                    </button>
                </div>
            </div>
            <div class="visualization">
                <svg :width="svgWidth" :height="svgHeight">
                    <polygon :points="interpolatedPolygonPoints" fill="none" stroke="blue" stroke-width="2" />
                    <circle v-for="(point, index) in interpolatedPoints" :key="`circle-${index}`" :cx="point.x"
                        :cy="point.y" r="4" fill="red" />
                </svg>
            </div>
        </div>
        <div ref="patcherContainer"></div>
    </div>
</template>

<script>
import { ref, computed } from 'vue';
import { createDevice, MessageEvent, TimeNow } from '@rnbo/js';
import gsap from 'gsap';

export default {
    name: 'RNBOPatcher',
    setup() {
        const patcherContainer = ref(null);
        const isStarted = ref(false);
        let device;
        let context;

        const chords = [
            [51, 57, 58, 62, 65, 70, 74],
            [50, 57, 62, 63, 69, 70],
            [45, 57, 74, 82],
            [39, 51, 65, 77, 86],
            [48, 55, 60, 67, 72],
            [46, 58, 62, 65, 70, 74],
            [48, 55, 60, 67, 72],
            [41, 53, 60, 65, 69, 74],
            [41, 53, 60, 65, 69, 74],
            [49, 56, 61, 68, 73],
            [43, 55, 59, 62, 67, 71],
            [52, 59, 64, 68, 71, 76]
        ];

        const currentChord = ref([]);
        const previousShapePoints = ref([]);
        const transition = ref(1);
        const svgWidth = 400;
        const svgHeight = 400;
        const centerX = svgWidth / 2;
        const centerY = svgHeight / 2;

        const lowestNote = computed(() => Math.min(...chords.flat()));
        const highestNote = computed(() => Math.max(...chords.flat()));

        const calculatePoints = (chord) => {
            if (chord.length === 0) return [];

            const angleStep = (2 * Math.PI) / chord.length;
            const radius = Math.min(svgWidth, svgHeight) / 2 - 50;

            return chord.map((note, index) => {
                const angle = index * angleStep - Math.PI / 2; // Start from top
                const distanceFromCenter = ((note - lowestNote.value) / (highestNote.value - lowestNote.value)) * radius;
                const x = centerX + distanceFromCenter * Math.cos(angle);
                const y = centerY + distanceFromCenter * Math.sin(angle);
                return { x, y };
            });
        };

        const shapePoints = computed(() => calculatePoints(currentChord.value));

        const interpolatedPoints = computed(() => {
            if (previousShapePoints.value.length === 0) return shapePoints.value;

            const progress = transition.value;
            return shapePoints.value.map((point, index) => {
                const prevPoint = previousShapePoints.value[index] || point;
                return {
                    x: prevPoint.x + (point.x - prevPoint.x) * progress,
                    y: prevPoint.y + (point.y - prevPoint.y) * progress
                };
            });
        });

        const interpolatedPolygonPoints = computed(() => interpolatedPoints.value.map(p => `${p.x},${p.y}`).join(' '));

        const startAudio = async () => {
            if (isStarted.value) return;
            try {
                const WAContext = window.AudioContext || window.webkitAudioContext;
                context = new WAContext();
                const response = await fetch('/patch.export.json');
                const patcher = await response.json();
                device = await createDevice({ context, patcher });
                device.node.connect(context.destination);
                isStarted.value = true;
                console.log('Audio started and RNBO patch loaded');
            } catch (error) {
                console.error('Error starting audio and loading RNBO patcher:', error);
            }
        };

        const selectChord = (chordIndex) => {
            if (device && isStarted.value) {
                previousShapePoints.value = [...shapePoints.value];
                const chord = chords[chordIndex];
                currentChord.value = chord;
                transition.value = 0;
                gsap.to(transition, {
                    value: 1,
                    duration: 1,
                    ease: 'power2.inOut'
                });
                device.scheduleEvent(new MessageEvent(TimeNow, 'selectchord', [chordIndex]));
                device.scheduleEvent(new MessageEvent(TimeNow, 'chord', chord));
                console.log(`Sent chord selection: ${chordIndex}, Chord: ${chord}`);
            }
        };

        return {
            patcherContainer,
            startAudio,
            isStarted,
            selectChord,
            chords,
            svgWidth,
            svgHeight,
            interpolatedPoints,
            interpolatedPolygonPoints,
        };
    }
}
</script>

<style scoped>
.chord-grid-container {
    display: flex;
    justify-content: center;
    align-items: center;
}

.chord-grid {
    display: grid;
    grid-template-columns: repeat(4, 1fr);
    gap: 10px;
    margin-top: 10px;
    margin-bottom: 20px;
    max-width: 300px;
}

.chord-button {
    width: 60px;
    height: 60px;
    border: 2px solid #007bff;
    background-color: #f8f9fa;
    border-radius: 5px;
    cursor: pointer;
    transition: background-color 0.3s;
}

.chord-button:hover {
    background-color: #007bff;
}

.visualization {
    margin-top: 20px;
    display: flex;
    justify-content: center;
}
</style>