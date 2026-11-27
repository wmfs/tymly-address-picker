# tymly-address-picker

[![Tymly Cardscript](https://img.shields.io/badge/tymly-cardscript-blue.svg)](https://tymly.io/)
[![npm (scoped)](https://img.shields.io/npm/v/@wmfs/tymly-address-picker.svg)](https://www.npmjs.com/package/@wmfs/tymly-address-picker)
[![Commitizen friendly](https://img.shields.io/badge/commitizen-friendly-brightgreen.svg)](http://commitizen.github.io/cz-cli/)
[![license](https://img.shields.io/github/license/mashape/apistatus.svg)](https://github.com/wmfs/tymly/blob/master/packages/concrete-paths/LICENSE)

> Tymly gazetteer address picker for the cardscript `Input.Address` element, as a Quasar component

States: search (as you type, or on Enter), results with Official/Temporary tags and UPRN, too many matches,
no matches, search unavailable, validation error, address chosen (with map), manual entry (with a draggable pin),
and a phone layout.

## <a name="install"></a>Install
```bash
$ npm install @wmfs/tymly-address-picker --save
```

## <a name="usage"></a>Usage

Register the component with the components your cardscript templates are compiled against:

```js
import TymlyAddressPicker from '@wmfs/tymly-address-picker'

export default {
  components: { TymlyAddressPicker, ... }
}
```

Then use `Input.Address` in a card template. `@wmfs/cardscript-to-quasar` renders it as `<tymly-address-picker>`:

```json
{ "type": "Input.Address", "id": "selectedUprn" }
```

### Props

| Prop | | |
| --- | --- | --- |
| `config` | required | Resolved settings, made by cardscript-to-quasar from the element |
| `model` | required | Card data. The picker reads it and emits `update` with the values to merge; it never changes it directly |
| `validation` | | `v$.data` from vuelidate |
| `invalidFields` | | Fields that failed on submit, with messages |
| `execute` | | `async ({ stateMachineName, input, sendResponse }) => ({ status, ctx })`. Defaults to `this.$tymly.executions.execute` (TymlySDK from `@wmfs/tymly-cardscript-sdk`) |
| `mapboxAccessToken` | | For the still map of the chosen address. Defaults to `process.env.MAPBOX_ACCESS_TOKEN`, as `@wmfs/quasar-map-mapbox` uses |

### Events

`update (patch)`: values to merge into the card data.

### Element settings

All optional, see the `Input.Address` definition in `@wmfs/cardscript-schema`:
`idPrefix`, `title`, `subtitle`, `placeholder`, `searchEndpoint` (default `wmfs_addressPickerSearch_1_0`),
`detailsEndpoint` (default `wmfs_addressPickerDetails_1_0`, `false` to turn off),
`searchFields`, `resultsPerPage`, `tooManyResults`, `searchAsYouType`, `allowGeoLocation`, `showMap`, `markerClick` (`confirm` or `select`), `markerActionLabel`,
`allowManualEntry`, `manualEntryTitle`, `manualEntryNotice`, `manualEntryFields`, `onSelectionEndpoint`, `validation`.

### Data written (prefixed with `idPrefix_` when set)

| Key | |
| --- | --- |
| `<id>` | Selected UPRN |
| `addressLabel` | Label of the chosen address |
| `selectedAddress` | `{ uprn, label, line, locality, postcode, status, dataSource, dataSourceLabel, temporary, latitude, longitude, details, ... }` |
| `cannotFindAddress` | `true` when entered by hand |
| `addressLine1`, `addressLine2`, `townOrCity`, `county`, `postcode` (and any other `manualEntryFields`) | Manual entry |
| `latitude`, `longitude` | Pin from manual entry |

The search endpoint is given `{ filter, page, limit, fields?, useGeoLocation?, $GEOLOCATION? }` and should return
`{ totalHits, totalPages, page, results: [{ uprn, label, line, locality, postcode, reference, status, dataSource, dataSourceLabel, temporary, latitude, longitude }] }`.

Once an address is chosen, the details endpoint is given `{ uprn }` and should return
`{ found, maskingNumber, classification: { code, title, path }, status, hierarchy: { parentUprn, parentAddressLabel, siblingCount, childrenCount } }`.
The picker keeps them in `selectedAddress.details` and shows them as on the gazetteer property card.
If they fail or take longer than 15 seconds, it says they aren't available, with a link to try again.

Each result's tag shows `dataSourceLabel` (no tag when it's empty). `temporary` results get the highlighted tag.
`wmfs_addressPickerSearch_1_0` fills these from the gazetteer `data_source`, with labels from
`gazetteer-property-blueprint/shared/address-data-sources.json`.

## <a name="license"></a>License
[MIT](https://github.com/wmfs/tymly/blob/master/packages/concrete-paths/LICENSE)
