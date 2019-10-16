<script>
  import { onMount } from "svelte";
  import Mousetrap from "mousetrap";
  import { isUninked } from "./uninked-keypresses.js";

  const debug = false;

  const modes = {
    fresh: "fresh",
    idle: "idle",
    editingText: "editingText",
    countingDown: "countingDown",
    editingCountdownMinutes: "editingCountdownMinutes"
  };

  const events = {
    return: "return",
    tab: "tab"
  };

  // Simple state machine for `mode`
  // fresh => editingText <=> countingDown <=> editingCountdownMinutes
  let mode = "fresh";
  let content = "";
  let milliseconds = 0;
  let countdownMinutes = 0;
  let lastTime = window.performance.now();
  let elapsed = 0;
  let frame;

  const s = mils => {
    return Math.floor(mils / 1000) % 60;
  };

  const m = mils => {
    return Math.floor(mils / 60 / 1000) % 60;
  };
  const h = mils => {
    return Math.floor(mils / 60 / 60 / 1000);
  };

  let setMinutes = minutes => {
    milliseconds = minutes * 60 * 1000;

    // https://svelte.dev/examples#7guis-timer
    // Invalidate the timer
    cancelAnimationFrame(frame);
    // Start the timer
    (function update() {
      frame = requestAnimationFrame(update);

      const time = window.performance.now();
      elapsed += Math.min(time - lastTime, milliseconds - elapsed);

      lastTime = time;
    })();
  };

  function returnHandler() {
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

    if (mode === modes.fresh) {
      fullscreen();
      return;
    }

    if (mode === modes.editingCountdownMinutes) {
      setMinutes(countdownMinutes);
    }

    mode = modeFrom(events.return, mode);
  }

  function tabHandler() {
    if (mode === modes.editingCountdownMinutes) {
      setMinutes(countdownMinutes);
    } else {
      countdownMinutes = 0;
      setMinutes(countdownMinutes);
    }

    mode = modeFrom(events.tab, mode);
    return false;
  }

  function modeFrom(event, oldMode) {
    if (event === events.return) {
      switch (oldMode) {
        case modes.fresh:
          return modes.editingText;

        case modes.idle:
          return modes.editingText;

        case modes.editingText:
          return modes.countingDown;

        case modes.editingCountdownMinutes:
          return modes.countingDown;

        case modes.countingDown:
          return modes.editingText;

        default:
          throw "Unknown mode";
      }
    } else if (event === events.tab) {
      switch (oldMode) {
        case modes.fresh:
          return modes.fresh;

        case modes.idle:
          return modes.editingText;

        case modes.editingText:
          return modes.editingCountdownMinutes;

        case modes.editingCountdownMinutes:
          return modes.countingDown;

        case modes.countingDown:
          return modes.editingCountdownMinutes;

        default:
          throw "Unknown mode";
      }
    }
  }

  const resetContent = () => {
    content = "";
    mode = modes.editingText;
  };

  const trimLastCharacter = () => {
    content = content.slice(0, -1);
  };

  onMount(() => {
    Mousetrap.bind(events.return, returnHandler);
    Mousetrap.bind(events.tab, tabHandler);
    Mousetrap.bind("command+backspace", resetContent);
    Mousetrap.bind("backspace", trimLastCharacter);
    setKeydownHandler();
  });

  const keydownHandler = event => {
    const key = event.key;

    if (isUninked.test(key)) return;

    if (mode === modes.editingText) {
      content = content + key;
    } else if (mode === modes.editingCountdownMinutes && /^[0-9]$/.test(key)) {
      countdownMinutes = countdownMinutes == 0 ? key : countdownMinutes + key;
    }
  };

  const setKeydownHandler = () => {
    document.addEventListener("keydown", keydownHandler);
  };

  // let stopKeyPresses = () => document.removeEventListener("keydown", keydownHandler);

  $: mainContentStyle =
    mode === modes.editingText
      ? "focussed-content"
      : mode === modes.editingCountdownMinutes
      ? "dimmed-content"
      : "idle-content";

  $: timerStyle =
    mode === modes.editingText
      ? "dimmed-timer"
      : mode === modes.editingCountdownMinutes
      ? "focussed-timer"
      : "idle-timer";
</script>

<style>
  main {
    position: absolute;
    width: 100%;
    text-align: center;
    margin: 0 auto;
    margin-top: 20vh;
  }

  .gray {
    color: lightgray;
    transition: font-size 250ms;
  }

  .help {
    position: absolute;
    bottom: 3rem;
    left: 3rem;
    font-size: 2rem;
    color: lightgray;
  }

  .timer {
    margin-top: 5rem;
  }

  .focussed-content {
    font-size: 8rem;
    transition: font-size 250ms;
  }

  .dimmed-content {
    font-size: 2rem;
    transition: font-size 250ms;
  }

  .idle-content {
    font-size: 5rem;
    transition: font-size 250ms;
  }

  .focussed-timer {
    font-size: 7rem;
    transition: font-size 250ms;
  }

  .dimmed-timer {
    font-size: 1rem;
    transition: font-size 250ms;
  }

  .idle-timer {
    font-size: 3rem;
    transition: font-size 250ms;
  }

  .debug {
    position: absolute;
    bottom: 3rem;
    right: 3rem;
    color: gray;
    width: 50%;
  }
</style>

<main>
  <div class={mainContentStyle}>
    {#if content.length > 0}
      {@html content}
    {:else}
      <span class="gray">
        What are you going to do
        <strong>right now</strong>
        ?
      </span>
    {/if}
  </div>

  <div class={`timer ${timerStyle}`}>
    {#if mode === modes.countingDown}
      {`${('0' + h(milliseconds - elapsed)).slice(-2)}:${('0' + m(milliseconds - elapsed)).slice(-2)}:${('0' + s(milliseconds - elapsed)).slice(-2)}`}
    {:else if mode === modes.editingCountdownMinutes}
      {`which should take about ${countdownMinutes} minutes`}
    {/if}
  </div>
</main>

<div class="help">Ctrl+Q to Quit</div>

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
      <strong>milliseconds:</strong>
      {milliseconds}
    </div>
    <div>
      <strong>seconds:</strong>
      {s(milliseconds)}
    </div>
    <div>
      <strong>minutes:</strong>
      {m(milliseconds)}
    </div>
    <div>
      <strong>countdownMinutes:</strong>
      {countdownMinutes}
    </div>
    <div>
      <strong>hours:</strong>
      {h(milliseconds)}
    </div>
    <div>
      <strong>elapsed:</strong>
      {elapsed}
    </div>
  </div>
{/if}
