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
          <div>* the time that the offer expires will be the end of the day chosen</div>
        </div>
        <md-datepicker v-model="expiresDateInput" :md-disabled-dates="disabledDates" md-immediately/>
      </div>
    </div>
    <div v-if="errorMessage" class="error text-center">
      {{errorMessage}}
    </div>
    <md-card-actions>
      <md-button :disabled="httpWait" v-if="selectedBidInput && selectedBidInput.quantity && selectedBidInput.quantity > 0 && selectedBidInput.maxBuyPrice" class="md-primary" :class="{ 'btn-disabled' : httpWait }" @click="submit">
        Submit
        <md-progress-spinner v-if="httpWait" class="btn-spin" :md-diameter="20" :md-stroke="3" md-mode="indeterminate"></md-progress-spinner>
      </md-button>
      <md-button v-else disabled>Submit</md-button>
    </md-card-actions>
    <div v-if="errorMessage" class="error text-center">
      {{errorMessage}}
    </div>
  </div>
</template>

<script>
import { apolloClient } from "../main";
import gql from 'graphql-tag';
import CurrencyInput from './CurrencyInput.vue';
import { DateTime } from "luxon";

export default {
  components: { CurrencyInput },
  name: "BidForm",
  data() {
    let expiresDate = DateTime.local().endOf('day');
    const now = DateTime.local();
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
      expiresDateInput: expiresDate.toJSDate(),
      httpWait: false
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
    }
  },
  methods: {
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
      if(this.validateBidInput()) {
        this.httpWait = true;
        const expiresAt = DateTime.fromJSDate(this.expiresDateInput).endOf('day').toJSDate().getTime();
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
</style>