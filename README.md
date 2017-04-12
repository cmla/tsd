# Automatic download, crop, registration, filtering, and equalization of Sentinel and Landsat images.

[Axel Davy](mailto:axel.davy@ens.fr),
[Carlo de Franchis](mailto:carlo.de-franchis@ens-cachan.fr),
[Martin Rais](mailto:martin.rais@cmla.ens-cachan.fr)
[Enric Meinhardt-Llopis](mailto:enric.meinhardt@cmla.ens-cachan.fr)
CMLA, ENS Cachan, Université Paris-Saclay, 2016-17

# Installation and dependencies
The main scripts are `get_sentinel2.py`, `get_sentinel1.py` and `get_landsat.py`.

They use the Python modules `search_*`, `download_*`, `register.py` and
the stable libraries and scripts.

## Python packages
The required Python packages are listed in the file `requirements.txt`. They
can be installed with `pip`:

    pip install -r requirements.txt

## GDAL (on macOS)
There are at least two ways of installing `gdal`:

### Using brew

    brew install gdal --with-complete --with-python3 --HEAD

Note that this version doesn't support JP2 files.

### Using the [GDAL Complete Compatibility Framework](http://www.kyngchaos.com/files/software/frameworks/GDAL_Complete-2.1.dmg).

Download and install the `.dmg` file from the link above. Don't forget to
update your `PATH` and `PYTHONPATH` after the installation by copying these lines
in your `~/.profile`:

    export PATH="/Library/Frameworks/GDAL.framework/Programs:$PATH"
    export PYTHONPATH="/Library/Frameworks/GDAL.framework/Versions/2.1/Python/2.7/site-packages:$PYTHONPATH"

Note that this version supports JP2 files, but Python bindings are available only for Python 2.


## GDAL (Linux)
On Linux `gdal` and its Python bindings are usually straightforward to install
through your package manager.

    sudo apt-get update
    sudo apt-get install libgdal-dev gdal-bin python-gdal


## Libraries (only for Sentinel-1)
The C++ code requires the `libtiff` library. On OSX you can install it with `brew`:

    brew install libtiff

Then the code can be compiled by running `make`. This should produce a `bin`
folder containing two compiled programs: `srtm4` and `srtm4_which_tile`.

## Docker (only to use Sen2cor)
The Sentinel-2 pipeline, implemented in `get_sentinel2.py`, can optionally use
the third party plugin Sen2Cor to detect clouds and vegetation. From ESA
website:

> Sen2Cor is a processor for Sentinel-2 Level 2A product generation and
> formatting; it performs the atmospheric-, terrain and cirrus correction of
> Top-Of- Atmosphere Level 1C input data. Sen2Cor creates Bottom-Of-Atmosphere,
> optionally terrain- and cirrus corrected reflectance images; additional,
> Aerosol Optical Thickness-, Water Vapor-, Scene Classification Maps and Quality
> Indicators for cloud and snow probabilities.

But as Sen2Cor is an awful spaghetti monster and is very hard to install, we
encapsulated it in a [Docker
container](https://hub.docker.com/r/carlodef/sen2cor/). Hence `docker` is
needed to run the Sentinel-2 pipeline. The installation is straightforward: follow
the [instructions on the Docker
website](https://docs.docker.com/engine/installation/). Once `docker` is
installed, download the Sen2Cor docker image with:

    docker pull carlodef/sen2cor


# Usage

## From the command line
The pipeline can be used from the command line through the Python scripts
`get_sentinel2.py` and `get_sentinel1_scihub.py`. For instance, to download
and process Sentinel-2 images of the Jamnagar refinery, located at latitude
22.34806 and longitude 69.86889, run

    python get_sentinel2.py --lat 22.34806 --lon 69.86889 -b 2 3 4 -r -o test

This will download crops of size 5000 x 5000 meters from the bands 2, 3 and 4,
corresponding to the blue, green and red channels, and register them through
time. To specify the desired bands, use the `-b` or `--band` flag. The crop
size can be changed with the `-w` flag. For instance

    python get_sentinel2.py --lat 22.34806 --lon 69.86889 -b 11 12 -w 8000

will download crops of size 8000 x 8000 meters, only for the SWIR channels (bands 11
and 12), without registration (no option `-r`).

All the available options are listed when using the `-h` or `--help` flag:

    python get_sentinel2.py -h
    python get_sentinel1_scihub.py -h

You can also run `search_sentinel2.py` or `registration.py` from
the command line separately to use only single blocks of the pipeline. Run them
with `-h` to get the list of available options.

## As Python modules

The Python modules can be imported to call their functions from Python. Refer
to their docstrings to get usage information. Here are some examples.

To check if the first image returned by Kayrros API is empty (ie black) on the
input location:

    import search_sentinel2
    lat, lon = 42, 3
    x = search_sentinel2.search_sentinel2_images_kayrros(lat, lon)
    search_sentinel2.is_image_empty_at_location(x[0], lat, lon)


To check if the last image returned by Kayrros API is covered by a cloud on
the input location:

    import search_sentinel2
    lat, lon = 42, 3
    x = search_sentinel2.search_sentinel2_images_kayrros(lat, lon)
    search_sentinel2.is_image_cloudy_at_location(x[-1], lat, lon, w=50)


To get the list of non-empty and non-cloudy images acquired from the 15/10/2016
on a given location:

    import datetime
    import search_sentinel2
    lat, lon = 42, 3
    search_sentinel2.list_usable_images(lat, lon, start_date=datetime.date(2016, 10, 15), api='kayrros')


To download crops and register them through time:

    #!/usr/bin/env python

    import os
    import datetime
    import search_sentinel2
    import download_sentinel2
    import registration

    lat, lon = 30.23114, -100.60898
    w, h = 5000, 5000
    bands = ['04']

    # list non-empty and non-cloudy images
    images = search_sentinel2.list_usable_images(lat, lon, start_date=datetime.date(2016, 1, 1), api='kayrros')

    # download crops
    crops = []
    for img in images:
        paths = download_sentinel2.get_crops_from_kayrros_api(img, bands, lon, lat,
                                                              w, h, 'raw')
        crops.append(paths)

    # register through time
    registered_crops = [[os.path.join('reg', os.path.basename(b)) for b in i] for i in crops]
    registration.main(crops, registered_crops)

![Image](doc/flow_chart_cloud_segmentor.png?raw=true)
