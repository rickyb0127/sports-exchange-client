<template>
  <div v-if="isPageReady" class="content-container">
    <md-card>
      <div v-if="successMessage" class="alert-success">
        {{successMessage}}
        <span @click="successMessage = null"><md-icon class="fa fa-times-circle light link"></md-icon></span>
      </div>
      <md-card-header>
        <div class="md-title">{{league.name}}</div>
      </md-card-header>

      <md-card-content>
        <div v-if="tournaments && tournaments.length > 0">
          <md-table v-model="tournaments" class="text-left">
            <md-table-row slot="md-table-row" slot-scope="{ item }">
              <md-table-cell md-label="Tournament Name" md-sort-by="name">{{ item.name }}</md-table-cell>
              <md-table-cell md-label="Created At" md-sort-by="createdAt">{{getReadableDate(item.createdAt)}}</md-table-cell>
              <md-table-cell v-if="!isAdmin" md-label="My Entries">
                <div v-if="userHasTournamentEntries(item.id)">
                  <md-button class="md-raised md-secondary" @click="goToEntry(userHasTournamentEntries(item.id))">View</md-button>
                </div>
                <div v-else>
                  Not Entered
                </div>
              </md-table-cell>
              <md-table-cell v-if="!isAdmin" md-label=""><md-button class="md-raised md-primary" @click="showJoinTournamentForm(item)">Join</md-button></md-table-cell>
              <md-table-cell v-if="isAdmin" md-label=""><md-button class="md-raised md-primary" @click="goToTournament(item.id)">Edit</md-button></md-table-cell>
            </md-table-row>
          </md-table>
        </div>
        <div v-else>
          There are currently no active tournaments. Please contact your commissioner to add a new tournament
        </div>
      </md-card-content>

      <md-card-actions>
        <div v-if="isAdmin">
          <md-button class="md-primary" @click="showCreateTournamentModal = true">Create New Tournament</md-button>
          <md-button class="md-primary" @click="showCreateTeamsModal = true">Create Teams for League</md-button>
        </div>
      </md-card-actions>
    </md-card>
    
    <md-dialog v-if="showCreateTournamentEntryModal" :md-active.sync="showCreateTournamentEntryModal" :md-fullscreen="false">
      <md-dialog-title>Create Entry for {{selectedTournament.name}}</md-dialog-title>
      <md-dialog-content>
        <entry-form :form-type="'create'" :success-cb="createEntrySuccessCb" :tournament-id="selectedTournament.id"></entry-form>
      </md-dialog-content>
    </md-dialog>
    <md-dialog v-if="isAdmin" :md-active.sync="showCreateTournamentModal" :md-fullscreen="false">
      <md-dialog-title>Create Tournament</md-dialog-title>
      <md-dialog-content>
        <tournament-form :form-type="'create'" :success-cb="successCb" :league-id="leagueId"></tournament-form>
      </md-dialog-content>
    </md-dialog>
    <md-dialog v-if="isAdmin" :md-active.sync="showCreateTeamsModal" :md-fullscreen="false">
      <md-dialog-title>Create Teams</md-dialog-title>
      <md-dialog-content>
        <league-team-form :form-type="'create'" :success-cb="createTeamsSuccessCb" :league-id="leagueId"></league-team-form>
      </md-dialog-content>
    </md-dialog>
  </div>
</template>

<script>
import { apolloClient } from "../main";
import gql from 'graphql-tag';
import TournamentForm from './TournamentForm.vue';
import LeagueTeamForm from './LeagueTeamForm.vue';
import EntryForm from './EntryForm.vue';

export default {
  components: { TournamentForm, LeagueTeamForm, EntryForm },
  name: "League",
  data() {
    return {
      isPageReady: false,
      league: null,
      showCreateTournamentModal: false,
      showCreateTeamsModal: false,
      showCreateTournamentEntryModal: false,
      successMessage: null,
      tournaments: null,
      selectedTournament: null,
      userEntries: null,
      tournamentEntries: {}
    }
  },
  props: {
    leagueId: {
      type: String
    },
    isAdmin: {
      type: Boolean
    }
  },
  methods: {
    async fetchLeagueTournaments() {
      const response = await apolloClient.query({
        fetchPolicy: 'no-cache',
        query: gql`
          query GetTournamentsByLeagueId($leagueId: ID!) {
            getTournamentsByLeagueId(leagueId: $leagueId) {
              id
              name,
              createdAt
            }
          }
        `,
        variables: {
          leagueId: this.leagueId
        }
      });

      this.tournaments = response.data.getTournamentsByLeagueId;
    },
    goToEntry(entryId) {
      this.$router.push({ 
        name: "Portfolio",
        params: {
          entryId
        }
      });
    },
    goToTournament(tournamentId) {
      if(this.isAdmin) {
        this.$emit(`update:selected-view`, 'tournament');
        this.$emit(`update:league-id`, this.leagueId);
        this.$emit(`update:tournament-id`, tournamentId);
      }
    },
    async getTournamentEntries() {
      if(this.tournaments && this.tournaments.length > 0) {
        for(let tournament of this.tournaments) {
          const response = await apolloClient.query({
            fetchPolicy: 'no-cache',
            query: gql`
              query EntriesByTournamentId($tournamentId: ID!) {
                entriesByTournamentId(tournamentId: $tournamentId) {
                  id,
                  name,
                  tournamentId
                }
              }
            `,
            variables: {
              tournamentId: tournament.id
            }
          });

          const entries = response.data.entriesByTournamentId;

          this.tournamentEntries[tournament.id] = entries;
        }
      }
    },
    async fetchUserEntries() {
      const response = await apolloClient.query({
        fetchPolicy: 'no-cache',
        query: gql`
          query UserEntries($email: String!) {
            userEntries(email: $email) {
              id,
              name,
              tournamentId
            }
          }
        `,
        variables: {
          email: sessionStorage.getItem('sports-exchange.email')
        }
      });

      this.userEntries = response.data.userEntries;
    },
    getReadableDate(timestamp) {
      const date = new Date(parseInt(timestamp));
      return `${date.getMonth()+1}/${date.getDate()}/${date.getFullYear()}`;
    },
    async createTeamsSuccessCb() {
      this.showCreateTeamsModal = false;
      await this.fetchLeagueTournaments();
      this.successMessage = "Successfully created new tournament teams!";
    },
    async createEntrySuccessCb() {
      this.showCreateTournamentEntryModal = false;
      this.selectedTournament = null;
      await this.fetchLeagueTournaments();
      await this.fetchUserEntries();
      await this.getTournamentEntries();
      this.successMessage = "Successfully registered entry for tournament!";
    },
    showJoinTournamentForm(selectedTournament) {
      this.selectedTournament = selectedTournament;
      this.showCreateTournamentEntryModal = true;
    },
    async successCb() {
      this.showCreateTournamentModal = false;
      await this.fetchLeagueTournaments();
      this.successMessage = "Successfully created new tournament!";
    },
    userHasTournamentEntries(tournamentId) {
      const tournamentEntries = this.tournamentEntries[tournamentId];

      if((this.userEntries && this.userEntries.length > 0) && (tournamentEntries && tournamentEntries.length > 0)) {
        const userEntryIds = this.userEntries.map(userEntry => userEntry.id);
        const intersectingUserTournamentEntries = tournamentEntries.filter(entry => userEntryIds.includes(entry.id));
        if(intersectingUserTournamentEntries.length > 0) {
          return intersectingUserTournamentEntries[0].id;
        }
      }

      return false;
    }
  },
  async created() {
    const response = await apolloClient.query({
      fetchPolicy: 'no-cache',
      query: gql`
        query League($id: ID!) {
          league(id: $id) {
            id
            name
            teams {
              id
              name
            }
          }
        }
      `,
      variables: {
        id: this.leagueId
      }
    });

    this.league = response.data.league;
    await this.fetchLeagueTournaments();
    await this.fetchUserEntries();
    await this.getTournamentEntries();
    this.isPageReady = true;
  }
}
</script>

<style scoped>
</style>