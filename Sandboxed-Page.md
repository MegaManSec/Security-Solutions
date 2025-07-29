```html
<script>
  window.addEventListener("message", function(event) {
    // Only allow from *.example.com. Can set some HTTP headers to ensure similar things, I suppose.
    const trustedOriginPattern = /^https:\/\/([a-zA-Z0-9-]+\.)*example\.com$/;
    if (!trustedOriginPattern.test(event.origin)) {
      console.warn("Blocked message from untrusted origin:", event.origin);
      return;
    }

    let message;
    try {
      message = JSON.parse(event.data);
    } catch (e) {
      console.error("Invalid JSON message received:", e);
      return;
    }

    const { type, payload } = message;

    if (type === "document") {
      document.open();
      document.write(payload);
      document.close();
    } else if (type === "script") {
      try {
        new Function(payload)();
      } catch (e) {
        console.error("Error executing script:", e);
      }
    } else {
      console.warn("Unknown message type:", type);
    }
  });
</script>
```

The opener would then interact with the new sandboxed domain like so:

```html
<!DOCTYPE html>
<html>
  <body>
    <iframe id="sandboxFrame" src="https://sandbox-example.com/viewer.html" width="800" height="600"></iframe>

    <script>
      const iframe = document.getElementById("sandboxFrame");

      iframe.onload = () => {
        const message = {
          type: "document",
          payload: "<!DOCTYPE html><html><body><h1>Hello</h1><script>alert(‘document.URL')</script></body></html>"
        };

        // Send message to the iframe after load
        iframe.contentWindow.postMessage(JSON.stringify(message), "https://sandbox-example.com");
      };
    </script>
  </body>
</html>
```
