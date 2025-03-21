.. _ccddata:

Using the ``CCDData`` image class: I/O, properties and arithmetic
=================================================================

Input and output
----------------

Getting data in
+++++++++++++++

The tools in `ccdproc` accept only `~astropy.nddata.CCDData` objects, a
subclass of `~astropy.nddata.NDData`.

Creating a `~astropy.nddata.CCDData` object from any array-like data is easy:

    >>> import numpy as np
    >>> from astropy.nddata import CCDData
    >>> import ccdproc
    >>> ccd = CCDData(np.arange(10), unit="adu")

Note that behind the scenes, `~astropy.nddata.NDData` creates references to
(not copies of) your data when possible, so modifying the data in ``ccd`` will
modify the underlying data.

You are **required** to provide a unit for your data. The most frequently used
units for these objects are likely to be ``adu``, ``photon`` and ``electron``, which
can be set either by providing the string name of the unit (as in the example
above) or from unit objects:

    >>> from astropy import units as u
    >>> ccd_photon = CCDData([1, 2, 3], unit=u.photon)
    >>> ccd_electron = CCDData([1, 2, 3], unit="electron")

If you prefer *not* to use the unit functionality then use the special unit
``u.dimensionless_unscaled`` when you create your `~astropy.nddata.CCDData`
images:

    >>> ccd_unitless = CCDData(np.zeros((10, 10)),
    ...                        unit=u.dimensionless_unscaled)

A `~astropy.nddata.CCDData` object can also be initialized from a FITS file:

    >>> ccd = CCDData.read('my_file.fits', unit="adu")  # doctest: +SKIP

If there is a unit in the FITS file (in the ``BUNIT`` keyword), that will be
used, but a unit explicitly provided in ``read`` will override any unit in the
FITS file.

There is no restriction at all on what the unit can be -- any unit in
`astropy.units` or that you create yourself will work.

In addition, the user can specify the extension in a FITS file to use:

    >>> ccd = CCDData.read('my_file.fits', hdu=1, unit="adu")  # doctest: +SKIP

If ``hdu`` is not specified, it will assume the data is in the primary
extension.  If there is no data in the primary extension, the first extension
with data will be used.

Getting data out
++++++++++++++++

A `~astropy.nddata.CCDData` object behaves like a numpy array (masked if the
`~astropy.nddata.CCDData` mask is set) in expressions, and the underlying
data (ignoring any mask) is accessed through ``data`` attribute:

    >>> ccd_masked = CCDData([1, 2, 3], unit="adu", mask=[0, 0, 1])
    >>> res = 2 * np.ones(3) * ccd_masked
    >>> res.mask   # one return value will be masked
    array([False, False,  True]...)
    >>> 2 * np.ones(3) * ccd_masked.data   # doctest: +FLOAT_CMP
    array([ 2.,  4.,  6.])

You can force conversion to a numpy array with:

    >>> np.asarray(ccd_masked)
    array([1, 2, 3])
    >>> np.ma.array(ccd_masked.data, mask=ccd_masked.mask)  # doctest: +SKIP

A method for converting a `~astropy.nddata.CCDData` object to a FITS HDU list
is also available. It converts the metadata to a FITS header:

    >>> hdulist = ccd_masked.to_hdu()

You can also write directly to a FITS file:

    >>> ccd_masked.write('my_image.fits')

Essential properties
--------------------

Metadata
++++++++

When initializing from a FITS file, the ``header`` property is initialized using
the header of the FITS file. Metadata is optional, and can be provided by any
dictionary or dict-like object:

    >>> ccd_simple = CCDData(np.arange(10), unit="adu")
    >>> my_meta = {'observer': 'Edwin Hubble', 'exposure': 30.0}
    >>> ccd_simple.header = my_meta  # or use ccd_simple.meta = my_meta

Whether the metadata is case sensitive or not depends on how it is
initialized. A FITS header, for example, is not case sensitive, but a python
dictionary is.

Masks and flags
+++++++++++++++

Although not required when a `~astropy.nddata.CCDData` image is created you
can also specify a mask and/or flags.

A mask is a boolean array the same size as the data in which a value of
``True`` indicates that a particular pixel should be masked, *i.e.* not be
included in arithmetic operations or aggregation.

Flags are one or more additional arrays (of any type) whose shape matches the
shape of the data. For more details on setting flags see
`astropy.nddata.NDData`.

WCS
+++

The  ``wcs`` attribute of `~astropy.nddata.CCDData` object can be set two ways.

+ If the `~astropy.nddata.CCDData` object is created from a FITS file that has
  WCS keywords in the header, the ``wcs`` attribute is set to a
  `astropy.wcs.WCS` object using the information in the FITS header.

+ The WCS can also be provided when the `~astropy.nddata.CCDData` object is
  constructed with the ``wcs`` argument.

Either way, the ``wcs`` attribute is kept up to date if the
`~astropy.nddata.CCDData` image is trimmed.

Uncertainty
+++++++++++

Pixel-by-pixel uncertainty can be calculated for you:

    >>> data = np.random.default_rng().normal(size=(10, 10), loc=1.0, scale=0.1)
    >>> ccd = CCDData(data, unit="electron")
    >>> ccd_new = ccdproc.create_deviation(ccd, readnoise=5 * u.electron)

See :ref:`create_deviation` for more details.

You can also set the uncertainty directly, either by creating a
`~astropy.nddata.StdDevUncertainty` object first:

    >>> from astropy.nddata.nduncertainty import StdDevUncertainty
    >>> uncertainty = 0.1 * ccd.data  # can be any array whose shape matches the data
    >>> my_uncertainty = StdDevUncertainty(uncertainty)
    >>> ccd.uncertainty = my_uncertainty

or by providing a `~numpy.ndarray` with the same shape as the data:

    >>> ccd.uncertainty = 0.1 * ccd.data  # doctest: +ELLIPSIS
    INFO: array provided for uncertainty; assuming it is a StdDevUncertainty. [...]

In this case the uncertainty is assumed to be
`~astropy.nddata.StdDevUncertainty`. Using `~astropy.nddata.StdDevUncertainty`
is required to enable error propagation in `~astropy.nddata.CCDData`

If you want access to the underlying uncertainty use its ``.array`` attribute:

    >>> ccd.uncertainty.array  # doctest: +ELLIPSIS
    array(...)

Arithmetic with images
----------------------

Methods are provided to perform arithmetic operations with a
`~astropy.nddata.CCDData` image and a number, an astropy
`~astropy.units.Quantity` (a number with units) or another
`~astropy.nddata.CCDData` image.

Using these methods propagates errors correctly (if the errors are
uncorrelated), take care of any necessary unit conversions, and apply masks
appropriately. Note that the metadata of the result is *not* set if the operation
is between two `~astropy.nddata.CCDData` objects.

    >>> result = ccd.multiply(0.2 * u.adu)
    >>> uncertainty_ratio = result.uncertainty.array[0, 0]/ccd.uncertainty.array[0, 0]
    >>> float(round(uncertainty_ratio, 5))   # doctest: +FLOAT_CMP
    0.2
    >>> result.unit
    Unit("adu electron")

.. note::
    In most cases you should use the functions described in
    :ref:`reduction_toolbox` to perform common operations like scaling by gain or
    doing dark or sky subtraction. Those functions try to construct a sensible
    header for the result and provide a mechanism for logging the action of the
    function in the header.


The arithmetic operators ``*``, ``/``, ``+`` and ``-`` are *not* overridden.

.. note::
   If two images have different WCS values, the wcs on the first
   `~astropy.nddata.CCDData` object will be used for the resultant object.
