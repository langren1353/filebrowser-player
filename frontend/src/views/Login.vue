<template>
  <div id="login" :class="{ recaptcha: recaptcha }">
    <form @submit="submit">
      <img :src="logoURL" alt="File Browser" />
      <h1>{{ name }}</h1>
      <div v-if="error !== ''" class="wrong">{{ error }}</div>

      <input
        autofocus
        class="input input--block"
        type="text"
        autocapitalize="off"
        v-model="username"
        :placeholder="$t('login.username')"
      />
      <input
        class="input input--block"
        type="password"
        v-model="password"
        :placeholder="$t('login.password')"
      />
      <input
        class="input input--block"
        v-if="createMode"
        type="password"
        v-model="passwordConfirm"
        :placeholder="$t('login.passwordConfirm')"
      />

      <div v-if="recaptcha" id="recaptcha"></div>
      <input
        class="button button--block"
        type="submit"
        :value="createMode ? $t('login.signup') : $t('login.submit')"
      />

      <p @click="toggleMode" v-if="signup">
        {{
          createMode ? $t("login.loginInstead") : $t("login.createAnAccount")
        }}
      </p>
    </form>
  </div>
</template>

<script>
import * as auth from "@/utils/auth";
import {
  name,
  logoURL,
  recaptcha,
  recaptchaKey,
  signup,
  enableGuest,
} from "@/utils/constants";

let isFromLogOut = false;

export default {
  name: "login",
  computed: {
    enableGuest: () => enableGuest,
    signup: () => signup,
    name: () => name,
    logoURL: () => logoURL,
    redirectURL() {
      let redirect = this.$route.query.redirect;
      if (redirect === "" || redirect === undefined || redirect === null) {
        redirect = "/files/";
      }
      return redirect;
    },
  },
  data: function () {
    return {
      createMode: false,
      error: "",
      username: "",
      password: "",
      recaptcha: recaptcha,
      passwordConfirm: "",
    };
  },
  beforeRouteEnter(to, from, next) {
    if (from.path.startsWith("/files")) {
      isFromLogOut = true;
    }
    next();
  },
  async beforeMount() {
    if (this.enableGuest && !isFromLogOut) {
      try {
        await auth.login("guest", "guest", "");
        await this.$router.push({ path: this.redirectURL });
      } catch (e) {
        this.error = this.$t("login.guestLoginFail");
      }
    }
  },
  mounted() {
    if (!recaptcha) return;

    window.grecaptcha.ready(function () {
      window.grecaptcha.render("recaptcha", {
        sitekey: recaptchaKey,
      });
    });
  },
  methods: {
    toggleMode() {
      this.createMode = !this.createMode;
    },
    async submit(event) {
      event.preventDefault();
      event.stopPropagation();

      let captcha = "";
      if (recaptcha) {
        captcha = window.grecaptcha.getResponse();

        if (captcha === "") {
          this.error = this.$t("login.wrongCredentials");
          return;
        }
      }

      if (this.createMode) {
        if (this.password !== this.passwordConfirm) {
          this.error = this.$t("login.passwordsDontMatch");
          return;
        }
      }

      try {
        if (this.createMode) {
          await auth.signup(this.username, this.password);
        }

        await auth.login(this.username, this.password, captcha);
        this.$router.push({ path: this.redirectURL });
      } catch (e) {
        console.error(e);
        if (e.message === 409) {
          this.error = this.$t("login.usernameTaken");
        } else {
          this.error = this.$t("login.wrongCredentials");
        }
      }
    },
  },
};
</script>
