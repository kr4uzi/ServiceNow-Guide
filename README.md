# Krauzi's Fails - A Guide to ServiceNow

In this reference we want to share my experience with the ServiceNow - Platform and ease the adoption of Best - Practice and ServiceNow specific clean code.

## How to g_form in your console
Service Portal UI: var g_form = angular.element(document).find('div.form-group[field*=formModel]').first().scope().getGlideForm()
Standard UI: Just select the correct frame and g_form is available right away
<img width="360" alt="image" src="https://user-images.githubusercontent.com/5402583/228592003-7e526123-c651-45e0-adf1-8a689e75ac6e.png">


## Transforming Filters into GlideRecord - Queries

I want to present you a few techniques on how to process data from a specific table.You probably want to process only certain records and your starting point is the ServiceDesk with the table opened(Tipp: Enter<table_name>.list or<table_name>.LIST in the Application Navigator's search bar to jump directly to the table's list).
Update: I've come to the conclusion, that using addActiveQuery shouldn't be used - it doesn't handle custom global fields ("u_active" - yeah you probably shouldn't be doing unscoped fields anyways) and it only encourages inconsistant coding.

### Simple Query

<img src = "https://github.com/kr4uzi/ServiceNow-Guide/blob/master/image-20191125142745663.png?raw=true" alt = "image-20191125143119776" style = "zoom:50%;" />

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

<img src = "https://github.com/kr4uzi/ServiceNow-Guide/blob/master/image-20191125160745911.png" alt = "image-20191125160745911" style = "zoom:50%;" />

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

** TL; DR **: if the query is static, use addEncodedQuery, otherwise use addQuery

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
  var gr = new GlideRecord("task");
  gr.setLimit(1);
  gr.query();
  gr.next(); // lets assume there is at least one record in the task table
  return !!gr.getValue("active"); // !! makes bool cast
}
```

## setLimit

```javascript
function emailIncidentExists() {
  var incidentGr = new GlideRecord("incident");
  incidentGr.addActiveQuery();
  incidentGr.addQuery("description", "CONTAINS", "email");
  incidentGr.setLimit(1);
  incidentGr.query();
  if (incidentGr.hasNext()) {
    return true;
  }
  
  return false;
}
```

By using "hasNext()" instead of "next()" in Line 7 we avoid loading the actual values of record into incidentGr and thus saving a view milliseconds.If you encounter queries of this type, form a habit of structuring your functions like the example code.

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

GlideRecord:: getRowCount() can have negative performance impact on the database and should therefore not be used in production.Use GlideAggregate instead.

## getValue

```javascript
function emailIncidentUsers() {
  var incidentGr = new GlideRecord("incident");
  incidentGr.addEncodedQuery("active=true^description=email");
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

When you want to retrieve the sys_id of a reference field, you should use getValue(field_name) instead of<field_name>.sys_id.toString().When using the latter, you will essentially load the whole record of<field_name>.
Be carefull however for there are the following things to bear in mind when using getValue:

-  getValue cannot be called on a GlideElement in scoped applications(`incidentGr.opened_by.getValue("company")` doesnt work)
- getValue will return `null` when a certain reference field is not set(use`getValue("field") || ""`, see code below)
- <glideRecord>.<glideElement>.toString() will always return a string even if the reference represented by <glideElement> is not set
- only use getValue when you are absolutely certain that the field retrieving can never be empty

```typescript
function getOpenedBy(incidentSysId) {
  var gr = new GlideRecord("incident");
  if (gr.get(incidentSysId)) {
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

## Business Rules

Do not dot Walk in Conditions, instead use: javascript: current.xyz

## UI Policies

Dot walking not working properly when used on script fields.

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
  dialog.setTitle(new GwtMessage().getMessage('Confirmation'));
  dialog.setPreference('warning', true);
  dialog.setPreference('title', 'Are you sure?');
  dialog.setPreference('onPromptComplete', onConfirmed); // inline function will not work reliable here!
  dialog.render();

  function onConfirmed() {
    gsftSubmit(null, g_form.getFormElement(), '<scope>_<action_identifier>');
  }
}

function handleServerSide() {
  current.update();
  action.setRedirectURL(current);
  // action.setRedirectUrl("task.do?" + current.sys_id);
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

## Global Client Side Script Include
The AbstractAjaxProcessor::getParameter function returns a Java String object. This can be problematic when using JavaScript functions on it. For example if you use .split on the value returned by getParameter, you will get an array of Java Strings. If you use .indexOf or .filter on this array, you will most likely not get anything useful from it as those functions than compare a Java String with a JavaScript String which will return false. Be sure to cast the value:
```javascript
var myParam = String(this.getParameter('my_param') || '');
myParam = myParam ? myParam.split(',') || []
```

## Unified Script Include (Client + Server-Side)

```javascript
var MyScriptInclude = Class.create();
MyScriptInclude.MY_ATTR = "hello world";
MyScriptInclude.prototype = Object.extendsObject(global.AbstractAjaxProcessor, {
  initialize: function(request, responseXML, gc) {
    global.AbstractAjaxProcessor.prototype.initialize.call(this, request, responseXML, gc);
    this.log = new global.GSLog("<sys_properties::log_level", this.type);

    if (typeof request === "object" && typeof request.type === "string" && request.type.startsWith("org.apache.catalina.connector.RequestFacade")) {
      this.log.info("Client Side");
    } else {
      this.log.info("Server Side");
    }
  },

  myFunc: function (task, info_str) {
    var taskGr = this._paramAsGlideRecord(task, "task", "task");
    info_str = this._paramAsString(info_str, "info_str");

    var parentGr = taskGr.parent.getRefRecord();
    return this._ret(parentGr);
  },

  _ret: function (val) {
    if (this.request) {
      if (typeof val === "string"
          || typeof val === "number"
          || typeof val === "boolean") {
        return val;
      }

      if (val instanceof GlideRecord) {
        var res = {};
        for (var attr in val) {
            res[attr] = val.getValue(attr) || "";
        }

        return JSON.stringify(res);
      } else if (val instanceof GlideElement) {
        return val.toString();
      }

      return JSON.stringify(val);
    }

    return val;
  },

  _paramAsGlideRecord: function (param, ajaxParam, tableName, defaultValue) {
    if (this.request) {
      var sysId = this.getParameter(ajaxParam);
      if (sysId === undefined) {
        if (defaultValue) {
          return defaultValue;
        }

        return new GlideRecord(tableName);
      }

      var clientGr = new GlideRecord(tableName);
      if (sysId == "-1") {
            clientGr.newRecord();
        return clientGr;
      } else if (sysId && clientGr.get(sysId)) {
        return clientGr;
      }

      this.log.error("invalid sys_id for " + tableName + " (" + sysId + ")");
      return defaultValue || clientGr;
    }

    if (param instanceof GlideRecord) {
      // TODO: check if param is in hirarchy of tableName
      return param;
    }

    var serverGr = new GlideRecord(tableName);
    if (param == "-1") {
      serverGr.newRecord();
      return serverGr;
    } else if (param && serverGr.get(param)) {
      return serverGr;
    }

    this.log.error("invalid sys_id for " + tableName + " (" + param + ")");
    return defaultValue || serverGr;
  },

  _paramAsString: function (param, ajaxParam, defaultValue) {
    if (this.request) {
      var str = this.getParameter(ajaxParam);
      if (str === undefined) {
        if (defaultValue) {
          return defaultValue;
        }

        return "";
      }

      return str;
    }

    if (typeof param === "string") {
      return param;
    } else if (param instanceof GlideRecord) {
      return param.isNewRecord() ? "-1" : param.getUniqueValue();
    } else if (param instanceof GlideElement) {
      return param.toString();
    } else if (param) {
      return String(param);
    } else if (defaultValue !== undefined) {
      return defaultValue;
    }

    return "";
  },

  type: "MyScriptInclude"
});
```

## Improved OOTB Logger
```javascript
var Logger = Class.create();
Logger.prototype = Object.extendsObject(global.GSLog, {
  initialize: function(traceProperty, caller) { global.GSLog.prototype.initialize.call(this, traceProperty, caller); },
  trace: function(){ global.GSLog.prototype.trace.call(this, this._stringify.apply(this, arguments)); },
  debug: function(){ global.GSLog.prototype.debug.call(this, this._stringify.apply(this, arguments)); },
  info: function (){ global.GSLog.prototype.info.call(this, this._stringify.apply(this, arguments)); },
  warn: function(){ global.GSLog.prototype.warn.call(this, this._stringify.apply(this, arguments)); },
  error: function(){ global.GSLog.prototype.error.call(this, this._stringify.apply(this, arguments)); },
  fatal: function(){ global.GSLog.prototype.fatal.call(this, this._stringify.apply(this, arguments)); },

  _stringify: function () {
    var msg = "";
    for (var i = 0; i < arguments.length; ++i) {
      msg += this._stringifyOne(arguments[i]);
    }
    
    return msg;
  },

  _stringifyOne: function (item) {
    if (item === undefined) {
      return;
    }
    
    if (item instanceof GlideRecord) {
      return item.isNewRecord() ? "-1" : item.getUniqueValue();
    } else if (item instanceof GlideElement) {
      return item.toString();
    }
    
    return JSON.stringify(item);
  },

  type: 'Logger'
});
```

# Hall of Shame
## ||

```javascript
function getIncidentDescription(incidentSysId) {
  var incidentGr = new GlideRecord("incident");
  if (incidentGr.get(incidentSysId)) {
		return incidentGr.description || incidentGr.short_description;
  }

  return "";
}
```

The requirement was: Return the incident description if set, or the short_description (mandatory) if the description was not set. The statement in line 4 is a javascript-short for `incidentGr.description ? incidentGr.description : incidentGr.short_description`. However the code will always return the description text. Why is that?
The answer is that ServiceNow javascript interpreter handles such statements in a very odd way: if evalutated within an `if`-context, the operands are "Boolean" casted which forces the internal value to be evaluated and cast to Boolean.
In this case we do not have an if-context and this doesnt apply. What gets evaluated instead looks like this: `return {} || {}`. Recall that the returned value from an attribute lookup is not the value of the column, but a GlideElement that represents the value . Otherwise dot-walking would not be possible, in this case however its very inconvenient.
The corrected version of this code is by the way:
`return incidentGr.description.toString() || incidentGr.short_description.toString()`

## get

```javascript
function closeIncident(titleString) {
  var incidentGr = new GlideRecord("incident");
  if (incidentGr.get("title", titleString)) {
    incidentGr.state = 3;
    incidentGr.update();
  }
}
```

The field `title` (the label for the short_description column is "Title") does not exist, thus the query renders valid and `getIncidentGr` returns a GlideRecord for which our assumption `incidentGr.short_description == titleString` is false.

## GlideElement

```javascript
function getGroupMembers(groupSysIdString) {
  var memberGr = new GlideRecord("sys_user_grmember");
  memberGr.addQuery("group", groupSysIdString);
  memberGr.query();

  var members = [];
  while (memberGr.next()) {
    members.push(memberGr.user);
  }

  return members;
}
```

expected result: `members.every(member => typeof member === "string") === true`
actual result: `members.every(member => member instanceof GlideElement) === true`

When retrieving a property ("dot walking") like sys_id on a GlideRecord (`gr.sys_id`), the value returned is not a String nor is it a GUID, but a GlideElement. In line 8 only GlideElements are appended to the Array. If that wasn't bad enough, they all represent the user column for current GlideRecord, and when the loop is done, we have esentially pushed getRowCount-times the same GlideElement of the last record.
Best practice for the code above would be to use `memberGr.getValue("user")` in line 8.
If we had used `memberGr.user.sys_id.toString()` instead, we would produce the same result, but to a higher performance cost because ServiceNow has to first load the whole user record and then return the sys_id for this record (which is already stored on memberGr - this is why we can use getValue to retrieve it).
Sadly, there is a draw back to this performance increase: getValue will return `null`in case no reference is set (see: Hall of Shame:||).

## is same (direction of dot-walking)

TODO: check if the following query can be constructed via "addJoinQuery".

When dot-walking in filter-queries (and thus when using them in "addEncodedQuery"), you will encounter a bug (present on at least London) that happens when you are not careful enough:

```javascript
function getDepartmentHeads() {
	var sysIds = [];
  var gr = new GlideRecord("sys_user");
  gr.addEncodedQuery("department.dept_head.sys_idSAMEASsys_id"); // good
  // gr.addEncodedQuery("sys_idSAMEASdepartment.dept_head.sys_id"); // bad
  gr.query();
  while (gr.next()) {
    sysIds.push(gr.getValue("sys_id"));
  }

  return sysIds;
}
```

## Workflow self destruction

Any WF : add 2 new actions

-> script

```typescript
var gr = new GlideRecord(current.getTableName());
gr.get(current.sys_id);
gr.state = 20;
gr.update();
```

-> wait for condition (state == 20)

-> will probably wait indefenitely

## Business Rule

do not set current.parent in onBefore business rules because this will make a potentially running workflow think that there is a parent<->child relationship between the current and current.parent object.
when the workflow of current runs it will also look into current.parent and use that workflow resulting in current running the workflow of current.parent.

# Dates

Date Time in current users TimeZone: gs.nowDateTime()

Date Time in system TimeZone: new GlideDateTime()

## GlideDate

```typescript
class GlideDate {
  constructor(); // constructs the object set to the current date
  setValue(value: string): void; // value shall have format yyyy-MM-dd

  getByFormat(format: string): string; // e.g. format = dd.MM.yyyy
  getDisplayValue(): string; // like getByFormat but for the current users format/timezone

  static substract(start: GlideDate, end: GlideDate): GlideDuration;
}
```

## GlideDateTime

```typescript
class GlideDateTime {
  constructor(); // constructs the object set to the current date and time (GMT)
  constructor(str: string); // use format "yyyy-MM-dd hh:mm:ss" (UTC)
  constructor(obj: GlideDateTime); // copy constructor

  add(date: GlideDateTime): void;
  add(milliseconds: Number): void;
  setValue(str: string): void; // format yyyy-MM-dd HH:mm:ss
  setValue(obj: GlideDateTime): void; // same as copy constructor
  setValue(milliseconds: number): void; // milliseconds passed since 1.1.1970

  getDate(): GlideDate;
  getTime(): GlideTime;

  getDisplayTime(): string; // Date time in the current users timezone
}
```

## GlideTime

```typescript
class GlideTime {

}
```

## GlideCalendarDateTime

```typescript
class GlideCalendarDateTime {
  constructor(); // current date and time in GMT format
  constructor(obj: GlideCalendarDateTime); // copy constructor
  constructor(str: string); // "yyyy-MM-dd hh:mm:ss"

  add(time: GlideTime): void;
  add(milliseconds: number): void;

  addSeoncds(seconds: number): void;
  addDaysLocalTime(days: number): void;
  addDaysUTC(days: number): void; // performs the calculation in UTC datetime values
  addWeeksLocalTime(weeks: number): void;
  addWeeksUTC(weeks: number): void;
  addMonthsLocalTime(months: number): void;
  addMonthsUTC(months: number): void; // performs the calculation in UTC datetime values
  addYearsLocalTime(years: number): void;
  addYearsUTC(years: number): void;

  // -1 (this is smaller/before), 0 (this is equal), 1 (this is greater/after)
  compareTo(dateTime: GlideCalendarDateTime): number;
  equals(dateTime: GlideCalendarDateTime): boolean;

  getDate(): GlideDate; // yyyy-MM-dd UTC
  getLocalDate(): GlideDate; // yyyy-MM-dd in current users time zone
  getTime(): GlideTime; // unix timestamp GMT
  getLocalTime(): GlideTime; // time in current users time zone

  getDisplayValue(): string; // date and time in current users format and time zone
  getDisplayValueInternal(): string; // yyyy-MM-dd HH:mm:ss

  isValid(): boolean;
}
```

# Best of ServiceNow official
## Creating new Records

```javascript
varname= source.u_name;
if(sgr.next()){
sid = sgr.sys_id;}else{// create it
sgr.initialize();
sgr.newRecord();
sgr.name=name;
sgr.version= version;
sid = sgr.insert();
```

Notice: The missing indentation
Notice: The use of both .initialize and newRecord
Notice: typeof sid === "string" and typeof sid === "GlideElement" both possible
Notice: use of .sys_id without toString() or missing usage of getValue
Notice: the use of =
Notice: varname = ...: this is technically valid (nonstrict) java code, line (6) will throw anyways

# Translations

If you are used to classic translation techniques like resource files you will probably feel kind of lost in ServiceNow. This guide should help you get all the translations done quickly and maybe even automate it to some degree.

## Modules

sys_translated:

Element = title

Table = sys_app_module

value = <module name>

label = <translation>

langugage = <lang>

## Dashbaords

sys_translated:

Label = <translated text>

element = name

Table = pa_dashboards

value = <dasboard name>

language = <lang>

## Reports

By default translation of report titles is not supported. You have to change sys_report's title field to a translated field in order to get the titles translated.

Go to System Definition > Dictionary, filter by table=sys_report and column name=title and change the Field Type to "Translated Text" and Save.

https://hi.service-now.com/kb_view.do?sysparm_article=KB0726995

## Translated Fields (common)

sys_translated:

Element = <field name>

Table=<table name>

Value=<Initial Value>

Label=<translated value>

## Client-Side-Scripts

```javascript
new GwtMessage().getMessage("<message_id>")
or: getMessage("<message_id>")
```

sys_ui_message.list:

Key = <message_id>

Language =....

Message = <translation of message_id>

## List Choices

Do not forget to "add to application file"

## Form Sections

sys_translated.list -> new:

Label= <translation>

element = caption

table = sys_ui_section

value = section name

language = de/en

section names can be modified in sys_ui_section

## Section Annotation

(layout form -> line seperator -> annotation)

sys_ui_message

automatically translated if sys_ui_message exists

you can also use

```javascript
${gs.getMessage("a message")} ... will get concatenated
```

## Relationships / Related Lists

1.) Relationship-Name -> "Rightclick + Configure on the respective Form -> List Control -> Label = Relationship-Name"
2.) sys_translated: value = relationship-name; language = de/en/...; table = sys_ui_list_control; element=label; label=<translation of relationship-name>


## Client-Side
# Import - Inbound Integrations

// https://docs.servicenow.com/bundle/newyork-platform-administration/page/script/server-scripting/reference/r_MapWithTransformationEventScripts.html

```typescript
function onStart(source: GlideRecord, import_set: GlideRecord, map: GlideTransformMap, log: function, ignore: Boolean, error: Boolean);
function onComplete(source: GlideRecord, target: GlideRecord, import_set: GlideRecord, map: GlideTransformMap, log: function, error: Boolean);
function onBefore() {
  // typeof source === "GlideRecord"
  // typeof target === "GlideRecord"
  // typeof import_set === "GlideRecord"
  // typeof map === "GlideTransformMap"
  // typeof log === "function" -> log.warn, log.info, log.error
  // typeof action === "string" -> "insert" or "update"
  // typeof ignore === Boolean -> when set to true, no record will be inserted and other transform scripts will not run (except onAfter which will still be called)
  // typeof status_string === "string" -> custom message for the XML response (??)
  // typeof error === "Boolean" -> when set to true, will halt the entire transformation of the import set with an error message

}
```
## Field Types

| Field Type         | Dictionary XML type | JS-Class              | (MySQL) DB type |
| ------------------ | ------------------- | --------------------- | --------------- |
| Date               | glide_date          | GlideDate             | DATE            |
| Date/Time          | glide_date_time     | GlideDateTime         | DATETIME        |
| Time               | glide_time          | GlideTime             | DATETIME        |
| Duration           | glide_duration      | GlideDuration         | DATETIME        |
| Due Date           | due_date            |                       | DATETIME        |
| Calendar Date/Time |                     | GlideCalendarDateTime |                 |
|                    |                     |                       |                 |

Import Date Fields -> Mit Field Maps werden Datums-Werte automatisch als setValue() (??? jedenfalls als GMT gesetzt -> falls display value oder localTime gewünscht ist muss man das per onBefore script machen)
