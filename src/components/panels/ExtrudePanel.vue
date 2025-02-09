<style scoped>
.v-btn-toggle {
	display: flex;
}
.v-btn-toggle > button {
	display: flex;
	flex: 1 1 auto;
}
</style>

<template>
	<v-card>
		<v-card-title class="pb-0">
			<v-icon small class="mr-1">opacity</v-icon> {{ $t('panel.extrude.caption') }}
		</v-card-title>

		<v-layout row class="px-3 py-1" align-center>
			<v-flex>
				<v-layout row wrap align-center>
					<v-flex v-if="currentTool && currentTool.extruders.length > 1" class="ma-1">
						<p class="mb-1">
							{{ $t('panel.extrude.mixRatio') }}
						</p>
						<v-btn-toggle v-model="mix" mandatory multiple>
							<v-btn flat value="mix" :disabled="uiFrozen" color="primary">
								{{ $t('panel.extrude.mix') }}
							</v-btn>
							<v-btn flat v-for="extruder in currentTool.extruders" :key="extruder" :value="extruder" :disabled="uiFrozen" color="primary">
								{{ `E${extruder}` }}
							</v-btn>
						</v-btn-toggle>
					</v-flex>
					<v-flex class="ma-1">
						<p class="mb-1">
							{{ $t('panel.extrude.amount', ['mm']) }}
						</p>
						<v-btn-toggle v-model="amount" mandatory>
							<v-btn flat v-for="(amount, index) in extruderAmounts" :key="index" :value="amount" :disabled="uiFrozen" color="primary" @contextmenu.prevent="editAmount(index)">
								{{ amount }}
							</v-btn>
						</v-btn-toggle>
					</v-flex>
					<v-flex class="ma-1">
						<p class="mb-1">
							{{ $t('panel.extrude.feedrate', ['mm/s']) }}
						</p>
						<v-btn-toggle v-model="feedrate" mandatory>
							<v-btn flat v-for="(feedrate, index) in extruderFeedrates" :key="index" :value="feedrate" :disabled="uiFrozen" color="primary" @contextmenu.prevent="editFeedrate(index)">
								{{ feedrate }}
							</v-btn>
						</v-btn-toggle>
					</v-flex>
				</v-layout>
			</v-flex>
			<v-flex shrink class="ml-2 mb-1">
				<v-btn block :disabled="uiFrozen || !canRetract" :loading="busy" @click="buttonClicked(false)">
					<v-icon>arrow_upward</v-icon> {{ $t('panel.extrude.retract') }}
				</v-btn>
				<v-btn block :disabled="uiFrozen || !canExtrude" :loading="busy" @click="buttonClicked(true)">
					<v-icon>arrow_downward</v-icon> {{ $t('panel.extrude.extrude') }}
				</v-btn>
			</v-flex>
		</v-layout>

		<input-dialog :shown.sync="editAmountDialog.shown" :title="$t('dialog.editExtrusionAmount.title')" :prompt="$t('dialog.editExtrusionAmount.prompt')" :preset="editAmountDialog.preset" is-numeric-value @confirmed="setAmount"></input-dialog>
		<input-dialog :shown.sync="editFeedrateDialog.shown" :title="$t('dialog.editExtrusionFeedrate.title')" :prompt="$t('dialog.editExtrusionFeedrate.prompt')" :preset="editFeedrateDialog.preset" is-numeric-value @confirmed="setFeedrate"></input-dialog>
	</v-card>
</template>

<script>
'use strict'

import { mapState, mapGetters, mapActions, mapMutations } from 'vuex'

export default {
	computed: {
		...mapGetters(['uiFrozen']),
		...mapState('machine/model', ['heat', 'tools']),
		...mapGetters('machine/model', ['currentTool']),
		...mapState('machine/settings', ['extruderAmounts', 'extruderFeedrates']),
		canExtrude() {
			if (this.currentTool && this.currentTool.heaters.length) {
				const selectedHeaters = (this.mixValue[0] === 'mix') ? this.currentTool.heaters : this.mixValue;
				const heaters = this.heat.heaters, minTemp = this.heat.coldExtrudeTemperature;
				return !selectedHeaters.some(heater => heaters[heater].current < minTemp);
			}
			return false;
		},
		canRetract() {
			if (this.currentTool && this.currentTool.heaters.length) {
				const selectedHeaters = (this.mixValue[0] === 'mix') ? this.currentTool.heaters : this.mixValue;
				const heaters = this.heat.heaters, minTemp = this.heat.coldRetractTemperature;
				return !selectedHeaters.some(heater => heaters[heater].current < minTemp);
			}
			return false;
		},
		mix: {
			get() {
				return this.mixValue;
			},
			set(value) {
				if (value.length > 1) {
					if (this.mixValue.indexOf('mix') !== value.indexOf('mix')) {
						// Mix is being toggled
						if (value.indexOf('mix') !== -1) {
							this.mixValue = ['mix'];
						} else {
							this.mixValue = value.filter(item => item !== 'mix');
						}
					} else {
						// Selecting another E drive
						this.mixValue = value.filter(item => item !== 'mix');
					}
				} else {
					// One value - OK
					this.mixValue = value;
				}
			}
		}
	},
	data() {
		return {
			busy: false,
			mixValue: ['mix'],
			amount: 10,
			feedrate: 5,
			editAmountDialog: {
				shown: false,
				index: 0,
				preset: 0
			},
			editFeedrateDialog: {
				shown: false,
				index: 0,
				preset: 0
			}
		}
	},
	methods: {
		...mapActions('machine', ['sendCode']),
		...mapMutations('machine/settings', ['setExtrusionAmount', 'setExtrusionFeedrate']),
		async buttonClicked(extrude) {
			if (!this.currentTool.extruders.length) {
				return;
			}

			let amounts;
			if (this.mix.length === 1 && this.mix[0] === 'mix') {
				// Split total amount to extrude evenly
				amounts = [this.amount];
			} else {
				// Extrude given amount via each selected extruder drive
				amounts = this.currentTool.extruders.map(extruder => (this.mix.indexOf(extruder) !== -1) ? this.amount : 0);
			}

			this.busy = true;
			try {
				const amount = amounts.map(amount => extrude ? amount : -amount).reduce((a, b) => `${a}:${b}`);
				await this.sendCode(`M120\nM83\nG1 E${amount} F${this.feedrate * 60}\nM121`);
			} catch (e) {
				// handled before we get here
			}
			this.busy = false;
		},
		editAmount(index) {
			this.editAmountDialog.index = index;
			this.editAmountDialog.preset = this.extruderAmounts[index];
			this.editAmountDialog.shown = true;
		},
		setAmount(value) {
			this.setExtrusionAmount({ index: this.editAmountDialog.index, value });
			this.amount = value;
		},
		editFeedrate(index) {
			this.editFeedrateDialog.index = index;
			this.editFeedrateDialog.preset = this.extruderFeedrates[index];
			this.editFeedrateDialog.shown = true;
		},
		setFeedrate(value) {
			this.setExtrusionFeedrate({ index: this.editFeedrateDialog.index, value });
			this.feedrate = value;
		}
	},
	mounted() {
		this.amount = this.extruderAmounts[3];
		this.feedrate = this.extruderFeedrates[3];
	},
	watch: {
		currentTool(to) {
			if (!to || to.extruders.length <= 1) {
				// Switch back to mixing mode if the selection panel is hidden
				this.mix = ['mix'];
			}
		},
		extruderAmounts() {
			this.amount = this.extruderAmounts[0];
		},
		extruderFeedrates() {
			this.feedrate = this.extruderFeedrates[0];
		}
	}
}
</script>
