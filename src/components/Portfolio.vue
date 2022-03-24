<template>
  <div v-if="isPageReady" class="content-container">
    <div v-if="!entries">
      You currently have no entries. Click <span class="link">here</span> to browse open contests.
    </div>
    <div v-else>
      <md-card>
        <md-card-header>
          <div class="md-title">Portfolio</div>
        </md-card-header>

        <md-tabs v-if="selectedEntry" md-alignment="centered">
          <md-tab md-label="Detail" @click="showContent('detail')"></md-tab>
          <md-tab md-label="Investment Metrics" @click="showContent('summary')"></md-tab>
          <md-tab md-label="Dividend Payouts" @click="showContent('payouts')"></md-tab>
          <md-tab md-label="Rankings" @click="showContent('rankings')"></md-tab>
        </md-tabs>

        <md-card-content>
          <div class="md-layout">
            <div class="md-layout-item"></div>
            <div class="md-layout-item md-large-size-33 md-medium-size-50 md-small-size-75 md-xsmall-size-100">
              <h3 class="label">Select an Entry</h3>
              <div class="custom-select-wrapper">
                <select class="custom-select" name="basic-dropdown" v-model="selectedEntry">
                  <option :value="null">Select Entry</option>
                  <option v-for="entry in entries" :key="entry.id" :value="entry">{{entry.name}}</option>
                </select>
              </div>
            </div>
            <div class="md-layout-item"></div>
          </div>
          <div v-if="selectedEntry">
            <div v-if="successMessage" class="alert-padding alert-success">
              {{successMessage}}
              <span @click="successMessage = null"><md-icon class="fa fa-times-circle light link"></md-icon></span>
            </div>
            <portfolio-detail v-if="contentToShow === 'detail'" :entry-id="selectedEntry.id" :is-ipo-open="selectedEntry.isIpoOpen"></portfolio-detail>
            <portfolio-summary v-if="contentToShow === 'summary'" :entry-id="selectedEntry.id" :tournament-id="selectedEntry.tournamentId"></portfolio-summary>
            <div v-if="contentToShow === 'payouts'" class="payouts-container">
              <md-table v-model="tournamentTeamData" class="text-left dividend-payouts-table">
                <md-table-toolbar>
                  <h1 class="md-title">Current Payouts</h1>
                </md-table-toolbar>

                <md-table-row slot="md-table-row" slot-scope="{ item }">
                  <md-table-cell md-label="Team Name" md-sort-by="teamName">{{ item.teamName }}</md-table-cell>
                  <md-table-cell md-label="IPO Price" md-sort-by="ipoPrice">{{ item.ipoPrice | toCurrency }}</md-table-cell>
                  <md-table-cell v-if="item.seed" md-label="Seed" md-sort-by="seed">{{ item.seed }}</md-table-cell>
                  <md-table-cell v-if="item.region" md-label="Region" md-sort-by="region">{{ item.region }}</md-table-cell>
                  <md-table-cell v-if="item.milestoneData && item.milestoneData.length > 0" md-label="Wins">{{ item.milestoneData[0].wins }}</md-table-cell>
                  <md-table-cell v-for="(milestone, index) in maxMilestoneData" :key="milestone.id" :md-label="milestone.milestoneName + ' Dividend'">
                    <span v-if="item.milestoneData[index]">{{ truncateDecimals(item.milestoneData[index].dividendPrice / item.numStocksInCirculation, 2) | toCurrency }}</span>
                    <span v-else>-</span>
                  </md-table-cell>
                  <md-table-cell md-label="Total Dividend">{{ getTotalDividendAmountForTeam(item) | toCurrency }}</md-table-cell>
                </md-table-row>
              </md-table>
            </div>
            <portfolio-rankings v-if="contentToShow === 'rankings'" :entry-id="selectedEntry.id" :tournament-id="selectedEntry.tournamentId"></portfolio-rankings>
          </div>
        </md-card-content>
      </md-card>
    </div>
  </div>
</template>

<script>
import { apolloClient } from "../main";
import gql from 'graphql-tag';
import PortfolioSummary from './PortfolioSummary.vue';
import PortfolioDetail from './PortfolioDetail.vue';
import PortfolioRankings from './PortfolioRankings.vue';

export default {
  components: { PortfolioSummary, PortfolioDetail, PortfolioRankings },
  name: "Portfolio",
  data() {
    return {
      isPageReady: false,
      entries: null,
      selectedEntry: null,
      contentToShow: 'detail',
      tournamentTeamData: [],
      maxLengthOfMilestoneData: 0
    }
  },
  props: {
    entryId: {
      type: String
    },
    successMessage: {
      type: String
    }
  },
  watch: {
    async selectedEntry(val) {
      if(val) {
        await this.fetchPayoutData();
      }
    }
  },
  methods: {
    getTotalDividendAmountForTeam(tournamentTeam) {
      if(tournamentTeam.milestoneData) {
        const totalDivendend = tournamentTeam.milestoneData.reduce((result, milestoneData) => {
          return result += milestoneData.dividendPrice;
        }, 0);
        return this.truncateDecimals(totalDivendend / tournamentTeam.numStocksInCirculation, 2);
      } else {
        return 0;
      }
    },
    async fetchPayoutData() {
      const response = await apolloClient.query({
      fetchPolicy: 'no-cache',
        query: gql`
          query TournamentTeams($tournamentId: ID!) {
            tournamentTeams(tournamentId: $tournamentId) {
              id,
              teamId,
              teamName,
              ipoPrice,
              seed,
              region,
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
          tournamentId: this.selectedEntry.tournamentId
        }
      });

      this.tournamentTeamData = response.data.tournamentTeams;
      const tournamentTeamData = [...this.tournamentTeamData];
      this.maxLengthOfMilestoneData = tournamentTeamData.reduce((result, teamData) => {
        const milestoneDataLength = teamData.milestoneData ? teamData.milestoneData.length: 0;
        const max = Math.max(result, milestoneDataLength);
        if(max === milestoneDataLength) {
          this.maxMilestoneData = teamData.milestoneData;
        }
        return max;
      }, 0);
    },
    async fetchUserEntries() {
      const email = sessionStorage.getItem('sports-exchange.email');
      const response = await apolloClient.query({
        fetchPolicy: 'no-cache',
        query: gql`
          query UserEntries($email: String!) {
            userEntries(email: $email) {
              id,
              name,
              tournamentId,
              tournament {
                isIpoOpen,
                isActive,
                masterSheetUpload,
                pricingSheetUpload,
                rulesSheetUpload,
                projectedPayoutSheetUpload,
                stockPayoutSheetUpload
              }
            }
          }
        `,
        variables: {
          email
        }
      });

      this.entries = response.data.userEntries;
      this.entries = this.entries.filter(entry => entry.tournament.isActive);

      if(!this.entryId) {
        if(this.entries.length === 1) {
          this.selectedEntry = this.entries[0];
        }
      } else {
        this.selectedEntry = this.entries.find(entry => entry.id === this.entryId);
      }
    },
    showContent(content) {
      this.contentToShow = content;
    },
    truncateDecimals(number, digits) {
      const multiplier = Math.pow(10, digits);
      const adjustedNum = number * multiplier;
      const truncatedNum = Math[adjustedNum < 0 ? 'ceil' : 'floor'](adjustedNum);

      return truncatedNum / multiplier;
    }
  },
  async created() {
    await this.fetchUserEntries();
    if(this.selectedEntry) {
      await this.fetchPayoutData();
    }
    this.isPageReady = true;
  }
}
</script>

<style scoped>
.alert-padding {
  margin-top: 20px;
}

.payouts-container {
  padding-top: 20px;
  width: 100%;
}
</style>