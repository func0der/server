<!--
  - @copyright Copyright (c) 2023 John Molakvoæ <skjnldsv@protonmail.com>
  -
  - @author John Molakvoæ <skjnldsv@protonmail.com>
  -
  - @license AGPL-3.0-or-later
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
	<tr :class="{'files-list__row--visible': visible, 'files-list__row--active': isActive}"
		data-cy-files-list-row
		:data-cy-files-list-row-fileid="fileid"
		:data-cy-files-list-row-name="source.basename"
		class="list__row"
		@contextmenu="onRightClick">
		<!-- Failed indicator -->
		<span v-if="source.attributes.failed" class="files-list__row--failed" />

		<!-- Checkbox -->
		<td class="files-list__row-checkbox">
			<NcCheckboxRadioSwitch v-if="visible"
				:aria-label="t('files', 'Select the row for {displayName}', { displayName })"
				:checked="selectedFiles"
				:value="fileid"
				name="selectedFiles"
				@update:checked="onSelectionChange" />
		</td>

		<!-- Link to file -->
		<td class="files-list__row-name" data-cy-files-list-row-name>
			<!-- Icon or preview -->
			<span class="files-list__row-icon" @click="execDefaultAction">
				<template v-if="source.type === 'folder'">
					<FolderIcon />
					<OverlayIcon :is="folderOverlay"
						v-if="folderOverlay"
						class="files-list__row-icon-overlay" />
				</template>

				<!-- Decorative image, should not be aria documented -->
				<span v-else-if="previewUrl && !backgroundFailed"
					ref="previewImg"
					class="files-list__row-icon-preview"
					:style="{ backgroundImage }" />

				<FileIcon v-else />

				<!-- Favorite icon -->
				<span v-if="isFavorite"
					class="files-list__row-icon-favorite"
					:aria-label="t('files', 'Favorite')">
					<FavoriteIcon :aria-hidden="true" />
				</span>
			</span>

			<!-- Rename input -->
			<form v-show="isRenaming"
				v-on-click-outside="stopRenaming"
				:aria-hidden="!isRenaming"
				:aria-label="t('files', 'Rename file')"
				class="files-list__row-rename"
				@submit.prevent.stop="onRename">
				<NcTextField ref="renameInput"
					:label="renameLabel"
					:autofocus="true"
					:minlength="1"
					:required="true"
					:value.sync="newName"
					enterkeyhint="done"
					@keyup="checkInputValidity"
					@keyup.esc="stopRenaming" />
			</form>

			<a v-show="!isRenaming"
				ref="basename"
				:aria-hidden="isRenaming"
				class="files-list__row-name-link"
				data-cy-files-list-row-name-link
				v-bind="linkTo"
				@click="execDefaultAction">
				<!-- File name -->
				<span class="files-list__row-name-text">
					<!-- Keep the displayName stuck to the extension to avoid whitespace rendering issues-->
					<span class="files-list__row-name-" v-text="displayName" />
					<span class="files-list__row-name-ext" v-text="extension" />
				</span>
			</a>
		</td>

		<!-- Actions -->
		<td v-show="!isRenamingSmallScreen"
			:class="`files-list__row-actions-${uniqueId}`"
			class="files-list__row-actions"
			data-cy-files-list-row-actions>
			<!-- Render actions -->
			<CustomElementRender v-for="action in enabledRenderActions"
				:key="action.id"
				:current-view="currentView"
				:render="action.renderInline"
				:source="source" />

			<!-- Menu actions -->
			<NcActions v-if="visible"
				ref="actionsMenu"
				:boundaries-element="getBoundariesElement()"
				:container="getBoundariesElement()"
				:disabled="source._loading"
				:force-menu="enabledInlineActions.length === 0 /* forceMenu only if no inline actions */"
				:inline="enabledInlineActions.length"
				:open.sync="openedMenu">
				<NcActionButton v-for="action in enabledMenuActions"
					:key="action.id"
					:class="'files-list__row-action-' + action.id"
					:close-after-click="true"
					:data-cy-files-list-row-action="action.id"
					@click="onActionClick(action)">
					<template #icon>
						<NcLoadingIcon v-if="loading === action.id" :size="18" />
						<CustomSvgIconRender v-else :svg="action.iconSvgInline([source], currentView)" />
					</template>
					{{ action.displayName([source], currentView) }}
				</NcActionButton>
			</NcActions>
		</td>

		<!-- Size -->
		<td v-if="isSizeAvailable"
			:style="{ opacity: sizeOpacity }"
			class="files-list__row-size"
			data-cy-files-list-row-size
			@click="openDetailsIfAvailable">
			<span>{{ size }}</span>
		</td>

		<!-- Mtime -->
		<td v-if="isMtimeAvailable"
			class="files-list__row-mtime"
			data-cy-files-list-row-mtime
			@click="openDetailsIfAvailable">
			<span>{{ mtime }}</span>
		</td>

		<!-- View columns -->
		<td v-for="column in columns"
			:key="column.id"
			:class="`files-list__row-${currentView?.id}-${column.id}`"
			class="files-list__row-column-custom"
			:data-cy-files-list-row-column-custom="column.id"
			@click="openDetailsIfAvailable">
			<CustomElementRender v-if="visible"
				:current-view="currentView"
				:render="column.render"
				:source="source" />
		</td>
	</tr>
</template>

<script lang='ts'>
import type { PropType } from 'vue'

import { CancelablePromise } from 'cancelable-promise'
import { debounce } from 'debounce'
import { emit } from '@nextcloud/event-bus'
import { extname } from 'path'
import { generateUrl } from '@nextcloud/router'
import { getFileActions, DefaultType, FileType, formatFileSize, Permission, Folder, File, Node } from '@nextcloud/files'
import { Type as ShareType } from '@nextcloud/sharing'
import { showError, showSuccess } from '@nextcloud/dialogs'
import { translate } from '@nextcloud/l10n'
import { vOnClickOutside } from '@vueuse/components'
import axios from '@nextcloud/axios'
import moment from '@nextcloud/moment'
import Vue from 'vue'

import AccountGroupIcon from 'vue-material-design-icons/AccountGroup.vue'
import FileIcon from 'vue-material-design-icons/File.vue'
import FolderIcon from 'vue-material-design-icons/Folder.vue'
import KeyIcon from 'vue-material-design-icons/Key.vue'
import LinkIcon from 'vue-material-design-icons/Link.vue'
import NetworkIcon from 'vue-material-design-icons/Network.vue'
import AccountPlusIcon from 'vue-material-design-icons/AccountPlus.vue'
import NcActionButton from '@nextcloud/vue/dist/Components/NcActionButton.js'
import NcActions from '@nextcloud/vue/dist/Components/NcActions.js'
import NcCheckboxRadioSwitch from '@nextcloud/vue/dist/Components/NcCheckboxRadioSwitch.js'
import NcLoadingIcon from '@nextcloud/vue/dist/Components/NcLoadingIcon.js'
import NcTextField from '@nextcloud/vue/dist/Components/NcTextField.js'

import { action as sidebarAction } from '../actions/sidebarAction.ts'
import { hashCode } from '../utils/hashUtils.ts'
import { isCachedPreview } from '../services/PreviewService.ts'
import { useActionsMenuStore } from '../store/actionsmenu.ts'
import { useFilesStore } from '../store/files.ts'
import { useKeyboardStore } from '../store/keyboard.ts'
import { useRenamingStore } from '../store/renaming.ts'
import { useSelectionStore } from '../store/selection.ts'
import { useUserConfigStore } from '../store/userconfig.ts'
import CustomElementRender from './CustomElementRender.vue'
import CustomSvgIconRender from './CustomSvgIconRender.vue'
import FavoriteIcon from './FavoriteIcon.vue'
import logger from '../logger.js'

// The registered actions list
const actions = getFileActions()

Vue.directive('onClickOutside', vOnClickOutside)

export default Vue.extend({
	name: 'FileEntry',

	components: {
		AccountGroupIcon,
		AccountPlusIcon,
		CustomElementRender,
		CustomSvgIconRender,
		FavoriteIcon,
		FileIcon,
		FolderIcon,
		KeyIcon,
		LinkIcon,
		NcActionButton,
		NcActions,
		NcCheckboxRadioSwitch,
		NcLoadingIcon,
		NcTextField,
		NetworkIcon,
	},

	props: {
		visible: {
			type: Boolean,
			default: false,
		},
		isMtimeAvailable: {
			type: Boolean,
			default: false,
		},
		isSizeAvailable: {
			type: Boolean,
			default: false,
		},
		source: {
			type: [Folder, File, Node] as PropType<Node>,
			required: true,
		},
		index: {
			type: Number,
			required: true,
		},
		nodes: {
			type: Array as PropType<Node[]>,
			required: true,
		},
		filesListWidth: {
			type: Number,
			default: 0,
		},
	},

	setup() {
		const actionsMenuStore = useActionsMenuStore()
		const filesStore = useFilesStore()
		const keyboardStore = useKeyboardStore()
		const renamingStore = useRenamingStore()
		const selectionStore = useSelectionStore()
		const userConfigStore = useUserConfigStore()
		return {
			actionsMenuStore,
			filesStore,
			keyboardStore,
			renamingStore,
			selectionStore,
			userConfigStore,
		}
	},

	data() {
		return {
			backgroundFailed: false,
			backgroundImage: '',
			loading: '',
		}
	},

	computed: {
		userConfig() {
			return this.userConfigStore.userConfig
		},

		currentView() {
			return this.$navigation.active
		},
		columns() {
			// Hide columns if the list is too small
			if (this.filesListWidth < 512) {
				return []
			}
			return this.currentView?.columns || []
		},

		currentDir() {
			// Remove any trailing slash but leave root slash
			return (this.$route?.query?.dir?.toString() || '/').replace(/^(.+)\/$/, '$1')
		},
		currentFileId() {
			return this.$route.params.fileid || this.$route.query.fileid || null
		},
		fileid() {
			return this.source?.fileid?.toString?.()
		},

		extension() {
			if (this.source.attributes?.displayName) {
				return extname(this.source.attributes.displayName)
			}
			return this.source.extension || ''
		},
		displayName() {
			const ext = this.extension
			const name = (this.source.attributes.displayName
				|| this.source.basename)

			// Strip extension from name if defined
			return !ext ? name : name.slice(0, 0 - ext.length)
		},

		size() {
			const size = parseInt(this.source.size, 10) || 0
			if (typeof size !== 'number' || size < 0) {
				return this.t('files', 'Pending')
			}
			return formatFileSize(size, true)
		},
		sizeOpacity() {
			// Whatever theme is active, the contrast will pass WCAG AA
			// with color main text over main background and an opacity of 0.7
			const minOpacity = 0.7
			const maxOpacitySize = 10 * 1024 * 1024

			const size = parseInt(this.source.size, 10) || 0
			if (!size || size < 0) {
				return minOpacity
			}

			return minOpacity + (1 - minOpacity) * Math.pow((this.source.size / maxOpacitySize), 2)
		},

		mtime() {
			if (this.source.mtime) {
				return moment(this.source.mtime).fromNow()
			}
			return this.t('files_trashbin', 'A long time ago')
		},
		mtimeTitle() {
			if (this.source.mtime) {
				return moment(this.source.mtime).format('LLL')
			}
			return ''
		},

		folderOverlay() {
			if (this.source.type !== FileType.Folder) {
				return null
			}

			// Encrypted folders
			if (this.source?.attributes?.['is-encrypted'] === 1) {
				return KeyIcon
			}

			// Link and mail shared folders
			const shareTypes = Object.values(this.source?.attributes?.['share-types'] || {}).flat() as number[]
			if (shareTypes.some(type => type === ShareType.SHARE_TYPE_LINK || type === ShareType.SHARE_TYPE_EMAIL)) {
				return LinkIcon
			}

			// Shared folders
			if (shareTypes.length > 0) {
				return AccountPlusIcon
			}

			switch (this.source?.attributes?.['mount-type']) {
			case 'external':
			case 'external-session':
				return NetworkIcon
			case 'group':
				return AccountGroupIcon
			}

			return null
		},

		linkTo() {
			if (this.source.attributes.failed) {
				return {
					title: this.t('files', 'This node is unavailable'),
					is: 'span',
				}
			}

			if (this.enabledDefaultActions.length > 0) {
				const action = this.enabledDefaultActions[0]
				const displayName = action.displayName([this.source], this.currentView)
				return {
					title: displayName,
					role: 'button',
				}
			}

			if (this.source?.permissions & Permission.READ) {
				return {
					download: this.source.basename,
					href: this.source.source,
					title: this.t('files', 'Download file {name}', { name: this.displayName }),
				}
			}

			return {
				is: 'span',
			}
		},

		selectedFiles() {
			return this.selectionStore.selected
		},
		isSelected() {
			return this.selectedFiles.includes(this.fileid)
		},

		cropPreviews() {
			return this.userConfig.crop_image_previews
		},
		previewUrl() {
			if (this.source.type === FileType.Folder) {
				return null
			}

			try {
				const previewUrl = this.source.attributes.previewUrl
					|| generateUrl('/core/preview?fileId={fileid}', {
						fileid: this.source.fileid,
					})
				const url = new URL(window.location.origin + previewUrl)

				// Request tiny previews
				url.searchParams.set('x', '32')
				url.searchParams.set('y', '32')
				url.searchParams.set('mimeFallback', 'true')

				// Handle cropping
				url.searchParams.set('a', this.cropPreviews === true ? '0' : '1')
				return url.href
			} catch (e) {
				return null
			}
		},

		// Sorted actions that are enabled for this node
		enabledActions() {
			if (this.source.attributes.failed) {
				return []
			}

			return actions
				.filter(action => !action.enabled || action.enabled([this.source], this.currentView))
				.sort((a, b) => (a.order || 0) - (b.order || 0))
		},

		// Enabled action that are displayed inline
		enabledInlineActions() {
			if (this.filesListWidth < 768) {
				return []
			}
			return this.enabledActions.filter(action => action?.inline?.(this.source, this.currentView))
		},

		// Enabled action that are displayed inline with a custom render function
		enabledRenderActions() {
			if (!this.visible) {
				return []
			}
			return this.enabledActions.filter(action => typeof action.renderInline === 'function')
		},

		// Default actions
		enabledDefaultActions() {
			return this.enabledActions.filter(action => !!action?.default)
		},

		// Actions shown in the menu
		enabledMenuActions() {
			return [
				// Showing inline first for the NcActions inline prop
				...this.enabledInlineActions,
				// Then the rest
				...this.enabledActions.filter(action => action.default !== DefaultType.HIDDEN && typeof action.renderInline !== 'function'),
			].filter((value, index, self) => {
				// Then we filter duplicates to prevent inline actions to be shown twice
				return index === self.findIndex(action => action.id === value.id)
			})
		},
		openedMenu: {
			get() {
				return this.actionsMenuStore.opened === this.uniqueId
			},
			set(opened) {
				this.actionsMenuStore.opened = opened ? this.uniqueId : null
			},
		},

		uniqueId() {
			return hashCode(this.source.source)
		},

		isFavorite() {
			return this.source.attributes.favorite === 1
		},

		renameLabel() {
			const matchLabel: Record<FileType, string> = {
				[FileType.File]: t('files', 'File name'),
				[FileType.Folder]: t('files', 'Folder name'),
			}
			return matchLabel[this.source.type]
		},

		isRenaming() {
			return this.renamingStore.renamingNode === this.source
		},
		isRenamingSmallScreen() {
			return this.isRenaming && this.filesListWidth < 512
		},
		newName: {
			get() {
				return this.renamingStore.newName
			},
			set(newName) {
				this.renamingStore.newName = newName
			},
		},

		isActive() {
			return this.fileid === this.currentFileId?.toString?.()
		},
	},

	watch: {
		/**
		 * When the source changes, reset the preview
		 * and fetch the new one.
		 */
		source() {
			this.resetState()
			this.debounceIfNotCached()
		},

		/**
		 * If renaming starts, select the file name
		 * in the input, without the extension.
		 */
		isRenaming(renaming) {
			if (renaming) {
				this.startRenaming()
			}
		},
	},

	/**
	 * The row is mounted once and reused as we scroll.
	 */
	mounted() {
		// ⚠ Init the debounce function on mount and
		// not when the module is imported  to
		// avoid sharing between recycled components
		this.debounceGetPreview = debounce(function() {
			this.fetchAndApplyPreview()
		}, 150, false)

		// Fetch the preview on init
		this.debounceIfNotCached()
	},

	beforeDestroy() {
		this.resetState()
	},

	methods: {
		async debounceIfNotCached() {
			if (!this.previewUrl) {
				return
			}

			// Check if we already have this preview cached
			const isCached = await isCachedPreview(this.previewUrl)
			if (isCached) {
				this.backgroundImage = `url(${this.previewUrl})`
				this.backgroundFailed = false
				return
			}

			// We don't have this preview cached or it expired, requesting it
			this.debounceGetPreview()
		},

		fetchAndApplyPreview() {
			// Ignore if no preview
			if (!this.previewUrl) {
				return
			}

			// If any image is being processed, reset it
			if (this.previewPromise) {
				this.clearImg()
			}

			// Store the promise to be able to cancel it
			this.previewPromise = new CancelablePromise((resolve, reject, onCancel) => {
				const img = new Image()
				// If visible, load the preview with higher priority
				img.fetchpriority = this.visible ? 'high' : 'auto'
				img.onload = () => {
					this.backgroundImage = `url(${this.previewUrl})`
					this.backgroundFailed = false
					resolve(img)
				}
				img.onerror = () => {
					this.backgroundFailed = true
					reject(img)
				}
				img.src = this.previewUrl

				// Image loading has been canceled
				onCancel(() => {
					img.onerror = null
					img.onload = null
					img.src = ''
				})
			})
		},

		resetState() {
			// Reset loading state
			this.loading = ''

			// Reset the preview
			this.clearImg()

			// Close menu
			this.openedMenu = false
		},

		clearImg() {
			this.backgroundImage = ''
			this.backgroundFailed = false

			if (this.previewPromise) {
				this.previewPromise.cancel()
				this.previewPromise = null
			}
		},

		async onActionClick(action) {
			const displayName = action.displayName([this.source], this.currentView)
			try {
				// Set the loading marker
				this.loading = action.id
				Vue.set(this.source, '_loading', true)

				const success = await action.exec(this.source, this.currentView, this.currentDir)

				// If the action returns null, we stay silent
				if (success === null) {
					return
				}

				if (success) {
					showSuccess(this.t('files', '"{displayName}" action executed successfully', { displayName }))
					return
				}
				showError(this.t('files', '"{displayName}" action failed', { displayName }))
			} catch (e) {
				logger.error('Error while executing action', { action, e })
				showError(this.t('files', '"{displayName}" action failed', { displayName }))
			} finally {
				// Reset the loading marker
				this.loading = ''
				Vue.set(this.source, '_loading', false)
			}
		},
		execDefaultAction(event) {
			if (this.enabledDefaultActions.length > 0) {
				event.preventDefault()
				event.stopPropagation()
				// Execute the first default action if any
				this.enabledDefaultActions[0].exec(this.source, this.currentView, this.currentDir)
			}
		},

		openDetailsIfAvailable(event) {
			event.preventDefault()
			event.stopPropagation()
			if (sidebarAction?.enabled?.([this.source], this.currentView)) {
				sidebarAction.exec(this.source, this.currentView, this.currentDir)
			}
		},

		onSelectionChange(selection) {
			const newSelectedIndex = this.index
			const lastSelectedIndex = this.selectionStore.lastSelectedIndex

			// Get the last selected and select all files in between
			if (this.keyboardStore?.shiftKey && lastSelectedIndex !== null) {
				const isAlreadySelected = this.selectedFiles.includes(this.fileid)

				const start = Math.min(newSelectedIndex, lastSelectedIndex)
				const end = Math.max(lastSelectedIndex, newSelectedIndex)

				const lastSelection = this.selectionStore.lastSelection
				const filesToSelect = this.nodes
					.map(file => file.fileid?.toString?.())
					.slice(start, end + 1)

				// If already selected, update the new selection _without_ the current file
				const selection = [...lastSelection, ...filesToSelect]
					.filter(fileId => !isAlreadySelected || fileId !== this.fileid)

				logger.debug('Shift key pressed, selecting all files in between', { start, end, filesToSelect, isAlreadySelected })
				// Keep previous lastSelectedIndex to be use for further shift selections
				this.selectionStore.set(selection)
				return
			}

			logger.debug('Updating selection', { selection })
			this.selectionStore.set(selection)
			this.selectionStore.setLastIndex(newSelectedIndex)
		},

		// Open the actions menu on right click
		onRightClick(event) {
			// If already opened, fallback to default browser
			if (this.openedMenu) {
				return
			}

			// If the clicked row is in the selection, open global menu
			const isMoreThanOneSelected = this.selectedFiles.length > 1
			this.actionsMenuStore.opened = this.isSelected && isMoreThanOneSelected ? 'global' : this.uniqueId

			// Prevent any browser defaults
			event.preventDefault()
			event.stopPropagation()
		},

		/**
		 * Check if the file name is valid and update the
		 * input validity using browser's native validation.
		 * @param event the keyup event
		 */
		checkInputValidity(event?: KeyboardEvent) {
			const input = event.target as HTMLInputElement
			const newName = this.newName.trim?.() || ''
			logger.debug('Checking input validity', { newName })
			try {
				this.isFileNameValid(newName)
				input.setCustomValidity('')
				input.title = ''
			} catch (e) {
				input.setCustomValidity(e.message)
				input.title = e.message
			} finally {
				input.reportValidity()
			}
		},
		isFileNameValid(name) {
			const trimmedName = name.trim()
			if (trimmedName === '.' || trimmedName === '..') {
				throw new Error(this.t('files', '"{name}" is an invalid file name.', { name }))
			} else if (trimmedName.length === 0) {
				throw new Error(this.t('files', 'File name cannot be empty.'))
			} else if (trimmedName.indexOf('/') !== -1) {
				throw new Error(this.t('files', '"/" is not allowed inside a file name.'))
			} else if (trimmedName.match(OC.config.blacklist_files_regex)) {
				throw new Error(this.t('files', '"{name}" is not an allowed filetype.', { name }))
			} else if (this.checkIfNodeExists(name)) {
				throw new Error(this.t('files', '{newName} already exists.', { newName: name }))
			}

			return true
		},
		checkIfNodeExists(name) {
			return this.nodes.find(node => node.basename === name && node !== this.source)
		},

		startRenaming() {
			this.$nextTick(() => {
				// Using split to get the true string length
				const extLength = (this.source.extension || '').split('').length
				const length = this.source.basename.split('').length - extLength
				const input = this.$refs.renameInput?.$refs?.inputField?.$refs?.input
				if (!input) {
					logger.error('Could not find the rename input')
					return
				}
				input.setSelectionRange(0, length)
				input.focus()

				// Trigger a keyup event to update the input validity
				input.dispatchEvent(new Event('keyup'))
			})
		},
		stopRenaming() {
			if (!this.isRenaming) {
				return
			}

			// Reset the renaming store
			this.renamingStore.$reset()
		},

		// Rename and move the file
		async onRename() {
			const oldName = this.source.basename
			const oldSource = this.source.source
			const newName = this.newName.trim?.() || ''
			if (newName === '') {
				showError(this.t('files', 'Name cannot be empty'))
				return
			}

			if (oldName === newName) {
				this.stopRenaming()
				return
			}

			// Checking if already exists
			if (this.checkIfNodeExists(newName)) {
				showError(this.t('files', 'Another entry with the same name already exists'))
				return
			}

			// Set loading state
			this.loading = 'renaming'
			Vue.set(this.source, '_loading', true)

			// Update node
			this.source.rename(newName)

			try {
				await axios({
					method: 'MOVE',
					url: oldSource,
					headers: {
						Destination: encodeURI(this.source.source),
					},
				})

				// Success 🎉
				emit('files:node:updated', this.source)
				emit('files:node:renamed', this.source)
				showSuccess(this.t('files', 'Renamed "{oldName}" to "{newName}"', { oldName, newName }))

				// Reset the renaming store
				this.stopRenaming()
				this.$nextTick(() => {
					this.$refs.basename.focus()
				})
			} catch (error) {
				logger.error('Error while renaming file', { error })
				this.source.rename(oldName)
				this.$refs.renameInput.focus()

				// TODO: 409 means current folder does not exist, redirect ?
				if (error?.response?.status === 404) {
					showError(this.t('files', 'Could not rename "{oldName}", it does not exist any more', { oldName }))
					return
				} else if (error?.response?.status === 412) {
					showError(this.t('files', 'The name "{newName}" is already used in the folder "{dir}". Please choose a different name.', { newName, dir: this.currentDir }))
					return
				}

				// Unknown error
				showError(this.t('files', 'Could not rename "{oldName}"', { oldName }))
			} finally {
				this.loading = false
				Vue.set(this.source, '_loading', false)
			}
		},

		/**
		 * Making this a function in case the files-list
		 * reference changes in the future. That way we're
		 * sure there is one at the time we call it.
		 */
		getBoundariesElement() {
			return document.querySelector('.app-content > .files-list')
		},

		t: translate,
		formatFileSize,
	},
})
</script>

<style scoped lang='scss'>
/* Hover effect on tbody lines only */
tr {
	&:hover,
	&:focus {
		background-color: var(--color-background-dark);
	}
}

// Folder overlay
.files-list__row-icon-overlay {
	position: absolute;
	max-height: 18px;
	max-width: 18px;
	color: var(--color-main-background);
	// better alignment with the folder icon
	margin-top: 2px;
}

/* Preview not loaded animation effect */
.files-list__row-icon-preview:not([style*='background']) {
    background: var(--color-loading-dark);
	// animation: preview-gradient-fade 1.2s ease-in-out infinite;
}
</style>

<style>
/* @keyframes preview-gradient-fade {
    0% {
        opacity: 1;
    }
    50% {
        opacity: 0.5;
    }
    100% {
        opacity: 1;
    }
} */
</style>
