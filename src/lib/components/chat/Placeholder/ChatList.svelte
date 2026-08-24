<script lang="ts">
	import { getContext } from 'svelte';
	import type { Writable } from 'svelte/store';

	type ChatListI18n = {
		t: (key: string, options?: Record<string, unknown>) => string;
	};

	const i18n: Writable<ChatListI18n> = getContext('i18n');

	import dayjs from 'dayjs';
	import localizedFormat from 'dayjs/plugin/localizedFormat';
	import { getTimeRange } from '$lib/utils';
	import ChevronUp from '$lib/components/icons/ChevronUp.svelte';
	import ChevronDown from '$lib/components/icons/ChevronDown.svelte';
	import ChevronLeft from '$lib/components/icons/ChevronLeft.svelte';
	import ChevronRight from '$lib/components/icons/ChevronRight.svelte';
	import Tooltip from '$lib/components/common/Tooltip.svelte';
	import Spinner from '$lib/components/common/Spinner.svelte';
	import Pencil from '$lib/components/icons/Pencil.svelte';
	import Check from '$lib/components/icons/Check.svelte';
	import XMark from '$lib/components/icons/XMark.svelte';
	import { chatId, socket } from '$lib/stores';
	import ChatMenu from '$lib/components/layout/Sidebar/ChatMenu.svelte';
	import MoreHorizontalIcon from '$lib/components/layout/Sidebar/icons/MoreHorizontal.svelte';
	import DeleteConfirmDialog from '$lib/components/common/ConfirmDialog.svelte';
	import ShareChatModal from '$lib/components/chat/ShareChatModal.svelte';
	import { deleteChatById, archiveChatById, cloneChatById, updateChatById } from '$lib/apis/chats';
	import { refreshChatList } from '$lib/stores/chatList';
	import { toast } from 'svelte-sonner';

	dayjs.extend(localizedFormat);

	type ChatListItem = {
		id: string;
		title?: string;
		updated_at?: number | null;
		created_at?: number | null;
		last_read_at?: number | null;
		active?: boolean;
		time_range?: string;
		user_id?: string;
		owner_name?: string;
		[key: string]: unknown;
	};

	export let chats: ChatListItem[] = [];

	export let chatListLoading = false;
	export let showOwnerInfo = false;
	export let showModel = false;
	export let page = 1;
	export let total = 0;
	export let perPage = 10;
	export let orderBy: 'title' | 'updated_at' | 'model' = 'updated_at';
	export let direction: 'asc' | 'desc' = 'desc';
	export let onPageChange: (page: number) => void | Promise<void> = () => {};
	export let onSort: (key: 'title' | 'updated_at' | 'model') => void | Promise<void> = () => {};

	let chatList: ChatListItem[] | null = null;
	let totalPages = 1;
	let pages: (number | 'ellipsis')[] = [];

	let showShareChatModal = false;
	let showDeleteConfirm = false;
	let selectedChatForAction: ChatListItem | null = null;

	let editingChatId: string | null = null;
	let editingTitle = '';

	const startRename = (chat: ChatListItem) => {
		editingChatId = chat.id;
		editingTitle = chat.title || '';
	};

	const cancelRename = () => {
		editingChatId = null;
		editingTitle = '';
	};

	const saveRename = async (chat: ChatListItem) => {
		if (!editingTitle.trim()) {
			cancelRename();
			return;
		}
		const newTitle = editingTitle.trim();
		const currentTitle = chat.title || '';
		editingChatId = null;

		if (newTitle === currentTitle) {
			return;
		}

		// Immediate local binding update
		chat.title = newTitle;
		chatList = chatList;

		const res = await updateChatById(localStorage.token, chat.id, {
			title: newTitle
		}).catch((err) => {
			toast.error(`${err}`);
			chat.title = currentTitle;
			chatList = chatList;
			return null;
		});

		if (res) {
			toast.success($i18n.t('Chat title updated'));
			await refreshChatList(localStorage.token);
		}
	};

	const handleClone = async (chat: ChatListItem) => {
		const res = await cloneChatById(
			localStorage.token,
			chat.id,
			$i18n.t('Clone of {{TITLE}}', { TITLE: chat.title || '' })
		).catch((err) => {
			toast.error(`${err}`);
			return null;
		});

		if (res) {
			toast.success($i18n.t('Chat cloned successfully'));
			await refreshChatList(localStorage.token);
			init();
		}
	};

	const handleArchive = async (chat: ChatListItem) => {
		const res = await archiveChatById(localStorage.token, chat.id).catch((err) => {
			toast.error(`${err}`);
			return null;
		});

		if (res) {
			toast.success($i18n.t('Chat archived successfully'));
			await refreshChatList(localStorage.token);
			init();
		}
	};

	const handleDelete = async (chat: ChatListItem) => {
		const res = await deleteChatById(localStorage.token, chat.id).catch((err) => {
			toast.error(`${err}`);
			return null;
		});

		if (res) {
			toast.success($i18n.t('Chat deleted successfully'));
			if (chatList) {
				chatList = chatList.filter((c) => c.id !== chat.id);
			}
			await refreshChatList(localStorage.token);
		}
	};

	const init = async () => {
		if (chats.length === 0) {
			chatList = [];
		} else {
			chatList = chats.map((chat) => ({
				...chat,
				time_range: getTimeRange(chat.updated_at)
			}));
		}
	};

	const setSortKey = (key: 'title' | 'updated_at' | 'model') => {
		onSort(key);
	};

	const markChatRead = (chat: ChatListItem, unread: boolean) => {
		if (!unread) {
			return;
		}

		const lastReadAt = Date.now() / 1000;
		chatList = (chatList ?? []).map((item) =>
			item.id === chat.id ? { ...item, last_read_at: lastReadAt } : item
		);

		$socket?.emit('events:chat', {
			chat_id: chat.id,
			data: { type: 'last_read_at' }
		});
	};

	const buildPages = (currentPage: number, pageCount: number): (number | 'ellipsis')[] => {
		if (pageCount <= 7) {
			return Array.from({ length: pageCount }, (_, i) => i + 1);
		}

		const items: (number | 'ellipsis')[] = [1];
		if (currentPage > 3) {
			items.push('ellipsis');
		}

		const start = Math.max(2, currentPage - 1);
		const end = Math.min(pageCount - 1, currentPage + 1);
		for (let i = start; i <= end; i++) {
			items.push(i);
		}

		if (currentPage < pageCount - 2) {
			items.push('ellipsis');
		}
		items.push(pageCount);

		return items;
	};

	$: if (chats) {
		init();
	}

	$: totalPages = Math.max(1, Math.ceil(total / perPage));
	$: pages = buildPages(page, totalPages);
</script>

{#if chatList}
	{#if chatList.length > 0}
		<div class="flex text-xs font-normal mb-1 items-center px-3">
			<button
				class="py-1 cursor-pointer select-none min-w-0 flex-1 text-left"
				on:click={() => setSortKey('title')}
			>
				<div class="flex gap-1.5 items-center">
					{$i18n.t('Title')}

					{#if orderBy === 'title'}
						<span class="font-normal"
							>{#if direction === 'asc'}
								<ChevronUp className="size-2" />
							{:else}
								<ChevronDown className="size-2" />
							{/if}
						</span>
					{:else}
						<span class="invisible">
							<ChevronUp className="size-2" />
						</span>
					{/if}
				</div>
			</button>

			{#if showModel}
				<button
					class="py-1 cursor-pointer select-none hidden sm:flex w-44 shrink-0 justify-start"
					on:click={() => setSortKey('model')}
				>
					<div class="flex gap-1.5 items-center">
						{$i18n.t('Model')}

						{#if orderBy === 'model'}
							<span class="font-normal"
								>{#if direction === 'asc'}
									<ChevronUp className="size-2" />
								{:else}
									<ChevronDown className="size-2" />
								{/if}
							</span>
						{:else}
							<span class="invisible">
								<ChevronUp className="size-2" />
							</span>
						{/if}
					</div>
				</button>
			{/if}

			<button
				class="py-1 cursor-pointer select-none hidden sm:flex w-36 shrink-0 justify-end"
				on:click={() => setSortKey('updated_at')}
			>
				<div class="flex gap-1.5 items-center">
					{$i18n.t('Updated at')}

					{#if orderBy === 'updated_at'}
						<span class="font-normal"
							>{#if direction === 'asc'}
								<ChevronUp className="size-2" />
							{:else}
								<ChevronDown className="size-2" />
							{/if}
						</span>
					{:else}
						<span class="invisible">
							<ChevronUp className="size-2" />
						</span>
					{/if}
				</div>
			</button>

			<div class="w-6 shrink-0"></div>
		</div>
	{/if}

	<div class="text-left text-sm w-full mb-3">
		{#if chatList.length === 0}
			<div
				class="text-xs text-gray-500 dark:text-gray-400 text-center px-5 min-h-20 w-full h-full flex justify-center items-center"
			>
				{$i18n.t('No chats found')}
			</div>
		{/if}

		{#each chatList as chat, idx (chat.id)}
			{@const unread =
				chat.id !== $chatId &&
				!chat.active &&
				(chat.last_read_at == null ||
					(chat.updated_at != null && chat.updated_at > chat.last_read_at))}
			{#if (idx === 0 || (idx > 0 && chat.time_range !== chatList[idx - 1].time_range)) && chat?.time_range}
				<div
					class="w-full text-xs text-gray-500 dark:text-gray-500 font-normal {idx === 0
						? ''
						: 'pt-5'} pb-2 px-2"
				>
					{$i18n.t(chat.time_range)}
					<!-- localisation keys for time_range to be recognized from the i18next parser (so they don't get automatically removed):
							{$i18n.t('Today')}
							{$i18n.t('Yesterday')}
							{$i18n.t('Previous 7 days')}
							{$i18n.t('Previous 30 days')}
							{$i18n.t('January')}
							{$i18n.t('February')}
							{$i18n.t('March')}
							{$i18n.t('April')}
							{$i18n.t('May')}
							{$i18n.t('June')}
							{$i18n.t('July')}
							{$i18n.t('August')}
							{$i18n.t('September')}
							{$i18n.t('October')}
							{$i18n.t('November')}
							{$i18n.t('December')}
							-->
				</div>
			{/if}

			<div
				class="group w-full flex justify-between items-center rounded-lg text-sm py-2 px-3 hover:bg-gray-50 dark:hover:bg-gray-850 transition gap-2"
			>
				{#if editingChatId === chat.id}
					<div class="flex min-w-0 items-center flex-1 gap-1.5 pr-2">
						<input
							type="text"
							class="w-full text-sm bg-transparent border border-gray-300 dark:border-gray-600 rounded px-2 py-0.5 outline-none focus:border-blue-500 text-gray-800 dark:text-gray-100"
							bind:value={editingTitle}
							on:keydown={(e) => {
								if (e.key === 'Enter') {
									saveRename(chat);
								} else if (e.key === 'Escape') {
									cancelRename();
								}
							}}
							autoFocus
						/>
						<button
							class="p-1 text-gray-500 hover:text-green-600 dark:hover:text-green-400"
							on:click|preventDefault|stopPropagation={() => saveRename(chat)}
							type="button"
							title={$i18n.t('Save')}
						>
							<Check className="size-4" />
						</button>
						<button
							class="p-1 text-gray-500 hover:text-red-600 dark:hover:text-red-400"
							on:click|preventDefault|stopPropagation={() => cancelRename()}
							type="button"
							title={$i18n.t('Cancel')}
						>
							<XMark className="size-4" />
						</button>
					</div>
				{:else}
					<div class="flex min-w-0 items-center flex-1 pr-2">
						<a
							class="flex min-w-0 items-center max-w-full"
							draggable="false"
							href={`/c/${chat.id}`}
							on:click={() => markChatRead(chat, unread)}
						>
							{#if chat.active}
								<div class="shrink-0 self-center pr-2">
									<Spinner className="size-3" />
								</div>
							{:else if unread}
								<div class="shrink-0 self-center pr-2.5 flex transition-opacity duration-300">
									<div class="size-1.5 bg-sky-500 rounded-full"></div>
								</div>
							{/if}

							<div
								class="truncate {unread
									? 'font-normal text-gray-800 dark:text-gray-200'
									: ''}"
							>
								{chat?.title}
							</div>
						</a>

						<button
							class="p-1 ml-1 text-gray-400 hover:text-gray-700 dark:hover:text-gray-200 opacity-0 group-hover:opacity-100 transition-opacity shrink-0"
							on:click|preventDefault|stopPropagation={() => startRename(chat)}
							type="button"
							title={$i18n.t('Edit title')}
						>
							<Pencil className="size-3.5" />
						</button>
					</div>
				{/if}

				{#if showModel}
					<div class="hidden sm:flex w-44 shrink-0 items-center justify-start min-w-0">
						{#if chat.model || chat.models}
							{@const modelName = (Array.isArray(chat.models) ? chat.models[0] : (chat.model || chat.models))}
							{#if modelName}
								<div class="shrink-0 max-w-[160px] truncate text-[11px] font-normal px-2 py-0.5 rounded-full bg-gray-100 dark:bg-gray-800 text-gray-600 dark:text-gray-400">
									{modelName}
								</div>
							{/if}
						{/if}
					</div>
				{/if}

				<a
					class="hidden sm:flex w-36 shrink-0 items-center justify-end gap-2"
					draggable="false"
					href={`/c/${chat.id}`}
					on:click={() => markChatRead(chat, unread)}
				>
					<div class=" text-gray-500 dark:text-gray-400 text-xs whitespace-nowrap">
						{dayjs((chat.updated_at ?? chat.created_at ?? 0) * 1000).calendar()}
					</div>

					{#if showOwnerInfo && chat.user_id && chat.owner_name}
						<Tooltip content={chat.owner_name}>
							<img
								src="/api/v1/users/{chat.user_id}/profile/image"
								alt=""
								class="size-4 rounded-full shrink-0 object-cover"
							/>
						</Tooltip>
					{/if}
				</a>

				<div class="flex items-center shrink-0 opacity-0 group-hover:opacity-100 transition-opacity">
					<ChatMenu
						chatId={chat.id}
						cloneChatHandler={() => handleClone(chat)}
						shareHandler={() => {
							selectedChatForAction = chat;
							showShareChatModal = true;
						}}
						archiveChatHandler={() => handleArchive(chat)}
						deleteHandler={() => {
							selectedChatForAction = chat;
							showDeleteConfirm = true;
						}}
						onClose={() => {}}
					>
						<button
							aria-label={$i18n.t('More')}
							class="flex size-6 items-center justify-center rounded-lg hover:bg-gray-100 dark:hover:bg-gray-800 text-gray-500 dark:text-gray-400 transition"
							type="button"
						>
							<MoreHorizontalIcon className="size-3.5" strokeWidth="2" />
						</button>
					</ChatMenu>
				</div>
			</div>
		{/each}

		{#if totalPages > 1}
			<div class="flex items-center justify-center gap-1.5 pt-2">
				<button
					class="inline-flex size-7 items-center justify-center rounded-lg text-xs font-medium text-gray-500 transition hover:bg-gray-50 hover:text-gray-700 disabled:cursor-not-allowed disabled:opacity-25 dark:text-gray-500 dark:hover:bg-gray-850 dark:hover:text-gray-300"
					disabled={chatListLoading || page <= 1}
					aria-label={$i18n.t('Previous page')}
					on:click={() => onPageChange(page - 1)}
				>
					<ChevronLeft className="size-3.5" strokeWidth="2" />
				</button>

				{#each pages as item, index (item === 'ellipsis' ? `ellipsis-${index}` : item)}
					{#if item === 'ellipsis'}
						<span
							class="inline-flex size-7 items-center justify-center text-xs text-gray-400 select-none dark:text-gray-600"
						>
							...
						</span>
					{:else}
						<button
							class="inline-flex size-7 items-center justify-center rounded-lg text-xs font-medium transition hover:bg-gray-50 hover:text-gray-700 dark:hover:bg-gray-850 dark:hover:text-gray-300 {item ===
							page
								? 'bg-gray-50 text-gray-800 dark:bg-gray-850 dark:text-gray-100'
								: 'text-gray-500 dark:text-gray-500'}"
							aria-label={`Page ${item}`}
							aria-current={item === page ? 'page' : undefined}
							on:click={() => onPageChange(item)}
						>
							{item}
						</button>
					{/if}
				{/each}

				<button
					class="inline-flex size-7 items-center justify-center rounded-lg text-xs font-medium text-gray-500 transition hover:bg-gray-50 hover:text-gray-700 disabled:cursor-not-allowed disabled:opacity-25 dark:text-gray-500 dark:hover:bg-gray-850 dark:hover:text-gray-300"
					disabled={chatListLoading || page >= totalPages}
					aria-label={$i18n.t('Next page')}
					on:click={() => onPageChange(page + 1)}
				>
					<ChevronRight className="size-3.5" strokeWidth="2" />
				</button>
			</div>
		{/if}
	</div>

	{#if selectedChatForAction}
		<ShareChatModal
			bind:show={showShareChatModal}
			chatId={selectedChatForAction.id}
		/>

		<DeleteConfirmDialog
			bind:show={showDeleteConfirm}
			title={$i18n.t('Delete chat?')}
			on:confirm={() => {
				if (selectedChatForAction) {
					handleDelete(selectedChatForAction);
				}
			}}
		>
			<div class="text-sm text-gray-500 truncate">
				{$i18n.t('This will delete')} <span class="font-normal">{selectedChatForAction?.title}</span>.
			</div>
		</DeleteConfirmDialog>
	{/if}
{/if}
