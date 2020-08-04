Request hooks

Often when using a third party API you want to verify that the returned response is indeed valid. Requests offers the shorthand helper raise_for_status() which asserts that the response HTTP status code is not a 4xx or a 5xx, i.e that the request didn't result in a client or a server error.

For example

```
response = requests.get('https://api.github.com/user/repos?page=1')
# Assert that there were no errors
response.raise_for_status()
```

This can get repetitive if you need to raise_for_status() for each call. Luckily the requests library offers a 'hooks' interface where you can attach callbacks on certain parts of the request process.

We can use hooks to ensure raise_for_status() is called for each response object.

```
# Create a custom requests object, modifying the global module throws an error
http = requests.Session()

assert_status_hook = lambda response, *args, **kwargs: response.raise_for_status()
http.hooks["response"] = [assert_status_hook]

http.get("https://api.github.com/user/repos?page=1")
```

> HTTPError: 401 Client Error: Unauthorized for url: https://api.github.com/user/repos?page=1

Setting base URLs

Suppose you are only using one API hosted at api.org. You'll end up repeating the protocol and domain for every http call:

```
requests.get('https://api.org/list/')
requests.get('https://api.org/list/3/item')
```

You can save yourself some typing by using BaseUrlSession. This allows you to specify the base url for the HTTP client and to only specify the resource path at the time of the request.

```
from requests_toolbelt import sessions
http = sessions.BaseUrlSession(base_url="https://api.org")
http.get("/list")
http.get("/list/item")
```

Note that the requests toolbelt isn't included in the default requests installation, so you'll have to install it separately.
Setting default timeouts

The requests documentation recommends that you set timeouts on all production code. If you forget to set timeouts a misbehaving server may cause your application to hang, especially considering most Python code is synchronous.

```
requests.get('https://github.com/', timeout=0.001)
```

However this becomes repetitive and may cause future table flips when you realize someone has forgot to set a timeout and halted the program in production.

Using Transport Adapters we can set a default timeout for all HTTP calls. This ensures that a sensible timeout is set even if the developer forgets to add the timeout=1 parameter to his individual call, but allows for overrides on a per-call basis.

Below is an example of a custom Transport Adapter with default timeouts, inspired by this Github comment. We override the constructor to provide a default timeout when constructing the http client and the send() method to ensure that the default timeout is used if a timeout argument isn't provided.

```
from requests.adapters import HTTPAdapter

DEFAULT_TIMEOUT = 5 # seconds

class TimeoutHTTPAdapter(HTTPAdapter):
    def __init__(self, *args, **kwargs):
        self.timeout = DEFAULT_TIMEOUT
        if "timeout" in kwargs:
            self.timeout = kwargs["timeout"]
            del kwargs["timeout"]
        super().__init__(*args, **kwargs)

    def send(self, request, **kwargs):
        timeout = kwargs.get("timeout")
        if timeout is None:
            kwargs["timeout"] = self.timeout
        return super().send(request, **kwargs)
```


We can use it like so:

```
import requests

http = requests.Session()

# Mount it for both http and https usage
adapter = TimeoutHTTPAdapter(timeout=2.5)
http.mount("https://", adapter)
http.mount("http://", adapter)

# Use the default 2.5s timeout
response = http.get("https://api.twilio.com/")

# Override the timeout as usual for specific requests
response = http.get("https://api.twilio.com/", timeout=10)
```


Retry on failure

Network connections are lossy, congested and servers fail. If we want to build a truly robust program we need to account for failures and have a retry strategy.

Add a retry strategy to your HTTP client is straightforward. We create a HTTPAdapter and pass our strategy to the adapter.


```
from requests.adapters import HTTPAdapter
from requests.packages.urllib3.util.retry import Retry

retry_strategy = Retry(
    total=3,
    status_forcelist=[429, 500, 502, 503, 504],
    method_whitelist=["HEAD", "GET", "OPTIONS"]
)
adapter = HTTPAdapter(max_retries=retry_strategy)
http = requests.Session()
http.mount("https://", adapter)
http.mount("http://", adapter)

response = http.get("https://en.wikipedia.org/w/api.php")
```

The default Retry class offers sane defaults, but is highly configurable so here is a rundown of the most common parameters I use.

The parameters below include the default parameters the requests library uses.

```
total=10
```

The total number of retry attempts to make. If the number of failed requests or redirects exceeds this number the client will throw the urllib3.exceptions.MaxRetryError exception. I vary this parameter based on the API I'm working with, but I usually set it to lower than 10, usually 3 retries is enough.

```
status_forcelist=[413, 429, 503]
```

The HTTP response codes to retry on. You likely want to retry on the common server errors (500, 502, 503, 504) because servers and reverse proxies don't always adhere to the HTTP spec. Always retry on 429 rate limit exceeded because the urllib library should by default incrementally backoff on failed requests.

```
method_whitelist=["HEAD", "GET", "PUT", "DELETE", "OPTIONS", "TRACE"]
```

The HTTP methods to retry on. By default this includes all HTTP methods except POST because POST can result in a new insert. Modify this parameter to include POST because most API's I deal with don't return an error code and perform an insert in the same call. And if they do, you should probably issue a bug report.

backoff_factor=0

This is an interesting one. It allows you to change how long the processes will sleep between failed requests. The algorithm is as follows:

{backoff factor} * (2 ** ({number of total retries} - 1))

For example, if the backoff factor is set to:

    1 second the successive sleeps will be 0.5, 1, 2, 4, 8, 16, 32, 64, 128, 256.
    2 seconds - 1, 2, 4, 8, 16, 32, 64, 128, 256, 512
    10 seconds - 5, 10, 20, 40, 80, 160, 320, 640, 1280, 2560

The value is exponentially increasing which is a sane default implementation for retry strategies.

This value is by default 0, meaning no exponential backoff will be set and retries will immediately execute. Make sure to set this to 1 in to avoid hammering your servers!.

The full documentation on the retry module is here.
Combining timeouts and retries

Since the HTTPAdapter is comparable we can combine retries and timeouts like so:

```
retries = Retry(total=3, backoff_factor=1, status_forcelist=[429, 500, 502, 503, 504])
http.mount("https://", TimeoutHTTPAdapter(max_retries=retries))
```

Debugging HTTP requests

Sometimes requests fail and you can't figure out why. Logging the request and response might give you insight to the failure. There are two ways to do this - either by using the built in debug logging settings or by using request hooks.
Printing HTTP headers

Changing the logging debug level greater than 0 will log the response HTTP headers. This is the simplest option, but it doesn't allow you to see the HTTP request or the response body. It's useful if you're dealing with an API that returns a large body payload that is not suitable for logging or contains binary content.

Any value that is greater than 0 will enable debug logging.

```
import requests
import http

http.client.HTTPConnection.debuglevel = 1

requests.get("https://www.google.com/")
```

# Output
send: b'GET / HTTP/1.1\r\nHost: www.google.com\r\nUser-Agent: python-requests/2.22.0\r\nAccept-Encoding: gzip, deflate\r\nAccept: */*\r\nConnection: keep-alive\r\n\r\n'
reply: 'HTTP/1.1 200 OK\r\n'
header: Date: Fri, 28 Feb 2020 12:13:26 GMT
header: Expires: -1
header: Cache-Control: private, max-age=0

Printing everything

If you want to log the entire HTTP lifecycle, including both the textual representation of the request and response you can use request hooks and the dump utils from requests_toolbelt.

I prefer this option any time I'm dealing with a REST based API that doesn't return very large responses.

```
import requests
from requests_toolbelt.utils import dump

def logging_hook(response, *args, **kwargs):
    data = dump.dump_all(response)
    print(data.decode('utf-8'))

http = requests.Session()
http.hooks["response"] = [logging_hook]

http.get("https://api.openaq.org/v1/cities", params={"country": "BA"})
```

# Output
< GET /v1/cities?country=BA HTTP/1.1
< Host: api.openaq.org

> HTTP/1.1 200 OK
> Content-Type: application/json; charset=utf-8
> Transfer-Encoding: chunked
> Connection: keep-alive
>
{
   "meta":{
      "name":"openaq-api",
      "license":"CC BY 4.0",
      "website":"https://docs.openaq.org/",
      "page":1,
      "limit":100,
      "found":1
   },
   "results":[
      {
         "country":"BA",
         "name":"Goražde",
         "city":"Goražde",
         "count":70797,
         "locations":1
      }
   ]
}

See https://toolbelt.readthedocs.io/en/latest/dumputils.html
Testing and mocking requests

Using third-party API's introduce a pain point in development - they're difficult to unit test. The engineers at Sentry have alleviated some of this pain by writing a library to mock requests during development.

Instead of sending the HTTP response to the server getsentry/responses intercepts the HTTP request and returns a pre-defined response you've added during tests.

It's better demonstrated with an example.

```
import unittest
import requests
import responses


class TestAPI(unittest.TestCase):
    @responses.activate  # intercept HTTP calls within this method
    def test_simple(self):
        response_data = {
                "id": "ch_1GH8so2eZvKYlo2CSMeAfRqt",
                "object": "charge",
                "customer": {"id": "cu_1GGwoc2eZvKYlo2CL2m31GRn", "object": "customer"},
            }
        # mock the Stripe API
        responses.add(
            responses.GET,
            "https://api.stripe.com/v1/charges",
            json=response_data,
        )

        response = requests.get("https://api.stripe.com/v1/charges")
        self.assertEqual(response.json(), response_data)
```


If a HTTP request that doesn't match the mocked responses is made, a ConnectionError is thrown.

```
class TestAPI(unittest.TestCase):
    @responses.activate
    def test_simple(self):
        responses.add(responses.GET, "https://api.stripe.com/v1/charges")
        response = requests.get("https://invalid-request.com")
```

Output

requests.exceptions.ConnectionError: Connection refused by Responses - the call doesn't match any registered mock.

Request:
- GET https://invalid-request.com/

Available matches:
- GET https://api.stripe.com/v1/charges

Mimicking browser behaviors

If you've written enough web scraper code, you'll notice how certain websites return different HTML depending on if you're using a browser or accessing the site programmatically. Sometimes this is an anti-scraping measure, but usually servers engage in User-Agent sniffing to find out what content best fits the device (e.g desktop or mobile).

If you want to return the same content as the browser displays you can override the User-Agent header requests sets with something Firefox or Chrome would send.

```
import requests
http = requests.Session()
http.headers.update({
    "User-Agent": "Mozilla/5.0 (X11; Ubuntu; Linux x86_64; rv:68.0) Gecko/20100101 Firefox/68.0"
})
```
