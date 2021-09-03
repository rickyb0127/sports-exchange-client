<template>
  <div v-if="isPageReady">
    <h2>Offers</h2>
    <div v-if="successMessage" class="alert-success">
      {{successMessage}}
      <span @click="successMessage = null"><md-icon class="fa fa-times-circle light link"></md-icon></span>
    </div>
    <div v-if="errorMessage" class="alert-error">
      {{errorMessage}}
      <span @click="errorMessage = null"><md-icon class="fa fa-times-circle light link"></md-icon></span>
    </div>
    <div class="offers-table">
      <md-table md-sort="teamName" md-sort-order="asc" md-card v-model="myOfferData" class="text-left">
        <md-table-toolbar>
          <h3 class="md-title">My Current Offers</h3>
          <md-button @click="initializeShowOfferForm('new')" class="md-primary">Create New Offer</md-button>
        </md-table-toolbar>

        <md-table-row slot="md-table-row" slot-scope="{ item }">
          <md-table-cell md-label="Team Name" md-sort-by="teamName">{{ item.teamName }}</md-table-cell>
          <md-table-cell md-label="Quantity" md-sort-by="numStocksForSale">{{ item.numStocksForSale }}</md-table-cell>
          <md-table-cell md-label="Current Ask Price" md-sort-by="currentAskPrice">{{ item.currentAskPrice | toCurrency}}</md-table-cell>
          <md-table-cell md-label="Expires At" md-sort-by="offerExpiresAt">{{ formatDate(parseInt(item.offerExpiresAt)) }}</md-table-cell>
          <md-table-cell md-label="Edit">
            <span @click="initializeShowOfferForm('edit', item)"><md-icon class="fas fa-edit link"></md-icon></span>
          </md-table-cell>
          <md-table-cell md-label="Delete">
            <span @click="deleteOffer(item.tournamentTeamId)"><md-icon class="fas fa-trash link"></md-icon></span>
          </md-table-cell>
        </md-table-row>
      </md-table>

      <md-table md-sort="teamName" md-sort-order="asc" md-card v-model="leagueOfferData" class="text-left">
        <md-table-toolbar>
          <h3 class="md-title">Current Open League Offers</h3>
        </md-table-toolbar>

        <md-table-row slot="md-table-row" slot-scope="{ item }">
          <md-table-cell md-label="Team Name" md-sort-by="teamName">{{ item.teamName }}</md-table-cell>
          <md-table-cell md-label="Quantity" md-sort-by="numStocksForSale">{{ item.numStocksForSale }}</md-table-cell>
          <md-table-cell md-label="Current Ask Price" md-sort-by="currentAskPrice">{{ item.currentAskPrice | toCurrency}}</md-table-cell>
          <md-table-cell md-label="Expires At" md-sort-by="offerExpiresAt">{{ formatDate(parseInt(item.offerExpiresAt)) }}</md-table-cell>
          <md-table-cell md-label="Buy">
            <span @click="initializeBuyForm(item)"><md-icon class="fas fa-shopping-cart link"></md-icon></span>
          </md-table-cell>
        </md-table-row>
      </md-table>
    </div>

    <div class="offers-table">
      <md-table md-sort="teamName" md-sort-order="asc" md-card v-model="myBidData" class="text-left">
        <md-table-toolbar>
          <h3 class="md-title">My Current Bids</h3>
          <md-button @click="initializeShowBidForm('new')" class="md-primary">Create New Bid</md-button>
        </md-table-toolbar>

        <md-table-row slot="md-table-row" slot-scope="{ item }">
          <md-table-cell md-label="Team" md-sort-by="teamName">{{ item.teamName }}</md-table-cell>
          <md-table-cell md-label="Quantity" md-sort-by="quantity">{{ item.quantity }}</md-table-cell>
          <md-table-cell md-label="Max Bid Price" md-sort-by="price">{{ item.price | toCurrency }}</md-table-cell>
          <md-table-cell md-label="Expires At" md-sort-by="expiresAt">{{ formatDate(parseInt(item.expiresAt)) }}</md-table-cell>
          <md-table-cell md-label="Edit">
            <span @click="initializeShowBidForm('edit', item)"><md-icon class="fas fa-edit link"></md-icon></span>
          </md-table-cell>
          <md-table-cell md-label="Delete">
            <span @click="deleteBid(item.id)"><md-icon class="fas fa-trash link"></md-icon></span>
          </md-table-cell>
        </md-table-row>
      </md-table>

      <md-table md-sort="teamName" md-sort-order="asc" md-card v-model="leagueBidData" class="text-left">
        <md-table-toolbar>
          <h3 class="md-title">Current Open League Bids</h3>
        </md-table-toolbar>

        <md-table-row slot="md-table-row" slot-scope="{ item }">
          <md-table-cell md-label="Team" md-sort-by="teamName">{{ item.teamName }}</md-table-cell>
          <md-table-cell md-label="Quantity" md-sort-by="quantity">{{ item.quantity }}</md-table-cell>
          <md-table-cell md-label="Max Bid Price" md-sort-by="price">{{ item.price | toCurrency }}</md-table-cell>
          <md-table-cell md-label="Expires At" md-sort-by="expiresAt">{{ formatDate(parseInt(item.expiresAt)) }}</md-table-cell>
          <md-table-cell md-label="Sell">
            <span v-if="hasStockToSell(item)" @click="initializeSellForm(item)"><md-icon class="fas fa-dollar-sign link"></md-icon></span>
          </md-table-cell>
        </md-table-row>
      </md-table>
    </div>

    <div class="offers-table">
      <md-table md-sort="createdAt" md-sort-order="asc" md-card v-model="transactionData" class="text-left">
        <md-table-toolbar>
          <h3 class="md-title">Recent Transactions</h3>
          <md-button @click="showMoreTransactionsModal = true" class="md-primary">See More</md-button>
        </md-table-toolbar>

        <md-table-row slot="md-table-row" slot-scope="{ item }">
          <md-table-cell md-label="Transaction Detail" md-sort-by="text">{{ item.text }}</md-table-cell>
          <md-table-cell md-label="Date" md-sort-by="createdAt">{{ item.createdAt }}</md-table-cell>
        </md-table-row>
      </md-table>
    </div>

    <md-dialog v-if="showOfferForm" :md-active.sync="showOfferForm" :md-fullscreen="false">
      <md-dialog-title>Create Offer</md-dialog-title>
      <md-dialog-content>
        <offer-form :form-type="offerFormOptions.type" :selected-offer="offerFormOptions.selectedOffer" :create-offer-success-cb="createOfferSuccessCb" :trade-accepted-cb="tradeAcceptedCb" :entry-id="entryId"></offer-form>
      </md-dialog-content>
    </md-dialog>

    <md-dialog v-if="showBidForm" :md-active.sync="showBidForm" :md-fullscreen="false">
      <md-dialog-title>Create New Bid</md-dialog-title>
      <md-dialog-content>
        <bid-form :form-type="bidFormOptions.type" :selected-bid="bidFormOptions.selectedBid" :create-new-bid-success-cb="createNewBidSuccessCb" :trade-accepted-cb="tradeAcceptedCb" :entry-id="entryId" :tournament-id="tournamentId"></bid-form>
      </md-dialog-content>
    </md-dialog>

    <md-dialog v-if="showBuyForm" :md-active.sync="showBuyForm" :md-fullscreen="false">
      <md-dialog-title>Buy Stock</md-dialog-title>
      <md-dialog-content>
        <buy-form :buy-stock-data="buyStockData" :buy-stock-success-cb="buyStockSuccessCb" :entry-id="entryId"></buy-form>
      </md-dialog-content>
    </md-dialog>

    <md-dialog v-if="showSellForm" :md-active.sync="showSellForm" :md-fullscreen="false">
      <md-dialog-title>Sell Stock</md-dialog-title>
      <md-dialog-content>
        <sell-form :sell-stock-data="sellStockData" :sell-stock-success-cb="sellStockSuccessCb" :entry-id="entryId"></sell-form>
      </md-dialog-content>
    </md-dialog>

    <md-dialog v-if="showMoreTransactionsModal" :md-active.sync="showMoreTransactionsModal" :md-fullscreen="false">
      <md-dialog-title>All Transactions</md-dialog-title>
      <md-dialog-content>
        <md-table md-sort="createAt" md-sort-order="asc" md-card v-model="rawTransactionData" class="text-left">
          <md-table-row slot="md-table-row" slot-scope="{ item }">
            <md-table-cell md-label="Transaction Detail" md-sort-by="text">{{ item.text }}</md-table-cell>
            <md-table-cell md-label="Date" md-sort-by="createdAt">{{ item.createdAt }}</md-table-cell>
          </md-table-row>
        </md-table>
      </md-dialog-content>
    </md-dialog>
  </div>
</template>

<script>
/* eslint-disable */
import { apolloClient } from "../main";
import gql from 'graphql-tag';
import OfferForm from './OfferForm.vue';
import BidForm from './BidForm.vue';
import BuyForm from './BuyForm.vue';
import SellForm from './SellForm.vue';
import { DateTime } from "luxon";

export default {
  components: { OfferForm, BidForm, BuyForm, SellForm },
  name: "Offers",
  data() {
    return {
      isPageReady: false,
      successMessage: null,
      showOfferForm: false,
      offerFormOptions: {},
      bidFormOptions: {},
      showBidForm: false,
      myOfferData: [],
      leagueOfferData: [],
      myBidData: [],
      leagueBidData: [],
      transactionData: [],
      buyStockData: null,
      showBuyForm: false,
      showSellForm: false,
      availableStocks: null,
      showMoreTransactionsModal: false,
      rawTransactionData: [],
      errorMessage: null
    }
  },
  props: {
    tournamentId: {
      type: String
    },
    entryId: {
      type: String
    }
  },
  methods: {
    async createNewBidSuccessCb() {
      this.showBidForm = false;
      await this.initBidsAndAsks();
      this.successMessage = "Successfully created new bid!";
    },
    async createOfferSuccessCb() {
      this.showOfferForm = false;
      await this.initBidsAndAsks();
      this.successMessage = "Successfully set new stock offer!";
    },
    async buyStockSuccessCb() {
      this.showBuyForm = false;
      await this.initBidsAndAsks();
      this.successMessage = "Successfully bought stock!";
    },
    async sellStockSuccessCb() {
      this.showSellForm = false;
      await this.initBidsAndAsks();
      this.successMessage = "Successfully sold stock!";
    },
    async deleteOffer(tournamentTeamId) {
      await apolloClient.mutate({
        fetchPolicy: 'no-cache',
        mutation: gql`
          mutation SetTournamentTeamStockPriceToNull($tournamentTeamId: ID!, $entryId: ID!) {
            setTournamentTeamStockPriceToNull(tournamentTeamId: $tournamentTeamId, entryId: $entryId) {
              id
            }
          }
        `,
        variables: {
          tournamentTeamId,
          entryId: this.entryId
        }
      });

      await this.initBidsAndAsks();
      this.successMessage = "Successfully deleted offer!";
    },
    async deleteBid(entryBidId) {
      await apolloClient.mutate({
        fetchPolicy: 'no-cache',
        mutation: gql`
          mutation DeleteEntryBid($id: ID!) {
            deleteEntryBid(id: $id) {
              id
            }
          }
        `,
        variables: {
          id: entryBidId
        }
      });

      await this.initBidsAndAsks();
      this.successMessage = "Successfully deleted bid!";
    },
    hasStockToSell(stockData) {
      return this.availableStocks.find(stock => stock.tournamentTeamId === stockData.tournamentTeamId);
    },
    async tradeAcceptedCb(initializedFrom, numAccepted, numOffered) {
      await this.initBidsAndAsks();
      if(initializedFrom === "BID") {
        this.showBidForm = false;
        this.successMessage = `${numAccepted} of ${numOffered} bid(s) been accepted! These trades have processed`;
      }
      if(initializedFrom === "OFFER") {
        this.showOfferForm = false;
        this.successMessage = `${numAccepted} of ${numOffered} offer(s) has been accepted! These trades have processed`;
      }
    },
    formatDate(timestamp) {
      const date = new Date(timestamp).toLocaleString('en-US', {
        weekday: 'short', // long, short, narrow
        day: 'numeric', // numeric, 2-digit
        year: 'numeric', // numeric, 2-digit
        month: 'long', // numeric, 2-digit, long, short, narrow
        hour: 'numeric', // numeric, 2-digit
        minute: 'numeric' // numeric, 2-digit
      })
      return date;
    },
    async fetchEntryBids() {
      const response = await apolloClient.query({
        fetchPolicy: 'no-cache',
        query: gql`
          query GetBidsForEntry($entryId: ID!) {
            getBidsForEntry(entryId: $entryId) {
              myBids {
                id,
                tournamentTeamId,
                teamName,
                price,
                quantity,
                expiresAt
              },
              leagueBids {
                id,
                tournamentTeamId,
                teamName,
                price,
                quantity,
                expiresAt
              }
            }
          }
        `,
        variables: {
          entryId: this.entryId
        }
      });

      this.myBidData = response.data.getBidsForEntry.myBids;
      this.leagueBidData = response.data.getBidsForEntry.leagueBids;
    },
    async fetchOffers() {
      const response = await apolloClient.query({
        fetchPolicy: 'no-cache',
        query: gql`
          query GetOfferedStocksForTournament($tournamentId: ID!, $entryId: ID!) {
            getOfferedStocksForTournament(tournamentId: $tournamentId, entryId: $entryId) {
              myStockOffers {
                teamName,
                numStocksForSale,
                currentAskPrice,
                tournamentTeamId,
                offerExpiresAt
              },
              leagueStockOffers {
                teamName,
                numStocksForSale,
                currentAskPrice,
                tournamentTeamId,
                offerExpiresAt
              }
            }
          }
        `,
        variables: {
          tournamentId: this.tournamentId,
          entryId: this.entryId
        }
      });

      this.myOfferData = response.data.getOfferedStocksForTournament.myStockOffers;
      this.leagueOfferData = response.data.getOfferedStocksForTournament.leagueStockOffers;
    },
    async fetchTransactions() {
      const response = await apolloClient.query({
        fetchPolicy: 'no-cache',
        query: gql`
          query GetTournamentTransactions($tournamentId: ID!) {
            getTournamentTransactions(tournamentId: $tournamentId) {
              id,
              entry { id, name },
              teamName,
              tournamentTeamId,
              quantity,
              cost,
              createdAt
            }
          }
        `,
        variables: {
          tournamentId: this.tournamentId
        }
      });

      this.rawTransactionData = response.data.getTournamentTransactions;
      if(this.rawTransactionData.length > 10) {
        this.transactionData = this.rawTransactionData.slice(0, 10);
      } else {
        this.transactionData = this.rawTransactionData;
      }

      this.transactionData = this.transactionData.map((data) => {
        const verb = data.cost > 0 ? 'bought' : 'sold';
        return {
          text: `${data.entry.name} ${verb} ${data.quantity} share of ${data.teamName} for ${this.$options.filters.toCurrency(Math.abs(data.cost))}`,
          createdAt: this.formatDate(parseInt(data.createdAt))
        }
      });

      this.rawTransactionData = this.rawTransactionData.map((data) => {
        const verb = data.cost > 0 ? 'bought' : 'sold';
        return {
          text: `${data.entry.name} ${verb} ${data.quantity} share of ${data.teamName} for ${this.$options.filters.toCurrency(Math.abs(data.cost))}`,
          createdAt: this.formatDate(parseInt(data.createdAt))
        }
      });
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
    async refreshExpiredBidsAndAsks() {
      await apolloClient.mutate({
        fetchPolicy: 'no-cache',
        mutation: gql`
          mutation RemoveExpiredBidsAndAsks($tournamentId: ID!) {
            removeExpiredBidsAndAsks(tournamentId: $tournamentId) {
              response
            }
          }
        `,
        variables: {
          tournamentId: this.tournamentId
        }
      });
    },
    formatDate(timestamp) {
      const date = new Date(timestamp).toLocaleString('en-US', {
        weekday: 'short', // long, short, narrow
        day: 'numeric', // numeric, 2-digit
        year: 'numeric', // numeric, 2-digit
        month: 'long', // numeric, 2-digit, long, short, narrow
        hour: 'numeric', // numeric, 2-digit
        minute: 'numeric' // numeric, 2-digit
      })
      return date;
    },
    initializeShowOfferForm(type, selectedOffer) {
      this.showOfferForm = true;
      this.offerFormOptions.type = type;
      if(selectedOffer) {
        this.offerFormOptions.selectedOffer = selectedOffer;
      } else {
        this.offerFormOptions.selectedOffer = null;
      }
    },
    initializeShowBidForm(type, selectedBid) {
      this.showBidForm = true;
      this.bidFormOptions.type = type;
      if(selectedBid) {
        this.bidFormOptions.selectedBid = selectedBid;
      } else {
        this.bidFormOptions.selectedBid = null;
      }
    },
    async initializeBuyForm(buyStockData) {
      const now = DateTime.local();
      if(now > buyStockData.offerExpiresAt) {
        this.errorMessage = "Offer has expired";
        await this.initBidsAndAsks();
        return;
      }
      this.showBuyForm = true;
      this.buyStockData = buyStockData;
    },
    async initializeSellForm(sellStockData) {
      const now = DateTime.local();
      if(now > sellStockData.expiresAt) {
        this.errorMessage = "Offer has expired";
        await this.initBidsAndAsks();
        return;
      }
      this.showSellForm = true;
      this.sellStockData = sellStockData;
    },
    async initBidsAndAsks() {
      await this.refreshExpiredBidsAndAsks();
      await this.fetchOffers();
      await this.fetchEntryBids();
      await this.fetchTransactions();
      await this.getStocksForEntry();
    }
  },
  async created() {
    await this.initBidsAndAsks();
    this.isPageReady = true;
  }
}
</script>

<style scoped>
.offers-table {
  padding-bottom: 20px;
}
</style>