<template>
  <div v-if="isPageReady">
    <div v-if="serverError" class="alert-error text-center">
      {{serverError}}
      <span @click="serverError = null"><md-icon class="fa fa-times-circle light link"></md-icon></span>
    </div>
    <md-table v-model="teamsInput" class="text-left">
      <md-table-row slot="md-table-row" slot-scope="{ item }">
        <md-table-cell md-label="Team" md-sort-by="name">{{ item.teamName }}</md-table-cell>
        <md-table-cell md-label="Is Eliminated">
          <md-checkbox v-model="item.isEliminated"></md-checkbox>
        </md-table-cell>
      </md-table-row>
    </md-table>

    <md-card-actions>
      <md-button :disabled="httpWait" @click="toggleElimiatedTeams" :class="{ 'btn-disabled' : httpWait }">
        Submit
        <md-progress-spinner v-if="httpWait" class="btn-spin" :md-diameter="20" :md-stroke="3" md-mode="indeterminate"></md-progress-spinner>
      </md-button>
    </md-card-actions>
  </div>
</template>

<script>
/* eslint-disable */
import { apolloClient } from "../main";
import gql from 'graphql-tag';

export default {
  name: "EliminateTeamsForm",
  data() {
    return {
      isPageReady: false,
      tournamentTeamData: null,
      serverError: null,
      teamsInput: [],
      httpWait: false
    }
  },
  props: {
    successCb: {
      type: Function
    },
    leagueId: {
      type: String
    },
    tournamentId: {
      type: String
    }
  },
  methods: {
    async toggleElimiatedTeams() {
      this.httpWait = true;
      for(let team of this.teamsInput) {
        try {
          await apolloClient.mutate({
            mutation: gql`
              mutation ToggleTournamentTeamEliminated($tournamentTeamId: ID!, $isEliminated: Boolean!) {
                toggleTournamentTeamEliminated(tournamentTeamId: $tournamentTeamId, isEliminated: $isEliminated) {
                  id
                }
              }
            `,
            variables: {
              tournamentTeamId: team.id,
              isEliminated: team.isEliminated
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
        this.successCb();
      }
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
              ipoPrice,
              isEliminated
            }
          }
        `,
        variables: {
          tournamentId: this.tournamentId
        }
      });

      this.tournamentTeamData = response.data.tournamentTeams;
    }
  },
  async created() {
    await this.fetchIpoData();
    this.teamsInput = this.tournamentTeamData.map((team) => {
      return {
        teamName: team.teamName,
        id: team.id,
        isEliminated: team.isEliminated
      }
    });
    this.isPageReady = true;
  }
}
</script>

<style scoped>
</style>