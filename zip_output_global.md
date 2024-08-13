no know way to do this scoped though...
```javascript
var os = new Packages.java.io.ByteArrayOutputStream();
var zip = new Packages.java.util.zip.ZipOutputStream(os);

// example with text only file content
var filePath = 'this/can/be/a/path.txt';
var fileContent = 'hello world\nform a file';
zip.putNextEntry(new Packages.java.util.zip.ZipEntry(filePath));
try {
	var bytes = new Packages.java.lang.String(fileContent).getBytes();
	zip.write(bytes, 0, bytes.length);
} finally {
	zip.closeEntry();
}

// example with attachment
var sourceAttSysID = '';
var attGr = new GlideRecord('sys_attachment');
if (attGr.get(sourceAttSysID)) {
  var attOS = new Packages.java.io.ByteArrayOutputStream();
  new GlideSysAttachmentInputStream(sourceAttSysID).writeTo(attOS);
  zip.putNextEntry(new Packages.java.util.zip.ZipEntry(attGr.file_name));
	try {
		var bytes = attOS.toByteArray();
		zip.write(bytes, 0, bytes.length);
	} finally {
		zip.closeEntry();
	}
}

zip.close();

var fakeGr = new GlideRecord('sc_cart_item');
fakeGr.setNewGuidValue(gs.generateGUID());
var attSysID = new GlideSysAttachment().write(fakeGr, 'my_zip_file.zip', 'application/zip', os.toByteArray());
gs.info('attSysID: ' + attSysID);
```
