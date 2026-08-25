<script setup lang="ts">
import { useChatPanelStore } from '@/features/ai/assistant/chatPanel.store';
import { useChatHubPanelStore } from '@/features/ai/chatHub/chatHubPanel.store';
import { useUIStore } from '@/app/stores/ui.store';
import { provideWorkflowDocumentStore } from '@/app/stores/workflowDocument.store';
import {
	computed,
	defineAsyncComponent,
	nextTick,
	onBeforeUnmount,
	onMounted,
	ref,
	watch,
} from 'vue';

// The assistant/builder hub pulls in large trees (AI builder, NDV setup
// cards, markdown/chat rendering); load it on first panel open so it stays
// out of the boot bundle.
const AssistantsHub = defineAsyncComponent(
	async () => await import('@/features/ai/assistant/components/AssistantsHub.vue'),
);

const props = defineProps<{
	layoutRef: Element | null;
}>();

// The assistant/builder chat is mounted globally (App.vue #aside) and renders
// NDV-store consumers (setup cards, node issues, node execution) that may
// outlive the workflow editor (e.g. after navigating to a settings route).
// Re-provide the resolved workflow document store so those components resolve a
// scoped NDV store via injectNDVStore() even when no workflow is loaded.
provideWorkflowDocumentStore();

const chatPanelStore = useChatPanelStore();
const chatHubPanelStore = useChatHubPanelStore();
const uiStore = useUIStore();

const chatPanelWidth = computed(() => chatPanelStore.width);

// Mount the hub on first open and keep it mounted afterwards (the hub itself
// hides via v-show) so chat state survives closing the panel.
const hasOpenedChatPanel = ref(chatPanelStore.isOpen);
watch(
	() => chatPanelStore.isOpen,
	(isOpen) => {
		if (isOpen) hasOpenedChatPanel.value = true;
	},
);

const updateGridWidth = async () => {
	await nextTick();
	if (props.layoutRef) {
		const { width, height } = props.layoutRef.getBoundingClientRect();
		uiStore.appGridDimensions = { width, height };
	}
};

onMounted(async () => {
	window.addEventListener('resize', updateGridWidth);
	await updateGridWidth();
});

onBeforeUnmount(() => {
	window.removeEventListener('resize', updateGridWidth);
});

// As chat panel width changes, recalculate the total width regularly
// Skip when chatHub is open since it floats over the canvas
watch(chatPanelWidth, async () => {
	if (chatHubPanelStore.isOpen) return;
	await updateGridWidth();
});
</script>

<template>
	<AssistantsHub v-if="hasOpenedChatPanel" />
</template>
