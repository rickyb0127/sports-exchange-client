<template>
  <div v-if="isPageReady">
    <!-- <div @click="importNewLeague()">import league</div> -->
    <!-- <leagues v-if="selectedView === 'main'" :is-admin="true" :selected-view.sync="selectedView" :league-id.sync="leagueId"></leagues>
    <league v-if="selectedView === 'league'" :is-admin="true" :selected-view.sync="selectedView" :league-id.sync="leagueId" :tournament-id.sync="tournamentId"></league>
    <tournament v-if="selectedView === 'tournament'" :is-admin="true" :selected-view.sync="selectedView" :league-id.sync="leagueId" :tournament-id.sync="tournamentId"></tournament> -->
    <h1>Admin</h1>
    <div v-if="successMessage" class="alert-success">
      {{successMessage}}
      <span @click="successMessage = null"><md-icon class="fa fa-times-circle light link"></md-icon></span>
    </div>
    <div v-if="selectedView === 'main'">
      <md-button @click="showCreateNewTournamentModal = true" class="md-raised md-primary">Create New Tournament</md-button>
      <div class="content-container">
        <md-card class="tournament-card" v-for="tournament in tournaments" v-bind:key="tournament.id">
          <div>
            <md-card-header>
              <div class="md-title">{{tournament.name}}</div>
            </md-card-header>
            <md-card-content>
              <div class="md-layout">
                <div class="md-layout-item">League: {{getLeagueName(tournament.leagueId)}}</div>
              </div>
              <div class="md-layout">
                <div class="md-layout-item">Created At: {{formatDate(parseInt(tournament.createdAt))}}</div>
              </div>
              <div class="md-layout">
                <div class="md-layout-item">Is Active: {{tournament.isActive}}</div>
              </div>
              <!-- <div class="md-layout">
                <div class="md-layout-item">Number of Entries</div>
              </div>
              <div class="md-layout">
                <div class="md-layout-item">Number of Tournament Teams</div>
              </div> -->
              <div>
                <md-button @click="goToTournamentDetails(tournament)" class="md-raised md-primary">View Details/Edit</md-button>
              </div>
            </md-card-content>
          </div>
        </md-card>
      </div>
    </div>
    <tournament v-if="selectedView === 'tournamentDetail'" :selected-view.sync="selectedView" :selected-entry.sync="selectedEntry" :league-id.sync="selectedLeagueId" :tournament-id.sync="selectedTournamentId"></tournament>
    <edit-entry-data v-if="selectedView === 'editEntry'" :selected-view.sync="selectedView" :selected-entry="selectedEntry" :league-id.sync="selectedLeagueId" :tournament-id.sync="selectedTournamentId"></edit-entry-data>

    <md-dialog :md-active.sync="showCreateNewTournamentModal">
      <md-dialog-title class="text-center">Create Tournament</md-dialog-title>
      <md-dialog-content>
        <tournament-form :form-type="'new'" :success-cb="createTournamentSuccessCb" :selectable-leagues="leagues"></tournament-form>
      </md-dialog-content>
    </md-dialog>
  </div>
</template>

<script>
import { apolloClient } from "../main";
import gql from 'graphql-tag';
import Tournament from './Tournament.vue';
import TournamentForm from './TournamentForm.vue';
import EditEntryData from './EditEntryData.vue';

export default {
  components: { Tournament, TournamentForm, EditEntryData },
  name: "Admin",
  data() {
    return {
      isPageReady: false,
      selectedView: "main",
      leagues: null,
      tournaments: null,
      selectedLeagueId: null,
      selectedTournamentId: null,
      showCreateNewTournamentModal: false,
      successMessage: null,
      selectedEntry: null
      // leagueId: null,
      // tournamentId: null
    }
  },
  props: {
    entryId: {
      type: String
    }
  },
  methods: {
    async importNewLeague() {
      await apolloClient.mutate({
        fetchPolicy: 'no-cache',
        mutation: gql`
          mutation ImportLeague($leagueName: String!) {
            importLeague(leagueName: $leagueName) {
              id,
              name
            }
          }
        `,
        variables: {
          leagueName: "ncaa-mens-basketball-march-madness-2022"
        }
      });
    },
    async fetchAllLeagues() {
      const response = await apolloClient.query({
        fetchPolicy: 'no-cache',
        query: gql`
          query Leagues {
            leagues {
              id
              name,
              defaultSettings {
                ipoBudget,
                secondaryMarketBudget,
                milestones {
                  name
                },
                regions
              },
              tournaments {
                id,
                name,
                isIpoOpen,
                leagueId,
                createdAt,
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
      });

      this.leagues = response.data.leagues;
      this.tournaments = this.leagues.flatMap(league => league.tournaments);
    },
    formatDate(timestamp) {
      const date = new Date(timestamp).toLocaleString('en-US', {
        timeZone: 'America/New_York',
        weekday: 'short', // long, short, narrow
        day: 'numeric', // numeric, 2-digit
        year: 'numeric', // numeric, 2-digit
        month: 'long', // numeric, 2-digit, long, short, narrow
        hour: 'numeric', // numeric, 2-digit
        minute: 'numeric' // numeric, 2-digit
      })
      return date;
    },
    goToTournamentDetails(tournament) {
      this.selectedLeagueId = tournament.leagueId;
      this.selectedTournamentId = tournament.id;
      this.selectedView = "tournamentDetail";
    },
    getLeagueName(leagueId) {
      const selectedLeague = this.leagues.find(league => league.id === leagueId);
      return selectedLeague.name;
    },
    showContent(content) {
      this.contentToShow = content;
    },
    async createTournamentSuccessCb() {
      this.showCreateNewTournamentModal = false;
      this.successMessage = "Successfully created new tournament!";
      await this.fetchAllLeagues();
    }
  },
  async created() {
    if(this.$auth) {
      setTimeout(async () => {
        const rolesList = this.$auth.user['https://sports-exchange/roles'];
        this.isAdmin = rolesList.includes('ADMIN');
        await this.fetchAllLeagues();
        this.isPageReady = true;
      }, 1000)
    }
  }
}
</script>

<style scoped>
.custom-select {
  position: relative;
  display: flex;
  flex-direction: column;
  border: 1px solid black;
  width: 100%;
}

.custom-select select {
  padding-left: 10px;
  background-color: transparent;
  font-size: 18px;
  outline: none;
  appearance: none;
  cursor: pointer;
}

.custom-select-wrapper {
  position: relative;
  user-select: none;
  width: 100%;
}

.tournament-card {
  margin-bottom: 20px;
}
</style>