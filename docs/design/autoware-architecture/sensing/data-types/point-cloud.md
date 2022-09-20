# Point cloud pre-processing design

## Overview

Point cloud pre-processing is a collection of modules that apply some primitive pre-processing to the raw sensor data.

This pipeline covers the flow of data from drivers to the perception stack.

## Recommended processing pipeline

```mermaid
graph TD
    Driver["Lidar Driver"] -->|"Cloud XYZIRCT"| FilterDC["Distortion Corrector Filter"]

    subgraph "sensing"
    FilterDC -->|"Cloud XYZIRC"| FilterOF["Outlier Remover Filter"]
    FilterOF -->|"Cloud XYZI"| FilterDS["Downsampler Filter"]
    FilterDS -->|"Cloud XYZI"| FilterTrans["Cloud Transformer"]
    FilterTrans -->|"Cloud XYZI"| FilterPR["Polygon Remover Filter / CropBox Filter"]

    FilterPR -->|"Cloud XYZI"| FilterC["Cloud Concatenator"]
    FilterX["..."] -->|"Cloud XYZI (i)"| FilterC
    end

    FilterC -->|"Cloud XYZI"| SegGr["Ground Segmentation"]
```

## List of modules

The modules used here are from [pointcloud_preprocessor package](https://github.com/autowarefoundation/autoware.universe/tree/main/sensing/pointcloud_preprocessor).

For details about the modules, see [the following table](https://github.com/autowarefoundation/autoware.universe/tree/main/sensing/pointcloud_preprocessor#inner-workings--algorithms).

It is recommended that these modules are used in a single container as components. For details see [ROS2 Composition](https://docs.ros.org/en/rolling/Tutorials/Intermediate/Composition.html)

## Point cloud fields

In the ideal case, the driver is expected to output a point cloud with the `PointXYZIRCT` point type.

| name            | datatype | description                                                                  |
| --------------- | -------- | ---------------------------------------------------------------------------- |
| X               | FLOAT32  | X position                                                                   |
| Y               | FLOAT32  | Y position                                                                   |
| Z               | FLOAT32  | Z position                                                                   |
| I (intensity)   | UINT8    | Measured reflectivity, intensity of the point                                |
| R (return type) | UINT8    | Laser return type for dual return lidars                                     |
| C (channel)     | UINT16   | Vertical channel id of the laser that measured the point                     |
| T (time)        | UINT32   | Nanoseconds passed since the time of the header when this point was measured |

### Intensity

We will use following ranges for intensity, compatible with [the VLP16 User Manual](https://usermanual.wiki/Pdf/VLP16Manual.1719942037/view):

Quoting from the VLP-16 User Manual:

> For each laser measurement, a reflectivity byte is returned in addition to distance.
> Reflectivity byte values are segmented into two ranges, allowing software to distinguish diffuse reflectors (e.g. tree trunks, clothing) in the low range from retrore-
> flectors (e.g. road signs, license plates) in the high range.
> A retroreflector reflects light back to its source with a minimum of scattering.
> The VLP-16 provides its own light, with negligible separation between transmitting laser and receiving detector, so retroreflecting surfaces pop with reflected IR light
> compared to diffuse reflectors that tend to scatter reflected energy.
>
> - Diffuse reflectors report values from 0 to 100 for reflectivities from 0% to 100%.
> - Retroreflectors report values from 101 to 255, where 255 represents an ideal reflection.

In a typical point cloud without retroreflectors, all intensity points will be between 0 and 100.

<img src="https://upload.wikimedia.org/wikipedia/commons/6/6d/Retroreflective_Gradient_road_sign.jpg" width="200">

[Retroreflective Gradient road sign, Image Source](https://commons.wikimedia.org/wiki/File:Retroreflective_Gradient_road_sign.jpg)

But in a point cloud with retroreflectors, the intensity points will be between 0 and 255.

#### Intensity mapping for other lidar brands

##### Hesai PandarXT16

[Hesai Pandar XT16 User Manual](https://www.oxts.com/wp-content/uploads/2021/01/Hesai-PandarXT16_User_Manual.pdf)

This lidar has 2 modes for reporting reflectivity:

- Linear mapping
- Non-linear mapping

If you are using linear mapping mode, you should map from [0, 255] to [0, 100] when constructing the point cloud.

If you are using non-linear mapping mode, you should map (hesai to autoware)

- [0, 251] to [0, 100] and
- [252, 254] to [101, 255]

when constructing the point cloud.

##### Livox Mid-70

[Livox Mid-70 User Manual](https://terra-1-g.djicdn.com/65c028cd298f4669a7f0e40e50ba1131/Download/Mid-70/new/Livox%20Mid-70%20User%20Manual_EN_v1.2.pdf)

This lidar has 2 modes for reporting reflectivity similar to Velodyne VLP-16, only the ranges are slightly different.

You should map (livox to autoware)

- [0, 150] to [0, 100] and
- [151, 255] to [101, 255]

when constructing the point cloud.

##### Robosense RS-LiDAR-16

[Robosense RS-LiDAR-16 User Manual](https://cdn.robosense.cn/20200723161715_42428.pdf)

No mapping required, same as Velodyne VLP-16.

##### Ouster OS-1-64

[Software User Manual v2.0.0 for all Ouster sensors](https://data.ouster.io/downloads/software-user-manual/software-user-manual-v2p0.pdf)

In the manual it is stated:

> Reflectivity [16 bit unsigned int] - sensor Signal Photons measurements are scaled based on measured range and sensor sensitivity at that range, providing an indication of target reflectivity. Calibration of this measurement has not currently been rigorously implemented, but this will be updated in a future firmware release.

So it is advised to map the 16 bit reflectivity to [0, 100] range.

##### Leishen CH64W

[I couldnt get the english user manual, link of website](http://www.lslidar.com/en/down)

In a user manual I was able to find it says:

> Byte 7 represents echo strength, and the value range is 0-255. (Echo strength can reflect
> the energy reflection characteristics of the measured object in the actual measurement
> environment. Therefore, the echo strength can be used to distinguish objects with
> different reflection characteristics.)

So it is advised to map the [0, 255] to [0, 100] range.

### Return type

### Channel

The channel field is used to identify the vertical channel of the laser that measured the point.

For Velodyne VLP-16, there are 16 channels. The channels are numbered from 0 to 15, starting from the top of the lidar.

#### Timing of the channels

#### Solid state and petal pattern lidars

### Time
