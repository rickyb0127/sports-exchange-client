<template>
  <div v-if="isPageReady">
    <div v-if="selectedStockToSell">
      <div class="md-layout">
        Team Name: {{offerFormInput.teamName}}
      </div>
      <div class="md-layout">
				Sell Price: {{offerFormInput.sellPrice | toCurrency}}
      </div>
      <div class="md-layout">
        <md-field>
          <label>Quantity</label>
          <md-input v-model="offerFormInput.quantity" type="number" min="1" :max="selectedStockToSell.quantity"></md-input>
        </md-field>
      </div>
    </div>
    <div v-if="errorMessage" class="error text-center">
      {{errorMessage}}
    </div>
    <md-card-actions>
      <md-button :disabled="httpWait" v-if="selectedStockToSell" class="md-primary md-raised" :class="{ 'btn-disabled' : httpWait }" @click="submit">
        Submit
        <md-progress-spinner v-if="httpWait" class="btn-spin" :md-diameter="20" :md-stroke="3" md-mode="indeterminate"></md-progress-spinner>
      </md-button>
      <md-button v-else disabled>Submit</md-button>
    </md-card-actions>
  </div>
</template>

<script>
import { apolloClient } from "../main";
import gql from 'graphql-tag';

export default {
  name: "SellForm",
  data() {
    return {
      isPageReady: false,
      availableStocks: null,
      offerFormInput: null,
      errorMessage: null,
      selectedStockToSell: true,
      httpWait: false
    }
  },
  props: {
    sellStockSuccessCb: {
      type: Function
    },
    entryId: {
      type: String
    },
    sellStockData: {
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
        const input = {
          email,
          entryId: this.entryId,
          tournamentTeamId: this.offerFormInput.tournamentTeamId,
          quantity: parseInt(this.offerFormInput.quantity),
          newPrice: parseFloat(this.offerFormInput.sellPrice)
        };

        try {
          await apolloClient.mutate({
            mutation: gql`
              mutation SetStockAskPrice($input: StockPriceInput!) {
                setStockAskPrice(input: $input) {
                  teamName,
                  teamId,
                  ipoPrice,
                  quantity
                }
              }
            `,
            variables: {
              input
            }
          });
        } catch(err) {
          if(err.graphQLErrors && err.graphQLErrors.length > 0) {
            this.errorMessage = err.graphQLErrors[0].message;
          } else {
            this.errorMessage = "Server Error";
          }
          this.httpWait = false;
          return err;
        }

        this.sellStockSuccessCb();
      }
    },
    validateStockInput() {
      if(this.offerFormInput.quantity > this.selectedStockToSell.quantity) {
        this.errorMessage = "Desired stock amount exceeds current amount on hand";
        return false;
      }
      return true;
    }
  },
  async created() {
    await this.getStocksForEntry();
    this.selectedStockToSell = this.availableStocks.find(stock => stock.tournamentTeamId === this.sellStockData.tournamentTeamId);
    this.offerFormInput = {
      teamName: this.sellStockData.teamName,
      tournamentTeamId: this.sellStockData.tournamentTeamId,
      quantity: this.sellStockData.quantity,
      sellPrice: this.sellStockData.price
    }
    this.isPageReady = true;
  }
}
</script>

<style scoped>
</style>