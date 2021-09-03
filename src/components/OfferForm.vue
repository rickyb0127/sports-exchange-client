<template>
  <div>
    <div>
      <h3 class="label"><span v-if="formType === 'new'">Select a </span>Stock for Offer</h3>
      <div v-if="formType === 'new'" class="custom-select-wrapper">
        <select class="custom-select" name="basic-dropdown" v-model="selectedOfferFormInput">
          <option :value="null">Select Stock</option>
          <option v-for="stock in offerFormInput" :key="stock.id" :value="stock">{{stock.teamName}}</option>
        </select>
      </div>
      <div v-else>
        {{selectedOffer.teamName}}
      </div>
    </div>
    <div v-if="selectedOfferFormInput">
      <div class="md-layout">
        <md-field>
          <label>Sell Price</label>
          <currency-input :currency-to-format.sync="selectedOfferFormInput.sellPrice" :allows-negative="false"></currency-input>
        </md-field>
      </div>
      <div class="md-layout">
        <md-field>
          <label>Quantity</label>
          <md-input v-model="selectedOfferFormInput.quantity" type="number" min="1" :max="selectedOfferFormInput.quantityOwned"></md-input>
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
      <md-button :disabled="httpWait" v-if="selectedOfferFormInput && selectedOfferFormInput.quantity && selectedOfferFormInput.quantity > 0 && selectedOfferFormInput.sellPrice" class="md-primary" :class="{ 'btn-disabled' : httpWait }" @click="submit">
        Submit
        <md-progress-spinner v-if="httpWait" class="btn-spin" :md-diameter="20" :md-stroke="3" md-mode="indeterminate"></md-progress-spinner>
      </md-button>
      <md-button v-else disabled>Submit</md-button>
    </md-card-actions>
  </div>
</template>

<script>
/* eslint-disable */
import { apolloClient } from "../main";
import gql from 'graphql-tag';
import CurrencyInput from './CurrencyInput.vue';
import { DateTime } from "luxon";

export default {
  components: { CurrencyInput },
  name: "OfferForm",
  data() {
    let expiresDate = DateTime.local().endOf('day');
    const now = DateTime.local();
    return {
      disabledDates: date => {
        const beginningOfDay = now.startOf('day');
        const dateTime = date.getTime();
        return dateTime < beginningOfDay;
      },
      availableStocks: null,
      offerFormInput: [],
      errorMessage: null,
      selectedOfferFormInput: null,
      expiresDateInput: expiresDate.toJSDate(),
      httpWait: false
    }
  },
  props: {
    formType: {
      type: String
    },
    createOfferSuccessCb: {
      type: Function
    },
    tradeAccpetedCb: {
      type: Function
    },
    entryId: {
      type: String
    },
    selectedOffer: {
      type: Object
    }
  },
  methods: {
    async getStocksForEntry() {
      const response = await apolloClient.query({
        fetchPolicy: 'no-cache',
        query: gql`
          query StocksByEntryId($entryId: ID!) {
            stocksByEntryId(entryId: $entryId) {
              teamName,
              teamId,
              tournamentTeamId,
              ipoPrice,
              quantity
            }
          }
        `,
        variables: {
          entryId: this.entryId
        }
      });

      this.availableStocks = response.data.stocksByEntryId;
    },
    async submit() {
      if(this.validateStockInput()) {
        this.httpWait = true;
        const email = sessionStorage.getItem('sports-exchange.email');
        const offerExpiresAt = DateTime.fromJSDate(this.expiresDateInput).endOf('day').toJSDate().getTime();
        const input = {
          email,
          entryId: this.entryId,
          tournamentTeamId: this.selectedOfferFormInput.tournamentTeamId,
          quantity: parseInt(this.selectedOfferFormInput.quantity),
          newPrice: parseFloat(this.selectedOfferFormInput.sellPrice),
          offerExpiresAt
        };
        try {
          const response = await apolloClient.mutate({
            mutation: gql`
              mutation SetStockAskPrice($input: StockPriceInput!) {
                setStockAskPrice(input: $input) {
                  teamName,
                  teamId,
                  ipoPrice,
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

          const offerData = response.data.setStockAskPrice;
          this.httpWait = false;
          if(offerData.trades && offerData.trades.length > 0) {
            const numAccepted = offerData.trades.length/2;
            this.tradeAcceptedCb("OFFER", numAccepted, this.selectedOfferFormInput.quantity);
          } else {
            this.createOfferSuccessCb();
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
        this.httpWait = false;
      }
    },
    validateStockInput() {
      if(this.selectedOfferFormInput.quantity > this.selectedOfferFormInput.quantityOwned) {
        this.errorMessage = "Desired stock amount exceeds current amount on hand";
        return false;
      } else if(!this.expiresDateInput) {
        this.errorMessage = "Must select an expires date";
        return false;
      }
      return true;
    }
  },
  async created() {
    await this.getStocksForEntry();
    this.offerFormInput = this.availableStocks.map((stock, index) => {
      return {
        id: index,
        ...stock,
        quantityOwned: stock.quantity,
        quantity: 0,
        sellPrice: stock.ipoPrice
      }
    });
    if(this.selectedOffer) {
      const index = this.offerFormInput.findIndex(stock => stock.tournamentTeamId === this.selectedOffer.tournamentTeamId);
      if(index >= 0) {
        this.offerFormInput[index].sellPrice = this.selectedOffer.currentAskPrice;
        this.selectedOfferFormInput = this.offerFormInput[index];
      }
    }
  }
}
</script>

<style scoped>
</style>