<template>
	<v-app class="task-app">
		<SettingsDialog v-model="settingsDialog" />
		<UndoDialog v-model="showUndoDialog" />

		<v-snackbar
			v-model="snackbar"
			:color="notification.color"
			:timeout="4000"
		>
			{{ notification.text }}

			<template v-slot:action="{ attrs }">
				<v-btn
					dark
					text
					v-bind="attrs"
					@click="snackbar = false"
				>
					Close
				</v-btn>
			</template>
		</v-snackbar>

		<v-app-bar height="54px" fixed app>
			<v-icon class="mr-2" color="blue">
				mdi-sticker-check-outline
			</v-icon>
			<v-toolbar-title class="mr-4">
				Taskwarrior WebUI
			</v-toolbar-title>
			<v-text-field
				id="cmdSearchBar"
				ref="searchInputRef"
				v-model="searchQuery"
				dense
				flat
				solo-inverted
				hide-details
				prepend-inner-icon="mdi-magnify"
				placeholder="Search bar (Ctrl+Shift+K)"
				style="max-width: 320px;"
				class="mx-2"
			/>
			<ActiveTaskBar />
			<v-spacer />
			<v-icon class="mr-4" size="28px" @click="dark = !dark" title="Theme">
				{{ dark ? 'mdi-brightness-4' : 'mdi-brightness-7' }}
			</v-icon>
			<v-icon
				class="mr-2"
				size="28px"
				title="Settings"
				@click="settingsDialog = true"
			>
				mdi-cog
			</v-icon>
		</v-app-bar>

		<v-main>
			<v-container fluid>
				<nuxt />
			</v-container>
		</v-main>
	</v-app>
</template>

<script lang="ts">
import { defineComponent, useContext, useStore, computed, onErrorCaptured, ref, onMounted, onUnmounted } from '@nuxtjs/composition-api';
import SettingsDialog from '../components/SettingsDialog.vue';
import ActiveTaskBar from '../components/ActiveTaskBar.vue';
import UndoDialog from '../components/UndoDialog.vue';
import { HotkeyBus } from '../plugins/hotkeys';
import { accessorType  } from "../store";

export default defineComponent({
	setup(_props, ctx) {
		const context = useContext();
		const store = useStore<typeof accessorType>();
		store.dispatch('fetchSettings');
		store.dispatch('fetchHiddenColumns');

		context.$vuetify.theme.dark = store.state.settings.dark;

		const dark = computed({
			get: () => context.$vuetify.theme.dark,
			set: val => {
				context.$vuetify.theme.dark = val;
			}
		});

		const settingsDialog = ref(false);
		const showUndoDialog = ref(false);
		const searchQuery = ref('');
		const searchInputRef = ref<any>(null);

		const handleTriggerUndo = () => {
			showUndoDialog.value = true;
		};

		onMounted(() => {
			HotkeyBus.$on('trigger-undo', handleTriggerUndo);
		});

		onUnmounted(() => {
			HotkeyBus.$off('trigger-undo', handleTriggerUndo);
		});

		if (process.client) {
			window.addEventListener('keydown', (e: KeyboardEvent) => {
				if (e.ctrlKey && e.shiftKey && (e.key === 'K' || e.key === 'k')) {
					e.preventDefault();
					if (searchInputRef.value && searchInputRef.value.focus) {
						searchInputRef.value.focus();
					}
				}
			});
		}

		const notification = computed(() => store.state.notification);
		const snackbar = computed({
			get: () => store.state.snackbar,
			set: val => store.commit('setSnackbar', val)
		});

		onErrorCaptured((err: any) => {
			// axios error
			let notification: any;
			if (err?.response) {
				const { status, data } = err.response!;
				notification = {
					color: 'error',
					text: `Error ${status}: ${data}`
				};
			}
			else {
				const { name, message } = err as Error;
				notification = {
					color: 'error',
					text: `Error ${name}: ${message}`
				};
			}
			store.commit('setNotification', notification);
			return false;
		});

		return {
			dark,
			snackbar,
			notification,
			settingsDialog,
			showUndoDialog,

			SettingsDialog,
			ActiveTaskBar,
			UndoDialog,
			searchQuery,
			searchInputRef
		};
	}
});
</script>
