<template>
	<AuthenticatedLayout>
		<template #header>
      Diagrama de secuencia: {{props.diagram.name}}
    </template>
		<div class="app">
			<Editor :diagram="props.diagram" :project="props.project"></Editor>
		</div>
	</AuthenticatedLayout>
</template>
  
<script setup>
// @ is an alias to /src
import Editor from '@/Components/Editor.vue'
import AuthenticatedLayout from '@/Layouts/AuthenticatedLayout.vue';

const props = defineProps({
	diagram: { type: Object },
	project: { type: Object }
})

</script>
  
<style scoped lang="scss">
.home {
	width: 1000px;
	margin: 0 auto;
}
</style>
  