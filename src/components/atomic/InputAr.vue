<template>
	<div class="input-ar input-box" :class="{ focus }">
		<div class="input">
			<Icon :icon="require('@/assets/logos/arweave.svg')" />
			<input v-model="model" inputmode="numeric" class="text" placeholder="AR" @focus="focus = 1" @blur="focus = 0" :disabled="$attrs.disabled">
		</div>
		<div v-if="currentPrice" class="spacer"></div>
		<div v-if="currentPrice" class="input">
			<input v-model="model2" inputmode="numeric" class="text right" :placeholder="currency" @focus="focus = 2" @blur="focus = 0" :disabled="$attrs.disabled">
			<Icon :icon="currencySymbol" />
		</div>
	</div>
</template>



<script>
import Icon from '@/components/atomic/Icon.vue'
import ArweaveStore from '@/store/ArweaveStore'
import { computed, ref, watch } from 'vue'

export default {
	components: { Icon },
	props: ['modelValue'],
	setup (props, { emit }) {
		const model = computed({
			get () { 
				const value = props.modelValue
				if (focus.value === 0) {
					input2.value = value && !isNaN(value) ? +(value * currentPrice.value).toFixed(2) : ''
				}
				return value
			},
			set (value) {
				if (focus.value === 1 || focus.value === 0) {
					input2.value = value && !isNaN(value) ? +(value * currentPrice.value).toFixed(2) : ''
					emit('update:modelValue', value)
				}
			}
		})
		const input2 = ref('')
		const model2 = computed({
			get () { return input2.value },
			set (value) {
				if (focus.value === 2) {
					input2.value = value
					emit('update:modelValue', value && !isNaN(value) ? value / currentPrice.value : '')
				}
			}
		})
		const currentPrice = computed(() => ArweaveStore.redstone.currentPrice)
		const currency = computed(() => ArweaveStore.redstone.currency)
		const currencySymbol = computed(() => new Intl.NumberFormat(navigator.languages, { style: 'currency', currency: currency.value }).format(0).replace(/[\w\d\.\,\s]/g, '') || '$')
		const focus = ref(0)
		watch(() => model.value, (newVal, oldVal) => {
			if (focus.value === 1 && !newVal.match(/^(?:\d*\.?\d*)?$/)) { model.value = oldVal }
		})
		watch(() => model2.value, (newVal, oldVal) => {
			if (focus.value === 2 && !newVal.match(/^(?:\d*\.?\d*)?$/)) { model2.value = oldVal }
		})
		return { model, model2, currentPrice, currency, currencySymbol, focus }
	}
}
</script>



<style scoped>
.input-ar {
	--height: 3.5em;
	height: var(--height);
	display: flex;
	align-items: center;
	transition: 0.3s ease;
}

.vertical.input-ar {
	--height: 7em;
	flex-direction: column;
}

.input {
	width: 100%;
	height: inherit;
	flex: 1 1 0;
	display: flex;
	align-items: center;
	justify-content: center;
	/* background: var(--background3); */
	border-radius: inherit;
}

.vertical .input {
	height: calc(var(--height) / 2);
}

.spacer {
	width: 1px;
	height: 1.2em;
	background: #ffffff18;
	transition: 0.3s ease;
}

.vertical .spacer {
	display: none;
}

.focus .spacer {
	background: #ffffff60;
}

.icon {
	font-size: 1.4em;
	width: 2em;
	opacity: var(--element-secondary-opacity);
}

.focus .icon {
	opacity: 1;
}

.text {
	height: inherit;
	font-size: 1em;
	padding: 0 var(--spacing);
	outline: none;
	border: none;
	flex: 1 1 auto;
	background-color: transparent;
	color: var(--element-secondary);
	width: 100%;
}

.text.right {
	text-align: end;
}
</style>