# PicoCTF 2021

Ctf At: https://play.picoctf.org/ - PicoGym

## It is my Birthday

**Challenge Link**

http://mercury.picoctf.net:50970/
<br />

**Solution**

Basically what it wants you to do is to upload 2 different PDF files with the same MD5 Hash. There are different files out there that contain the same MD5 hash, just google them.  To upload them as a pdf, all you have to do is change the file extension.  It won't affect the MD5 hash, but it will be verified as a PDF as their PHP only checks file extensions.  
The files I used are uploaded as birthday1 and 2 in this repo.

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
 
 
