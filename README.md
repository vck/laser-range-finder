# laser-range-finder
Calculates distance using images taken with a generic webcam and a low-power laser

# Overview

This package implements a Python class that calculates distance from the camera given two images taken by the camera, where one image contains a laser line projection parallel to the camera's focal plane and the other image contains no laser projection.

This method is very similar to the approach described [here](https://sites.google.com/site/todddanko/home/webcam_laser_ranger), although it has been extended to apply to a line instead of a single point.

# Installation

Recommended installation is via pip (preferrably inside a virtualenv):

    pip install laser_range_finder

# Usage

Before you can use the class, you'll need to [calibrate](#calibration) your camera. Calculate your camera's:

* rpc = radians per pixel pitch
* ro = radian offset
* h = distance between camera center and laser center

Then, pass these values when you instantiate the class:

    from laser_range_finder import LaserRangeFinder
    lrf = LaserRangeFinder(rpc=0.00103, ro=0.01, h=30)

The package is agnostic to how the images are captured, but assumes you can provide them as either a filename or as a PIL Image instance:

    distances = lrf.get_distance(off_img='~/laser-off.png', on_img='~/laser-on.png')

The value returned by `get_distance()` will be a list equal in length to the pixel width of the source images and each element will be the estimated distance from the camera in millimeters. A value of -1 indicates that no reliable distance measurement could be made in the associated column.

If you don't need the specificity of a per-pixel distance measurement, but just want a more general estimate of distance in N blocks across the image, then you can pass these distances to the function `compress_list(lst, bins)`:

    from laser_range_finder.utils import compress_list
    distances = lrf.get_distance(off_img='~/laser-off.png', on_img='~/laser-on.png')
    distances = compress_list(distances, bins=10)

# Methodology

The method assumes we start with two images, a reference image A known to contain no laser line, and an image B that definitely contains a laser line but which may possibly be distorted. We could potentially just use the single image containing the laser line, but having a negative image greatly helps us remove noise. So we start with the following sample images:

![Sample Image A](docs/images/sample1/sample1-a-0.jpg) ![Sample Image B](docs/images/sample1/sample1-b-0.jpg)

Notice that the brightness differs considerably between the two images. Later, we'll want to isolate the laser line by finding the difference between the two images, but this difference in overall brightness will interfere with that. So we fix this by using `utils.normalize()`, giving us:

![Sample Image A](docs/images/sample1/_sample1-a-1.jpg) ![Sample Image B](docs/images/sample1/_sample1-b-1.jpg)

Since these images are RGB, but the laser is red, we remove some noise by stripping out the blue and green channels using `utils.only_red()`, giving us:

![Sample Image A](docs/images/sample1/_sample1-a-2.jpg) ![Sample Image B](docs/images/sample1/_sample1-b-2.jpg)

Now we calculate the difference of the images. If the only change between the images was the laser line, then the resulting image should be completely black with the laser appearing as bright white. In practice, there will still be considerable noise, but the difference will eliminate most of that, giving us:

![Difference Image](docs/images/sample1/_sample1-diff-3.jpg)

Finally, we make a "best guess" as to where the laser line is. Since we know the line will be roughly horizontal and the laser line should now correspond to the brightest pixels in the image, we scan each column and find the rows with the brightest pixel, giving us:

![Line Estimate 1](docs/images/sample1/_sample1-line-1.jpg)

Notice the estimate is pretty close, but there's still some noise, mostly on the left-hand side of the image where the laser line is absorbed by a matte black finish. We fix this by calculating the line's mean and standard deviation brightness, and ignore anything that's less than `mean - stddev`, giving us:

![Line Estimate 2](docs/images/sample1/_sample1-line-2.jpg)

That's removed a lot of the noise, but notice there's still a tiny bit of noise in the top-left part of the image. For this setup, the laser was positioned below the camera, which means that the line should never appear above the middle row. So we can assume any laser line pixels detected above the middle row are noise. That gives us our final image, which almost perfectly detects the laser projection:

![Line Estimate 3](docs/images/sample1/_sample1-line-3.jpg)

We can then use a little trigonometry to convert the white pixel's offset from the middle row into a physical distance of the laser projection.

For reference, the source images used in this example were captured with a 5MP NoIR camera connected to a Raspberry Pi 2 and a 1mW 3V red laser line diode mounted 22.5 mm from the camera and controlled directly from a Raspberry Pi GPIO pin.

# Calibration

Although this package is agnostic about how you acquire your images, it still assumes the capture source is calibrated, and that these calibration parameters are entered correctly. The calibration sequence is nearly identical to the process outlined [here](https://sites.google.com/site/todddanko/home/webcam_laser_ranger) and [here](https://shaneormonde.wordpress.com/2014/01/25/webcam-laser-rangefinder/).

1. Position your apparatus towards a wall, at the maximum distance still detectable, with no other objects in view and take a measurement. Confirm the laser line appears close to the center of the image. Ensure the laser line is level across the image. 

2. Position your apparatus in front of several objects at known distances, each visibly marked, shine the laser and capture an image.

Create a file called ![calibrate.yml](docs/data/calibrate.yml) and populate several fields.

First, add a list attribute called `readings`, which contains the row in each column that contains the laser projection.

Next, for each manual measurement, identify the corresponding list indexes and enter these measurements into a map attribute called `distances`. For example, if you manually measured the point `A` to be 34cm from the camera, and you found that point `A` was roughly in column 56, then you'd have a `distances` value that looked like:

    distances:
        56: 34

You should enter at least 5 values or more to ensure an accurate calibration.

Next, enter your `h` value, which is the distance between the center of the camera lens and the center of the laser. Ensure the units of this measurement match the units you used for `distances`.

Next, enter the image width and height in pixels.

Then, run `python lrf_calibrate.py <path/to/calibrate.yml>`. This will calculate and output your `rpc` and `ro` values. Pass these into your instance of `LaserRangeFinder(rpc=<rpc>, ro=<ro>)` and it should output accurate distance readings.

# Testing

To run tests:

    tox
    tox -e py27 -- -s laser_range_finder/tests/test_distance.py::test_samples

or:

    py.test -s
