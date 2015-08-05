# image-steam

A simple, fast, and highly customizable on-the-fly image manipulation web server built atop Node.js.

[![NPM version](https://badge.fury.io/js/image-steam.png)](http://badge.fury.io/js/image-steam) [![Dependency Status](https://gemnasium.com/asilvas/node-image-steam.png)](https://gemnasium.com/asilvas/node-image-steam) [![Bitdeli Badge](https://d2weczhvl823v0.cloudfront.net/asilvas/node-image-steam/trend.png)](https://bitdeli.com/free "Bitdeli Badge")

[![NPM](https://nodei.co/npm/image-steam.png?downloads=true&stars=true&downloadRank=true)](https://www.npmjs.org/package/image-steam)

***State: Beta***


# Why Image Steam?

There are a number of options out there, but differentiates itself by:

* Separating itself from a Web Server, so core logic can be used elsewhere.
  Routing, throttling, image processing, storage make up the core components.
* Optimizes originally uploaded asset to account for large uploads, enabling
  a higher quality service by making the pipeline for image operations
  substantially faster.
* Quality of service features such as throttling and memory thresholds, to best
  take advantage of your hardware under ideal and non ideal scenarios.
* Highly configurable. Everything all the way down to how image operations are mapped
  can be overridden. Most solutions are very prescriptive on how it must work.
* Supports, but does not require a CDN to front it.
* Provides an abstraction atop image processing libraries, enabling per-operation
  level of control to enable using the right tool for the given operation. Bugs,
  features, performance are a few of the factors that may influence this.
* Friendly CLI to create your web server. No forking necessary.
* Good *Nix & Windows support. 
* Device centric responses, where more than a URI may influence response.
  Compression and Accepts header (i.e. WebP) being examples.



# Installation

The speed and power of this module would not be possible without the incredible
work of libvips (low level image processor), Sharp (depends on libvips), and xxHash
for lightning-fast hashing.

1. Install Sharp via http://sharp.dimens.io/en/stable/install/ -
   This should take care of the libvips dependency.
2. Run `npm install`
   * May need to run as Admin on Windows.


# Usage

## Standalone Server

While Routing, Throttling, and Storage are all independently usable and configurable,
a basic usage example that pulls everything together can be as simple as:

```
npm install image-steam -g
isteam --isConfig './myconfig.json'
```

Config can also be a CommonJS file:

```
isteam --isConfig './myconfig.js'
```

## Connect Middleware

Or if you prefer to incorporate into your own app:

```
var http = require('http');
var imgSteam = require('image-steam');

http.createServer(new imgSteam.http.Connect({ /* options */ }).getHandler())
  .listen(13337, '127.0.0.1')
;
```

Which is equivalent of cloning this repo and invoking `npm start`.


# Performance

While this module provides granular control over HTTP throttling to provide
the highest quality of service possible, performance is entirely from Sharp
and libvips: http://sharp.dimens.io/en/stable/performance/#performance



# Options

```
isteam --isConfig './myconfig.json'
```

## Http Options

```
{ "http": { "port": 80 } }
```

* `http.port` (default: `13337`) - Port to bind to.
* `http.host` (default: `"localhost"`) - Host to bind to.
* `http.backlog` (default: `511`) - [TCP backlog](https://nodejs.org/api/net.html#net_server_listen_port_host_backlog_callback).
* `http.ssl` (optional) - If object provided, will bind with [TLS Options](https://nodejs.org/api/tls.html#tls_tls_createserver_options_secureconnectionlistener).
* `http.ssl.pfx` - If string, will auto-load from file system.
* `http.ssl.key` - If string, will auto-load from file system.
* `http.ssl.cert` - If string, will auto-load from file system.

## Storage Options

```
{ "storage": { "driver": "s3", "endpoint": "s3.amazonaws.com", "accessKey": "abc", "secretKey": "123" } }
```

Bundled storage support includes:

* `storage.driver=fs` - File System driver
  * `path` (***required***) - Root path on file system.
* `storage.driver=s3` - Should work with any S3-compatible storage.
  * `endpoint` (default: `"s3.amazonaws.com"`) - Endpoint of S3 service.
  * `port` (default: `443`) - Non-443 port will auto-default secure to `false`.
  * `secure` (default: `true` only if port `443`) - Override as needed.
  * `accessKey` (***required***) - S3 access key.
  * `secretKey` (***required***) - S3 secret key.
  * `style` (default: `"path"`) - May use `virtualHosted` if bucket is not in path.

Custom storage types can easily be added via exporting `fetch` and `store`.
See `lib/storage/fs` or `lib/storage/s3` for reference.


## Throttle Options

```
{
  "throttle": {
    "ccProcessors": 4,
    "ccPrefetchers": 20,
    "ccRequests": 100
  }
}
```

Throttling allows for fine grain control over quality of service, as well as optimizing to your hardware.

* `throttle.ccProcessors` (default: `4`) - Number of concurrent image processing operations.
  Anything to exceed this value will wait (via semaphore) for next availability.
* `throttle.ccPrefetchers` (default: `20`) - Number of concurrent storage request operations.
  This helps prevent saturation of your storage and/or networking interfaces to provide the
  optimal experience.
  Anything to exceed this value will wait (via semaphore) for next availability.
* `throttle.ccRequests` (default: `100`) - Number of concurrent http requests.
  Anything to exceed this value will result in a 503 (too busy), to avoid an indefinite pileup.


## Router Options

```
{
  "router": {
    "originalSteps": {
      "resize": {
        "width": "2560", "height": "1440", "max": "true", "canGrow": "false"
      }
    }
  }
}
```

Most router defaults should suffice, but you have full control over routing. See [Routing](#routing) for more details.

Options:
* `router.pathDelimiter` (default: `"/:/"`) - Unique (uri-friendly) string to break apart image path, and image steps.
* `router.cmdKeyDelimiter` (default: `"/"`) - Seperator between commands (aka image steps).
* `router.cmdValDelimiter` (default: `"="`) - Seperator between a command and its parameters.
* `router.paramKeyDelimiter` (default: `","`) - Seperator between command parameters.
* `router.paramValDelimiter` (default: `":"`) - Seperator between a parameter key and its value.
* `router.originalSteps` (default: [Full Defaults](https://github.com/asilvas/node-image-steam/blob/master/lib/router/router-defaults.js)) - Steps performed on the original asset to optimize subsequent image processing operations. This can greatly improve the user experience for very large, uncompressed, or poorly compressed images.
* `router.steps` (default: [Full Defaults](https://github.com/asilvas/node-image-steam/blob/master/lib/router/router-defaults.js)) - Mapping of URI image step commands and their parameters. This allows you to be as verbose or laconic as desired.





# Routing

Routing is left-to-right for legibility.


Routing format:

  `{path}{pathDelimiter}{cmd1}{cmdValDelimiter}{cmd1Param1Key}{paramValDelimiter}{cmd1Param1Value}{paramKeyDelimiter}{cmdKeyDelimiter}?{queryString}`

Example URI using [Default Options](#router-options):

  `some/image/path/:/cmd1=param1:val,param2:val,param3noVal/cmd2NoParams?cache=false`

Or a more real-world example:  
  
  `/my-s3-bucket/big-image.jpg/:/rs=w:640/cr=w:90%,h:90%`

See [Things to Try](#things-to-try) for many more examples.


# Supported Operations

## Resize (rs)

Resize an image, preserving aspect or not.

Arguments:

* Width (`w`, optional*) - Width of new size. Supports Dimension Modifiers.
* Height (`h`, optional*) - Height of new size. Supports Dimension Modifiers.
* Max (`mx`, default) - Retain aspect and use dimensions as the maximum
  permitted during resize.
* Min (`m`, optional) - Retain aspect and use dimensions as the minimum
  permitted during resize. Set to any value to enable.
* Ignore Aspect Ratio (`i`, default: `false`) - If true will break aspect and
  resize to exact dimensions.
* Can Grow (`cg`, default: `false`) - If `true`, will allow image to exceed the dimensions of the original.

Note: Width or Height are optional, but at least one must be provided.

### Examples

1. `rs=w:640` - Resize up to 640px wide, preserving aspect.
2. `rs=h:480` - Resize up to 480px tall, preserving aspect.
3. `rs=w:1024,h:768,m,cg=true` - Resize to a minimum of 1024 by 768, preserving aspect, and allow it to exceed size of original.


## Crop (cr)

Crop an image to an exact size.

Arguments:

* Top (`t`, default: `0`) - Offset from top. Supports Dimension Modifiers.
* Left (`l`, default: `0`) - Offset from left. Supports Dimension Modifiers.
* Width (`w`, default: width-left) - Width of new size. Supports Dimension Modifiers.
* Height (`h`, default: height-top) - Height of new size. Supports Dimension Modifiers.
* Anchor (`a`, default: `cc`) - Where to anchor from, using center-center by default. Top
  and Left are applied from the anchor. Possible horizontal axis
  values include left (l), center (c), and right (r). Possible vertical axis
  values include top (t), center (c), and bottom (b).

### Examples

1. `cr=t:10%,l:10%,w:80%,h:80%` - Crop 10% around the edges
2. `cr=w:64,h:64,a=cc` - Crop 64x64 anchored from center.
3. `cr=l:10,w:64,h:64` - Crops 64x64 from the left at 10px (ignoring the horizontal
   axis value of `c`), and vertically anchors from center since top is not provided.


## Background (bg)

***Not yet supported***

Arguments:

* Red (`r`) - Red component of the RGB(A) spectrum.
  Do not use in conjunction with Hex color.
* Green (`g`) - Green component of the RGB(A) spectrum.
  Do not use in conjunction with Hex color. 
* Blue (`b`) - Blue component of the RGB(A) spectrum.
  Do not use in conjunction with Hex color.
* Alpha (`a`) - Optional Alpha component of the RGB(A) spectrum.
  Do not use in conjunction with Hex color.
* Hex (`#`) - Full hex color (i.e. #ffffff).
  Do not use in conjunction with RGB(A) color.


## Flatten (ft)

***Not yet supported***

Merge alpha transparency channel, if any, with background.


## Rotate (rt)

Arguments:

* Degrees (`d`) - Degrees to rotate the image, in increments of 90.
  Future implementations may support non-optimized degrees of rotation.

### Examples

1. `rt=d=90` - Rotate 90 degrees.


## Flip (fl)

Not to be confused with rotation, flipping is the process of flipping
an image on its horizontal and/or vertical axis.

Arguments:

* X (`x`) - Flip on the horizontal axis. No value required.
* Y (`y`) - Flip on the vertical axis. No value required.

### Examples

1. `fl=x` - Flip horizontally.
2. `fl=x,y` - Flip on x and y axis.


## Quality (qt)

The output quality to use for lossy JPEG, WebP and TIFF output formats. 

* Quality (`q`, default: `80`) - Value between 1 (worst, smallest) and
  100 (best, largest).  


## Compression (cp)

An advanced setting for the zlib compression level of the lossless
PNG output format. The default level is 6.

* Compression (`c`, default: `6`) - Number between 0 and 9.  


## Progressive (pg)

Use progressive (interlace) scan for JPEG and PNG output. This
typically reduces compression performance by 30% but results in
an image that can be rendered sooner when decompressed.

Can be useful for images that always need to be seen ASAP, but should
not be used otherwise to save bandwidth.

### Examples

1. `rs=w:3840/pg` - Create a big 4K-ish image and use progressive rendering
   to demonstrate value in some use cases.


## Interpolation (ip)

***Not yet supported***

Use the given interpolator for image resizing. Defaults to "bilinear".

Arguments:

* Interpolator (`i`, optional) - Process to use for resizing, from fastest to slowest:
  * nearest - Use nearest neighbour interpolation, suitable for image enlargement only.
  * bilinear - Use bilinear interpolation, the default and fastest image reduction interpolation.
  * bicubic - Use bicubic interpolation, which typically reduces performance by 5%.
  * vertexSplitQuadraticBasisSpline - Use VSQBS interpolation, which prevents "staircasing" and typically reduces performance by 5%.
  * locallyBoundedBicubic - Use LBB interpolation, which prevents some "acutance" and typically reduces performance by a factor of 2.
  * nohalo - Use Nohalo interpolation, which prevents acutance and typically reduces performance by a factor of 3.


## Format (fm) 

***Not yet supported***

Override the auto-detected optimal format to output. Do not use this unless
you have good reason.

Arguments:

* Format (`f`, required) - Format to output: "jpeg", "png", or "webp".


## Metadata (md)

Carry metadata from the original image into the outputted image. Enabled by default.

Arguments:

* Enabled (`e`, default: 'true') - Set to `false` to not preserve metadata from original.


## Sharpen (fx-sp)

***Not yet supported***

Arguments:

* Radius (r) - Optional sharpening mask to apply in pixels, but comes at
  a performance cost.
* Flat (f) - Optional sharpening to apply to flat areas. Defaults to 1.0.
* Jagged (j) - Optional sharpening to apply to jagged areas. Defaults to 2.0.
 
 
## Blur (fx-bl)

Fast mild blur by default, but can override the default sigma for more
control (at cost of performance).

Arguments:

* Sigma (s) - The approximate blur radius in pixels, from 0.3 to 1000.

### Examples

1. `fx-bl=s:5` - Blur using a stima radius of 5 pixels.


## Greyscale (fx-gs)

Convert to 8-bit greyscale.


## Normalize (fx-nm)

***Not yet supported***

Enhance output image contrast by stretching its luminance to cover the full
dynamic range. This typically reduces performance by 30%.


# Dimension Modifiers

Dimension modifiers can be applied to any values where size and
location are represented.

## Pixels

Any numeric value around measurement without explicit unit type
specified is implicitly of type px.

### Examples
1. `rs=w:200,h:300` - 200x300 pixels
2. `rs=w:200px,h:300px` - Identical to #1
3. `cr=t:15,l:10,w:-10,h:-15` - Using pixel offsets


## Percentage

A percentage applied to original value by supplying the percentage (%) modifier.

### Examples
1. `rs=w:50%,h:50%` - 50% of source width and height
2. `cr=t:15%,l:10%,w:80%,h:70%` - 15% from top and bottom, 10% from left and right


## Offset

To be used in conjunction with locations or dimensions,
a plus (+) or minus (-) may be used to imply offset from original.

### Examples

1. `rs=w:+50px,h:-50px` - 50px wider than original, 50px shorter than original
2. `rs=w:+10%,h:-10%` - 10% wider than original, 10% shorter than original


# Error Handling

All major classes inherit from EventEmitter. By default `http.start` will
log errors to `stderr`, but can be disabled in options by setting
`log.errors` to `false` if you want more fine grained control.

## Connect Errors

The next level down is Connect, and all child classes (shown below) will
bubble up through this class:

```
var http = require('image-steam').http;
var connect = new http.Connect();
connect.on('error', function(err) { /* do something */ });
```


## Throttle Errors

A lower level class with no children:

```
var http = require('image-steam').http;
var throttle = new http.Throttle();
throttle.on('error', function(err) { /* do something */ });
```


## Processor Errors

A lower level class with no children:

```
var Processor = require('image-steam').Processor;
var processor = new Processor();
processor.on('error', function(err) { /* do something */ });
```


## Storage Errors

A lower level class with no children:

```
var Storage = require('image-steam').Storage;
var storage = new Storage();
storage.on('error', function(err) { /* do something */ });
```




# Things to try:

* `rs=w:640` - Resize to 640 width, retain aspect
* `rs=w:640/cr=l:5%,t:10%,w:90%,h:80%` - Same as above, and
  crop 5% of the sides and 10% of the top and bottom
* `rs=w:640/cr=l:5%,t:10%,w:90%,h:80%/fx-gs` - Same as above, and
  apply greyscale effect.
* `rs=w:640/cr=l:5%,t:10%,w:90%,h:80%/fx-gs/qt=q:20` - Same as above, and
  use a low quality of 20.
* `rs=w:64,h:64,m/cr=w:64,h:64/fx-gs` - Resize image to a *minimum* of 64x64
  w/o breaking aspect so that we can then crop the image and apply
  greyscale.
* `fx-bl=s:5` - Apply a blur



## License

[MIT](https://github.com/asilvas/node-image-steam/blob/master/LICENSE.txt)
