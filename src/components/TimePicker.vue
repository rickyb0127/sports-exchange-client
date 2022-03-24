<template>
	<div class="timepicker-wrapper">
		<div class="md-layout">
			<div class="md-layout-item md-size-50">
				<div class="md-layout">
					<input ref="hourInput" class="md-layout-item md-size-30" type="text" v-model="selectedHourInput" @blur="validateHour" @click="selectAllHourInput()" maxlength="2" /> : <input ref="minuteInput" class="md-layout-item md-size-30" type="text" v-model="selectedMinuteInput" @blur="validateMinute" @click="selectAllMinuteInput()" maxlength="2" />
				</div>
			</div>
			<div class="md-layout-item md-size-50">
				<div class="md-layout">
					<button @click="selectAmp('AM')" class="md-layout-item md-size-40 amp-button" :class="[{'selected': selectedAmp === 'AM'}]">AM</button><button @click="selectAmp('PM')" :class="[{'selected': selectedAmp === 'PM'}]" class="md-layout-item md-size-40 amp-button">PM</button>
				</div>
			</div>
		</div>
	</div>
</template>

<script>
export default {
	name: 'TimePicker',
	data() {
		return {
			attrs: {
				currentSelectedHour: this.selectedHour,
				currentSelectedMinute: this.selectedMinute
			},
			selectedHourInput: null,
			selectedMinuteInput: null,
			hours: [],
			minutes: [],
			amps: ["AM", "PM"],
			selectedAmp: null
		}
	},
	computed: {
		currentSelectedHour: {
			get() {
				return this.attrs.currentSelectedHour;
			},
			set(value) {
				this.attrs.currentSelectedHour = value;
				this.$emit(`update:selected-hour`, value);
			}
		},
		currentSelectedMinute: {
			get() {
				return this.attrs.currentSelectedMinute;
			},
			set(value) {
				this.attrs.currentSelectedMinute = value;
				this.$emit(`update:selected-minute`, value);
			}
		}
	},
	props: {
		selectedHour: Number,
		selectedMinute: Number
	},
	methods: {
		parseHourMinutesToString() {
			this.selectedMinuteInput = this.currentSelectedMinute.toString(10);
			if(this.currentSelectedHour < 12) {
				this.selectedAmp = "AM";
				this.selectedHourInput = this.currentSelectedHour === 0 ? "12" : this.currentSelectedHour.toString(10);
			} else {
				this.selectedAmp = "PM";
				this.selectedHourInput = this.currentSelectedHour === 12 ? "12" : (this.currentSelectedHour - 12).toString(10);
			}

			if(this.selectedHourInput.length === 1) {
				this.selectedHourInput = `0${this.selectedHourInput}`;
			}
			if(this.selectedMinuteInput.length === 1) {
				this.selectedMinuteInput = `0${this.selectedMinuteInput}`;
			}
		},
		selectHour() {
			let hourString = this.selectedHourInput;
			let hourInt;
			if(this.selectedHourInput.charAt(0) === "0") {
				hourString = this.selectedHourInput.substring(1);
			}
			if(this.selectedAmp === "AM" && hourString === "12") {
				hourString = "0";
			}
			hourInt = parseInt(hourString);
			if(this.selectedAmp === "PM" && hourString !== "12") {
				hourInt += 12;
			}
			this.currentSelectedHour = hourInt;
		},
		selectMinute() {
			let minuteString = this.selectedMinuteInput;
			let minuteInt;
			if(this.selectedMinuteInput.charAt(0) === "0") {
				minuteString = this.selectedMinuteInput.substring(1);
			}
			minuteInt = parseInt(minuteString);
			this.currentSelectedMinute = minuteInt;
		},
		selectAmp(amp) {
			this.selectedAmp = amp;
			if(this.currentSelectedHour === 0 || this.currentSelectedHour === 12) {
				this.selectedHourInput = "12";
			} else {
				if(this.currentSelectedHour < 12) {
					this.selectedHourInput = this.currentSelectedHour.toString(10);
				} else {
					this.selectedHourInput = (this.currentSelectedHour - 12).toString(10);
				}
			}
			this.validateHour();
		},
		validateHour() {
			if(this.selectedHourInput.length === 1) {
				this.selectedHourInput = `0${this.selectedHourInput}`;
			}
			if(!this.hours.includes(this.selectedHourInput)) {
				this.selectedHourInput = "01";
			}
			this.selectHour();
		},
		validateMinute() {
			if(this.selectedMinuteInput.length === 1) {
				this.selectedMinuteInput = `0${this.selectedMinuteInput}`;
			}
			if(!this.minutes.includes(this.selectedMinuteInput)) {
				this.selectedMinuteInput = "00";
			}
			this.selectMinute();
		},
		selectAllHourInput() {
			const input = this.$refs.hourInput;
			input.focus();
			input.select();
		},
		selectAllMinuteInput() {
			const input = this.$refs.minuteInput;
			input.focus();
			input.select();
		}
	},
	created() {
		const hoursList = Array.from({length: 12}, (_, i) => i + 1);
		this.hours = hoursList.map((hour) => {
			let stringVal = hour.toString(10);
			if(stringVal.length === 1) {
				stringVal = `0${stringVal}`;
			}
			return stringVal;
		});

		const minutesList = [...Array(60).keys()];
		this.minutes = minutesList.map((minute) => {
			let stringVal = minute.toString(10);
			if(stringVal.length === 1) {
				stringVal = `0${stringVal}`;
			}
			return stringVal;
		});

		if(this.currentSelectedHour !== null && this.currentSelectedMinute !== null) {
			this.parseHourMinutesToString();
		}
	}
}
</script>

<style scoped>
.timepicker-wrapper {
	position: relative;
	max-width: 150px;
}

.selected {
	background-color: #43b0f9;
}

.disabled {
	opacity: 0.5;
	cursor: default;
}

.amp-button {
	padding: 0;
	line-height: 17px;
	cursor: pointer;
}
</style>