<script lang="ts">
  import {
    ActiveCollisionTypes,
    CoefficientCombineRule,
    type Collider
  } from '@dimforge/rapier3d-compat'
  import { createRawEventDispatcher, SceneGraphObject } from '@threlte/core'
  import { onDestroy, onMount } from 'svelte'
  import { Group } from 'three'
  import { useCollisionGroups } from '../../hooks/useCollisionGroups'
  import { useHasEventListeners } from '../../hooks/useHasEventListener'
  import { useRapier } from '../../hooks/useRapier'
  import { useRigidBody } from '../../hooks/useRigidBody'
  import { applyColliderActiveEvents } from '../../lib/applyColliderActiveEvents'
  import { createCollidersFromChildren } from '../../lib/createCollidersFromChildren'
  import { eulerToQuaternion } from '../../lib/eulerToQuaternion'
  import type { ColliderEventMap } from '../../types/types'
  import type { AutoCollidersProps, MassDef } from './AutoColliders.svelte'

  type TMassDef = $$Generic<MassDef>
  type $$Props = AutoCollidersProps<TMassDef>

  export let shape: $$Props['shape'] = 'convexHull' as $$Props['shape']
  export let restitution: $$Props['restitution'] = undefined as $$Props['restitution']
  export let restitutionCombineRule: $$Props['restitutionCombineRule'] =
    undefined as $$Props['restitutionCombineRule']
  export let friction: $$Props['friction'] = undefined as $$Props['friction']
  export let frictionCombineRule: $$Props['frictionCombineRule'] =
    undefined as $$Props['frictionCombineRule']
  export let sensor: $$Props['sensor'] = undefined as $$Props['sensor']
  export let contactForceEventThreshold: $$Props['contactForceEventThreshold'] =
    undefined as $$Props['contactForceEventThreshold']
  export let density: $$Props['density'] = undefined
  export let mass: $$Props['mass'] = undefined
  export let centerOfMass: $$Props['centerOfMass'] = undefined
  export let principalAngularInertia: $$Props['principalAngularInertia'] = undefined
  export let angularInertiaLocalFrame: $$Props['angularInertiaLocalFrame'] = undefined

  const group = new Group()

  const rigidBody = useRigidBody()

  const { world, addColliderToContext, removeColliderFromContext } = useRapier()

  export let colliders: Collider[] = []

  const collisionGroups = useCollisionGroups()

  /**
   * Events setup
   */
  type $$Events = ColliderEventMap
  const dispatcher = createRawEventDispatcher<$$Events>()

  const { hasEventListeners: colliderHasEventListeners } = useHasEventListeners<typeof dispatcher>()

  const cleanup = () => {
    collisionGroups.removeColliders(colliders)
    colliders.forEach((c) => {
      removeColliderFromContext(c)
      world.removeCollider(c, true)
    })
    colliders.length = 0
  }

  export const create = () => {
    cleanup()
    colliders = createCollidersFromChildren(group, shape ?? 'convexHull', world, rigidBody)
    colliders.forEach((c) => addColliderToContext(c, group, dispatcher))

    collisionGroups.registerColliders(colliders)

    colliders.forEach((collider) => {
      applyColliderActiveEvents(
        collider,
        colliderHasEventListeners,
        rigidBody?.userData?.hasEventListeners
      )
      collider.setActiveCollisionTypes(ActiveCollisionTypes.ALL)
      collider.setRestitution(restitution ?? 0)
      collider.setRestitutionCombineRule(restitutionCombineRule ?? CoefficientCombineRule.Average)
      collider.setFriction(friction ?? 0.7)
      collider.setFrictionCombineRule(frictionCombineRule ?? CoefficientCombineRule.Average)
      collider.setSensor(sensor ?? false)
      collider.setContactForceEventThreshold(contactForceEventThreshold ?? 0)
      if (density) collider.setDensity(density)
      if (mass) {
        if (centerOfMass && principalAngularInertia && angularInertiaLocalFrame)
          collider.setMassProperties(
            mass,
            { x: centerOfMass[0], y: centerOfMass[1], z: centerOfMass[2] },
            {
              x: principalAngularInertia[0],
              y: principalAngularInertia[1],
              z: principalAngularInertia[2]
            },
            eulerToQuaternion(angularInertiaLocalFrame)
          )
        else collider.setMass(mass)
      }
    })
  }

  onMount(() => {
    create()
  })

  export const refresh = () => {
    create()
  }

  /**
   * Cleanup
   */
  onDestroy(cleanup)
</script>

<SceneGraphObject object={group}>
  <slot
    {colliders}
    {refresh}
  />
</SceneGraphObject>
