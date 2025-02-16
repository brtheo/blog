---
tags:
  - salesforce
  - lwc
  - javascript
  - js
---
We're all gonna agree on this, using `LWC` over `Aura` is way more enjoyable as it brings a much more robust and up to date API, using the [Web Component](https://developer.mozilla.org/en-US/docs/Web/API/Web_components/Using_custom_elements) native web technology, to work with.

What's troublesome though, in my opinion, is the use of the functions `getFieldValue()` and `getRecord()`.

Here's what's the [official doc](https://developer.salesforce.com/docs/component-library/documentation/en/lwc/reference_get_field_value) is saying :
```typescript
import { LightningElement, api, wire } from 'lwc';
import { getRecord, getFieldValue } from 'lightning/uiRecordApi';

import REVENUE_FIELD from '@salesforce/schema/Account.AnnualRevenue';
import CREATED_FIELD from '@salesforce/schema/Account.CreatedDate';
import EXP_FIELD from '@salesforce/schema/Account.SLAExpirationDate__c';

const fields = [REVENUE_FIELD, CREATED_FIELD, EXP_FIELD];

export default class WireGetValue extends LightningElement {
    @api recordId;

    @wire(getRecord, { recordId: '$recordId', fields })
    account;

    get revenue() {
        return getFieldValue(this.account.data, REVENUE_FIELD);
    }

    get created() {
        return getFieldValue(this.account.data, CREATED_FIELD);
    }

    get expiration() {
        return getFieldValue(this.account.data, EXP_FIELD);
    }
}

// .html
<div>
  Field 1: {revenue}
  Field 2: {created}
  Field 3: {expiration}
</div>
```

Following this example leads to a **lack of reusability** of the code and a **lack of context** (as it's not explicit what does the getters `revenue`,`created` and `expiration` of this example represent).

So, let's bring more flexibility to this code by using native javascript capabilities.

## Clear context = Clear code
First I would advise to run the sfdx command `SFDX: Refresh SObject Definitions` from the VSCODE command palette.

It will allow you to have the typescript LSP running and giving you hint on what to type, especially for the import statement.
```typescript
// stop doing this as you lose information about the context
import REVENUE_FIELD from '@salesforce/schema/Account.AnnualRevenue';

// do this instead, the name and the import location are suggested by the typescript LSP
import AnnualRevenue from '@salesforce/schema/Account.AnnualRevenue';

```
The part about the `fields` array constant doesn't need to be changed, it'll be really useful.
```typescript
const fields = [AnnualRevenue, CreatedDate, SLAExpirationDate__c];
```

The big change is moving from a model where we need to create as many getters as we have fields imported to a model where we would have **only one getter** called by the SObject API name it belongs to that will return an object with each key being the field imported.

```typescript
@wire(getRecord, { recordId: '$recordId', fields })
_account

get Account() {
    return Object.fromEntries(
        new Map(fields.map(field =>
            [field.fieldApiName, getFieldValue(this._account.data, field)]
        ))
    )
}
//equivalent of :
get Account() {
  return {
    'AnnualRevenue': getFieldValue(this._account.data, AnnualRevenue),
    'CreatedDate': getFieldValue(this._account.data, CreatedDate),
    'SLAExpirationDate__c': getFieldValue(this._account.data, SLAExpirationDate__c),
  }
}
```

Here we use the previously created `fields` array to map it into a `Map` that takes as a key the `fieldApiName` of the current iteration and as a value the results of the `getFieldValue()` function.
And we using `Object.fromEntries` to simply convert the created `Map` into an object literal.

Now, when referencing these values in the html or js, it'll be more readable to have somewhere in your code `Account.AnnualRevenue` compared to just `revenue` as it was presented on the official doc as now you have a direct clear view of what the data represents.

## Composition and reusability
Imagine you have 50+ LWC in your project. Each needs to access datas on an object.
Unless you're a fool you won't copy paste this code 50 times.

So let's make it flexible.

Start by creating a new LWC that will serve as a mixin to compose our needs. I called mine `useRecordFields`.
```typescript
import { wire } from 'lwc';
import { getRecord, getFieldValue } from 'lightning/uiRecordApi';

export function useAccountFields(constructorLike, fields) {
  return class extends constructorLike {
    @wire(getRecord, {recordId: '$recordId', fields: fields })
    _account;

    get Account() {
      return Object.fromEntries(
        new Map(fields.map(field =>
          [field.fieldApiName, getFieldValue(this._account.data, field)]
        ))
      );
    }
  }
}
```
Then for each SObject that I want to import fields from, I would export a new mixin.
```typescript
import { wire } from 'lwc';
import { getRecord, getFieldValue } from 'lightning/uiRecordApi';

export function useAccountFields(constructorLike, fields) {
  ...
}

export function useCaseFields(constructorLike, fields) {
  return class extends constructorLike {
    @wire(getRecord, {recordId: '$recordId', fields: fields })
    _case;

    get Case() {
      return Object.fromEntries(
        new Map(fields.map(field =>
          [field.fieldApiName, getFieldValue(this._case.data, field)]
        ))
      );
    }
  }
}
```

Now inside your main component all you need to do is this
```typescript
import { LightningElement, api } from 'lwc';

import AnnualRevenue from '@salesforce/schema/Account.AnnualRevenue';
import CreatedDate from '@salesforce/schema/Account.CreatedDate';
import SLAExpirationDate__c from '@salesforce/schema/Account.SLAExpirationDate__c';

import { useAccountFields } from 'c/useRecordFields';

const fields = [AnnualRevenue, CreatedDate, SLAExpirationDate__c];

export default class myComponent extends useAccountFields(LightningElement, fields) {
  @api recordId; // still important

  handleClick() {
    console.log(this.Account.AnnualRevenue);
  }
}
```

Hope this helps !

## UPDATE

The way we wrote the mixin classes `useAccountRecords` or `useCaseRecords` is still very imperative.
Instead we can think of a way to make it more generic like so
```typescript
export function useRecordFields(genericConstructor, {recordId, fields}) {
  const {objectApiName} = fields[0];
  class placeholder extends genericConstructor {
    @wire(getRecord, {recordId: '$recordId', fields: fields})
    _fields
  }
  Object.defineProperty(placeholder.prototype, objectApiName, {
      get() {
        return Object.fromEntries(fields.map((field) => {
          return [field.fieldApiName, getFieldValue(this._fields.data, field)]
        }))
      }
  });
  return placeholder;
}
```

Here on line 2, we are grabbing the `objectApiName` prop from the first element of our fields array we provided as a parameter of our mixin, giving us the `string` value of the SObject it belongs to.

When you import a field in LWC it has this shape, imagine we're in typescript
```typescript
type SObjectField = {
  fieldApiName: string; // => Id/LastModifiedDate/CustomField__c/whatever...
  objectApiName: string; // => Case/Quote/CustomObject__c/whatever...
}
```

So that then we can use `Object.defineProperty()` on the prototype of our placeholder class to dynamically create a getter whose `key` will be this `objectApiName`.
