<!--
Copyright (C) 2022 Josh Boudreau <jboudreau@45drives.com>

This file is part of Cockpit Identities.

Cockpit Identities is free software: you can redistribute it and/or modify it under the terms
of the GNU General Public License as published by the Free Software Foundation, either version 3
of the License, or (at your option) any later version.

Cockpit Identities is distributed in the hope that it will be useful, but WITHOUT ANY WARRANTY;
without even the implied warranty of MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE. See the
GNU General Public License for more details.

You should have received a copy of the GNU General Public License along with Cockpit Identities.
If not, see <https://www.gnu.org/licenses/>. 
-->

<template>
	<Table
		stickyHeaders
		noShrink
	>
		<template #header>
			<div class="self-stretch flex flex-row flex-wrap gap-3 justify-between items-baseline">
				<div class="flex flex-row space-x-2 items-center">
					<div v-if="user !== null">{{ user.name === "" ? user.user : user.name }}'s Login History</div>
					<div v-else>User Login History</div>
					<button
						v-if="!processing"
						@click="saveCSV"
						title="Save data as CSV"
					>
						<DocumentDownloadIcon class="size-icon icon-default" />
					</button>
					<LoadingSpinner
						v-if="processing"
						class="size-icon"
					/>
				</div>
				<Datepicker
					v-model="range"
					range
					:partialRange="false"
					placeholder="Date Range"
					:dark="darkMode"
					:format="rangePreviewFormatter"
					autoApply
					teleport="#app"
					enableSeconds
					class="font-normal"
					:startTime="[{ hours: 0, minutes: 0, seconds: 0 }, { hours: 23, minutes: 59, seconds: 59 }]"
				/>
			</div>
		</template>
		<template #thead>
			<tr>
				<th
					v-if="user === null"
					scope="col"
				>
					<div class="flex flex-row flex-nowrap space-x-2">
						<div class="grow">User</div>
						<SimpleFilter
							:set="filters.user.set"
							v-model="filters.user.callback"
							ref="userFilterRef"
						/>
						<SortCallbackButton
							v-model="sortCallback"
							:compareFunc="compareFuncs.user"
						/>
					</div>
				</th>
				<th scope="col">
					<div class="flex flex-row flex-nowrap space-x-2">
						<div class="grow">Session Start</div>
						<SortCallbackButton
							v-model="sortCallback"
							:compareFunc="compareFuncs.sessionStart"
							initialFuncIsMine
							startReversed
						/>
					</div>
				</th>
				<th scope="col">
					<div class="flex flex-row flex-nowrap space-x-2">
						<div class="grow">Session End</div>
						<SortCallbackButton
							v-model="sortCallback"
							:compareFunc="compareFuncs.sessionEnd"
							startReversed
						/>
					</div>
				</th>
				<th scope="col">
					<div class="flex flex-row flex-nowrap space-x-2">
						<div class="grow">Time Logged In</div>
						<SortCallbackButton
							v-model="sortCallback"
							:compareFunc="compareFuncs.sessionTime"
							startReversed
						/>
					</div>
				</th>
				<th scope="col">
					<div class="flex flex-row flex-nowrap justify-between space-x-5">
						<span>IP Address</span>
						<SimpleFilter
							:set="filters.ip.set"
							v-model="filters.ip.callback"
							ref="ipFilterRef"
						/>
					</div>
				</th>
				<th scope="col">
					<div class="flex flex-row flex-nowrap justify-between space-x-5">
						<span>TTY</span>
						<SimpleFilter
							:set="filters.tty.set"
							v-model="filters.tty.callback"
							ref="ttyFilterRef"
						/>
					</div>
				</th>
				<th scope="col">
					<div class="flex flex-row flex-nowrap justify-between space-x-5">
						<span>Auth Result</span>
						<SimpleFilter
							:set="filters.authResult.set"
							v-model="filters.authResult.callback"
							ref="authResultFilterRef"
						/>
					</div>
				</th>
			</tr>
		</template>
		<template #tbody>
			<tr
				v-for="(entry, index) in historyReactive"
				v-show="!Object.keys(filters).map(key => filters[key].callback(entry[key])).includes(false)"
			>
				<td v-if="user === null">{{ entry.user }}</td>
				<td>{{ formatDate(entry.sessionStart) }}</td>
				<td>{{ entry.overrideEndText ?? formatDate(entry.sessionEnd) }}</td>
				<td>{{ entry.sessionTime ?? "" }}</td>
				<td>{{ entry.ip }}</td>
				<td>{{ entry.tty }}</td>
				<td>{{ entry.authResult }}</td>
			</tr>
		</template>
	</Table>
</template>

<script>
import { ref, reactive, watch, inject, onMounted } from 'vue';
import Datepicker from '@vuepic/vue-datepicker';
import '@vuepic/vue-datepicker/dist/main.css';
import { useSpawn, errorStringHTML, objectURLDownload } from '@45drives/cockpit-helpers';
import { FilterIcon, DocumentDownloadIcon } from '@heroicons/vue/solid';
import SimpleFilter from './SimpleFilter.vue';
import LoadingSpinner from './LoadingSpinner.vue';
import SortCallbackButton from './SortCallbackButton.vue';
import moment from 'moment';
import { darkModeInjectionKey, notificationsInjectionKey } from '../keys';
import Table from './Table.vue';

function formatDateForLast(date) {
	const year = date.getFullYear().toString().padStart(4, '0');
	const month = (date.getMonth() + 1).toString().padStart(2, '0');
	const day = date.getDate().toString().padStart(2, '0');
	const hour = date.getHours().toString().padStart(2, '0');
	const minute = date.getMinutes().toString().padStart(2, '0');
	return `${year}-${month}-${day} ${hour}:${minute}`;
}

function sessionTimeToSentence(sessionTime) {
	const matchGroups = sessionTime.match(/^(?:(\d+)\+)?(\d{2}):(\d{2})$/)?.slice(1).map(num => num === null ? num : parseInt(num)) ?? null;
	if (!matchGroups)
		return sessionTime;
	return (matchGroups[0] ? `${matchGroups[0]} Day${matchGroups[0] === 1 ? '' : 's'}, ` : "")
		+ (matchGroups[1] !== 0 ? `${matchGroups[1]} Hour${matchGroups[1] === 1 ? '' : 's'}, ` : "")
		+ `${matchGroups[2]} Minute${matchGroups[2] === 1 ? '' : 's'}`;
}

function sessionTimeToObj(sessionTime) {
	const matchGroups = sessionTime?.match(/^(?:(\d+)\+)?(\d{2}):(\d{2})$/)?.slice(1).map(num => num === null ? num : parseInt(num)) ?? null;
	if (!matchGroups)
		return { days: 0, hours: 0, minutes: 0 };
	return {
		days: isNaN(matchGroups[0]) ? 0 : matchGroups[0],
		hours: matchGroups[1],
		minutes: matchGroups[2],
	};
}

function timeSince(start) {
	const diff = new Date() - new Date(start); // milliseconds
	let minutes = Math.floor(diff / 1000 / 60);
	let hours = Math.floor(minutes / 60);
	let days = Math.floor(hours / 24);
	minutes %= 60;
	hours %= 24;
	return (days ? `${days} Day${days === 1 ? '' : 's'}, ` : "")
		+ (hours !== 0 ? `${hours} Hour${hours === 1 ? '' : 's'}, ` : "")
		+ `${minutes} Minute${minutes === 1 ? '' : 's'}`;
}

function isSameDay(a, b) {
	return a.getFullYear() === b.getFullYear()
		&& a.getMonth() === b.getMonth()
		&& a.getDate() === b.getDate();
}

function isFullDay(a, b) {
	return isSameDay(a, b)
		&& a.getHours() === 0
		&& a.getMinutes() === 0
		&& a.getSeconds() === 0
		&& b.getHours() === 23
		&& b.getMinutes() === 59
		&& b.getSeconds() === 59
}

function tryDate(date) {
	const dateObj = new Date(date);
	if (isNaN(dateObj.getTime()))
		return null;
	return dateObj;
}

export default {
	props: {
		user: {
			type: Object,
			required: false,
			default: null,
		},
		initialRangeDays: {
			type: Number,
			required: false,
			default: 7,
		},
	},
	setup(props) {
		const range = ref();
		const userFilterRef = ref();
		const ipFilterRef = ref();
		const ttyFilterRef = ref();
		const authResultFilterRef = ref();
		const history = ref([]);
		const historyReactive = reactive(history);
		const processing = ref(0);
		const darkMode = inject(darkModeInjectionKey);
		const notifications = inject(notificationsInjectionKey);
		const filters = reactive({
			user: {
				set: new Set([]),
				callback: () => true,
			},
			ip: {
				set: new Set([]),
				callback: () => true,
			},
			tty: {
				set: new Set([]),
				callback: () => true,
			},
			authResult: {
				set: new Set([]),
				callback: () => true,
			},
		});
		const compareFuncs = reactive({
			user: (entry1, entry2) => entry1.user.localeCompare(entry2.user),
			sessionStart: (entry1, entry2) => entry1.sessionStart - entry2.sessionStart,
			sessionEnd: (entry1, entry2) => entry1.sessionEnd - entry2.sessionEnd,
			sessionTime: (entry1, entry2) => {
				const match1 = entry1.sessionTime.match(/^(?:(\d+) Days?, )?(?:(\d+) Hours?, )?(\d+) Minutes?$/).slice(1);
				const match2 = entry2.sessionTime.match(/^(?:(\d+) Days?, )?(?:(\d+) Hours?, )?(\d+) Minutes?$/).slice(1);
				return ((match1[0] ? parseInt(match1[0]) * 1440 : 0) + (match1[1] ? parseInt(match1[1]) * 60 : 0) + (match1[2]))
					- ((match2[0] ? parseInt(match2[0]) * 1440 : 0) + (match2[1] ? parseInt(match2[1]) * 60 : 0) + (match2[2]));
			},
		});
		const sortCallback = ref(() => 0);

		const getHistory = async () => {
			processing.value++;
			try {
				Object.keys(filters).map(key => filters[key].set.clear());

				const opts = ['--fullnames', '--time-format=iso', '--ip'];

				if (range.value?.[0])
					opts.push('--since', formatDateForLast(range.value[0]));
				if (range.value?.[1])
					opts.push('--until', formatDateForLast(range.value[1]));

				if (props.user !== null)
					opts.push(props.user.user);

				try {
					let tmpHistory = [];
					for (let arg of ['last', 'lastb']) {
						tmpHistory.push(
							...(await useSpawn([arg, ...opts], { superuser: 'try' }).promise()).stdout.split('\n')
								.filter(line => !(/^\s*$/.test(line) || /^[wb]tmp begins/.test(line))) // remove empty lines and last line
								.map(line => {
									const bad = arg === 'lastb';
									const match = line.match(/^(\S+)\s+(\S+(?: \S+)*)\s+(\d{1,3}(?:.\d{1,3}){3})\s+(\S+)(?: - (\S+)\s+\(([^\)]+)\)|\s+(\S+(?: \S+)*))/)?.slice(1);
									if (!match)
										return null;
									try {
										const obj = reactive({
											user: match[0],
											tty: match[1],
											ip: match[2],
											sessionStart: tryDate(match[3]),
											sessionEnd: null, // end time or "still logged in" (or something else?)
											sessionTime: match[5] ? sessionTimeToSentence(match[5]) : "0 Minutes",
											overrideEndText: null,
											authResult: bad ? 'bad' : 'good',
										});
										if (match[6] === "still logged in" || match[6] === "still running") {
											// live update time
											setInterval(() => {
												obj.sessionTime = timeSince(match[3]);
												obj.sessionEnd = new Date();
											}, 60 * 1000);
											obj.sessionTime = timeSince(match[3]);
											obj.overrideEndText = match[6].replace(/\b(\w)/g, w => w.toUpperCase());
										}
										const sessionTimeObj = sessionTimeToObj(match[5]);
										obj.sessionEnd = tryDate(match[4])
											?? ((!match[5])
												? new Date()
												: moment(obj.sessionStart)
													.add(sessionTimeObj.days, "days")
													.add(sessionTimeObj.hours, "hours")
													.add(sessionTimeObj.minutes, "minutes")
													.toDate()
											);
										filters.user.set.add(obj.user);
										filters.ip.set.add(obj.ip);
										filters.tty.set.add(obj.tty);
										filters.authResult.set.add(obj.authResult);
										return obj;
									} catch (error) {
										throw new Error(error.message + `, trigger: ${line}`);
									}
								}).filter(entry => entry !== null)
						)
					}
					history.value = tmpHistory.sort(sortCallback.value);
				} catch (state) {
					notifications.value.constructNotification("Error getting login history", errorStringHTML(state), 'error');
					return;
				}
			} finally {
				processing.value--;
			}
		}

		const rangePreviewFormatter = (previewRange) => {
			let result = "";
			result += previewRange[0].toLocaleDateString([], { dateStyle: "short" });
			if (previewRange[1] !== null) {
				if (isFullDay(...previewRange)) { }
				else if (isSameDay(...previewRange))
					result += `, ${previewRange[0].getHours().toString().padStart(2, '0')}:${previewRange[0].getMinutes().toString().padStart(2, '0')} - ${previewRange[1].getHours()}:${previewRange[1].getMinutes()}`;
				else
					result += ` - ${previewRange[1].toLocaleDateString([], { dateStyle: "short" })}`;
			}
			return result;
		}

		const formatDate = (date) => {
			if (!(date instanceof Date))
				return date;
			const useBrowserLocale = [];
			try {
				return new Intl.DateTimeFormat(useBrowserLocale, { dateStyle: "short", timeStyle: "short" }).format(new Date(date));
			} catch (error) {
				console.error(`Error formatting date: ${JSON.stringify(date)} ` + error.message);
				return date;
			}
		}

		const saveCSV = async () => {
			try {
				const makeCSV = rows => rows.map(row => row.map(field => field.includes(',') ? `"${field}"` : field).join(',')).join('\n');
				const dateFormatter = new Intl.DateTimeFormat([], {
					year: 'numeric',
					month: 'short',
					day: '2-digit',
					hourCycle: 'h24',
					hour: '2-digit',
					minute: '2-digit',
					second: '2-digit',
					timeZoneName: 'longOffset',
				});
				let user = props.user?.user ?? "";
				let hostname = "";
				let fullUser = user;
				try {
					hostname = (await useSpawn(['hostname'], { superuser: 'try' }).promise()).stdout.trim();
					fullUser = (user ? user + '@' + hostname : hostname);
				} catch { }
				if (fullUser)
					fullUser += ' ';
				const dataRange = range.value ? rangePreviewFormatter(range.value) : 'all time';
				const info = [
					['exported from', `${hostname} (${cockpit.transport.origin})`],
					['exported by', ((await cockpit.user()).name)],
					['exported on', dateFormatter.format(new Date())],
					['history for', user ? fullUser : '(all users)' + (hostname ? `@${hostname}` : '')],
					['range start', range.value?.[0] ? dateFormatter.format(range.value[0]) : 'all time'],
					['range end', range.value?.[1] ? dateFormatter.format(range.value[1]) : 'all time'],
				];
				const filterHeader = [
					"Filter",
					"Selected",
				];
				const filters = [
					props.user ? null : ['user', userFilterRef.value.asString()],
					['ip address', ipFilterRef.value.asString()],
					['tty', ttyFilterRef.value.asString()],
					['auth result', authResultFilterRef.value.asString()],
				].filter(f => f !== null);
				const activityHeader = [
					"User",
					"Session Start",
					"Session End",
					"Time Logged In",
					"IP Address",
					"TTY",
					"Authorization Result",
				];
				const activity = history.value
					.map((entry) => [
						entry.user,
						dateFormatter.format(entry.sessionStart),
						entry.overrideEndText ?? dateFormatter.format(entry.sessionEnd),
						entry.sessionTime ?? '',
						entry.ip,
						entry.tty,
						entry.authResult,
					]);
				const data = makeCSV([
					...info,
					[],
					filterHeader,
					...filters,
					[],
					activityHeader,
					...activity,
				]);
				const filename = `${fullUser}login history - ${dataRange}.csv`;
				objectURLDownload(data, filename);
			} catch (error) {
				notifications.value.constructNotification("Failed to download file", errorStringHTML(error), 'error');
			}
		}

		onMounted(() => {
			const to = new Date();
			const from = new Date(new Date().setDate(to.getDate() - props.initialRangeDays));
			to.setHours(23, 59, 59);
			from.setHours(0, 0, 0);
			range.value = [from, to];
			watch(() => props.user, getHistory, { immediate: true, deep: true });
			watch(range, getHistory);
			watch(sortCallback, () => {
				processing.value++;
				try {
					history.value.sort(sortCallback.value);
				} finally {
					processing.value--;
				}
			});
		});

		return {
			range,
			userFilterRef,
			ipFilterRef,
			ttyFilterRef,
			authResultFilterRef,
			history,
			historyReactive,
			processing,
			darkMode,
			filters,
			compareFuncs,
			sortCallback,
			rangePreviewFormatter,
			formatDate,
			saveCSV,
		}
	},
	components: {
		Datepicker,
		FilterIcon,
		SimpleFilter,
		LoadingSpinner,
		SortCallbackButton,
		Table,
		DocumentDownloadIcon,
	}
}
</script>
