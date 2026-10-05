# Python SOCKS5 Proxy: requests, aiohttp and httpx setups that work, why `socks5://` leaks DNS, and what a proxy actually costs per GB

Most people searching for this have already hit one of two walls. Either their code throws `Missing dependencies for SOCKS support`, or their script "works" while quietly resolving every hostname from their own laptop. Both problems have short answers, and neither is fixed by copying a longer snippet.

Here's the two-line version first, then the parts that actually break.

bash
pip install "requests[socks]"


python
import requests

proxies = {
    "http": "socks5h://USER:PASS@gw.dataimpulse.com:824",
    "https": "socks5h://USER:PASS@gw.dataimpulse.com:824",
}
r = requests.get("https://httpbin.org/ip", proxies=proxies, timeout=30)
print(r.json())


That's the whole shape of it. Everything below is about the details that turn this from "runs on my machine" into "runs for six hours without supervision".

## SOCKS5 isn't something requests supports out of the box

`requests` hands SOCKS traffic to the `socks` extra, which is PySocks under the hood. Install `requests` alone and a `socks5://` URL fails with `Missing dependencies for SOCKS support` — not a connection error, an import error wearing a connection error's clothes.

Two install routes:

bash
pip install "requests[socks]"   # pulls in PySocks
pip install PySocks             # same thing, installed directly


For async work the package differs. `aiohttp` needs `aiohttp_socks`, and `httpx` needs its own extra:

bash
pip install aiohttp aiohttp_socks
pip install "httpx[socks]"


## `socks5://` vs `socks5h://`: the difference is who resolves the domain

This is the single most useful thing to understand about SOCKS5 in Python, and it's the reason half the tutorials out there are subtly wrong.

| Scheme | Where DNS resolution happens | Consequence |
| --- | --- | --- |
| `socks5://` | Your machine, locally | Your real resolver sees every hostname you request |
| `socks5h://` | The proxy server | Target hostnames never touch your local DNS |

If you're using a proxy so the destination site sees a different IP, using `socks5://` gives you half the job. The request egress looks fine, but `example.com` still got asked about by your ISP's resolver. Use `socks5h://` unless you have a specific reason not to — urllib3's own documentation makes the same recommendation, and it's why the scheme exists.

The practical tell: `curl --socks5-hostname` is the equivalent flag there, and `urllib3` exposes both schemes as `socks5h://` and `socks5://`.

## Credentials go inside the URL, and special characters need encoding

There's no separate auth parameter. Username and password are embedded in the proxy URL:

python
"http": "socks5h://myuser:mypassword@gw.dataimpulse.com:824"


If the password contains `@`, `:`, `#` or `/`, the URL parser will read it as structure instead of text. Encode first:

python
from urllib.parse import quote

user = quote("my@user")
pw = quote("p@ss:word")
url = f"socks5h://{user}:{pw}@gw.dataimpulse.com:824"


Hardcoding them is a bad habit, not a moral failing — env vars are usually enough:

python
import os
proxy = os.getenv("PROXY_URL")
proxies = {"http": proxy, "https": proxy} if proxy else None


`requests` also reads `HTTP_PROXY` / `HTTPS_PROXY` from the environment on its own. Pass `trust_env=False` if you want it to ignore them.

## Sessions, timeouts and retries: the boring parts that decide whether it survives

Three settings separate a demo from a scraper.

**Session reuse.** One `Session` object carries the proxy, headers, cookies and a connection pool across requests:

python
with requests.Session() as s:
    s.proxies = proxies
    s.headers.update({"User-Agent": "Mozilla/5.0"})
    for url in urls:
        s.get(url, timeout=(5, 20))


**Split timeouts.** `timeout=(5, 20)` means 5 seconds to connect, 20 to read. A single number tells you *something* took too long; a tuple tells you which layer to blame. Dead proxies hang rather than fail, so never leave `timeout` unset.

**Retries.** A SOCKS5 handshake can fail on a bad node and succeed one request later. Let `urllib3` handle it instead of writing a loop:

python
from requests.adapters import HTTPAdapter
from urllib3.util.retry import Retry

retry = Retry(total=3, backoff_factor=0.5, status_forcelist=[429, 502, 503])
s.mount("socks5h://", HTTPAdapter(max_retries=retry))


One thing worth knowing: session reuse does **not** mean the same egress IP. Whether the IP changes depends on the proxy provider's rotation rules, not on your client. That matters for the next section.

## Async: aiohttp and httpx

`aiohttp` doesn't accept a `proxies` dict at all. It takes a connector, and for SOCKS5 that connector comes from `aiohttp_socks`:

python
import asyncio
import aiohttp
from aiohttp_socks import ProxyConnector

async def fetch(url):
    connector = ProxyConnector.from_url("socks5://USER:PASS@gw.dataimpulse.com:824")
    async with aiohttp.ClientSession(connector=connector) as session:
        async with session.get(url, timeout=aiohttp.ClientTimeout(total=30)) as resp:
            return await resp.text()

print(asyncio.run(fetch("https://httpbin.org/ip")))


Note `ProxyConnector` sets remote DNS, so the `h` distinction is handled by the connector rather than the scheme string.

`httpx` changed its API recently — `proxies=` was deprecated in 0.26 and removed in 0.28, replaced by a singular `proxy=`:

python
import httpx

with httpx.Client(proxy="socks5://USER:PASS@gw.dataimpress.com:824") as client:
    print(client.get("https://httpbin.org/ip", timeout=30).text)


(Use your real host there. `httpx[socks]` must be installed or it fails the same way `requests` does.)

## Libraries that don't know about your proxy: PySocks monkeypatching

Some third-party packages build their own sockets and ignore `proxies=` entirely. The workaround is to replace the socket class globally:

python
import socks, socket

socks.set_default_proxy(
    socks.SOCKS5, "gw.dataimpulse.com", 824,
    username="USER", password="PASS",
)
socket.socket = socks.socksocket


Blunt, effective, and applies to every socket in the process. If a library still routes around it, a local HTTP bridge such as `tinyproxy` with an `upstream socks5` directive is the standard escape hatch — then you're back to a normal `http://` proxy URL and every library cooperates. Alternatively, `sshuttle` tunnels everything at the IP layer, no library configuration required.

## Rotating vs sticky: how a gateway handles it for you

Two connection models, and picking wrong shows up as login flows breaking halfway through.

**Rotating.** A new egress IP on each request. With DataImpulse that's a single endpoint — `gw.dataimpulse.com:823` for HTTP/HTTPS, port `824` for SOCKS5 — and the provider rotates server-side. No pool list to maintain, no `random.choice()` picking dead entries.

**Sticky.** The IP stays attached to a specific port (10000–20000 on DataImpulse) for 1 to 120 minutes, defaulting to 30 if you don't specify. Sticky sessions are the right choice when a site wants session continuity: logins, carts, multi-step forms.

Geo-targeting rides along in the username rather than a separate parameter:


USER__cr.us                          # country: US
USER__cr.us_session-abc123           # sticky session labelled abc123
USER__cr.us_city-newyork             # city targeting


For a Python scraper, that means one string constant change instead of a proxy-list rotation module.

👉 [Grab the $5 / 5 GB residential plan and test your SOCKS5 code](https://bit.ly/dataimPulse)

## Errors you'll actually see

| Symptom | Cause | Fix |
| --- | --- | --- |
| `Missing dependencies for SOCKS support` | PySocks not installed | `pip install "requests[socks]"` |
| Connects, but the destination logs your real resolver | Using `socks5://` | Switch to `socks5h://` |
| `407`/auth failure | Special characters unencoded in the URL | `urllib.parse.quote()` on user and password |
| Immediate connection refused | Wrong port — HTTP and SOCKS5 listen on different ones | SOCKS5 on 824, HTTP/HTTPS on 823 with DataImpulse |
| Script hangs with no exception | No timeout set | `timeout=(5, 20)` |
| Tight `SSL: CERTIFICATE_VERIFY_FAILED` loop | Tempting fix is `verify=False` | Don't — it hides MITM risk; fix the actual cause |

That last row gets ignored a lot. `verify=False` makes the error disappear without making anything better.

## What this costs, plan by plan

Provider choice for Python work comes down to per-GB rate, protocol support, and whether traffic expires. Subscription-only providers punish you when a scraping job finishes early and you've pre-paid for 100 GB.

DataImpulse's model is pay-as-you-go with no expiry — bought GBs stay in the account until consumed. Here's the full current lineup:

| Plan | What you get | Price | Billing | Buy |
| --- | --- | --- | --- | --- |
| Residential (Intro) | 5 GB, 90M+ IP pool, 195 countries, HTTP/HTTPS + SOCKS5, rotating + sticky, country targeting included | **$5** ($1/GB) | Pay-as-you-go, no subscription | [ 5 GB residential](https://bit.ly/dataimPulse) |
| Residential (volume) | 1 TB+ | **$800 / 1 TB** ($0.80/GB) | Pay-as-you-go | [ 1 TB residential](https://bit.ly/dataimPulse) |
| Datacenter (Intro) | 10 GB, 99.9% uptime, randomized subnet access | **$5** ($0.50/GB) | Pay-as-you-go | [ 10 GB datacenter](https://bit.ly/dataimPulse) |
| Datacenter (mid) | 100 GB | **$50** | Pay-as-you-go | [ 100 GB datacenter](https://bit.ly/dataimPulse) |
| Datacenter (bulk) | 1 TB | **$450** ($0.45/GB) | Pay-as-you-go | [ 1 TB datacenter](https://bit.ly/dataimPulse) |
| Datacenter (custom) | 5 TB+ | **from $2,250** | Custom | [ Talk to DataImpulse about 5 TB+ datacenter](https://bit.ly/dataimPulse) |
| Mobile (Intro) | 2.5 GB, 4G/5G/LTE IPs across 195 countries | **$5** ($2/GB) | Pay-as-you-go | [ 2.5 GB mobile](https://bit.ly/dataimPulse) |
| Mobile (mid) | 25 GB | **$50** | Pay-as-you-go | [ 25 GB mobile](https://bit.ly/dataimPulse) |
| Mobile (bulk) | 1 TB | **$1,600** ($1.60/GB) | Pay-as-you-go | [ 1 TB mobile](https://bit.ly/dataimPulse) |
| Mobile (custom) | 5 TB+ | **from $8,000** | Custom | [ Talk to DataImpulse about 5 TB+ mobile](https://bit.ly/dataimPulse) |
| Premium residential (Intro) | 1 GB, high-speed pool, dedicated account manager, all targeting included | **$5** ($5/GB) | Pay-as-you-go | [ 1 GB premium residential](https://bit.ly/dataimPulse) |
| Premium residential (mid) | 10 GB | **$50** | Pay-as-you-go | [ 10 GB premium residential](https://bit.ly/dataimPulse) |
| Premium residential (custom) | 5 TB+ | **from $20,000** | Custom | [ Talk to DataImpulse about premium residential at scale](https://bit.ly/dataimPulse) |

Some context for those numbers. Across the proxy market, residential traffic generally lands between roughly $1 and $8 per GB in 2026, datacenter between $0.50 and $3, and mobile between $2 and $15. Residential at $1/GB sits at the floor of that range, and datacenter at $0.50/GB is about as cheap as server IPs get without going to per-IP monthly billing.

For a Python script hitting defended targets, residential is the tier that matters. Datacenter IPs are fast and cheap but get categorized as hosting traffic on sight, which is fine for public endpoints and useless for SERPs or e-commerce.

👉 [Compare all four DataImpulse proxy types side by side](https://bit.ly/dataimPulse)

## Four details that change the bill

**Advanced targeting costs extra on standard residential.** Country selection and ASN exclusion are included. City, ZIP, state and specific ASN selection are billed at roughly double the standard per-GB rate on standard residential plans. If your script pins `_city-newyork` on every request, your effective cost isn't $1/GB. Datacenter plans list this targeting as included — worth confirming with support before you budget around it, since billing treatment has shifted before.

**There's no free trial.** The cheapest entry is $5, which buys 5 GB residential, 10 GB datacenter or 2.5 GB mobile. Reviews note that the minimum can be higher on later top-ups than on the first purchase, so read the checkout page rather than assuming.

**Refund terms are conditional.** New users on Intro plans get a 7-day money-back window on card payments, provided most of the traffic is still unused. Crypto purchases aren't refundable. That's a genuine limitation, not a formality.

**Non-expiring traffic is the real differentiator.** If you're testing a scraper and don't know whether the next job is 20 GB or 200 GB, the ability to leave unused bandwidth sitting in the account removes the guesswork that monthly subscriptions force on you.

On reputation: DataImpulse publishes a 99.51% success rate on its own site, which is a vendor figure and should be read as one. Third-party coverage is thinner than for the big enterprise names — TechRadar has a review that highlights the ethically sourced pool and the non-expiring traffic as the two things that stand out, and G2 shows a 4.8/5 average. For Python work specifically, the things that matter are the ones you can verify in ten minutes: SOCKS5 with authentication on port 824, `socks5h` behavior supported by the gateway, and rotating sessions that don't require you to maintain a proxy list.

## A working end-to-end pattern

Putting the pieces together — rotating IPs, sticky fallback for login steps, retries, split timeouts:

python
import requests
from requests.adapters import HTTPAdapter
from urllib3.util.retry import Retry

USER, PASS = "myuser", "mypass"
HOST, SOCKS_PORT = "gw.dataimpulse.com", 824

def proxy_url(country="us", session=None):
    login = f"{USER}__cr.{country}"
    if session:
        login += f"_session-{session}"
    return f"socks5h://{login}:{PASS}@{HOST}:{SOCKS_PORT}"

def build_session(session_id=None):
    s = requests.Session()
    url = proxy_url(session=session_id)
    s.proxies = {"http": url, "https": url}
    s.headers.update({"User-Agent": "Mozilla/5.0"})
    retry = Retry(total=3, backoff_factor=0.5, status_forcelist=[429, 502, 503])
    s.mount("https://", HTTPAdapter(max_retries=retry))
    return s

# rotating: fresh IP per request
with build_session() as s:
    print(s.get("https://httpbin.org/ip", timeout=(5, 20)).json())

# sticky: same IP for the whole flow
with build_session(session_id="checkout1") as s:
    s.get("https://example.com/login", timeout=(5, 20))
    s.post("https://example.com/cart", timeout=(5, 20))


Two sessions, one function, and the sticky one keeps the same egress IP across both calls because the session label is in the username. That's the pattern that holds up when a target site cares about IP continuity — and it's much less code than maintaining a proxy pool.

## Quick answers

**Does `requests` support SOCKS5 natively?** No. It needs PySocks via `requests[socks]`, `PySocks`, or the `socks` extra on urllib3.

**Should I use `socks5://` or `socks5h://`?** `socks5h://`, unless you specifically want local DNS resolution. The DNS leak from plain `socks5://` defeats much of the point of proxying.

**Can I rotate IPs from Python?** Yes, two ways. Maintain a list and pick with `random.choice()`, or point every request at one rotating gateway and let the provider handle it. The second approach means fewer failing nodes in rotation.

**Do I need SOCKS5 at all, or is HTTP fine?** HTTP/HTTPS proxies carry the same request-level information for most scraping work. SOCKS5 works at the transport layer, so it handles non-HTTP traffic and avoids a proxy-level protocol translation step. For pure `requests` scraping, the difference is smaller than tutorials imply — the bigger variables are IP quality and rotation.

**How much does this cost for a small project?** At $1/GB, 20 GB of residential traffic is $20. The usual mistake isn't the per-GB rate, it's paying for a monthly commitment in a month where the job ran twice.

👉 [Start with DataImpulse's $5 residential plan and measure your own cost per successful request](https://bit.ly/dataimPulse)
