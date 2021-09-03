<template>
  <div>
    <div class="md-layout">
      <div class="md-layout-item"></div>
      <md-field class="md-layout-item md-large-size-50 md-medium-size-50 md-small-size-75 md-xsmall-size-100">
        <label>Name</label>
        <md-input v-model="league.name"></md-input>
        <span v-if="inputError === 'name'" class="error form-error">Please enter a league name</span>
      </md-field>
      <div class="md-layout-item"></div>
    </div>
    <div class="md-layout">
      <div class="md-layout-item"></div>
        <md-button @click="createLeague()" class="md-raised md-primary">Submit</md-button>
      <div class="md-layout-item"></div>
    </div>
  </div>
</template>

<script>
import { apolloClient } from "../main";
import gql from 'graphql-tag';

export default {
  name: "LeagueForm",
  data() {
    return {
      league: {},
      inputError: null
    }
  },
  props: {
    formType: {
      type: String
    },
    successCb: {
      type: Function
    }
  },
  methods: {
    async createLeague() {
      const response = await apolloClient.mutate({
        mutation: gql`
          mutation CreateLeague($input: CreateLeagueInput!) {
            createLeague(input: $input) {
              id,
              name
            }
          }
        `,
        variables: {
          input: this.league
        }
      });

      if(response) {
        this.successCb();
      }
    }
  },
  async created() {
  }
}
</script>

<style scoped>
</style>