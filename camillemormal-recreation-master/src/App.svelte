<script>
  // @ts-nocheck
  import { state } from "./stores.js";
  import NavButton from "./NavButton.svelte";
  import { flip } from "svelte/animate";
  import { expoOut, quintOut } from "svelte/easing";
  import { spring } from "svelte/motion";
  import { onDestroy } from "svelte";

  let imgs = [
    "falafel-2.png",
    "carousel-2.png",
    "carousel-3.png",
    "carousel-4.png",
    "carousel-5.png",
    "carousel-6.png",
    "carousel-1.png",
    "Falafel-38.png",
    "falafel-index-64e613f5e0f10.png",
    "TOA109_18-1.png",
    "Untitled design (11).png",
  ];
  imgs = imgs.map((s) => "/imgs/" + s);

  let active = 0;
  let skipTransition = [];
  let delay = false;
  let carouselScroll = 0;
  const bgPosMax = 180;
  let innerWidth = 1,
    innerHeight = 1;
  $: ar = innerWidth / innerHeight;
  $: hvw = innerWidth / 2;
  $: d = ar > 16 / 9 ? (innerHeight / 2) * 0.76 : hvw * 0.48;

  // ========== Transition Guard ==========
  let transitioning = false;
  let transitionTimeout = null;
  // Becomes true only after entry animation fully completes AND smooth loop settles
  let entryComplete = false;

  // ========== Smooth Drag Scrolling ==========
  let mouseDownAt = 0;
  let isDragging = false;
  let hasDragged = false;
  let dragPrevScroll = 0;
  let targetScroll = 0;
  let currentScroll = 0;
  let rafId = null;
  let isSmoothing = false;
  let isDragSmoothing = false;

  const LERP_FACTOR = 0.07;
  const DRAG_THRESHOLD = 3;
  const DRAG_SPEED = 2.5;

  $: maxScroll = Math.max(0, (imgs.length - 1) * d);

  function clampScroll(val) {
    return Math.max(0, Math.min(maxScroll, val));
  }

  function smoothLoop() {
    const diff = targetScroll - currentScroll;
    currentScroll += diff * LERP_FACTOR;
    carouselScroll = currentScroll;

    if (Math.abs(diff) > 0.15) {
      rafId = requestAnimationFrame(smoothLoop);
    } else {
      currentScroll = targetScroll;
      carouselScroll = targetScroll;
      rafId = null;
      isSmoothing = false;
      isDragSmoothing = false;
      // Mark entry as complete when smooth loop settles after transitioning ends
      if (!transitioning && $state === "carousel") {
        entryComplete = true;
      }
    }
  }

  function startSmooth(fromDrag = false) {
    isSmoothing = true;
    if (fromDrag) isDragSmoothing = true;
    if (!rafId) {
      rafId = requestAnimationFrame(smoothLoop);
    }
  }

  function stopSmooth() {
    if (rafId) {
      cancelAnimationFrame(rafId);
      rafId = null;
    }
    isSmoothing = false;
    isDragSmoothing = false;
  }

  function syncScroll(val) {
    stopSmooth();
    targetScroll = val;
    currentScroll = val;
    carouselScroll = val;
  }

  // Only use fast transitions after entry is fully complete
  $: useFastTransition =
    entryComplete && isDragSmoothing && $state === "carousel";

  function enterCarousel() {
    $state = "carousel";
    syncScroll(active * d);
    delay = true;
    transitioning = true;
    entryComplete = false;
    clearTimeout(transitionTimeout);
    transitionTimeout = setTimeout(() => {
      transitioning = false;
      // If not smoothing, mark entry complete immediately
      if (!isSmoothing) {
        entryComplete = true;
      }
      // Otherwise, smoothLoop will set entryComplete when it settles
    }, 1500);
  }

  function handlePointerDown(e) {
    isDragging = true;
    hasDragged = false;
    mouseDownAt = e.clientX;

    if ($state === "carousel") {
      dragPrevScroll = targetScroll;
    } else if ($state === "landing") {
      // Prepare for potential drag-to-carousel
      dragPrevScroll = active * d;
    }
  }

  function handlePointerMove(e) {
    if (!isDragging) return;

    const delta = mouseDownAt - e.clientX;

    if (Math.abs(delta) > DRAG_THRESHOLD) {
      hasDragged = true;

      // If we're in landing and user drags, transition to carousel
      if ($state === "landing") {
        enterCarousel();
        // Reset drag origin to current position so drag continues smoothly
        dragPrevScroll = active * d;
      }
    }

    if ($state === "carousel" && hasDragged) {
      targetScroll = clampScroll(dragPrevScroll + delta * DRAG_SPEED);
      startSmooth(true);
    }
  }

  function handlePointerUp() {
    isDragging = false;
  }

  onDestroy(() => {
    stopSmooth();
    clearTimeout(transitionTimeout);
  });
  // =============================================

  function setActive(n) {
    if ($state === "landing" && n >= 0 && n <= imgs.length - 1) {
      active = n;
    }
  }

  function handleKey(e) {
    if (e.key === "ArrowLeft" || e.key === "ArrowUp") setActive(active - 1);
    if (e.key === "ArrowRight" || e.key === "ArrowDown") setActive(active + 1);
  }

  function handleScroll(e) {
    switch ($state) {
      case "landing":
        enterCarousel();
        break;
      case "carousel":
        targetScroll = clampScroll(targetScroll + e.deltaY * 0.5);
        startSmooth(true);
        break;
    }
  }

  function calcCarouselX(i, carouselScroll, ar) {
    let r =
      ar > 16 / 9
        ? `calc(50% + ${i} * (50vh * 0.76) - 50vw - ${carouselScroll}px)`
        : `calc(50% + ${i} * 24vw - 50vw - ${carouselScroll}px)`;
    return r;
  }

  $: if (skipTransition.length) {
    requestAnimationFrame(() => (skipTransition = []));
  }
  $: if (delay) {
    requestAnimationFrame(() => (delay = false));
  }

  function calcTranslate(i, active, state) {
    let r = 0;
    if (i > active) r = 100;
    else if (i < active) r = -100;
    return `${r}%`;
  }
</script>

<svelte:window
  on:keydown={handleKey}
  on:wheel={handleScroll}
  on:pointerdown={handlePointerDown}
  on:pointermove={handlePointerMove}
  on:pointerup={handlePointerUp}
  bind:innerWidth
  bind:innerHeight
/>

<main
  class="mx-auto bg-zinc-900 text-zinc-50 h-screen"
  style:cursor={$state === "carousel"
    ? isDragging
      ? "default"
      : "default"
    : "default"}
  style:touch-action={$state === "carousel" || isDragging ? "none" : "auto"}
  class:select-none={isDragging}
>
  <div
    class="overflow-hidden duration-[1.5s] ease-out-smooth relative h-screen w-screen translate-x-12 flex items-center justify-center"
    style:transform={`translateX(${$state === "carousel" ? calcCarouselX(active, carouselScroll, ar) : "0"})`}
    style:transition-property={skipTransition[active]
      ? "none"
      : "transform, background-position, clip-path"}
    style:transition-duration={useFastTransition ? "0.05s" : "1.5s"}
  >
    {#each imgs as src, i (src)}
      <!-- svelte-ignore a11y-click-events-have-key-events -->
      <div
        style:background-image="url({src})"
        class="z-10 h-screen w-screen shrink-0 bg-center bg-no-repeat absolute duration-[1.5s] ease-out-smooth"
        style:transition-property={skipTransition[i]
          ? "none"
          : "width, height, transform, background-position, background-size"}
        style:transition-duration={useFastTransition ? "0.05s" : "1.5s"}
        style:width={$state === "carousel"
          ? ar > 16 / 9
            ? "35vh"
            : "22vw"
          : "100vw"}
        style:height={$state === "carousel"
          ? ar > 16 / 9
            ? "50vh"
            : "31.43vw"
          : "100vh"}
        style:display={$state === "carousel" && active !== i ? "none" : "block"}
        style:background-size={$state === "carousel"
          ? ar > 3 / 2
            ? "80vw 53.33vw"
            : "120vh 80vh"
          : ar > 3 / 2
            ? "100vw 66.66vw"
            : "150vh 100vh"}
        style:transform={`translateX(${calcTranslate(i, active, $state)})`}
        style:background-position={$state === "carousel"
          ? `calc(50% + (${bgPosMax}px * ${i * (d / hvw) + -carouselScroll / hvw}))`
          : i > active
            ? "-50vw"
            : i < active
              ? "50vw"
              : "center"}
        style:z-index={Math.max(0, active - i)}
        style:will-change={$state === "carousel" && isSmoothing
          ? "transform"
          : "auto"}
        on:click={() => {
          if ($state === "carousel") {
            if (hasDragged) return;
            setActive(i);
            stopSmooth();
            $state = "landing";
            delay = true;
          }
        }}
      ></div>
    {/each}
  </div>

  {#if $state === "landing"}
    <NavButton dir="left" {active} on:click={() => setActive(active - 1)} />
  {/if}
  {#if $state === "landing"}
    <NavButton dir="right" {active} on:click={() => setActive(active + 1)} />
  {/if}

  <div class="z-30 flex gap-1">
    {#each imgs as src, i (src)}
      <!-- svelte-ignore a11y-click-events-have-key-events -->
      <div
        style:background-image="url({src})"
        class="shrink-0 bg-no-repeat absolute right-0 bottom-0 duration-[1.5s] ease-out-smooth linear z-30"
        style:transition-property={skipTransition[i]
          ? "none"
          : "clip-path, transform, width, height, background-position, background-size"}
        style:transition-duration={useFastTransition && active !== i
          ? "0.05s"
          : "1.5s"}
        style:transition-delay={delay ? i * 25 + "ms" : null}
        style:background-size={$state === "carousel" && active !== i
          ? ar > 3 / 2
            ? "80vw 53.33vw"
            : "120vh 80vh"
          : ar > 3 / 2
            ? "64px 36px"
            : "64px 36px"}
        style:width={$state === "carousel" && active !== i
          ? ar > 16 / 9
            ? "35vh"
            : "22vw"
          : "64px"}
        style:height={$state === "carousel" && active !== i
          ? ar > 16 / 9
            ? "50vh"
            : "31.43vw"
          : "36px"}
        style:transform={$state === "carousel"
          ? active === i
            ? `translate(-${48 + 68 * (imgs.length - 1 - i)}px, 40px)`
            : `translate(calc(${calcCarouselX(i, carouselScroll, ar)}), calc(-50vh + 50%))`
          : `translate(-${48 + 68 * (imgs.length - 1 - i)}px, -48px)`}
        style:background-position={$state === "carousel" && active !== i
          ? `calc(50% + (${bgPosMax}px * ${i * (d / hvw) + -carouselScroll / hvw}))`
          : "center"}
        style:will-change={$state === "carousel" && isSmoothing && active !== i
          ? "transform"
          : "auto"}
        on:click={() => {
          switch ($state) {
            case "landing":
              let distance = Math.abs(active - i);
              while (--distance > 0) {
                skipTransition[Math.max(i, active) - distance] = true;
              }
              active = i;
              break;
            case "carousel":
              if (hasDragged) return;
              stopSmooth();
              skipTransition = Array(imgs.length);
              skipTransition.fill(true);
              active = i;
              requestAnimationFrame(() => {
                $state = "landing";
                delay = true;
              });
              break;
          }
        }}
      ></div>
    {/each}
  </div>
</main>

<style>
</style>
