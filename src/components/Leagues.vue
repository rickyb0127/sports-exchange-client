<template>
  <div v-if="isPageReady" class="content-container">
    <md-card>
      <div v-if="successMessage" class="alert-success">
        {{successMessage}}
        <span @click="successMessage = null"><md-icon class="fa fa-times-circle light link"></md-icon></span>
      </div>
      <md-card-header>
        <div class="md-title">Leagues</div>
      </md-card-header>

      <md-card-content>
        <div v-if="leagues && leagues.length > 0">
          <div v-for="league in leagues" :key="league.id" class="leagues-list link" @click="goToLeague(league.id)">
            {{league.name}}
          </div>
        </div>
        <div v-else>
          There are currently no active leagues. Please contact your commissioner to add your league
        </div>
      </md-card-content>

      <md-card-actions>
        <md-button v-if="isAdmin" class="md-primary" @click="showCreateLeagueModal = true">Create New League</md-button>
      </md-card-actions>
    </md-card>
    
    <md-dialog v-if="isAdmin" :md-active.sync="showCreateLeagueModal" :md-fullscreen="false">
      <md-dialog-title>Create League</md-dialog-title>
      <md-dialog-content>
        <league-form :form-type="'create'" :success-cb="successCb"></league-form>
      </md-dialog-content>
    </md-dialog>
  </div>
</template>

<script>
import { apolloClient } from "../main";
import gql from 'graphql-tag';
import LeagueForm from './LeagueForm.vue';

export default {
  components: { LeagueForm },
  name: "Leagues",
  data() {
    return {
      isPageReady: false,
      showCreateLeagueModal: false,
      leagues: null,
      successMessage: null
    }
  },
  props: {
    isAdmin: {
      type: Boolean
    }
  },
  methods: {
    async fetchLeagues() {
      const response = await apolloClient.query({
        fetchPolicy: 'no-cache',
        query: gql`
          query Leagues {
            leagues {
              id
              name
              teams {
                id
                name
              }
            }
          }
        `
      });

      this.leagues = response.data.leagues;
    },
    goToLeague(leagueId) {
      if(!this.isAdmin) {
        this.$router.push({ 
          name: "League",
          params: {
            leagueId
          }
        });
      } else {
        this.$emit(`update:selected-view`, 'league');
        this.$emit(`update:league-id`, leagueId);
      }
    },
    successCb() {
      this.showCreateLeagueModal = false;
      this.fetchLeagues();
      this.successMessage = "Successfully created new league!";
    }
  },
  async created() {
    await this.fetchLeagues();
    this.isPageReady = true;
  }
}
</script>

<style scoped>
</style>