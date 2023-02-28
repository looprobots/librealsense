
This document aims to list all the topics that realdds object publishes or subscribes to, enabling users to write their own implementations that communicate with realdds applications.

# Topics

Information in realdds domain is being passed using 4 main topics
* **realdds/device-info** of type flexible - The server sends a sample of this topic to inform readers of a connected camera.
  Some of the details included are the *camera model* and *serial number*, this information will be used in the other topic names to ease communication with various cameras in the network.
* **realdds/\<model\>/\<serial\>/notification** of type flexible - Notifications are sent from the camera's server to the camera's user.
  Some notifications are being sent at discovery to convey the full device information, e.g. streams, supported options, etc...
  Other notifications are sent as needed, mostly as reply to a control message
* **realdds/\<model\>/\<serial\>/control** of type flexible - Controls are commands from the camera's user to the camera, e.g. start streaming, set option value etc...
* **realdds/\<model\>/\<serial\>/\<stream name\>** of type image - Frames of the selected stream from the camera's server to the user.

All the topics in the realdds domain use one of these 2 topic types
* **flexible** - Carries a buffer of data that can be JSON, CBOR or other user custom data. See [flexible.idl](https://github.com/IntelRealSense/librealsense/blob/dds/third-party/realdds/include/realdds/topics/flexible/flexible.idl).
* **image** - A buffer of data with some extra fields that describe the image. See [image.idl](https://github.com/IntelRealSense/librealsense/blob/dds/third-party/realdds/include/realdds/topics/image/image.idl).

For instructions about how to generate code from the IDL files see [here](https://github.com/IntelRealSense/librealsense/blob/dds/third-party/realdds/include/realdds/topics/readme.md).

# Messages

## device-info

On the camera's user side, `realdds::dds_device_watcher` can be used to listen to incoming device-info messages.
When compiling librealsense with the cmake `BUILD_WITH_DDS` flag the context automatically creates and uses a `realdds::dds_device_watcher` object to identify DDS connected cameras.

The server side can use `realdds::dds_device_broadcaster` to add or remove cameras from the DDS network.
When a camera is added to the network the broadcaster will create a dds writer for it. If there is a **realdds/device-info** reader that matches the broadcaster writer, topic sample will be sent with the appropriate device details.
When a camera is removed from the network the broadcaster will close the writer, thus informing possible readers that the camera is disconnected.

### device-info message format example

device-info uses the **flexible** type with a JSON format of the following structure

    "locked":false,
    "name":"Intel RealSense D405",
    "product-id":"0B5B",
    "product-line":"D400",
    "serial":"123622270732",
    "topic-root":"realsense/D405/123622270732"

## Notifications

Notifications are messages from the camera to the user. There are several types of notification messages. Some are sent at the discovery stage and some ("instant") are sent as needed, after the initialization is finished.

When a user receives a `device-info` topic he can create a reader for this camera's notifications using the `topic-root` field of the message.
This can also be achieved by using `realdds::dds_device` that will create the reader and wait for discovery notification messages to build appropriate `realdds::dds_stream` objects.

When using librealsense API, received notifications will automatically be handled and reflected in the `rs2::device` and `rs2::sensor` objects state.

All notifications use the **flexible** type with JSON format

### Discovery notifications

On the discovery phase the camera server will send a **device-header** message, a **device-options** message, if applicable[*], and for each stream a **stream-header** and **stream-options** messages, in this order.

[*] Most options affect a stream (e.g. gain, exposure, etc...) not all cameras have options that affect all the streams at once. **device-options** message will be sent only for cameras that support device level options.

#### device-header format example

    "id":"device-header",
    "n-streams": 5        //Integer, number of streams
    "extrinsics":         //Array of extrinsics data, describing transformation between different streams in the device
    [
        ["Depth","Gyro",[1.0,0.0,0.0,0.0,1.0,0.0,0.0,0.0,1.0,-0.005520000122487545,0.005100000184029341,0.011739999987185001]]
        ["Depth","Infrared 2",[1.0,0.0,0.0,0.0,1.0,0.0,0.0,0.0,1.0,-0.04986396059393883,0.0,0.0]]
    ]

#### device-options format example

    "id":"device-options",
    "n-options": 2,       //Integer, number of device options
    "options":            //Array of supported options, the size of n-options
    [
        {"description":"Enable/Disable global timestamp","name":"Global Time Enabled","range-default":1.0,"range-max":1.0,"range-min":0.0,"range-step":1.0,"value":1.0},
        {"description":"HDR Option","name":"Hdr Enabled","range-default":0.0,"range-max":1.0,"range-min":0.0,"range-step":1.0,"value":0.0}
    ]

#### stream-header format example

    "default-profile-index":0,
    "id":"stream-header",
    "name":"Infrared 1",
    "profiles":           //All stream profiles. Each profile is an array of frequency, format, width, height.
    [
        [25,"Y16",1288,808],[15,"Y16",1288,808],[30,"GREY",1280,720],[15,"GREY",1280,720],[5,"GREY",1280,720],[90,"GREY",848,480],[60,"GREY",848,480],[30,"GREY",848,480],[15,"GREY",848,480],[5,"GREY",848,480],[90,"GREY",640,480],[60,"GREY",640,480],[30,"GREY",640,480],[15,"GREY",640,480],[5,"GREY",640,480],[90,"GREY",640,360],[60,"GREY",640,360],[30,"GREY",640,360],[15,"GREY",640,360],[5,"GREY",640,360],[90,"GREY",480,270],[60,"GREY",480,270],[30,"GREY",480,270],[15,"GREY",480,270],[5,"GREY",480,270],[90,"GREY",424,240],[60,"GREY",424,240],[30,"GREY",424,240],[15,"GREY",424,240],[5,"GREY",424,240]
    ],
    "sensor-name":"Stereo Module",
    "type":"ir"

#### stream-options format example

    "id":"stream-options",
    "intrinsics":         //Array of intrinsics data, describing internal properties of the stream.
    [
        [640,480,320.14276123046875,238.4058837890625,378.80572509765625,378.80572509765625,4,0.0,0.0,0.0,0.0,0.0]
        [1280,720,640.2379150390625,357.3431396484375,631.3428955078125,631.3428955078125,4,0.0,0.0,0.0,0.0,0.0]
    ]
    "n-options":3,        //Integer, number of device options
    "options":            //Array of supported options, the size of n-options
    [
        {"description":"Enable / disable backlight compensation","name":"Backlight Compensation","range-default":0.0,"range-max":1.0,"range-min":0.0,"range-step":1.0,"value":0.0},
        {"description":"UVC image brightness","name":"Brightness","range-default":0.0,"range-max":64.0,"range-min":-64.0,"range-step":1.0,"value":0.0},
        {"description":"UVC image contrast","name":"Contrast","range-default":50.0,"range-max":100.0,"range-min":0.0,"range-step":1.0,"value":50.0}
    ],
    "stream-name":"Infrared 1"

### Instant notifications

Other notifications are sent as needed, as a reply to **set-option** or **query-option**

#### query-option reply format example

    "id":"query-option",
    "counter": 123,       //Integer, identical to the set-option command counter
    "value": 12.34,       //Float
    "successful":false,
    "failure-reason":""

#### set-option reply format example

    "id":"set-option",
    "counter": 123,       //Integer, identical to the set-option command counter
    "successful":false,
    "failure-reason":""

## Controls

Controls are commands from the user to the camera.
Using controls, the user can open or close streaming, or he can query current option values or set new values affecting the camera behaviour.

When using librealsense API commands will automatically be converted to the appropriate commands that will be sent to the camera over the DDS network.
For instance, using `rs2::sensor.open( profile )` will be converted to DDS **open-streams** command using a control topic.

### open-streams command format example

    "id":"open-streams",
    "stream-profiles":{"Color":[30,"RGB8",848,480],"Depth":[30,"Z16",848,480]} //A list of streams to open and the requested profile for each one

### close-streams command format example

    "id":"close-streams",
    "stream-names":["Depth","Color"] //A list of streams to close

### query-option command format example

    "id":"query-option",
    "counter":107,        //Integer, increasing counter to uniquely identify this message and match a reply to it
    "option-name":"Auto Gain Limit Toggle",
    "owner-name":"Depth"

### set-option command format example

    "id":"set-option",
    "counter":108,        //Integer, increasing counter to uniquely identify this message and match a reply to it
    "option-name":"Enable Auto Exposure",
    "owner-name":"Depth",
    "value":0.0

