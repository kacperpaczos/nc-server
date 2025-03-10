<!--
  - @copyright Copyright (c) 2018 John Molakvoæ <skjnldsv@protonmail.com>
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
	<div id="app-content" class="user-list-grid" @scroll.passive="onScroll">
		<NcModal v-if="showConfig.showNewUserForm" size="small" @close="closeModal">
			<form id="new-user"
				:disabled="loading.all"
				class="modal__content"
				@submit.prevent="createUser">
				<h2>{{ t('settings','New user') }}</h2>
				<input id="newusername"
					ref="newusername"
					v-model="newUser.id"
					:disabled="settings.newUserGenerateUserID"
					:placeholder="settings.newUserGenerateUserID
						? t('settings', 'Will be autogenerated')
						: t('settings', 'Username')"
					autocapitalize="none"
					autocomplete="off"
					autocorrect="off"
					class="modal__item"
					name="username"
					pattern="[a-zA-Z0-9 _\.@\-']+"
					required
					type="text">
				<input id="newdisplayname"
					v-model="newUser.displayName"
					:placeholder="t('settings', 'Display name')"
					autocapitalize="none"
					autocomplete="off"
					autocorrect="off"
					class="modal__item"
					name="displayname"
					type="text">
				<input id="newuserpassword"
					ref="newuserpassword"
					v-model="newUser.password"
					:minlength="minPasswordLength"
					:maxlength="469"
					:placeholder="t('settings', 'Password')"
					:required="newUser.mailAddress===''"
					autocapitalize="none"
					autocomplete="new-password"
					autocorrect="off"
					class="modal__item"
					name="password"
					type="password">
				<input id="newemail"
					v-model="newUser.mailAddress"
					:placeholder="t('settings', 'Email')"
					:required="newUser.password==='' || settings.newUserRequireEmail"
					autocapitalize="none"
					autocomplete="off"
					autocorrect="off"
					class="modal__item"
					name="email"
					type="email">
				<div class="groups modal__item">
					<!-- hidden input trick for vanilla html5 form validation -->
					<input v-if="!settings.isAdmin"
						id="newgroups"
						:class="{'icon-loading-small': loading.groups}"
						:required="!settings.isAdmin"
						:value="newUser.groups"
						tabindex="-1"
						type="text">
					<NcMultiselect v-model="newUser.groups"
						:close-on-select="false"
						:disabled="loading.groups||loading.all"
						:multiple="true"
						:options="canAddGroups"
						:placeholder="t('settings', 'Add user to group')"
						:tag-width="60"
						:taggable="true"
						class="multiselect-vue"
						label="name"
						tag-placeholder="create"
						track-by="id"
						@tag="createGroup">
						<!-- If user is not admin, he is a subadmin.
							Subadmins can't create users outside their groups
							Therefore, empty select is forbidden -->
						<span slot="noResult">{{ t('settings', 'No results') }}</span>
					</NcMultiselect>
				</div>
				<div v-if="subAdminsGroups.length>0 && settings.isAdmin"
					class="subadmins modal__item">
					<NcMultiselect v-model="newUser.subAdminsGroups"
						:close-on-select="false"
						:multiple="true"
						:options="subAdminsGroups"
						:placeholder="t('settings', 'Set user as admin for')"
						:tag-width="60"
						class="multiselect-vue"
						label="name"
						track-by="id">
						<span slot="noResult">{{ t('settings', 'No results') }}</span>
					</NcMultiselect>
				</div>
				<div class="quota modal__item">
					<NcMultiselect v-model="newUser.quota"
						:allow-empty="false"
						:options="quotaOptions"
						:placeholder="t('settings', 'Select user quota')"
						:taggable="true"
						class="multiselect-vue"
						label="label"
						track-by="id"
						@tag="validateQuota" />
				</div>
				<div v-if="showConfig.showLanguages" class="languages modal__item">
					<NcMultiselect v-model="newUser.language"
						:allow-empty="false"
						:options="languages"
						:placeholder="t('settings', 'Default language')"
						class="multiselect-vue"
						group-label="label"
						group-values="languages"
						label="name"
						track-by="code" />
				</div>
				<div v-if="showConfig.showStoragePath" class="storageLocation" />
				<div v-if="showConfig.showUserBackend" class="userBackend" />
				<div v-if="showConfig.showLastLogin" class="lastLogin" />
				<div class="user-actions">
					<NcButton id="newsubmit"
						type="primary"
						native-type="submit"
						value="">
						{{ t('settings', 'Add a new user') }}
					</NcButton>
				</div>
			</form>
		</NcModal>
		<div id="grid-header"
			:class="{'sticky': scrolled && !showConfig.showNewUserForm}"
			class="row">
			<div id="headerAvatar" class="avatar" />
			<div id="headerName" class="name">
				<div class="subtitle">
					<strong>
						{{ t('settings', 'Display name') }}
					</strong>
				</div>
				{{ t('settings', 'Username') }}
			</div>
			<div id="headerPassword" class="password">
				{{ t('settings', 'Password') }}
			</div>
			<div id="headerAddress" class="mailAddress">
				{{ t('settings', 'Email') }}
			</div>
			<div id="headerGroups" class="groups">
				{{ t('settings', 'Groups') }}
			</div>
			<div v-if="subAdminsGroups.length>0 && settings.isAdmin"
				id="headerSubAdmins"
				class="subadmins">
				{{ t('settings', 'Group admin for') }}
			</div>
			<div id="headerQuota" class="quota">
				{{ t('settings', 'Quota') }}
			</div>
			<div v-if="showConfig.showLanguages"
				id="headerLanguages"
				class="languages">
				{{ t('settings', 'Language') }}
			</div>

			<div v-if="showConfig.showUserBackend || showConfig.showStoragePath"
				class="headerUserBackend userBackend">
				<div v-if="showConfig.showUserBackend" class="userBackend">
					{{ t('settings', 'User backend') }}
				</div>
				<div v-if="showConfig.showStoragePath"
					class="subtitle storageLocation">
					{{ t('settings', 'Storage location') }}
				</div>
			</div>
			<div v-if="showConfig.showLastLogin"
				class="headerLastLogin lastLogin">
				{{ t('settings', 'Last login') }}
			</div>

			<div class="userActions" />
		</div>

		<user-row v-for="user in filteredUsers"
			:key="user.id"
			:external-actions="externalActions"
			:groups="groups"
			:languages="languages"
			:quota-options="quotaOptions"
			:settings="settings"
			:show-config="showConfig"
			:sub-admins-groups="subAdminsGroups"
			:user="user"
			:is-dark-theme="isDarkTheme" />
		<InfiniteLoading ref="infiniteLoading" @infinite="infiniteHandler">
			<div slot="spinner">
				<div class="users-icon-loading icon-loading" />
			</div>
			<div slot="no-more">
				<div class="users-list-end" />
			</div>
			<div slot="no-results">
				<div id="emptycontent">
					<div class="icon-contacts-dark" />
					<h2>{{ t('settings', 'No users in here') }}</h2>
				</div>
			</div>
		</InfiniteLoading>
	</div>
</template>

<script>
import { subscribe, unsubscribe } from '@nextcloud/event-bus'
import InfiniteLoading from 'vue-infinite-loading'
import Vue from 'vue'
import NcModal from '@nextcloud/vue/dist/Components/NcModal.js'
import NcButton from '@nextcloud/vue/dist/Components/NcButton.js'
import NcMultiselect from '@nextcloud/vue/dist/Components/NcMultiselect.js'

import userRow from './UserList/UserRow.vue'

const unlimitedQuota = {
	id: 'none',
	label: t('settings', 'Unlimited'),
}
const defaultQuota = {
	id: 'default',
	label: t('settings', 'Default quota'),
}
const newUser = {
	id: '',
	displayName: '',
	password: '',
	mailAddress: '',
	groups: [],
	subAdminsGroups: [],
	quota: defaultQuota,
	language: {
		code: 'en',
		name: t('settings', 'Default language'),
	},
}

export default {
	name: 'UserList',
	components: {
		NcModal,
		userRow,
		NcMultiselect,
		InfiniteLoading,
		NcButton,
	},
	props: {
		users: {
			type: Array,
			default: () => [],
		},
		showConfig: {
			type: Object,
			required: true,
		},
		selectedGroup: {
			type: String,
			default: null,
		},
		externalActions: {
			type: Array,
			default: () => [],
		},
	},
	data() {
		return {
			unlimitedQuota,
			defaultQuota,
			loading: {
				all: false,
				groups: false,
			},
			scrolled: false,
			searchQuery: '',
			newUser: Object.assign({}, newUser),
		}
	},
	computed: {
		settings() {
			return this.$store.getters.getServerData
		},
		selectedGroupDecoded() {
			return decodeURIComponent(this.selectedGroup)
		},
		filteredUsers() {
			if (this.selectedGroup === 'disabled') {
				return this.users.filter(user => user.enabled === false)
			}
			if (!this.settings.isAdmin) {
				// we don't want subadmins to edit themselves
				return this.users.filter(user => user.enabled !== false)
			}
			return this.users.filter(user => user.enabled !== false)
		},
		groups() {
			// data provided php side + remove the disabled group
			return this.$store.getters.getGroups
				.filter(group => group.id !== 'disabled')
				.sort((a, b) => a.name.localeCompare(b.name))
		},
		canAddGroups() {
			// disabled if no permission to add new users to group
			return this.groups.map(group => {
				// clone object because we don't want
				// to edit the original groups
				group = Object.assign({}, group)
				group.$isDisabled = group.canAdd === false
				return group
			})
		},
		subAdminsGroups() {
			// data provided php side
			return this.$store.getters.getSubadminGroups
		},
		quotaOptions() {
			// convert the preset array into objects
			const quotaPreset = this.settings.quotaPreset.reduce((acc, cur) => acc.concat({
				id: cur,
				label: cur,
			}), [])
			// add default presets
			if (this.settings.allowUnlimitedQuota) {
				quotaPreset.unshift(this.unlimitedQuota)
			}
			quotaPreset.unshift(this.defaultQuota)
			return quotaPreset
		},
		minPasswordLength() {
			return this.$store.getters.getPasswordPolicyMinLength
		},
		usersOffset() {
			return this.$store.getters.getUsersOffset
		},
		usersLimit() {
			return this.$store.getters.getUsersLimit
		},
		usersCount() {
			return this.users.length
		},

		/* LANGUAGES */
		languages() {
			return [
				{
					label: t('settings', 'Common languages'),
					languages: this.settings.languages.commonLanguages,
				},
				{
					label: t('settings', 'Other languages'),
					languages: this.settings.languages.otherLanguages,
				},
			]
		},
		isDarkTheme() {
			return window.getComputedStyle(this.$el)
				.getPropertyValue('--background-invert-if-dark') === 'invert(100%)'
		},
	},
	watch: {
		// watch url change and group select
		selectedGroup(val, old) {
			// if selected is the disabled group but it's empty
			this.redirectIfDisabled()
			this.$store.commit('resetUsers')
			this.$refs.infiniteLoading.stateChanger.reset()
			this.setNewUserDefaultGroup(val)
		},

		// make sure the infiniteLoading state is changed if we manually
		// add/remove data from the store
		usersCount(val, old) {
			// deleting the last user, reset the list
			if (val === 0 && old === 1) {
				this.$refs.infiniteLoading.stateChanger.reset()
				// adding the first user, warn the infiniteLoader that
				// the list is not empty anymore (we don't fetch the newly
				// added user as we already have all the info we need)
			} else if (val === 1 && old === 0) {
				this.$refs.infiniteLoading.stateChanger.loaded()
			}
		},
	},

	mounted() {
		if (!this.settings.canChangePassword) {
			OC.Notification.showTemporary(t('settings', 'Password change is disabled because the master key is disabled'))
		}

		/**
		 * Reset and init new user form
		 */
		this.resetForm()

		/**
		 * Register search
		 */
		subscribe('nextcloud:unified-search.search', this.search)
		subscribe('nextcloud:unified-search.reset', this.resetSearch)

		/**
		 * If disabled group but empty, redirect
		 */
		this.redirectIfDisabled()
	},
	beforeDestroy() {
		unsubscribe('nextcloud:unified-search.search', this.search)
		unsubscribe('nextcloud:unified-search.reset', this.resetSearch)
	},

	methods: {
		onScroll(event) {
			this.scrolled = event.target.scrollTo > 0
		},

		/**
		 * Validate quota string to make sure it's a valid human file size
		 *
		 * @param {string} quota Quota in readable format '5 GB'
		 * @return {object}
		 */
		validateQuota(quota) {
			// only used for new presets sent through @Tag
			const validQuota = OC.Util.computerFileSize(quota)
			if (validQuota !== null && validQuota >= 0) {
				// unify format output
				quota = OC.Util.humanFileSize(OC.Util.computerFileSize(quota))
				this.newUser.quota = { id: quota, label: quota }
				return this.newUser.quota
			}
			// Default is unlimited
			this.newUser.quota = this.quotaOptions[0]
			return this.quotaOptions[0]
		},

		infiniteHandler($state) {
			this.$store.dispatch('getUsers', {
				offset: this.usersOffset,
				limit: this.usersLimit,
				group: this.selectedGroup !== 'disabled' ? this.selectedGroup : '',
				search: this.searchQuery,
			})
				.then((usersCount) => {
					if (usersCount > 0) {
						$state.loaded()
					}
					if (usersCount < this.usersLimit) {
						$state.complete()
					}
				})
		},

		/* SEARCH */
		search({ query }) {
			this.searchQuery = query
			this.$store.commit('resetUsers')
			this.$refs.infiniteLoading.stateChanger.reset()
		},
		resetSearch() {
			this.search({ query: '' })
		},

		resetForm() {
			// revert form to original state
			this.newUser = Object.assign({}, newUser)

			/**
			 * Init default language from server data. The use of this.settings
			 * requires a computed variable, which break the v-model binding of the form,
			 * this is a much easier solution than getter and setter on a computed var
			 */
			if (this.settings.defaultLanguage) {
				Vue.set(this.newUser.language, 'code', this.settings.defaultLanguage)
			}

			/**
			 * In case the user directly loaded the user list within a group
			 * the watch won't be triggered. We need to initialize it.
			 */
			this.setNewUserDefaultGroup(this.selectedGroup)

			this.loading.all = false
		},
		createUser() {
			this.loading.all = true
			this.$store.dispatch('addUser', {
				userid: this.newUser.id,
				password: this.newUser.password,
				displayName: this.newUser.displayName,
				email: this.newUser.mailAddress,
				groups: this.newUser.groups.map(group => group.id),
				subadmin: this.newUser.subAdminsGroups.map(group => group.id),
				quota: this.newUser.quota.id,
				language: this.newUser.language.code,
			})
				.then(() => {
					this.resetForm()
					this.$refs.newusername.focus()
					this.closeModal()
				})
				.catch((error) => {
					this.loading.all = false
					if (error.response && error.response.data && error.response.data.ocs && error.response.data.ocs.meta) {
						const statuscode = error.response.data.ocs.meta.statuscode
						if (statuscode === 102) {
							// wrong username
							this.$refs.newusername.focus()
						} else if (statuscode === 107) {
							// wrong password
							this.$refs.newuserpassword.focus()
						}
					}
				})
		},
		setNewUserDefaultGroup(value) {
			if (value && value.length > 0) {
				// setting new user default group to the current selected one
				const currentGroup = this.groups.find(group => group.id === value)
				if (currentGroup) {
					this.newUser.groups = [currentGroup]
					return
				}
			}
			// fallback, empty selected group
			this.newUser.groups = []
		},

		/**
		 * Create a new group
		 *
		 * @param {string} gid Group id
		 * @return {Promise}
		 */
		createGroup(gid) {
			this.loading.groups = true
			this.$store.dispatch('addGroup', gid)
				.then((group) => {
					this.newUser.groups.push(this.groups.find(group => group.id === gid))
					this.loading.groups = false
				})
				.catch(() => {
					this.loading.groups = false
				})
			return this.$store.getters.getGroups[this.groups.length]
		},

		/**
		 * If the selected group is the disabled group but the count is 0
		 * redirect to the all users page.
		 * we only check for 0 because we don't have the count on ldap
		 * and we therefore set the usercount to -1 in this specific case
		 */
		redirectIfDisabled() {
			const allGroups = this.$store.getters.getGroups
			if (this.selectedGroup === 'disabled'
						&& allGroups.findIndex(group => group.id === 'disabled' && group.usercount === 0) > -1) {
				// disabled group is empty, redirection to all users
				this.$router.push({ name: 'users' })
				this.$refs.infiniteLoading.stateChanger.reset()
			}
		},
		closeModal() {
			// eslint-disable-next-line vue/no-mutating-props
			this.showConfig.showNewUserForm = false
		},
	},
}
</script>
<style scoped>
	.modal-wrapper {
		margin: 2vh 0;
		align-items: flex-start;
	}
	.modal__content {
		display: flex;
		padding: 20px;
		flex-direction: column;
		align-items: center;
		text-align: center;
	}
	.modal__item {
		margin-bottom: 16px;
		width: 100%;
	}
	.modal__item:not(:focus):not(:active) {
		border-color: var(--color-border-dark);
	}
	.modal__item::v-deep .multiselect {
		width: 100%;
	}
	.user-actions {
		margin-top: 20px;
	}
	.modal__content::v-deep .multiselect__single {
		text-align: left;
		box-sizing: border-box;
	}
	.modal__content::v-deep .multiselect__content-wrapper {
		box-sizing: border-box;
	}
	.row::v-deep .multiselect__single {
		z-index: auto !important;
	}

	/* fake input for groups validation */
	input#newgroups {
		position: absolute;
		opacity: 0;
		/* The "hidden" input is behind the Multiselect, so in general it does
		 * not receives clicks. However, with Firefox, after the validation
		 * fails, it will receive the first click done on it, so its width needs
		 * to be set to 0 to prevent that ("pointer-events: none" does not
		 * prevent it). */
		width: 0;
	}
</style>
