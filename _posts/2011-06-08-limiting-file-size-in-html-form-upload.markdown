---
layout: post
title: "Limiting file size in HTML form uploads"
author: claudiob
excerpt: "When users are allowed to upload files through an HTML form, they should get an immediate notification if the file is too large. Here is a Javascript snippet to achieve this result."
---

When users are allowed to upload files through an HTML form, they should get an immediate notification if the file is too large. Here is a Javascript snippet to achieve this result.

## Checking input file size in the front end

{% highlight javascript %}
$("#some_form").submit(function (e) {
	e.preventDefault();

	if ($.browser.msie) {
		var myFSO = new ActiveXObject("Scripting.FileSystemObject");
		var filepath = document.some_form.some_file.value;
		var thefile = myFSO.getFile(filepath);
		var fs = thefile.size;
	} else {
		var f = $("#some_file");
		var fs = f[0].files[0].size;
	}
	
	alert(fs +" bytes");
});
{% endhighlight %}

To see this code running, paste it in an HTML file together with a `<form id="some_form">` object, and include an `<input type="file" id="some_file">` field. Submitting the form will result in an alert message, informing of the file size in bytes.
  
Change the `alert` line at the bottom of the javascript for more interesting results, such as disabling the submit button if the file is too large.

The convenience of checking file size in the client is that **no portion of the file is actually transferred to the server** if the file is too large.

## That's only meant for usability

Never trust front-end only when dealing with form validation. You should still make sure the server will not accept very large files, in order to prevent malicious users from filling up the hard disk.



For security, add server-side validation both in the web framework settings and in the web server settings. 
At RED Interactive we mainly deal with Django projects served through nginx, so we always:

* add a `clean` method that will raise a `forms.ValidationError` if the file is too large, as explained [in this Django snippet](http://djangosnippets.org/snippets/1303/), and
* set up an appropriate `client_max_body_size` setting in the `nginx.conf` file.
