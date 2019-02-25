<template>
  <div class="row">
    <div v-if="table.cols < 11" class="col"></div>
    <div v-bind:class="'col-' + table.cols">
      <div class="row">
        <div class="col">
          <div class="card rounded-0 p-2">
            <div class="card-header rounded-0 border-0 bg-secondary text-white" v-if="table.header">{{table.header}}</div>
            <div class="card-body p-0">
              <div class="table-responsive m-0 p-0">
                <div v-if="!loaded">Loading...</div>
                <template v-if="loaded">
                  <table class="table table-hover table-bordered data-table bg-white m-0" style="white-space: nowrap">
                    <thead class="thead-light">
                    <tr v-if="filters" class="bg-light">
                      <td v-for="column in table.columns" v-bind:width="column.width || ''" class="p-1 border-right" ref="filterRow">
                        <input v-if="column.filter" @keyup="filter($event, column.name)" type="text" v-bind:placeholder="column.placeholder" class="form-control form-control-sm">
                      </td>
                      <td v-bind:colspan="[button.view, button.edit, button.remove].map(b => b ? 1 : 0).reduce((a, b) => a + b)" width="15%">
                        <button @click="reset" class="btn-sm btn btn-outline-secondary btn-block">{{text.reset}}</button>
                      </td>
                    </tr>
                    <tr>
                      <template v-for="(column, index) in table.columns">
                        <template v-if="!column.hidden">
                          <th v-if="!column.sortable" v-bind:width="column.width || ''" class="">{{column.header}}</th>
                          <th v-else v-on:sort="sort" v-bind:width="column.width || ''" @click="sort({event: $event, column: table.columns[index].name || index, index})" class="sortable ">
                            {{column.header}}
                            <span v-if="index===lastIndex||table.orderBy===column.name">
                                <span class="float-right" v-if="table.orderDirection==='asc'">&#x25B2;</span>
                                <span class="float-right" v-else>&#x25BC;</span>
                              </span>
                            <span v-else class="float-right"><strong>&#9776;</strong></span>
                          </th>
                        </template>
                      </template>
                      <th width="10%" class="actions" colspan="3">Options</th>
                    </tr>
                    </thead>
                    <tbody>
                    <tr v-if="rows.length === 0">
                      <td v-bind:colspan="table.columns.length+1">{{text.empty}}</td>
                    </tr>
                    <tr v-if="rows.length > 0" v-for="(row, rowIndex) in rows" v-bind:class="table.rowClass(row)" v-bind:style="row.style || ''">
                      <td v-if="!table.columns[colIndex].hidden" v-for="(col, colIndex) in row.data" v-bind:class="(table.columns[colIndex].className || '') + ' border-right'">
                        <span v-html="col"></span>
                      </td>
                      <td v-if="button.view">
                        <button class="btn-sm btn btn-outline-secondary btn-block pt-0 pb-0" @click="view({id:row.data[idCol], index: rowIndex})">{{text.view}}</button>
                      </td>
                      <td v-if="button.edit">
                        <button class="btn-sm btn btn-outline-secondary btn-block pt-0 pb-0" @click="edit({id:row.data[idCol],  index: rowIndex})">{{text.edit}}</button>
                      </td>
                      <td v-if="button.remove">
                        <button class="btn-sm btn btn-outline-danger btn-block pt-0 pb-0" @click="remove({id:row.data[idCol],  index: rowIndex})">{{text.remove}}</button>
                      </td>
                    </tr>
                    <tr class="bg-light">
                      <td class="align-middle text-right">{{viewCount}}/{{total}}</td>
                      <td class="p-2" v-bind:colspan="table.columns.length+2">
                        <select v-model="size" class="align-middle form-control-sm m-0">
                          <option v-for='value in limits' :value='value'>{{value}}</option>
                        </select>
                        <div class="float-right btn-group-sm align-middle" role="group">
                          <button :disabled="page===0" type="button" class="btn-primary btn" @click="load(0)">|&#9665; {{text.first}}</button>
                          <button :disabled="page===0" type="button" class="btn-primary btn" @click="load(Math.max(0, page-1))">&#9665; {{text.back}}</button>
                          <button :disabled="viewCount===total" type="button" class="btn-primary btn" @click="load(page+1)">{{text.next}} &#9655;</button>
                          <button :disabled="viewCount===total" type="button" class="btn-primary btn" @click="load(Math.ceil(total/pageSize)-1)">{{text.end}} &#9655;|</button>
                        </div>
                      </td>
                    </tr>
                    </tbody>
                  </table>
                </template>
              </div>
            </div>
          </div>
        </div>
      </div>
    </div>
    <div v-if="table.cols < 11" class="col"></div>
  </div>
</template>

<script>
  /**
   * @author Saman Sedighi Rad<ssedighi@posteo.de>
   * @version 0.1.0
   * @example ../../../readme.md
   */
  export default {
    props: {
      // Routes and URLs
      "get-url": String, "remove-url": String, "view-url": String, "edit-url": String, "query-url": String,

      // Query parameters
      "controller": String, "order-by": String, "order-direction": String,

      // Callbacks
      "row-mapper": Function, "row-class": String,

      "header": String, "title": String, "csrf-token": String, "primary-key": Number, "show-filters": Boolean, "columns": Array, "cols": Number,

      // Buttons
      "button-view": Boolean, "button-edit": Boolean, "button-remove": Boolean,

      // Labels and texts
      "text-view": String, "text-edit": String, "text-remove": String, "text-confirm": String, "text-reset": String, "text-empty": String,
      "text-first": String, "text-back": String, "text-next": String, "text-end": String
    },
    data() {
      return {
        filters: this.showFilters || false,
        loaded: false,
        dataController: this.controller,
        lastIndex: null,
        headerBuilder: this.header || function () { return { "X-CSRF-Token": this.token } },
        rows: [],
        idCol: this.primaryKey || 0,
        total: 0,
        viewCount: 0,
        pageSize: this.pageSize || 15,
        page: 0,
        /** @model */
        size: this.pageSize || 15,
        // Select box of page sizes
        limits: new Array(18).fill(0).map((e, index) => (index + 1) * 5 + 10),
        token: this.csrfToken,
        button: {
          view: this.buttonView,
          edit: this.buttonEdit,
          remove: this.buttonRemove,
        },
        search: {
          query: "",
          field: ""
        },
        text: {
          view: this.textView || "Show",
          edit: this.textEdit || "Edit",
          remove: this.textRemove || "Delete",
          confirm: this.textConfirm || "Delete?",
          reset: this.textReset || "Reset",
          empty: this.textEmpty || "No data",
          first: this.textFirst || "First",
          back: this.textBack || "Back",
          next: this.textNext || "Next",
          end: this.textEnd || "Last",
        },
        url: {
          query: this.queryUrl || '/:controller/index.json?:query',
          get: this.getUrl || '/:controller',
          remove: this.removeUrl || '/:controller/:id',
          view: this.viewUrl || '/:controller/view/:id',
          edit: this.editUrl || '/:controller/edit/:id',
        },
        table: {
          header: this.title,
          columns: this.columns,
          orderBy: this.orderBy,
          orderDirection: this.orderDirection,
          rowMapper: this.rowMapper || this._rowMapper,
          rowClass: this.rowClass || this._rowClass,
          cols: this.cols || 12,
        },
      };
    },

    watch: {
      size() {
        this.pageSize = this.size;
        this.load(0);
      }
    },

    methods: {
      /**
       * Initiates the data loading and emits the update event.
       * @param {Number} page
       * @public
       */
      load(page = 0) {
        this.page = page;
        this.update();
        /**
         * @event update
         * @type {Object}
         */
        this.$emit("update", {
          pageSize: this.pageSize,
          page: page
        });
      },

      /**
       * Builds the query parameters and initiates a GET request
       * to the server with the following parameters:
       * ```javascript
       * const query = {
       *     limit,
       *     page,
       *     sort,
       *     direction,
       *     query,
       *     field,
       * };
       * ```
       * @public
       */
      update() {
        const query = {
          limit: this.pageSize,
          page: this.page,
          sort: this.table.orderBy,
          direction: this.table.orderDirection,
          query: this.search.query,
          field: this.search.field,
        };

        const url = this.url.query.replace(":query", this.buildQuery(query))
          .replace(":controller", this.dataController);

        fetch(url)
          .then((response) => {
            response.json()
              .then((json) => {
                if (!response.ok) {
                  alert("Error processing GET request: " + json.error);
                  return;
                }
                const filter = this.rowMapper || this._rowMapper;
                this.rows = json.data.map(filter);
                this.rows.style = this.rows.style || "";
                this.total = json.total;
                this.viewCount = Math.min((this.page + 1) * this.pageSize, this.total);
                this.loaded = true;
              });
          })
          .catch(alert);
      },

      /**
       * @private
       * @param event
       */
      remove(event) {
        if (confirm(this.text.confirm)) {
          const url = this.url.remove.replace(":id", event.id)
            .replace(":controller", this.dataController);

          fetch(url, {
            headers: this.headerBuilder(),
            method: "DELETE"
          }).then(response => {
            if (!response.ok) {
              alert("Error processing DELETE request");
              return;
            }
            this.rows.splice(event.index, 1);
          }).catch(error => {
            alert(error.response.data.message)
          });
        }
      },

      /**
       * Reset all header input fields. Is internally used
       * via a callback but can also be called on the object.
       * @public
       */
      reset() {
        /**
         * @event reset
         * @type {Object}
         */
        this.$emit("reset");
        this.$refs.filterRow.forEach(td => {
          const el = td.querySelector("input");
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

      /**
       * @private
       * @param {Object} event
       */
      view(event) {
        const url = this.url.view.replace(":id", event.id)
          .replace(":controller", this.dataController);

        this.$router.push({ path: url, params: { id: event.id } });
      },

      /**
       * @private
       * @param {Object} event
       */
      edit(event) {
        const url = this.url.edit.replace(":id", event.id)
          .replace(":controller", this.dataController);

        this.$router.push({ path: url, params: { id: event.id } });
      },

      /**
       * @private
       * @param {Object} event
       */
      sort(event) {
        // Remove any earlier column indicator
        if (this.lastIndex) {
          const col = this.table.columns[this.lastIndex];
          let classes = col.className.split(" ");
          classes.splice(classes.indexOf("sorting"), 1);
          col.className = classes.join(" ");
        }
        this.table.columns[event.index].className += " sorting";
        // reverse order
        if (this.table.orderBy === event.column) {
          this.table.orderDirection = (this.table.orderDirection === "asc" ? "desc" : "asc");
        }
        this.table.orderBy = event.column;
        this.lastIndex = event.index;
        this.load();
      },

      /**
       * Apply filter and reload from server.
       * @param {object} event
       * @param {string} field
       * @private
       */
      filter(event, field) {
        this.search.query = event.target.value;
        this.search.field = field;
        this.load();
      },

      /**
       * Maps every single row by based on the order in
       * the columns array. Can be overriden by row-mapper
       * prop to customize the object returned by the server.
       * @param {object} row
       * @private
       */
      _rowMapper(row) {
        const cols = this.table.columns.map(col => {
          // Allow dot notation for objects
          const name = col.name.split(".");
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

        return {
          data: cols
        };
      },

      /**
       * Default callback for every single table row
       * to return a class-name. Can be overriden by
       * row-class callback.
       * @param {Array} row
       * @private
       */
      _rowClass(row) {
        return "";
      },

      /**
       * Creates a properly escaped URL query string.
       * @param {object} params KV pairs.
       * @private
       * @static
       */
      buildQuery(params) {
        return Object.keys(params)
          .map(key => key + "=" + encodeURIComponent(params[key]))
          .join("&");
      }
    },

    created() {
      this.load();
    }
  }
</script>

<style scoped>
  th.sortable {
    cursor: pointer;
    user-select: none;
  }

  th.sorting, td.sorting {
    background-color: rgba(128, 128, 128, 0.07);
  }

  /* Stablizes column width, doesn't influence relative width. */
  .table td, .table th {
    max-width: 200px;
    overflow: hidden;
    text-overflow: ellipsis;
    white-space: nowrap;
  }

  tr.dim td {
    border-color: lightgray !important;
  }

  tr.dim {
    opacity: 0.5;
  }

  .hidden {
    display: none;
  }
</style>
