<script>
  import { onMount } from "svelte";
  import Mousetrap from "mousetrap";
  import { isUninked } from "./uninked-keypresses.js";

  const debug = true;

  const modes = {
    fresh: "fresh",
    idle: "idle",
    enteringText: "enteringText",
    countingDown: "countingDown",
    enteringCountdownMinutes: "enteringCountdownMinutes"
  };

  const events = {
    return: "return",
    tab: "tab"
  };

  const placeholderContent =
    "What are you going to do <strong>right now</strong>?";

  // Simple state machine for `mode`
  // fresh => enteringText <=> countingDown <=> enteringCountdownMinutes
  let mode = "fresh";
  let isUserGenerated = false;
  let content = placeholderContent;
  let countdownMinutes = 0;

  $: Mousetrap.bind("return", () => returnHandler());
  $: Mousetrap.bind("tab", () => tabHandler());

  function returnHandler() {
    mode = modeFrom(events.return, mode);
  }

  function tabHandler() {
    mode = modeFrom(events.tab, mode);
  }

  function modeFrom(event, oldMode) {
    if (event === events.return) {
      switch (oldMode) {
        case modes.fresh:
          return modes.enteringText;

        case modes.idle:
          return modes.enteringText;

        case modes.enteringText:
          return modes.countingDown;

        case modes.enteringCountdownMinutes:
          return modes.countingDown;

        case modes.countingDown:
          return modes.enteringText;

        default:
          throw "Unknown mode";
      }
    } else if (event === events.tab) {
      switch (oldMode) {
        case modes.fresh:
          return modes.fresh;

        case modes.idle:
          return modes.enteringText;

        case modes.enteringText:
          return modes.enteringCountdownMinutes;

        case modes.enteringCountdownMinutes:
          return modes.countingDown;

        case modes.countingDown:
          return modes.enteringCountdownMinutes;

        default:
          throw "Unknown mode";
      }
    }
  }

  const resetContent = () => {
    content = placeholderContent;
    mode = modes.enteringText;
    isUserGenerated = false;
  };

  onMount(() => {
    const fullscreen = () => {
      // This take 2 hits of the [return] key…no idea why!
      var docElm = document.documentElement;
      if (docElm.requestFullscreen) {
        docElm.requestFullscreen();
      } else if (docElm.mozRequestFullScreen) {
        docElm.mozRequestFullScreen();
      } else if (docElm.webkitRequestFullScreen) {
        docElm.webkitRequestFullScreen();
      }
      mode = modeFrom("return", mode);
    };

    Mousetrap.bind("return", fullscreen);
    Mousetrap.bind("command+backspace", resetContent);
    displayAndResetKeyPresses();
  });

  const keydownHandler = event => {
    const key = event.key;

    if (isUninked.test(key)) return;

    if (mode === modes.enteringText) {
      content = content === placeholderContent ? key : content + key;
      isUserGenerated = true; // This is weak. Refactor, please!
    } else if (mode === modes.enteringCountdownMinutes && /^[0-9]$/.test(key)) {
      countdownMinutes = countdownMinutes + key;
    }
  };

  const displayAndResetKeyPresses = () => {
    document.addEventListener("keydown", keydownHandler);
  };

  let stopKeyPresses = () =>
    document.removeEventListener("keydown", keydownHandler);
</script>

<style>
  main {
    font-size: 5rem;
    text-align: center;
    margin: 0 auto;
    margin-top: 20vh;
  }

  .black {
    color: black;
  }

  .gray {
    color: lightgray;
  }

  .help {
    position: absolute;
    bottom: 3rem;
    left: 3rem;
    font-size: 2rem;
    color: lightgray;
  }

  .debug {
    position: absolute;
    bottom: 3rem;
    right: 3rem;
    color: gray;
    width: 50%;
  }
</style>

<main class={isUserGenerated ? 'black' : 'gray'}>
  {@html content}
  <div class="help">Ctrl+Q to Quit</div>
</main>

{#if debug}
  <div class="debug">
    <div>
      <strong>content:</strong>
      {content}
    </div>
    <div>
      <strong>mode:</strong>
      {mode}
    </div>
    <div>
      <strong>isUserGenerated:</strong>
      {isUserGenerated}
    </div>
  </div>
{/if}
