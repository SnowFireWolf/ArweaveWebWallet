<template>
	<div class="icon">
		<transition name="slide-up" mode="out-in">
			<div v-if="icon" :key="icon" class="icon-background">
				<svg v-if="icon == 'loader'" class="loader" height="100" width="100" viewBox="0 0 100 100" :class="{ spin: progress == null }">
					<circle stroke="#ffffff33" :stroke-width="thickness" fill="transparent" :r="normalizedRadius" :cx="50" :cy="50" />
					<circle stroke="white" :stroke-dasharray="circumference + ' ' + circumference" :style="{ strokeDashoffset }" :stroke-width="thickness" fill="transparent" :r="normalizedRadius" :cx="50" :cy="50" @animationiteration="finishAnimation = false"  :class="{ spin: progress == null || finishAnimation }" />
				</svg>
				<span v-else-if="isSymbol" class="symbol no-select">{{ icon }}</span>
				<img v-else class="img no-select" :src="icon" draggable="false">
			</div>
		</transition>
	</div>
</template>



<script>
import { computed, ref, watch } from 'vue'
export default {
	props: ['icon', 'progress'],
	setup (props) {
		const isSymbol = computed(() => typeof props.icon === 'string' && props.icon.length <= 2)
		const thickness = 10
		const normalizedRadius = 50 - thickness / 2
		const circumference = normalizedRadius * 2 * Math.PI
		const strokeDashoffset = computed(() => circumference - (props.progress || 25) / 100 * circumference)
		const finishAnimation = ref(false)
		watch(() => props.progress, (value, oldValue) => {
			if (value != null && oldValue == null) { finishAnimation.value = true }
		})
		return { isSymbol, thickness, normalizedRadius, circumference, strokeDashoffset, finishAnimation }
	},
}
</script>



<style scoped>
.icon {
	flex: 0 0 auto;
	height: 1em;
	width: 1em;
	border-radius: inherit;
	/* display: flex;
	align-items: center;
	justify-content: center; */
	transition: 0.3s ease;
}

.icon-background {
	width: 100%;
	height: 100%;
	/* background: var(--background2); */
	border-radius: inherit;
	display: flex;
	align-items: center;
	justify-content: center;
}

.img {
	width: 1em;
	height: 1em;
	object-fit: contain;
}

.symbol {
	font-size: 1em;
}

.loader {
	transform: rotate(-90deg);
	transition: transform 0.5s ease;
}

.loader.spin {
	transform: rotate(-135deg);
}

circle.spin {
	animation: loader-animation 2s infinite linear;
	animation-timing-function: cubic-bezier(0.6, 0.4, 0.4, 0.6);
}

@keyframes loader-animation {
	0% {
		transform: rotate(0deg);
	}
	100% {
		transform: rotate(360deg);
	}
}

svg {
	width: 100%;
	height: 100%;
}

circle {
	transition: stroke-dashoffset 0.5s ease;
	transform-origin: 50% 50%;
}
</style>