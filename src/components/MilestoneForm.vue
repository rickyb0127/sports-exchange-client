<template>
  <div v-if="isPageReady">
    <md-table v-model="tournamentTeamData" class="text-left">
      <md-table-row slot="md-table-row" slot-scope="{ item }">
        <md-table-cell md-label="Team" md-sort-by="name">{{ item.teamName }}</md-table-cell>
        <md-table-cell md-label="Dividend Price" md-sort-by="dividendPrice">
          $<input :ref="'dividendPriceInput-' + item.id" @change="updateInput(item.id)" class="dividend-price-input" type="number" step="1" min="0" max="" :value="item.milestoneInput.dividendPrice">
        </md-table-cell>
        <md-table-cell md-label="Wins" md-sort-by="wins">
          <input :ref="'winsInput-' + item.id" @change="updateInput(item.id)" class="wins-input" type="number" step="1" min="0" max="" :value="item.milestoneInput.wins">
        </md-table-cell>
        <md-table-cell md-label="Losses" md-sort-by="losses">
          <input :ref="'lossesInput-' + item.id" @change="updateInput(item.id)" class="losses-input" type="number" step="1" min="0" max="" :value="item.milestoneInput.losses">
        </md-table-cell>
        <md-table-cell md-label="Ties" md-sort-by="ties">
          <input :ref="'tiesInput-' + item.id" @change="updateInput(item.id)" class="ties-input" type="number" step="1" min="0" max="" :value="item.milestoneInput.ties">
        </md-table-cell>
      </md-table-row>
    </md-table>

    <md-card-actions>
      <md-button :disabled="httpWait" @click="saveMilestoneData" class="md-primary md-raised" :class="{ 'btn-disabled' : httpWait }">
        Save Milestone Data
        <md-progress-spinner v-if="httpWait" class="btn-spin" :md-diameter="20" :md-stroke="3" md-mode="indeterminate"></md-progress-spinner>
      </md-button>
    </md-card-actions>
    <div v-if="serverError" class="alert-error text-center">
      {{serverError}}
      <span @click="serverError = null"><md-icon class="fa fa-times-circle light link"></md-icon></span>
    </div>
  </div>
</template>

<script>
import { apolloClient } from "../main";
import gql from 'graphql-tag';

export default {
  name: "MilestoneForm",
  data() {
    return {
      isPageReady: false,
      leagueTeams: null,
      tournamentTeams: [],
      inputError: null,
      tournamentTeamData: null,
      serverError: null,
      httpWait: false
    }
  },
  props: {
    formType: {
      type: String
    },
    successCb: {
      type: Function
    },
    leagueId: {
      type: String
    },
    tournamentId: {
      type: String
    },
    milestone: {
      type: Object
    }
  },
  methods: {
    async fetchTournamentTeams() {
      const response = await apolloClient.query({
      fetchPolicy: 'no-cache',
        query: gql`
          query TournamentTeams($tournamentId: ID!) {
            tournamentTeams(tournamentId: $tournamentId) {
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
              }
            }
          }
        `,
        variables: {
          tournamentId: this.tournamentId
        }
      });

      const tournamentTeamData = response.data.tournamentTeams;

      if(this.milestone.id === '1') {
        if(!tournamentTeamData[0].milestoneData || !tournamentTeamData[0].milestoneData[0]) {
          this.tournamentTeamData = tournamentTeamData.map((tournamentTeam) => {
            return {
              id: tournamentTeam.id,
              teamId: tournamentTeam.teamId,
              teamName: tournamentTeam.teamName,
              milestoneInput: {
                milestoneId: this.milestone.id,
                milestoneName: this.milestone.name,
                dividendPrice: 0,
                wins: 0,
                losses: 0,
                ties: 0
              }
            }
          });
        } else {
          const index = parseInt(this.milestone.id) - 1;
          this.tournamentTeamData = tournamentTeamData.map((tournamentTeam) => {
            const milestoneInput = {
              milestoneId: tournamentTeam.milestoneData[index].milestoneId,
              milestoneName: tournamentTeam.milestoneData[index].milestoneName,
              dividendPrice: tournamentTeam.milestoneData[index].dividendPrice,
              wins: tournamentTeam.milestoneData[index].wins,
              losses: tournamentTeam.milestoneData[index].losses,
              ties: tournamentTeam.milestoneData[index].ties
            }
            return {
              ...tournamentTeam,
              milestoneInput
            }
          });
        }
      } else {
        const nonEliminatedTeams = tournamentTeamData.filter(teamData => !teamData.isEliminated)
        const index = parseInt(this.milestone.id) - 1;
        if(!nonEliminatedTeams[0].milestoneData || nonEliminatedTeams[0].milestoneData.length <= index) {
          this.tournamentTeamData = nonEliminatedTeams.map((tournamentTeam) => {
            return {
              id: tournamentTeam.id,
              teamId: tournamentTeam.teamId,
              teamName: tournamentTeam.teamName,
              milestoneInput: {
                milestoneId: this.milestone.id,
                milestoneName: this.milestone.name,
                dividendPrice: 0,
                wins: 0,
                losses: 0,
                ties: 0
              }
            }
          });
        } else {
          this.tournamentTeamData = nonEliminatedTeams.map((tournamentTeam) => {
            const milestoneInput = {
              milestoneId: tournamentTeam.milestoneData[index].milestoneId,
              milestoneName: tournamentTeam.milestoneData[index].milestoneName,
              dividendPrice: tournamentTeam.milestoneData[index].dividendPrice,
              wins: tournamentTeam.milestoneData[index].wins,
              losses: tournamentTeam.milestoneData[index].losses,
              ties: tournamentTeam.milestoneData[index].ties
            }
            return {
              ...tournamentTeam,
              milestoneInput
            }
          });
        }
      }
    },
    async saveMilestoneData() {
      this.httpWait = true;
      for(let team of this.tournamentTeamData) {
        const parsedMilestoneInput = {
          milestoneId: team.milestoneInput.milestoneId,
          milestoneName: team.milestoneInput.milestoneName,
          dividendPrice: parseFloat(team.milestoneInput.dividendPrice),
          wins: parseInt(team.milestoneInput.wins),
          losses: parseInt(team.milestoneInput.losses),
          ties: parseInt(team.milestoneInput.ties)
        }
        const input = {
          id: team.id,
          milestoneInput: parsedMilestoneInput
        }
        try {
          await apolloClient.mutate({
          fetchPolicy: 'no-cache',
            mutation: gql`
              mutation createOrUpdateMilestoneData($input: TournamentTeamMilestoneInput!) {
                createOrUpdateMilestoneData(input: $input) {
                  id,
                  name
                }
              }
            `,
            variables: {
              input
            }
          });

        } catch(err) {
          if(err.graphQLErrors && err.graphQLErrors.length > 0) {
            this.serverError = err.graphQLErrors[0].message;
          } else {
            this.serverError = "Server Error";
          }
          this.httpWait = false;
          return err;
        }
      }
      this.successCb();
    },
    updateInput(id) {
      const dividendInputValue = this.$refs['dividendPriceInput-' + id].value;
      const winsInputValue = this.$refs['winsInput-' + id].value;
      const lossesInputValue = this.$refs['lossesInput-' + id].value;
      const tiesInputValue = this.$refs['tiesInput-' + id].value;
      const index = this.tournamentTeamData.findIndex(team => team.id === id);
      this.tournamentTeamData[index].milestoneInput.dividendPrice = dividendInputValue;
      this.tournamentTeamData[index].milestoneInput.wins = winsInputValue;
      this.tournamentTeamData[index].milestoneInput.losses = lossesInputValue;
      this.tournamentTeamData[index].milestoneInput.ties = tiesInputValue;
    }
  },
  async created() {
    // await this.fetchIpoData();
    // await this.fetchLeagueTeams();
    await this.fetchTournamentTeams();
    this.isPageReady = true;
  }
}
</script>

<style scoped>
  .dividend-price-input {
    width: 60px;
  }

  .wins-input {
    width: 60px;
  }
</style>