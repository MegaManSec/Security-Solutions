Defensive urllib against SSRF and DNS Rebinding
============================================

DNS rebinding is a TOCTOU vulnerability which abuses the fact that an initial check of the DNS records for a domain does not necessarily correspond to the DNS record which the request will actually use.
This is my small solution to creating a way to use urllib, and be safe from SSRF and DNS rebinding. More info [here](https://joshua.hu/solving-fixing-interesting-problems-python-dns-rebindind-requests).

The `is_local_address` function, which checks whether an IP address is local or not, is left as an exercize to the reader.

```python
import requests
import ssl
import sys
from urllib.parse import urlparse, urlunparse
from urllib3.poolmanager import PoolManager
from requests.adapters import HTTPAdapter

class HostHeaderSSLAdapter(HTTPAdapter):
    """Adapter that connects to an IP address but validates TLS for a different host."""
    def __init__(self, ssl_hostname, *args, **kwargs):
        self.ssl_hostname = ssl_hostname
        super().__init__(*args, **kwargs)

    def init_poolmanager(self, *args, **kwargs):
        context = ssl.create_default_context()
        kwargs['ssl_context'] = context
        kwargs['server_hostname'] = self.ssl_hostname  # This works for urllib3>=2
        self.poolmanager = PoolManager(*args, **kwargs)

url = "https://example.com/"

parsed_url = urlparse(url)
original_hostname = parsed_url.netloc
# Resolve the first IP address for the hostname
resolved_ip = resolve_first_ip(original_hostname)

# Check if the resolved address is local (or blocklisted)
if is_local_address(resolved_ip):
    sys.exit(1)

# Create a session and mount the custom adapter
session = requests.Session()
# Ensure https certificate is checked against cn=original_hostname
adapter = HostHeaderSSLAdapter(original_hostname)
session.mount("https://", adapter)

# Send request with proper Host header (original_hostname)
headers = {
    "Host": original_hostname
}

# Reconstruct netloc, replacing the hostname with its associated IP address
netloc = ""
if parsed_url.username and parsed_url.password:
    netloc = '{}:{}@'.format(parsed_url.username, parsed_url.password)
elif parsed_url.username:
    netloc = '{}@'.format(parsed_url.username)

netloc += resolved_ip

if parsed_url.port:
    netloc += ':{}'.format(parsed_url.port)

# Replace the netloc with the reconstructed netloc
url = parsed_url._replace(netloc=netloc)

# Do not follow redirects, as they may redirect to blocked addresses.
response = session.get(url, headers=headers, allow_redirects=False)

print("Status Code:", response.status_code)
print("Response Headers:", response.headers)
```
