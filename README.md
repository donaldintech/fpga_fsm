# Implementing a Finite State Machine(FSM) on an FPGA board.
## Background Info ##





## Requirements & Installation ##

1.	Quartus Web II software (This project uses ver. 13.0.1)
2.	Cyclone II FPGA Development kit (EP2C20F484C7N)

-The installation steps are similar to the tutorial on 4 segment display, which you can find in the following repo.
 [fpga segment display](https://github.com/donaldintech/fpga_segment_display)


## Programming in VHDL ##

Breaking down our program into sections provide an easier way to understand what each section does and represent.

**•	Libraries**

Define the standard libraries which facilitate design reuse and standard data types for model exchange, reuse, and synthesis. 
```
library ieee;
use ieee.std_logic_1164.all;
```
 
**•	Entity**

Define an entity which contains all the input & output ports to be used in the program
```
entity fsm_fpga is
    port (
        clk   : in std_logic;
        sw1   : in std_logic;
        rst   : in std_logic;
        LED1  : out std_logic;
        LED2  : out std_logic;
        LED3  : out std_logic;
        LED4  : out std_logic
    );
end entity;
```
 
**•	Architecture**

The main part of our program, contains two sections: 

**1.Signal declaration** 

-This is where you declare all the signals and variables to be used in the processes that follow
```
 type State_type is (A, B, C, D);  -- Define the states
    signal State : State_type;        -- Create a signal that uses the different states
    
    -- Signals for debouncing and synchronization
    signal btn_sync : std_logic_vector(1 downto 0);  -- Synchronized button signal
    signal btn_debounce : std_logic;  -- Debounced button signal
    signal btn_pressed : std_logic;   -- Detected button press
    
    signal rst_sync : std_logic_vector(1 downto 0);  -- Synchronized reset signal
    signal rst_debounce : std_logic;  -- Debounced reset signal
```

 **2.Process definitions** 

**•	Synchronize button signal**

-Our first process is to synchronize the button input signal(btn) with the fpga clock signal(clk) since button presses are asynchronous events, and directly using them in synchronous logic can lead to metastability and unpredictable behavior.
```
 sync_btn: process(clk)
    begin
        if rising_edge(clk) then
            btn_sync <= btn_sync(0) & sw1;
        end if;
    end process;
 ```
-The reset logic ensures the synchronized button signal starts from a safe initial condition, then during a rising edge of the clock signal, the current state of the least significant bit (btn_sync(0)) is concatenated with the current button input (btn) to obtain a new synchronized button signal.<br/>
-This shifts the previous button state to the left and inserts the current button state as the new least significant bit; thus providing a stable signal to be used in subsequent processes.

**•	Debounce button press**

-This process ensures that the system detects a stable and clean button press without any false triggers (bounced )
```
debounce_btn: process(clk)
    begin
        if rst = '0' then
            btn_debounce <= '1';  -- Set to '1' because button is not pressed
        elsif rising_edge(clk) then
            if btn_sync(1) = '0' then
                btn_debounce <= '0';  -- Button pressed
            else
                btn_debounce <= '1';  -- Button not pressed
            end if;
        end if;
    end process;
```

-The reset sets the debounced button to an off state(1) while a rising edge of the clock ensures that when the new value of synchronized button signal is asserted low (0), it is matched to the debounced signal, representing a button press.

**•	Detect button press**

-The process ensures the system detects a stable and valid button press event, avoiding multiple or false detections.
```
 detect_btn_press: process(clk)
    begin
        if rst_debounce = '0' then
            btn_pressed <= '1';  -- No button press during reset
        elsif rising_edge(clk) then
            if btn_debounce = '0' and btn_sync(1) = '1' then
                btn_pressed <= '0';  -- Button press detected
            else
                btn_pressed <= '1';  -- No button press
            end if;
        end if;
    end process;
```
 
-A low debounced signal and a stable synchronized signal provide conditions for a valid detection of a button press (0).

**•	Synchronize and Debounce reset signal**

-After synchronizing & debouncing the button input, we can now do the same for the reset button.
```
 sync_rst: process(clk)
    begin
        if rising_edge(clk) then
            rst_sync <= rst_sync(0) & rst;
        end if;
    end process;

    -- Debounce the reset input
    debounce_rst: process(clk)
    begin
        if rst_sync(1) = '0' then
            rst_debounce <= '0';  -- Reset is active
        elsif rising_edge(clk) then
            rst_debounce <= '1';  -- Reset is not active
        end if;
    end process;
```

**•	State Machine Logic**

-Now to our final 2 main processes, we need to understand how our FSM will be implemented.
```
process (clk, rst_debounce) 
    begin 
        if rst_debounce = '0' then  -- Active low reset
            State <= A;
 
        elsif rising_edge(clk) then    -- if there is a rising edge of the clock, then do the stuff below
            case State is
                when A => 
                    if btn_pressed = '0' then 
                        State <= B;
                    else 
                        State <= A;
                    end if; 
                when B => 
                    if btn_pressed = '0' then 
                        State <= C;
                    else 
                        State <= B;    
                    end if; 
                when C => 
                    if btn_pressed = '0' then 
                        State <= D;
                    else
                        State <= C;    
                    end if; 
                when D => 
                    if btn_pressed = '0' then 
                        State <= A;
                    else 
                        State <= D;    
                    end if; 
                when others =>
                    State <= A;
            end case; 
        end if; 
    end process;
```
  


**•	LED Output**

-
```
 process(State)
    begin
        case State is
            when A =>
                LED1 <= '1';
                LED2 <= '0';
                LED3 <= '0';
                LED4 <= '0';
            when B =>
                LED1 <= '0';
                LED2 <= '1';
                LED3 <= '0';
                LED4 <= '0';
            when C =>
                LED1 <= '0';
                LED2 <= '0';
                LED3 <= '1';
                LED4 <= '0';
            when D =>
                LED1 <= '0';
                LED2 <= '0';
                LED3 <= '0';
                LED4 <= '1';
            when others =>
                LED1 <= '0';
                LED2 <= '0';
                LED3 <= '0';
                LED4 <= '0';
        end case;
    end process;
```

-When the reset is asserted, all signals & segments are turned off but during the rising edge of the cock cycle; the period_count signal (which represents number of clock cycles and is computed from 50 MHz clock and 0.5 mS period)  increments until 25000, meaning it keeps displaying for that long.<br/>
-Otherwise the period is reset to 0 and thus the display_sel signal determines the segment to display using a modulo operator; whose result is utilized in the case statement.<br/><br/>
-FIND THE WHOLE CODE BELOW: 

```
library IEEE;
use ieee.std_logic_1164.all;

entity fsm_fpga is
    port (
        clk   : in std_logic;
        sw1   : in std_logic;
        rst   : in std_logic;
        LED1  : out std_logic;
        LED2  : out std_logic;
        LED3  : out std_logic;
        LED4  : out std_logic
    );
end entity;

architecture rtl of fsm_fpga is
    type State_type is (A, B, C, D);  -- Define the states
    signal State : State_type;        -- Create a signal that uses the different states
    
    -- Signals for debouncing and synchronization
    signal btn_sync : std_logic_vector(1 downto 0);  -- Synchronized button signal
    signal btn_debounce : std_logic;  -- Debounced button signal
    signal btn_pressed : std_logic;   -- Detected button press
    
    signal rst_sync : std_logic_vector(1 downto 0);  -- Synchronized reset signal
    signal rst_debounce : std_logic;  -- Debounced reset signal

begin

    -- Synchronize the button input to the clock domain
    sync_btn: process(clk)
    begin
        if rising_edge(clk) then
            btn_sync <= btn_sync(0) & sw1;
        end if;
    end process;

    -- Debounce the button input
    debounce_btn: process(clk)
    begin
        if rst = '0' then
            btn_debounce <= '1';  -- Set to '1' because button is not pressed
        elsif rising_edge(clk) then
            if btn_sync(1) = '0' then
                btn_debounce <= '0';  -- Button pressed
            else
                btn_debounce <= '1';  -- Button not pressed
            end if;
        end if;
    end process;

    -- Detect the button press
    detect_btn_press: process(clk)
    begin
        if rst_debounce = '0' then
            btn_pressed <= '1';  -- No button press during reset
        elsif rising_edge(clk) then
            if btn_debounce = '0' and btn_sync(1) = '1' then
                btn_pressed <= '0';  -- Button press detected
            else
                btn_pressed <= '1';  -- No button press
            end if;
        end if;
    end process;

    -- Synchronize the reset input to the clock domain
    sync_rst: process(clk)
    begin
        if rising_edge(clk) then
            rst_sync <= rst_sync(0) & rst;
        end if;
    end process;

    -- Debounce the reset input
    debounce_rst: process(clk)
    begin
        if rst_sync(1) = '0' then
            rst_debounce <= '0';  -- Reset is active
        elsif rising_edge(clk) then
            rst_debounce <= '1';  -- Reset is not active
        end if;
    end process;

    -- State machine process
    process (clk, rst_debounce) 
    begin 
        if rst_debounce = '0' then  -- Active low reset
            State <= A;
 
        elsif rising_edge(clk) then    -- if there is a rising edge of the clock, then do the stuff below
            case State is
                when A => 
                    if btn_pressed = '0' then 
                        State <= B;
                    else 
                        State <= A;
                    end if; 
                when B => 
                    if btn_pressed = '0' then 
                        State <= C;
                    else 
                        State <= B;    
                    end if; 
                when C => 
                    if btn_pressed = '0' then 
                        State <= D;
                    else
                        State <= C;    
                    end if; 
                when D => 
                    if btn_pressed = '0' then 
                        State <= A;
                    else 
                        State <= D;    
                    end if; 
                when others =>
                    State <= A;
            end case; 
        end if; 
    end process;

    -- LED Output Control
    process(State)
    begin
        case State is
            when A =>
                LED1 <= '1';
                LED2 <= '0';
                LED3 <= '0';
                LED4 <= '0';
            when B =>
                LED1 <= '0';
                LED2 <= '1';
                LED3 <= '0';
                LED4 <= '0';
            when C =>
                LED1 <= '0';
                LED2 <= '0';
                LED3 <= '1';
                LED4 <= '0';
            when D =>
                LED1 <= '0';
                LED2 <= '0';
                LED3 <= '0';
                LED4 <= '1';
            when others =>
                LED1 <= '0';
                LED2 <= '0';
                LED3 <= '0';
                LED4 <= '0';
        end case;
    end process;

end architecture;


``` 





## Preparing the Code ##

-Click on the SIMULATE icon to ensure no errors exist in the code and monitor the following section of the software window, which shows the progress.

![image](https://github.com/user-attachments/assets/45c9c377-3b10-4117-9238-4e086b12e27b)


-If everything is green, click on the PIN PLANNER icon to specify FPGA pins. <br/>
A new window appears where you can also specify the Voltage levels alongside the location of the pins; all with the help of the User Manual or Datasheet.

![image](https://github.com/user-attachments/assets/28d17c3a-8ece-471a-a189-6dd31f523092)


-SIMULATE one more time to ensure the pins are allocated.<br/>
-The final section is to program the FPGA ; Click on the PROGRAMMER icon which opens a new window

![image](https://github.com/user-attachments/assets/a84de246-2dbc-4779-82cd-3ceaf1effa62)


-You might need to setup your hardware if it does not show up. Some FPGA boards use an external USB blaster but ours is embedded-so if you have any problem here you need to update drivers in the Device Manager automatically or manually by navigating to the location of the downloaded Altera package.

 ![image](https://github.com/user-attachments/assets/ae85f031-d358-4e6d-a6f3-11023d5ac519)


-If everything is okay, click on ADD FILE , then click on the .sof file which has a volatile memory, therefore disconnecting the board loses the program; otherwise go for the .pof file.

-Click on START  and you should see a green progress bar

 ![image](https://github.com/user-attachments/assets/12e4cd6e-1cf4-430b-9b6a-bb800f9ba2e5)


-The board should now behave as programmed.

