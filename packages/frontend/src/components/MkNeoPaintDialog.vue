<!--
SPDX-FileCopyrightText: syuilo and misskey-project
SPDX-License-Identifier: AGPL-3.0-only
-->

<template>
<MkModalWindow
	ref="dialogEl"
	:width="dialogWidth"
	:height="dialogHeight"
	:scroll="false"
	@close="cancel()"
	@closed="emit('closed')"
>
	<template #header><i class="ti ti-brush"></i> {{ i18n.ts.neoPaint }}</template>

	<div :class="$style.root">
		<Transition name="fade">
			<div v-if="loading" :class="$style.loading">
				<MkLoading/>
			</div>
		</Transition>
		<div ref="neoRootEl" :class="$style.neoRoot"></div>
	</div>

	<template v-if="!neoWindowExpanded" #footer>
		<div class="_buttonsCenter">
			<MkButton rounded @click="cancel()"><i class="ti ti-x"></i> {{ i18n.ts.cancel }}</MkButton>
		</div>
	</template>
</MkModalWindow>
</template>

<script lang="ts" setup>
import { computed, nextTick, onBeforeUnmount, onMounted, ref, useTemplateRef } from 'vue';
import MkModalWindow from '@/components/MkModalWindow.vue';
import MkButton from '@/components/MkButton.vue';
import { i18n } from '@/i18n.js';
import neoScriptUrl from '../../assets/neo/neo.js?url';
import neoStyleUrl from '../../assets/neo/neo.css?url';

type PaintBBSCallback = (message: string) => string | number | boolean | void;
type PaintBBSCallbackDocument = Document & {
	paintBBSCallback?: PaintBBSCallback;
	paintbbs?: unknown;
	neo?: NeoGlobal;
};
type NeoGlobal = {
	applet: Element | null;
	container: HTMLElement | null;
	fullScreen: boolean;
	uploaded: boolean;
	viewer: boolean;
	init: () => boolean;
	updateWindow: () => void;
	start: (isApp?: boolean) => void;
	painter?: {
		getPNG: () => Blob | null;
		isDirty: () => boolean;
	};
};

const emit = defineEmits<{
	(ev: 'done', file: File): void;
	(ev: 'cancel'): void;
	(ev: 'closed'): void;
}>();

const dialogEl = useTemplateRef('dialogEl');
const neoRootEl = useTemplateRef('neoRootEl');
const loading = ref(true);
const neoWindowExpanded = ref(false);
const viewportWidth = ref(window.innerWidth);
const viewportHeight = ref(window.innerHeight);
const dialogWidth = computed(() => neoWindowExpanded.value ? viewportWidth.value : 560);
const dialogHeight = computed(() => neoWindowExpanded.value ? viewportHeight.value : 620);

let previousCallback: PaintBBSCallback | undefined;
let installedCallback: PaintBBSCallback | undefined;
let submitted = false;
let neoScriptPromise: Promise<void> | null = null;
let previousUpdateWindow: NeoGlobal['updateWindow'] | undefined;

function ensureNeoStyle(): void {
	if (window.document.querySelector<HTMLLinkElement>('link[data-mk-neo-paint-style="true"]')) return;

	const link = window.document.createElement('link');
	link.rel = 'stylesheet';
	link.href = neoStyleUrl;
	link.dataset.mkNeoPaintStyle = 'true';
	window.document.head.appendChild(link);
}

function ensureNeoScript(): Promise<void> {
	const doc = window.document as PaintBBSCallbackDocument;
	if (doc.neo != null) return Promise.resolve();
	if (neoScriptPromise) return neoScriptPromise;

	neoScriptPromise = new Promise((resolve, reject) => {
		const script = window.document.createElement('script');
		script.src = neoScriptUrl;
		script.async = true;
		script.onload = () => resolve();
		script.onerror = () => {
			neoScriptPromise = null;
			reject(new Error('Failed to load PaintBBS NEO'));
		};
		window.document.head.appendChild(script);
	});

	return neoScriptPromise;
}

function createApplet(): void {
	if (neoRootEl.value == null) return;

	neoRootEl.value.innerHTML = `
		<applet-dummy name="paintbbs" width="500" height="560">
			<param name="image_width" value="400">
			<param name="image_height" value="400">
			<param name="neo_show_right_button" value="true">
			<param name="neo_disable_grid_touch_move" value="true">
			<param name="neo_disable_turn_original_glitch" value="true">
			<param name="neo_enable_zoom_out" value="true">
			<param name="neo_emulation_mode" value="2.22_8x">
		</applet-dummy>
	`;
}

function imageFileFromNeo(): File | null {
	const neo = (window.document as PaintBBSCallbackDocument).neo;
	const blob = neo?.painter?.getPNG();
	if (blob == null) return null;

	return new File([blob], `neo-${Date.now()}.png`, { type: 'image/png' });
}

function installCallback(): void {
	const doc = window.document as PaintBBSCallbackDocument;
	previousCallback = doc.paintBBSCallback;
	installedCallback = (message: string) => {
		if (message !== 'check') return undefined;
		if (submitted) return false;

		const file = imageFileFromNeo();
		if (file == null) return false;

		submitted = true;
		emit('done', file);
		dialogEl.value?.close();

		return false;
	};
	doc.paintBBSCallback = installedCallback;
}

function restoreCallback(): void {
	const doc = window.document as PaintBBSCallbackDocument;
	if (doc.paintBBSCallback !== installedCallback) return;
	doc.paintBBSCallback = previousCallback;
	installedCallback = undefined;
}

function updateViewportSize(): void {
	viewportWidth.value = window.innerWidth;
	viewportHeight.value = window.innerHeight;
}

function syncNeoWindowState(): void {
	const neo = (window.document as PaintBBSCallbackDocument).neo;
	neoWindowExpanded.value = neo?.fullScreen ?? false;
	updateViewportSize();
}

function patchNeoWindowState(): void {
	const neo = (window.document as PaintBBSCallbackDocument).neo;
	if (neo == null || previousUpdateWindow != null) return;

	previousUpdateWindow = neo.updateWindow.bind(neo);
	neo.updateWindow = () => {
		previousUpdateWindow?.();
		syncNeoWindowState();
	};
	syncNeoWindowState();
}

function restoreNeoWindowState(): void {
	const neo = (window.document as PaintBBSCallbackDocument).neo;
	if (neo != null && previousUpdateWindow != null) {
		neo.updateWindow = previousUpdateWindow;
	}
	previousUpdateWindow = undefined;
	neoWindowExpanded.value = false;
}

function cleanupNeoDom(): void {
	if (neoRootEl.value != null) {
		neoRootEl.value.innerHTML = '';
	}
	const doc = window.document as PaintBBSCallbackDocument;
	const neo = doc.neo;
	if (neo != null) {
		neo.fullScreen = false;
		neo.applet = null;
		neo.container = null;
		neo.uploaded = true;
		neo.viewer = false;
	}
}

function cancel(): void {
	emit('cancel');
	dialogEl.value?.close();
}

onMounted(async () => {
	window.addEventListener('resize', updateViewportSize);
	ensureNeoStyle();
	installCallback();
	createApplet();
	await ensureNeoScript();
	await nextTick();

	const neo = (window.document as PaintBBSCallbackDocument).neo;
	if (neo != null) {
		neo.uploaded = false;
		neo.viewer = false;
	}
	if (neo?.init()) {
		patchNeoWindowState();
		neo.start();
	}
	loading.value = false;
});

onBeforeUnmount(() => {
	window.removeEventListener('resize', updateViewportSize);
	restoreCallback();
	restoreNeoWindowState();
	cleanupNeoDom();
});
</script>

<style lang="scss" module>
.root {
	position: relative;
	height: 100%;
	overflow: auto;
	background: var(--MI_THEME-bg);
}

.loading {
	position: absolute;
	inset: 0;
	z-index: 1;
	display: grid;
	place-items: center;
	background: var(--MI_THEME-bg);
}

.neoRoot {
	min-width: 500px;
	padding: 12px;
}

.neoRoot :global(#neo-pageView) {
	margin: 0 auto !important;
}
</style>
