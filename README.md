# Krauzi's Fails - A Guide to ServiceNow

In this reference we want to share my experience with the ServiceNow - Platform and ease the adoption of Best - Practice and ServiceNow specific clean code.

## How to g_form in your console
Service Portal UI:
```javascript
var g_form = angular.element(document).find('div.form-group[field*=formModel]').first().scope().getGlideForm()
```
Standard UI: Just select the correct frame and g_form is available right away
<img width="360" alt="image" src="https://user-images.githubusercontent.com/5402583/228592003-7e526123-c651-45e0-adf1-8a689e75ac6e.png">


## Transforming Filters into GlideRecord - Queries

I want to present you a few techniques on how to process data from a specific table. You probably want to process only certain records and your starting point is the ServiceDesk with the table opened(Tipp: Enter<table_name>.list or<table_name>.LIST in the Application Navigator's search bar to jump directly to the table's list).
Update 2022: I've come to the conclusion, that using addActiveQuery shouldn't be used - it doesn't handle custom global fields ("u_active" - yeah you probably shouldn't be doing unscoped fields anyways) and it only encourages inconsistant coding.
Update November 2023: Simply use the DevOps+ Background Script feature:
![image](https://github.com/kr4uzi/ServiceNow-Guide/assets/5402583/0f1b9463-6125-490f-b3a3-58bd29cd598a)


### Simple Query

![image](https://user-images.githubusercontent.com/5402583/228593358-330adc39-f837-45ec-8002-c08096468fe5.png)

Using encoded Query:

```javascript
function getEmailIncidents() {
    var incidentGr = new GlideRecord("incident");
    incidentGr.addEncodedQuery("active=true^description=email");
    incidentGr.query();

    var sysIds = [];
    while (incidentGr.next()) {
        sysIds.push(incidentGr.sys_id.toString());
    }

    return sysIds;
}
```

Using explicit queries:

```javascript
function getEmailIncidents() {
    var incidentGr = new GlideRecord("incident");
    incidentGr.addQuery("active", "true");
    incidentGr.addQuery("description", "CONTAINS", "email");
    incidentGr.query();

    var sysIds = [];
    while (incidentGr.next()) {
        sysIds.push(incidentGr.sys_id.toString());
    }

    return sysIds;
}
```

### Advanced Query
![image](https://user-images.githubusercontent.com/5402583/228593436-4bda9116-1034-4e17-80ea-515e62c69fb5.png)

```javascript
var incidentGr = new GlideRecord("incident");
incidentGr.addEncodedQuery("active=true^descriptionLIKEemail^ORdescriptionLIKEserver");
incidentGr.query();
```

```javascript
var incidentGr = new GlideRecord("incident");
incidentGr.addQuery("active", "true");
var descCond = incidentGr.addQuery("description", "CONTAINS", "email");
descCond.addOrCondition("description", "CONTAINS", "server");
incidentGr.query();
```

## addQuery vs.addEncodedQuery

** TL; DR **: avoid addEncodedQuery when possible

Recall the very first example of how to transform a filter into a GlideRecord - Query: If you do not have to modify the query - string afterwards, you are save to go with addEncodedQuery.If you add an dynamic part to the query string that might contain user input data, you can fall into the same pit as if you were handling raw database queries(SQL - Injection).See the following code for example:

```javascript
function getIncidentsByDescriptionBad(descriptionText) {
    var incidentGr = new GlideRecord("incident");
    incidentGr.addEncodedQuery("descriptionLIKE" + descriptionText + "active=true");
    incidentGr.query();
    gs.info(incidentGr.getRowCount());
}

function getIncidentsByDescriptionGood(descriptionText) {
    var incidentGr = new GlideRecord("incident");
    incidentGr.addQuery("description", descriptionText);
  	incidentGr.addActiveQuery();
    incidentGr.query();
    gs.info(incidentGr.getRowCount());
}

getIncidentsByDescriptionBad("^CRASH"); // prints xx (number of incidents)
// addEncodedQuery can be hijacked
getIncidentsByDescriptionGood("^CRASH"); // prints 0 (most likely)
// addQuery can *not* be hijacked
```

addEncodedQuery and get are one of the functions that if not called very carefully can not only be potentially a performance bottleneck but also dangerous to your data integrety.

The property controlling this behavior is: **glide.invalid_query.returns_no_rows** (this property must be set to **true** if no results should be returned in case of an invalid query).

> An incorrectly constructed encoded query, such as including an ** invalid field name **, produces an invalid query.When the invalid query is run, the invalid part of the query condition is dropped, and the results are based on the valid part of the query, which may return all records from the table.Using an ** insert() **, ** update() **, ** deleteRecord() **, or ** deleteMultiple() ** method on bad query results can result in data loss.

[Source: ServiceNow API Doc](https://developer.servicenow.com/app.do#!/api_doc?v=newyork&id=r_ScopedGlideRecordAddEncodedQuery_String)

  ## GlideRecord Specification

### Lookup
[Source: ServiceNow API Doc](https://developer.servicenow.com/dev.do#!/reference/api/orlando/server/no-namespace/c_GlideRecordScopedAPI)
```typescript
class GlideQueryCondition {
  addCondition(field: string, operator: string, value?: object): GlideQueryCondition;
  addOrCondition(field: string, operator: string, value?: object): GlideQueryCondition;
}

class GlideRecord {
  constructor(table: string);

  addEncodedQuery(query: string): void;

  // == addQuery("active", true)
  addActiveQuery(): GlideQueryCondition;
  // == addQuery(field, "=", value)
  addQuery(field: string, value: object): GlideQueryCondition;
  // operators:
  // =, !=, >, >=, <, <=
  // "IN", "NOT IN"
  // "STARTSWITH", "CONTAINS", "DOES NOT CONTAIN"
  // "INSTANCEOF"
  // object-> cannot be an array
  addQuery(field: string, operator: string, value: object): GlideQueryCondition;
  addNotNullQuery(field: string): GlideQueryCondition;
  // equivalent to: addQuery(field, "");
  addNullQuery(field: string): GlideQueryCondition;

  orderBy(field: string): void;
  orderByDesc(field: string): void;

  // INNER JOIN <table> ON <GlideRecord>.grField = <table>.tableField
  addJoinQuery(table: string, grField: object, tableField: object): GlideQueryCondition;

  // see best-practice:setLimit
  setLimit(count: number): void;

  // dont forget to actually perform the query!
  query(): void;

  // be extremly cautious when using the other overload of this method:
  // get(field: string, value: object), see: Hall of Shame:get
  get(sys_id: object): boolean;

  hasNext(): boolean;  
  next(): boolean;

  getRowCount(): number; // not the best-practice, see Best-Practice:getRowCount
  getTableName(): string;
  getRecordClassName(): string;

  // see: best-practice:cleancode
  isValidRecord(): boolean;
  // see: best-practice:getValue
  getValue(field: string): string;
}
```

### Insert

```typescript
class GlideRecord {	
  // prepare for new insertion (doesnt insert yet, doesnt set defaults)
  initialize(): void;
  
  // performs insert and returns the sys_id
  insert(): string;
  
  // sets default values (+ sys_id!), insert() must be called to persist the record
  newRecord(): void;
  
  // optional reason for audit record
  update(reason?: string): void;
   
  // must be used in batch updates
  setValue(field: string, value: string): void;
  
  // when updating/insert: enable or disable if business rules are applied
  setWorkflow(enabled: boolean): void;
  
  // see: batch update/delete
  updateMultiple(): void;
}
```

### Delete

```typescript
class GlideRecord {
  // delete current record
  deleteRecord(): void;
  
  // see: batch update/delete
  deleteMultiple(): void;
}
```

## GlideElement Specification

```typescript
class GlideElement {
  getTableName(): string; // var gr = new GlideRecord("incident"); getTableName() == "incident"
  getRecordClassName(): string; // var gr = new GlideRecord
}
```

## Update / Delete multiple Records at once

```javascript
function deleteEmailIncidents() {
  var incidentGr = new GlideRecord("incident");
  incidentGr.addActiveQuery();
  incidentGr.addQuery("description", "CONTAINS", "email");
  incidentGr.deleteMultiple(); // no .query()!
}

function closeEmailIncidents() {
  var incidentGr = new GlideRecord("incident");
  incidentGr.addActiveQuery();
  incidentGr.addQuery("description", "CONTAINS", "email");
  incidentGr.setValue("state", 3); // DO NOT incidentGr.state = 3; !!
  incidentGr.updateMultiple();
}
```

Regarding Line 12: 

> When using ** updateMultiple() **, directly setting the field(gr.state = 3) results in all records in the table being updated instead of just the records returned by the query.

[Source: ServiceNow API Doc](https://developer.servicenow.com/app.do#!/api_doc?v=newyork&id=r_ScopedGlideRecordUpdateMultiple)

Batch update/delete only works with static updates, if you want to dynamically update, you have to use an loop in conjuction with "gr.update()".

```javascript
function replaceEmailIncidents() {
  var incidentGr = new GlideRecord("incident");
  incidentGr.addActiveQuery();
  incidentGr.addQuery("description", "CONTAINS", "email");
  incidentGr.query();
  while (incidentGr.next()) {
    incidentGr.description = incidentGr.description.toString().replace("email", "");
    incidentGr.update();
  }
}
```

# Best - Practice Code Examples

## getValue on non - string Fields

```typescript
function getBoolean() {
  var taskGr = new GlideRecord("task");
  taskGr.setLimit(1);
  taskGr.query();
  taskGr.next(); // lets assume there is at least one record in the task table
  return !!taskGr.active; // !! does a cast to bool
}
```

## setLimit

```javascript
function emailIncidentExists() {
  var incidentGr = new GlideRecord("incident");
  incidentGr.addQuery("active", true);
  incidentGr.addQuery("description", "CONTAINS", "email");
  incidentGr.setLimit(1);
  incidentGr.query();
  if (incidentGr.hasNext()) {
    return true;
  }
  
  return false;
}
```

By using "hasNext()" instead of "next()" in Line 7 we avoid loading the actual values of record into incidentGr and thus saving a few milliseconds. If you encounter queries of this type, form a habit of structuring your functions like the example code.
Note: The same applies to a Sys ID lookup - instead of using .get, use .addQuery("sys_id", ...) to save a few milliseconds.

## getRowCount

```typescript
function numberOfEmailIncidentsBad() {
  var gr = new GlideRecord("incident");
  gr.addActiveQuery();
  gr.addQuery("description", "CONTAINS", "email");
  gr.query();
  return gr.getRowCount();
}

function numberOfEmailIncidentsGood() {
  var ga = new GlideAggregate("incident");
  ga.addActiveQuery();
  ga.addQuery("description", "CONTAINS", "email");
  ga.addAggregate("COUNT");
  ga.query();
  return ga.next() ? +ga.getAggregate("COUNT") : 0;
}
```

GlideRecord:: getRowCount() can have negative performance impact on the database and should therefore not be used in production. Use GlideAggregate instead.

## getValue

```javascript
function emailIncidentUsers() {
  var incidentGr = new GlideRecord("incident");
  incidentGr.addEncodedQuery("active", true);
  incidentGr.addQuery("description", "email");
  incidentGr.addNotNullQuery("opened_by");
  incidentGr.query();

  var sysIds = [];
  while (incidentGr.next()) {
    var userSysId = incidentGr.getValue("opened_by");
    // vs. incidentGr.opened_by.sys_id.toString();
    sysIds.push(userSysId);
  }

  return sysIds;
}
```

When you want to retrieve the sys_id of a reference field, you should use getValue(field_name) instead of<field_name>.sys_id.toString(). When using the latter, you will essentially load the whole record of<field_name>.
Be carefull however for there are the following things to bear in mind when using getValue:

- getValue cannot be called on a GlideElement in scoped applications(`incidentGr.opened_by.getValue("company")` doesnt work)
- getValue will return `null` when a certain reference field is not set(use`getValue("field") || ""`, see code below)
- <glideRecord>.<glideElement>.toString() will always return a string even if the reference represented by <glideElement> is not set
- only use getValue when you are absolutely certain that the field retrieving can never be empty

```typescript
function getOpenedBy(incidentSysId) {
  var incidentGr = new GlideRecord("incident");
  incidentGr.addQuery("sys_id", incidentSysId);
  incidentGr.setLimit(1);
  incidentGr.query();
  if (incidentGr.next()) {
    return gr.getValue("opened_by") || "";
  }

  return "";
}
```

## Clean Code

Try not to build fatal code, this usually implies not returning "null" or "undefined". Consider the following code examples:

```javascript
function getIncidentGrGood(sysIdStr) {
  var incidentGr = new GlideRecord("incident");
  incidentGr.get(sysIdStr);
  return incidentGr;
}

function getIncidentGrBad(sysIdStr) {
  var incidentGr = new GlideRecord("incident");
  if (incidentGr.get(sysIdStr)) {
    return incidentGr;
  }

  return null;
}
```

Why should "getIncidentGrGood" be prefered over "getIncidentGrBad"?
The following example should illustrate why it is of advantage to always return the same type of value.

```javascript
function deleteIncidentGr(incidentGr) {
  // no null check required
  if (incidentGr.isValidRecord()) {
        incidentGr.deleteRecord();
  }
}

deleteIncidentGr(getIncientGrGood("<a valid sys_id>"));
```

Not only do you save quite some typing, but you also provide a stable interface (getValue for instance is not a stable interface because it returns null in some cases, see: getValue), that returns an valid object in any possible situation. Plus you'd have to do a isValidRecord on the returned object anyways.

## Client Side Dates

```javascript
function getDateFromDateField(fieldName) {
  var date_str = getDateFromFormat(g_form.getValue(fieldName), g_user_date_format);
  return new Date(date_str);
}

function getDateFromDateTimeField(fieldName) {
  var date_str = getDateFromFormat(g_form.getValue(fieldName), g_user_date_time_format);
  return new Date(date_str);
}
```

## UI-Action

```javascript
// action_name: <scope>_<action_identifier>
// client: true
// Onclick: handleClientSide()
// script:
if (typeof window === "undefined") {
  handleServerSide();
}

function handleClientSide() {
  var dialog = new GlideModal('glide_confirm_standard');
  dialog.setTitle(getMessage('Confirmation'));
  dialog.setPreference('warning', true);
  dialog.setPreference('title', 'Are you sure?');
  dialog.setPreference('onPromptComplete', onConfirmed); // inline function will not work reliable here!
  dialog.render();

  function onConfirmed() {
    gsftSubmit(null, g_form.getFormElement(), '<Action name e.g. sysverb_update>');
  }
}

function handleServerSide() {
  current.update();
  action.setRedirectURL(current);
}
```

## UI-Page

HTML

```xml
<?xml version="1.0" encoding="utf-8" ?>
<j:jelly trim="false" xmlns:j="jelly:core" xmlns:g="glide" xmlns:j2="null" xmlns:g2="null">
	<g:ui_slushbucket name="slushBucket" />
	<div class="modal-footer">
    <span class="pull-right">
      <g:dialog_buttons_ok_cancel ok="return okClicked()" ok_id="ui_page_ok" cancel="return cancelClicked()" cancel_id="ui_page_cancel" />
		</span>
  </div>
</j:jelly>
```

Client script:

```javascript
function okClicked() {
  var sysIds = slushBucket.getValues(slushBucket.getRightSelect());
  if (!sysIds) {
    alert("Select something!");
  } else {
    var modal = GlideModal.get();
    var onSubmit = modal.getPreference("onSubmit");
    if (typeof onSubmit === "function") {
      onSubmit(sysIds);
    }
  }

  return false;
}

function cancelClicked() {
  // any function registered on onCancel shall handle the close-process
  // if no function is registered, the dialog will close
  var modal = GlideModal.get();
  var onCancel = modal.getPreference("onCancel");
  if (typeof onSubmit === "function") {
    onCancel();
  } else {
    GlideModal.get().destroy();
  }

  return false;
}

function initSlushBucket() {
  slushBucket.clear();
  slushBucket.addLeftChoice("sys_id", "value 1");

  document.getElementById("ui_page_ok").disabled = false;
  document.getElementById("ui_page_cancel").disabled = false;

  var gdw = GlideDialogWindow.get();
  var myParam = gdw.getPreference("my_param");

  var ajax = new GlideAjax("MyScriptInclude");
  ajax.addParam("sysparm_name", "myFunction");
  ajax.addParam("my_param", myParam);
  ajax.getXMLAnswer(function (answer) {
    var result = JSON.parse(answer);
    callback(result);
  });
}

addLoadEvent(function() {
  if (typeof g_form !== "undefined" && !g_form.modified) {
    console.warn("The Page has been opened from a dirty form. The changes will be discarded on publish!");
  }

  document.getElementById("ui_page_ok").disabled = disabled;
  document.getElementById("ui_page_cancel").disabled = disabled;
  initSlushBucket();
});
```

## Client Side Script Includes
Naming Convention by ServiceNow is AJAX. I personally dislike this as AJAX is a special technology 

```javascript
/* global global */
/* global Class */
/* eslint no-undef: "error" */
var MyScriptIncludeClient = Class.create();
MyScriptIncludeClient.prototype = Object.extendsObject(global.AbstractAjaxProcessor, {
  initialize: function(request, responseXML, gc) {
    global.AbstractAjaxProcessor.prototype.initialize.apply(this, arguments);
  },

  myFunc: function () {
    var taskGr = this._paramAsGlideRecord(task, "task", "task");
    info_str = this._paramAsString(info_str, "info_str");

    var parentGr = taskGr.parent.getRefRecord();
    return this._ret(parentGr);
  },

  type: "MyScriptInclude"
});
```
