Communicating With Sandboxed Iframes Containing Arbitrary / Unsafe HTML
============================================

Sometimes, it is necessary to render arbitrary HTML which may contain javascript -- either legitmiately or malicious. Therefore, the best thing to do is to sandbox the HTML. It is trivial to do this with the `sandbox` attribute of `iframe` ([source](https://developer.mozilla.org/en-US/docs/Web/HTML/Reference/Elements/iframe#sandbox)).

The problem using using `<iframe sandbox="allow-scripts">` is that the sandbox created also disallows the _parent_ from interacting with the iframe, as the iframe is treated cross-origin.

In order to create a sandboxed iframe which allows interaction from the parent to the iframe, we can create a custom listener inside the sandboxed iframe that listens for arbitrary javascript messags, and then executes them. Here's an example:

```html
<!DOCTYPE html>
<html>
  <body>
    <iframe
      id="sandboxFrame"
      sandbox="allow-scripts allow-modals"
      width="800"
      height="600"
    ></iframe>
    <script>
      const BOOTSTRAP = `
        <!DOCTYPE html>
        <html>
          <head>
          <meta charset="utf-8">
          <script>
            // Receive a MessagePort from the parent, then handle messages on it.
            window.addEventListener("message", function(event) {
              const port = event.ports && event.ports[0];
              if (!port) return;
              port.start();
              port.onmessage = function(ev) {
                let message;
                try {
                  message = JSON.parse(ev.data);
                } catch (e) {
                  console.error("Invalid JSON message received:", e);
                  return;
                }
                const { type, payload } = message;
                if (type === "script") {
                  try {
                    new Function(payload)();
                  } catch (e) {
                    console.log("Error executing script:", e);
                  }
                } else {
                  console.warn("Unknown message type:", type);
                }
              };
            });
          <\/script>
        </head>
        <body>
      `;
      const BOOTSTRAP_AFTER = `
          </body>
        </html>
      `;

      const iframe = document.getElementById("sandboxFrame");
      iframe.srcdoc = BOOTSTRAP + '<!DOCTYPE html><html><body><h1>Hello</h1><script>alert("hello!")<\/script></body></html>' + BOOTSTRAP_AFTER;
      iframe.onload = () => {
        // Create a MessageChannel and transfer one port to the iframe.
        const channel = new MessageChannel();
        iframe.contentWindow.postMessage("init", "*", [channel.port2]);
        channel.port1.start();
        const message2 = {
          type: "script",
          payload: 'alert(window.origin);'
        };
        channel.port1.postMessage(JSON.stringify(message2));
      };
    </script>
  </body>
</html>
```

The arbitrary HTML content to be loaded here is `<!DOCTYPE html><html><body><h1>Hello</h1><script>alert("hello!")</script></body></html>`. A sandboxed iframe is created and a MessagePort listener is installed in the iframe, then the arbitrary HTML content is added.

More information about this setup can be found on my blog post, [One-Way Sandboxed Iframes: Creating a Read-Only Iframe Sandbox That Can't Read Back](https://joshua.hu/rendering-sandboxing-arbitrary-html-content-iframe-interacting).
