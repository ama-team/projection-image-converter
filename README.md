# AmaTeam\Image\Projection

[![Packagist](https://img.shields.io/packagist/v/ama-team/projection-framework.svg?style=flat-square)](https://packagist.org/packages/ama-team/projection-framework)
[![CircleCI branch](https://img.shields.io/circleci/project/github/ama-team/php-projection-framework/master.svg?style=flat-square)](https://circleci.com/gh/ama-team/php-projection-framework/tree/master)
[![Coveralls](https://img.shields.io/coveralls/ama-team/php-projection-framework/master.svg?style=flat-square)](https://coveralls.io/github/ama-team/php-projection-framework?branch=master)
[![Scrutinizer](https://img.shields.io/scrutinizer/g/ama-team/php-projection-framework/master.svg?style=flat-square)](https://scrutinizer-ci.com/g/ama-team/php-projection-framework)
[![Code Climate](https://img.shields.io/codeclimate/github/ama-team/php-projection-framework.svg?style=flat-square)](https://codeclimate.com/github/ama-team/php-projection-framework)

<sup><sub>
Don't believe the coverage, it's [lying](https://github.com/sebastianbergmann/php-code-coverage/issues/409).
</sub></sup>

This is a simple library created for common work with sphere 
projections - at the moment of writing, to convert equirectangular and
cube map projection types one into another.

## Installation

```bash
composer require ama-team/image-projection-framework
```

PHP 5.6+ and GD / Imagick are required. All file operations are done 
through `league/flysystem`, which is explicitly set as dependency.

## Usage

### Sixty-second start

```php
use \AmaTeam\Image\Projection\Framework;
use \AmaTeam\Image\Projection\Framework\EncodingOptions;
use \AmaTeam\Image\Projection\Specification;
use \AmaTeam\Image\Projection\Image\Format;

$framework = new Framework();
$source = new Specification('equirect', 'tmp/uploads/source.jpg');
$target = new Specification(
    'cube',
    'static/pano/{f}/{x}/{y}.jpg',
    new Box(512, 512) // tile size,
    new Box(2, 2) // layout, amount of tiles horizontally and vertically
);

$options = (new EncodingOptions())->setQuality(0.9);

$framework->convert($source, $target, Format::JPEG, $options);
```

You'll need a `Framework` instance to start off, it will use `cwd()` as
fs root. Next, `Framework` has `#convert()` and `#convertAll()` methods
to turn one specification into other (others), optionally specifying
format and encoding options (don't worry, it's already jpeg / 90% by 
default). Specification is basically a description of a pano: it's 
type, it's location, it's face size and tile size. Location allows 
several placeholders - {f} or {face}, {x} and {y} to automatically 
recognize / populate it with tile parameters. You won't need them at 
all for equirectangular images, while faces for cubemap are named using
their first letters - u(p), f(ront), b(ack), l(eft), r(ight), d(own). 
Type is found using case-insensitive prefix search, so 
`EQUIRECTANGULAR` and `equirect` is basically the same.

Chances are you will be creating multi-resolution cube map out of 
equirectangular projection, and to lessen resource consumption,
source projection should be read just once - this can be easily 
done using `#convertAll()`, which uses single reader:

```php
$targets = array_map(function ($index) {
    $side = 256 * (int) pow($index, 2);
    $tileSide = min($side, 512);
    $size = $side / $side;
    $tileSize = new Box($tileSide, $tileSide);
    $layout = new Box($size, $size);
    $path = sprintf('static/pano/%d/{f}/{x}/{y}.jpg', $index);
    return new Specification('cube', $path, $tileSize, $layout);
}, range(0, 3));

$framework->convertAll($source, $targets);
```

This should do the trick.

The last thing to tell: **it is extremely slow**. The current 
processing model fills target projection pixel-by-pixel, so it takes
a while (up to minute on my laptop, cheap servers may be quite slower).
And yes, it will burn a single core as much as it can. Also don't 
forget that source projection has to fit in memory, don't blame me for 
your images being too big.

## Conventions

Projections are represented as a three-level deep tile structures. 
Every projection consists of arbitrary amount of faces (six for cube 
map, one for equirectangular and planet), and each face is a 
two-dimension grid of tiles. Tile position, which is determined as
{face, x, y}, is expressed through curly-braces placeholders, as in the
main usage example. All projections follow that rule (even though 
equirectangular is unlikely to have multiple tiles), but that may change 
in future releases.

Framework assumes that all tiles have the exact sizes. Not following 
that rule may result in undefined behavior and errors.

Flysystem usage assumes that all paths are specified using slash
(not backslash), and there are no absolute paths at all, all paths
are relative to root.

## Considerations

This library processes images by creating virtual sphere accessor from
source projection and then using it to populate target projection, 
texel by texel. This raises following problems:

- To complete target projection, source projection has to be fully
loaded into RAM. This could be quite a lot for big projections, and
the library itself doesn't do any size validations, so you have to 
watch RAM yourself. Also, image loading adds time penalty itself.
- The main workload is giant loop that asks for color of specific
sphere coordinates and then populates target image with that color.
This results in {number of pixels} function calls, and that number is
usually huge even for modern processor - generating 2048x2048 cube map
will need to populate 24M pixels, and all that work would be done on
single CPU core, and there is not much overhead to optimize.
**It is slow as hell and not much can be done**. Generating such a
projection for two minutes is OK, having one core occupied at 100% is
expected as well. All you can do is to parallelize work on several
cores by filtering which tiles should be generated; this, however,
will require to load source projection into RAM for every core.

If you want to convert projections faster or exploit GPU, you will need
to implement the same thing yourself, most probably using other 
language. Image processing was always a complex thing and PHP is not 
enough to solve it efficiently.

## Filtering generated tiles, adding processors and listeners

All the specified above needs more fine work and other methods.
`Framework` instance provides method `getConverter()` that allows more
detailed work using methods `createConversion()` and
`createConversions()`. Those will create an object ready to 
perform conversion, but not yet launched (which is done through
`#run()` method), and which expose additional processing capabilities.

If you've considered parallelizing work by scheduling different tiles
to different workers, you'll need to restrict some tiles from being 
created on particular worker. Filter functionality exists just for 
this case:

```php
// only specified faces will be created
$filter = new FaceFilter('f', 'b');
$options = (new ConversionOptions())->setFilters([$filter]);
$framework
    ->getConverter()
    ->createConversion($source, $target, $options)
    ->run();
```

If you need to add some kind of watermark or apply custom antialiasing,
you can do it via processor which is run on tile after it's generation:

```php
$fxaaProcessor = new FXAAProcessor();
$watermarkProcessor = new WatermarkProcessor();
$conversion = $framework
    ->getConverter()
    ->createConversion($source, $target);
    // 50 is order in which processor will run, so it will run before
    // watermark processor
    ->addProcessor(50, $fxaaProcessor)
    ->addProcessor(99, $watermarkProcessor);
    ->run();
```

Last thing to mention is listeners - those are simply some side-effect
generators that accept fully generated tiles. The most common example
is SaveListener that is not included in conversion by default:

```php
$filesystem = $framework->getRegistry()->getFilesystem();
$listener = new SaveListener($filesystem, Format::JPEG);
$conversion = $framework
    ->getConverter()
    ->createConversion($source, $target)
    ->addListener($listener)
    ->run();
```

This is what actually default methods in sixty second example do.

## Adding custom projection type

All projections are quite the same: they are bunch of images with some
rules of mapping sphere on it and vice versa. All you actually need is
to implement `MappingInterface`, bury it in AbstractHandler child and
register in framework:

```php
$framework = new Framework();
$framework->register('LittlePlanet', new LittlePlanetHandler()); 
``` 

You're ready to go.

## Contributing

Feel free to fork and send PR to **dev** branch.

### Testing

Testing is done using Codeception, and everything, except for directory
structure and installable fixtures, is quite the usual.

To install the fixtures, simply run `test:setup` task:

```bash
bin/robo test:setup
```

This will download some public license test images from flickr 
and put them into `tests/Data/External/Projections`, as well as
cur them into faces.

You can run tests by using `test` and `test:<suite>` commands:

```bash
bin/robo test
bin/robo test:acceptance
bin/robo test:unit --coverage
```

Acceptance tests are doing some real work mangling millions of texels, 
so you should never ever turn on coverage for them unless you are on
Intel Stress Test team. They are already slow enough, believe me.

Following command will generate coverage and [Allure][] reports:

```bash
bin/robo test:report
```

### Dev branch state

[![CircleCI](https://img.shields.io/circleci/project/github/ama-team/php-projection-framework/dev.svg?style=flat-square)](https://circleci.com/gh/ama-team/php-projection-framework/tree/dev)
[![Coveralls](https://img.shields.io/coveralls/ama-team/php-projection-framework/dev.svg?style=flat-square)](https://coveralls.io/github/ama-team/php-projection-framework?branch=dev)
[![Scrutinizer](https://img.shields.io/scrutinizer/g/ama-team/php-projection-framework/dev.svg?style=flat-square)](https://scrutinizer-ci.com/g/ama-team/php-projection-framework)

## License

MIT License

AMA Team, 2017

  [allure]: https://github.com/allure-framework/allure
