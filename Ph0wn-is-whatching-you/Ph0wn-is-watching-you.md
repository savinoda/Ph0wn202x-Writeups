I created Ph0wn-is-watching-you for Ph0wn 2021: the write up follows the challenge flow as a normal player -and not the creator of the challenge- would do. Anyhow, I'll put some **admin comments** that reflect what I thought you should have noticed or done :)

The challenge is providing the following banner:

```
Ph0wn is watching you through its eyes but at the same time it hides a sweet secret: are you able to find it by exploring the non-explorable?

http://challenges.ph0wn.org:8002
```

If we open the page, we have an image that comes from a camera and on the left side 4 control buttons to move the camera

<img src="1.png" width="500">

**The image deployed for the challenge it's only a panorama taken with a smartphone. The infrastructure was supposed to acquire live images from a camera located in the room where Ph0wn should have taken place. But, unfortunately, we had to cancel the live event**

If we try to move the camera, we notice that we cannot move the camera to reach the leftmost part of the picture.
`http://challenges.ph0wn.org:8002/left`

<img src="2.png" width="500">

**Is this maybe the non-explorable area that the challenge is mentioning ?**

Let's check what happens when we perform an action that is not going left.
`http://challenges.ph0wn.org:8002/up`

<img src="3.png" width="500">

It looks like that the entity that replies to our request is `Ph0wn-1korn`.
Although a bit **(deliberately)** obfuscated, the entity behing the request is quite clear: 1korn ~ gunicorn

However, the WAF that blocks our requests when we speficy the `left` option is still unknown.
If we try to interact with it by providing crafted input (e.g., changing the HTTP version in the HTTP request with something that does not exist) we discover that the system is using MitmProxy 6.0.0 (thanks `tv0g` for providing it)

<img src="4.png" width="600">

**Now, you should already have identified the (possible) vulnerability but in case you have not, it will be clear if you simply google MitmProxy and gunicorn**

<img src="5.png" width="600">

The vulnerability is indeed the possibility of smuggling HTTP requests on the reverse proxy and reach the end point bypassing the filtering.
In this case, our goal is to bypass the restriction on the `left` movement.
The process of solving the challenge is the same as the one reported in this blogpost:
https://blog.deteact.com/gunicorn-http-request-smuggling/
Our goal is indeed to exploit the confusion between gunicorn and mitmproxy on the content-length and transfer-encoding headers.

The only difference is that we need to also attach a session cookie that is probably used by the backend to keep track of the camera position.
The requests to move left the camera are reported in the following picture (thanks `La Brosse Adam` for providing it):

<img src="6.png" width="700">



