<template>
	<div class="send card">
		<!-- Autocomplete local addrs -->
		<h2 class="heading"><img class="img" src="@/assets/icons/north_east.svg">Send</h2>
		<div class="row">
			<Input v-model.trim="InterfaceStore.wallet.send.target" :icon="require('@/assets/icons/person.svg')" placeholder="Address" autocomplete="ar" :mask="maskAddress" />
			<AddressIcon class="address-icon" :address="InterfaceStore.wallet.send.target" />
		</div>
		<div class="row" style="justify-content:flex-end;"></div>

		<h3 class="heading">Amount</h3>
		<InputAr v-model="InterfaceStore.wallet.send.quantity" />
		<div class="row" style="justify-content:flex-end;">
			<button class="secondary" style="padding-top:1em;" @click="setMax">Max</button>
		</div>

		<h3 class="heading">Data</h3>
		<InputData v-model="InterfaceStore.wallet.send.data" @files="(files) => InterfaceStore.wallet.send.data = files ? files[0] : ''" />
		<div class="row" style="justify-content:flex-end;"></div>

		<div class="row">
			<h3 class="heading" style="display:block;">Tags</h3>
			<div v-if="!InterfaceStore.wallet.send.tags.length"><button class="secondary" style="padding-top:1em;" @click="addTag()">Add</button></div>
		</div>
		<InputGrid :schema="InterfaceStore.wallet.send.tags" :deletable="true" />
		<div v-if="InterfaceStore.wallet.send.tags.length" class="row" style="justify-content:flex-end;">
			<button class="secondary" style="padding-top:1em;" @click="addTag()">Add</button>
		</div>

		<div class="row" style="align-items:flex-end; margin-top:3em;">
			<div>
				<div>Size {{ txSizeDisplay }}</div>
				<div>Fee
					<Ar class="ar" :ar="txFee" />&nbsp;<LocaleCurrency class="small secondary" :ar="txFee">|</LocaleCurrency>
				</div>
			</div>
			<Button @click="postTx" :style="submitStyle" :disabled="loading" :icon="require('@/assets/icons/north_east.svg')">
				Submit
			</Button>
		</div>
		<!-- QR -->
	</div>
</template>



<script>
import Input from '@/components/atomic/Input.vue'
import InputAr from '@/components/atomic/InputAr.vue'
import InputData from '@/components/atomic/InputData.vue'
import InputGrid from '@/components/atomic/InputGrid.vue'
import AddressIcon from '@/components/atomic/AddressIcon.vue'
import Ar from '@/components/atomic/Ar.vue'
import LocaleCurrency from '@/components/atomic/LocaleCurrency.vue'
import Button from '@/components/atomic/Button.vue'
import Icon from '@/components/atomic/Icon.vue'
import ArweaveStore, { arweave } from '@/store/ArweaveStore'
import InterfaceStore from '@/store/InterfaceStore'
import { buildTransaction, manageUpload } from '@/functions/Transactions'
import Ledger from '@/functions/Ledger'
import axios from 'axios'
import { debounce, humanFileSize, addressToColor } from '@/functions/Utils'
import { computed, ref, watch } from 'vue'

export default {
	components: { Input, InputAr, InputData, InputGrid, AddressIcon, Ar, LocaleCurrency, Button, Icon },
	props: ['wallet'],
	setup (props) {
		const maskAddress = (address) => { return address.match(/^[a-z0-9_-]{0,43}$/i) }
		const setMax = () => InterfaceStore.wallet.send.quantity = props.wallet.balance
		watch(() => InterfaceStore.wallet.send.data, (value) => {
			let contentTypeTag = InterfaceStore.wallet.send.tags.find(row =>
				row.items[0].value === 'Content-Type')
			if (typeof value === 'object') {
				if (!contentTypeTag) {
					contentTypeTag = tagSchema('Content-Type')
					addTag(contentTypeTag)
				}
				contentTypeTag.items[1].value = value.type
			} else if (contentTypeTag) {
				const index = InterfaceStore.wallet.send.tags.indexOf(contentTypeTag)
				InterfaceStore.wallet.send.tags.splice(index, 1)
			}
		})
		const tagSchema = (name, value) => ({
			items: [
				{ name: 'Tag', value: name || '', icon: require('@/assets/icons/label.svg') },
				{ name: 'Value', value: value || '' }
			], deletable: true, key: Math.random()
		})
		const addTag = (tag) => InterfaceStore.wallet.send.tags.push(tag || tagSchema())
		const txSize = computed(() => {
			const data = InterfaceStore.wallet.send.data
			return data.size || data.length || '0'
		})
		const txSizeDisplay = computed(() => humanFileSize(txSize.value))
		const txFee = ref(null)
		const feeUrl = computed(() => {
			const address = InterfaceStore.wallet.send.target
			return ArweaveStore.gatewayURL + 'price/' + txSize.value + '/' + (address.match(/^[a-z0-9_-]{43}$/i) ? address : '')
		})
		const updateFee = async () => { txFee.value = arweave.ar.winstonToAr((await axios.get(feeUrl.value)).data) }
		const updateFeeDebounced = debounce(updateFee)
		updateFee()
		watch(() => feeUrl.value, () => updateFeeDebounced())
		const getTagsFromSchema = (tagsSchema) => {  // TODO validate tags length
			const result = []
			for (const row of tagsSchema) { result.push({ name: row.items[0].value, value: row.items[1].value }) }
			return result
		}
		const loading = ref(false)
		const postTx = async () => {
			if (loading.value) { return }
			loading.value = true
			const settings = InterfaceStore.wallet.send
			const tx = await buildTransaction(settings.target, settings.quantity, getTagsFromSchema(settings.tags), settings.data)
			if (props.wallet.jwk) { await arweave.transactions.sign(tx, props.wallet.jwk) }
			else if (props.wallet.metaData.provider === 'Ledger') {
				if (props.wallet.key !== await Ledger.getActiveAddress()) { alert('Wrong account'); return }
				await Ledger.sign(tx)
			}
			manageUpload(tx)
			InterfaceStore.wallet.send = { target: '', quantity: '', data: '', tags: [], }
			loading.value = false
			// TODO visual feedback and clear interface values
		}
		const submitStyle = {
			'--border': `rgba(${addressToColor(props.wallet.key).join(',')},0.8)`,
			'--glow-color': `rgba(${addressToColor(props.wallet.key).join(',')},0.2)`,
			'background-image': `radial-gradient(circle at center, rgba(${addressToColor(props.wallet.key).join(',')},0.4), 
			rgba(${addressToColor(props.wallet.key).join(',')},0.3))`
		}
		return { InterfaceStore, maskAddress, setMax, addTag, txSizeDisplay, txFee, postTx, submitStyle, loading }
	}
}
</script>



<style scoped>
.send {
	display: flex;
	flex-direction: column;
}

.input {
	flex: 1 1 0;
}

.address-icon {
	flex: 0 0 auto;
	width: 3.5em;
	height: 3.5em;
	padding: 0;
}

.heading {
	display: flex;
	align-items: center;
	gap: var(--spacing);
}

.row {
	min-height: 2em;
	display: flex;
	align-items: center;
	justify-content: space-between;
	gap: var(--spacing);
}

.img {
	height: 1em;
}

.secondary {
	color: var(--element-secondary);
}

.small {
	font-size: 0.75em;
}

.button {
	width: 50%;
}
</style>