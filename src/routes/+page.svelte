<script>
    let maxClick = $state(2);
    let cnt = $state(2); // tip: https://svelte.dev/docs/svelte/$state

    function onClick() {
      // tip: Since DOM (i.e., the webpage content) will automatically update based on values, [<p id="info">Remaining Number of Clicks: {cnt}</p>]
      // we only need to change the cnt number here. 
      //random comment to trigger change
      if (cnt > 0) {
        cnt--;
      }
    }
  </script>
  
  <h1>John's VIS Site</h1>
  <img
    width="200px"
    src="/images/img_1.jpg"
  />
  <div>
    You can click up to
    <select 
        bind:value={maxClick} 
        onchange={() => (cnt = maxClick)}>
      {#each [2, 4, 6] as optionNum}
        <option value={optionNum}>
          {optionNum}
        </option>
      {/each}
    </select>
    times
  </div>
  <button onclick={onClick}> Click Me </button>

  <!-- tip: use {#if...} template syntax here (https://svelte.dev/docs/svelte/if) so that the content below will automatically update when cnt value changes -->
  {#if cnt > 0}
    <!-- `the content below should only show when cnt >0` -->
    <p id="info">Remaining Number of Clicks: {cnt}</p>
  {:else}
    <!-- The content below should only shown when cnt =0 -->
    <p>No more clicks allowed</p>
  {/if}
  
  <style>
    body {
      font-family: Arial, Helvetica, sans-serif;
    }
    button {
      background-color: #44aa66;
      /* background-color: blue; */
      color: white;
      font-size: xx-large;
      padding: 10px 20px;
      border: none;
      cursor: pointer;
      border-radius: 5px;
    }
  </style>