# cap-real-world
My findings from using CAP and Fiori Elements in real-world projects. Issues, workarounds and tips.

## Abbreviations used
| Abbreviation | Description |
| ------------ | ----------- |
| FE           | Fiori Elements |
| CAP          | Cloud Application Programming model |

## Service Handlers
`each` has a special meaning in handler parameter names.
By naming the event or parameter in a handler `each` it will be called as a per-row handler - as a convenience shortcut.
```
this.after('each','Books', (book)=>{
  book.stock > 111 && book.discount='11%'
})
```

```
this.after('READ','Books', (each)=>{
  each.stock > 111 && each.discount='11%'
})
```

[Details](https://cap.cloud.sap/docs/node.js/services#event-handlers)

## Remote Services
tbd.

## Efficient (dare I say best practice) annotation file structure
tbd.

## Value Helps
When creating value helps with dependent values on other fields I have noticed that they do not work when the dependent value is a `0`.

For example - this value help on ActivityType is dependent (or should be filtered) on the category, which is a separate field on the page:
```
annotate CustomersService.Activities with {
    activity_type @Common.ValueList: {
        CollectionPath : 'ActivityType',
        Label : 'Activity Type',
        Parameters : [
            {$Type: 'Common.ValueListParameterInOut', LocalDataProperty: activity_category_code, ValueListProperty: 'category'},
            {$Type: 'Common.ValueListParameterInOut', LocalDataProperty: activity_type_code, ValueListProperty: 'code'},
            {$Type: 'Common.ValueListParameterDisplayOnly', ValueListProperty: 'name'}
        ]
    };
}
```
If the category_code values can be 0, 1, 2, 3, 4 for example and 0 is chosen by the user, the filtering will not work!

__So your dependent values MUST NOT BE 0-based if they are Integers.__

## FE General
Please always lock your app to a specific version of SAPUI5. I have found that FE is notorious for breaking when new patch levels are released.

## CodeLists
Make sure to only have one key field in a code list, otherwise FE won't be able to display the values properly. See examples here, particularly the last one which shows how to add additional fields (non key):
```
entity ActivityStatus : CodeList {
    key code : Integer enum {
            Planned    = 0;
            InProgress = 1;
            Cancelled  = 2;
            Complete   = 3;
        } default 0;
};

entity ActivityCategory : CodeList {
    key code : Integer enum {
            Adoption    = 0;
            Transaction = 1;
        }; // default 0;
};

entity ActivityType : CodeList {
    key code : Integer enum {
            Renewal     = 0;
            Replacement = 1;
            Upsell      = 2;
            Go_live     = 3;
            Reference   = 4;
            Kick_off    = 5;
            Business_Process_Improvement = 6;
            Harmonisation = 7;
            LOB_Extension = 8;
            Pilot = 9;
            Value_Prop = 10;
            Architecture = 11;
        };
    category : Integer;
}
```

__Note: If a field with a codelist attached is going to be used as an additional binding on a value help then there seems to be a bug that it cannot have 0's. So make your codelist 1-based if its and Integer key.__
