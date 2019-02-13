<!--
License: MIT
Author: github.com/srad
-->

<template>
  <div class="card shadow-sm">
    <div class="card-body p-2">
      <div class="table-responsive m-0 p-0">
        <div v-if="!loaded">Lade...</div>
        <template v-if="loaded">
          <table class="table table-hover table-bordered bg-white m-0" style="white-space: nowrap">
            <thead class="thead-light">
            <tr v-if="filters" class="bg-light">
              <td v-for="column in tableColumns" v-bind:width="column.width || ''" class="p-1" ref="filterRow">
                <input v-if="column.filter" @keyup="filter($event, column.name)" type="text" v-bind:placeholder="column.placeholder" class="form-control form-control-sm">
              </td>
              <td v-bind:colspan="[showView, showEdit, showRemove].map(b => b ? 1 : 0).reduce((a, b) => a + b)" width="15%">
                <button @click="reset" class="btn-sm btn btn-outline-secondary btn-block">Reset</button>
              </td>
            </tr>
            <tr>
              <template v-for="(column, index) in tableColumns">
                <template v-if="!column.hidden">
                  <th v-if="!column.sortable" v-bind:width="column.width || ''" class="">{{column.header}}</th>
                  <th v-else v-on:sort="sort" v-bind:width="column.width || ''" @click="sort({event: $event, column: tableColumns[index].name || index})" class="sortable ">
                    {{column.header}}
                    <ion-icon class="float-right mr-1" name="funnel"></ion-icon>
                  </th>
                </template>
              </template>
              <th width="10%" class="actions" colspan="3">Options</th>
            </tr>
            </thead>
            <tbody>
            <tr v-if="rows.length === 0">
              <td v-bind:colspan="tableColumns.length+1">Keine Daten</td>
            </tr>
            <tr v-if="rows.length > 0" v-for="(row, rowIndex) in rows">
              <td v-if="!tableColumns[colIndex].hidden" v-for="(col, colIndex) in row" v-bind:class="tableColumns[colIndex].className || ''">{{col}}</td>
              <td v-if="showView">
                <button class="btn-sm btn btn-outline-secondary btn-block pt-0 pb-0" @click="view({id:row[idCol], index: rowIndex})">View</button>
              </td>
              <td v-if="showEdit">
                <button class="btn-sm btn btn-outline-secondary btn-block pt-0 pb-0" @click="edit({id:row[idCol],  index: rowIndex})">Edit</button>
              </td>
              <td v-if="showRemove">
                <button class="btn-sm btn btn-outline-danger btn-block pt-0 pb-0" @click="remove({id:row[idCol],  index: rowIndex})">Delete</button>
              </td>
            </tr>
            <tr class="bg-light">
              <td class="align-middle text-right">{{Math.min(page*pageSize,total)}}/{{total}}</td>
              <td class="p-2" v-bind:colspan="tableColumns.length+2">
                <select v-model="size" class="align-middle form-control-sm m-0">
                  <option v-for='value in limits' :value='value'>{{value}}</option>
                </select>
                <div class="float-right btn-group-sm align-middle" role="group">
                  <button :disabled="page===1" type="button" class="btn-primary btn" @click="load(1)">&#171; First</button>
                  <button :disabled="page===1" type="button" class="btn-primary btn" @click="load(Math.max(1, page-1))">&#8249; Prev</button>
                  <button :disabled="total<=(pageSize*page)" type="button" class="btn-primary btn" @click="load(page+1)">Next &#8250;</button>
                  <button :disabled="total<=(pageSize*page)" type="button" class="btn-primary btn" @click="load(Math.ceil(total/pageSize))">Last &#8250;</button>
                </div>
              </td>
            </tr>
            </tbody>
          </table>
        </template>
      </div>
    </div>
  </div>
</template>

<script>
    module.exports = {
        props: ["get-url", "remove-url", "view-url", "edit-url", "query-url",
            "csrf-token", "row-mapper", "title", "primary-key", "show-filters",
            "controller", "order-by", "order-direction", "columns",
            "button-view", "button-edit", "button-remove"],
        data() {
            return {
                filters: this.showFilters || false,
                loaded: false,
                dataController: this.controller,
                tableColumns: this.columns,
                tableOrderBy: this.orderBy,
                tableOrderDirection: this.orderDirection,
                tableRowMapper: this.rowMapper || this._rowMapper,
                showView: this.buttonView,
                showEdit: this.buttonEdit,
                showRemove: this.buttonRemove,
                rows: [],
                idCol: this.primaryKey || 0,
                total: 0,
                pageSize: getCookie("pageSize") || 15,
                page: 1,
                size: 15,
                // Select box of page sizes
                limits: new Array(18).fill(0).map((e, index) => (index + 1) * 5 + 10),
                token: this.csrfToken,
                search: {
                    query: "",
                    field: ""
                },
                url: {
                    query: this.queryUrl || '/:controller/index.json?:query',
                    get: this.getUrl || '/:controller',
                    remove: this.removeUrl || '/:controller/delete/:id',
                    view: this.viewUrl || '/:controller/view/:id',
                    edit: this.editUrl || '/:controller/edit/:id',
                }
            };
        },

        watch: {
            size() {
                this.pageSize = this.size;
                this.load(1);
                setCookie("pageSize", this.pageSize);
            }
        },

        methods: {
            async remove(event) {
                try {
                    if (confirm("Delete?")) {
                        // Dont use await, axios raises an exception and it can't be
                        // catched within the reposnse scope.
                        const url = this.url.remove.replace(':id', event.id)
                            .replace(':controller', this.dataController);

                        http.delete(url, {
                            headers: {"X-CSRF-Token": this.token}
                        }).then(response => {
                            this.rows.splice(event.index, 1);
                        }).catch(error => {
                            alert(error.response.data.message)
                        });
                    }
                } catch (e) {
                    alert(e.message);
                }
            },

            reset() {
                this.$emit("reset");
                this.$refs.filterRow.forEach(td => {
                    const el = td.querySelector('input');
                    if (el) {
                        el.value = "";
                    }
                });
                this.search = {
                    query: "",
                    field: ""
                };
                this.load();
            },

            async load(page = 1) {
                this.page = page;
                this.update();
                this.$emit("update", {
                    pageSize: this.pageSize,
                    page: page
                });
            },

            view(event) {
                window.location = this.url.view.replace(':id', event.id)
                    .replace(':controller', this.dataController);
            },

            edit(event) {
                window.location = this.url.edit.replace(':id', event.id)
                    .replace(':controller', this.dataController);
            },

            sort(event) {
                // reverse order
                if (this.tableOrderBy === event.column) {
                    this.tableOrderDirection = (this.tableOrderDirection === 'asc' ? 'desc' : 'asc');
                }
                this.tableOrderBy = event.column;
                this.load();
            },

            filter(event, field) {
                this.search.query = event.target.value;
                this.search.field = field;
                this.load();
            },

            async update() {
                const query = {
                    limit: this.pageSize,
                    page: this.page,
                    sort: this.tableOrderBy,
                    direction: this.tableOrderDirection,
                    query: this.search.query,
                    field: this.search.field,
                };
                this.size = this.pageSize;

                const url = this.url.query.replace(':query', this.buildQuery(query))
                    .replace(':controller', this.dataController);

                const data = (await http.get(url)).data;
                const filter = this.rowMapper || this._rowMapper;
                this.rows = (data[this.dataController] || []).map(filter);
                this.total = data.total;
                this.loaded = true;
            },

            _rowMapper(row) {
                return this.tableColumns.map(col => {
                    // Allow dot notation for objects
                    const name = col.name.split('.');
                    if (name.length > 1) {
                        let val = row[name[0]];
                        for (let i = 1; i < name.length; i += 1) {
                            val = val[name[i]];
                        }
                        return val;
                    }
                    // No object
                    return row[col.name];
                });
            },

            buildQuery: (params) => Object.keys(params).map(key => key + '=' + encodeURIComponent(params[key])).join('&')
        },

        async created() {
            this.load();
        }
    };
</script>

<style scoped>
  th.sortable {
    cursor: pointer;
    user-select: none;
  }
</style>
