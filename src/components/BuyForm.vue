<template>
  <div>
    <div>
      <div class="md-layout">
				Team Name: {{buyStockDataInput.teamName}}
      </div>
			<div class="md-layout">
				Price: {{buyStockDataInput.price | toCurrency}}
      </div>
      <div class="md-layout quantity-container">
        <md-field>
          <label>Quantity</label>
          <md-input v-model="buyStockDataInput.quantity" type="number" min="1" :max="buyStockData.numStocksForSale"></md-input>
        </md-field>
      </div>
    </div>
    <div v-if="errorMessage" class="error text-center">
      {{errorMessage}}
    </div>
    <md-card-actions>
      <md-button @click="submit" class="md-primary md-raised">Buy</md-button>
    </md-card-actions>
  </div>
</template>

<script>
import { apolloClient } from "../main";
import gql from 'graphql-tag';

export default {
  name: "BuyForm",
  data() {
    return {
      errorMessage: null,
      buyStockDataInput: null
    }
  },
  props: {
    buyStockSuccessCb: {
      type: Function
    },
    entryId: {
      type: String
    },
    buyStockData: {
      type: Object
    }
  },
  methods: {
    async submit() {
      const input = {
        entryId: this.entryId,
        tournamentTeamId: this.buyStockData.tournamentTeamId,
        price: parseFloat(this.buyStockDataInput.price),
        quantity: parseInt(this.buyStockDataInput.quantity)
      };
			try {
				await apolloClient.mutate({
					mutation: gql`
						mutation CreateEntryBid($input: NewEntryBidInput!) {
							createEntryBid(input: $input) {
								entryId,
								tournamentTeamId,
								price,
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
        return err;
			}

			this.buyStockSuccessCb();
    }
  },
  async created() {
    this.buyStockDataInput = {
			teamName: this.buyStockData.teamName,
			price: this.buyStockData.currentAskPrice,
			quantity: this.buyStockData.numStocksForSale
		}
  }
}
</script>

<style scoped>
.quantity-container {
	width: 50px;
}
</style>