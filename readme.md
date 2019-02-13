vue data-table
--------------

General Vue data-table compoent, example:

![](https://github.com/srad/vue-components/raw/master/doc/demo1.png)

Example
-------

```
<div id="root">
  <data-table v-bind:columns="columns"
              v-bind:primary-key="0"
              v-bind:button-view="true"
              v-bind:button-edit="false"
              v-bind:button-remove="true"
              v-bind:row-mapper="rowMapper"
              v-bind:view-url="'/:controller/view/:id'"
              v-bind:edit-url="'/:controller/edit/:id'"
              v-bind:remove-url="'/:controller/delete/:id'"
              v-bind:query-url="'/:controller/index.json?:query'"
              v-bind:get-url="'/:controller')"
              csrf-token="<?= $this->getRequest()->getParam('_csrfToken') ?>"
              controller="invoices"
              order-by="invoice_id"
              show-filters="true"
              order-direction="desc">
  </data-table>
</div>

<script>
    CreateVue({
        el: "#root",
        data: {
            rowMapper: (row) => [
                row.invoice_id, (row.customer.name.length > 15 ? (row.customer.name.substr(0, 15) + '...') : row.customer.name),
                moment(row.invoice_date).format('DD.MM.Y'), moment(row.delivery_date).format('DD.MM.Y'),
                row.invoice_type.display, row.invoice_payment.display, row.invoice_status.display,
                row.item_count, (row.total || 0) + "€",
            ],
            columns: [
                {name: "invoice_id", header: "Rchn.Nr.", width: "10%", sortable: true},
                {name: "customers.name", filter: true, header: "Kunde", width: "20%", sortable: true},
                {name: "invoice_date", filter: true, sortable: true, header: "Rechnungsdatum", width: "15%"},
                {name: "delivery_date", filter: true, sortable: true, header: "Lieferdatum", width: "15%"},
                {header: "Art", width: "15%", sortable: true},
                {header: "Zahlung", width: "5%", sortable: true},
                {header: "Status", width: "5%", sortable: true},
                {header: "Bstl.", width: "5%"},
                {header: "Summe", width: "10%"},
            ],
        },
        components: {
            "data-table": httpVueLoader("../js/src/component/vue/datatable.vue")
        },
    })
</script>
```