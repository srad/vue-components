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

```javascript
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

```javascript
const response = {
    data: [{InvoiceId: 123, ...}, ...],
    total: 123
};
```

## Example

```html
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
  import DataTable from "components/data-table";

  export default {
    data() {
      return {
        rowMapper: (row) => [
          row.InvoiceId, row.FirstName, row.LastName,
          row.InvoiceDate ? row.InvoiceDate.split(" ")[0] : "-", row.DeliveryDate ? row.DeliveryDate.split(" ")[0] : "-",
          row.InvoiceType, row.InvoicePayment, row.InvoiceStatus,
          row.ItemCount, row.ItemSum + "€",
        ],
        columns: [
          { name: "InvoiceId", header: "Invoice-Id", width: "10%", sortable: true, className: "text-right" },
          { name: "FirstName", filter: true, header: "Firstname", width: "10%" },
          { name: "LastName", filter: true, header: "Lastname", width: "10%" },
          { name: "InvoiceDate", filter: true, sortable: true, header: "Invoice-Date", width: "10%" },
          { name: "DeliveryDate", filter: true, sortable: true, header: "Delivery-Date", width: "10%" },
          { header: "Type", width: "10%" },
          { header: "Payment", width: "5%" },
          { header: "State", width: "5%" },
          { header: "Items", width: "5%", className: "text-right" },
          { header: "Value", width: "5%", className: "text-right" },
        ]
      }
    },
    components: {
      "data-table": DataTable
    },
  }
</script>
```

## Todo

1. ~~Translation via props.~~
1. ~~Provide REST callback, to remove the use of global axios.~~
1. ~~Provide proper for HTML class names.~~

## License
MIT, do what you want.