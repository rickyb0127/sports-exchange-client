<template>
  <div>
    <div class="md-layout">
      <div class="md-layout-item"></div>
      <md-field class="md-layout-item md-large-size-50 md-medium-size-50 md-small-size-75 md-xsmall-size-100">
        <label>Entry Team Name</label>
        <md-input v-model="createEntryInput.name"></md-input>
        <span v-if="inputError === 'name'" class="error form-error">Please enter an entry name</span>
      </md-field>
      <!-- TODO add multi select in order to add additional owners by email -->
      <div class="md-layout-item"></div>
    </div>
    <div class="md-layout">
      <div class="md-layout-item"></div>
        <md-button :disabled="httpWait" @click="createEntry()" class="md-raised md-primary" :class="{ 'btn-disabled' : httpWait }">
          Submit
          <md-progress-spinner v-if="httpWait" class="btn-spin" :md-diameter="20" :md-stroke="3" md-mode="indeterminate"></md-progress-spinner>
        </md-button>
      <div class="md-layout-item"></div>
    </div>
  </div>
</template>

<script>
import { apolloClient } from "../main";
import gql from 'graphql-tag';

export default {
  name: "TeamForm",
  data() {
    return {
      createEntryInput: {},
      inputError: null,
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
    tournamentId: {
      type: String
    }
  },
  methods: {
    async createEntry() {
      this.httpWait = true;
      try {
        this.createEntryInput.userEmails = [sessionStorage.getItem('sports-exchange.email')];
        this.createEntryInput.tournamentId = this.tournamentId;
        // TODO append userEmails if additional owners are added from multi-select
        const response = await apolloClient.mutate({
          mutation: gql`
            mutation CreateEntry($input: EntryInput!) {
              createEntry(input: $input) {
                id,
                name
              }
            }
          `,
          variables: {
            input: this.createEntryInput
          }
        });

        const entry = response.data.createEntry;
        this.successCb(entry.id);
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
  },
  async created() {
  }
}
</script>

<style scoped>
</style>