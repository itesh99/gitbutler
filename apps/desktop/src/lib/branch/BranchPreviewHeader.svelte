<script lang="ts">
	import BranchLabel from './BranchLabel.svelte';
	import { Project } from '$lib/backend/projects';
	import { getForge } from '$lib/forge/interface/forge';
	import { ModeService } from '$lib/modes/service';
	import { error } from '$lib/utils/toasts';
	import { openExternalUrl } from '$lib/utils/url';
	import { BranchController } from '$lib/vbranches/branchController';
	import { getContext } from '@gitbutler/shared/context';
	import Badge from '@gitbutler/ui/Badge.svelte';
	import Button from '@gitbutler/ui/Button.svelte';
	import Icon from '@gitbutler/ui/Icon.svelte';
	import Modal from '@gitbutler/ui/Modal.svelte';
	import Tooltip from '@gitbutler/ui/Tooltip.svelte';
	import type { PullRequest } from '$lib/forge/interface/types';
	import type { BranchData } from '$lib/vbranches/types';
	import { goto } from '$app/navigation';

	interface Props {
		localBranch: BranchData | undefined;
		remoteBranch: BranchData | undefined;
		pr: PullRequest | undefined;
	}

	const { localBranch, remoteBranch, pr }: Props = $props();

	const branch = $derived(remoteBranch || localBranch!);
	const upstream = $derived(remoteBranch?.givenName);

	const branchController = getContext(BranchController);
	const project = getContext(Project);
	const forge = getForge();
	const modeSerivce = getContext(ModeService);
	const mode = modeSerivce.mode;
	const forgeBranch = $derived(upstream ? $forge?.branch(upstream) : undefined);

	let isApplying = $state(false);
	let isDeleting = $state(false);
	let deleteBranchModal = $state<Modal>();
</script>

<div class="header__wrapper">
	<div class="header card">
		<div class="header__info">
			<BranchLabel disabled name={branch.givenName} />
			<div class="header__remote-branch">
				{#if remoteBranch}
					<Tooltip text="At least some of your changes have been pushed">
						<Badge size="tag" icon="branch-small" style="neutral" kind="solid">
							{localBranch ? 'local and remote' : 'remote'}
						</Badge>
					</Tooltip>

					{#if forgeBranch}
						<Button
							size="tag"
							icon="open-link"
							style="ghost"
							outline
							shrinkable
							onclick={(e: MouseEvent) => {
								const url = forgeBranch.url;
								if (url) openExternalUrl(url);
								e.preventDefault();
								e.stopPropagation();
							}}
						>
							{branch.displayName}
						</Button>
					{/if}
				{:else}
					<div class="status-tag text-11 text-semibold remote">
						<Icon name="branch-small" /> local
					</div>
				{/if}
				{#if pr?.htmlUrl}
					<Button
						size="tag"
						icon="pr-small"
						style="ghost"
						outline
						onclick={(e: MouseEvent) => {
							const url = pr?.htmlUrl;
							if (url) openExternalUrl(url);
							e.preventDefault();
							e.stopPropagation();
						}}
					>
						View PR
					</Button>
				{/if}
			</div>
		</div>
		<div class="header__actions">
			<div class="header__buttons">
				<Button
					style="ghost"
					outline
					tooltip="Restores these changes into your working directory"
					icon="plus-small"
					loading={isApplying}
					disabled={$mode?.type !== 'OpenWorkspace'}
					onclick={async () => {
						isApplying = true;
						try {
							if (localBranch) {
								await branchController.createvBranchFromBranch(
									localBranch.name,
									remoteBranch?.name
								);
							} else {
								await branchController.createvBranchFromBranch(
									remoteBranch!.name,
									undefined,
									pr?.number
								);
							}
							goto(`/${project.id}/board`);
						} catch (e) {
							const err = 'Failed to apply branch';
							error(err);
							console.error(err, e);
						} finally {
							isApplying = false;
						}
					}}
				>
					Apply
				</Button>
				<Button
					style="ghost"
					outline
					tooltip="Deletes the local branch. If this branch is also present on a remote, it will not be deleted there."
					icon="bin-small"
					loading={isDeleting}
					disabled={!localBranch}
					onclick={async () => {
						if (localBranch) {
							deleteBranchModal?.show(branch);
						}
					}}
				>
					Delete local
				</Button>
			</div>
		</div>
	</div>
	<div class="header__top-overlay"></div>
</div>

<Modal
	width="small"
	bind:this={deleteBranchModal}
	onSubmit={async (close) => {
		try {
			isDeleting = true;
			await branchController.deleteLocalBranch(branch.name, branch.givenName);
		} catch (e) {
			const err = 'Failed to delete local branch';
			error(err);
			console.error(err, e);
		} finally {
			isDeleting = false;
			close();
		}
		goto(`/${project.id}/board`);
	}}
>
	{#snippet children(branch)}
		Are you sure you want to delete <code class="code-string">{branch.name}</code>?
	{/snippet}
	{#snippet controls(close)}
		<Button style="ghost" outline onclick={close}>Cancel</Button>
		<Button style="error" type="submit" kind="solid" loading={isDeleting}>Delete</Button>
	{/snippet}
</Modal>

<style lang="postcss">
	.header__wrapper {
		z-index: var(--z-lifted);
		position: sticky;
		top: 12px;
	}
	.header {
		z-index: var(--z-lifted);
		position: relative;
		flex-direction: column;
		gap: 2px;
	}
	.header__top-overlay {
		z-index: var(--z-ground);
		position: absolute;
		top: -16px;
		left: 0;
		width: 100%;
		height: 20px;
		background: var(--clr-bg-2);
	}
	.header__info {
		display: flex;
		flex-direction: column;
		transition: margin var(--transition-slow);
		padding: 10px;
		gap: 10px;
		overflow: hidden;
	}
	.header__actions {
		display: flex;
		gap: 4px;
		background: var(--clr-bg-2);
		padding: 14px;
		justify-content: flex-end;
		border-radius: 0 0 var(--radius-m) var(--radius-m);
		user-select: none;
	}
	.header__buttons {
		display: flex;
		position: relative;
		gap: 4px;
	}

	.header__remote-branch {
		color: var(--clr-scale-ntrl-50);
		padding-left: 2px;
		padding-right: 2px;
		display: flex;
		gap: 4px;
		text-overflow: ellipsis;
		overflow-x: hidden;
		white-space: nowrap;
		align-items: center;
	}

	.status-tag {
		cursor: default;
		display: flex;
		align-items: center;
		gap: 2px;
		padding: 2px 6px 2px 4px;
		border-radius: var(--radius-m);
	}

	.remote {
		color: var(--clr-scale-ntrl-100);
		background: var(--clr-scale-ntrl-40);
	}
</style>
