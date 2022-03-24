<template>
  <div v-if="isPageReady">
    <h2>Investment Metrics</h2>
    <div v-if="errorMessage" class="alert-error text-center">
      {{errorMessage}}
      <span @click="errorMessage = null"><md-icon class="fa fa-times-circle light link"></md-icon></span>
    </div>
    <md-table>
      <md-table-row>
        <md-table-cell class="text-left">Total $ IPO Investment</md-table-cell>
        <md-table-cell>{{entry.ipoCashSpent | toCurrency}}</md-table-cell>
      </md-table-row>

      <md-table-row>
        <md-table-cell class="text-left">Total IPO Shares Purchased</md-table-cell>
        <md-table-cell>{{originallyPurchasedStocks.length}}</md-table-cell>
      </md-table-row>

      <md-table-row>
        <md-table-cell class="text-left">Total Current Shares Owned</md-table-cell>
        <md-table-cell>{{getTotalNumStocks(tournamentTeamStocks)}}</md-table-cell>
      </md-table-row>

      <md-table-row>
        <md-table-cell class="text-left">Shares Remaining</md-table-cell>
        <md-table-cell>{{getNumSharesRemaining()}}</md-table-cell>
      </md-table-row>

      <md-table-row>
        <md-table-cell class="text-left">% Remaining</md-table-cell>
        <!-- is this percent of teams remaining or stocks? -->
        <md-table-cell>{{((getNumSharesRemaining()/getTotalNumStocks(tournamentTeamStocks)) * 100).toFixed(2)}}%</md-table-cell>
      </md-table-row>

      <md-table-row>
        <md-table-cell class="text-left">Current Teams Owned</md-table-cell>
        <md-table-cell>{{tournamentTeamStocks.length}}</md-table-cell>
      </md-table-row>

      <md-table-row>
        <md-table-cell class="text-left">Teams Remaining</md-table-cell>
        <md-table-cell>{{remainingTeams.length}}</md-table-cell>
      </md-table-row>

      <md-table-row>
        <md-table-cell class="text-left">Current Shares Owned Dividend Total</md-table-cell>
        <md-table-cell>{{getDividendTotal() | toCurrency}}</md-table-cell>
      </md-table-row>

      <md-table-row>
        <!-- secondary market costs at negative value (parse to abs) -->
        <md-table-cell class="text-left">Share Sales Income</md-table-cell>
        <md-table-cell v-if="entry.secondaryMarketCashSpent < 0">{{Math.abs(entry.secondaryMarketCashSpent) | toCurrency}}</md-table-cell>
        <md-table-cell v-else>{{0 | toCurrency}}</md-table-cell>
      </md-table-row>

      <md-table-row>
        <!-- secondary market costs at positive value -->
        <md-table-cell class="text-left">Share Purchase Costs</md-table-cell>
        <md-table-cell v-if="entry.secondaryMarketCashSpent > 0">{{entry.secondaryMarketCashSpent | toCurrency}}</md-table-cell>
        <md-table-cell v-else>{{0 | toCurrency}}</md-table-cell>
      </md-table-row>

      <md-table-row>
        <md-table-cell class="text-left">Total Costs</md-table-cell>
        <md-table-cell>{{entry.ipoCashSpent + entry.secondaryMarketCashSpent | toCurrency}}</md-table-cell>
      </md-table-row>

      <md-table-row>
        <!-- todo i assume this should be a year over year calc -->
        <md-table-cell class="text-left">Total $ Won to Date</md-table-cell>
        <md-table-cell>{{getDividendTotal() | toCurrency}}</md-table-cell>
      </md-table-row>

      <md-table-row>
        <!-- num of shares of remaining teams in tournament * ipo price -->
        <md-table-cell class="text-left">$ Remaining (At IPO Price)</md-table-cell>
        <md-table-cell>{{getRemainingTeamIpoPrice() | toCurrency}}</md-table-cell>
      </md-table-row>

      <md-table-row>
        <md-table-cell class="text-left">Profit/Loss</md-table-cell>
        <md-table-cell>{{(getDividendTotal() - (entry.ipoCashSpent + entry.secondaryMarketCashSpent)) | toCurrency}}</md-table-cell>
      </md-table-row>

      <md-table-row>
        <!-- profit || loss / totalCost -->
        <md-table-cell class="text-left">% Total Cost</md-table-cell>
        <md-table-cell>{{(((getDividendTotal() - (entry.ipoCashSpent + entry.secondaryMarketCashSpent)) / (entry.ipoCashSpent + entry.secondaryMarketCashSpent)) * 100).toFixed(2)}}%</md-table-cell>
      </md-table-row>
    </md-table>
  </div>
</template>

<script>
import { apolloClient } from "../main";
import gql from 'graphql-tag';

export default {
  name: "PortfolioSummary",
  data() {
    return {
      isPageReady: false,
      entry: null,
      tournamentTeamStocks: [],
      originallyPurchasedStocks: [],
      tournamentTeamData: null,
      remainingTeams: [],
      errorMessage: null
    }
  },
  props: {
    entryId: {
      type: String
    },
    tournamentId: {
      type: String
    }
  },
  methods: {
    async fetchEntry() {
      try {
        const response = await apolloClient.query({
          fetchPolicy: 'no-cache',
          query: gql`
            query Entry($id: ID!) {
              entry(id: $id) {
                id,
                tournamentId,
                name,
                ipoCashSpent,
                secondaryMarketCashSpent
              }
            }
          `,
          variables: {
            id: this.entryId
          }
        });

        this.entry = response.data.entry;
      } catch(err) {
        if(err.graphQLErrors && err.graphQLErrors.length > 0) {
          this.errorMessage = err.graphQLErrors[0].message;
        } else {
          this.errorMessage = "Server Error";
        }
        return err;
      }
    },
    async fetchTournamentTeams() {
      this.tournamentTeamData = await Promise.all(
        this.tournamentTeamStocks.map(async (stock) => {
          const teamId = stock.teamId;
          try {
            const response = await apolloClient.query({
            fetchPolicy: 'no-cache',
              query: gql`
                query TournamentTeamByTeamId($tournamentId: ID!, $teamId: ID!) {
                  tournamentTeamByTeamId(tournamentId: $tournamentId, teamId: $teamId) {
                    id,
                    teamId,
                    teamName,
                    isEliminated,
                    milestoneData {
                      milestoneId,
                      milestoneName,
                      dividendPrice,
                      wins,
                      losses,
                      ties
                    },
                    numStocksInCirculation
                  }
                }
              `,
              variables: {
                tournamentId: this.tournamentId,
                teamId
              }
            });

            const tournamentTeamData = response.data.tournamentTeamByTeamId;
            return tournamentTeamData;
          } catch(err) {
            if(err.graphQLErrors && err.graphQLErrors.length > 0) {
              this.errorMessage = err.graphQLErrors[0].message;
            } else {
              this.errorMessage = "Server Error";
            }
            return err;
          }
        })
      );
    },
    async getStocks() {
      const response = await apolloClient.query({
        fetchPolicy: 'no-cache',
        query: gql`
          query StocksByEntryId($entryId: ID!) {
            stocksByEntryId(entryId: $entryId) {
              teamName,
              ipoPrice,
              quantity,
              teamId,
              tournamentTeamId
            }
          }
        `,
        variables: {
          entryId: this.entryId
        }
      });

      this.tournamentTeamStocks = response.data.stocksByEntryId;
    },
    getNumSharesRemaining() {
      const remainingTeamIds = this.remainingTeams.map(team => team.teamId);
      return this.tournamentTeamStocks.reduce((result, teamStock) => {
        const found = remainingTeamIds.find(id => id === teamStock.teamId);
        if(found) {
          result += teamStock.quantity;
        }
        return result;
      }, 0);
    },
    truncateDecimals(number, digits) {
      const multiplier = Math.pow(10, digits);
      const adjustedNum = number * multiplier;
      const truncatedNum = Math[adjustedNum < 0 ? 'ceil' : 'floor'](adjustedNum);

      return truncatedNum / multiplier;
    },
    getDividendTotal() {
      const milestoneTeamData = this.tournamentTeamData.filter(teamData => teamData.milestoneData);
      if(milestoneTeamData.length === 0) {
        return 0;
      } else {
        const milestoneList = milestoneTeamData.map((team) => {
          const dataList = team.milestoneData.reduce((_result, _data) => {
            const dividendPrice = this.truncateDecimals(_data.dividendPrice / team.numStocksInCirculation, 2);
            const tournamentTeamId = team.id;
            if(!_result[tournamentTeamId]) {
              _result[tournamentTeamId] = dividendPrice;
            } else {
              _result[tournamentTeamId] += dividendPrice;
            }
            return _result;
          }, {});

          return dataList;
        });

        const total = milestoneList.reduce((result, _data) => {
          const found = this.tournamentTeamStocks.find(teamStock => teamStock.tournamentTeamId === Object.keys(_data)[0]);
          if(found) {
            const multiplier = found.quantity;
            result += (Object.values(_data)[0] * multiplier);
          }
          return result;
        }, 0);

        return total;
      }
    },
    async getOriginallyPurchasedStocks() {
      const response = await apolloClient.query({
        fetchPolicy: 'no-cache',
        query: gql`
          query GetOriginallyPurchasedStocks($entryId: ID!) {
            getOriginallyPurchasedStocks(entryId: $entryId) {
              id,
              tournamentTeamId,
              price,
              originalIpoEntryId
            }
          }
        `,
        variables: {
          entryId: this.entryId
        }
      });

      this.originallyPurchasedStocks = response.data.getOriginallyPurchasedStocks;
    },
    getTotalNumStocks(groupedStocks) {
      return groupedStocks.reduce((result, stock) => {
        result += stock.quantity;
        return result;
      }, 0);
    },
    getRemainingTeamIpoPrice() {
      const stocksRemaining = this.remainingTeams.reduce((result, team) => {
        const found = this.tournamentTeamStocks.find(stock => stock.teamId === team.teamId);
        if(found) {
          result.push(found);
        }

        return result;
      }, []);

      return stocksRemaining.reduce((result, stock) => {
        result += (stock.ipoPrice * stock.quantity);
        return result;
      }, 0);
    }
  },
  async created() {
    await this.fetchEntry();
    await this.getStocks();
    await this.getOriginallyPurchasedStocks();
    await this.fetchTournamentTeams();
    this.remainingTeams = this.tournamentTeamData.filter(teamData => !teamData.isEliminated);
    this.isPageReady = true;
  }
}
</script>

<style scoped>
</style>