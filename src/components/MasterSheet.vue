<template>
  <div v-if="isPageReady">
    <!-- <a class="link" @click="generateXlsx()">Download</a> -->
    <md-card>
      <md-card-content>
        <md-table md-card class="text-left entry-stock-table">
          <md-table-toolbar>
            <h1 class="md-title">Entry Owned Stocks</h1>
          </md-table-toolbar>
          <div class="card-padding">
            <a class="link" @click="showRawEntryStockData = !showRawEntryStockData">
              <span v-if="!showRawEntryStockData">Show Raw Data</span>
              <span v-else>Hide Raw Data</span>
            </a>
            <div v-if="showRawEntryStockData">
              <a class="link" @click="copyEntryStockData()">Copy Stock Entry Data to Clipboard</a>
              <input ref="entry-stock-data" :value="JSON.stringify(masterSheetEntryStockData)" />
              <a class="link" @click="copyEntryTransactionData()">Copy Entry Transaction Data to Clipboard</a>
              <input ref="entry-transaction-data" :value="JSON.stringify(masterSheetTransactionData)" />
            </div>
          </div>

          <md-table-row>
            <md-table-head></md-table-head>
            <template v-for="entry in masterSheetEntryStockData">
              <md-table-head :key="entry.id">{{entry.name}}</md-table-head>
            </template>
          </md-table-row>

          <md-table-row>
            <md-table-cell>IPO Cash Spent</md-table-cell>
            <template v-for="entry in masterSheetEntryStockData">
              <md-table-cell :key="entry.id">{{entry.ipoCashSpent | toCurrency}}</md-table-cell>
            </template>
          </md-table-row>
          <md-table-row>
            <md-table-cell>Secondary Market Cash Spent</md-table-cell>
            <template v-for="entry in masterSheetEntryStockData">
              <md-table-cell :key="entry.id">{{entry.secondaryMarketCashSpent | toCurrency}}</md-table-cell>
            </template>
          </md-table-row>
          <md-table-row>
            <md-table-cell>Dividend Payout</md-table-cell>
            <template v-for="entry in masterSheetEntryStockData">
              <md-table-cell :key="entry.id">{{entry.dividendPayout | toCurrency}}</md-table-cell>
            </template>
          </md-table-row>
          <md-table-row>
            <md-table-cell>Owner(s)</md-table-cell>
            <template v-for="entry in masterSheetEntryStockData">
              <md-table-cell :key="entry.id">{{entry.ownerName}}</md-table-cell>
            </template>
          </md-table-row>
          <md-table-row>
            <md-table-cell>Email(s)</md-table-cell>
            <template v-for="entry in masterSheetEntryStockData">
              <md-table-cell :key="entry.id">{{entry.ownerEmail}}</md-table-cell>
            </template>
          </md-table-row>
          <md-table-row v-for="team in tournamentTeamData" :key="team.id">
            <md-table-cell>{{team.teamName}}</md-table-cell>
            <template v-for="entry in masterSheetEntryStockData">
              <md-table-cell :key="entry.id">{{getStockQuantity(entry, team.teamId)}}</md-table-cell>
            </template>
          </md-table-row>
        </md-table>
      </md-card-content>
    </md-card>

    <md-table md-card v-model="tournamentTeamData" class="text-left tournament-team-table">
      <md-table-toolbar>
        <h1 class="md-title">Tournament Team Data</h1>
      </md-table-toolbar>

      <md-table-row slot="md-table-row" slot-scope="{ item }">
        <md-table-cell md-label="Team Name" md-sort-by="teamName">{{ item.teamName }}</md-table-cell>
        <md-table-cell md-label="IPO Price" md-sort-by="ipoPrice">{{ item.ipoPrice | toCurrency }}</md-table-cell>
        <md-table-cell v-if="item.milestoneData && item.milestoneData.length > 0" md-label="Wins">{{ item.milestoneData[0].wins }}</md-table-cell>
        <md-table-cell v-for="(milestone, index) in maxMilestoneData" :key="milestone.id" :md-label="milestone.milestoneName + ' Dividend'">
          <span v-if="item.milestoneData[index]">{{ item.milestoneData[index].dividendPrice | toCurrency }}</span>
          <span v-else>-</span>
        </md-table-cell>
        <md-table-cell md-label="Total Dividend">{{ getTotalDividendAmountForTeam(item) | toCurrency }}</md-table-cell>
      </md-table-row>
    </md-table>
  </div>
</template>

<script>
/* eslint-disable */
import { apolloClient } from "../main";
import gql from 'graphql-tag';
// import XLSX from "xlsx";
// import { saveAs } from "file-saver";

export default {
  name: "MasterSheet",
  data() {
    return {
      isPageReady: false,
      tournamentTeamData: [],
      maxLengthOfMilestoneData: 0,
      showRawEntryStockData: false,
      maxMilestoneData: []
    }
  },
  props: {
    tournamentId: {
      type: String
    },
    masterSheetEntryStockData: {
      type: Array
    },
    masterSheetTransactionData: {
      type: Array
    }
  },
  methods: {
    // generateXlsx() {
    //   var wb = XLSX.utils.book_new();
    //   wb.Props = {
    //     Title: "Master Sheet",
    //     Subject: "Test",
    //     Author: "Fantasy Sports Stock Exchange",
    //     CreatedDate: new Date(2017,12,19)
    //   };
    //   wb.SheetNames.push("Test Sheet");
    //   var ws_data = [['hello' , 'world']];
    //   var ws = XLSX.utils.json_to_sheet(ws_data);
    //   wb.Sheets["Test Sheet"] = ws;
    //   var wbout = XLSX.write(wb, {bookType:'xlsx',  type: 'binary'});
    //   var buf = new ArrayBuffer(wbout.length); //convert s to arrayBuffer
    //   var view = new Uint8Array(buf);  //create uint8array as viewer
    //   for (var i=0; i<wbout.length; i++) view[i] = wbout.charCodeAt(i) & 0xFF; //convert to octet
    //   var blob = new Blob([buf], {type: "application/octet-stream"});
    //   saveAs(blob, "test.xlsx");
    // },
    copyEntryStockData() {
      const copyText = this.$refs["entry-stock-data"];
      copyText.select();
      copyText.setSelectionRange(0, 99999); /* For mobile devices */

      document.execCommand("copy");
    },
    copyEntryTransactionData() {
      const copyText = this.$refs["entry-transaction-data"];
      copyText.select();
      copyText.setSelectionRange(0, 99999); /* For mobile devices */

      document.execCommand("copy");
    },
    getStockQuantity(entry, teamId) {
      const entryStocks = entry.stocks;
      const index = entryStocks.findIndex(stock => stock.teamId === teamId);
      return entryStocks[index].quantity;
    },
    async fetchTournamentTeams() {
      const response = await apolloClient.query({
      fetchPolicy: 'no-cache',
        query: gql`
          query TournamentTeams($tournamentId: ID!) {
            tournamentTeams(tournamentId: $tournamentId) {
              id,
              teamId,
              teamName,
              ipoPrice,
              isEliminated,
              milestoneData {
                milestoneId,
                milestoneName,
                dividendPrice,
                wins,
                losses,
                ties
              }
            }
          }
        `,
        variables: {
          tournamentId: this.tournamentId
        }
      });

      this.tournamentTeamData = response.data.tournamentTeams;
      const tournamentTeamData =[...this.tournamentTeamData];
      this.maxLengthOfMilestoneData = tournamentTeamData.reduce((result, teamData) => {
        const milestoneDataLength = teamData.milestoneData ? teamData.milestoneData.length: 0;
        const max = Math.max(result, milestoneDataLength);
        if(max === milestoneDataLength) {
          this.maxMilestoneData = teamData.milestoneData;
        }
        return max;
      }, 0);
    },
    getTotalDividendAmountForTeam(tournamentTeam) {
      if(tournamentTeam.milestoneData) {
        return tournamentTeam.milestoneData.reduce((result, milestoneData) => {
          return result += milestoneData.dividendPrice;
        }, 0);
      } else {
        return 0;
      }
    }
  },
  async created() {
    await this.fetchTournamentTeams();
    this.isPageReady = true;
  }
}
</script>

<style scoped>
.entry-stock-table {
  margin-bottom: 20px;
}

.tournament-team-table {
  margin-top: 20px;
}

.entry-data {
  padding-bottom: 20px;
}

.entry-stock-title {
  font-weight: bold;
}

.hidden {
  z-index: -1000;
}

.card-padding {
  padding-left: 25px;
  padding-bottom: 20px;
}
</style>