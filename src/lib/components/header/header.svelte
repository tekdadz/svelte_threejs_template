<script>
  import { darkMode, xPlane, yPlane, zPlane, animationSpeed, lastAnimationSpeed, fullDatabaseView, showFilters, screenType, visual } from '$lib/store/store';
  
  $: expanded = $showFilters;

  // let toggleShowRightSidebar = () => {
  //   showRightSidebar.update(value => !value);
  // };
  
  let toggleFullDatabaseView = () => {
    fullDatabaseView.update(value => !value);

    if (showFilters) {
      showFilters.update(value => false);
    }
  };

  let toggleShowFilters = () => {
    showFilters.update(value => !value);
  };

  let bumpAndShuffle = () => {
    // do nothing
  };

  // Function to update animation speed
  function updateAnimationSpeed(newValue) {
    animationSpeed.update(value => newValue);
  }

  function toggleAnimations() {
    if ($animationSpeed) {
      lastAnimationSpeed.update(value => $animationSpeed);
      animationSpeed.update(value => 0);
    } else {
      animationSpeed.update(value => $lastAnimationSpeed);
    }
  }

  let toggleDarkMode = () => {
		darkMode.set(!$darkMode);
		document.querySelector(':root').classList.toggle('dark-mode');
		};

		$: darkModeClass = $darkMode ? 'darkMode hidden' : 'darkMode visible';

  // let toggleVisual = () => {
  //   if ($visual === 'space') {
  //     visual.set('roi');
  //   } else {
  //     visual.set('space');
  //   }
  // };

  // Function to toggle the visibility of a plane
  function togglePlaneVisibility(plane) {
    plane.update(current => ({ ...current, visible: !current.visible }));
  }

  // Function to update the value of a plane
  function updatePlaneValue(plane, newValue) {
    plane.update(current => ({ ...current, value: newValue, visible: true }));
  }

  // For direct access in the markup, subscribe to the stores
  $: xPlaneValue = $xPlane;
  $: yPlaneValue = $yPlane;
  $: zPlaneValue = $zPlane;

  $: xPlaneClass = $xPlane.visible ? 'visible' : 'hidden';
  $: yPlaneClass = $yPlane.visible ? 'visible' : 'hidden';
  $: zPlaneClass = $zPlane.visible ? 'visible' : 'hidden';

  $: animationsClass = $animationSpeed ? 'visible' : 'hidden';

  // $: showRightSidebarClass = $showRightSidebar ? 'visible' : 'hidden';
  $: fullDatabaseViewClass = $fullDatabaseView ? 'visible' : 'hidden';
  $: showFiltersClass = $showFilters ? 'hidden' : 'visible';

  $: visualSpaceClass = $visual === 'space' ? 'visible' : 'hidden';
  $: visualGraphClass = $visual === 'graph' ? 'visible' : 'hidden';

  $: showSidebar = $screenType == 3 && window.innerWidth > 1000 ? true : false;

</script>
<header>
  <div class="controls">
      <!-- <div class="control-group filters-control">
        <button on:click={() => bumpAndShuffle()}>
          <p class="selectable">BUMP AND SHUFFLE</p>
        </button>
      </div> -->
        <!-- TOGGLE VISUAL MODE -->
      <!-- <div class="control-group filters-control">
        <button on:click={() => toggleVisual()}>
          <p><span class={visualSpaceClass}>CUBE</span> / <span class={visualGraphClass}>ROI</span></p>
        </button>
      </div> -->
    <!-- X Plane Control -->
    <div class="control-group">
      <button on:click={() => togglePlaneVisibility(xPlane)}>
        <p><span class={xPlaneClass}>{xPlaneValue.name}</span> / <span class={xPlaneClass === 'visible' ? 'hidden' : 'visible'}>off</span></p>
      </button>
      <input type="range" bind:value={xPlaneValue.value} min={xPlaneValue.min} max={xPlaneValue.max} on:input={(e) => updatePlaneValue(xPlane, parseFloat(e.target.value))}>
      <p class="control-value"><span class={xPlaneClass}>{xPlaneValue.value}</span></p>
    </div>
    <!-- Y Plane Control -->
    <div class="control-group">
      <button on:click={() => togglePlaneVisibility(yPlane)}>
        <p><span class={yPlaneClass}>{yPlaneValue.name}</span> / <span class={yPlaneClass === 'visible' ? 'hidden' : 'visible'}>off</span></p>
      </button>
      <input type="range" bind:value={yPlaneValue.value} min={yPlaneValue.min} max={yPlaneValue.max} on:input={(e) => updatePlaneValue(yPlane, parseFloat(e.target.value))}>
      <p class="control-value"><span class={yPlaneClass}>{yPlaneValue.value}</span></p>
    </div>
    <!-- Z Plane Control -->
    <div class="control-group">
      <button on:click={() => togglePlaneVisibility(zPlane)}>
        <p><span class={zPlaneClass}>{zPlaneValue.name}</span> / <span class={zPlaneClass === 'visible' ? 'hidden' : 'visible'}>off</span></p>
      </button>
      <input type="range" bind:value={zPlaneValue.value} min={zPlaneValue.min} max={zPlaneValue.max} on:input={(e) => updatePlaneValue(zPlane, parseFloat(e.target.value))}>
      <p class="control-value"><span class={zPlaneClass}>{zPlaneValue.value}</span></p>
    </div>
    <!-- ANIMATIONS -->
    <!-- <div class="control-group filters-control">
      <button on:click={() => toggleAnimations()}>
        <p><span class={animationsClass}>ANIMATIONS</span></p>
      </button>
      <input type="range" bind:value={$animationSpeed} min=0 max=1 step=0.1 on:input={(e) => updateAnimationSpeed(parseFloat(e.target.value))}>
    </div> -->


    <!-- <div class="control-group filters-control">
      <button on:click={() => toggleDarkMode()} on:keydown={() => toggleDarkMode()}>
        <p class={darkModeClass}>toggle dark mode</p>
      </button>
    </div> -->

        <!-- FILTERS -->
    <!-- <div class="control-group filters-control">
      <button on:click={() => toggleShowFilters()}>
        <p> <span class={showFiltersClass === 'visible' ? 'hidden' : 'visible'}>FILTERS [ 0 ]</span></p>
      </button>
    </div> -->
  <!-- </div>

  {#if showSidebar}

  <div class="right-sidebar-controls"> -->

      <!-- <button class="unit" on:click={() => toggleFullDatabaseView()} on:keydown={() => toggleFullDatabaseView()}>
        <p class={fullDatabaseViewClass} >Full Databse View</p>
      </button> -->

      <!-- <button  class="unit" >
        <p class={showRightSidebarClass} on:click={() => toggleShowRightSidebar()} on:keydown={() => toggleShowRightSidebar()}>Show Results</p>
      </button> -->
      

  </div>
  <!-- {/if} -->

</header>

<main class:expanded={expanded}>
  {#if $showFilters}
    <p>filters</p>
  {/if}
</main>

<style>
  header {
    position: absolute;
    color: var(--primary);
    width: 100%;
    height: 56px;
    margin: auto;
    z-index: 1;
    border-bottom: solid 1px var(--primary-50);
    padding: 20px;
    display: flex;
    align-items: center;
    justify-content: space-around;
    backdrop-filter: blur(20px);
  }

  .control-group {
    display: flex;
    flex-direction: row;
    width: 27.5%;
    gap: 10px;
    align-items: center;
    justify-content: center;
  }

  .control-group.filters-control {
    width: 17.5%;
  }
  
  .control-value {
    width: 1em;
    text-align: right;
    white-space: nowrap;
  }

  .controls {
    display: flex;
    justify-content: space-around;
    width: 100%;

  }

  .right-sidebar-controls {
    display: flex;
    flex-flow: row nowrap;
    justify-content: space-around;
    width: 380px;
    max-width: 380px;
    min-width: 380px;
    height: 100%;
    border-left: solid 1px var(--primary-50); 
  }

  .unit {
    display: flex;
    flex-flow: column nowrap;
    justify-content: center;
    align-items: center;
    width: 17.5%;
    height: 20px;
  }

  .selectable {
    opacity: .5;
    cursor: pointer;
  }

  .selectable:hover {
    opacity: 1;
  }

  main {
    position: absolute;
    top: 56px;
    height: 0;
    width: 100%;
    width: calc(100vw - 400px);
    transition: height 0.3s ease;
    /* background-color: var(--background); */
		background-color: var(--background);
  }

  main.expanded {
    height: calc(100vh - 56px);
    transition: height 0.3s ease;
    border-bottom: solid 1px var(--primary-50);
  }

  input[type=range] {
    width: 30%;
  }

  .visible {
    opacity: 1;
  }

  .hidden {
    opacity: 0.5;
  }
  
  /* @media only screen and (max-width: 1440px) {
    header {
      left: 0;
      width: 100%;
    }
		.controls {
			width: 100%;
		}

    .right-sidebar-controls {
      width: 100%;
    }
	} */

  /* @media only screen and (max-width: 768px) {
    header {
      left: 0;
      width: 100%;
      height: 80px;
      padding: 0;
      padding-top: 5px;
    }
		.controls {
			width: 100%;
		}

    p {
      font-size: 10px;
    }

    .control-group {
      flex-flow: column wrap;
      gap: 5px;
      width: 100%;
    }

    .right-sidebar-controls {
      width: 100%;
    }

    input[type=range] {
      width: 67%;
    }
	} */
</style>