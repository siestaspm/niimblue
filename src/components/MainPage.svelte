<script lang="ts">
  import { Utils } from "@mmote/niimbluelib";
  import BrowserWarning from "$/components/basic/BrowserWarning.svelte";
  import LabelDesigner from "$/components/LabelDesigner.svelte";
  import PrinterConnector from "$/components/PrinterConnector.svelte";
  import axios from "axios";
  import { locale, locales, tr } from "$/utils/i18n";

  const appCommit = __APP_COMMIT__;
  const buildDate = __BUILD_DATE__;

  let isStandalone = Utils.getAvailableTransports().capacitorBle;

  // LOGIN STATE
  let token = sessionStorage.getItem("authToken");
  let user = sessionStorage.getItem("xpertUser");
  let user_id = sessionStorage.getItem('xpertId')
  let showLogin = !token; // manually update

  // LOGIN FORM STATE
  let username = "";
  let password = "";
  let error = "";
  let loading = false;

  async function handleLogin() {
    error = "";
    loading = true;

    const payload = {
      version_number: "2.2.6",
      Username: username,
      Password: password,
      app_name: "store"
    };

    try {
      const res = await axios.post(
        "https://pk9blqxffi.execute-api.us-east-1.amazonaws.com/xdeal/LoginXpert",
        payload
      );

      const data = res.data;
      console.log(data);

      const receivedToken = data?.XpertData?.[0]?.token;
      const receivedUserId = data?.XpertData?.[0]?.xpert_id;



      if (!receivedToken) {
        error = "Login failed: no token returned";
        return;
      }

      // SAVE TOKEN & USER
      sessionStorage.setItem("authToken", receivedToken);
      sessionStorage.setItem("xpertUser", JSON.stringify(data.XpertData[0]));
      sessionStorage.setItem("xpertId", receivedUserId);

      // UPDATE STATE MANUALLY
      token = receivedToken;
      user = data.XpertData[0];
      showLogin = false; // <- update manually, no `$:` needed

    } catch (err: any) {
      console.error(err);
      if (err.response) {
        error = err.response.data?.message || `Login failed: ${err.response.status}`;
      } else if (err.request) {
        error = "Server did not respond";
      } else {
        error = "Login error: " + err.message;
      }
    } finally {
      loading = false;
    }
  }
</script>

{#if showLogin}
  <!-- LOGIN SCREEN -->
  <div class="login-container">
    <h2>Login</h2>
    {#if error}<p class="error">{error}</p>{/if}
    <input type="text" placeholder="Username" bind:value={username} />
    <input type="password" placeholder="Password" bind:value={password} />
    <button on:click={handleLogin} disabled={loading}>
      {#if loading}Logging in...{:else}Login{/if}
    </button>
  </div>
{:else}
  <!-- MAIN APP -->
  <div class="container my-2">
    <div class="row align-items-center mb-3">
      <div class="col-md-3">
        <PrinterConnector />
      </div>
    </div>

    <div class="row">
      <div class="col">
        <BrowserWarning />
      </div>
    </div>

    <div class="row">
      <div class="col">
        <LabelDesigner />
      </div>
    </div>
  </div>
{/if}

<style>
  /* LOGIN FORM */
  .login-container {
    max-width: 400px;
    margin: 5rem auto;
    padding: 2rem;
    border: 1px solid #ddd;
    border-radius: 10px;
    text-align: center;
    background: #fff;
  }

  .login-container input {
    display: block;
    width: 100%;
    margin-bottom: 1rem;
    padding: 0.5rem;
  }

  .login-container button {
    padding: 0.5rem 1rem;
  }

  .error {
    color: red;
    margin-bottom: 1rem;
  }

  /* MAIN APP COLORS */
  .niim { color: #ff5349; }
  .blue { color: #0b7eff; }
</style>
