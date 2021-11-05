# ![](favicon.png) Xiaomi Miot Serial Communication

Translated and converted to .md from [Xiaomi IoT Developer Platform](https://iot.mi.com/new/doc/embedded-development/wifi/module-dev/serial-communication)

Serial command rules
--------------------

When developers develop product firmware functions on the hardware product MCU, they need to design instructions according to the following rules:

*   Serial port configuration: 115200,8,1,N,N, does not support changes (in order to fully use the serial port command, please upgrade the module's firmware to 2.xx);
*   Command length: The total length must be less than **512** characters, and the serial port processing capability of the MCU must support the processing of commands with more than 512 bytes;
*   Command format: The first continuous string is the command name, followed by several parameters, cmd\_name \[ cmd\_params \] ... \[ cmd\_params \] '\\r';
*   Separator: Serial commands are separated by spaces, and spaces in non-character strings need to be escaped into \\u0020;
*   End character: carriage return character'\\r' (ie CR or 0x0D);
*   Instructions and parameters should be composed of legal characters, including letters, numbers, and underscores;
*   The local configuration command does not have a string type, and there is no need to add extra quotes \[ "" \] to the parameters ;
*   The communication command of the server is based on JSON, and the string type in the communication command parameter of the server needs to add quotation marks \[ "" \] ;

> **Note:** If a developer encounters an error when developing product firmware using serial port commands, please refer to the document ["Wi-Fi Product Development FAQ"](/new/doc/embedded-development/wifi/FAQ) to solve the problem.

Basic function instructions
---------------------------

model
-----

After a developer creates a product on the Xiaomi IoT platform, he can get the product Model. The Model string consists of the company name, product category name, and product version, and the total length cannot exceed 23 characters.

> **Note: After the** hardware product MCU is powered on, the product model should be reported to the Xiaomi IoT platform as soon as possible. The Xiaomi IoT platform will bind the module and the product model. Therefore, the developer should correctly configure the product model and the same module Do not reuse different models

*   Parameters: none or \<model\_name\_string>
*   return value:
    *   When there are no parameters, return the current model string;
    *   With parameters, set the chip model to the parameter string and return ok;
    *   If the parameter string is invalid, error is returned.

```      
    ↑model   
    ↓xiaomi.dev.v1   
    ↑model xiaomi.prod.v2  
    ↓ok   
    ↑model   
    ↓xiaomi.prod.v2   
    ↑model company.product_name_that_is_too_long.v2   
    ↓error   
    ↑model plug   
    ↓error  
```

mcu\_version
------------

MCU should call this command to report the version immediately after the application firmware is booted, and ensure that the setting is successful and the module responds "ok"; if the downstream command receives MIIO\_mcu\_version\_req, it should also report the MCU firmware version immediately, otherwise the product will fail Acceptance Test.

*   Parameters: \<mcu\_version\_string>
*   Return value: If the parameter is legal, it returns ok, if it is illegal, it returns error.

> **Note: The** MCU firmware version must be a 4-digit number.

    
        ↑mcu_version 0001  
        ↓ok  
        ↑mcu_version A001                                   
        ↓error  
        ↑mcu_version 1  
        ↓error 

ble\_config (only dual-mode modules need to use this command)
-------------------------------------------------------------

Set and view the module PID and ble firmware version number.

*   Parameters: set \<pid> \<ble\_version> or dump.
*   Return value: The command is ble\_config set \<pid> \<ble\_version> is used to set the Bluetooth PID of the dual-mode module (using decimal; for Bluetooth network configuration, PID is less than 65536), ble\_version must be consistent with the MCU version; if the command is ble\_config , Dump returns the current PID and ble firmware version of the module, otherwise it returns ok/error.
      
```
    ↑ ble_config set 156 0001                          
    ↓ ok  
    ↑ ble_config dump   
    ↓ ["product id":190,"version":1.3.0_0000]           
```      

Server communication instructions
---------------------------------

get\_down
---------

Get the downstream instructions.

*   Parameters: none
*   Return value：down \<method\_name> \<arg1> \<arg2> \<arg3> ...

*   If there is no down command, it will immediately return to down none;
*   If there is a downstream command, it will immediately return the command name and command parameters (if there are parameters); the command name and the parameters are separated by spaces, and the parameters are separated by spaces; the parameters can be a string enclosed in double quotation marks or a number ; The returned command name may be get\_properties, set\_properties or action, corresponding to the corresponding device function definition, and the MCU needs to process the corresponding operation according to the device instance definition.
*   After the application firmware is booted, the MCU should call this command cyclically to obtain whether there is a command (the time range is 100~200ms, the recommended time interval is 200ms). If the MCU receives a downlink command, it should also respond immediately. For the command, the developer needs to use the error response module, otherwise the product will fail the acceptance test.

    
        ↑get_down                                          
        ↓down none  
        ↑get_down   
        ↓down set_properties 1 1 10   
        ↑result 1 1 0   
        ↓ok   
        ↑get_down   
        ↓down action 1 1   
        ↑get_down   
        ↓error  
      

result
------

Return the corresponding execution result according to different downstream commands

*   Parameters: \<value1> \<value2> ...
*   Return value: Return ok on success, error on failure.

> **Note:** Up to 64 parameters are used after the instruction, and the entire instruction is up to 512 bytes.

    
        ↑get_down   
        ↓down set_properties 1 1 10                     
        ↑result 1 1 0   
        ↓ok  
      

error
-----

*   Parameters：<message> \<error\_code>
*   Return value: Return ok on success, error on failure.

> **Note:** If the downstream command is executed incorrectly or receives an unrecognized command, the MCU can use this command to send error information. After sending successfully, return to MIIO chip to reply ok. The message must be a string enclosed in double quotes, and the error code must be an integer between -9999 and -5000 (including boundary values). The error message and error code can be customized.

    
        ↑get_down  
        ↓down error_cmd   
        ↑error "memory error" -5003                        
        ↓ok   
        ↑error "stuck" -5001   
        ↓ok 
      

properties\_changed
-------------------

Used to report attributes, the parameter must have at least a set of attribute iid and value.

*   format：properties\_changed \<siid> \<piid> \<value> ... \<siid> \<piid> \<value>
*   Return value: ok or error.

> **Note:** properties\_changed is updated once per second, and the property with the same name within one second is only the last one reported to the Xiaomi IoT platform; when there is a corresponding property change locally, this command is called, and a maximum of 64 parameters are used after the command (the following example uses 6 Parameter), the entire instruction is up to 512 bytes.

    
        ↑properties_changed 1 1 17 1 2  "hi"               
        ↓ok   
        ↑properties_changed 1 1      
        ↓error   
      

> **Note:** If the device needs to report SN, the developer can use the Spec definition in the product function definition (siid=1 piid=5), when the device is connected to the network, that is, net=cloud, properties\_changed report the device SN code.
 
>     // Example
>     ↑get_down
>     ↓down MIIO_net_change cloud
>     // The format of SN needs to conform to the definition of String in spec, and the initial length should be 30 bytes
>     ↑properties_changed 1 5 “12345/A026Z00001”
>     ↓ok

event\_occured
--------------

Used to report events

*   format：event\_occured \<siid> \<eiid> \<piid> \<value> ... \<piid> \<value>
*   Return value: ok or error.

> **illustrate:**   
> When the corresponding event changes locally, call this command to report the event event, event\_occured is not more than 3 times per second;
> *   Compared with properties\_changed, the information when an event occurs will be updated to the Xiaomi IoT platform immediately. The properties\_changed is updated every second, and the property with the same name within one second is only the last one reported to the Xiaomi IoT platform;
> *   Up to 64 parameters are used after the instruction (6 parameters are used in the following example), and the entire instruction is up to 512 bytes.

    
        ↑event_occured 1 1 1 17 2  "hi"                    
        ↓ok   
        ↑event_occured 1       
        ↓error   
      

Serial tool debugging instructions
----------------------------------

echo
----

*   Parameters: on or off
*   Return value: ok.

> **illustrate:**   
> Turn on the serial port echo function. When this function is enabled, the serial port output of the MIIO chip will echo the input;
> *   When serial port debugging is performed through the serial port tool, normal debugging may not be possible if the echo function is not turned on.
>     

Control instruction
-------------------

reboot
------

*   Parameters: none
*   Return value: ok.

> **Note: After the** module receives this command, it will restart within 0.5 seconds.

    
        ↑reboot                             
        ↓ok  
    

restore
-------

After the Xiaomi IoT module receives the command, the module will clear the Wi-Fi network configuration and remote control related information, and restart within 0.5s.

*   Parameters: none
*   Return value: ok

```    
    ↑restore                                          
    ↓ok  
```      

factory
-------

After receiving the instruction, the MIIO chip enters the factory test mode within 2 seconds.

*   Parameters: none
*   Return value: ok

> **illustrate:**
> In this mode, the chip will connect to the router according to the preset information;
> *   Default router SSID: miio\_default, password: 0x82562647;
> *   The factory mode needs to ensure that it can only be triggered in the factory, and the device network status is local and connected to the router;
> *   You can use reboot/restore to exit factory mode.

    
        ↑factory                                
        ↓ok  
      

Get module information command
------------------------------

net
---

Ask about the network status.

*   Parameters: none
*   Return value: offline or local or cloud or updating or uap or unprov

> **illustrate**:
> offline-connecting (or offline);
> *   local-connected to the router but not connected to the Mi Cloud Server;
> *   updating-firmware upgrading, uap mode waiting for connection, unprov-close wifi (not connected quickly in half an hour).

    
        ↑net                                             
        ↓offline   
        ↑net   
        ↓local  
        ↑net  
        ↓cloud  
      

time
----

*   Parameters: none or posix
*   Return value: current date and time

```   
    ↑time
    ↓2015-06-04 16:58:07
    ↑time posix
    ↓1434446397
```   
      

mac
---

Get the mac address of the current module.

*   Parameters: none
*   Return value: the mac address of the current module

```    
    ↑mac               
    ↓34ce00892ab7      
```      

version
-------

*   Parameters: none
*   Return value: Module firmware version number

> **Note:** Some modules return the version of the serial port protocol (this protocol), and the firmware version of the module after the 2.xx version.

```      
    ↑version                                            
    ↓2.0.0  
```   
      

help
----

*   Parameters: none
*   Return value: all supported commands and parameter formats

Reserved downstream serial port commands
----------------------------------------

The MIIO chip reserves part of the command name to notify the hardware product MCU of specific events. This type of command can be obtained with get\_down, but the MCU in the hardware product needs to be processed accordingly.

set\_properties
---------------

Serial RPC Downlink Related Commands (Controlled by Xiaomi IoT Platform)

*   format： set\_properties \<siid> \<piid> \<value> ... \<siid> \<piid> \<value>
*   Return value： result \<siid> \<piid> \<code> ... \<siid> \<piid>
\<code> 

The code value can refer to the "code status code definition" at the end of this document

> **Note:** The set property of Xiaomi IoT platform needs to be processed by the MCU and the result is returned immediately. The general RPC communication of the module is 4S timeout failure. The reply result command uses up to 64 parameters (the following example uses 6 parameters), and the entire command is up to 512 words Section, otherwise the product will fail the acceptance test.

```    
    ↑ get_down  
    ↓ down set_properties 1 1 10 1 88  "str_value"       
    ↑ result 1 1 0 1 88 -4003
    ↑properties_changed 1 1 10
    ↓ok
```      

get\_properties
---------------

*   format：get\_properties \<siid> \<piid> ... \<siid> \<piid>
*   Return value：result \<siid> \<piid> \<code> \[value\] ... \<siid> \<piid> \<code> \[value\]

> **illustrate:**   
> For the specific content of the code value, please refer to the "code status code definition" at the end of this document;
> *   The get property of the Xiaomi IoT platform needs to be processed by the MCU and the result is returned immediately. The general RPC communication of the module is a 4S timeout failure. The reply result command uses up to 64 parameters (the following example uses 8 parameters), and the entire command is up to 512 bytes.

    
        ↑ get_down  
        ↓ down get_properties 1 2 1 3                        
        ↑ result 1 2 0 10 1 3 0 "str_value"
      

action
------

*   format：action \<siid> \<aiid> \<piid> \<value> ... \<piid> \<value>
*   Return value：result \<siid> \<aiid> \<code> \<piid> \<value> ...
\<piid> \<value>

> **illustrate:**   For the specific content of the code value, please refer to the "code status code definition" at the end of this document;
> *   The Xiaomi IoT platform executes Action, which needs to be processed by the MCU and returns the result immediately. The general RPC communication of the module is a 4s timeout failure. The command uses up to 64 parameters (the following example uses 5 parameters), and the entire command is up to 512 bytes.
>     

    
        ↑ get_down  
        ↓ down action 1 1 1 10                              
        ↑ result 1 1 0 3 10
      

MIIO\_net\_change
-----------------

The network connection status of the Xiaomi IoT module has changed.  
**Return value: The** parameter represents the latest network status of the chip. For detailed description of the network status, please refer to the net command. The developer can change the state of the Wi-Fi status light according to this command.

    
        ↑get_down  
        ↓down MIIO_net_change cloud                          
      

update\_fw
----------

Upgrade the MCU in the hardware product through the serial port. Different platforms have different limitations on the size of the MCU firmware (ESP(32)-WROOM-32D(32U) module MCU firmware is not greater than 1MB, MHCWB4P module MCU firmware is not greater than 236KB), details Please refer to the serial OTA document for introduction.

**return value:**

*   mcu can return "ready" and "busy";
*   Return "ready" to enter the normal upgrade process;
*   "Busy" is returned, indicating that the current status of the MCU cannot be upgraded, and the upgrade failure is returned to the server.

```    
    ↑get_down  
    ↓down update_fw  
    ↑result "ready"  
    ↓ok  
    Enter xmodem to start downloading                                    
```      

status code
----------------

 code | explanation
---|---
0 | success
1 | The request was received, but the operation has not been completed
\-4001 | Attribute is not readable
\-4002 | Property is not writable
\-4003 | Properties, methods, and events do not exist
\-4004 | Other internal errors
\-4005 | Attribute value error
\-4006 | Method in parameter error
\-4007 | did error