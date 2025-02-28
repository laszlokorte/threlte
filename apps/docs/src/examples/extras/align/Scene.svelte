<script lang="ts">
  import { T, useRender } from '@threlte/core'
  import { Align, OrbitControls, RoundedBoxGeometry, TransformControls } from '@threlte/extras'
  import { tick } from 'svelte'
  import type { Box3, Vector3 } from 'three'

  export let x: number = 0
  export let y: number = 0
  export let z: number = 0
  export let precise: boolean = false
  export let showSphere: boolean = false
  export let autoAlign: boolean = false

  let boundingBox: Box3 | undefined
  let center: Vector3 | undefined

  useRender(async ({ renderer, scene, camera }) => {
    await tick()
    renderer.render(scene, camera.current)
  })
</script>

<T.PerspectiveCamera
  makeDefault
  position.z={10}
>
  <OrbitControls />
</T.PerspectiveCamera>

<!-- The property autoAlign is not reactive, therefore we need to key it here. -->
{#key autoAlign}
  <Align
    {x}
    {y}
    {z}
    {precise}
    auto={autoAlign}
    on:align={({ boundingBox: newBoundingBox, center: newCenter }) => {
      center = newCenter
      boundingBox = newBoundingBox
    }}
    let:align
  >
    <T.Mesh
      position.x={-1}
      let:ref
    >
      <TransformControls
        object={ref}
        on:objectChange={align}
      />
      <RoundedBoxGeometry args={[1, 2, 1]} />
      <T.MeshStandardMaterial color="white" />
    </T.Mesh>

    <T.Mesh
      position.x={-4}
      position.y={1}
    >
      <RoundedBoxGeometry args={[1, 2, 3]} />
      <T.MeshStandardMaterial color="white" />
    </T.Mesh>

    {#if showSphere}
      <T.Mesh
        position.x={-2}
        position.y={3}
      >
        <T.SphereGeometry />
        <T.MeshStandardMaterial color="white" />
      </T.Mesh>
    {/if}
  </Align>
{/key}

{#if boundingBox && center}
  <T.Group
    position.x={center.x}
    position.y={center.y}
    position.z={center.z}
  >
    <T.Box3Helper args={[boundingBox, 'white']} />
  </T.Group>
{/if}

<T.DirectionalLight position={[3, 10, 5]} />
<T.AmbientLight intensity={0.1} />

<T.AxesHelper />
