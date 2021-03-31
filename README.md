# PicoCTF 2021

Ctf At: https://play.picoctf.org/ - PicoGym

## It is my Birthday

**Challenge Link**

http://mercury.picoctf.net:50970/
<br />

**Solution**

Basically what it wants you to do is to upload 2 different PDF files with the same MD5 Hash. There are different files out there that contain the same MD5 hash, just google them.  To upload them as a pdf, all you have to do is change the file extension.  It won't affect the MD5 hash, but it will be verified as a PDF as their PHP only checks file extensions.  
The files I used are uploaded as birthday1 and 2 in this repo.
<br />
## Super Serial

**Challenge Link**

http://mercury.picoctf.net:3449/
<br />

**Solution**

Immediately after opening the site, you see a login page.  This login page looks nearly identical to the login pages used for some of their other challenges, so I thought this was an sql injection type of thing first.  However, that isn't the case.  Instead, I looked around and found that the robots.txt file contained something interesting.

robots.txt: 
```
User-agent: *
Disallow: /admin.phps
```

This gave an interesting file, but after navigating to the link, you find that the file doesn't exist.  It also has a weird extension that I never heard of, `phps`.

A quick google search leads shows that phps files are just php files that display code that is color coded to make it more readable.  Using this information, I tried to go to the link `http://mercury.picoctf.net:3449/index.phps`, because index.php was a file, and boom!  It showed the source code for the website. 

After quickly reading skimming through the file, you can find that the login creates a `permissions` object that is serialized and then placed into the `login` object.  You can also see that there are 2 more php files that are readable using the phps extension, `cookie.phps` and `authentication.phps`.  

In cookie.phps you can see the permissions class, while in authentication.phps there is an access_log class.  However, in cookie.phps you can see something interesting.  The deserialize method is called, which in this case creates an object from the serialized data.  In theory this would take the cookie created during logging in and deserialize it to get the username and password.  However since we know that there is another class, the `access_log` class, we can insert a serialization that would cause the deserialize in cookie.phps to create an access_log class.  As for the parameters for the class, we can put the location of `../flag`.

After taking the code and messing around with it in some online php compiler (I used this one: https://paiza.io/en/projects/new), I found that the serialization of an access_log object with the "../flag" parameter was: 
```
O:10:"access_log":1:{s:8:"log_file";s:7:"../flag";}
```
Base64 encode it and stick it into the login cookie, and navigate to `authentication.php`, and you get your flag!
<br />
## Most Cookies

**Challenge Link**

http://mercury.picoctf.net:18835/
<br />

**Solution**

Like the other cookies problems, essentially what they want you to do is to create fake cookie that allows you to pretend to be an admin.  This time they gave you a server file, and you find that the cookie is made with flask.  The cookie is also signed using a random key chosen from an array the admins kindly wrote in the server.py file:
```
cookie_names = ["snickerdoodle", "chocolate chip", "oatmeal raisin", "gingersnap", "shortbread", "peanut butter", "whoopie pie", "sugar", "molasses", "kiss", "biscotti", "butter", "spritz", "snowball", "drop", "thumbprint", "pinwheel", "wafer", "macaroon", "fortune", "crinkle", "icebox", "gingerbread", "tassie", "lebkuchen", "macaron", "black and white", "white chocolate macadamia"]
```

A good tool you can use for this problem is flask-unsign.  This program allows us to decode and encode our own flask cookies.  Taking the cookie, you can run the command:
```
flask-unsign --decode --cookie [insert cookie]
```
to decode the cookie.  From doing this with our cookie value, we can find that the value our cookie contains is `{'very_auth':'blank'}`.  We can assume that the value needed for admin permissions is `{'very_auth':'admin'}.`

Flask-Unsign also has another interesting function.  It can guess the brute force the password used to encode the cookie given a list.  So what we do is we take the list of possible passwords from above, stick it & format it inside a text file, and run the following command:
```
flask-unsign --unsign --cookie [insert cookie] --wordlist [insert wordlist]
```

https://prnt.sc/110dxi3 - Example SS from me, you're cookie and key are likely to be different.

Afterwards, you can just run the following command to sign your own cookie that lets you in as admin:

 ```
 flask-unsign --sign --cookie "{'very_auth':'admin'}" --secret [Your Secret]
 Which in my case was:
 flask-unsign --sign --cookie "{'very_auth':'admin'}" --secret 'butter'
 ```
 
 Stick that value into the session cookie on the website, and reload to find the flag!
 <br />
 ## Web Gauntlet 2/3

**Challenge Link**

http://mercury.picoctf.net:35178/ - Web Gauntlet 2<br />
http://mercury.picoctf.net:32946/ - Web Gauntlet 3

<br />

**Solution**
In these two problems, the filters are the exact same. The only difference is the length limit, so the following solution will work for both problems.  If you can't go to filter.php after solving, try refreshing your cookies or doing it in incognito.

In both of these problems, we can find the filter at `/filter.php`.  In there we find that each of the following operators are filtered:

```
Filters: or and true false union like = > < ; -- /* */ admin
```
They happen to print the sql query when you try logging in with anything, so we find that the query is
```
SELECT username, password FROM users WHERE username='[insert stuff here]' AND password='[insert stuff here]'
```

There is also a character limit of 25 total characters (35 for Web Gauntlet 3) for username + password added together.  To solve this problem, we need to look at the sqlite operators that are not filtered, which are listed at: 
```
https://www.tutorialspoint.com/sqlite/sqlite_operators.htm
```
First things first, we need to find a way to get admin to not be filtered.  Fortunately they haven't banned `||` which is concatenates strings in sqlite.  We can get the string admin by just putting `adm'||'in`.

Next, we need to find a way to bypass the password checking.  We see that they haven't filtered any of the binary operators, which also happen to be 1 character long.  We can use these, especially the `| (binary or operator)` to bypass the password checking.

We can also use `is` and `is not` to replace `=` and `!=`.

From this I created the inputs:<br />
Username: `adm'||'in`<br />
Password: `' | '' IS '`<br />
Which would query: `SELECT username, password FROM users WHERE username='adm'||'in' AND password='' | '' IS ''`<br />

However for some reason this didn't seem to work.  I opened up an online sqlite compiler at `https://sqliteonline.com/` to do some more testing, and found that for some reason the | operator would return true if I put `'' IS NOT ''`.  So I replaced `IS` with `IS NOT` in the query, and it worked!

Final Input:<br />
Username: `adm'||'in`<br />
Password: `' | '' IS NOT '`<br />
Which would query: `SELECT username, password FROM users WHERE username='adm'||'in' AND password='' | '' IS NOT ''`<br />

Which happens to be exactly 25 characters long.
Navigate to filter.php to find the flag afterwards.
