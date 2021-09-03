<template>
  <div>
  </div>
</template>

<script>
import { apolloClient } from "../main";
import gql from 'graphql-tag';

export default {
  name: "Login",
  data() {
    return {
      authUrl: "https://dev-8duzx03a.us.auth0.com/api/v2/"
    }
  },
  methods: {
    async createUser() {
      const response = await apolloClient.mutate({
        mutation: gql`
          mutation CreateUser($userInput: UserInput!) {
            createUser(input: $userInput) {
              id,
              firstname,
              lastname,
              email
            }
          }
        `,
        variables: {
          userInput: this.userInput
        }
      });

      const user = response.data.createUser;
      sessionStorage.setItem('sports-exchange.email', user.email);
      this.$router.push({
        name: "Home"
      });
    },
    async login() {
      const email = this.$auth.user.email;
      const username = this.$auth.user.nickname;
      // this is due to weird rules around auth0 and fetching metadata, the namespace needs to be in url format
      const firstname = this.$auth.user['https://sports-exchange/firstname'] ? this.$auth.user['https://sports-exchange/firstname'] : "exchange";
      const lastname = this.$auth.user['https://sports-exchange/lastname'] ? this.$auth.user['https://sports-exchange/lastname'] : "user";
      const phoneNumber = this.$auth.user['https://sports-exchange/phone'];

      this.userInput = {
        firstname,
        lastname,
        cash: 0,
        email,
        username,
        phoneNumber
      }
      await this.createUser();
    }
  },
  created() {
    if(this.$auth && !this.$auth.loading && !this.$auth.isAuthenticated) {
      this.$auth.loginWithRedirect();
    } else {
      this.login();
    }
  }
}
</script>

<style scoped>
</style>