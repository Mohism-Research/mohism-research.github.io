# Principle

We usually use a general mathematical eqation to calculate the velocity:

$$
Velocity = \frac{\Delta Position}{\Delta T} 
$$

If we want to calculate the velocity of rotary encoder, the changed position $\Delta Position$ should be changed radian.

# Implementation

The BISS commnucation protocol is usually used for reading position information from encoder. At present We don't need to consider the details of BISS, we only need to know there is a sample frequence to fetch the encoding information. In our application, the sample frequence is:

$$
F_{s} = \frac{125,000,000}{9300}
$$

The general solution uses an array in VHDL to store consecutive position information, then let the latest position minus the oldest position, and finally divide the period elapsed.

## Confirm the number of elements of the array

Actually we can choose the number of elements arbitrarily, but in our application, we need to simulate the scenario that 26 elements collected at the frequence of 2000 HZ, because our algorithm engineer thought the velocity calculaed under this case is good, and our system control frequence is 2000HZ. Therefore the velocity's calculation equation is below:

$$
\begin{align*}
& F_{System} = 2000 HZ \\
& Velocity = \frac{P(25) - p(0)}{\frac{1}{F_{System}}*25} = (P(25) - p(0))*80 \\
\end{align*}
$$

If we want to completely mimic the computation above, we have to know how many times the system control frequence over the encoder sample frequence. This number is calculated below:

$$
n = \frac{\frac{1}{F_{System}}}{\frac{1}{F_{s}}} = \frac{F_{s}}{F_{System}} = \frac{\frac{125,000,000}{9300}}{2000} = 6.72
$$

Then we use this number $n$ to multipile 25, the result is our array's size. It equas 168.

## The VHDL implementation

Firstly I define a constant to indicate the number of elements in an array:

```vhdl
  constant Buffer_Size : integer := 169;
```

Then we build the array type, and then instantiate it.

```vhdl
type encoder_buffer is array (0 to Buffer_Size) of std_logic_vector(31 downto 0);
signal buffer_position : encoder_buffer := (others => (others => '0'));
```

The main part of calculating velocity is below:

```vhdl
VELOCITY_CALCULATE : process (CLK)
        constant full_range : integer := 2**Encoder_BitWidth;
        constant one_quater_full_range    : integer :=  2**(Encoder_BitWidth - 2);
        constant three_quater_full_range  : integer :=  3*(2**(Encoder_BitWidth - 2));
        variable velocity_integer : integer := 0;
        variable current_p : integer := 0;
        variable previous_p : integer := 0;
    begin
        if (CLK'event and CLK = '1') then
            if sig_biss_latching_falling = '1' then
                l_encoder: for i in 0 to (Buffer_Size - 1) loop
                    buffer_position(i + 1) <= buffer_position(i);
                end loop;
                buffer_position(0) <= sig_position;

                current_p  := to_integer(unsigned(buffer_position(0)));
                previous_p := to_integer(unsigned(buffer_position(Buffer_Size)));

                if current_p < one_quater_full_range and previous_p > three_quater_full_range then
                  velocity_integer := current_p - previous_p + full_range;
                elsif current_p > three_quater_full_range and previous_p < one_quater_full_range then
                  velocity_integer := current_p - previous_p - full_range;
                else
                  velocity_integer := current_p - previous_p;
                end if;

                sig_veloctiy  <= std_logic_vector(to_signed(velocity_integer*80, 32));
            end if;
        end if;
    end process;
```