<template>
  <div>
    <div>
			<h3>You Get</h3>
      <div class="md-layout">
				Team Name: {{tradeStockDataInput.teamName}}
      </div>
			<div class="md-layout">
				Quantity: {{tradeStockDataInput.quantity}}
      </div>
    </div>
		<div>
			<h3>You Give</h3>
			<div class="tradable-teams" v-for="tradableTeam in tradeStockDataInput.tradableTeams" :key="tradableTeam.tournamentTeamId">
				<div class="md-layout">
					Team Name: {{tradableTeam.teamName}}
				</div>
				<div class="md-layout">
					Quantity: {{tradableTeam.quantity}}
				</div>
			</div>
    </div>
    <div v-if="errorMessage" class="error text-center">
      {{errorMessage}}
    </div>
    <md-card-actions>
      <md-button @click="submit" class="md-primary md-raised">Execute Trade</md-button>
    </md-card-actions>
  </div>
</template>

<script>
import { apolloClient } from "../main";
import gql from 'graphql-tag';

export default {
  name: "TradeForm",
  data() {
    return {
      errorMessage: null,
      tradeStockDataInput: null
    }
  },
  props: {
    tradeStockSuccessCb: {
      type: Function
    },
    entryId: {
      type: String
    },
    tradeStockData: {
      type: Object
    }
  },
  methods: {
    async submit() {
			const email = sessionStorage.getItem('sports-exchange.email');
      const input = {
				email,
				entryId: this.entryId,
				stockIdToTradeFor: this.tradeStockDataInput.stockId,
				quantity: this.tradeStockDataInput.quantity,
				tradableTeams: this.tradeStockDataInput.tradableTeams,
				teamName: this.tradeStockDataInput.teamName
      };
			try {
				await apolloClient.mutate({
					mutation: gql`
						mutation TradeStocks($input: TradeStocksInput!) {
							tradeStocks(input: $input) {
								id
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

			this.tradeStockSuccessCb();
    }
  },
  async created() {
		const tradableTeams = this.tradeStockData.tradableTeams.map((tradableTeam) => {
			return {
				price: tradableTeam.price,
				quantity: tradableTeam.quantity,
				teamName: tradableTeam.teamName,
				tournamentTeamId: tradableTeam.tournamentTeamId
			}
		});
    this.tradeStockDataInput = {
			stockId: this.tradeStockData.stockId,
			teamName: this.tradeStockData.teamName,
			quantity: this.tradeStockData.numStocksForSale,
			tradableTeams
		}
  }
}
</script>

<style scoped>
.quantity-container {
	width: 50px;
}

.tradable-teams {
	padding-bottom: 10px;
}
</style>