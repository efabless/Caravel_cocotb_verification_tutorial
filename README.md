# Caravel cocotb verification tutorial

This tutorial will show you how to use cocotb Caravel APIs in order to create a testbench which sets Caravel gpios to have a certain value. To do this, you first need to write the firmware which will run on the mangament SoC and you have to set the gpios mode to be management output and then monitor those gpios using python cocotb testbench and check them against the expected value.  

## The steps: 

### 1. Install prerequisites:
Make sure you followed the [quickstart guide](https://caravel-sim-infrastructure.readthedocs.io/en/latest/usage.html#quickstart-guide) to install the prerequisites and cloned the [Caravel cocotb simulation infrastructure repo](https://github.com/efabless/caravel-sim-infrastructure) 
### 2. Update design_info.yaml file:
Make sure you updated the paths inside the ``design_info.yaml`` to match your paths as shown [here](https://github.com/efabless/caravel-sim-infrastructure/tree/main/cocotb#configure-the-repo). You can find the file in the ```caravel-sim-infrastructure/cocotb/``` directory
### 3. Create the firmware program:
The firmware is written in C code and it is the program that will be running on the Caravel management SoC. You can use it to make any configurations you want. You can find a description for all the firmware C APIs [here](https://caravel-sim-infrastructure.readthedocs.io/en/latest/C_api.html#firmware-apis)
For example, you can use it to configure the GPIO pins to have a certain value as shown in the code below:
```
#include <common.h> 
void main(){
  mgmt_gpio_o_enable();
  mgmt_gpio_wr(0);
  enable_hk_spi(0); 
  configure_all_gpios(GPIO_MODE_MGMT_STD_OUT);
  gpio_config_load();
  set_gpio_l(0x8F);
  mgmt_gpio_wr(1);
  return;
}
```
* ``#include <common.h>``  is used to include the firmware APIs. This must be included in any firmware that will use the APIs provided. 
* ``mgmt_gpio_o_enable();`` is a function used to set the management gpio to output (this is a single gpio pin inside used by the management soc). You can read more about this function [here](https://caravel-sim-infrastructure.readthedocs.io/en/latest/C_api.html#_CPPv418mgmt_gpio_o_enablev). 
* ``mgmt_gpio_wr(0);`` is a function to set the management gpio pin to a certain value. Here I am setting it to 0 and later will set it to 1 after the configurations are finished. This is to make sure in the python testbench that the configurations are done and you can begin to check the gpios value. You can read more about this function [here](https://caravel-sim-infrastructure.readthedocs.io/en/latest/C_api.html#_CPPv412mgmt_gpio_wrb). 
* ``enable_hk_spi(0);`` is used to disable housekeeping spi and this is required for gpio 3 to function as a normal gpio.  
* ``configure_all_gpios(GPIO_MODE_MGMT_STD_OUTPUT);`` is a function used to configure all caravel’s 38 gpio pins with a certain mode. Here I chose the ``GPIO_MODE_MGMT_STD_OUTPUT`` mode because I will use the gpios as output and the management SoC will be the one using the gpios not the user project. You can read more about this function [here](https://caravel-sim-infrastructure.readthedocs.io/en/latest/C_api.html#_CPPv419configure_all_gpios9gpio_mode). 
* ``gpio_config_load();`` is a function to load the gpios configuration. It must be called whenever we change gpio configuration. 
* ``set_gpio_l(0x8F);`` is a function used to set the value of the lower 32 gpios with a certain value. In this example the first 4 gpios and the 8th gpio will be set to 1 and the rest will be set to 0. you can read more about this function [here](https://caravel-sim-infrastructure.readthedocs.io/en/latest/C_api.html#_CPPv410set_gpio_lj). 
* ``mgmt_gpio_wr(1);`` is a function to set the management gpio to 1 to indicate configurations are done as explained above.

### 4. Create the python testbench:
The python testbench is used to monitor the signals of the Caravel chip just like the testbenches used in hardware simulators. 
Continuing on the example above,  if we want to check whether the gpios are set to the correct value, we can do that using the following code:

```
from cocotb_includes import * 
import cocotb

@cocotb.test() 
@repot_test 
async def gpio_test(dut):
   caravelEnv = await test_configure(dut)
   await caravelEnv.release_csb()
   await caravelEnv.wait_mgmt_gpio(1)
   gpios_value_str = caravelEnv.monitor_gpio(37, 0).binstr
   cocotb.log.info (f"All gpios '{gpios_value_str}'")
   gpio_value_int = caravelEnv.monitor_gpio(37, 0).integer
   expected_gpio_value = 0xF8
   if (gpio_value_int == expected_gpio_value):
      cocotb.log.info (f"[TEST] Pass the gpio value is '{hex(gpio_value_int)}'")
   else:
      cocotb.log.error (f"[TEST] Fail the gpio value is :'{hex(gpio_value_int)}' expected {hex(expected_gpio_value)}")
```
* ``from cocotb_includes import *`` is to include the python APIs for Caravel. It must be included in any python testbench you create 
* ``import cocotb`` is to import cocotb library 
* ``@cocotb.test()`` is a function wrapper which must be used before any cocotb test. You can read more about it [here](https://docs.cocotb.org/en/stable/quickstart.html#creating-a-test)
* ``@repot_test `` is a function wrapper which is used to configure the test reports
* ``async def gpio_test(dut):``  is to define the test function. The async keyword is the syntax used to define python coroutine function (a function which can run in the background and does need to complete executing in order to return to the caller function). You must name this function the same name you will give the test in the ``-test`` argument while running. Here for example I used ``gpio_test`` for both. 
* ``caravelEnv = await test_configure(dut)`` is used to set up what is needed for the caravel testing environment such as reset, clock, and timeout cycles.This function must be called before any test as it returns an object with type Cravel_env which has the functions we can use to monitor different Caravel signals.
* ``await caravelEnv.release_csb()`` is to release housekeeping spi. By default, the csb is gpio value is 1 in order to disable housekeeping spi. This function drives csb gpio pin to Z to enable using it as output.  
* ``await caravelEnv.wait_mgmt_gpio(1)`` is to wait until the management gpio is 1 to ensure that all the configurations done in the firmware are finished. The ``await`` keyword is used to stop the execution of the coroutine until it returns the results. You can read more about the function [here](https://caravel-sim-infrastructure.readthedocs.io/en/latest/python_api.html#interfaces.caravel.Caravel_env.wait_mgmt_gpio)
* ``gpios_value_str = caravelEnv.monitor_gpio(37, 0).binstr`` is used to get the value of the gpios. The monitor_gpio() function takes the gpio number or range as a tuple and returns a [BinaryValue](https://docs.cocotb.org/en/stable/library_reference.html#cocotb.binary.BinaryValue) object. You can read more about the function [here](https://caravel-sim-infrastructure.readthedocs.io/en/latest/python_api.html#interfaces.caravel.Caravel_env.monitor_gpio). One of the functions   of the BinaryValue object is binstr which returns the binary value as a string (string consists of 0s and 1s)
* ``cocotb.log.info (f"All gpios '{gpios_value_str}'")``will print the given string to the full.log file which can be useful to check what went wrong if the test fails
* ``gpio_value_int = caravelEnv.monitor_gpio(37, 0).integer`` will return the value of the gpios as an integer
* ``` 
  if (gpio_value_int == expected_gpio_value):
      cocotb.log.info (f"[TEST] Pass the gpio value is '{gpio_value_int}'")
   else:
      cocotb.log.error (f"[TEST] Fail the gpio value is :'{gpio_value_int}' expected {expected_gpio_value}")
   ```
   This compares the gpio value with the expected value and print a string to the log file if they are equal and raises an error if they are not equal. 

### 5. Run the test:
To run the test you have to be in ``caravel-sim-infrastructure/cocotb/`` directory and run the ``verify_cocotb.py`` script using the following command
```
python3 verify_cocotb.py -test gpio_test -sim RTL -tag first_test
```
You can know more about the argument options [here](https://github.com/efabless/caravel-sim-infrastructure/tree/main/cocotb#run-a-test)
### 6. Check if the test passed or failed:
When you run the above you will get this ouput:
![image](https://user-images.githubusercontent.com/79912650/224953877-3a3f11b3-d99b-4391-8912-0d9f3c50ed03.png)
It shows that there is an error in the firmware c-code and it could'nt be compiled. You should check the `firmware.log` log file in the `caravel-sim-infrastructure/cocotb/sim/first_test/RTL-gpio_test/` directory to check any firmware errors.
In the log file you will find this error:
```
/home/nouran/caravel_user_project/verilog/dv/cocotb/gpio_test/gpio_test.c: In function 'main':
/home/nouran/caravel_user_project/verilog/dv/cocotb/gpio_test/gpio_test.c:7:24: error: 'GPIO_MODE_MGMT_STD_OUT' undeclared (first use in this function); did you mean 'GPIO_MODE_MGMT_STD_OUTPUT'?
    7 |    configure_all_gpios(GPIO_MODE_MGMT_STD_OUT);
      |                        ^~~~~~~~~~~~~~~~~~~~~~
      |                        GPIO_MODE_MGMT_STD_OUTPUT
/home/nouran/caravel_user_project/verilog/dv/cocotb/gpio_test/gpio_test.c:7:24: note: each undeclared identifier is reported only once for each function it appears in
Error: when generating hex
```
### 7. Modify the firmware:
The error was because passign the wrong gpio mode name. To fix this, you should change `configure_all_gpios(GPIO_MODE_MGMT_STD_OUT);` to `configure_all_gpios(GPIO_MODE_MGMT_STD_OUTPUT);` and rerun. 
### 8. Check if the test passed or failed:
 When you rerun you will get the following output:
 ![image](https://user-images.githubusercontent.com/79912650/224955545-b0b6f113-7fe2-4fb5-9ec7-1e8f86388d26.png)

The test has failed. You should check the `gpio_test.log` log file in the directory `caravel-sim-infrastructure/cocotb/sim/first_test/RTL-gpio_test/`. You will find the following error message:
```
1516250.00ns ERROR    cocotb                             [TEST] Fail the gpio value is :'0x8f' expected 0xf8
1516250.00ns INFO     cocotb.regression                  repot_test.<locals>.wrapper_func [31mfailed[49m[39m
                                                         Traceback (most recent call last):
                                                           File "/home/nouran/caravel-sim-infrastructure/cocotb/tests/common_functions/test_functions.py", line 120, in wrapper_func
                                                             raise cocotb.result.TestComplete(f"Test failed {msg}")
                                                         cocotb.result.TestComplete: Test failed with (0)criticals (1)errors (0)warnings 
1516250.00ns INFO     cocotb.regression                  ********************************************************************************************************************************
                                                         ** TEST                                                                    STATUS  SIM TIME (ns)  REAL TIME (s)  RATIO (ns/s) **
                                                         ********************************************************************************************************************************
                                                         ** tests.common_functions.test_functions.repot_test.<locals>.wrapper_func  [31m FAIL [49m[39m    1516250.00          23.21      65328.32  **
                                                         ********************************************************************************************************************************
                                                         ** TESTS=1 PASS=0 FAIL=1 SKIP=0                                                      1516250.00          23.23      65274.07  **
                                                         ********************************************************************************************************************************
```
This means the result weren't as expected and test failed message was raised because of the cocotb.log.error() function. 
### 9. Modify the python test bench:
The error is because the expected value (0xF8) is not equal to the gpios value (0x8F). To fix this change the expected value to 0x8F `expected_gpio_value = 0x8F` and rerun
### 8. Check if the test passed or failed:
When you rerun you will get this output:
![image](https://user-images.githubusercontent.com/79912650/224956534-16bb223b-a6bd-4467-b1c3-2b90df381003.png)

This means the test has passed you can check the `compilation.log` file in the directory `caravel-sim-infrastructure/cocotb/sim/first_test/RTL-gpio_test/` which contains all the logs (for the C firmware and python testbench) and you can see the following messages:
```
 1050.00ns INFO     cocotb                              [caravel] finish resetting
1516250.00ns INFO     cocotb                             All gpios '00000000000000000000000000000010001111'
1516250.00ns INFO     cocotb                             [TEST] Pass the gpio value is '0x8f'
1516250.00ns INFO     cocotb                             Test passed with (0)criticals (0)errors (0)warnings 
1516250.00ns INFO     cocotb                             Cycles consumed = 60650 recommened timeout = 61257 cycles
1516250.00ns INFO     cocotb.regression                  repot_test.<locals>.wrapper_func passed
1516250.00ns INFO     cocotb.regression                  ********************************************************************************************************************************
                                                         ** TEST                                                                    STATUS  SIM TIME (ns)  REAL TIME (s)  RATIO (ns/s) **
                                                         ********************************************************************************************************************************
                                                         ** tests.common_functions.test_functions.repot_test.<locals>.wrapper_func   PASS     1516250.00          23.38      64861.05  **
                                                         ********************************************************************************************************************************
                                                         ** TESTS=1 PASS=1 FAIL=0 SKIP=0                                                      1516250.00          23.39      64817.47  **
                                                         ********************************************************************************************************************************
```
which shows that the test has passed successfully. 
