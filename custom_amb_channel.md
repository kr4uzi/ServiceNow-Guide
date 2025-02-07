1. create ACLs for sys_amb_processor
2. create a new *JavaScript* sys_amb_processor for the desired channel e.g. /mytestchannel
3. subscribe to the channel:
   ```javascript
   (window.g_ambClient || window.top.g_ambClient)
     .getChannel('/mytestchannel')
     .subscribe(function(message) {
       console.info('My Custom Channel', message);
     });
   ```
4. Publish a message:
   ```javascript
   var msgGr = new GlideRecord('sys_amb_message');
   msgGr.newRecord();
   msgGr.serialized_cometd_message = JSON.stringify({ 'data': { 'content': 'hello world' }, 'channel': '/mytestchannel' });
   gs.info(msgGr.insert());
   ```
   The message will not insert if ill-formated.
