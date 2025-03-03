<style lang="scss">
  .noselection {
    -webkit-touch-callout: none; /* iOS Safari */
    -webkit-user-select: none; /* Safari */
    -khtml-user-select: none; /* Konqueror HTML */
    -moz-user-select: none; /* Old versions of Firefox */
    -ms-user-select: none; /* Internet Explorer/Edge */
    user-select: none; /* Non-prefixed version, currently
									  supported by Chrome, Edge, Opera and Firefox */
  }
  .wrapper {
    position: absolute;
    left: 0;
    right: 0;
    top: 0;
    bottom: 0;
  }
  .btn {
    display: grid;
    justify-items: center;
    align-items: center;
    position: absolute;
    width: 50px;
    height: 50px;
    border-radius: 50%;
    font-family: 'Oswald', sans-serif;
    background: transparent;
  }

  .page {
    left: 0;
    right: 0;
    top: 0;
    bottom: 0;
    position: absolute;
    display: grid;
  }
  .page > * {
    width: 100%;
    height: 100%;
  }
  svg {
    position: absolute;
    left: 0;
    top: 0;
    width: 100%;
    height: 100%;
    z-index: 1;
    pointer-events: none;
  }
</style>

<script lang="ts">
  import { createEventDispatcher, onMount, tick } from 'svelte'
  import { cubicOut } from 'svelte/easing'
  import { spring, tweened, type Spring, type Tweened } from 'svelte/motion'

  export let width: number
  export let height: number
  export let x: number
  export let y: number
  export let btnText = '>'
  export let btnColor = '#fff'
  export let btnBorderColor = 'rgba(255,255,255,0.5)'
  export let step = 10
  export let trigger = 250
  export let precision = 3
  export let slideDuration = 1000
  export let slideEasing = cubicOut
  export let buttonSize = 10
  export let veilExpansionStiffness = 0.03
  export let veilExpansionDamping = 0.1
  export let buttonExpansionStiffness = 0.05
  export let buttonExpansionDamping = 0.2
  export let enabled = true

  const emit = createEventDispatcher()
  const IDLE = 0,
    START_SLIDING = 1,
    SLIDING = 2,
    DONE = 3

  function uuid(short = false) {
    let dt = new Date().getTime()
    const BLUEPRINT = short ? 'xyxxyxyx' : 'xxxxxxxx-xxxx-yxxx-yxxx-xxxxxxxxxxxx'
    const RESULT = BLUEPRINT.replace(/[xy]/g, function (c) {
      let r = (dt + Math.random() * 16) % 16 | 0
      dt = Math.floor(dt / 16)
      return (c == 'x' ? r : (r & 0x3) | 0x8).toString(16)
    })
    return RESULT
  }

  let id = `clip-${uuid()}`
  let status: typeof IDLE | typeof START_SLIDING | typeof SLIDING | typeof DONE = IDLE
  let active = false

  let btn: HTMLDivElement
  let slide: Tweened<number>
  let tip: Spring<{
    x: number
    y: number
  }>
  let size: Spring<{
    buttonSize: number
  }>
  let originalTipPosition: Tweened<{
    x: number
    y: number
  }>
  let path: string
  let mounted: boolean
  let btnOpacity: number
  let base: number
  let lastX: number
  let lastY: number
  let rendering: boolean
  let currentAnimationFrameRequest: number

  function render() {
    if (DONE === status) return

    let x = $tip.x < $originalTipPosition.x ? $originalTipPosition.x : $tip.x
    rendering = true
    let y = height + x - $tip.y
    let s

    if (status < START_SLIDING && x > trigger) status = START_SLIDING

    if (status === START_SLIDING) {
      status = SLIDING
      slide.set(width)
    }

    if (status >= START_SLIDING) {
      size.set({
        buttonSize: $size.buttonSize + x,
      })
      if (base !== $slide) {
        base = $slide
      }
    }

    if (base >= width) {
      status = DONE
      emit('complete')
    } else if (
      lastX !== undefined &&
      lastY !== undefined &&
      (x !== lastX || y !== lastY)
    ) {
      emit('buttonmove', { x, y } as { x: number; y: number })
    }

    s = $size.buttonSize + x * 0.5

    btnOpacity = 1 + ($originalTipPosition.x - x) / 100

    // start drawing
    let nx = 0
    let ny = height + x
    let coords = `M-${step},0\n`
    for (ny = height + x; ny >= 0; ny -= step) {
      nx = x / Math.pow(Math.E, Math.pow(ny - y, 2) / (2 * s * s))
      coords += ` L${(base + nx).toFixed(precision)},${(height + x - ny).toFixed(
        precision
      )}\n`
    }
    coords += ` L-${step},${height + x}`
    //finish drawing

    path = `
    M-100,0
    L281.000,0.000
    L281.000,100.000
    L281.000,200.000
    L281.000,300.000
    L281.000,400.000
    L281.000,500.000
    L281.000,600.000
    L281.000,700.000
    L-100,755
    `

    path = coords
    rendering = false
    lastX = x
    lastY = y
    currentAnimationFrameRequest = requestAnimationFrame(render)
  }

  export function init(x: number, y: number) {
    if (currentAnimationFrameRequest !== 0) {
      cancelAnimationFrame(currentAnimationFrameRequest)
    }

    status = IDLE
    active = false
    path = ''
    btnOpacity = 1.0
    base = 0
    rendering = false
    currentAnimationFrameRequest = 0

    originalTipPosition = tweened(
      { x: 0, y },
      {
        duration: slideDuration,
        easing: slideEasing,
      }
    )

    originalTipPosition.set({ x, y })

    slide = tweened(0, {
      duration: slideDuration,
      easing: slideEasing,
    })

    size = spring(
      { buttonSize },
      {
        stiffness: veilExpansionStiffness,
        damping: veilExpansionDamping,
      }
    )

    tip = spring($originalTipPosition, {
      stiffness: buttonExpansionStiffness,
      damping: buttonExpansionDamping,
    })
    requestAnimationFrame(render)
  }

  function activate(e: TouchEvent | MouseEvent) {
    if (e.target === btn) active = true
  }

  function deactivate(e: TouchEvent | MouseEvent) {
    active = false
    if (status === IDLE) tip.set($originalTipPosition)
  }

  function watch(e: TouchEvent | MouseEvent, x: number, y: number) {
    if (active && status === IDLE && !rendering && (lastX !== x || lastY !== y)) {
      tip.set({ x, y })
      lastX = x
      lastY = y
    }
  }

  onMount(() => (mounted = true))
  $: if (mounted) init(x, y)
</script>

{#if mounted && enabled}
  <!-- useful for debugging purposes -->
  <!-- <div
    style="position:absolute; background: #000; color: #fff;padding:1rem;z-index:9;top:200px"
  >
    mounted:{JSON.stringify(mounted)}<br />
    width:{JSON.stringify(width)}<br />
    height:{JSON.stringify(height)}<br />
    id:{JSON.stringify(id)}<br />
    status:{JSON.stringify(status)}<br />
    base:{JSON.stringify(base)}<br />
    originalTipPosition:{JSON.stringify($originalTipPosition)}<br />
    tip:{JSON.stringify($tip)}<br />
    left:{base -
      ($tip.x < $originalTipPosition.x ? $originalTipPosition.x : $tip.x) -
      55}<br />
  </div> -->
  <div
    class="wrapper"
    on:mousedown={activate}
    on:mouseup={deactivate}
    on:mousemove={e => watch(e, e.clientX, e.clientY)}
    on:touchstart={activate}
    on:touchend={deactivate}
    on:touchcancel={deactivate}
    on:touchmove={e => watch(e, e.touches[0].clientX, e.touches[0].clientY)}
  >
    <svg {width} {height} viewBox="0 0 {width} {height}">
      <clipPath {id}>
        <path d={path} fill="rgba(0,0,0,0.5)" stroke="black" />
      </clipPath>
    </svg>

    <div class="page" style="clip-path: url(#{id});">
      <slot name="page" />
    </div>

    <div
      bind:this={btn}
      class="btn noselection"
      style="
				  opacity:{btnOpacity};
				  color:{btnColor};
				  border: 1px solid {btnBorderColor};
				  left:{base +
        ($tip.x < $originalTipPosition.x ? $originalTipPosition.x : $tip.x) -
        55}px;
				  top:{$tip.y - 25 - 2.5}px
				  "
    >
      {btnText}
    </div>
  </div>
{/if}
