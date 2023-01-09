Tuya-MCU
========

.. seo::
    :description: Instructions for setting up the Tuya component.
    :image: tuya.png

The ``tuya`` component creates a serial connection to a Tuya-MCU for platforms to use.

.. figure:: /images/tuya.png
    :align: center
    :width: 40%

This component requires a :ref:`UART bus <uart>` to be configured.
Put the ``tuya`` component in the config and it will list the possible devices for you in the config log.

.. code-block:: yaml

    # Make sure logging is not using the serial port
    logger:
      baud_rate: 0

    uart:
      rx_pin: GPIO3
      tx_pin: GPIO1
      baud_rate: 9600

    # Register the Tuya MCU connection
    tuya:

Here is an example output for a Tuya ME82BH thermostat, other devices will produce similar results:

.. code-block:: text


    [19:05:45][C][tuya:059]: Tuya:
    [19:05:45][C][tuya:078]:   Datapoint  17 (0x11): int value    (value: 70)
    [19:05:45][C][tuya:083]:   Datapoint  23 (0x17): enum         (value: 0)
    [19:05:45][C][tuya:075]:   Datapoint   1 (0x01): switch       (value: ON)
    [19:05:45][C][tuya:078]:   Datapoint  24 (0x18): int value    (value: 20)
    [19:05:45][C][tuya:083]:   Datapoint   2 (0x02): enum         (value: 1)
    [19:05:45][C][tuya:078]:   Datapoint  19 (0x13): int value    (value: 40)
    [19:05:45][C][tuya:078]:   Datapoint  26 (0x1A): int value    (value: 10)
    [19:05:45][C][tuya:078]:   Datapoint 106 (0x6A): int value    (value: 1)
    [19:05:45][C][tuya:078]:   Datapoint  27 (0x1B): int value    (value: 0)
    [19:05:45][C][tuya:083]:   Datapoint  43 (0x2B): enum         (value: 0)
    [19:05:45][C][tuya:085]:   Datapoint  45 (0x2D): bitmask      (value: 0)
    [19:05:45][C][tuya:083]:   Datapoint 104 (0x68): enum         (value: 3)
    [19:05:45][C][tuya:075]:   Datapoint 103 (0x67): switch       (value: OFF)
    [19:05:45][C][tuya:091]:   GPIO Configuration: status: pin 2, reset: pin 17
    [19:05:45][C][tuya:094]:   Product: '{"p":"psssxcwgbx6m9apv","v":"1.0.0","m":0}'

Configuration variables:
------------------------

- **time_id** (*Optional*, :ref:`config-id`): Some Tuya devices support obtaining local time from ESPHome.
  Specify the ID of the :doc:`time/index` which will be used.

- **status_pin** (*Optional*, :ref:`Pin Schema <config-pin_schema>`): Defines the pin to use if the Tuya-MCU requires WiFi status updates via GPIO.
  Not all devices require this - See :ref:`tuya-status_pin_configuration`.

- **ignore_mcu_update_on_datapoints** (*Optional*, list): A list of datapoints to ignore Tuya-MCU updates for. Useful for certain broken/erratic hardware and debugging.

- **status_mode** (*Optional*): One of ``auto``, ``manual``. 
  Whether to send WiFi status updates to the Tuya-MCU automatically, or manually by use of Lambda calls (See :ref:`tuya-lambda_calls`).
  Defaults to ``auto``.

- **minute_sync** (*Optional*, boolean): If set to ``true``, the time is only sent to the Tuya-MCU when time-seconds rolls over from 59 to 0 seconds. This is required by some Tuya-MCU products that ignore the 'seconds' element of the time sent to the device. Setting to ``false`` will send the time to the Tuya-MCU directly following a request from the Tuya-MCU for a time update, or when the ESP ``time`` component is periodically re-synchronized with the time server. See :doc:`time/index`. Defaults to ``false``.

Advanced Options:

- **command_delay** (*Optional*, int): Delay time between sending of messages to the Tuya-MCU. Generally not required to be modified. Can be increased from default value if the MCU fails to respond to some messages. Defaults to ``10``.

- **receive_timeout** (*Optional*, int): Timeout between sending a message and receiving a response from the Tuya-MCU. Generally not required to be modified. Can be increased from default value if the MCU fails to respond to some messages. Defaults to ``300``.

- **dbg_suppress_dp_update_msgs** (*Optional*, boolean): Allows suppressing received datapoint messages in the debug logs. Useful to reduce 'debug' log-spam each time a datapoint set is received (every 10 secs). Defaults to ``false``.


.. _tuya-status_pin_configuration:

Status Pin Configuration
************************

Some Tuya products support WiFi status reporting **ONLY** through a GPIO pin.
More about this in the `Tuya Serial Port Specification - Query Working Mode <https://developer.tuya.com/en/docs/iot/tuya-cloud-universal-serial-port-access-protocol?id=K9hhi0xxtn9cb#title-7-Query%20working%20mode>`__.

During initialization of the ``tuya`` component, the config log will indicate the OEM's intended WiFi module GPIO pin to be used for this purpose.

Example config log output from a device that requires WiFi status to be sent via GPIO:

.. code-block:: text

    [19:05:45][C][tuya:091]:   GPIO Configuration: status: pin 2, reset: pin 17

If the log ``status: pin`` value is ``-1``:

-  ``status_pin`` should be omitted from the configuration. It is not required: WiFi status will be handled by the API.

If the log ``status: pin`` is any other value:

-  The Tuya-MCU is expecting a connected / disconnected boolean state indication via the ``status_pin`` GPIO:

  - If the WiFi module is a **Tuya** ESP-based WiFi module: set ``status_pin`` to the value shown in the logs.

  - If the WiFi module is **NOT** a **Tuya** ESP-based WiFi module (e.g. DIY replacement): see :ref:`tuya-diy_module_replacement_status_pin_configuration`.

.. _tuya-diy_module_replacement_status_pin_configuration:

DIY Module Replacement: Status Pin Configuration
************************************************

.. note::

  Some non-ESP Tuya WiFi modules have pin-compatible ESP-based WiFi modules available from other vendors, e.g.
  the `Ai-Thinker ESP8266MOD (ESP-12F) <https://docs.ai-thinker.com/_media/esp8266/docs/esp-12f_product_specification_en.pdf>`__
  and `Tuya (non-ESP) WBR3 <https://developer.tuya.com/en/docs/iot/wbr3-module-datasheet?id=K9dujs2k5nriy#title-4-Dimensions%20and%20footprint>`__ WiFi modules
  are pin-compatible and hence a Tuya product that has a non-ESP based Tuya WiFi module fitted (e.g. Tuya-WBR3) can be modified to contain an ESP WiFi module,
  therefore allowing ESPHome to be used on the device.

If (as above) the Tuya WiFi module was replaced with an ESP WiFi module, the GPIO number (``2`` in this example) will *NOT* be the correct GPIO
to use on the replacement ESP WiFi module.

To find the correct ESP-specific GPIO number to use in this situation, match the given ``status: pin`` from the Tuya WiFi module to the same physical
location on the equivalent ESP WiFi module. 

Example scenario:

- A product contained a Tuya-WBR3 WiFi module.

- It was replaced with a ESP8266MOD WiFi module and flashed with ESPHome.

- The Tuya-WBR3 and ESP8266MOD WiFi modules are pin-compatible, e.g. the pin in the same physical location.

- The MCU reports pin ``2`` which is ``A_2`` on the Tuya-WBR3.

- Using the datasheets to match the pinouts, ``status_pin`` should be set to GPIO ``14``.

.. figure:: /images/tuya_wbr3_vs_esp8266_pinout.png
  :align: center
  :width: 80%

  Pinout comparison: (Top) ESP-12F vs (Bottom) Tuya-WBR3


.. _tuya-automations:

Automations
-----------

- **on_datapoint_update** (*Optional*): An automation to perform when a Tuya datapoint update is received. See :ref:`tuya-on_datapoint_update`.

.. _tuya-on_datapoint_update:

``on_datapoint_update``
***********************

This automation will be triggered when a a Tuya datapoint update is received.
A variable ``x`` is passed to the automation for use in lambdas.
The type of ``x`` variable is depending on ``datapoint_type`` configuration variable:

- *raw*: ``x`` is ``std::vector<uint8_t>``
- *string*: ``x`` is ``std::string``
- *bool*: ``x`` is ``bool``
- *int*: ``x`` is ``int``
- *uint*: ``x`` is ``uint32_t``
- *enum*: ``x`` is ``uint8_t``
- *bitmask*: ``x`` is ``uint32_t``
- *any*: ``x`` is :apistruct:`tuya::TuyaDatapoint`

.. code-block:: yaml

    tuya:
      on_datapoint_update:
        - sensor_datapoint: 6
          datapoint_type: raw
          then:
            - lambda: |-
                ESP_LOGD("main", "on_datapoint_update %s", format_hex_pretty(x).c_str());
                id(voltage).publish_state((x[0] << 8 | x[1]) * 0.1);
                id(current).publish_state((x[3] << 8 | x[4]) * 0.001);
                id(power).publish_state((x[6] << 8 | x[7]) * 0.1);
        - sensor_datapoint: 7 # sample dp
          datapoint_type: string
          then:
            - lambda: |-
                ESP_LOGD("main", "on_datapoint_update %s", x.c_str());
        - sensor_datapoint: 8 # sample dp
          datapoint_type: bool
          then:
            - lambda: |-
                ESP_LOGD("main", "on_datapoint_update %s", ONOFF(x));
        - sensor_datapoint: 6
          datapoint_type: any # this is optional
          then:
            - lambda: |-
                if (x.type == tuya::TuyaDatapointType::RAW) {
                  ESP_LOGD("main", "on_datapoint_update %s", format_hex_pretty(x.value_raw).c_str());
                } else {
                  ESP_LOGD("main", "on_datapoint_update %hhu", x.type);
                }

Configuration variables

- **sensor_datapoint** (**Required**, int): The datapoint id number of the sensor.
- **datapoint_type** (**Required**, string): The datapoint type one of *raw*, *string*, *bool*, *int*, *uint*, *enum*, *bitmask* or *any*.
- See :ref:`Automation <automation>`.

.. _tuya-lambda_calls:

Lambda Calls
------------

Using :ref:`config-lambda`, you can call class methods to perform advanced functionality.

.. note:: 
  Only the members listed below are supported. Calling any other tuya class member function is not supported and may cause unpredictable results.

- **.set_status_update_mode_manual(void)**: WiFi status will not be sent to the Tuya MCU when the network status changes. The user can control the WiFi status message by calling ``force_wifi_status(*int state*)``.

- **.set_status_update_mode_auto(void)**: WiFi status messages reflect the current network status.

- **.force_wifi_status(int state)**: Sends a user-defined WiFi status update message to the Tuya-MCU. 
  Use to inform the Tuya-MCU of the network state. E.g. Can be used to toggle the WiFi icon (if the device has one) on and off etc.
  Note that the configured ``status_mode`` will be changed to ``manual`` when this Lambda is called.
  ``state`` can be 0 (no network) or 1 (network connected) when ``status_pin`` is defined, otherwise valid values are 0 to 5.
  See `Tuya Serial Port Specification - Report Network Status <https://developer.tuya.com/en/docs/iot/tuya-cloud-universal-serial-port-access-protocol?id=K9hhi0xxtn9cb#title-9-Report%20network%20status>`__ for further details.

- **.send_default_time(void)** : Sets the device date/time to ``2020:01:01 00:00:00`` (which was a Saturday). Useful for debugging.


See Also
--------

- :doc:`/components/fan/tuya`
- :doc:`/components/light/tuya`
- :doc:`/components/switch/tuya`
- :doc:`/components/climate/tuya`
- :doc:`/components/binary_sensor/tuya`
- :doc:`/components/sensor/tuya`
- :doc:`/components/text_sensor/tuya`
- :apiref:`tuya/tuya.h`
- :ghedit:`Edit`
