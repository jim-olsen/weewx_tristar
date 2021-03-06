# Tristar Data Service for WeeWX

This is an implementation for adding tristar charge controller records
to the standard weather record.  It polls the modbus interface of the
tristar controller and pulls the major records and adds them onto
the standard archive weather record as additional fields.

I have also included an implementation of a bootstrap skin modified to
have energy data associated with the weather data.  This ends up providing
a great overview of your charge status.  This is the skin provided by:

https://github.com/brewster76/fuzzy-archer

extended to have graphs and controls related to solar energy charging.

![screenshot image](https://github.com/jim-olsen/weewx_tristar/blob/master/screenshot.png "Screenshot of Daily Energy Scree")

## Installation Instructions
First, you will need to copy the TristarModbusService.py to the user
directory in the standard weewx install location.  This will add the
code necessary to communicate with the tristar.

Additionally, you will need to make several configuration changes in
the weewx.conf file.

First, add a new section to configure the Tristar connection:

```
[Tristar]
        # This section is for configuring the service to append tristar charge
        # controller data to the weather records

        # The charge controller's TCP address for modbus
        address = <tristar's ip address>

        # The modbus port to connect to
        port = 502

```

Now modify the standard schema using our new schema by modifying the
schema line below to match:

```
#   This section binds a data store to a database.

[DataBindings]

    [[wx_binding]]
        # The database must match one of the sections in [Databases].
        # This is likely to be the only option you would want to change.
        database = archive_sqlite
        # The name of the table within the database
        table_name = archive
        # The manager handles aggregation of data for historical summaries
        manager = weewx.wxmanager.WXDaySummaryManager
        # The schema defines the structure of the database.
        # It is *only* used when the database is created.
        schema = user.TristarModbusService.schema_with_tristar
```

Finally, update the Engine section to add our service upon load.  Note the
addition of the user.TristarModbusService.AddTristarData service to the
data_services.

```
#   This section configures the internal weewx engine.

[Engine]

    [[Services]]
        # This section specifies the services that should be run. They are
        # grouped by type, and the order of services within each group
        # determines the order in which the services will be run.
        prep_services = weewx.engine.StdTimeSynch
        data_services = user.TristarModbusService.AddTristarData
        process_services = weewx.engine.StdConvert, weewx.engine.StdCalibrate, weewx.engine.StdQC, weewx.wxservices.StdWXCalculate
        archive_services = weewx.engine.StdArchive
        restful_services = weewx.restx.StdStationRegistry, weewx.restx.StdWunderground, weewx.restx.StdPWSweather, weewx.restx.StdCWOP, weewx.restx.StdWOW, weewx.restx.StdAWEKAS
        report_services = weewx.engine.StdPrint, weewx.engine.StdReport
```

Now you must extend the existing db to include the new fields.  I highly
suggest backing up the database before you do this to a new file.  I am
only including instructions for sqlite here, so if you want to know more
about this process, I suggest looking in the weewx documentation at:

http://weewx.com/docs/customizing.htm#add_archive_type

Change to the directory where your sqlite files are located, and execute
the following commands:

```
wee_database /etc/weewx/weewx.conf --reconfigure
mv weewx.sdb weewx.sdb.old
mv weewx.sdb_new weewx.sdb
```

After you perform this, these new columns will be available:

| Attribute                       | Description                                                | Units |
| ------------------------------- | ---------------------------------------------------------- | ----- |
| battery_voltage                 | The battery voltage used in charge calculations            | Volts |
| battery_sense_voltage           | The battery voltage sensed by the remote sensor wires      | Volts |
| battery_voltage_slow            | The battery voltage used in charge calcs heavily filtered  | Volts | 
| battery_daily_minimum_voltage   | The minimum voltage encountered since last night mode      | Volts |
| battery_daily_maximum_voltage   | The maximum voltage encountered since last night mode      | Volts |
| target_regulation_voltage       | The target voltage to charge the batteries to              | Volts |
| array_voltage                   | The current voltage of the solar array                     | Volts |
| array_charge_current            | The current being produced by the solar panels             | Amps  |
| battery_charge_current          | The current currently used to charge the battery           | Amps  |
| battery_charge_current_slow     | The current charging the battery heavily filtered          | Amps  |
| input_power                     | The current charge power coming in from the array          | Watts |
| output_power                    | The current charge power being sent to the batteries       | Watts |
| heatsink_temperature            | The temperature of the controller heat sink                | Temp  |
| battery_temperature             | The temperature of the batteries                           | Temp  |
| charge_state                    | The current state the charge cycle is in                   | Int   |
| seconds_in_absorption_daily     | The time spent in the absorption cycle state since night   | secs  |
| seconds_in_float_daily          | The time spent in the float cycle state since night        | secs  |
| seconds_in_equalize_daily       | The time spent in the equalize state since night           | secs  |