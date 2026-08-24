<script lang="ts">
	import dayjs from 'dayjs';
	import relativeTime from 'dayjs/plugin/relativeTime';
	import { toast } from 'svelte-sonner';
	import { onMount, getContext, tick, onDestroy } from 'svelte';
	import { goto } from '$app/navigation';

	dayjs.extend(relativeTime);

	const i18n = getContext('i18n');

	import { WEBUI_NAME, user, models, workspaceActions } from '$lib/stores';
	import {
		getChatList,
		getChatById,
		deleteChatById,
		cloneChatById,
		archiveChatById,
		updateChatById
	} from '$lib/apis/chats';

	import Tooltip from '../common/Tooltip.svelte';
	import DeleteConfirmDialog from '$lib/components/common/ConfirmDialog.svelte';
	import EllipsisHorizontal from '../icons/EllipsisHorizontal.svelte';
	import GarbageBin from '../icons/GarbageBin.svelte';
	import Search from '../icons/Search.svelte';
	import XMark from '../icons/XMark.svelte';
	import Spinner from '../common/Spinner.svelte';
	import ViewSelector from './common/ViewSelector.svelte';
	import Badge from '$lib/components/common/Badge.svelte';
	import Pagination from '../common/Pagination.svelte';
	import ChevronDown from '../icons/ChevronDown.svelte';
	import ChevronUp from '../icons/ChevronUp.svelte';
	import Dropdown from '$lib/components/common/Dropdown.svelte';
	import DropdownMenu from '$lib/components/common/DropdownMenu.svelte';
	import DocumentDuplicate from '$lib/components/icons/DocumentDuplicate.svelte';
	import ArchiveBox from '$lib/components/icons/ArchiveBox.svelte';
	import ChatHoverPreview from '$lib/components/layout/Sidebar/ChatHoverPreview.svelte';
	import ChatList from '$lib/components/chat/Placeholder/ChatList.svelte';

	let loaded = false;
	let loading = false;
	let query = '';

	let rawChats = [];
	let filteredItems = null;

	let sortKey: 'updated_at' | 'title' | 'model' = 'updated_at';
	let sortDirection: 'asc' | 'desc' = 'desc';

	let viewOption = '';
	let page = 1;
	let perPage = 30;

	let selectedChat = null;
	let showDeleteConfirm = false;
	let openChatMenuId: string | null = null;

	// Inline title editing state
	let editingChatId: string | null = null;
	let editingTitle = '';
	let editInputEl: HTMLInputElement | null = null;

	// Hover preview state
	let hoverChatId: string | null = null;

	$: if (loaded) {
		workspaceActions.set([]);
	}

	const loadChats = async () => {
		loading = true;
		try {
			// Fetch user chat list summary
			const res = await getChatList(localStorage.token, null, true, true).catch((err) => {
				toast.error(`${err}`);
				return [];
			});

			if (res) {
				rawChats = (res || []).map((item) => ({
					...item,
					title: item.title || item.chat?.title || '',
					model: ''
				}));

				// Asynchronously fetch full chat data for model info & title fallback in batch
				Promise.all(
					(res || []).map(async (item) => {
						const fullChat = await getChatById(localStorage.token, item.id).catch(() => null);
						if (fullChat?.chat) {
							let usedModel = '';
							const m = fullChat.chat.models ?? fullChat.models;
							if (Array.isArray(m) && m.length > 0) {
								usedModel = m[0];
							} else if (typeof m === 'string') {
								usedModel = m;
							}

							const idx = rawChats.findIndex((c) => c.id === item.id);
							if (idx !== -1) {
								rawChats[idx].model = usedModel;
								if (fullChat.chat.title) {
									rawChats[idx].title = fullChat.chat.title;
								}
							}
						}
					})
				).then(() => {
					rawChats = [...rawChats];
				});
			}
		} catch (err) {
			console.error(err);
		} finally {
			loading = false;
		}
	};

	$: processItems(rawChats, query, sortKey, sortDirection);

	const processItems = (chatsList, searchQuery, key, direction) => {
		let items = [...(chatsList || [])];

		if (searchQuery.trim()) {
			const q = searchQuery.toLowerCase().trim();
			items = items.filter((chat) => {
				const titleMatch = (chat.title || '').toLowerCase().includes(q);
				const modelMatch = (chat.model || '').toLowerCase().includes(q);
				return titleMatch || modelMatch;
			});
		}

		items.sort((a, b) => {
			let valA = a[key] ?? '';
			let valB = b[key] ?? '';

			if (key === 'title') {
				valA = (a.title || '').toLowerCase();
				valB = (b.title || '').toLowerCase();
			} else if (key === 'model') {
				valA = (a.model || '').toLowerCase();
				valB = (b.model || '').toLowerCase();
			} else if (key === 'updated_at') {
				valA = a.updated_at || a.created_at || 0;
				valB = b.updated_at || b.created_at || 0;
			}

			if (valA < valB) return direction === 'asc' ? -1 : 1;
			if (valA > valB) return direction === 'asc' ? 1 : -1;
			return 0;
		});

		filteredItems = items;
	};

	const handleSearchInput = () => {
		page = 1;
	};

	const handleSort = (key: 'title' | 'updated_at' | 'model') => {
		if (sortKey === key) {
			sortDirection = sortDirection === 'asc' ? 'desc' : 'asc';
		} else {
			sortKey = key;
			sortDirection = key === 'updated_at' ? 'desc' : 'asc';
		}
	};

	const getModelName = (modelId: string) => {
		if (!modelId) return '';
		const modelObj = ($models || []).find((m) => m.id === modelId);
		return modelObj?.name || modelId;
	};

	const openChat = (chat) => {
		goto(`/c/${chat.id}`);
	};

	const shouldIgnoreRowClick = (target: EventTarget | null) => {
		return target instanceof Element && !!target.closest('button, a, input, [role="menu"]');
	};

	const startEditingTitle = (chat, event?: Event) => {
		if (event) {
			event.stopPropagation();
			event.preventDefault();
		}
		editingChatId = chat.id;
		editingTitle = chat.title;
		tick().then(() => {
			if (editInputEl) {
				editInputEl.focus();
				editInputEl.select();
			}
		});
	};

	const saveTitle = async (chat) => {
		const newTitle = editingTitle.trim();
		editingChatId = null;

		if (!newTitle || newTitle === chat.title) return;

		const oldTitle = chat.title;
		chat.title = newTitle;
		rawChats = [...rawChats];

		const res = await updateChatById(localStorage.token, chat.id, {
			title: newTitle
		}).catch((err) => {
			toast.error(`${err}`);
			chat.title = oldTitle;
			rawChats = [...rawChats];
			return null;
		});

		if (res) {
			toast.success($i18n.t('Chat title updated'));
		}
	};

	const cloneHandler = async (chat) => {
		const res = await cloneChatById(
			localStorage.token,
			chat.id,
			$i18n.t('Clone of {{TITLE}}', { TITLE: chat.title })
		).catch((err) => {
			toast.error(`${err}`);
			return null;
		});

		if (res) {
			toast.success($i18n.t('Chat cloned successfully'));
			loadChats();
		}
	};

	const archiveHandler = async (chat) => {
		const res = await archiveChatById(localStorage.token, chat.id).catch((err) => {
			toast.error(`${err}`);
			return null;
		});

		if (res) {
			toast.success($i18n.t('Chat archived successfully'));
			loadChats();
		}
	};

	const deleteHandler = async (chat) => {
		const res = await deleteChatById(localStorage.token, chat.id).catch((err) => {
			toast.error(`${err}`);
			return null;
		});

		if (res) {
			toast.success($i18n.t('Chat deleted successfully'));
			rawChats = rawChats.filter((c) => c.id !== chat.id);
		}
	};

	onMount(async () => {
		viewOption = localStorage?.workspaceViewOption || '';
		loaded = true;
		await loadChats();
	});
</script>

<svelte:head>
	<title>
		{$i18n.t('Chats')} / {$WEBUI_NAME}
	</title>
</svelte:head>

{#if loaded}
	<div class="space-y-2">
		<div class="flex h-8 w-full items-center gap-2">
			<div class="flex min-w-0 flex-1">
				<div class="self-center ml-1 mr-3">
					<Search className="size-3.5" />
				</div>
				<input
					class="w-full text-sm pr-4 py-1 rounded-r-xl outline-hidden bg-transparent"
					bind:value={query}
					on:input={handleSearchInput}
					aria-label={$i18n.t('Search Chats')}
					placeholder={$i18n.t('Search Chats')}
				/>
				{#if query}
					<div class="self-center pl-1.5 translate-y-[0.5px] rounded-l-xl bg-transparent">
						<button
							class="p-0.5 rounded-full hover:bg-gray-100 dark:hover:bg-gray-900 transition"
							aria-label={$i18n.t('Clear search')}
							on:click={() => {
								query = '';
								handleSearchInput();
							}}
						>
							<XMark className="size-3" strokeWidth="2" />
						</button>
					</div>
				{/if}
			</div>

			<div class="flex shrink-0">
				<ViewSelector
					bind:value={viewOption}
					align="end"
					onChange={async (value) => {
						localStorage.workspaceViewOption = value;
						page = 1;
						await tick();
					}}
				/>
			</div>
		</div>

		{#if filteredItems === null || loading}
			<div class="w-full h-full flex justify-center items-center my-16 mb-24">
				<Spinner className="size-5" />
			</div>
		{:else if (filteredItems ?? []).length !== 0}
			{@const pagedItems = filteredItems.slice((page - 1) * perPage, page * perPage)}
			
			<ChatList
				chats={pagedItems}
				total={filteredItems.length}
				{page}
				{perPage}
				showModel={true}
				orderBy={sortKey}
				direction={sortDirection}
				onPageChange={(p) => {
					page = p;
				}}
				onSort={(k) => {
					handleSort(k);
				}}
			/>
		{:else}
			<div class="flex w-full flex-col items-center justify-center py-16 pb-24">
				<div class="max-w-sm text-center text-gray-900 dark:text-gray-100">
					<div class="mb-1.5 text-sm">{$i18n.t('No chats found')}</div>
					<div class="text-center text-xs leading-5 text-gray-500">
						{$i18n.t('Try adjusting your search or filter to find what you are looking for.')}
					</div>
				</div>
			</div>
		{/if}
	</div>

	<DeleteConfirmDialog
		bind:show={showDeleteConfirm}
		title={$i18n.t('Delete chat?')}
		on:confirm={() => {
			if (selectedChat) {
				deleteHandler(selectedChat);
			}
		}}
	>
		<div class="text-sm text-gray-500 truncate">
			{$i18n.t('This will delete')} <span class="font-normal">{selectedChat?.title}</span>.
		</div>
	</DeleteConfirmDialog>
{:else}
	<div class="w-full h-full flex justify-center items-center">
		<Spinner className="size-5" />
	</div>
{/if}
