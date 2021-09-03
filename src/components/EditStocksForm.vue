<template>
  <div>
    <md-table md-sort="teamName" md-sort-order="asc" v-model="editStockInput" class="text-left">
      <md-table-row slot="md-table-row" slot-scope="{ item }">
        <md-table-cell md-label="Team" md-sort-by="teamName">{{ item.teamName }}</md-table-cell>
        <md-table-cell md-label="Price" md-sort-by="ipoPrice">{{ item.ipoPrice | toCurrency }}</md-table-cell>
        <md-table-cell md-label="Quantity" md-sort-by="quantity">
          <input :ref="'quantityInput-' + item.teamId" @change="calculateTotal(item.teamId)" @keyup="calculateTotal(item.teamId)" :value="item.quantity" class="quantity-input" type="number" step="1" min="0" :max="item.quantityOwned">
        </md-table-cell>
        <md-table-cell md-label="Amount Owned" md-sort-by="quantityOwned">{{ item.quantityOwned }}</md-table-cell>
        <md-table-cell md-label="Total" md-sort-by="total">{{ item.total | toCurrency }}</md-table-cell>
      </md-table-row>
    </md-table>
    <div v-if="errorMessage" class="error text-center">
      {{errorMessage}}
    </div>
    <md-card-actions>
      <md-button class="md-primary" @click="submit">Submit</md-button>
    </md-card-actions>
  </div>
</template>

<script>
import { apolloClient } from "../main";
import gql from 'graphql-tag';

export default {
  name: "EditStocksForm",
  data() {
    return {
      availableStocks: null,
      editStockInput: [],
      errorMessage: null
    }
  },
  props: {
    formType: {
      type: String
    },
    successCb: {
      type: Function
    },
    entryId: {
      type: String
    },
  },
  computed: {
    filteredStockInput() {
      return this.editStockInput.filter(item => item.quantity > 0 && item.total > 0);
    }
  },
  methods: {
    calculateTotal(id) {
      const quantityValue = this.$refs['quantityInput-' + id].value;
      const index = this.editStockInput.findIndex(stock => stock.teamId === id);
      this.editStockInput[index].quantity = quantityValue;
      this.editStockInput[index].total = this.editStockInput[index].quantity * this.editStockInput[index].ipoPrice;
    },
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
        const email = sessionStorage.getItem('sports-exchange.email');
        for(let stock of this.filteredStockInput) {
          const input = {
            email,
            entryId: this.entryId,
            tournamentTeamId: stock.tournamentTeamId,
            quantity: parseInt(stock.quantity)
          };
          await apolloClient.mutate({
            mutation: gql`
              mutation SellEntryStocks($input: SellStockInput!) {
                sellEntryStocks(input: $input) {
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
        }

        this.successCb();
      }
    },
    validateStockInput() {
      for(let input of this.filteredStockInput) {
        if(input.quantity > input.quantityOwned) {
          this.errorMessage = "Desired stock amount exceeds current amount on hand"
          return false;
        }
      }
      return true;
    }
  },
  async created() {
    await this.getStocksForEntry();
    this.editStockInput = this.availableStocks.map((stock) => {
      return {
        ...stock,
        quantityOwned: stock.quantity,
        quantity: 0,
        total: 0
      }
    });
  }
}
</script>

<style scoped>
</style>