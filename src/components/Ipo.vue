<template>
  <div v-if="isPageReady">
    <h2>IPO</h2>
    Checkout Total: {{checkoutTotal | toCurrency}}
    <div v-if="checkoutTotal > ipoBudget - ipoCashSpent">Checkout would exceed max IPO budget</div>
    <div v-if="successMessage" class="alert-success">
      {{successMessage}}
      <span @click="successMessage = null"><md-icon class="fa fa-times-circle light link"></md-icon></span>
    </div>
    <div class="md-layout">
      <div class="md-layout-item"></div>
      <div class="md-layout-item"></div>
      <div class="md-layout-item">
        <md-button v-if="checkoutTotal > 0 && checkoutTotal <= ipoBudget - ipoCashSpent" class="md-raised md-primary" @click="placeOrderPressed()">Place Order</md-button>
        <md-button v-else disabled>Place Order</md-button>
      </div>
    </div>
    <md-table v-model="teamPurchaseInput" md-sort="teamName" class="text-left" md-sort-order="asc">
      <md-table-row slot="md-table-row" slot-scope="{ item }">
        <md-table-cell md-label="Team" md-sort-by="teamName">{{ item.teamName }}</md-table-cell>
        <md-table-cell md-label="Price" md-sort-by="ipoPrice">{{ item.ipoPrice | toCurrency }}</md-table-cell>
        <md-table-cell md-label="Quantity">
          <input :ref="'quantityInput-' + item.id" @change="calculateTotal(item.id)" @keyup="calculateTotal(item.id)" :value="item.quantity" class="quantity-input" type="number" step="1" min="0" max="">
        </md-table-cell>
        <md-table-cell md-label="Total">{{ item.total | toCurrency }}</md-table-cell>
      </md-table-row>
    </md-table>

    <md-dialog :md-active.sync="showConfirmCheckoutModal" :md-fullscreen="false">
      <md-dialog-title>Confirm Checkout</md-dialog-title>  
      <md-dialog-content>
        <div v-if="serverError" class="alert-error text-center">
          {{serverError}}
          <span @click="serverError = null"><md-icon class="fa fa-times-circle light link"></md-icon></span>
        </div>
        <md-table v-model="filteredCheckoutItems" class="text-left">
          <md-table-row slot="md-table-row" slot-scope="{ item }">
            <md-table-cell md-label="Team" md-sort-by="teamName">{{ item.teamName }}</md-table-cell>
            <md-table-cell md-label="Price" md-sort-by="ipoPrice">{{ item.ipoPrice | toCurrency }}</md-table-cell>
            <md-table-cell md-label="Quantity" md-sort-by="quantity">{{ item.quantity }}</md-table-cell>
            <md-table-cell md-label="Total" md-sort-by="total">{{ item.total | toCurrency }}</md-table-cell>
          </md-table-row>
        </md-table>
        <div class="totals">
          <div class="text-right">Checkout Total: {{checkoutTotal | toCurrency}}</div>
        </div>
      </md-dialog-content>

      <md-card-actions>
        <md-button :disabled="httpWait" @click="submitOrder" class="md-primary md-raised" :class="{ 'btn-disabled' : httpWait }">
          Submit
          <md-progress-spinner v-if="httpWait" class="btn-spin" :md-diameter="20" :md-stroke="3" md-mode="indeterminate"></md-progress-spinner>
        </md-button>
      </md-card-actions>
    </md-dialog>
  </div>
</template>

<script>
/* eslint-disable */
import { apolloClient } from "../main";
import gql from 'graphql-tag';

export default {
  name: "Ipo",
  data() {
    return {
      isPageReady: false,
      teamData: null,
      teamPurchaseInput: [],
      showConfirmCheckoutModal: false,
      // currentBalance: 0,
      user: null,
      successMessage: null,
      serverError: null,
      httpWait: false
    }
  },
  props: {
    tournamentId: {
      type: String
    },
    entryId: {
      type: String
    },
    ipoBudget: {
      type: Number
    },
    ipoCashSpent: {
      type: Number
    }
  },
  watch: {
    async entryId(newVal, oldVal) {
      if(newVal && newVal !== oldVal) {
        await this.init();
      }
    }
  },
  // TODO download ipo pricing sheet
  computed: {
      checkoutTotal() {
        if(this.teamPurchaseInput && this.teamPurchaseInput.length > 0) {
          const teamPurchaseTotalList = this.teamPurchaseInput.map(team => team.total);
          return teamPurchaseTotalList.reduce((acc, curr) => acc + curr);
        }
        return 0;
      },
      filteredCheckoutItems() {
        if(this.teamPurchaseInput && this.teamPurchaseInput.length > 0) {
          const filteredCheckoutItems = this.teamPurchaseInput.filter(item => item.quantity > 0 && item.total > 0);
          return filteredCheckoutItems;
        }
        return [];
      },
      remainingBalance() {
        return this.currentBalance - this.checkoutTotal;
      }
    },
  methods: {
    calculateTotal(id) {
      const inputValue = this.$refs['quantityInput-' + id].value;
      const index = this.teamPurchaseInput.findIndex(team => team.id === id);
      this.teamPurchaseInput[index].quantity = inputValue;
      this.teamPurchaseInput[index].total = this.teamPurchaseInput[index].quantity * this.teamPurchaseInput[index].ipoPrice;
    },
    async fetchIpoData() {
      const response = await apolloClient.query({
      fetchPolicy: 'no-cache',
        query: gql`
          query TournamentTeams($tournamentId: ID!) {
            tournamentTeams(tournamentId: $tournamentId) {
              id,
              teamId,
              teamName,
              seed,
              ipoPrice
            }
          }
        `,
        variables: {
          tournamentId: this.tournamentId
        }
      });

      this.teamData = response.data.tournamentTeams;
    },
    async fetchEntry() {
      const response = await apolloClient.query({
        fetchPolicy: 'no-cache',
        query: gql`
          query Entry($id: ID!) {
            entry(id: $id) {
              id,
              name,
              ipoCashSpent
            }
          }
        `,
        variables: {
          id: this.entryId
        }
      });

      const entry = response.data.entry;
      // change to update the prop cashSpent and send back to parent
      // this.currentBalance = entry.ipoCashSpent;
    },
    async placeOrderPressed() {
      await this.fetchEntry();
      this.showConfirmCheckoutModal = true;
    },
    async submitOrder() {
      if(this.checkoutTotal <= this.ipoBudget - this.ipoCashSpent) {
        const filteredTeamPuchaseInput = this.teamPurchaseInput.filter(_input => _input.quantity > 0);
        const userEmail = sessionStorage.getItem('sports-exchange.email');
        this.httpWait = true;
        try {
          for(let team of filteredTeamPuchaseInput) {
            const input = {
              quantity: parseInt(team.quantity),
              tournamentTeamId: team.id,
              userEmail,
              entryId: this.entryId
            };
            await apolloClient.mutate({
              mutation: gql`
                mutation IpoPurchase($input: IPOPurchaseInput!) {
                  ipoPurchase(input: $input) {
                    teamName,
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
        } catch(err) {
          if(err.graphQLErrors && err.graphQLErrors.length > 0) {
            this.serverError = err.graphQLErrors[0].message;
          } else {
            this.serverError = err.message;
          }
          this.httpWait = false;
          return err;
        }

        this.showConfirmCheckoutModal = false;
        this.successMessage = "Successfully made IPO purchase!";
        this.$router.push({ 
          name: "Portfolio",
          params: {
            entryId: this.entryId,
            successMessage: this.successMessage
          }
        });
      }
    },
    async init() {
      await this.fetchIpoData();
      this.teamPurchaseInput = this.teamData.map((team) => {
        return {
          ...team,
          quantity: 0,
          total: 0
        }
      });

      this.isPageReady = true;
    }
  },
  async created() {
    await this.init();
  }
}
</script>

<style scoped>
.quantity-input {
  width: 60px;
}
</style>