<!-- <script lang="ts" setup>
import { useUserStore } from "@/store/modules/user"
import Admin from "./components/Admin.vue"
import Editor from "./components/Editor.vue"
// import EChartsComponent from './EChartsComponent.vue'

const userStore = useUserStore()
const isAdmin = userStore.roles.includes("admin")
</script>

<template>
  <EChartsComponent />
  <component :is="isAdmin ? Admin : Editor" />
</template> -->

<script lang="ts" setup>
import { useUserStore } from "@/store/modules/user"
import Admin from "./components/Admin.vue"
import Editor from "./components/Editor.vue"

const userStore = useUserStore()
const isAdmin = userStore.roles.includes("admin")
</script>

<template>
  <component :is="isAdmin ? Admin : Editor" />
</template>

