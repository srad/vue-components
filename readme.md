# vue data-table

General Vue data-table component, example:

![](https://github.com/srad/vue-components/raw/master/doc/demo1.png)

Example
-------

Features:

1. CRUD button with proper route redirects for vue.
1. Automatic pagination with automatic button activation/deactivation.
1. Out of the box support for sorting, if you respond to the GET query parameters (see example).
1. Support for column filters (generate a new GET query based on the `columns` declaraction).
1. An optional callback `row-mapper` can alter the server response, i.e. to format dates.
1. The default class declarations are for Bootstrap 4, could also be improved by making optional props.
1. You can provide a CSRF token (see example).
1. You can provide and format the REST URLs in `/:route/:style`.

This object is queried on the GET request, the sever can appropriate react to them:

```
const query = {
    limit,
    page,
    sort,
    direction,
    query,
    field,
};
```

Following response is expected:

```
const response = {
    data: [...],
    total: 123
};
```

## Example

This example uses a PHP CSRF token from CakePHP 3.x and httpVueLoader, to load the component.

```
<template>
  <data-table v-bind:columns="columns"
              v-bind:primary-key="0"
              v-bind:button-view="true"
              v-bind:button-edit="false"
              v-bind:button-remove="true"
              v-bind:row-mapper="rowMapper"
              view-url="/invoice/view/:id"
              edit-url="/invoice/edit/:id"
              remove-url="/api/:controller/delete/:id"
              query-url="/api/:controller?:query"
              get-url="/api/:controller"
              csrf-token=""
              controller="invoices"
              order-by="InvoiceId"
              show-filters="true"
              order-direction="desc">
  </data-table>
</template>

<script>
    new Vue({
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
        }
    })
</script>
```

## Todo

1. ~~Translation via props.~~
1. ~~Provide REST callback, to remove the use of global axios.~~
1. Provide proper for HTML class names.

## License
MIT, do what you want.