<template>
	<div v-show="visible" class="results-container" :disabled="request">
		<span v-if="request" class="fa fa-spinner fa-spin searchIndicator" style="position:absolute; left: 50%; top:15px"></span>

		<div class="crumbs-totals">
			<ol class="breadcrumb resultscrumb">
				<!-- no disabled state; use active class instead... -->
				<li v-for="(crumb, index) in breadCrumbs" :key="index" :class="{'active': crumb.active || !!request}">
					<a v-if="!crumb.active && !request"
						role="button"
						:title="crumb.title"
						:disabled="request"
						:class="request ? 'disabled' : undefined"
						@click.prevent="!request && crumb.onClick ? crumb.onClick() : undefined"
					>{{crumb.label}}</a>
					<template v-else>{{crumb.label}}</template>
				</li>
			</ol>

			<Totals v-if="results"
				class="result-totals"
				:initialResults="results"
				:type="type"
				:indexId="indexId"

				@update="paginationResults = $event"
			/>
		</div>

		<template v-if="resultsHaveData">
			<component
				:is="resultComponentName"
				v-bind="resultComponentData"

				@sort="sort = $event"
				@viewgroup="originalGroupBySettings = {page, sort}; log(page, sort); viewGroup = $event.id; _viewGroupName = $event.displayName;"
			>
				<GroupBy slot="groupBy" :type="type"
					:disabled="!!request"
					:originalGroupBySettings="originalGroupBySettings"
					@viewgroupLeave="leaveViewgroup"
				/>

				<Pagination slot="pagination"
					style="display: block; margin: 10px 0;"

					:page="pagination.shownPage"
					:maxPage="pagination.maxShownPage"
					:disabled="!!request"

					@change="page = $event"
				/>
			</component>
			<hr>
		</template>
		<div v-else-if="results" class="no-results-found">No results found.</div>
		<div v-else-if="!valid" class="no-results-found">
			This view is inactive because no search criteria for words were specified.
		</div>
		<div v-else-if="error != null" class="no-results-found">
			<span class="fa fa-exclamation-triangle text-danger"></span><br>
			<span v-html="error"></span>
			<br>
			<br>
			<button type="button" class="btn btn-default" title="Try again with current search settings" @click="markDirty();">Try again</button>
		</div>

		<div v-show="resultsHaveData" class="text-right">
			<SelectPicker
				data-class="btn-sm btn-default"
				placeholder="Sort by..."
				data-menu-width="grow"

				allowHtml
				hideDisabled
				allowUnknownValues
				right

				:searchable="sortOptions.flatMap(o => o.options && !o.disabled ? o.options.filter(opt => !opt.disabled) : o).length > 12"
				:options="sortOptions"
				:disabled="!!request"

				v-model="sort"
			/>

			<button v-if="isDocs && resultsHaveHits"
				type="button"
				class="btn btn-primary btn-sm"

				@click="showDocumentHits = !showDocumentHits"
			>
				{{showDocumentHits ? 'Hide Hits' : 'Show Hits'}}
			</button>
			<button v-if="isHits"
				type="button"
				class="btn btn-primary btn-sm"

				@click="showTitles = !showTitles"
			>
				{{showTitles ? 'Hide' : 'Show'}} Titles
			</button>

			<div class="btn-group" v-if="results && exportEnabled">
				<button
					type="button"
					class="btn btn-default btn-sm"
					:disabled="downloadInProgress || !resultsHaveData || !!request"
					:title="downloadInProgress ? 'Downloading...' : 'Export results as a CSV file'"

					@click="downloadCsv(false)"
				>
					<template v-if="downloadInProgress">&nbsp;<span class="fa fa-spinner fa-spin"></span>&nbsp;</template>Export
				</button>
				<button
					type="button"
					class="btn btn-default btn-sm"
					:disabled="downloadInProgress || !resultsHaveData || !!request"
					:title="downloadInProgress ? 'Downloading...' : 'Export Results as a CSV file for use with Excel'"

					@click="downloadCsv(true)"
				>
					<template v-if="downloadInProgress">&nbsp;<span class="fa fa-spinner fa-spin"></span>&nbsp;</template>Export for Excel
				</button>
				<!-- <button type="button"  class="btn btn-default btn-sm dropdown-toggle" data-toggle="dropdown" aria-expanded="false">
					<span class="caret"></span>
				</button> -->
				<!-- <ul class="dropdown-menu dropdown-menu-right" @click.stop>
					<li><a class="checkbox" title="Adds a header describing the query used to generate these results.">
						<label><input type="checkbox" v-model="exportSummary">Include summary</label></a>
					</li>
					<li><a class="checkbox"
						title="Adds a header line declaring that the file is comma-separated,
						for some versions of microsoft excel this is required to correctly display the file."
					><label><input type="checkbox" v-model="exportSeparator">Export for excel</label></a></li>
					<li v-if="isHits"><a class="checkbox"
						title="Also export document metadata. Warning: this might result in very large exports!"
					><label><input type="checkbox" v-model="exportHitMetadata">Export metadata</label></a></li>
				</ul> -->
			</div>
		</div>
		<hr>
		<Debug>
			<div>
				<div>BlackLab response: </div>
				<pre v-if="results">{{JSON.stringify(results.summary, undefined, 2)}}</pre>
			</div>
		</Debug>
	</div>
</template>

<script lang="ts">
import Vue from 'vue';
import {saveAs} from 'file-saver';

import jsonStableStringify from 'json-stable-stringify';

import * as Api from '@/api';

import * as RootStore from '@/store/search/';
import * as CorpusStore from '@/store/search/corpus';
import * as ResultsStore from '@/store/search/results';
import * as GlobalStore from '@/store/search/results/global';
import * as QueryStore from '@/store/search/query';
import * as InterfaceStore from '@/store/search/form/interface';
import * as UIStore from '@/store/search/ui';
import * as GlossModule from '@/pages/search/form/concept/glossStore' // Jesse

import GroupResults from '@/pages/search/results/table/GroupResults.vue';
import HitResults from '@/pages/search/results/table/HitResults.vue';
import DocResults from '@/pages/search/results/table/DocResults.vue';
import Totals from '@/pages/search/results/ResultTotals.vue';
import GroupBy from '@/pages/search/results/groupby/GroupBy.vue';

import Pagination from '@/components/Pagination.vue';
import SelectPicker, {OptGroup} from '@/components/SelectPicker.vue';

import debug, { debugLog } from '@/utils/debug';

import * as BLTypes from '@/types/blacklabtypes';
import cloneDeep from 'clone-deep';
import { getAnnotationSubset, getMetadataSubset } from '@/utils';

export default Vue.extend({
	components: {
		Pagination,
		GroupResults,
		HitResults,
		DocResults,
		Totals,
		GroupBy,
		SelectPicker
	},
	props: {
		type: {
			type: String as () => ResultsStore.ViewId,
			required: true,
		}
	},
	data: () => ({
		isDirty: true, // since we don't have any results yet
		request: null as null|Promise<BLTypes.BLSearchResult>,
		results: null as null|BLTypes.BLSearchResult,
		error: null as null|string,
		cancel: null as null|Api.Canceler,

		_viewGroupName: null as string|null,
		showTitles: true,
		showDocumentHits: false,

		downloadInProgress: false, // csv download
		// exportSummary: false,
		// exportSeparator: false,
		// exportHitMetadata: false,

		paginationResults: null as null|BLTypes.BLSearchResult,

		// Should we scroll when next results arrive - set when main form submitted
		scroll: true,
		// Should we clear the results when we begin the next request? - set when main for submitted.
		clearResults: false,

		originalGroupBySettings: null as null|{
			page: number;
			sort: string;
		},

		debug
	}),
	methods: {
		log: console.log,
		markDirty() {
			this.isDirty = true;
			if (this.cancel) {
				debugLog('cancelling search request');
				this.cancel();
				this.cancel = null;
				this.request = null;
			}
			if (this.visible) {
				this.refresh();
			}
		},
		refresh() {
			this.isDirty = false;
			debugLog('this is when the search should be refreshed');

			if (this.cancel) {
				debugLog('cancelling previous search request');
				this.cancel();
				this.request = null;
				this.cancel = null;
			}

			if (!this.valid) {
				this.results = null;
				this.paginationResults = null;
				this.error = null;
				this.clearResults = false;
				return;
			}

			if (this.clearResults) { this.results = this.error = null; this.clearResults = false; }

			const params = RootStore.get.blacklabParameters()!;
			const apiCall = this.type === 'hits' ? Api.blacklab.getHits : Api.blacklab.getDocs;
			debugLog('starting search', this.type, params);

			const r = apiCall(this.indexId, params, {headers: { 'Cache-Control': 'no-cache' }});
			this.request = r.request;
			this.cancel = r.cancel;

			setTimeout(() => this.scrollToResults(), 1500);

			this.request
			.then(this.setSuccess, e => this.setError(e, !!params.group))
			.finally(() => this.scrollToResults())
		},
		setSuccess(data: BLTypes.BLSearchResult) {
			debugLog('search results', data);
			this.results = data;
			this.paginationResults = data;
			this.error = null;
			this.request = null;
			this.cancel = null;
            
			// Jesse (glosses): hier ook een keer de page hits in de gloss store updaten
    
            const currentHitIds = this.results.hits.map(h => GlossModule.get.settings().get_hit_id(h))
    
            GlossModule.actions.setCurrentPage(currentHitIds)
		},
		setError(data: Api.ApiError, isGrouped?: boolean) {
			if (data.title !== 'Request cancelled') { // TODO
				debugLog('Request failed: ', data);
				this.error = UIStore.getState().global.errorMessage(data, isGrouped ? 'groups' : this.type);
				this.results = null;
				this.paginationResults = null;
			}
			this.request = null;
			this.cancel= null;
		},
		downloadCsv(excel: boolean) {
			if (this.downloadInProgress || !this.results) {
				return;
			}

			this.downloadInProgress = true;
			const apiCall = this.type === 'hits' ? Api.blacklab.getHitsCsv : Api.blacklab.getDocsCsv;
			const params = cloneDeep(this.results.summary.searchParam);
			if (UIStore.getState().results.shared.detailedAnnotationIds) {
				params.listvalues = UIStore.getState().results.shared.detailedAnnotationIds!.join(',');
			}
			if (UIStore.getState().results.shared.detailedMetadataIds) {
				params.listmetadatavalues = UIStore.getState().results.shared.detailedMetadataIds!.join(',');
			}
			(params as any).csvsepline = !!excel;
			(params as any).csvsummary = true;

			debugLog('starting csv download', this.type, params);
			apiCall(this.indexId, params).request
			.then(
				blob => saveAs(blob, 'data.csv'),
				error => debugLog('Error downloading csv file', error)
			)
			.finally(() => {
				this.downloadInProgress = false;
			});
		},
		scrollToResults() {
			if (this.scroll) {
				this.scroll = false;
				window.scroll({
					behavior: 'smooth',
					top: (this.$el as HTMLElement).offsetTop - 150
				});
			}
		},
		leaveViewgroup() {
			this.viewGroup = null;
			this.page = this.originalGroupBySettings?.page || 0;
			this.sort = this.originalGroupBySettings?.sort || null;
			this.originalGroupBySettings = null;
		}
	},
	computed: {
		// Store properties
		storeModule(): ReturnType<(typeof ResultsStore)['get']['resultsModules']>[number] { return ResultsStore.get.resultsModules().find(m => m.namespace === this.type)!; },
		groupBy: {
			get(): string[] { return this.storeModule.getState().groupBy; },
			set(v: string[]) { this.storeModule.actions.groupBy(v); }
		},
		groupByAdvanced: {
			get(): string[] { return this.storeModule.getState().groupByAdvanced; },
			set(v: string[]) { this.storeModule.actions.groupByAdvanced(v); }
		},
		page: {
			get(): number { const n = this.storeModule.getState().page; console.log('got page', n); return n; },
			set(v: number) { this.storeModule.actions.page(v); console.log('set page', v); }
		},
		sort: {
			get(): string|null { return this.storeModule.getState().sort; },
			set(v: string|null) { this.storeModule.actions.sort(v); }
		},
		viewGroup: {
			get(): string|null { return this.storeModule.getState().viewGroup; },
			set(v: string|null) { this.storeModule.actions.viewGroup(v); }
		},

		exportEnabled(): boolean { return UIStore.getState().results.shared.exportEnabled; },

		refreshParameters(): string {
			/*
				NOTE: we return this as a string so we can remove properties
				If we don't the watcher on this computed will fire regardless
				because some property somewhere in the object is a new instance and thus not equal...
				This would cause new results to be requested even when just changing the table display mode...
			*/
			return jsonStableStringify({
				global: GlobalStore.getState(),
				self: {
					...this.storeModule.getState(),
					groupDisplayMode: null // ignore this property
				} as Partial<ResultsStore.PartialRootState[ResultsStore.ViewId]>,
				query: QueryStore.getState()
			});
		},

		// When these change, the form has been resubmitted, so we need to initiate a scroll event
		querySettings() { return QueryStore.getState(); },

		pagination(): {
			shownPage: number,
			maxShownPage: number
		} {
			const r: BLTypes.BLSearchResult|null = this.paginationResults || this.results;
			if (r == null) {
				return {
					shownPage: 0,
					maxShownPage: 0,
				};
			}

			const pageSize = this.results!.summary.requestedWindowSize;
			const shownPage = Math.floor(this.results!.summary.windowFirstResult / pageSize);
			const totalResults =
				BLTypes.isGroups(r) ? r.summary.numberOfGroups :
				BLTypes.isHitResults(r) ? r.summary.numberOfHitsRetrieved :
				r.summary.numberOfDocsRetrieved;

			// subtract one page if number of results exactly divisible by page size
			// e.g. 20 results for a page size of 20 is still only one page instead of 2.
			const pageCount = Math.floor(totalResults / pageSize) - ((totalResults % pageSize === 0 && totalResults > 0) ? 1 : 0);

			return {
				shownPage,
				maxShownPage: pageCount
			};
		},

		visible(): boolean { return InterfaceStore.get.viewedResults() === this.type; },
		valid(): boolean {
			if (this.type === 'hits') {
				const params = RootStore.get.blacklabParameters();
				return !!(params && params.patt);
			} else {
				return true;
			}
		},
		// simple view variables
		indexId(): string { return CorpusStore.getState().id; },
		resultsHaveData(): boolean {
			if (BLTypes.isDocGroups(this.results)) { return this.results.docGroups.length > 0; }
			if (BLTypes.isHitGroups(this.results)) { return this.results.hitGroups.length > 0; }
			if (BLTypes.isHitResults(this.results)) { return this.results.hits.length > 0; }
			if (BLTypes.isDocResults(this.results)) { return this.results.docs.length > 0; }
			return false;
		},
		isHits(): boolean { return BLTypes.isHitResults(this.results); },
		isDocs(): boolean { return BLTypes.isDocResults(this.results); },
		isGroups(): boolean { return BLTypes.isGroups(this.results); },
		resultsHaveHits(): boolean { return this.results != null && !!this.results.summary.searchParam.patt; },
		viewGroupName(): string {
			if (this.viewGroup == null) { return ''; }
			return this._viewGroupName ? this._viewGroupName :
				this.viewGroup.substring(this.viewGroup.indexOf(':')+1) || '[unknown]';
		},

		breadCrumbs(): any {
			const r = [];
			r.push({
				label: this.type === 'hits' ? 'Hits' : 'Documents',
				title: 'Go back to ungrouped results',
				active: false, //(this.groupBy.length + this.groupByAdvanced.length) === 0,
				onClick: () => {
					this.groupBy = [];
					this.groupByAdvanced = [];
					GlobalStore.actions.sampleSize(null);
				}
			});
			if ((this.groupBy.length + this.groupByAdvanced.length) > 0) {
				r.push({
					label: 'Grouped by ' + this.groupBy.concat(this.groupByAdvanced).toString(),
					title: 'Go back to grouped results',
					active: false, //this.viewGroup == null,
					onClick: () => {
						this.leaveViewgroup();
						GlobalStore.actions.sampleSize(null);
					}
				});
			}
			if (this.viewGroup != null) {
				r.push({
					label: 'Viewing group ' + this.viewGroupName,
					title: '',
					active: false,
					onClick: () => GlobalStore.actions.sampleSize(null)
				});
			}
			const {sampleMode, sampleSize} = GlobalStore.getState();
			if (sampleSize != null) {
				r.push({
					label: `Random sample (${sampleSize}${sampleMode === 'percentage' ? '%' : ''})`,
					title: `Showing only some (${sampleSize}${sampleMode === 'percentage' ? '%' : ''}) results`,
					active: false,
					onClick: () => {
						$('#settings').modal('show')
					}
				})
			}
			r[r.length -1].active = true;
			return r;
		},
		sortOptions(): OptGroup[] {
			// NOTE: we need to always pass all available options, then hide invalids based on displayed results
			// if we don't do this, sorting will be cleared on initial page load
			// This happens because results aren't loaded yet, thus isHits/isDocs/isGroups all return false, and no options would be available
			// then the selectpicker will reset value to undefined, which clears it in the store, which updates the url, etc.
			const opts = [] as OptGroup[];

			if (this.isGroups) {
				opts.push({
					label: 'Groups',
					options: [{
						label: 'Sort by Group Name',
						value: 'identity',
					}, {
						label: 'Sort by Group Name (descending)',
						value: '-identity',
					}, {
						label: 'Sort by Size',
						value: 'size',
					}, {
						label: 'Sort by Size (ascending)',
						value: '-size', // numeric sorting is inverted: https://github.com/INL/corpus-frontend/issues/340
					}]
				});
			}

			if (this.isHits) {
				opts.push(...getAnnotationSubset(
					UIStore.getState().results.shared.sortAnnotationIds,
					CorpusStore.get.annotationGroups(),
					CorpusStore.get.allAnnotationsMap(),
					'Sort',
					CorpusStore.get.textDirection(),
					this.debug.debug
				));
			}
			if (this.isDocs) {
				opts.push({
					label: 'Documents',
					options: [{
						label: 'Sort by hits',
						value: 'numhits'
					}, {
						label: 'Sort by hits (ascending)',
						value: '-numhits' // numeric sorting is inverted: https://github.com/INL/corpus-frontend/issues/340
					}]
				});
			}

			if (!this.isGroups) {
				opts.push(...getMetadataSubset(
					UIStore.getState().results.shared.sortMetadataIds,
					CorpusStore.get.metadataGroups(),
					CorpusStore.get.allMetadataFieldsMap(),
					'Sort',
					this.debug.debug
				));
			}

			return opts;
		},

		resultComponentName(): string {
			if (this.isGroups) {
				return 'GroupResults';
			} else if (this.isHits) {
				return 'HitResults';
			} else {
				return 'DocResults';
			}
		},
		resultComponentData(): any {
			switch (this.resultComponentName) {
				case 'GroupResults': return {
					results: this.results,
					disabled: !!this.request,
					sort: this.sort,
				};
				case 'HitResults': return {
					results: this.results,
					disabled: !!this.request,
					sort: this.sort,
					showTitles: this.showTitles,
				};
				case 'DocResults': return {
					results: this.results,
					disabled: !!this.request,
					sort: this.sort,
					showDocumentHits: this.showDocumentHits
				};
			}
		},
	},
	watch: {
		querySettings: {
			deep: true,
			handler() {
				this.scroll = true;
				this.clearResults = true;
			},
		},
		refreshParameters: {
			handler(cur, prev) {
				if (this.visible) {
					this.refresh();
				} else {
					this.markDirty();
				}
			},
		},
		visible: {
			handler(visible) {
				if (visible && this.isDirty) {
					this.refresh();
				}
			},
			immediate: true
		},
	}
});
</script>

<style lang="scss">

.crumbs-totals {
	margin: 0 -15px 10px;
	display:flex;
	flex-wrap:nowrap;
	align-items:flex-start;
	justify-content:space-between;

	> .breadcrumb.resultscrumb {
		background: white;
		border-bottom: 1px solid rgba(0,0,0,0.1);
		border-radius: 0;
		padding: 12px 15px;
		margin-bottom: 0;
		flex-grow: 1;
	}
	> .result-totals {
		background: white;
		padding: 8px 8px 0 15px;
		flex: none;
	}
}

.no-results-found {
	padding: 1.25em;
	text-align: center;
	font-style: italic;
	font-size: 16px;
	color: #777;
}



table {
	> thead > tr > th {
		text-align: left;
		background-color: white;
		border-bottom: 1px solid #aaa;
		padding-bottom: 5px;
	}

	> tbody > tr > td {
		vertical-align: top;
	}
}

.results-container {
	position: relative;
}

// Make entire row clickable even when the document title is short
.doctitle {
	display: block;
}

</style>
