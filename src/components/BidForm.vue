<template>
  <div>
    <div>
      <h3 class="label"><span v-if="formType === 'new'">Select a </span>Stock for Bid</h3>
      <div v-if="formType === 'new'" class="custom-select-wrapper">
        <select class="custom-select" name="basic-dropdown" v-model="selectedBidInput">
          <option :value="null">Select Stock</option>
          <option v-for="stock in bidInput" :key="stock.id" :value="stock">{{stock.teamName}}</option>
        </select>
      </div>
      <div v-else>
        {{selectedBid.teamName}}
      </div>
    </div>
    <div v-if="selectedBidInput">
      <div class="md-layout">
        <md-field>
          <label>Max Buy Price</label>
          <currency-input :currency-to-format.sync="selectedBidInput.maxBuyPrice" :allows-negative="false"></currency-input>
        </md-field>
      </div>
      <div class="md-layout">
        <md-field>
          <label>Quantity</label>
          <md-input v-model="selectedBidInput.quantity" type="number" min="1"></md-input>
        </md-field>
      </div>
      <div class="md-layout">
        <div>
          <h3 class="label">Expires Date</h3>
        </div>
        <md-datepicker v-model="datePickerInput" :md-disabled-dates="disabledDates" md-immediately/>
        <time-picker class="timepicker-container" :selected-hour.sync="expiresHoursMinutes.hour" :selected-minute.sync="expiresHoursMinutes.minute"></time-picker>
      </div>
    </div>
    <div v-if="errorMessage" class="error text-center">
      {{errorMessage}}
    </div>
    <md-card-actions>
      <span class="submit-confirmation" v-if="selectedBidInput && selectedBidInput.quantity && selectedBidInput.quantity > 0 && selectedBidInput.maxBuyPrice">
        Are you sure you want to submit this bid?
      </span>
      <md-button :disabled="httpWait" v-if="selectedBidInput && selectedBidInput.quantity && selectedBidInput.quantity > 0 && selectedBidInput.maxBuyPrice" class="md-primary" :class="{ 'btn-disabled' : httpWait }" @click="submit">
        Submit
        <md-progress-spinner v-if="httpWait" class="btn-spin" :md-diameter="20" :md-stroke="3" md-mode="indeterminate"></md-progress-spinner>
      </md-button>
      <md-button v-else disabled>Submit</md-button>
      <md-button class="md-accent" @click="closeCb">Cancel</md-button>
    </md-card-actions>
  </div>
</template>

<script>
import { apolloClient } from "../main";
import gql from 'graphql-tag';
import CurrencyInput from './CurrencyInput.vue';
import { DateTime } from "luxon";
import TimePicker from './TimePicker.vue';

export default {
  components: { CurrencyInput, TimePicker },
  name: "BidForm",
  data() {
    let expiresDate = DateTime.fromObject({ zone: "America/New_York" }).endOf('day');
    const now = DateTime.fromObject({ zone: "America/New_York" });
    return {
      disabledDates: date => {
        const beginningOfDay = now.startOf('day');
        const dateTime = date.getTime();
        return dateTime < beginningOfDay;
      },
      tournamentTeamData: null,
      bidInput: [],
      errorMessage: null,
      selectedBidInput: null,
      datePickerInput: expiresDate.toJSDate(),
      httpWait: false,
      expiresHoursMinutes: {},
      startOfDayMillis: null,
      expiresDateInput: null
    }
  },
  props: {
    formType: {
      type: String
    },
    createNewBidSuccessCb: {
      type: Function
    },
    tradeAcceptedCb: {
      type: Function
    },
    entryId: {
      type: String
    },
    tournamentId: {
      type: String
    },
    selectedBid: {
      type: Object
    },
    closeCb: {
      type: Function
    }
  },
  watch: {
    expiresHoursMinutes: {
      handler(val) {
        if(val) {
          this.checkIfValueChanged();
        }
      },
      deep: true
    },
    datePickerInput(val) {
      if(val) {
        this.checkIfValueChanged();
      }
    }
  },
  methods: {
    checkIfValueChanged() {
      this.startOfDayMillis = DateTime.fromJSDate(this.datePickerInput, { zone: "America/New_York" }).startOf('day').toMillis();
      this.expiresDateInput = DateTime.fromMillis(this.startOfDayMillis).plus({ hours: this.expiresHoursMinutes.hour, minutes: this.expiresHoursMinutes.minute }).toMillis();
    },
    async fetchTournamentTeams() {
      const response = await apolloClient.query({
      fetchPolicy: 'no-cache',
        query: gql`
          query TournamentTeams($tournamentId: ID!) {
            tournamentTeams(tournamentId: $tournamentId) {
              id,
              teamId,
              teamName
            }
          }
        `,
        variables: {
          tournamentId: this.tournamentId
        }
      });

      this.tournamentTeamData = response.data.tournamentTeams;
    },
    async submit() {
      const nowMilliseconds = DateTime.fromObject({ zone: "America/New_York" }).toMillis();
      if(this.expiresDateInput < nowMilliseconds) {
        this.errorMessage = "Expires date must be in the future";
        return;
      }

      if(this.validateBidInput()) {
        this.httpWait = true;
        const expiresAt = DateTime.fromMillis(this.expiresDateInput).toJSDate().getTime();
        const input = {
          entryId: this.entryId,
          tournamentTeamId: this.selectedBidInput.id,
          price: parseFloat(this.selectedBidInput.maxBuyPrice),
          quantity: parseInt(this.selectedBidInput.quantity),
          expiresAt
        };
        try {
          const response = await apolloClient.mutate({
            mutation: gql`
              mutation CreateEntryBid($input: NewEntryBidInput!) {
                createEntryBid(input: $input) {
                  entryId,
                  tournamentTeamId,
                  price,
                  quantity,
                  trades {
                    id,
                    entry { id, name },
                    teamName,
                    tournamentTeamId,
                    quantity,
                    cost,
                    createdAt
                  }
                }
              }
            `,
            variables: {
              input
            }
          });

          const bidData = response.data.createEntryBid;
          if(bidData.trades && bidData.trades.length > 0) {
            const numAccepted = bidData.trades.length/2;
            this.tradeAcceptedCb("BID", numAccepted, this.selectedBidInput.quantity);
          } else {
            this.createNewBidSuccessCb();
          }
        } catch(err) {
          if(err.graphQLErrors && err.graphQLErrors.length > 0) {
            this.errorMessage = err.graphQLErrors[0].message;
          } else {
            this.errorMessage = "Server Error";
          }
          this.httpWait = false;
          return err;
        }
      }
    },
    validateBidInput() {
      if(!this.expiresDateInput) {
        this.errorMessage = "Must select an expires date";
        return false;
      }
      return true;
    }
  },
  async created() {
    await this.fetchTournamentTeams();
    this.startOfDayMillis = DateTime.fromJSDate(this.datePickerInput, { zone: "America/New_York" }).startOf('day').toMillis();
    this.$set(this.expiresHoursMinutes, 'hour', DateTime.fromMillis(this.startOfDayMillis).hour);
    this.$set(this.expiresHoursMinutes, 'minute', DateTime.fromMillis(this.startOfDayMillis).minute);
    this.bidInput = this.tournamentTeamData.map((teamData) => {
      return {
        ...teamData,
        maxBuyPrice: 0,
        quantity: 0
      }
    });
    if(this.selectedBid) {
      const index = this.bidInput.findIndex(stock => stock.id === this.selectedBid.tournamentTeamId);
      if(index >= 0) {
        this.bidInput[index].maxBuyPrice = this.selectedBid.price;
        this.selectedBidInput = this.bidInput[index];
      }
    }
  }
}
</script>

<style scoped>
.submit-confirmation {
  padding-right: 20px;
}
</style>