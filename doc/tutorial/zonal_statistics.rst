======================================
Zonal statistics with point cloud
======================================

:Author: Luca Delucchi
:Contact: luca.delucchi@fmach.it
:Date: 28/12/2015

Introduction
-----------------
Sometimes it is useful to obtain information from a subset of the point cloud
input data. This tutorial describes a method to get statistic about
the point cloud data contained in polygonal features. This is similar to
a zonal statistics using point cloud.

Requirements
-----------------
To run this tutorial it is required to have PDAL along with its
`Python package <https://pypi.python.org/pypi/PDAL>`_ and the
Python libraries `numpy`_, `GDAL`_, `xml.etree`_ installed.

This tutorial utilizes a Trentino (Italy) dataset, it includes a LAZ file
and a SHAPE file. You can download it from
.. TODO add link

Approach
---------------
The idea to obtain the statistics is to cycle around all the features of
our input vector data, for each feature the :ref:`filters.crop` will be
used to get the point cloud inside the feature boundary and after this
the statistics are computed.

To do this some lines of Python code are required. The first step is to define
the statistic to calculate, most of them are already present in `numpy`_.
If you are looking for more complex methods you can also search in `scipy`_.
If you prefer to define your own method you will define a Python function as follows

.. code-block:: python

    import numpy as np

    def hcv(inps):
        """Calculate coefficient of variation height points,
        standard_deviation/mean

        :param array inps: numpy array with input height values
        """
        return (np.std(inps) / np.mean(inps))

The next code defines a function to create a XML Pipeline file using the
:ref:`filters.crop`, this function is really important because for every
feature a different XML pipeline file is required.

.. code-block:: python

    from xml.etree.ElementTree import Element, tostring
    import tempfile

    def clip_xml_pdal(self, inp, out, bbox, compres, inverted=False):
        """Clip a LAS file using a polygon

        :param str inp: full path for the input LAS file
        :param str out: full path for the output LAS file
        :param str bbox: a well-known text string to crop the LAS file
        :param bool compres: the output has to be compressed
        :param bool inverted: invert the cropping logic and only take points
                              outside the cropping polygon
        """
        # set a temporary file
        tmp_file = tempfile.NamedTemporaryFile(delete=False)
        # prepare the xml
        root = Element('Pipeline')
        root.set('version', '1.0')
        root.set('encoding', 'utf-8')
        # option for the writer
        optionw = Element('Option')
        optionw.set("name", "filename")
        optionw.text = str(out)
        # set the writer element
        writer = Element('Writer')
        writer.set("type", "writers.las")
        writer.append(optionw)
        # if compress is true create a LAZ file
        if compres:
            optionc = Element('Option')
            optionc.set("name", "compression")
            optionc.text = str('true')
            writer.append(optionc)
        # create the filter element
        filt = Element('Filter')
        filt.set("type", "filters.crop")
        # prepare the clip option
        clip = Element('Option')
        clip.set("name", "polygon")
        clip.text = str(bbox)
        if inverted:
            clip.set("outside", "true")
        # add clip option to filter element
        filt.append(clip)
        # option for the reader
        optionr = Element('Option')
        optionr.set("name", "filename")
        optionr.text = str(inp)
        # set the reader element
        reader = Element('Reader')
        reader.set("type", "readers.las")
        reader.append(optionr)
        # add reader to filter
        filt.append(reader)
        # add filter to writer
        write.append(filt)
        # add writer to root
        root.append(write)
        # write the xml to string into the temporary file
        if sys.platform == 'win32':
            tmp_file.write(tostring(root, 'iso-8859-1'))
        else:
            tmp_file.write(tostring(root, 'utf-8'))
        tmp_file.close()
        return tmp_file.name

At this moment some variables are defined (if you need to change some
parameters, like input or statistics, you should be able to do that here)

.. code-block:: python

    # the input vector with polygonal features
    invect = 'polygon.shp'
    # the input las/laz file
    inlas = 'trentino.laz'
    # the driver to use for the output file
    ogrdriver = 'ESRI Shapefile'
    # all the supported statistics
    statistics = {'hcv': hcv, 'max': np.max, 'mean': np.mean, 'min': np.min}
    # the stats to calculate
    stats = ['min', 'mean', 'max', 'hcv']

The next step is to initialize the input and the output vector files
using the `GDAL`_ Python API

.. code-block:: python

    import osgeo.ogr as ogr
    import pdal
    from pdal import libpdalpython

    #read the input vector
    vect = ogr.Open(invect)
    # get the first layer
    layer = vect.GetLayer(0)
    # check if layer has polygon features
    if layer.GetGeomType() not in [ogr.wkbPolygon, ogr.wkbPolygon25D,
                                   ogr.wkbMultiPolygon, ogr.wkbMultiPolygon25D]:
        raise Exception("Geometry type is not supported, please use a "
                        "polygon vector file")
    # remove output if exists
    try:
        driver.DeleteDataSource(outvect)
    except Exception:
        pass
    # create the driver for new output file
    newdata = driver.CreateDataSource(outvect)
    # copy the input layer in the output datasource
    newdata.CopyLayer(layer, 'las_zonal_stats')
    # get the new layer
    newlayer = newdata.GetLayer()

The last operation before starting the analysis is to create the fields
in the attribute table to store the result of our analysis

.. code-block:: python

    for s in stats:
        field = ogr.FieldDefn(s, ogr.OFTReal)
        newlayer.CreateField(field)

It is time to run the cycle for each feature and calculate the required
stats

.. code-block:: python

    for inFeature in newlayer:
        # get a temporary file name for the output las
        tmp_file = tempfile.NamedTemporaryFile(delete=False)
        tmp_file.close()
        outlas = tmp_file.name
        # get the geometry
        inGeom = inFeature.GetGeometryRef()
        # prepare the xml for the clip
        pdalxml = clip_xml_pdal(inlas, outlas, inGeom.ExportToWkt())
        with open(pdalxml, 'rb') as f:
            if sys.platform == 'win32':
                xml = f.read().decode('iso-8859-1')
            else:
                xml = f.read().decode('utf-8')
        # set up the xml pipeline and execute it
        pipe = libpdalpython.PyPipeline(xml)
        pipe.execute()
        # read the output, saving the point cloud as a numpy matrix
        data = pipe.arrays()[0]
        # get all the Z from the subset
        zs = map(lambda x: x[2], data)
        # if some data are founded
        if len(zs) != 0:
            # calculate the statistics and save the result into attribute table
            for s in stats:
                val = statistics[s](zs)
                inFeature.SetField(s, val)
        # set the feature into output file
        newlayer.SetFeature(inFeature)
        inFeature = None
    # when cycle is finished destroy the datasource to save it
    newdata.Destroy()

Once your script is completed and ready to be run, you just have to put all pieces
together into a single file and run it in the same directory where you stored
the input data.

.. _`numpy`: http://www.numpy.org/
.. _`GDAL`: https://pypi.python.org/pypi/GDAL
.. _`scipy`: http://www.scipy.org/
.. _`xml.etree`: https://docs.python.org/2/library/xml.etree.elementtree.html
