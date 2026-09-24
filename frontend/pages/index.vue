<template>
	<div class="px-md-6 px-lg-12">
		<v-row class="px-4 pt-4 align-center">
			<div class="headline d-flex align-center">{{ mode }}</div>
			
			<v-select
				class="mb-3 ml-4"
				:items="projectOptions"
				label="Project"
				v-model="selectedProject"
				style="max-width: 180px"
				hide-details
				dense
			/>

			<v-select
				class="mb-3 ml-3"
				:items="assigneeOptions"
				label="Assignee"
				v-model="selectedAssignee"
				style="max-width: 180px"
				hide-details
				dense
			/>

			<v-spacer />
			<v-select
				class="mb-3 ml-3"
				:items="allModes"
				label="Display Mode"
				v-model="mode"
				style="max-width: 140px"
				hide-details
				dense
			/>
		</v-row>

		<KanbanBoard
			v-if="mode === 'Kanban'"
			:tasks="filteredTasks"
			:active-task-uuid="activeTaskUuid"
			@column-change="handleColumnChange"
			@toggle-timer="handleToggleTimer"
			@complete="handleCompleteTask"
			@delete="handleDeleteTask"
			@edit="handleEditTask"
		/>
		<TaskList v-else :tasks="filteredTasks" ref="taskListRef" />

		<TaskDialog v-model="showTaskDialog" :task="editingTask || undefined" />
	</div>
</template>

<script lang="ts">
import { defineComponent, ref, computed, watch, ComputedRef, useStore, useContext } from '@nuxtjs/composition-api';
import TaskList from '../components/TaskList.vue';
import KanbanBoard from '../components/KanbanBoard.vue';
import TaskDialog from '../components/TaskDialog.vue';
import { Task } from 'taskwarrior-lib';
import { accessorType  } from "../store";

export default defineComponent({
	components: {
		TaskList,
		KanbanBoard,
		TaskDialog
	},
	setup() {
		const store = useStore<typeof accessorType>();
		const context = useContext();
		store.dispatch('fetchTasks');

		// Auto Refresh
		let refreshInterval: NodeJS.Timeout | null = null;
		const setAutoRefresh = () => {
			if (refreshInterval)
				clearInterval(refreshInterval);
			const freq = +store.state.settings.autoRefresh;
			if (freq > 0) {
				refreshInterval = setInterval(() => {
					store.dispatch('fetchTasks');
				}, +store.state.settings.autoRefresh * 60000);
			}
		};
		setAutoRefresh();

		// Auto Sync
		let syncInterval: NodeJS.Timeout | null = null;
		const setAutoSync = () => {
			if (syncInterval)
				clearInterval(syncInterval);
			const freq = +store.state.settings.autoSync;
			if (freq > 0) {
				syncInterval = setInterval(() => {
					store.dispatch('syncTasks');
				}, +store.state.settings.autoSync * 60000);
			}
		};
		setAutoSync();

		// Update settings
		watch(() => store.state.settings, () => {
			setAutoSync();
			setAutoRefresh();
			context.$vuetify.theme.dark = store.state.settings.dark;
		});

		const mode = ref('Tasks');
		const allModes = ['Tasks', 'Kanban'];

		const selectedProject = ref('(All Projects)');
		const projects = computed(() => store.getters.projects);
		const projectOptions = computed(() => ['(All Projects)', '(No Project)', ...projects.value]);

		const selectedAssignee = ref('(All Assignees)');
		const assignees = computed(() => store.getters.assignees);
		const assigneeOptions = computed(() => ['(All Assignees)', '(Unassigned)', ...assignees.value]);

		const allTasks: ComputedRef<Task[]> = computed(() => store.state.tasks);
		const activeTaskUuid = computed(() => store.state.activeTask?.uuid || '');

		const filteredTasks = computed(() => {
			let result = store.state.tasks || [];

			// Project filter
			if (selectedProject.value === '(No Project)') {
				result = result.filter(t => !t.project);
			} else if (selectedProject.value && selectedProject.value !== '(All Projects)') {
				result = result.filter(t => t.project === selectedProject.value);
			}

			// Assignee filter
			if (selectedAssignee.value === '(Unassigned)') {
				result = result.filter(t => !(t as any).assignee && !(t as any).assignees);
			} else if (selectedAssignee.value && selectedAssignee.value !== '(All Assignees)') {
				result = result.filter(t => (t as any).assignee === selectedAssignee.value || (t as any).assignees === selectedAssignee.value);
			}

			return result;
		});

		const showTaskDialog = ref(false);
		const editingTask = ref<Task | null>(null);

		const handleEditTask = (task: Task) => {
			editingTask.value = task;
			showTaskDialog.value = true;
		};

		const handleCompleteTask = async (task: Task) => {
			await store.dispatch('updateTasks', [{
				...task,
				status: 'completed'
			}]);
		};

		const handleDeleteTask = async (task: Task) => {
			await store.dispatch('deleteTasks', [task]);
		};

		const handleToggleTimer = async (task: Task) => {
			if (task.start || task.uuid === activeTaskUuid.value) {
				await store.dispatch('stopTimer', task.uuid);
			} else {
				await store.dispatch('startTimer', task.uuid);
			}
		};

		const handleColumnChange = async ({ task, targetColumnId }: { task: Task; targetColumnId: string }) => {
			if (targetColumnId === 'inProgress') {
				await store.dispatch('startTimer', task.uuid);
			} else if (targetColumnId === 'done') {
				if (task.start || task.uuid === activeTaskUuid.value) {
					await store.dispatch('stopTimer', task.uuid);
				}
				await store.dispatch('updateTasks', [{ ...task, status: 'completed' }]);
			} else if (targetColumnId === 'todo') {
				if (task.start || task.uuid === activeTaskUuid.value) {
					await store.dispatch('stopTimer', task.uuid);
				}
				if (task.status === 'completed') {
					await store.dispatch('updateTasks', [{ ...task, status: 'pending' }]);
				}
			}
		};

		return {
			mode,
			allModes,
			selectedProject,
			projectOptions,
			selectedAssignee,
			assigneeOptions,
			filteredTasks,
			activeTaskUuid,
			showTaskDialog,
			editingTask,
			handleEditTask,
			handleCompleteTask,
			handleDeleteTask,
			handleToggleTimer,
			handleColumnChange
		};
	}
});
</script>
