<script>import "./app.css";
import mobotLogo from './assets/mobot.png'
import techmoLogo from './assets/techmo.png'
import Updater from './lib/Updater.svelte';</script>

<main>

  <div class="card">
    <Updater></Updater>
  </div>

    

</main>



<style>
  .logo {
    height: 4em;
    padding: 1.5em;
    will-change: filter;
    transition: filter 300ms;
  }
  .logo.techmo:hover {
    filter: drop-shadow(0 0 2em #16d75daa);
  }
  .logo.mobot:hover {
    filter: drop-shadow(0 0 2em #ffff00aa);
  }
</style>
