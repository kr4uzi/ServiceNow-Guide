The following example code implements a record watcher in the classic UI which detects attachment upload/modify/delete on a given record.
It is intended to be added to a UI Macro, but and also be adapted to work "onLoad" (just take the callback of the addLoadEvent).
```javascript
(function () {
   addLoadEvent(function () {		
			var watchedTable = 'sys_attachment';
			var watchedFilter = 'table_sys_id=' + g_form.getUniqueValue();
			var watchedFields = ['file_name'];
			var watcherChannel = getChannelRW(watchedTable, watchedFilter);
			watcherChannel.subscribe(function(message) {
				if (message.data.operation == 'update') {
					var changedFields = message.data.changes.filter(function (f) {
						return watchedFields.indexOf(f) != -1;
					});

					if (changedFields.length) {
						// file_name of one of an attachment changed
					}
				} else if (message.data.operation == 'insert') {
          // new attachment uploaded
				} else if (message.data.operation == 'delete') {
					// attachment deleted
				}
			});
		});

		// the following functions are taken/inspired by the 'amb' service ('ng.amb' module)
		// source code: /scripts/app.ng.amb/service.AMB.js
		function getFilterString(filter) {
			filter = filter.
				replace(/\^EQ/g, '').
				replace(/\^ORDERBY(?:DESC)?[^^]*/g, '').
				replace(/^GOTO/, '');
				return btoa(unescape(encodeURIComponent(filter))).replace(/=/g, '-');
		}

		function getChannelRW(table, filter) {
			var ambClient = window.g_ambClient || window.top.g_ambClient;
			var t = '/rw/default/' + table + '/' + getFilterString(filter);
			return ambClient.getChannel(t);
		}
})();
```
