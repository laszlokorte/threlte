<script lang="ts">
  import { T, createRawEventDispatcher, forwardEventHandlers, useFrame } from '@threlte/core'
  import { onMount } from 'svelte'
  import { Box3, Group, Sphere, Vector3 } from 'three'
  import type { AlignEvents, AlignProps, AlignSlots } from './Align.svelte'
  import { injectAlignPlugin } from './alignPlugin'

  type $$Props = AlignProps
  type $$Events = AlignEvents
  type $$Slots = AlignSlots

  /** Align the object on the x-axis. If a number between -1 and 1 is provided, it will be used as the alignment on the x-axis. */
  export let x: $$Props['x'] = 0
  /** Align the object on the y-axis. If a number between -1 and 1 is provided, it will be used as the alignment on the y-axis. */
  export let y: $$Props['y'] = 0
  /** Align the object on the z-axis. If a number between -1 and 1 is provided, it will be used as the alignment on the z-axis. */
  export let z: $$Props['z'] = 0
  /** See https://threejs.org/docs/index.html?q=box3#api/en/math/Box3.setFromObject */
  export let precise: $$Props['precise'] = false
  /** Injects a plugin in all child `<T>` components to automatically align whenever a component mounts or unmounts, defaults to false */
  export let auto: $$Props['auto'] = false
  /** @deprecated Please use the prop "auto" */
  export let autoCenter: $$Props['autoCenter'] = false

  const dispatch = createRawEventDispatcher<$$Events>()

  const containerGroup = new Group()
  const innerGroup = new Group()
  const outerGroup = new Group()

  let mounted = false
  onMount(() => {
    mounted = true
  })

  export const calculate = () => {
    // we're only aligning when the component mounted, meaning all child
    // component have been mounted as well.
    if (!mounted) return

    // return early if all axes are false
    if (x === false && y === false && z === false) return

    outerGroup.matrixWorld.identity()
    const box3 = new Box3().setFromObject(innerGroup, precise)
    const align = new Vector3()
    const sphere = new Sphere()
    const width = box3.max.x - box3.min.x
    const height = box3.max.y - box3.min.y
    const depth = box3.max.z - box3.min.z

    box3.getCenter(align)
    box3.getBoundingSphere(sphere)

    const vAlign = ((y || 0) * height) / 2
    const hAlign = ((x || 0) * width) / 2
    const dAlign = ((z || 0) * depth) / 2

    outerGroup.position.set(
      x === false ? 0 : -align.x + hAlign,
      y === false ? 0 : -align.y + vAlign,
      z === false ? 0 : -align.z + dAlign
    )

    const eventData = {
      boundingBox: box3,
      center: outerGroup.position.clone(),
      boundingSphere: sphere,
      container: containerGroup,
      depth,
      depthAlignment: dAlign,
      height,
      verticalAlignment: vAlign,
      width,
      horizontalAlignment: hAlign
    }

    dispatch('align', eventData)

    // Will be emitted as the event is marked as deprecated and will only be
    // removed in the next major version.
    dispatch('center', eventData)
  }

  /**
   * We're only aligning at most *once* per frame, so even if a lot of child
   * components request aligning, it's only done *once*.
   */
  const { start: scheduleAligning, stop } = useFrame(
    ({ invalidate }) => {
      calculate()
      invalidate()
      stop()
    },
    {
      autostart: false,
      invalidate: false
    }
  )

  /** Force a recalculation of the bounding box. */
  export const align = () => {
    scheduleAligning()
  }

  /**
   * We're only recalculating when the props change and the component is
   * mounted.
   */
  $: mounted === true && x, y, z, precise, scheduleAligning()

  /**
   * Inject the align plugin if autoAlign is true. This will automatically
   * align the object whenever a child component mounts or unmounts.
   */
  if (auto || autoCenter) injectAlignPlugin(scheduleAligning)

  const component = forwardEventHandlers()
</script>

<T
  is={containerGroup}
  {...$$restProps}
  bind:this={$component}
  let:ref
>
  <T is={outerGroup}>
    <T is={innerGroup}>
      <slot
        align={scheduleAligning}
        center={scheduleAligning}
        {ref}
      />
    </T>
  </T>
</T>
