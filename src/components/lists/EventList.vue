<style>
td.log-cell {
	padding-top: 8px !important;
	padding-bottom: 8px !important;
	height: auto !important;
}

td.title-cell {
	vertical-align: top;
}
</style>

<style scoped>
.message {
	white-space: pre-wrap;
}
</style>

<template>
	<div class="component">
		<v-data-table :headers="headers" :items="events" :pagination.sync="pagination" hide-actions class="elevation-3" :class="{ 'empty-table-fix' : !events.length }">
			<template slot="no-data">
				<v-alert :value="true" type="info" class="ma-0" @contextmenu.prevent="">
					{{ $t('list.eventLog.noEvents') }}
				</v-alert>
			</template>

			<template slot="headers" slot-scope="props">
				<tr>
					<th v-for="header in props.headers" :key="header.value" :class="['text-xs-left column sortable', pagination.descending ? 'desc' : 'asc', header.value === pagination.sortBy ? 'active' : '']" :width="header.width" @click="changeSort(header.value)" v-tab-control>
						{{ getHeaderText(header) }}
						<v-icon small>arrow_upward</v-icon>
					</th>
				</tr>
			</template>

			<template slot="items" slot-scope="{ item }">
				<tr :class="getClassByEvent(item.type)" @touchstart="onItemTouchStart(item, $event)" @touchend="onItemTouchEnd" @contextmenu.prevent="onItemContextmenu(item, $event)" v-tab-control.contextmenu>
					<td class="log-cell title-cell">
						{{ item.date.toLocaleString() }}
					</td>
					<td class="log-cell content-cell">
						<strong>{{ item.title }}</strong>
						<br v-if="item.title && item.message"/>
						<span v-if="item.message" class="message" v-html="formatMessage(item.message)"></span>
					</td>
				</tr>
			</template>
		</v-data-table>

		<v-menu v-model="contextMenu.shown" :position-x="contextMenu.x" :position-y="contextMenu.y" absolute offset-y v-tab-control.contextmenu>
			<v-list>
				<v-list-tile @click="clearLog" tabindex="0">
					<v-icon class="mr-1">clear_all</v-icon> {{ $t('list.eventLog.clear') }}
				</v-list-tile>
				<v-list-tile :disabled="!events.length" @click="downloadText" tabindex="0">
					<v-icon class="mr-1">font_download</v-icon> {{ $t('list.eventLog.downloadText') }}
				</v-list-tile>
				<v-list-tile :disabled="!events.length" @click="downloadCSV" tabindex="0">
					<v-icon class="mr-1">cloud_download</v-icon> {{ $t('list.eventLog.downloadCSV') }}
				</v-list-tile>
			</v-list>
		</v-menu>
	</div>
</template>

<script>
'use strict'

import i18n from '../../i18n'

import saveAs from 'file-saver'
import { mapState, mapMutations } from 'vuex'

export default {
	computed: {
		...mapState('machine', ['events']),
		...mapState('machine/cache', ['sorting']),
		...mapState('settings', ['darkTheme'])
	},
	data() {
		return {
			contextMenu: {
				shown: false,
				touchTimer: undefined,
				item: null,
				x: 0,
				y: 0
			},
			headers: [
				{
					text: () => i18n.t('list.eventLog.date'),
					value: 'date',
					width: '15%'
				},
				{
					text: () => i18n.t('list.eventLog.message'),
					value: 'message',
					sortable: false,
					width: '75%'
				}
			],
			pagination: {
				sortBy: 'date',
				descending: true,
				rowsPerPage: -1
			}
		}
	},
	methods: {
		...mapMutations('machine', ['clearLog']),
		...mapMutations('machine/cache', ['setSorting']),
		getHeaderText: (header) => (header.text instanceof(Function)) ? header.text() : header.text,
		getClassByEvent(type) {
			if (this.darkTheme) {
				switch (type) {
					case 'success': return 'green darken-1';
					case 'warning': return 'amber darken-1';
					case 'error': return 'red darken-1';
				}
				return 'blue darken-1';
			} else {
				switch (type) {
					case 'success': return 'green accent-2';
					case 'warning': return 'amber accent-1';
					case 'error': return 'red accent-1';
				}
				return 'light-blue accent-1';
			}
		},
		formatMessage(message) {
			return message.replace(/Error:/g, '<strong>Error:</strong>').replace(/Warning:/g, '<strong>Warning:</strong>');
		},
		changeSort(column) {
			if (this.pagination.sortBy === column) {
				this.pagination.descending = !this.pagination.descending;
			} else {
				this.pagination.sortBy = column;
				this.pagination.descending = false;
			}
		},
		onItemTouchStart(item, e) {
			const that = this;
			this.contextMenu.touchTimer = setTimeout(function() {
				that.contextMenu.touchTimer = undefined;
				that.onItemContextmenu(item, { clientX: e.targetTouches[0].clientX, clientY: e.targetTouches[0].clientY });
			}, 1000);
		},
		onItemTouchEnd() {
			if (this.contextMenu.touchTimer) {
				clearTimeout(this.contextMenu.touchTimer);
				this.contextMenu.touchTimer = undefined;
			}
		},
		onItemContextmenu(item, e) {
			this.onItemTouchEnd();

			this.contextMenu.shown = false;
			this.contextMenu.item = item;
			this.contextMenu.x = e.clientX;
			this.contextMenu.y = e.clientY;
			this.$nextTick(() => {
				this.contextMenu.shown = true;
			});
		},
		downloadText() {
			let textContent = '';
			this.events.forEach(function(e) {
				const title = e.title.replace(/\n/g, '\r\n');
				const message = e.message ? e.message.replace(/\n/g, '\r\n') : '';
				textContent += `${e.date.toLocaleString()}: ${message ? (title + ": " + message) : title}\r\n`;
			});

			const file = new File([textContent], "console.txt", {type: "text/plain;charset=utf-8"});
			saveAs(file);
		},
		downloadCSV() {
			var csvContent = '"date","time","title","message"\r\n';
			this.events.forEach(function(e) {
				const title = e.title.replace(/"/g, '""').replace(/\n/g, '\r\n');
				const message = e.message ? e.message.replace(/"/g, '""').replace(/\n/g, '\r\n') : '';
				csvContent += `"${e.date.toLocaleDateString()}","${e.date.toLocaleTimeString()}","${title}","${message}"\r\n`;
			});

			const file = new File([csvContent], "console.csv", {type: "text/csv;charset=utf-8"});
			saveAs(file);
		}
	},
	mounted() {
		this.pagination.sortBy = this.sorting.events.column;
		this.pagination.descending = this.sorting.events.descending;
	},
	watch: {
		pagination: {
			deep: true,
			handler(to) {
				if (this.sorting.events.column !== to.sortBy || this.sorting.events.descending !== to.descending) {
					this.setSorting({ table: 'events', column: to.sortBy, descending: to.descending });
				}
			}
		},
		'sorting.events': {
			deep: true,
			handler(to) {
				this.pagination.sortBy = to.column;
				this.pagination.descending = to.descending;
			}
		}
	}
}
</script>
