<template>
<div class="cbbedffa">
	<canvas ref="chartEl"></canvas>
	<div v-if="fetching" class="fetching">
		<MkLoading/>
	</div>
</div>
</template>

<script lang="ts">
import { defineComponent, onMounted, ref, watch, PropType, onUnmounted, shallowRef } from 'vue';
import {
	Chart,
	ArcElement,
	LineElement,
	BarElement,
	PointElement,
	BarController,
	LineController,
	CategoryScale,
	LinearScale,
	TimeScale,
	Legend,
	Title,
	Tooltip,
	SubTitle,
	Filler,
} from 'chart.js';
import 'chartjs-adapter-date-fns';
import { enUS } from 'date-fns/locale';
import zoomPlugin from 'chartjs-plugin-zoom';
import gradient from 'chartjs-plugin-gradient';
import * as os from '@/os';
import { defaultStore } from '@/store';
import MkChartTooltip from '@/components/chart-tooltip.vue';

Chart.register(
	ArcElement,
	LineElement,
	BarElement,
	PointElement,
	BarController,
	LineController,
	CategoryScale,
	LinearScale,
	TimeScale,
	Legend,
	Title,
	Tooltip,
	SubTitle,
	Filler,
	zoomPlugin,
	gradient,
);

const sum = (...arr) => arr.reduce((r, a) => r.map((b, i) => a[i] + b));
const negate = arr => arr.map(x => -x);
const alpha = (hex, a) => {
	const result = /^#?([a-f\d]{2})([a-f\d]{2})([a-f\d]{2})$/i.exec(hex)!;
	const r = parseInt(result[1], 16);
	const g = parseInt(result[2], 16);
	const b = parseInt(result[3], 16);
	return `rgba(${r}, ${g}, ${b}, ${a})`;
};

const colors = {
	blue: '#008FFB',
	green: '#00E396',
	yellow: '#FEB019',
	red: '#FF4560',
	purple: '#e300db',
	orange: '#fe6919',
	lime: '#c7f400',
};
const colorSets = [colors.blue, colors.green, colors.yellow, colors.red, colors.purple];
const getColor = (i) => {
	return colorSets[i % colorSets.length];
};

export default defineComponent({
	props: {
		src: {
			type: String,
			required: true,
		},
		args: {
			type: Object,
			required: false,
		},
		limit: {
			type: Number,
			required: false,
			default: 90
		},
		span: {
			type: String as PropType<'hour' | 'day'>,
			required: true,
		},
		detailed: {
			type: Boolean,
			required: false,
			default: false
		},
		stacked: {
			type: Boolean,
			required: false,
			default: false
		},
		bar: {
			type: Boolean,
			required: false,
			default: false
		},
		aspectRatio: {
			type: Number,
			required: false,
			default: null
		},
	},

	setup(props) {
		const now = new Date();
		let chartInstance: Chart = null;
		let data: {
			series: {
				name: string;
				type: 'line' | 'area';
				color?: string;
				borderDash?: number[];
				hidden?: boolean;
				data: {
					x: number;
					y: number;
				}[];
			}[];
		} = null;

		const chartEl = ref<HTMLCanvasElement>(null);
		const fetching = ref(true);

		const getDate = (ago: number) => {
			const y = now.getFullYear();
			const m = now.getMonth();
			const d = now.getDate();
			const h = now.getHours();

			return props.span === 'day' ? new Date(y, m, d - ago) : new Date(y, m, d, h - ago);
		};

		const format = (arr) => {
			return arr.map((v, i) => ({
				x: getDate(i).getTime(),
				y: v
			}));
		};

		const tooltipShowing = ref(false);
		const tooltipX = ref(0);
		const tooltipY = ref(0);
		const tooltipTitle = ref(null);
		const tooltipSeries = ref(null);
		let disposeTooltipComponent;

		os.popup(MkChartTooltip, {
			showing: tooltipShowing,
			x: tooltipX,
			y: tooltipY,
			title: tooltipTitle,
			series: tooltipSeries,
		}, {}).then(({ dispose }) => {
			disposeTooltipComponent = dispose;
		});

		function externalTooltipHandler(context) {
			if (context.tooltip.opacity === 0) {
				tooltipShowing.value = false;
				return;
			}

			tooltipTitle.value = context.tooltip.title[0];
			tooltipSeries.value = context.tooltip.body.map((b, i) => ({
				backgroundColor: context.tooltip.labelColors[i].backgroundColor,
				borderColor: context.tooltip.labelColors[i].borderColor,
				text: b.lines[0],
			}));

			const rect = context.chart.canvas.getBoundingClientRect();

			tooltipShowing.value = true;
			tooltipX.value = rect.left + window.pageXOffset + context.tooltip.caretX;
			tooltipY.value = rect.top + window.pageYOffset + context.tooltip.caretY;
		}

		const render = () => {
			if (chartInstance) {
				chartInstance.destroy();
			}

			const gridColor = defaultStore.state.darkMode ? 'rgba(255, 255, 255, 0.1)' : 'rgba(0, 0, 0, 0.1)';
			const vLineColor = defaultStore.state.darkMode ? 'rgba(255, 255, 255, 0.2)' : 'rgba(0, 0, 0, 0.2)';

			// フォントカラー
			Chart.defaults.color = getComputedStyle(document.documentElement).getPropertyValue('--fg');

			const maxes = data.series.map((x, i) => Math.max(...x.data.map(d => d.y)));

			chartInstance = new Chart(chartEl.value, {
				type: props.bar ? 'bar' : 'line',
				data: {
					labels: new Array(props.limit).fill(0).map((_, i) => getDate(i).toLocaleString()).slice().reverse(),
					datasets: data.series.map((x, i) => ({
						parsing: false,
						label: x.name,
						data: x.data.slice().reverse(),
						tension: 0.3,
						pointRadius: 0,
						borderWidth: props.bar ? 0 : 2,
						borderColor: x.color ? x.color : getColor(i),
						borderDash: x.borderDash || [],
						borderJoinStyle: 'round',
						borderRadius: props.bar ? 3 : undefined,
						backgroundColor: props.bar ? (x.color ? x.color : getColor(i)) : alpha(x.color ? x.color : getColor(i), 0.1),
						gradient: props.bar ? undefined : {
							backgroundColor: {
								axis: 'y',
								colors: {
									0: alpha(x.color ? x.color : getColor(i), 0),
									[maxes[i]]: alpha(x.color ? x.color : getColor(i), 0.175),
								},
							},
						},
						barPercentage: 0.9,
						categoryPercentage: 0.9,
						fill: x.type === 'area',
						clip: 8,
						hidden: !!x.hidden,
					})),
				},
				options: {
					aspectRatio: props.aspectRatio || 2.5,
					layout: {
						padding: {
							left: 0,
							right: 8,
							top: 0,
							bottom: 0,
						},
					},
					scales: {
						x: {
							type: 'time',
							stacked: props.stacked,
							offset: false,
							time: {
								stepSize: 1,
								unit: props.span === 'day' ? 'month' : 'day',
							},
							grid: {
								color: gridColor,
								borderColor: 'rgb(0, 0, 0, 0)',
							},
							ticks: {
								display: props.detailed,
								maxRotation: 0,
								autoSkipPadding: 16,
							},
							adapters: {
								date: {
									locale: enUS,
								},
							},
							min: getDate(props.limit).getTime(),
						},
						y: {
							position: 'left',
							stacked: props.stacked,
							suggestedMax: 50,
							grid: {
								color: gridColor,
								borderColor: 'rgb(0, 0, 0, 0)',
							},
							ticks: {
								display: props.detailed,
								//mirror: true,
							},
						},
					},
					interaction: {
						intersect: false,
						mode: 'index',
					},
					elements: {
						point: {
							hoverRadius: 5,
							hoverBorderWidth: 2,
						},
					},
					animation: false,
					plugins: {
						legend: {
							display: props.detailed,
							position: 'bottom',
							labels: {
								boxWidth: 16,
							},
						},
						tooltip: {
							enabled: false,
							mode: 'index',
							animation: {
								duration: 0,
							},
							external: externalTooltipHandler,
						},
						zoom: props.detailed ? {
							pan: {
								enabled: true,
							},
							zoom: {
								wheel: {
									enabled: true,
								},
								pinch: {
									enabled: true,
								},
								drag: {
									enabled: false,
								},
								mode: 'x',
							},
							limits: {
								x: {
									min: 'original',
									max: 'original',
								},
								y: {
									min: 'original',
									max: 'original',
								},
							}
						} : undefined,
						gradient,
					},
				},
				plugins: [{
					id: 'vLine',
					beforeDraw(chart, args, options) {
						if (chart.tooltip._active && chart.tooltip._active.length) {
							const activePoint = chart.tooltip._active[0];
							const ctx = chart.ctx;
							const x = activePoint.element.x;
							const topY = chart.scales.y.top;
							const bottomY = chart.scales.y.bottom;

							ctx.save();
							ctx.beginPath();
							ctx.moveTo(x, bottomY);
							ctx.lineTo(x, topY);
							ctx.lineWidth = 1;
							ctx.strokeStyle = vLineColor;
							ctx.stroke();
							ctx.restore();
						}
					}
				}]
			});
		};

		const exportData = () => {
			// TODO
		};

		const fetchFederationChart = async (): Promise<typeof data> => {
			const raw = await os.api('charts/federation', { limit: props.limit, span: props.span });
			return {
				series: [{
					name: 'Received',
					type: 'area',
					data: format(raw.inboxInstances),
					color: colors.blue,
				}, {
					name: 'Delivered',
					type: 'area',
					data: format(raw.deliveredInstances),
					color: colors.green,
				}, {
					name: 'Stalled',
					type: 'area',
					data: format(raw.stalled),
					color: colors.red,
				}, {
					name: 'Pub & Sub',
					type: 'line',
					data: format(raw.pubsub),
					color: colors.lime,
				}, {
					name: 'Pub',
					type: 'line',
					data: format(raw.pub),
					color: colors.purple,
				}, {
					name: 'Sub',
					type: 'line',
					data: format(raw.sub),
					color: colors.orange,
				}],
			};
		};

		const fetchApRequestChart = async (): Promise<typeof data> => {
			const raw = await os.api('charts/ap-request', { limit: props.limit, span: props.span });
			return {
				series: [{
					name: 'In',
					type: 'area',
					color: '#008FFB',
					data: format(raw.inboxReceived)
				}, {
					name: 'Out (succ)',
					type: 'area',
					color: '#00E396',
					data: format(raw.deliverSucceeded)
				}, {
					name: 'Out (fail)',
					type: 'area',
					color: '#FEB019',
					data: format(raw.deliverFailed)
				}]
			};
		};

		const fetchNotesChart = async (type: string): Promise<typeof data> => {
			const raw = await os.api('charts/notes', { limit: props.limit, span: props.span });
			return {
				series: [{
					name: 'All',
					type: 'line',
					data: format(type == 'combined'
						? sum(raw.local.inc, negate(raw.local.dec), raw.remote.inc, negate(raw.remote.dec))
						: sum(raw[type].inc, negate(raw[type].dec))
					),
					color: '#888888',
				}, {
					name: 'Renotes',
					type: 'area',
					data: format(type == 'combined'
						? sum(raw.local.diffs.renote, raw.remote.diffs.renote)
						: raw[type].diffs.renote
					),
					color: colors.green,
				}, {
					name: 'Replies',
					type: 'area',
					data: format(type == 'combined'
						? sum(raw.local.diffs.reply, raw.remote.diffs.reply)
						: raw[type].diffs.reply
					),
					color: colors.yellow,
				}, {
					name: 'Normal',
					type: 'area',
					data: format(type == 'combined'
						? sum(raw.local.diffs.normal, raw.remote.diffs.normal)
						: raw[type].diffs.normal
					),
					color: colors.blue,
				}, {
					name: 'With file',
					type: 'area',
					data: format(type == 'combined'
						? sum(raw.local.diffs.withFile, raw.remote.diffs.withFile)
						: raw[type].diffs.withFile
					),
					color: colors.purple,
				}],
			};
		};

		const fetchNotesTotalChart = async (): Promise<typeof data> => {
			const raw = await os.api('charts/notes', { limit: props.limit, span: props.span });
			return {
				series: [{
					name: 'Combined',
					type: 'line',
					data: format(sum(raw.local.total, raw.remote.total)),
				}, {
					name: 'Local',
					type: 'area',
					data: format(raw.local.total),
				}, {
					name: 'Remote',
					type: 'area',
					data: format(raw.remote.total),
				}],
			};
		};

		const fetchUsersChart = async (total: boolean): Promise<typeof data> => {
			const raw = await os.api('charts/users', { limit: props.limit, span: props.span });
			return {
				series: [{
					name: 'Combined',
					type: 'line',
					data: format(total
						? sum(raw.local.total, raw.remote.total)
						: sum(raw.local.inc, negate(raw.local.dec), raw.remote.inc, negate(raw.remote.dec))
					),
				}, {
					name: 'Local',
					type: 'area',
					data: format(total
						? raw.local.total
						: sum(raw.local.inc, negate(raw.local.dec))
					),
				}, {
					name: 'Remote',
					type: 'area',
					data: format(total
						? raw.remote.total
						: sum(raw.remote.inc, negate(raw.remote.dec))
					),
				}],
			};
		};

		const fetchActiveUsersChart = async (): Promise<typeof data> => {
			const raw = await os.api('charts/active-users', { limit: props.limit, span: props.span });
			return {
				series: [{
					name: 'Read & Write',
					type: 'area',
					data: format(raw.readWrite),
					color: colors.orange,
				}, {
					name: 'Write',
					type: 'area',
					data: format(raw.write),
					color: colors.lime,
				}, {
					name: 'Read',
					type: 'area',
					data: format(raw.read),
					color: colors.blue,
				}, {
					name: '< Week',
					type: 'area',
					data: format(raw.registeredWithinWeek),
					color: colors.green,
				}, {
					name: '< Month',
					type: 'area',
					data: format(raw.registeredWithinMonth),
					color: colors.yellow,
				}, {
					name: '< Year',
					type: 'area',
					data: format(raw.registeredWithinYear),
					color: colors.red,
				}, {
					name: '> Week',
					type: 'area',
					data: format(raw.registeredOutsideWeek),
					color: colors.yellow,
				}, {
					name: '> Month',
					type: 'area',
					data: format(raw.registeredOutsideMonth),
					color: colors.red,
				}, {
					name: '> Year',
					type: 'area',
					data: format(raw.registeredOutsideYear),
					color: colors.purple,
				}],
			};
		};

		const fetchDriveChart = async (): Promise<typeof data> => {
			const raw = await os.api('charts/drive', { limit: props.limit, span: props.span });
			return {
				bytes: true,
				series: [{
					name: 'All',
					type: 'line',
					borderDash: [5, 5],
					data: format(
						sum(
							raw.local.incSize,
							negate(raw.local.decSize),
							raw.remote.incSize,
							negate(raw.remote.decSize)
						)
					),
				}, {
					name: 'Local +',
					type: 'area',
					data: format(raw.local.incSize),
				}, {
					name: 'Local -',
					type: 'area',
					data: format(negate(raw.local.decSize)),
				}, {
					name: 'Remote +',
					type: 'area',
					data: format(raw.remote.incSize),
				}, {
					name: 'Remote -',
					type: 'area',
					data: format(negate(raw.remote.decSize)),
				}],
			};
		};

		const fetchDriveFilesChart = async (): Promise<typeof data> => {
			const raw = await os.api('charts/drive', { limit: props.limit, span: props.span });
			return {
				series: [{
					name: 'All',
					type: 'line',
					borderDash: [5, 5],
					data: format(
						sum(
							raw.local.incCount,
							negate(raw.local.decCount),
							raw.remote.incCount,
							negate(raw.remote.decCount)
						)
					),
				}, {
					name: 'Local +',
					type: 'area',
					data: format(raw.local.incCount),
				}, {
					name: 'Local -',
					type: 'area',
					data: format(negate(raw.local.decCount)),
				}, {
					name: 'Remote +',
					type: 'area',
					data: format(raw.remote.incCount),
				}, {
					name: 'Remote -',
					type: 'area',
					data: format(negate(raw.remote.decCount)),
				}],
			};
		};

		const fetchInstanceRequestsChart = async (): Promise<typeof data> => {
			const raw = await os.api('charts/instance', { host: props.args.host, limit: props.limit, span: props.span });
			return {
				series: [{
					name: 'In',
					type: 'area',
					color: '#008FFB',
					data: format(raw.requests.received)
				}, {
					name: 'Out (succ)',
					type: 'area',
					color: '#00E396',
					data: format(raw.requests.succeeded)
				}, {
					name: 'Out (fail)',
					type: 'area',
					color: '#FEB019',
					data: format(raw.requests.failed)
				}]
			};
		};

		const fetchInstanceUsersChart = async (total: boolean): Promise<typeof data> => {
			const raw = await os.api('charts/instance', { host: props.args.host, limit: props.limit, span: props.span });
			return {
				series: [{
					name: 'Users',
					type: 'area',
					color: '#008FFB',
					data: format(total
						? raw.users.total
						: sum(raw.users.inc, negate(raw.users.dec))
					)
				}]
			};
		};

		const fetchInstanceNotesChart = async (total: boolean): Promise<typeof data> => {
			const raw = await os.api('charts/instance', { host: props.args.host, limit: props.limit, span: props.span });
			return {
				series: [{
					name: 'Notes',
					type: 'area',
					color: '#008FFB',
					data: format(total
						? raw.notes.total
						: sum(raw.notes.inc, negate(raw.notes.dec))
					)
				}]
			};
		};

		const fetchInstanceFfChart = async (total: boolean): Promise<typeof data> => {
			const raw = await os.api('charts/instance', { host: props.args.host, limit: props.limit, span: props.span });
			return {
				series: [{
					name: 'Following',
					type: 'area',
					color: '#008FFB',
					data: format(total
						? raw.following.total
						: sum(raw.following.inc, negate(raw.following.dec))
					)
				}, {
					name: 'Followers',
					type: 'area',
					color: '#00E396',
					data: format(total
						? raw.followers.total
						: sum(raw.followers.inc, negate(raw.followers.dec))
					)
				}]
			};
		};

		const fetchInstanceDriveUsageChart = async (total: boolean): Promise<typeof data> => {
			const raw = await os.api('charts/instance', { host: props.args.host, limit: props.limit, span: props.span });
			return {
				bytes: true,
				series: [{
					name: 'Drive usage',
					type: 'area',
					color: '#008FFB',
					data: format(total
						? raw.drive.totalUsage
						: sum(raw.drive.incUsage, negate(raw.drive.decUsage))
					)
				}]
			};
		};

		const fetchInstanceDriveFilesChart = async (total: boolean): Promise<typeof data> => {
			const raw = await os.api('charts/instance', { host: props.args.host, limit: props.limit, span: props.span });
			return {
				series: [{
					name: 'Drive files',
					type: 'area',
					color: '#008FFB',
					data: format(total
						? raw.drive.totalFiles
						: sum(raw.drive.incFiles, negate(raw.drive.decFiles))
					)
				}]
			};
		};

		const fetchPerUserNotesChart = async (): Promise<typeof data> => {
			const raw = await os.api('charts/user/notes', { userId: props.args.user.id, limit: props.limit, span: props.span });
			return {
				series: [...(props.args.withoutAll ? [] : [{
					name: 'All',
					type: 'line',
					data: format(sum(raw.inc, negate(raw.dec))),
					color: '#888888',
				}]), {
					name: 'With file',
					type: 'area',
					data: format(raw.diffs.withFile),
					color: colors.purple,
				}, {
					name: 'Renotes',
					type: 'area',
					data: format(raw.diffs.renote),
					color: colors.green,
				}, {
					name: 'Replies',
					type: 'area',
					data: format(raw.diffs.reply),
					color: colors.yellow,
				}, {
					name: 'Normal',
					type: 'area',
					data: format(raw.diffs.normal),
					color: colors.blue,
				}],
			};
		};

		const fetchPerUserDriveChart = async (): Promise<typeof data> => {
			const raw = await os.api('charts/user/drive', { userId: props.args.user.id, limit: props.limit, span: props.span });
			return {
				series: [{
					name: 'Inc',
					type: 'area',
					data: format(raw.incSize),
				}, {
					name: 'Dec',
					type: 'area',
					data: format(raw.decSize),
				}],
			};
		};

		const fetchAndRender = async () => {
			const fetchData = () => {
				switch (props.src) {
					case 'federation': return fetchFederationChart();
					case 'ap-request': return fetchApRequestChart();
					case 'users': return fetchUsersChart(false);
					case 'users-total': return fetchUsersChart(true);
					case 'active-users': return fetchActiveUsersChart();
					case 'notes': return fetchNotesChart('combined');
					case 'local-notes': return fetchNotesChart('local');
					case 'remote-notes': return fetchNotesChart('remote');
					case 'notes-total': return fetchNotesTotalChart();
					case 'drive': return fetchDriveChart();
					case 'drive-files': return fetchDriveFilesChart();
					
					case 'instance-requests': return fetchInstanceRequestsChart();
					case 'instance-users': return fetchInstanceUsersChart(false);
					case 'instance-users-total': return fetchInstanceUsersChart(true);
					case 'instance-notes': return fetchInstanceNotesChart(false);
					case 'instance-notes-total': return fetchInstanceNotesChart(true);
					case 'instance-ff': return fetchInstanceFfChart(false);
					case 'instance-ff-total': return fetchInstanceFfChart(true);
					case 'instance-drive-usage': return fetchInstanceDriveUsageChart(false);
					case 'instance-drive-usage-total': return fetchInstanceDriveUsageChart(true);
					case 'instance-drive-files': return fetchInstanceDriveFilesChart(false);
					case 'instance-drive-files-total': return fetchInstanceDriveFilesChart(true);

					case 'per-user-notes': return fetchPerUserNotesChart();
					case 'per-user-drive': return fetchPerUserDriveChart();
				}
			};
			fetching.value = true;
			data = await fetchData();
			fetching.value = false;
			render();
		};

		watch(() => [props.src, props.span], fetchAndRender);

		onMounted(() => {
			fetchAndRender();
		});

		onUnmounted(() => {
			if (disposeTooltipComponent) disposeTooltipComponent();
		});

		return {
			chartEl,
			fetching,
		};
	},
});
</script>

<style lang="scss" scoped>
.cbbedffa {
	position: relative;

	> .fetching {
		position: absolute;
		top: 0;
		left: 0;
		width: 100%;
		height: 100%;
		-webkit-backdrop-filter: var(--blur, blur(12px));
		backdrop-filter: var(--blur, blur(12px));
		display: flex;
		justify-content: center;
		align-items: center;
		cursor: wait;
	}
}
</style>
