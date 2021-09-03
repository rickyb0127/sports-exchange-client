<template>
  <div>
    <div class="md-layout">
      <div class="md-layout-item"></div>
      <div>
        <md-field v-for="team in leagueTeams" :key="team.id" class="md-layout-item md-large-size-50 md-medium-size-50 md-small-size-75 md-xsmall-size-100">
          <label>Name</label>
          <md-input v-model="team.name"></md-input>
          <span v-if="team.id === leagueTeams.length" @click="addTeamRow(team.id + 1)"><md-icon class="fa fa-plus link"></md-icon></span>
          <span v-if="inputError === 'name'" class="error form-error">Please enter a team name</span>
        </md-field>
      </div>
      <div class="md-layout-item"></div>
    </div>
    <div class="md-layout">
      <div class="md-layout-item"></div>
        <md-button @click="createLeagueTeams()" class="md-raised md-primary">Submit</md-button>
      <div class="md-layout-item"></div>
    </div>
  </div>
</template>

<script>
import { apolloClient } from "../main";
import gql from 'graphql-tag';

export default {
  name: "LeagueTeamForm",
  data() {
    return {
      leagueTeams: [{
        id: 1,
        name: ""
      }],
      inputError: null
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
  },
  methods: {
    addTeamRow(newId) {
      this.leagueTeams.push({
        id: newId,
        name: ""
      })
    },
    async createLeagueTeams() {
      this.leagueTeams = this.leagueTeams.filter( team => team.name );

      for(let team of this.leagueTeams) {
        const input = {
          name: team.name,
          leagueId: this.leagueId
        };
        await apolloClient.mutate({
          mutation: gql`
            mutation CreateTeam($input: CreateTeamInput!) {
              createTeam(input: $input) {
                id,
                name
              }
            }
          `,
          variables: {
            input
          }
        });
      }

      this.successCb();
    }
  },
  async created() {
  }
}
</script>

<style scoped>
</style>