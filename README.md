# ember-google-publisher-tags

[![npm version](https://badge.fury.io/js/ember-google-publisher-tags.svg)](https://badge.fury.io/js/ember-google-publisher-tags)

An Ember component for adding [GPT](https://support.google.com/dfp_sb/answer/1649768?hl=en)
ads to your site.

## Usage

```hbs
{{google-publisher-tag adId="/6355419/Travel/Europe/France/Paris" width=300 height=250}}
```

The `adId` is taken straight from DFP's "Generate Tags" link. The above is a
sample ad on Google's ad network.

Optional properties:

* `placement=N`: Differentiate ads that use the same `adId` on a single page.
  For example, one ad might be `placement="upper right"`, while another might be
  `placement="lower left"`.

* `refresh=N`: Refresh the ad after `N` seconds. Each refresh also increments
  the `refreshCount` property, which might be useful.

* `refreshLimit=N` Limit refreshing to `N` times. For example, setting to 5 would
  stop refreshing after the 5th time.

* `tracing=true`: Turn on `Ember.Logger.log` tracing.

* `shouldWatchViewport=false`: Turn off checks for ad in view, if using tons of
  ads slows down your page.

* `backgroundRefresh=true`: By default, we do not refresh ads in backgrounded pages,
  according to the `document.hidden` property. If, for some strange reason, you
  want to refresh ads while nobody is looking, set this to true.

Additionally, if you want to use GPT's `setTargeting` function to serve targeted
ads, extend the `GooglePublisherTag` component and override the `addTargeting`
function in your child component. Inside this function, set the `targeting`
property to an object:

```js
// components/your-ad.js
import GPT from 'ember-google-publisher-tags/components/google-publisher-tag';

export default GPT.extend({
    tracing: true, // useful for development, especially if it's computed

    addTargeting() {
        // depending on your application, you might want to check Ember's
        // `isDestroyed` and `isDestroying` properties before calling `set`
        set(this, 'targeting', {
            placement: get(this, 'placement'),
            refresh_count: get(this, 'refreshCount'),
            planet: 'Earth'
        });
    }
};
```

```hbs
<!-- application.hbs -->
{{your-ad adId="..." width=300 height=250}}
```

## Installation

1. `ember install ember-google-publisher-tags`

2. If your app uses Ember's default `index.html`, no further installation is needed: this
  addon uses Ember's `head-footer` hook to insert the GPT initialization code into your
  page `<head>`s (unless you use `iframeJail`, see below).

3. If #2 does not apply to you, you'll have to manually add the GPT initialization
  `<script>` tag to your page `<head>`. Copy it from either [index.js](index.js) or
  [gpt-iframe.html](public/gpt-iframe.html), and paste it wherever you need to
  in your app's structure.

## Configuration

### gpt.iframeJail: boolean (default: false)

By default, GPT runs in your page's window. Since ads do all sorts of
malicious crap, you can have the ads run inside an `<iframe>` jail of their
very own. This addon comes with its own [gpt-iframe.html](public/gpt-iframe.html) file
for exactly this purpose. Set this property to `true` to:

1. Put all GPT javascript inside its own `<iframe>`

2. *disable* the `head-footer` hook for this addon, so that your page `<head>` is
unaffected

### gpt.iframeRootUrl: string (default: ENV.rootURL)

If your `dist` folder is not accessible at your application-defined `rootURL`,
use this property.

```js
// config/environment.js

module.exports = function(environment) {
    var ENV = {
        gpt: {
            // your config settings
            iframeJail: true,
            iframeRootUrl: '/somewhere/else/'
        }
    };
```

### gpt.iframeExternalUrl: string (default: none)

Use this url to host your ad iframe on a completely separate domain. Because
of cross-origin policy, using this kind of iframe requires passing things like
ad ID and targeting info via url query param, instead of via `.contentWindow`.
This also means whatever is handling the url needs to decode the query
parameter and pass it along to the GPT javascript.

`iframeExternalUrl` will ignore `iframeRootUrl`, if both are defined in your
environment.js.

Here is a full example:

1. Define `iframeExternalUrl` as `//www.example.com/gpt-iframe.php?q=`. Note
that url ends with an unset query parameter: **this is required**.

2. The addon will create an iframe like this:

    ```html
    <iframe src="//wwww.example.com/gpt-iframe.php?q=%7B%22ad%22%3A%7B%22adId%22
    %3A%22%2F6355419%2FTravel%2FEurope%2FFrance%2FParis%22%2C%22width%22%3A300
    %2C%22height%22%3A250%7D%2C%22targeting%22%3A%5B%5B%22planet%22%2C%22Earth
    %22%5D%2C%5B%22refreshCount%22%3A%222%22%5D%5D%7D%0A" ...></iframe>
    ```

    The above query-encoded string is JSON, if it was decoded and pretty-printed
    it would be:

    ```js
    {
      "ad": {
        "adId": "/6355419/Travel/Europe/France/Paris",
        "width":300,
        "height":250
      },
      "targeting": [
        ["planet","Earth"],
        ["refreshCount":"2"]
      ]
    }
    ```

3. The PHP that handlesgq}J the request looks very similar to
[gpt-iframe.html](public/gpt-iframe.html), except it decodes the query string and
assigns `window.ad` and `window.targeting`. It also doesn't have to worry
about polling the `window.startCallingGpt` variable, so that code is removed.

    ```php
    <?php
    $params = json_decode($_GET['q']);
    function json2js($k) {
        global $params;
        return json_encode($params->$k, JSON_UNESCAPED_SLASHES) . ";\n";
    }
    ?>
    <html>
        <head>
            <script type='text/javascript'>
    var googletag = googletag || {};
    googletag.cmd = googletag.cmd || [];
    (function() {
        var gads = document.createElement('script');
        gads.async = true;
        gads.type = 'text/javascript';
        var useSSL = 'https:' == document.location.protocol;
        gads.src = (useSSL ? 'https:' : 'http:') +
          '//www.googletagservices.com/tag/js/gpt.js';
        var node = document.getElementsByTagName('script')[0];
        node.parentNode.insertBefore(gads, node);
    })();
            </script>
            <style>
    body {
        margin: 0;
    }
            </style>
        </head>
        <body>
            <div id="gpt-ad"></div>
            <script>

    window.ad = <?= json2js('ad') ?>
    window.targeting = <?= json2js('targeting') ?>

    var g = googletag;
    g.cmd.push( function() {
        var slot = g.defineSlot(
            window.ad.adId,
            [ window.ad.width, window.ad.height ],
            "gpt-ad"
        ).addService(g.pubads());

        if (window.targeting) {
            window.targeting.forEach(function(pair) {
                slot.setTargeting(pair[0], pair[1]);
            });
        }

        g.enableServices();
        g.display("gpt-ad");
    });
            </script>
        </body>
    </html>
    ```

## Troubleshooting

1. Make sure your ad blocker isn't interfering.

2. Set `tracing` to true, and follow what the addon does in your browser's console.

3. Add `googfc` as a query parameter to your url, this will activate an in-page
debugging console. Eg, `http://localhost:4200/?googfc`. [More info here](https://support.google.com/dfp_sb/answer/181070?hl=en).

## Docs

* [GPT Boilerplate (annotated)](https://support.google.com/dfp_premium/answer/1638622?hl=en&ref_topic=4389931)
* [GPT Reference](https://developers.google.com/doubleclick-gpt/reference)
* [Ad Sizes](https://support.google.com/adsense/answer/185666)
* [Common Mistakes](https://developers.google.com/doubleclick-gpt/common_implementation_mistakes)
