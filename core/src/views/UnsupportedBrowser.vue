 <!--
  - @copyright 2022 John Molakvoæ <skjnldsv@protonmail.com>
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
  - along with this program.  If not, see <http://www.gnu.org/licenses/>.
  -
  -->
<template>
	<div class="content-unsupported-browser guest-box">
		<NcEmptyContent>
			{{ t('core', 'This browser is not supported') }}
			<template #icon>
				<Web />
			</template>
			<template #action>
				<div>
					<h2>
						{{ t('core', 'Your browser is not supported. Please upgrade to a newer version or a supported one.') }}
					</h2>
					<NcButton class="content-unsupported-browser__continue" type="primary" @click="forceBrowsing">
						{{ t('core', 'Continue with this unsupported browser') }}
					</NcButton>
				</div>

				<ul class="content-unsupported-browser__list">
					<h3>{{ t('core', 'Supported versions') }}</h3>
					<li v-for="browser in formattedBrowsersList" :key="browser">
						{{ browser }}
					</li>
				</ul>
			</template>
		</NcEmptyContent>
	</div>
</template>

<script>
import { generateUrl } from '@nextcloud/router'
import { translate as t, translatePlural as n } from '@nextcloud/l10n'
import NcButton from '@nextcloud/vue/dist/Components/NcButton.js'
import NcEmptyContent from '@nextcloud/vue/dist/Components/NcEmptyContent.js'
import Web from 'vue-material-design-icons/Web.vue'

import { browserStorageKey } from '../utils/RedirectUnsupportedBrowsers.js'
import { supportedBrowsers } from '../services/BrowsersListService.js'
import browserStorage from '../services/BrowserStorageService.js'
import logger from '../logger.js'

logger.debug('Supported browsers', { supportedBrowsers })

export default {
	name: 'UnsupportedBrowser',
	components: {
		Web,
		NcButton,
		NcEmptyContent,
	},

	data() {
		return {
			agents: {},
		}
	},

	computed: {
		isMobile() {
			return /Android|webOS|iPhone|iPad|iPod|BlackBerry|IEMobile|Opera Mini/i.test(navigator.userAgent)
		},

		/**
		 * Filter out or include mobile/desktop browsers depending
		 * on the current user platform/device
		 */
		filteredSupportedBrowsers() {
			return supportedBrowsers.filter(browser => {
				if (!browser) {
					return false
				}

				if (this.isMobile) {
					return this.isMobileBrowser(browser)
				}
				return !this.isMobileBrowser(browser)
			})
		},

		formattedBrowsersList() {
			const list = {}

			// supportedBrowsers is generated by webpack at compilation time
			this.filteredSupportedBrowsers.forEach(browser => {
				const [id, version] = browser.split(' ')
				if (!list[id] || list[id] < parseFloat(version, 10)) {
					list[id] = parseFloat(version, 10)
				}
			})

			return Object.keys(list).map(id => {
				if (!this.agents[id]?.browser) {
					return null
				}

				const version = list[id]
				const name = this.agents[id]?.browser
				return this.t('core', '{name} version {version} and above', {
					name, version,
				})
			}).filter(entry => entry !== null)
		},
	},

	async beforeMount() {
		// Dynamic load big list of user agents
		// eslint-disable-next-line n/no-extraneous-import
		const { agents } = await import('caniuse-lite')
		this.agents = agents
	},

	methods: {
		t,
		n,

		// Set the flag allowing this browser and redirect to home
		forceBrowsing() {
			browserStorage.setItem(browserStorageKey, true)

			// Redirect if there is the data
			const urlParams = new URLSearchParams(window.location.search)
			if (urlParams.has('redirect_url')) {
				const redirectPath = Buffer.from(urlParams.get('redirect_url'), 'base64').toString() || '/'
				window.location = redirectPath
				return
			}
			window.location = generateUrl('/')
		},

		/**
		 * Detect if the browserslist browser is a mobile one
		 * https://github.com/browserslist/browserslist#query-composition
		 *
		 * @param {string} browser a valid browserlist browser. e.g `and_chr 90`
		 */
		isMobileBrowser(browser) {
			browser = browser.toLowerCase()
			return browser.includes('and_')
				|| browser.includes('android')
				|| browser.includes('ios_')
				|| browser.includes('mobile')
				|| browser.includes('_mob')
				|| browser.includes('samsung')
		},
	},
}
</script>

<style lang="scss" scoped>
$spacing: 30px;

.content-unsupported-browser {
	display: flex;
	justify-content: center;
	width: 400px;
	max-width: calc(90vw - 2 * $spacing);
	margin: auto;
	padding: $spacing;

	.empty-content {
		margin: 0;
		&::v-deep .empty-content__icon {
			opacity: 1;
		}
	}

	&__continue {
		display: block;
		margin: $spacing auto;
	}

	&__list {
		margin-top: 2 * $spacing;
		margin-bottom: $spacing;
		li {
			text-align: left;
		}
	}
}

</style>
