<template>
    <div class="wrapper" :style="{ height: height, width: width }">
        <table class="scrolling scrolly scrollx table" ref="table"
               :class="{ freezeFirstColumn: freezeFirstColumn }">
            <thead name="thead" class="scrollsync" ref="thead" v-columns-resizable>
                <tr ref="header">
                    <th v-for="c in visibleColumns" :key="c.id"
                        :style="getWidthStyle(c.width)">
                        <span class="move" @click="sortColumn(c.id)">
                            {{ c.title }}
                            <span v-if="sortedColumn == c.id"
                                  :class="{ up: sortedOrder, down: !sortedOrder }">
                            </span>
                        </span>
                    </th>
                </tr>
                <tr ref="filter" v-if="true">
                    <th v-for="c in visibleColumns" :key="c.id"
                        :style="getWidthStyle(c.width)">
                        <input type="text" class="form-control" v-model="c.filter" />
                    </th>
                </tr>
            </thead>
            <tbody name="tbody" ref="tbody" @scroll="updateSyncedScroll">
                <tr v-for="(r, i) in itemsFilteredSorted" :key="i">
                    <td v-for="c in visibleColumns" :key="c.id"
                        :style="getWidthStyle(c.width)">
                        {{ r[c.id] }}
                    </td>
                </tr>
                <tr v-if="itemsFilteredSorted.length == 0" style="visibility: hidden;">
                    <td v-for="c in visibleColumns" :key="c.id"
                        :style="getWidthStyle(c.width)">
                    </td>
                </tr>
            </tbody>
            <tfoot name="tfoot" class="scrollsync" ref="tfoot" v-if="footer">
                <tr>
                    <th v-for="c in visibleColumns" :key="c.id"
                        :style="getWidthStyle(c.width)">
                        {{ footer[c.id] || '&nbsp;' }}
                    </th>
                </tr>
            </tfoot>
        </table>
    </div>
</template>

<script>
import sortable from 'html5sortable/dist/html5sortable.es';

export default {
    props: {
        items: {
            type: Array,
            required: true,
        },
        columns: {
            type: Array,
            required: true,
        },
        footer: {
            type: Object,
            required: false,
            default: null,
        },
        sortingColumn: {
            type: String,
            required: false,
            default: '',
        },
        sortingOrder: {
            type: Boolean,
            required: false,
            default: true,
        },
        scrollPosition: {
            type: Object,
            required: false,
            default(){
                return {};
            },
        },
        freezeFirstColumn: {
            type: Boolean,
            required: false,
            default: false,
        },
        height: {
            type: String,
            required: false,
            default: '100vh',
        },
        width: {
            type: String,
            required: false,
            default: '100vw',
        },
    },
    data: function(){
        return {
            sortedColumn: '',
            sortedOrder: true,
        };
    },
    computed: {
        visibleColumns(){
            return this.columns.filter((i) => {
                if(i.visible === undefined){
                    i.visible = true;
                }
                return i.visible;
            });
        },
        itemsFilteredSorted(){
            let filterValues = {};

            this.columns.forEach((item) => {
                if(item.filter){
                    filterValues[item.id] = String(item.filter).toLowerCase().trim();
                }
            });

            return [...this.items].sort((a, b) => {
                let cmp = 0;
                let c = this.sortedOrder ? 1 : -1;

                if(a[this.sortedColumn] > b[this.sortedColumn]){
                    cmp = 1;
                } else if(a[this.sortedColumn] < b[this.sortedColumn]){
                    cmp = -1;
                }

                return cmp * c;
            }).filter((item) => {
                let ok = true;

                for(let k in filterValues){
                    if(!String(item[k]).toLowerCase().includes(filterValues[k])){
                        ok = false;
                        break;
                    }
                }

                return ok;
            });
        },
    },
    mounted: function(){
        this.sortedColumn = String(this.sortingColumn);
        this.sortedOrder = Boolean(this.sortingOrder);
        this.initColumnSortable();
        this.updateScrollPosition();
        this.updateSyncedScroll();
    },
    methods: {
        initColumnSortable: function(){
            // Using https://github.com/lukasoppermann/html5sortable
            sortable(this.$refs.header, {
                handle: 'span.move',
                placeholderClass: '<tr><td>&nbsp;</td></tr>',
                forcePlaceholderSize: true,
            });
            this.$refs.header.addEventListener('sortupdate', (e) => {
                let from = e.detail.origin.elementIndex;
                let to = e.detail.destination.elementIndex;
                let col = this.columns[from];
                this.columns.splice(from, 1);
                this.columns.splice(to, 0, col);
                this.$emit('column-reorder');
            });
        },
        updateScrollPosition: function(){
            this.$refs.tbody.scrollTop = this.scrollPosition.top || 0;
            this.$refs.tbody.scrollLeft = this.scrollPosition.left || 0;
        },
        updateSyncedScroll: function(){
            const b = this.$refs.tbody;
            const l = b.scrollLeft;

            const h = this.$refs.thead
            if (h.scrollLeft !== l) {
                h.scrollLeft = l;
            }

            if(this.footer){
                const f = this.$refs.tfoot
                if (f.scrollLeft !== l) {
                    f.scrollLeft = l;
                }
            }

            this.$emit('body-scroll', b.scrollTop, b.scrollLeft);
        },
        sortColumn: function(id){
            if(this.sortedColumn == id){
                this.sortedOrder = !this.sortedOrder;
            } else {
                this.sortedColumn = id;
                this.sortedOrder = true;
            }
            this.$emit('column-sorted', this.sortedColumn, this.sortedOrder);
        },
        getWidthStyle: function(width){
            let w = parseInt(width);
            if(isNaN(w)){
                w = 100;
            }
            w = `${w}px`;
            return {
                width: w,
                minWidth: w,
                maxWidth: w,
            };
        },
    },
    directives: {
        // Forked from https://github.com/Fuxy526/vue-columns-resizable
        'columns-resizable': {
            inserted: function(el, binding, vnode){
                const elementIndex = (parent, child) => {
                    let index = Array.prototype.indexOf.call(parent.children, child);
                    return index;
                };

                const nodeName = el.nodeName;
                if (['TABLE', 'THEAD'].indexOf(nodeName) < 0) return;

                const table = nodeName === 'TABLE' ? el : el.parentElement;
                const thead = table.querySelector('thead');
                const trow = thead.querySelector('tr:first-child');
                const ths = trow.querySelectorAll('th');
                let barHeight = nodeName === 'TABLE' ? table.offsetHeight : trow.offsetHeight;

                if(ths[0] !== undefined){
                    let css = getComputedStyle(ths[0]);
                    ['top', 'bottom'].forEach((side) => {
                        let value = css.getPropertyValue(`padding-${side}`);
                        value = value.replace('px', '');
                        value = parseInt(value);
                        if(isNaN(value)){
                            value = 0;
                        }
                        barHeight -= value;
                    });
                }

                let moving = false;
                let movingIndex = 0;
                let movingWidth = null;

                ths.forEach((th, index) => {
                    th.style.width = th.offsetWidth + 'px';
                    th.style.minWidth = th.style.width;
                    th.style.maxWidth = th.style.width;
                    th.style.paddingRight = '0px';

                    const bar = document.createElement('div');

                    bar.style.height = barHeight + 'px';
                    bar.style.width = '8px';
                    bar.style.cursor = 'col-resize';
                    bar.style.zIndex = 1000;
                    bar.style.float = 'right';
                    bar.style.display = 'inline-block';
                    bar.className = 'columns-resize-bar';

                    bar.addEventListener('mousedown', (e) => {
                        if(e.button === 0){
                            moving = true;
                            movingIndex = index; // elementIndex(th.parentElement, th);
                            document.body.style.cursor = 'col-resize';
                            document.body.style.userSelect = 'none';
                        }
                    });

                    th.appendChild(bar);
                });

                const triggerEventResize = (_index, _width) => {
                    vnode.context.$emit('column-resize', _index, _width);
                    vnode.context.columns[_index].width = +_width.replace(/\D/g, '') || 0;
                };

                const bars = thead.querySelectorAll('.columns-resize-bar');

                document.addEventListener('mouseup', () => {
                    if (!moving) return;

                    moving = false;
                    document.body.style.cursor = '';
                    document.body.style.userSelect = '';
                    const new_index = elementIndex(ths[movingIndex].parentElement, ths[movingIndex]);
                    triggerEventResize(new_index, movingWidth);

                    bars.forEach((bar, index) => {
                        const th = ths[index];
                        th.style.width = th.offsetWidth + 'px';
                        th.style.minWidth = th.style.width;
                        th.style.maxWidth = th.style.width;
                    });
                });

                const cutPx = str => +str.replace('px', '');

                const handleResize = e => {
                    if (moving) {
                        const th = ths[movingIndex];
                        const new_index = elementIndex(ths[movingIndex].parentElement, ths[movingIndex]);
                        movingWidth = cutPx(th.style.width) + e.movementX + 'px';
                        th.style.width = movingWidth;
                        th.style.minWidth = th.style.width;
                        th.style.maxWidth = th.style.width;
                        setTimeout(() => {
                            triggerEventResize(new_index, movingWidth);
                        }, 100);
                    }
                };

                table.addEventListener('mousemove', handleResize);
            },
        },
    },
}
</script>

<!-- Add "scoped" attribute to limit CSS to this component only -->
<style scoped>
    table.scrolling {
        display: flex;
        flex-direction: column;
        flex: 1 1 auto;
        width: 100%;
        height: 100%;
        border-collapse: collapse;
        overflow: hidden;
        /* Use this to create a "dead" area color if table is too wide for cells */
        background-color: #ddd;
        --dead-area-color: #ddd;
    }

    table.scrolling thead,
    table.scrolling tfoot {
        /* Grow automatically to fit content, don't shrink it proportionately to the body. */
        flex: 0 0 auto;
        display: block;
        /* Horizontal scrolling, when allowed, is controlled by JS, not a scroll bar. */
        overflow: hidden;
    }

    table.scrolling tbody {
        display: block;
        flex: 1 1 auto;
        /* Disable all scrolling by default */
        /*overflow: hidden;*/
    }

    /* Turn on vertical scrolling for all elements so scroll bars take up the same space */
    table.scrolling.scrolly tbody,
    table.scrolling.scrolly thead.scrollsync,
    table.scrolling.scrolly tfoot.scrollsync {
        overflow-y: scroll;
        background-color: var(--dead-area-color);
        scrollbar-base-color: var(--dead-area-color);
        scrollbar-face-color: var(--dead-area-color);
        scrollbar-highlight-color: var(--dead-area-color);
        scrollbar-track-color: var(--dead-area-color);
        scrollbar-arrow-color: var(--dead-area-color);
        scrollbar-shadow-color: var(--dead-area-color);
        scrollbar-darkshadow-color: var(--dead-area-color);
    }

    /* Turn on horizontal scrolling for the body only */
    table.scrolling.scrollx tbody {
        overflow-x: scroll;
    }

    /*
    For Webkit, use "dead area" color to hide vertical scrollbar functions in the header and footer.
    Since WebKit supports CSS variables and style attributes don't support pseudo-classes, use variables.
    Display is set because otherwise Chrome ignores the other styling.
    TODO: on Chrome/Safari for Mac, scrollbars are not shown anyway and this creates an extra block. No impact on iOS Safari.
    */
    table.scrolling.scrolly thead.scrollsync::-webkit-scrollbar,
    table.scrolling.scrolly tfoot.scrollsync::-webkit-scrollbar {
        display: block;
        background-color: var(--dead-area-color);
    }
    table.scrolling.scrolly thead.scrollsync::-webkit-scrollbar-track,
    table.scrolling.scrolly tfoot.scrollsync::-webkit-scrollbar-track {
        background-color: var(--dead-area-color);
    }

    /* IE11 adds an extra tbody, have to hide it. */
    table.scrolling tbody:nth-child(3) {
        display: none;
    }

    /* The one caveat to scrolling this way: a hard-set width is required. Can override in thead/tbody slot. */
    table.scrolling td,
    table.scrolling th {
        border: 1px solid #ddd;

        /* These must all be set the same in your overriding CSS */
        width: 10em;
        min-width: 10em;
        max-width: 10em;

        /* Important in case your data is too long for your cell */
        overflow: hidden;
        word-wrap: break-word;
    }

    table.scrolling td {
        background-color: #fff;
    }

    table.scrolling th {
        background-color: #f7f7f7;
    }

    table {
        margin: 0px !important;
    }

    table span.move {
        display: inline-block;
        width: calc(100% - 16px);
        cursor: pointer;
        overflow-x: hidden;
    }

    table span.move > .up:after {
        content: '\2191';
    }

    table span.move > .down:after {
        content: '\2193';
    }

    table.scrolling th span {
        white-space: nowrap;
    }

    table.freezeFirstColumn thead tr,
    table.freezeFirstColumn tbody tr,
    table.freezeFirstColumn tfoot tr {
        /*display: block;*/
        width: min-content;
    }

    table.freezeFirstColumn thead td:first-child,
    table.freezeFirstColumn tbody td:first-child,
    table.freezeFirstColumn tfoot td:first-child,
    table.freezeFirstColumn thead th:first-child,
    table.freezeFirstColumn tbody th:first-child,
    table.freezeFirstColumn tfoot th:first-child {
        position: sticky;
        position: -webkit-sticky;
        left: 0;
        background-color: #f7f7f7;
    }

    .wrapper {
        padding: 0;
        margin: 0;
        overflow: hidden;
    }
</style>
