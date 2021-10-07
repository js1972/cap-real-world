# cap-real-world
My findings from using CAP and Fiori Elements in real-world projects. Issues, workarounds and tips.

## Abbreviations used
| Abbreviation | Description |
| ------------ | ----------- |
| FE           | Fiori Elements |
| CAP          | Cloud Application Programming model |

# MTA (Mult-Target Archive)

### default-env plugin
The CF CLI default-env plugin is a great help in automatically creating your default-env.json file, which provides environment variables so that you can run services locally when developing.

Usage:
```
cf default-env adoptiontracker-srv
```
Where `myapp-srv` is the service instance on cloud foundry you want to pull down env vars from. So in this example I'm getting the env vars for a CAP service called myapp-srv.

NOTE: When you deploay an MTA, destinations seem to be overwritten and so when you try and execute your app locally again it won't find the destination on cloud foundry - requiring you to run `cf default-env myapp-srv` again...

```
resources:

#---------------------------------------------
# Destination to remote service
#---------------------------------------------
- name: remote-api-destination
  type: org.cloudfoundry.managed-service
  parameters:
    config:
      HTML5Runtime_enabled: true
      init_data:
        instance:
          destinations:
          - Authentication: NoAuthentication
            Name: remote-dest
            ProxyType: Internet
            Type: HTTP
            URL: https://blahblahblah/api/v1/search
          existing_destinations_policy: update
      version: 1.0.0
    service: destination
    service-plan: lite
```
Note the `existing_destinations_policy: update` value. To get around having to run default-env after every deploy we can change this setting from `update` to `ignore` (but then - dont forget to set it back if you really want to change the destination config).


# CAP
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

### Long running CAP service handlers
By default a CAP service destination will have a 30 second timeout. If you have a long running process then we can use the `HTML5.Timeout: 300000` (e.g. 5mins) parameter on the destination.
The destination needs to be the CAP service destination (the one which provides *srv-api*).

## Efficient (dare I say best practice) annotation file structure
tbd.

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

entity ActivityType : CodeList {
    key code : Integer enum {
            Renewal     = 0;
            Replacement = 1;
            Upsell      = 2;
            Go_live     = 3;
        };
    category : Integer;
}
```

__Note: If a field with a codelist attached is going to be used as an additional binding on a value help then there seems to be a bug that it cannot have 0's. So make your codelist 1-based if its and Integer key.__

## Security - where to put role collections
Where is the best place to place your apps role collections and security info?
The SAP CAP tutorial says this:

"Further, you can add role collections using the xs-security.json file. Since role collections need to be unique in a subaccount like the xsappname, you can add it here and use the ${space} variable to make them unique like for the xsappname."

However, I prefer to place the role collections for my apps in the xs-security.json file instead of in the mta.yaml, even though you can use either. More complexity is allowed for in the xs-security.json file incuding attribute security.
Also - the best practice is to separate your development layers by sub-account and not space (dev, test, prod). This is the only way to truly keep things separated as a lot of BTP services don't have the concept of a cloud foudry space.

# Fiori Elements

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

## Managed Approuter
When using the managed approuter be careful to set public: true in the sap.cloud node of the fiori apps manifest.json. Without this your app will not be available in the BTP cockpit under HTML5 apps and you won't be able to access it in the Launchpad service.

```
    "sap.cloud": {
        "service": "cappie-service",
        "public": true
    },
```
The public: true setting enables the app to be access by an approuter that is not in the same space (which I guess is the case when using the managed approuter).