<script lang="ts">
	import { onMount, getContext } from 'svelte';
	import { toast } from 'svelte-sonner';
	import { goto } from '$app/navigation';
	import { WEBUI_API_BASE_URL } from '$lib/constants';
	import { listImages } from '$lib/apis/images';
	import { deleteFileById } from '$lib/apis/files';
	import ImagePreview from '../common/ImagePreview.svelte';
	import ConfirmDialog from '../common/ConfirmDialog.svelte';
	import GarbageBin from '../icons/GarbageBin.svelte';
	import ChatBubble from '../icons/ChatBubble.svelte';
	import Spinner from '../common/Spinner.svelte';

	const i18n = getContext('i18n');

	const PER_PAGE = 25;

	let loaded = false;
	let images: any[] = [];
	let activeTab: 'generated' | 'uploaded' = 'generated';

	let currentPage = 1;

	let previewSrc = '';
	let previewAlt = '';
	let showPreview = false;

	let showDeleteConfirm = false;
	let deleteTargetId: string | null = null;

	$: filteredImages = images.filter((img) => img.source === activeTab);

	$: totalPages = Math.max(1, Math.ceil(filteredImages.length / PER_PAGE));

	$: if (currentPage > totalPages) {
		currentPage = totalPages;
	}

	$: paginatedImages = filteredImages.slice(
		(currentPage - 1) * PER_PAGE,
		currentPage * PER_PAGE
	);

	$: visiblePages = (() => {
		const pages: (number | null)[] = [];
		if (totalPages <= 7) {
			for (let i = 1; i <= totalPages; i++) pages.push(i);
		} else {
			pages.push(1);
			if (currentPage > 3) pages.push(null);
			const start = Math.max(2, currentPage - 1);
			const end = Math.min(totalPages - 1, currentPage + 1);
			for (let i = start; i <= end; i++) pages.push(i);
			if (currentPage < totalPages - 2) pages.push(null);
			pages.push(totalPages);
		}
		return pages;
	})();

	const openPreview = (src: string, alt: string) => {
		previewSrc = src;
		previewAlt = alt;
		showPreview = true;
	};

	const useInNewChat = (imageId: string, filename: string, contentType: string) => {
		sessionStorage.setItem(
			'chat-input',
			JSON.stringify({
				prompt: '',
				files: [
					{
						id: imageId,
						url: imageId,
						name: filename,
						type: 'file',
						status: 'uploaded',
						content_type: contentType
					}
				]
			})
		);
		goto('/');
	};

	const deleteHandler = async () => {
		if (!deleteTargetId) return;
		try {
			await deleteFileById(localStorage.token, deleteTargetId);
			toast.success($i18n.t('File deleted successfully.'));
			images = images.filter((img) => img.id !== deleteTargetId);
		} catch (error) {
			toast.error(`${error}`);
		}
		deleteTargetId = null;
	};

	const goToPage = (page: number) => {
		if (page >= 1 && page <= totalPages) {
			currentPage = page;
		}
	};

	onMount(async () => {
		try {
			images = await listImages(localStorage.token);
		} catch (e) {
			console.error(e);
		}
		loaded = true;
	});
</script>

<ImagePreview bind:show={showPreview} src={previewSrc} alt={previewAlt} />

<ConfirmDialog
	bind:show={showDeleteConfirm}
	title={$i18n.t('Delete Image')}
	message={$i18n.t('Are you sure you want to delete this image?')}
	onConfirm={deleteHandler}
	confirmLabel={$i18n.t('Delete')}
/>

<div class="flex flex-col w-full h-full">
	{#if loaded}
		<div class="flex items-center justify-between px-2 py-1.5 border-b border-gray-100 dark:border-gray-800">
			<div class="flex gap-1">
				<button
					class="px-4 py-1.5 text-sm rounded-xl transition {activeTab === 'generated'
						? 'bg-black text-white dark:bg-white dark:text-black'
						: 'text-gray-500 hover:text-gray-700 dark:hover:text-gray-300 hover:bg-gray-100 dark:hover:bg-gray-850'}"
					on:click={() => {
						activeTab = 'generated';
						currentPage = 1;
					}}
				>
					{$i18n.t('Generated')}
				</button>
				<button
					class="px-4 py-1.5 text-sm rounded-xl transition {activeTab === 'uploaded'
						? 'bg-black text-white dark:bg-white dark:text-black'
						: 'text-gray-500 hover:text-gray-700 dark:hover:text-gray-300 hover:bg-gray-100 dark:hover:bg-gray-850'}"
					on:click={() => {
						activeTab = 'uploaded';
						currentPage = 1;
					}}
				>
					{$i18n.t('Uploaded')}
				</button>
			</div>

			{#if totalPages > 1}
				<div class="flex items-center gap-1 text-sm">
					<button
						class="px-2 py-1 rounded-lg transition disabled:opacity-30 {currentPage === 1
							? 'text-gray-300 dark:text-gray-700'
							: 'text-gray-500 hover:text-gray-800 dark:hover:text-gray-200 hover:bg-gray-100 dark:hover:bg-gray-850'}"
						disabled={currentPage === 1}
						on:click={() => goToPage(currentPage - 1)}
						aria-label={$i18n.t('Previous page')}
					>
						<svg xmlns="http://www.w3.org/2000/svg" fill="none" viewBox="0 0 24 24" stroke-width="2" stroke="currentColor" class="size-4">
							<path stroke-linecap="round" stroke-linejoin="round" d="M15.75 19.5 8.25 12l7.5-7.5" />
						</svg>
					</button>

					{#each visiblePages as page}
						{#if page === null}
							<span class="text-gray-400 dark:text-gray-600 text-xs select-none">...</span>
						{:else}
							<button
								class="min-w-[1.75rem] h-7 rounded-lg text-xs font-medium transition {page === currentPage
									? 'bg-black text-white dark:bg-white dark:text-black'
									: 'text-gray-500 hover:text-gray-800 dark:hover:text-gray-200 hover:bg-gray-100 dark:hover:bg-gray-850'}"
								on:click={() => goToPage(page)}
							>
								{page}
							</button>
						{/if}
					{/each}

					<button
						class="px-2 py-1 rounded-lg transition disabled:opacity-30 {currentPage === totalPages
							? 'text-gray-300 dark:text-gray-700'
							: 'text-gray-500 hover:text-gray-800 dark:hover:text-gray-200 hover:bg-gray-100 dark:hover:bg-gray-850'}"
						disabled={currentPage === totalPages}
						on:click={() => goToPage(currentPage + 1)}
						aria-label={$i18n.t('Next page')}
					>
						<svg xmlns="http://www.w3.org/2000/svg" fill="none" viewBox="0 0 24 24" stroke-width="2" stroke="currentColor" class="size-4">
							<path stroke-linecap="round" stroke-linejoin="round" d="m8.25 4.5 7.5 7.5-7.5 7.5" />
						</svg>
					</button>
				</div>
			{/if}
		</div>

		<div class="flex-1 overflow-y-auto p-3">
			{#if filteredImages.length === 0}
				<div class="w-full h-full flex justify-center items-center text-gray-400 text-sm">
					{$i18n.t('No images found')}
				</div>
			{:else}
				<div class="grid grid-cols-2 sm:grid-cols-3 md:grid-cols-4 lg:grid-cols-5 gap-3">
					{#each paginatedImages as image (image.id)}
						{@const src = `${WEBUI_API_BASE_URL}/files/${image.id}/content`}
						<div
							class="relative group aspect-square rounded-xl overflow-hidden bg-gray-100 dark:bg-gray-900 hover:ring-2 ring-gray-300 dark:ring-gray-700 transition cursor-pointer"
						>
							<button
								class="w-full h-full"
								on:click={() => openPreview(src, image.filename)}
								type="button"
								aria-label={$i18n.t('Show image preview')}
							>
								<img
									src={src}
									alt={image.filename}
									class="w-full h-full object-cover"
									loading="lazy"
								/>
							</button>

							<div class="absolute bottom-0 left-0 right-0 bg-black/50 text-white text-[11px] px-2 py-1 truncate text-center">
								{new Date(image.created_at * 1000).toLocaleString(undefined, { month: 'short', day: 'numeric', year: 'numeric', hour: '2-digit', minute: '2-digit' })}
							</div>

							<div
								class="absolute top-2 right-2 flex gap-1"
							>
								<button
									class="p-1.5 rounded-full bg-black/60 text-white hover:bg-blue-600 transition"
									on:click|stopPropagation={() => useInNewChat(image.id, image.filename, image.content_type)}
									aria-label={$i18n.t('Use in new chat')}
									type="button"
								>
									<ChatBubble class="size-4" strokeWidth="1.5" />
								</button>
								<button
									class="p-1.5 rounded-full bg-black/60 text-white hover:bg-red-600 transition"
									on:click|stopPropagation={() => {
										deleteTargetId = image.id;
										showDeleteConfirm = true;
									}}
									aria-label={$i18n.t('Delete File')}
									type="button"
								>
									<GarbageBin class="size-4" strokeWidth="1.5" />
								</button>
							</div>
						</div>
					{/each}
				</div>
			{/if}
		</div>
	{:else}
		<div class="w-full h-full flex justify-center items-center">
			<Spinner className="size-5" />
		</div>
	{/if}
</div>
