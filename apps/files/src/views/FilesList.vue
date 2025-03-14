<!--
  - @copyright Copyright (c) 2023 John Molakvoæ <skjnldsv@protonmail.com>
  -
  - @author John Molakvoæ <skjnldsv@protonmail.com>
  -
  - @license GNU AGPL version 3 or any later version
  -
  - This program is free software: you can redistribute it and/or modify
  - it under the terms of the GNU Affero General Public License as
  - published by the Free Software Foundation, either version 3 of the
  - License, or (at your option) any later version.
  -
  - This program is distributed in the hope that it will be useful,
  - but WITHOUT ANY WARRANTY; without even the implied warranty of
  - MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the
  - GNU Affero General Public License for more details.
  -
  - You should have received a copy of the GNU Affero General Public License
  - along with this program. If not, see <http://www.gnu.org/licenses/>.
  -
  -->
<template>
	<NcAppContent v-show="!currentView?.legacy"
		:class="{'app-content--hidden': currentView?.legacy}"
		data-cy-files-content>
		<div class="files-list__header">
			<!-- Current folder breadcrumbs -->
			<BreadCrumbs :path="dir" @reload="fetchContent" />

			<!-- Secondary loading indicator -->
			<NcLoadingIcon v-if="isRefreshing" class="files-list__refresh-icon" />
		</div>

		<!-- Initial loading -->
		<NcLoadingIcon v-if="loading && !isRefreshing"
			class="files-list__loading-icon"
			:size="38"
			:title="t('files', 'Loading current folder')" />

		<!-- Empty content placeholder -->
		<NcEmptyContent v-else-if="!loading && isEmptyDir"
			:title="t('files', 'No files in here')"
			:description="t('files', 'No files or folders have been deleted yet')"
			data-cy-files-content-empty>
			<template #action>
				<NcButton v-if="dir !== '/'"
					aria-label="t('files', 'Go to the previous folder')"
					type="primary"
					:to="toPreviousDir">
					{{ t('files', 'Go back') }}
				</NcButton>
			</template>
			<template #icon>
				<TrashCan />
			</template>
		</NcEmptyContent>

		<!-- File list -->
		<FilesListVirtual v-else
			ref="filesListVirtual"
			:current-view="currentView"
			:nodes="dirContents" />
	</NcAppContent>
</template>

<script lang="ts">
import { Folder, File, Node } from '@nextcloud/files'
import { join } from 'path'
import { orderBy } from 'natural-orderby'
import { translate } from '@nextcloud/l10n'
import NcAppContent from '@nextcloud/vue/dist/Components/NcAppContent.js'
import NcButton from '@nextcloud/vue/dist/Components/NcButton.js'
import NcEmptyContent from '@nextcloud/vue/dist/Components/NcEmptyContent.js'
import NcLoadingIcon from '@nextcloud/vue/dist/Components/NcLoadingIcon.js'
import TrashCan from 'vue-material-design-icons/TrashCan.vue'
import Vue from 'vue'

import Navigation, { ContentsWithRoot } from '../services/Navigation.ts'
import { useFilesStore } from '../store/files.ts'
import { usePathsStore } from '../store/paths.ts'
import { useSelectionStore } from '../store/selection.ts'
import { useSortingStore } from '../store/sorting.ts'
import BreadCrumbs from '../components/BreadCrumbs.vue'
import FilesListVirtual from '../components/FilesListVirtual.vue'
import logger from '../logger.js'

export default Vue.extend({
	name: 'FilesList',

	components: {
		BreadCrumbs,
		FilesListVirtual,
		NcAppContent,
		NcButton,
		NcEmptyContent,
		NcLoadingIcon,
		TrashCan,
	},

	setup() {
		const pathsStore = usePathsStore()
		const filesStore = useFilesStore()
		const selectionStore = useSelectionStore()
		const sortingStore = useSortingStore()
		return {
			filesStore,
			pathsStore,
			selectionStore,
			sortingStore,
		}
	},

	data() {
		return {
			loading: true,
			promise: null,
		}
	},

	computed: {
		/** @return {Navigation} */
		currentView() {
			return this.$navigation.active
				|| this.$navigation.views.find(view => view.id === 'files')
		},

		/**
		 * The current directory query.
		 *
		 * @return {string}
		 */
		dir() {
			// Remove any trailing slash but leave root slash
			return (this.$route?.query?.dir || '/').replace(/^(.+)\/$/, '$1')
		},

		/**
		 * The current folder.
		 *
		 * @return {Folder|undefined}
		 */
		currentFolder() {
			if (!this.currentView?.id) {
				return
			}

			if (this.dir === '/') {
				return this.filesStore.getRoot(this.currentView.id)
			}
			const fileId = this.pathsStore.getPath(this.currentView.id, this.dir)
			return this.filesStore.getNode(fileId)
		},

		sortingMode() {
			return this.sortingStore.getSortingMode(this.currentView.id)
				|| this.currentView.defaultSortKey
				|| 'basename'
		},
		isAscSorting() {
			return this.sortingStore.isAscSorting(this.currentView.id) === true
		},

		/**
		 * The current directory contents.
		 *
		 * @return {Node[]}
		 */
		dirContents() {
			if (!this.currentView) {
				return []
			}

			const customColumn = this.currentView.columns
				.find(column => column.id === this.sortingMode)

			// Custom column must provide their own sorting methods
			if (customColumn?.sort && typeof customColumn.sort === 'function') {
				const results = [...(this.currentFolder?._children || []).map(this.getNode).filter(file => file)]
					.sort(customColumn.sort)
				return this.isAscSorting ? results : results.reverse()
			}

			return orderBy(
				[...(this.currentFolder?._children || []).map(this.getNode).filter(file => file)],
				[
					// Sort folders first if sorting by name
					...this.sortingMode === 'basename' ? [v => v.type !== 'folder'] : [],
					// Use sorting mode
					v => v[this.sortingMode],
					// Fallback to name
					v => v.basename,
				],
				this.isAscSorting ? ['asc', 'asc', 'asc'] : ['desc', 'desc', 'desc'],
			)
		},

		/**
		 * The current directory is empty.
		 */
		isEmptyDir() {
			return this.dirContents.length === 0
		},

		/**
		 * We are refreshing the current directory.
		 * But we already have a cached version of it
		 * that is not empty.
		 */
		isRefreshing() {
			return this.currentFolder !== undefined
				&& !this.isEmptyDir
				&& this.loading
		},

		/**
		 * Route to the previous directory.
		 */
		toPreviousDir() {
			const dir = this.dir.split('/').slice(0, -1).join('/') || '/'
			return { ...this.$route, query: { dir } }
		},
	},

	watch: {
		currentView(newView, oldView) {
			if (newView?.id === oldView?.id) {
				return
			}

			logger.debug('View changed', { newView, oldView })
			this.selectionStore.reset()
			this.fetchContent()
		},

		dir(newDir, oldDir) {
			logger.debug('Directory changed', { newDir, oldDir })
			// TODO: preserve selection on browsing?
			this.selectionStore.reset()
			this.fetchContent()

			// Scroll to top, force virtual scroller to re-render
			if (this.$refs?.filesListVirtual?.$el) {
				this.$refs.filesListVirtual.$el.scrollTop = 0
			}
		},
	},

	methods: {
		async fetchContent() {
			if (this.currentView?.legacy) {
				return
			}

			this.loading = true
			const dir = this.dir
			const currentView = this.currentView

			// If we have a cancellable promise ongoing, cancel it
			if (typeof this.promise?.cancel === 'function') {
				this.promise.cancel()
				logger.debug('Cancelled previous ongoing fetch')
			}

			// Fetch the current dir contents
			/** @type {Promise<ContentsWithRoot>} */
			this.promise = currentView.getContents(dir)
			try {
				const { folder, contents } = await this.promise
				logger.debug('Fetched contents', { dir, folder, contents })

				// Update store
				this.filesStore.updateNodes(contents)

				// Define current directory children
				folder._children = contents.map(node => node.attributes.fileid)

				// If we're in the root dir, define the root
				if (dir === '/') {
					this.filesStore.setRoot({ service: currentView.id, root: folder })
				} else
				// Otherwise, add the folder to the store
				if (folder.attributes.fileid) {
					this.filesStore.updateNodes([folder])
					this.pathsStore.addPath({ service: currentView.id, fileid: folder.attributes.fileid, path: dir })
				} else {
					// If we're here, the view API messed up
					logger.error('Invalid root folder returned', { dir, folder, currentView })
				}

				// Update paths store
				const folders = contents.filter(node => node.type === 'folder')
				folders.forEach(node => {
					this.pathsStore.addPath({ service: currentView.id, fileid: node.attributes.fileid, path: join(dir, node.basename) })
				})
			} catch (error) {
				logger.error('Error while fetching content', { error })
			} finally {
				this.loading = false
			}

		},

		/**
		 * Get a cached note from the store
		 *
		 * @param {number} fileId the file id to get
		 * @return {Folder|File}
		 */
		 getNode(fileId) {
			return this.filesStore.getNode(fileId)
		},

		t: translate,
	},
})
</script>

<style scoped lang="scss">
.app-content {
	// Virtual list needs to be full height and is scrollable
	display: flex;
	overflow: hidden;
	flex-direction: column;
	max-height: 100%;

	// TODO: remove after all legacy views are migrated
	// Hides the legacy app-content if shown view is not legacy
	&:not(&--hidden)::v-deep + #app-content {
		display: none;
	}
}

$margin: 4px;
$navigationToggleSize: 50px;

.files-list {
	&__header {
		display: flex;
		align-content: center;
		// Do not grow or shrink (vertically)
		flex: 0 0;
		// Align with the navigation toggle icon
		margin: $margin $margin $margin $navigationToggleSize;
		> * {
			// Do not grow or shrink (horizontally)
			// Only the breadcrumbs shrinks
			flex: 0 0;
		}
	}
	&__refresh-icon {
		flex: 0 0 44px;
		width: 44px;
		height: 44px;
	}
	&__loading-icon {
		margin: auto;
	}
}

</style>
