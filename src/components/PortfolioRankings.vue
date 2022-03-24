<template>
  <div v-if="isPageReady">
    <h2>Portfolio Rankings</h2>

    <md-table md-sort="percentMoneyWonInvested" md-sort-order="desc" v-model="portfolioEntries">
      <md-table-row class="text-left link" @click="showRankDetailModal = true, selectedEntry = item" slot="md-table-row" slot-scope="{ item, index }">
        <md-table-cell md-label="Rank">{{ index + 1 }}</md-table-cell>
        <md-table-cell md-label="Owner" md-sort-by="owner">{{ item.owner }}</md-table-cell>
				<md-table-cell md-label="Entry Name" md-sort-by="entryName">{{ item.entryName }}</md-table-cell>
				<md-table-cell md-label="Total $ IPO Investment" md-sort-by="totalInitialInvestment">{{ item.totalInitialInvestment | toCurrency }}</md-table-cell>
				<md-table-cell md-label="Total IPO Shares Purchased" md-sort-by="totalInitialStocksOwned">{{ item.totalInitialStocksOwned }}</md-table-cell>
				<md-table-cell md-label="Total Current Shares Owned" md-sort-by="totalCurrentStocksOwned">{{ item.totalCurrentStocksOwned }}</md-table-cell>
				<md-table-cell md-label="Shares Remaining" md-sort-by="stocksRemaining">{{ item.stocksRemaining }}</md-table-cell>
				<md-table-cell md-label="% Remaining" md-sort-by="percentStocksRemaining">{{ item.percentStocksRemaining.toFixed(2) }}%</md-table-cell>
				<md-table-cell md-label="Current Teams Owned" md-sort-by="totalCurrentTeamsOwned">{{ item.totalCurrentTeamsOwned }}</md-table-cell>
				<md-table-cell md-label="Teams Remaining" md-sort-by="totalCurrentTeamsRemaining">{{ item.totalCurrentTeamsRemaining }}</md-table-cell>
				<md-table-cell md-label="$ Won to Date" md-sort-by="moneyWonToDate">{{ item.moneyWonToDate | toCurrency }}</md-table-cell>
				<md-table-cell md-label="Won % Money Invested" md-sort-by="percentMoneyWonInvested">{{ item.percentMoneyWonInvested.toFixed(2) }}%</md-table-cell>
				<md-table-cell md-label="$ Remaining (At IPO Price)" md-sort-by="originalMoneyRemaining">{{ item.originalMoneyRemaining | toCurrency }}</md-table-cell>
        <md-table-cell md-label="Profit/Loss" md-sort-by="profitLoss">{{ item.profitLoss | toCurrency }}</md-table-cell>
				<md-table-cell md-label="$ Remaining % Money Invested" md-sort-by="percentMoneyRemaining">{{ item.percentMoneyRemaining.toFixed(2) }}%</md-table-cell>
      </md-table-row>
    </md-table>

    <md-dialog v-if="showRankDetailModal && selectedEntry" :md-active.sync="showRankDetailModal">
      <md-dialog-title class="text-center">{{selectedEntry.entryName}}</md-dialog-title>
      <md-dialog-content>
        <div class="section">
          <div class="section-header">Owner</div>
          <div>{{selectedEntry.owner}}</div>
        </div>
        <div class="section">
          <div class="section-header">Total $ IPO Investment</div>
          <div>{{selectedEntry.totalInitialInvestment | toCurrency}}</div>
        </div>
        <div class="section">
          <div class="section-header">Total Initial Stocks Owned</div>
          <div>{{selectedEntry.totalInitialStocksOwned}}</div>
        </div>
        <div class="section">
          <div class="section-header">Total Current Stocks Owned</div>
          <div>{{selectedEntry.totalCurrentStocksOwned}}</div>
        </div>
        <div class="section">
          <div class="section-header">Stocks Remaining</div>
          <div>{{selectedEntry.stocksRemaining}}</div>
        </div>
        <div class="section">
          <div class="section-header">% Stocks Remaining</div>
          <div>{{selectedEntry.percentStocksRemaining.toFixed(2)}}</div>
        </div>
        <div class="section">
          <div class="section-header">Total Current Teams Owned</div>
          <div>{{selectedEntry.totalCurrentTeamsOwned}}</div>
        </div>
        <div class="section">
          <div class="section-header">Total Current Teams Remaining</div>
          <div>{{selectedEntry.totalCurrentTeamsRemaining}}</div>
        </div>
        <div class="section">
          <div class="section-header">$ Won to Date</div>
          <div>{{selectedEntry.moneyWonToDate | toCurrency}}</div>
        </div>
        <div class="section">
          <div class="section-header">Won % Money Invested</div>
          <div>{{selectedEntry.percentMoneyWonInvested.toFixed(2)}}</div>
        </div>
        <div class="section">
          <div class="section-header">$ Remaining (At IPO Price)</div>
          <div>{{selectedEntry.originalMoneyRemaining | toCurrency}}</div>
        </div>
        <div class="section">
          <div class="section-header">Profit/Loss</div>
          <div>{{selectedEntry.profitLoss | toCurrency}}</div>
        </div>
        <div class="section">
          <div class="section-header">$ Remaining % Money Invested</div>
          <div>{{selectedEntry.percentMoneyRemaining.toFixed(2)}}</div>
        </div>
      </md-dialog-content>

      <md-card-actions>
        <md-button @click="showRankDetailModal = false, selectedEntry = null" class="md-primary md-raised">
          Close
        </md-button>
      </md-card-actions>
    </md-dialog>
  </div>
  <div v-else>
    <md-progress-spinner class="spinner-primary" md-mode="indeterminate"></md-progress-spinner>
  </div>
</template>

<script>
import { apolloClient } from "../main";
import gql from 'graphql-tag';

export default {
  name: "PortfolioRankings",
  data() {
    return {
      isPageReady: false,
			portfolioEntries: [],
      showRankDetailModal: false,
      selectedEntry: null
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
  watch: {
    async entryId(newVal, oldVal) {
      if(newVal && newVal !== oldVal) {
        await this.init();
      }
    }
  },
  methods: {
		async getStocks(entryId) {
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
          entryId
        }
      });

      return response.data.stocksByEntryId;
    },
		async fetchTournamentTeams(tournamentTeamStocks) {
			let tournamentTeamData = [];

			for(const stock of tournamentTeamStocks) {
				const teamId = stock.teamId;
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

				const data = response.data.tournamentTeamByTeamId;
				tournamentTeamData.push(data);
			}

			return tournamentTeamData;
    },
		async fetchEntries() {
			const response = await apolloClient.query({
        fetchPolicy: 'no-cache',
        query: gql`
          query EntriesByTournamentId($tournamentId: ID!) {
            entriesByTournamentId(tournamentId: $tournamentId) {
              id,
              name,
              tournamentId,
              ipoCashSpent,
              secondaryMarketCashSpent
            }
          }
        `,
        variables: {
          tournamentId: this.tournamentId
        }
      });

      return response.data.entriesByTournamentId;
		},
		async getOwnersByEntry(entryId) {
      const response = await apolloClient.query({
        fetchPolicy: 'no-cache',
        query: gql`
          query usersByEntryId($entryId: ID!) {
            usersByEntryId(entryId: $entryId) {
              id,
              firstname,
              lastname,
              email
            }
          }
        `,
        variables: {
          entryId
        }
      });

			let ownerString = `${response.data.usersByEntryId[0].firstname} ${response.data.usersByEntryId[0].lastname}`;
			if(response.data.usersByEntryId.length > 1) {
				for(let i = 1; i < response.data.usersByEntryId.length; i++) {
					ownerString += `, ${response.data.usersByEntryId[i].firstname} ${response.data.usersByEntryId[i].lastname}`;
				}
			}

      return ownerString;
    },
		async getNumOriginallyPurchasedStocks(entryId) {
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
          entryId
        }
      });

      return response.data.getOriginallyPurchasedStocks.length;
    },
		async getNumCurrentStocks(entryId) {
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
          entryId
        }
      });

			return response.data.stocksByEntryId.reduce((result, stock) => {
        result += stock.quantity;
        return result;
      }, 0);
    },
		async getNumSharesRemaining(entryId) {
			const tournamentTeamStocks = await this.getStocks(entryId);
			const tournamentTeamData = await this.fetchTournamentTeams(tournamentTeamStocks);
			const remainingTeams = tournamentTeamData.filter(teamData => !teamData.isEliminated);
      const remainingTeamIds = remainingTeams.map(team => team.teamId);

      return tournamentTeamStocks.reduce((result, teamStock) => {
        const found = remainingTeamIds.find(id => id === teamStock.teamId);
        if(found) {
          result += teamStock.quantity;
        }
        return result;
      }, 0);
    },
		async getNumTeamsRemaining(entryId) {
			const tournamentTeamStocks = await this.getStocks(entryId);
			const tournamentTeamData = await this.fetchTournamentTeams(tournamentTeamStocks);
			const remainingTeams = tournamentTeamData.filter(teamData => !teamData.isEliminated);
			return remainingTeams.length;
		},
    truncateDecimals(number, digits) {
      const multiplier = Math.pow(10, digits);
      const adjustedNum = number * multiplier;
      const truncatedNum = Math[adjustedNum < 0 ? 'ceil' : 'floor'](adjustedNum);

      return truncatedNum / multiplier;
    },
		async getDividendTotal(entryId) {
			const tournamentTeamStocks = await this.getStocks(entryId);
			const tournamentTeamData = await this.fetchTournamentTeams(tournamentTeamStocks);
      const milestoneTeamData = tournamentTeamData.filter(teamData => teamData.milestoneData);
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
          const found = tournamentTeamStocks.find(teamStock => teamStock.tournamentTeamId === Object.keys(_data)[0]);
          if(found) {
            const multiplier = found.quantity;
            result += (Object.values(_data)[0] * multiplier);
          }
          return result;
        }, 0);

        return total;
      }
    },
		async getRemainingStocksIPOValue(entryId) {
			const tournamentTeamStocks = await this.getStocks(entryId);
			const tournamentTeamData = await this.fetchTournamentTeams(tournamentTeamStocks);
			const remainingTeams = tournamentTeamData.filter(teamData => !teamData.isEliminated);

      const stocksRemaining = remainingTeams.reduce((result, team) => {
        const found = tournamentTeamStocks.find(stock => stock.teamId === team.teamId);
        if(found) {
          result.push(found);
        }

        return result;
      }, []);

      return stocksRemaining.reduce((result, stock) => {
        result += (stock.ipoPrice * stock.quantity);
        return result;
      }, 0);
    },
		async init() {
			const entries = await this.fetchEntries();

			this.portfolioEntries = await Promise.all(
				entries.map(async (entry) => {
					const teamStocks = await this.getStocks(entry.id);

					const owner = await this.getOwnersByEntry(entry.id);
					const entryName = entry.name; 
					const totalInitialInvestment = entry.ipoCashSpent; 
					const totalInitialStocksOwned = await this.getNumOriginallyPurchasedStocks(entry.id); 
					const totalCurrentStocksOwned = await this.getNumCurrentStocks(entry.id); 
					const stocksRemaining = await this.getNumSharesRemaining(entry.id); 
					const percentStocksRemaining = (stocksRemaining/totalCurrentStocksOwned) * 100;
					const totalCurrentTeamsOwned = teamStocks.length; 
					const totalCurrentTeamsRemaining = await this.getNumTeamsRemaining(entry.id); 
					const moneyWonToDate = await this.getDividendTotal(entry.id); 
					const percentMoneyWonInvested = (moneyWonToDate / totalInitialInvestment) * 100; 
					const profitLoss = moneyWonToDate - (entry.ipoCashSpent + entry.secondaryMarketCashSpent); 
					const originalMoneyRemaining = await this.getRemainingStocksIPOValue(entry.id); 
					const percentMoneyRemaining = (originalMoneyRemaining / totalInitialInvestment) * 100;

					return {
						owner,
						entryName,
						totalInitialInvestment,
						totalInitialStocksOwned,
						totalCurrentStocksOwned,
						stocksRemaining,
						percentStocksRemaining,
						totalCurrentTeamsOwned,
						totalCurrentTeamsRemaining,
						moneyWonToDate,
						percentMoneyWonInvested,
						profitLoss,
						originalMoneyRemaining,
						percentMoneyRemaining
					}
				})
			);
		}
  },
  async created() {
    await this.init();
    this.isPageReady = true;
  }
}
</script>

<style scoped>
.total {
  font-weight: bold;
}

.add-to-stock-link {
  line-height: 4;
}

.section {
  padding-bottom: 10px;
}

.section-header {
  font-weight: bold;
}
</style>