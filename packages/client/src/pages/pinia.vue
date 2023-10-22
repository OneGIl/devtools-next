<script setup lang="ts">
import { onDevToolsClientConnected, useDevToolsBridgeRpc } from '@vue-devtools-next/app-core'

// eslint-disable-next-line ts/consistent-type-imports
import type { InspectorState } from '@vue-devtools-next/schema'
import { Pane, Splitpanes } from 'splitpanes'

const bridgeRpc = useDevToolsBridgeRpc()

const { selected } = createSelectContext('inspector-tree')
const tree = ref<{ id: string;label: string }[]>([])
const state = ref<Record<string, InspectorState[]>>({})

function getPiniaState(nodeId: string) {
  bridgeRpc.getInspectorState({ inspectorId: 'pinia', nodeId }).then(({ data }) => {
    state.value = data.state
  })
}

function selectStore(id: string) {
  getPiniaState(id)
}

createCollapseContext('inspector-state')

onDevToolsClientConnected(() => {
  bridgeRpc.getInspectorTree({ inspectorId: 'pinia', filter: '' }).then(({ data }) => {
    tree.value = data
    if (!selected.value && data.length) {
      selected.value = data[0].id
      getPiniaState(data[0].id)
    }
  })

  // @TODO: duplicated callback calls (different inspectorId)
  bridgeRpc.on.inspectorTreeUpdated(({ data }) => {
    if (!data)
      return
    tree.value = data
    if (!selected.value && data.length) {
      selected.value = data[0].id
      getPiniaState(data[0].id)
    }
  })

  bridgeRpc.on.inspectorStateUpdated((data) => {
    if (!data || !data.state || data.inspectorId !== 'pinia')
      return

    state.value = data.state
  })
})
</script>

<template>
  <div h-screen>
    <Splitpanes>
      <Pane flex flex-col border="r base">
        <div h-screen select-none overflow-scroll p-2 class="no-scrollbar">
          <InspectorTree v-for="(item) in tree" :key="item.id" :data="item" @select="selectStore" />
        </div>
      </Pane>
      <Pane flex flex-col overflow-y-scroll class="no-scrollbar">
        <div p-2>
          <InspectorState v-for="(item, key) in state" :id="key" :key="key + Date.now()" :data="item" :name="`${key}`" />
        </div>
      </Pane>
    </Splitpanes>
  </div>
</template>